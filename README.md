# Pocketful of Somethings

*collect anything. carry everything.*

A personal canvas for whatever you're collecting in life — skills, goals, achievements, books, places, anything. Each something is an orb on a dark canvas, gathered as you go, in any order, at any time. No sequence. No hierarchy. Just yours.

Built as a single `index.html` file. No framework, no build step, no backend. Hosted on GitHub Pages. Data persisted cross-device via a GitHub Gist.

---

## Table of contents

1. [What it looks like](#what-it-looks-like)
2. [Features](#features)
3. [Project structure](#project-structure)
4. [How to set it up from scratch](#how-to-set-it-up-from-scratch)
5. [Architecture deep-dive](#architecture-deep-dive)
   - [HTML skeleton](#step-1--html-skeleton)
   - [CSS design system](#step-2--css-design-system)
   - [Pixi.js setup](#step-3--pixijs-setup-webgl-canvas)
   - [Data model](#step-4--data-model)
   - [Color extraction](#step-5--color-extraction-canvas-pixel-sampling)
   - [Status dots](#step-6--status-dots)
   - [D3 force simulation and boundary enforcement](#step-7--d3-force-simulation--boundary-enforcement)
   - [Hybrid Pixi + DOM rendering](#step-8--hybrid-pixi--dom-rendering)
   - [Responsive icon and text sizing](#step-9--responsive-icon-and-text-sizing)
   - [Icon picker with Iconify](#step-10--icon-picker-with-iconify)
   - [Admin mode](#step-11--admin-mode)
   - [GitHub Gist sync](#step-12--github-gist-sync)
   - [GitHub Actions deployment and token injection](#step-13--github-actions-deployment--token-injection)
6. [Problems encountered and how they were solved](#problems-encountered-and-how-they-were-solved)
7. [Known limitations and future improvements](#known-limitations-and-future-improvements)
8. [Tech used](#tech-used)

---

## What it looks like

- A dark canvas filled with circles of varying sizes — each one a skill or goal
- **Achieved orbs**: filled background (darkened dominant color from the icon), glowing ring and status dot in the icon's most vibrant color
- **In-progress orbs**: transparent with a dashed white outline, purple status dot
- Status dots sit on the circumference of each orb, angled dynamically based on orb position on canvas
- Orbs are laid out using a custom 2D physics engine — they fall under gravity, pack like liquid particles, and never overlap the canvas edges
- XP bar and achieved/in-progress legend in the header
- Click any orb to see its full detail in a popup

---

## Features

- Custom 2D physics engine — gravity, mass, position-based collision solver, and velocity damping give orbs a liquid-particle feel
- Achieved orbs are heavier (mass 2.0) and fall first, settling at the bottom; in-progress orbs (mass 1.0) drop in after and land above
- Hard canvas boundary constraints — orbs (including glow ring and dashed outline) never go outside the visible area
- Dominant color extraction from each emoji/image (canvas pixel sampling) for background fill and glow — with automatic CORS proxy fallback for external image URLs
- Responsive icon and text sizing via CSS `--r` custom property driven by each orb's radius
- Admin mode with password protection (secret triple-click trigger)
- Add, edit, delete orbs — admin only
- Toggle achieved/in-progress status — admin only
- Icon support: searchable icon picker (200k+ Iconify icons), file upload (images + SVG), or image URL
- Images render at icon size (fit-within, no cropping or upscaling) — consistent with emoji icons
- Cross-device data sync via GitHub Gist (JSON)
- Single HTML file — deployable to GitHub Pages with no build step

---

## Project structure

```
your-repo/
├── .github/
│   └── workflows/
│       └── deploy.yml   ← GitHub Actions deployment pipeline
├── index.html            ← the entire app
├── README.md             ← this file
└── CLAUDE.md             ← AI project brief (for Claude Code)
```

---

## How to set it up from scratch

### Prerequisites

- A GitHub account
- A public or private GitHub repository
- A GitHub Personal Access Token with only `gist` scope

### Step A — Create a GitHub Gist

1. Go to [gist.github.com](https://gist.github.com)
2. Create a **secret** Gist
3. Filename: `pocketful.json`
4. Content: `[]`
5. Click **Create secret gist**
6. Copy the Gist ID from the URL bar — it's the long hex string at the end: `https://gist.github.com/YOUR_USERNAME/GIST_ID_HERE`

### Step B — Create a Personal Access Token

1. Go to github.com → **Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Scopes: tick **only `gist`** — nothing else
4. Copy the token immediately. GitHub will never show it again.

### Step C — Fill in your constants (local only)

At the top of the `<script>` block in `index.html`, fill in:

```js
const ADMIN_PASSWORD = 'pocketful2025';       // change this to something private
const OWNER_GIST_ID  = 'your-gist-id-here';   // from Step A
const OWNER_TOKEN    = 'PASTE_YOUR_NEW_TOKEN_HERE'; // keep this placeholder for git commits
const GIST_FILE      = 'pocketful.json';
const OWNER_USERNAME = 'YourGitHubUsername';
```

**Important:** The line `OWNER_TOKEN = 'PASTE_YOUR_NEW_TOKEN_HERE'` is exactly what gets committed to the repo. The real token is injected at deploy time by GitHub Actions. Do not replace the placeholder text in the file you commit.

For local development, you can temporarily paste your real token to test Gist sync. Replace it back with the placeholder before committing.

### Step D — Configure GitHub Pages

In your repo:
1. **Settings → Pages → Source: GitHub Actions** (not "Deploy from a branch")

### Step E — Add the GitHub Actions secret

1. Repo → **Settings → Secrets and variables → Actions → New repository secret**
2. Name: `INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS`
3. Value: your `ghp_...` token from Step B

### Step F — Add the deploy workflow

Create `.github/workflows/deploy.yml` with this exact content:

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - uses: actions/checkout@v4

      - name: Inject token
        run: |
          sed -i "s/PASTE_YOUR_NEW_TOKEN_HERE/${{ secrets.INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS }}/g" index.html

      - uses: actions/configure-pages@v4

      - uses: actions/upload-pages-artifact@v3
        with:
          path: .

      - uses: actions/deploy-pages@v4
        id: deployment
```

### Step G — Push and deploy

```bash
git add .
git commit -m "initial commit"
git push origin main
```

The Actions runner will replace `PASTE_YOUR_NEW_TOKEN_HERE` with the real secret and deploy to GitHub Pages. Your committed `index.html` always has the placeholder — the real token never touches the repo.

---

## Architecture deep-dive

Everything is in one `index.html`. The script is divided into sections with comment banners:

```
CONFIG → MODE DETECTION → STATE → PIXI.JS SETUP → COLOR EXTRACTION
→ DOT HELPERS → D3 FORCE SIMULATION → PIXI RENDER FRAME
→ INIT/RESIZE → XP → GIST SYNC → SECRET KNOCK → DETAIL MODAL
→ ADD/EDIT → DELETE → TOGGLE → ICON FORM HELPERS
→ ONBOARDING → MODAL UTILS → BOOT
```

---

### Step 1 — HTML skeleton

The page has three main areas:

```html
<header>          <!-- XP bar, legend, admin buttons -->
<div id="canvas-wrap">
  <div id="label-layer"></div>   <!-- DOM layer: emoji, images, labels -->
</div>
<!-- Pixi appends its <canvas> here at runtime -->
```

Pixi appends its `<canvas id="pixi-canvas">` into `#canvas-wrap` on initialization. The label layer is stacked on top via `z-index`:

```
z-index 3: .orb-hit divs    (invisible, handle click/hover)
z-index 2: #label-layer     (DOM divs: emoji, images, labels)
z-index 1: #pixi-canvas     (WebGL canvas, pointer-events: none)
```

---

### Step 2 — CSS design system

**CSS variables (`:root`):**

```css
:root {
  --bg:      #07070f;                   /* near-black background */
  --surface: #0f0f1a;                   /* modal backgrounds */
  --border:  rgba(255,255,255,0.08);
  --text:    rgba(255,255,255,0.85);
  --muted:   rgba(255,255,255,0.35);
  --faint:   rgba(255,255,255,0.1);
  --done:    #1D9E75;                   /* achieved green */
  --accent:  #7F77DD;                   /* purple accent */
}
```

**Fonts loaded from Google Fonts:**
- `Roboto` — display/UI font (weights 400–800)
- `Roboto Mono` — monospace for labels and admin UI (weight 400)

```html
<link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;600;700&family=Roboto+Mono:wght@300;400;500&display=swap" rel="stylesheet">
```

**Layout:** `body` is `display: flex; flex-direction: column`. The `header` is `flex-shrink: 0`. The `#canvas-wrap` is `flex: 1`, so it fills all remaining vertical space. `overflow: hidden` prevents scrolling.

**CDN libraries:**

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/pixi.js/7.3.2/pixi.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
```

---

### Step 3 — Pixi.js setup (WebGL canvas)

Pixi is initialized after the DOM is ready:

```js
const wrap     = document.getElementById('canvas-wrap');
const lblLayer = document.getElementById('label-layer');

const app = new PIXI.Application({
  resizeTo: wrap,          // auto-resizes to match the wrapper div
  backgroundAlpha: 0,      // transparent — the CSS body background shows through
  antialias: true,
  resolution: window.devicePixelRatio || 1,
  autoDensity: true,
});
app.view.id = 'pixi-canvas';
app.view.style.zIndex = '1';
wrap.appendChild(app.view);
```

Two containers are added to the stage — order matters for draw layering:

```js
const orbContainer  = new PIXI.Container();  // filled circles + dashed outlines + dots
const glowContainer = new PIXI.Container();  // glow rings (drawn behind orbs)
app.stage.addChild(glowContainer);  // added first = drawn below
app.stage.addChild(orbContainer);   // added second = drawn on top
```

Pixi drives rendering via its own ticker (rAF-based, 60fps). D3 runs physics independently via its own timer. The two are decoupled — the ticker reads whatever positions `simNodes` currently holds and redraws:

```js
app.ticker.add(() => { if (simNodes.length) renderFrame(); });
```

This separation means rendering continues at 60fps as long as the tab is visible, regardless of whether D3's simulation timer is ticking.

---

### Step 4 — Data model

Each skill/goal is a plain JavaScript object:

```js
{
  id: 1,                         // unique integer, auto-incremented
  name: 'QWorld cert',           // short name shown inside orb (max ~32 chars)
  desc: 'Completed quantum...',  // longer description shown in detail popup
  glyph: '⚛️',                   // emoji (used when img is null)
  img: null,                     // base64 data URL, external URL, Iconify SVG URL, or null
  cat: 'Quantum',                // legacy category, not shown on canvas
  achieved: true,                // true = achieved, false = in progress
  xp: 120,                       // tier * 80, used for XP bar
  tier: 2                        // 1–5, controls orb size
}
```

**Tier scale** — multipliers for radius computation:

```js
const TIER_SCALE = { 1: 0.48, 2: 0.78, 3: 1.15, 4: 1.68, 5: 2.35 };
```

**Radii computation** — computed fresh on every layout build from the canvas dimensions, so orbs always fill available space proportionally:

```js
function computeRadius(s, W, H) {
  const area    = W * H;
  const totalSq = SKILLS.reduce((a, x) => a + TIER_SCALE[x.tier] ** 2, 0);
  const unit    = Math.sqrt(area * 0.88 / (totalSq * Math.PI));
  return Math.max(22, Math.round(unit * TIER_SCALE[s.tier]));
}
```

The `0.88` factor means orbs collectively fill ~88% of canvas area (leaving breathing room). `Math.max(22, ...)` enforces a minimum radius so tiny tier-1 orbs don't become invisible.

**`DEFAULT_SKILLS`** seeds the Gist on first load if the file doesn't exist yet. It contains the actual skills of the owner. Do not touch this array without being explicitly asked.

---

### Step 5 — Color extraction (canvas pixel sampling)

Each orb derives two colors from its icon for achieved-state rendering:

1. **Background fill** (`bg`) — darkened average of all non-transparent pixels
2. **Glow color** (`glow`) — the most vibrant pixel, scored by `saturation × brightness`

```js
const colorCache = {};

function extractColors(src, isEmoji, id, cb) {
  if (colorCache[id]) { cb(colorCache[id]); return; }

  const SZ = 48;
  const cv  = document.createElement('canvas');
  cv.width = cv.height = SZ;
  const ctx = cv.getContext('2d');

  function process() {
    let rS=0, gS=0, bS=0, n=0, bestSat=0, gR=180, gG=180, gB=180;
    try {
      const px = ctx.getImageData(0, 0, SZ, SZ).data;
      for (let i = 0; i < px.length; i += 4) {
        if (px[i+3] < 30) continue;          // skip near-transparent pixels
        const r=px[i], g=px[i+1], b=px[i+2];
        rS+=r; gS+=g; bS+=b; n++;            // accumulate for average (bg)

        // score = saturation × relative brightness (picks most vivid pixel for glow)
        const mx=Math.max(r,g,b), mn=Math.min(r,g,b);
        const sat   = mx === 0 ? 0 : (mx - mn) / mx;
        const score = sat * (mx / 255);
        if (score > bestSat) { bestSat=score; gR=r; gG=g; gB=b; }
      }
    } catch(e) { cb(null); return; }
    if (!n) { cb(null); return; }

    const d  = 0.25;  // darken factor for background fill
    const bg   = hexN(Math.round(rS/n * d), Math.round(gS/n * d), Math.round(bS/n * d));
    const glow = hexN(gR, gG, gB);
    colorCache[id] = { bg, glow };
    cb(colorCache[id]);
  }

  if (isEmoji) {
    // render emoji to canvas using native font rendering
    ctx.clearRect(0, 0, SZ, SZ);
    ctx.font = `${SZ * 0.78}px serif`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(src, SZ/2, SZ/2);
    process();
  } else {
    const loadImg = (imgSrc, onFail) => {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => { ctx.clearRect(0,0,SZ,SZ); ctx.drawImage(img,0,0,SZ,SZ); process(); };
      img.onerror = () => onFail ? onFail() : cb(null);
      img.src = imgSrc;
    };
    if (src.startsWith('data:')) {
      // data: URLs must NOT have crossOrigin set — some browsers reject the combination
      const img = new Image();
      img.onload = () => { ctx.clearRect(0,0,SZ,SZ); ctx.drawImage(img,0,0,SZ,SZ); process(); };
      img.onerror = () => cb(null);
      img.src = src;
    } else {
      // try direct first; if CORS-blocked, retry via proxy that adds CORS headers
      loadImg(src, () => loadImg(`https://wsrv.nl/?url=${encodeURIComponent(src)}`, null));
    }
  }
}
```

**Why two separate branches for `data:` vs external URLs:**
- External URLs require `crossOrigin = 'anonymous'` to make `getImageData()` work — but only if the server sends `Access-Control-Allow-Origin` headers. If it doesn't, the `onerror` fires and the proxy fallback kicks in.
- `data:` URLs don't need CORS at all. Setting `crossOrigin = 'anonymous'` on a `data:` URL makes some browsers (notably Safari) treat it as a failed cross-origin load and fire `onerror` even though the image is perfectly valid. So `data:` URLs skip `crossOrigin` entirely.

**Fallback color:** if extraction fails (cb receives `null`), the fallback used for both glow ring and status dot is `0x7F77DD` (accent purple). This is a neutral placeholder, not a meaningful color. Previously `0x1D9E75` (green) was used, which looked like an intentional color — changed to purple to make it clearly identifiable as a fallback.

**Cache invalidation:** when a skill is edited, `delete colorCache[s.id]` forces re-extraction on the next render tick.

---

### Step 6 — Status dots

Each orb has a small filled dot on its circumference. For achieved orbs, it's colored with the glow color. For pending orbs, it's accent purple.

**Size** — scales with orb radius, clamped between 4 and 10 pixels:

```js
function dotRadius(r) {
  return Math.min(10, Math.max(4, r * 0.12));
}
```

**Position** — the dot angle is computed per orb so it's stable across frames but unique per skill, and biased toward the canvas centre so dots near edges don't get clipped:

```js
function dotOffset(r, id, cx, cy, W, H) {
  // Deterministic "random" angle seeded from id — stable across frames
  const seeded    = ((id * 2654435761) >>> 0) / 4294967296;
  const baseAngle = seeded * Math.PI * 2;

  // Angle pointing toward the canvas centre
  const toCentreAngle = Math.atan2(H/2 - cy, W/2 - cx);

  // Blend: 45% seeded angle + 55% toward-centre angle
  const blend = 0.55;
  const ax = Math.cos(baseAngle) * (1-blend) + Math.cos(toCentreAngle) * blend;
  const ay = Math.sin(baseAngle) * (1-blend) + Math.sin(toCentreAngle) * blend;
  const angle = Math.atan2(ay, ax);

  const dr = dotRadius(r);
  return {
    x: cx + Math.cos(angle) * r,   // placed exactly on the circumference
    y: cy + Math.sin(angle) * r,
    r: dr
  };
}
```

The dot is re-computed every render tick, so it tracks the orb as it moves during simulation.

---

### Step 7 — Physics engine & boundary enforcement

The layout uses a custom 2D position-based dynamics engine. D3 was removed — it was designed for graph layouts and could not guarantee hard collision avoidance or support gravity/mass.

#### Physics constants

```js
const GRAVITY             = 0.30;   // px/frame² downward acceleration
const ACHIEVED_MASS       = 2.0;    // achieved orbs are heavier — fall faster, settle lower
const PENDING_MASS        = 1.0;    // pending orbs are lighter — arrive later, land higher
const DAMPING             = 0.92;   // velocity multiplier per frame (fluid settling, not bouncy)
const RESTITUTION         = 0.04;   // near-zero bounce on collision/wall contact
const SOLVER_ITERS        = 12;     // collision resolution passes per frame
const COLLISION_MARGIN    = 2;      // px clearance between outer visual edges
const ORB_SPAWN_INTERVAL  = 6;      // frames between each sequential orb drop (~0.1s at 60fps)
const CANVAS_FILL_RATIO   = 0.62;   // fraction of canvas area the orb set fills
```

#### Sequential spawning — one by one, achieved first

On first load, orbs drop one at a time from a random x position at the top of the screen. Achieved orbs are queued first, in-progress orbs after. One orb is released every `ORB_SPAWN_INTERVAL` frames (~0.1s), so the full set rains in over ~1–2 seconds. Achieved orbs have more time to settle at the bottom before in-progress orbs arrive:

```js
// All orbs queued at init (achieved first, then pending)
[...achieved, ...pending].forEach(s => spawnQueue.push({ s, r: radii[s.id] }));

// physicsTick: drop one orb every ORB_SPAWN_INTERVAL frames
if (spawnQueue.length > 0 && physicsFrame % ORB_SPAWN_INTERVAL === 0) {
  const { s, r } = spawnQueue.shift();
  simNodes.push(spawnNode(s, r, w));
}
physicsFrame++;
```

#### physicsTick() — four steps per frame

**Step A — Gravity + damping + integration:**
```js
n.vy += GRAVITY * n.mass;   // heavier orbs accelerate faster
n.vx *= DAMPING;
n.vy *= DAMPING;
n.x  += n.vx;
n.y  += n.vy;
```

**Step B — Boundary walls** (floor, left, right, ceiling): position clamped, velocity reversed and damped by `RESTITUTION`.

**Step C — Position-based collision solver** (12 iterations):
```js
// For every overlapping pair (a, b):
const overlap = minDist - dist;
// Push apart proportional to inverse mass — heavier moves less
a.x -= nx * overlap * (b.mass / totalMass);
b.x += nx * overlap * (a.mass / totalMass);
// Impulse transfer along contact normal — gives sliding-past behaviour
const J = -(1 + RESTITUTION) * velDot / (1/a.mass + 1/b.mass);
a.vx -= J / a.mass * nx;  b.vx += J / b.mass * nx;
```

**Step D — Re-clamp** after solver may have shifted nodes past walls.

#### Visual outer edge values

- Achieved orbs: `r + 9` — the soft outer glow draws 3 rings at `r+3`, `r+6`, `r+9`
- Pending orbs: `r + 1` — accounts for half the 1.5px dashed stroke width

These are used by `outerR(n)` as the collision/boundary radius throughout the engine.

#### "Alive but settled" feel

No random noise is added. The settled state feels liquid rather than frozen because:
- `DAMPING = 0.92` means velocities decay over ~12 frames, not instantly — impulses from collisions propagate through the pile
- `RESTITUTION = 0.04` means a nudged orb moves slightly and adjusts, like a marble in a jar
- Gravity is always on — all orbs permanently press into each other and the floor

#### Resize handling

A `ResizeObserver` on `#canvas-wrap` calls `initCanvas()` on every size change. `initPhysics()` detects existing nodes (by checking `prevNodes`) and clamps their positions to the new bounds rather than re-spawning them, so orbs don't scatter on resize.

```js
const ro = new ResizeObserver(() => {
  app.resize();
  initCanvas();
});
ro.observe(wrap);
```

---

### Step 8 — Hybrid Pixi + DOM rendering

#### Why hybrid

Pixi renders circles, glows, and dots at 60fps with GPU acceleration. Emoji rendering via WebGL sprites is unreliable cross-platform — different operating systems render the same emoji in different sizes, colors, and styles. Putting emoji in the DOM keeps them crisp and consistent everywhere. Images (including uploaded SVGs and Iconify icons) follow the same DOM path.

#### Pixi rendering (circles, glows, dots)

For each orb, `updatePixiOrb()` redraws three `PIXI.Graphics` objects on every simulation tick:

**Achieved orb:**
```js
// Filled circle with darkened dominant color
circle.beginFill(bgColor, 1);
circle.drawCircle(x, y, r);
circle.endFill();

// Solid glow ring
glow.lineStyle(2.5, glowColor, 0.65);
glow.drawCircle(x, y, r + 1.5);

// Soft outer glow — 3 concentric rings with decreasing opacity
for (let i = 1; i <= 3; i++) {
  glow.lineStyle(i * 2, glowColor, 0.08 / i);
  glow.drawCircle(x, y, r + i * 3);   // r+3, r+6, r+9
}

// Status dot (filled circle on circumference)
const dOff = dotOffset(r, s.id, x, y, w, h);
dot.beginFill(colorCache[s.id]?.glow ? hexToNum(colorCache[s.id].glow) : 0x7F77DD, 1);
dot.drawCircle(dOff.x, dOff.y, dOff.r);
dot.endFill();
```

**Pending orb:**
```js
// Dashed outline — drawn as arc segments
circle.lineStyle(1.5, 0xffffff, 0.22);
const segs = Math.max(12, Math.round(r * 1.2));
for (let i = 0; i < segs; i += 2) {
  const a1 = (i / segs) * Math.PI * 2;
  const a2 = ((i + 0.8) / segs) * Math.PI * 2;
  circle.arc(x, y, r, a1, a2);
  circle.closePath();
}

// Purple status dot
dot.beginFill(0x7F77DD, 0.7);
dot.drawCircle(dOff.x, dOff.y, dOff.r);
dot.endFill();
```

#### DOM rendering (labels)

`updateDomLabel()` manages `.orb-label` divs:

```js
function updateDomLabel(s, node) {
  const { lbl, hit } = domLabels[s.id];
  const { x, y, r }  = node;
  const showName      = r >= 24;  // hide name text for very small orbs

  // Single CSS variable drives all calc() sizing rules
  lbl.style.setProperty('--r', r + 'px');
  lbl.style.left  = x + 'px';
  lbl.style.top   = y + 'px';
  lbl.style.width = (r * 2) + 'px';

  if (s.img) {
    // PERSISTENT <img> ELEMENT — critical for data: URLs
    // If we used lbl.innerHTML = '<img src="data:...">' every tick (60fps),
    // the browser would destroy and recreate the img element faster than it
    // can decode the base64. The image would never become visible.
    // Instead: create the <img> once, reuse it, only update src when it changes.
    let imgEl = lbl.querySelector('img.orb-img');
    if (!imgEl) {
      lbl.innerHTML = '';
      imgEl = document.createElement('img');
      imgEl.className = 'orb-img';
      imgEl.alt = '';
      lbl.appendChild(imgEl);
    }
    if (imgEl.dataset.orbSrc !== s.img) {
      imgEl.dataset.orbSrc = s.img;
      imgEl.src = s.img;
    }

    // Name label below the image
    let nameEl = lbl.querySelector('.orb-name');
    if (showName) {
      if (!nameEl) {
        nameEl = document.createElement('span');
        nameEl.className = 'orb-name';
        lbl.appendChild(nameEl);
      }
      nameEl.textContent = s.name;
    } else if (nameEl) {
      nameEl.remove();
    }
  } else {
    // Emoji path — clear any leftover img from a previous image state
    if (lbl.querySelector('img.orb-img')) lbl.innerHTML = '';
    lbl.innerHTML = `<span class="orb-emoji">${s.glyph}</span>`
      + (showName ? `<span class="orb-name">${s.name}</span>` : '');
  }

  // Hit area (invisible, handles click/hover, z-index 3)
  hit.style.left         = x + 'px';
  hit.style.top          = y + 'px';
  hit.style.width        = (r * 2) + 'px';
  hit.style.height       = (r * 2) + 'px';
  hit.style.pointerEvents = 'all';
}
```

Color extraction fires asynchronously. When it completes, it nudges the simulation to repaint:

```js
extractColors(src, !s.img, s.id, () => {
  if (simulation) simulation.alpha(0.1).restart();
});
```

---

### Step 9 — Responsive icon and text sizing

All icon and text dimensions are CSS-driven via a single custom property `--r` set per label element in JavaScript:

```js
lbl.style.setProperty('--r', r + 'px');
```

The stylesheet then handles everything with `calc()`:

```css
.orb-label {
  --r: 0px;
  position: absolute;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  pointer-events: none;
  text-align: center;
}

/* Emoji size: 50% of orb radius */
.orb-emoji {
  line-height: 1;
  display: block;
  font-size: calc(var(--r) * 0.5);
}

/* Name text: scales with radius, has a 7px floor */
.orb-name {
  font-family: 'Roboto', sans-serif;
  font-weight: 600;
  color: rgba(255,255,255,0.92);
  line-height: 1.2;
  text-align: center;
  word-break: break-word;
  text-shadow: 0 1px 4px rgba(0,0,0,0.7);
  font-size: max(7px, calc(var(--r) * 0.17));
  width: calc(var(--r) * 1.32);
  margin-top: calc(var(--r) * 0.04);
}

/* Image: constrained to orb radius, never upscaled */
.orb-img {
  display: block;
  flex-shrink: 0;
  max-width: var(--r);
  max-height: var(--r);
  width: auto;
  height: auto;
}
```

**Why `max-width`/`max-height` on images and not `width`/`height`:**
Setting `width: var(--r); height: var(--r)` would stretch a small icon to fill the entire allocated space. `max-width`/`max-height` with `width: auto; height: auto` means "scale down if too large, but never scale up." This makes images behave identically to emoji — they look the same visual weight regardless of the original image size.

**Detail modal image** uses the same pattern:

```css
.modal-img-preview {
  max-width: 72px;
  max-height: 72px;
  width: auto;
  height: auto;
  display: block;
  margin: 0 auto 12px;
  object-fit: contain;
}
```

---

### Step 10 — Icon picker with Iconify

The add/edit form offers three icon modes via tabs: **Icons**, **Upload**, **Link**.

#### Icons tab — Iconify search

The Iconify public API is free, requires no key, and covers 200k+ open-source icons from Material Design, Font Awesome, Tabler, Lucide, Phosphor, and hundreds more collections.

Empty state shows a built-in emoji grid. Typing into the search field triggers a debounced API call:

```js
let _iconSearchTimer = null;

function searchIcons(query) {
  clearTimeout(_iconSearchTimer);
  if (!query.trim()) { buildEmojiGrid(); return; }  // restore default grid

  _iconSearchTimer = setTimeout(async () => {
    const grid = document.getElementById('emoji-grid');
    grid.innerHTML = '<div style="grid-column:1/-1;text-align:center;color:var(--muted);padding:12px;font-size:12px;">searching…</div>';
    try {
      const res  = await fetch(`https://api.iconify.design/search?query=${encodeURIComponent(query.trim())}&limit=54`);
      const data = await res.json();
      if (!data.icons || !data.icons.length) {
        grid.innerHTML = '<div style="...">no results</div>';
        return;
      }
      // data.icons = ['prefix:name', 'prefix:name', ...]
      grid.innerHTML = data.icons.map(icon => {
        const [prefix, name] = icon.split(':');
        const url = `https://api.iconify.design/${prefix}/${name}.svg`;
        const sel = urlImg === url ? ' selected' : '';
        return `<button class="ep-btn${sel}" data-iconurl="${url}" title="${icon}"
          onclick="pickIconImg('${url}')"
          style="display:flex;align-items:center;justify-content:center;padding:4px;">
          <img src="${url}" alt="${name}">
        </button>`;
      }).join('');
    } catch(e) {
      grid.innerHTML = '<div style="...">search failed</div>';
    }
  }, 350);  // 350ms debounce to avoid hammering the API while typing
}

function pickIconImg(url) {
  urlImg = url;
  iconMode = 'url';
  // Highlight the selected button
  document.querySelectorAll('#emoji-grid .ep-btn').forEach(b =>
    b.classList.toggle('selected', b.dataset.iconurl === url)
  );
}
```

SVG icons from Iconify are rendered via `<img src="...svg">` at 19×19px in the grid, and stored as an external URL in `s.img`. They go through the same color extraction path as any external URL image (with CORS proxy fallback).

Icon buttons in the grid:
```css
.ep-btn img { width: 19px; height: 19px; display: block; pointer-events: none; }
```

The search input:
```css
.icon-search {
  width: 100%;
  background: rgba(255,255,255,0.04);
  border: 0.5px solid var(--border);
  border-radius: 8px;
  color: var(--text);
  font-family: 'Roboto', sans-serif;
  font-size: 13px;
  padding: 7px 10px;
  outline: none;
  margin-bottom: 8px;
  display: block;
  transition: border-color 0.15s;
}
.icon-search:focus { border-color: rgba(127,119,221,0.5); }
.icon-search::placeholder { color: var(--muted); }
```

#### Upload tab — file and SVG upload

The file input accepts all image types plus `.svg`:

```html
<input type="file" id="f-img-file" accept="image/*,.svg" onchange="handleFileUpload(this)">
```

```js
function handleFileUpload(input) {
  const file = input.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = e => {
    uploadedImg = e.target.result;  // base64 data: URL
    iconMode = 'upload';
    // show preview, clear emoji selection
  };
  reader.readAsDataURL(file);
}
```

SVG files are read as `data:image/svg+xml;base64,...` — they render correctly as `<img src="data:...">` in both the orb and the detail modal.

#### Link tab — external URL

```js
function handleUrlInput(input) {
  const raw = input.value.trim();
  // Reject non-http(s) protocols (e.g. javascript:) at the input boundary
  if (raw && !raw.startsWith('https://') && !raw.startsWith('http://')) {
    input.style.borderColor = 'rgba(220,80,80,0.6)';
    return;
  }
  input.style.borderColor = '';
  urlImg = raw || null;
  iconMode = urlImg ? 'url' : 'emoji';
  const previewEl = document.getElementById('url-preview');
  if (urlImg) { previewEl.src = urlImg; previewEl.classList.add('show'); }
  else previewEl.classList.remove('show');
}
```

---

### Step 11 — Admin mode

There is no visible admin button. The entry mechanism is a **secret triple-click** on an invisible 44×44px zone at the bottom-left corner of the canvas.

```js
const ADMIN_ZONE_SIZE = 44;
// ... click handler checks if click lands within bottom-left 44×44px of wrap
```

Three dots briefly flash in the bottom-left corner to confirm each click:

```html
<div id="knock-dots">
  <div class="k-dot" id="kd0"></div>
  <div class="k-dot" id="kd1"></div>
  <div class="k-dot" id="kd2"></div>
</div>
```

On the third click, a password prompt appears. Correct password adds `class="admin-mode"` to `<body>`, which CSS uses to reveal admin UI:

```css
body.admin-mode .admin-bar { display: flex; }
body.admin-mode #add-btn { display: block; }
body.admin-mode .modal-admin-actions { display: flex; }
```

**Admin unlocks:**
- `+ add` button in header
- `admin` badge + `exit ✕` in header
- Edit / toggle achieved / delete buttons at the bottom of the detail popup

Admin mode is purely UI-gated. The token is what actually controls write access to the Gist. A visitor using `?id=GIST_ID` who knows the password can enter admin mode but can only modify their own Gist.

---

### Step 12 — GitHub Gist sync

All skill data is stored as a JSON array in `pocketful.json` in a GitHub Gist.

#### Reading

Two-step read to work around CORS on the GitHub API:

```js
async function loadFromGist() {
  // Step 1: Fetch Gist metadata with auth token to get raw_url
  const meta = await fetch(`https://api.github.com/gists/${gistId}`, {
    headers: { Authorization: `token ${token}` }
  });
  const data = await meta.json();
  const rawUrl = data.files[GIST_FILE].raw_url;

  // Step 2: Fetch the raw_url directly — it's a public CDN URL, no CORS block
  const raw = await fetch(rawUrl);
  const skills = await raw.json();
  return skills;
}
```

Fallback if the API fetch fails: `gist.githubusercontent.com/USERNAME/GIST_ID/raw/FILENAME`.

#### Writing

Every mutation (add, edit, delete, toggle) calls `saveToGist()`:

```js
async function saveToGist(skills) {
  const res = await fetch(`https://api.github.com/gists/${gistId}`, {
    method: 'PATCH',
    headers: {
      Authorization: `token ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      files: {
        [GIST_FILE]: { content: JSON.stringify(skills, null, 2) }
      }
    })
  });
  // shows "saved ✓" or "sync failed" in bottom-right
}
```

#### Multi-user support

**Owner view** (no URL params): loads `OWNER_GIST_ID` with `OWNER_TOKEN`.

**Visitor view** (`?id=GIST_ID`): loads that Gist using a token the visitor previously stored in `localStorage` during onboarding.

**Onboarding** (`openOnboarding()`): 3-step modal — enter Gist ID → enter token → enter username. On finish, validates token, creates `pocketful.json` if it doesn't exist, stores token in `localStorage` as `pocketful_token_GIST_ID`, redirects to `?id=GIST_ID`.

---

### Step 13 — GitHub Actions deployment & token injection

The token is never committed. The workflow injects it from a GitHub Secret at deploy time:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - uses: actions/checkout@v4

      - name: Inject token
        run: |
          sed -i "s/PASTE_YOUR_NEW_TOKEN_HERE/${{ secrets.INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS }}/g" index.html

      - uses: actions/configure-pages@v4

      - uses: actions/upload-pages-artifact@v3
        with:
          path: .

      - uses: actions/deploy-pages@v4
        id: deployment
```

**How it works:**
1. The `sed` command does a find-and-replace on the checked-out `index.html` — replacing `PASTE_YOUR_NEW_TOKEN_HERE` with the secret value
2. The modified file is uploaded to GitHub Pages infrastructure — it is never committed back to the repo
3. `main` branch always contains the placeholder text

**Token rotation:** go to Repo → Settings → Secrets and variables → Actions → update the secret → push any commit to trigger a redeploy.

---

## Problems encountered and how they were solved

This section documents every significant problem hit during development, what caused it, and exactly how it was fixed.

---

### Problem 1 — Orbs going behind the header bar

**Symptom:** Orbs would drift up and partially disappear behind the XP bar at the top.

**Root cause:** The `bound` force was clamping `y` to `Math.max(r, ...)` — using just the radius as the top boundary. But the Pixi canvas and `#canvas-wrap` start directly below the header. The orbs were correctly constrained within `#canvas-wrap`'s coordinate space, but the glow rings (extending `r+9` beyond the radius) were visually bleeding above the boundary.

**Fix:** Changed the edge margin to account for the actual visual outer edge of each orb type:

```js
// Before:
const edge = r + dotRadius(r) + 4;  // used just radius + dot
n.y = Math.max(edge, Math.min(H - edge, n.y));

// After:
const edge = n.achieved ? n.r + 9 : n.r + 1;   // accounts for glow (r+9) or outline (r+1)
n.y = Math.max(edge, Math.min(H - edge, n.y));
```

Applied in three places: `bound` force, `simNodes` initial position clamping, and the `renderFrame` hard clamp.

---

### Problem 2 — Orbs glitching after boundary fix

**Symptom:** After fixing boundary enforcement, orbs started vibrating and stuttering.

**Root cause:** Initial attempt to prevent drift was to zero velocities in the bound force:

```js
// BAD — caused glitching:
.force('bound', () => {
  simNodes.forEach(n => {
    const edge = n.achieved ? n.r + 9 : n.r + 1;
    if (n.x < edge) { n.x = edge; n.vx = 0; }   // zeroing velocity
    if (n.x > W - edge) { n.x = W - edge; n.vx = 0; }
    // ... same for y
  });
})
```

D3's `forceCollide` modifies node positions directly (not via velocities). Zeroing `vx/vy` while collision was simultaneously adjusting positions created conflicting updates — each tick the collision force would push nodes apart, the bound force would stop them and zero their velocity, then the next tick would start the whole fight again.

**Fix:** Never touch velocities — only clamp positions. Add a second hard clamp in `renderFrame` after D3 finishes all updates:

```js
// Good bound force — positions only, no velocity modification:
.force('bound', () => {
  simNodes.forEach(n => {
    const edge = n.achieved ? n.r + 9 : n.r + 1;
    n.x = Math.max(edge, Math.min(W - edge, n.x));
    n.y = Math.max(edge, Math.min(H - edge, n.y));
  });
})

// And at the very start of renderFrame, re-clamp after D3's velocity → position update:
function renderFrame() {
  const w = W(), h = H();
  simNodes.forEach(n => {
    const edge = n.achieved ? n.r + 9 : n.r + 1;
    n.x = Math.max(edge, Math.min(w - edge, n.x));
    n.y = Math.max(edge, Math.min(h - edge, n.y));
  });
  // ... rest of rendering
}
```

---

### Problem 3 — Uploaded images completely invisible in orbs

**Symptom:** After uploading an image, orbs with images showed nothing — just empty circles with a name label.

**Root cause:** `updateDomLabel()` was setting `lbl.innerHTML` on every D3 tick (~60fps). For `data:` URL images this meant:

```js
lbl.innerHTML = `<img src="data:image/png;base64,...very long string...">`;
```

The browser creates and immediately destroys the `<img>` element 60 times per second. A `data:` URL image needs time to decode — but the element is destroyed before the browser can decode it. The image never renders.

**Fix:** Use a **persistent `<img>` element** — create it once when first needed, reuse it across ticks, and only update `src` when it changes:

```js
let imgEl = lbl.querySelector('img.orb-img');
if (!imgEl) {
  lbl.innerHTML = '';
  imgEl = document.createElement('img');
  imgEl.className = 'orb-img';
  imgEl.alt = '';
  lbl.appendChild(imgEl);
}
// Only update src if it changed — avoids unnecessary re-decoding
if (imgEl.dataset.orbSrc !== s.img) {
  imgEl.dataset.orbSrc = s.img;
  imgEl.src = s.img;
}
```

This also applies to the name label below the image — create it once, update its `textContent`.

---

### Problem 4 — Images expanding beyond allocated orb size

**Symptom:** Images in orbs were filling the entire circle diameter even when the original image was smaller, and overflowing when it was larger.

**Root cause:** CSS `background-size: contain` (original approach) scales images to fill the container, which means upscaling small images. Then switching to `width: var(--r); height: var(--r)` on the `<img>` also forced images to exactly fill the allocated space.

**Fix:** Use `max-width`/`max-height` instead:

```css
.orb-img {
  display: block;
  flex-shrink: 0;
  max-width: var(--r);   /* shrink to fit if too large */
  max-height: var(--r);
  width: auto;           /* but never upscale */
  height: auto;
}
```

This means a 19×19px SVG icon renders at 19×19px inside a large orb — exactly like how an emoji would render at its natural size, not stretched to fill everything.

---

### Problem 5 — Color extraction returning wrong colors (appearing to always be green or purple)

**Symptom:** All achieved orbs' glow and status dots showed either green (`#1D9E75`) or the fallback purple, never the actual icon color.

**Root cause (part 1):** The fallback color for failed extraction was `0x1D9E75` (the achieved green), making failed extractions indistinguishable from the intentional "done" color.

**Fix:** Changed fallback to `0x7F77DD` (accent purple) so it's obvious when extraction hasn't succeeded yet.

**Root cause (part 2):** External image URLs (e.g. from `learn.microsoft.com`, `qworld.net`) don't serve `Access-Control-Allow-Origin` headers. `crossOrigin = 'anonymous'` on an image from such a URL causes the browser to taint the canvas. `getImageData()` on a tainted canvas throws a security error → `catch(e)` fires → `cb(null)` returns → cache never gets populated → fallback color is always shown.

**Fix:** Two-step CORS proxy fallback:

```js
if (src.startsWith('data:')) {
  // data: URLs: skip crossOrigin entirely (some browsers reject the combination)
  const img = new Image();
  img.onload = () => { ctx.drawImage(img, 0, 0, SZ, SZ); process(); };
  img.onerror = () => cb(null);
  img.src = src;
} else {
  // External URLs: try direct first (works if server sends CORS headers)
  loadImg(src, () =>
    // On CORS failure, retry via proxy that adds Access-Control-Allow-Origin: *
    loadImg(`https://corsproxy.io/?${encodeURIComponent(src)}`, null)
  );
}
```

**Root cause (part 3):** At one point the glow color was changed to use the pixel average (same as background) instead of the most-vibrant pixel. Averaging all pixels in a multicolor icon (e.g. a Microsoft logo with red, green, blue, yellow sections) produces a muddy olive/grey color. The original most-vibrant pixel approach was correct.

**Fix:** Reverted glow to use the pixel with the highest `saturation × brightness` score:

```js
const sat   = mx === 0 ? 0 : (mx - mn) / mx;
const score = sat * (mx / 255);         // higher = more vivid
if (score > bestSat) { bestSat = score; gR = r; gG = g; gB = b; }
```

---

### Problem 6 — Detail modal image expanding beyond 72×72

**Symptom:** Clicking an orb to open the detail popup showed the image at its full original size, expanding the modal.

**Root cause:** The modal had `.modal-emoji { font-size: 48px; }` for emoji display, but when an image was set, the `<img>` in the modal had no size constraints.

**Fix:** Added `modal-img-preview` class with max-size constraints:

```css
.modal-img-preview {
  max-width: 72px;
  max-height: 72px;
  width: auto;
  height: auto;
  display: block;
  margin: 0 auto 12px;
  object-fit: contain;
}
```

Applied in the detail modal rendering code using DOM methods (avoids XSS for user-supplied glyph/img values):

```js
// In openDetail():
const iconWrapEl = document.getElementById('m-icon-wrap');
iconWrapEl.innerHTML = '';
if (s.img) {
  const imgEl = document.createElement('img');
  imgEl.className = 'modal-img-preview';
  imgEl.src = s.img;
  imgEl.alt = '';
  iconWrapEl.appendChild(imgEl);
} else {
  const emojiDiv = document.createElement('div');
  emojiDiv.className = 'modal-emoji';
  emojiDiv.textContent = s.glyph;   // textContent, not innerHTML — safe for any glyph value
  iconWrapEl.appendChild(emojiDiv);
}
```

---

### Problem 7 — Canvas blank on load (orbs never rendered)

**Symptom:** The page loaded with a working header and XP bar, but the canvas was completely empty — no orbs, no labels.

**Root cause:** `renderFrame()` was wired to D3's tick event via `.on('tick', renderFrame)`. D3's simulation timer is implemented using `d3.timer`, which internally uses `requestAnimationFrame`. In environments where the page is not the active focused tab (e.g. preview iframes in development tooling), `requestAnimationFrame` callbacks don't fire. The simulation physics ran zero ticks, so `renderFrame` was never called, so no orbs were ever drawn.

Diagnosis: `simNodes` had 19 entries, `simulation.alpha()` was stuck at 1.0000 (never advanced), and `pixiOrbs` was 0. Calling `renderFrame()` manually instantly produced all 19 orbs — confirming the code was correct but the trigger was broken.

**Fix:** Decouple rendering from D3's timer. Wire `renderFrame` to Pixi's own ticker instead:

```js
// In PIXI.JS SETUP — drives rendering at 60fps whenever the tab is visible
app.ticker.add(() => { if (simNodes.length) renderFrame(); });

// In buildSimulation — physics only, no rendering callback
simulation = d3.forceSimulation(simNodes)
  // ... forces ...
  .alphaDecay(ALPHA_DECAY);
  // .on('tick', renderFrame)  ← removed
```

Pixi's ticker is initialized at app creation (before `loadFromGist` resolves) and fires reliably for any visible browser tab. D3 handles physics (updates `simNodes` positions). Pixi reads `simNodes` and renders. They share state but run independently.

---

### Problem 8 — Icon picker showing only ~40 hardcoded emoji

**Symptom:** The icon picker in the add/edit form had a tiny static list of emoji. Not useful for custom skills with specific icons.

**Fix:** Replaced the static list with a dynamic Iconify search. The `buildEmojiGrid()` function now renders a default set of emoji when the search field is empty, and `searchIcons(query)` fires when the user types, replacing the grid with results from the Iconify public API.

The grid tab label was renamed from "emoji" to "icons" to reflect the expanded scope.

---

### Problem 9 — D3 force simulation replaced with custom physics engine

**Symptom / motivation:** D3's `forceCollide` is probabilistic — orbs visually overlapped, especially dashed-outline pending orbs. The layout had no sense of gravity or physicality. Resizing caused orbs to scatter unpredictably.

**Root cause:** D3 is a graph/network layout library, not a physics engine. It has no concept of gravity, mass, or velocity. Its collision avoidance iteratively relaxes constraints toward non-overlap but never guarantees it.

**Fix:** Replaced `d3.forceSimulation` and the D3 CDN script entirely with a custom 2D physics engine:

- **Gravity + mass**: `vy += GRAVITY * mass` per frame. Achieved orbs (`mass=2.0`) fall faster and settle lower. Pending orbs (`mass=1.0`) land on top.
- **Phased spawning**: Achieved orbs drop first (frame 0). Pending orbs are queued and released after 100 frames, by which time achieved orbs have settled at the bottom.
- **Position-based collision solver**: 12 iterations per frame, pushing overlapping pairs apart proportional to inverse mass.
- **Impulse transfer**: velocity exchanged along the contact normal gives the "sliding-past" liquid feel.
- **DAMPING = 0.92**: velocities decay over ~12 frames, so nudges propagate through the pile and die out naturally (marble-in-a-jar feel, not random noise).
- **RESTITUTION = 0.04**: near-zero bounce — orbs settle, not bounce.

The `CANVAS_FILL_RATIO` was reduced from `0.88` to `0.62` because random circle packing has a physical maximum of ~64% area coverage — at 88% the solver could never fully resolve overlaps within real-time iteration counts.

---

## Known limitations and future improvements

### Space coverage

After packing, there's often empty space near canvas edges and corners. A future pass could detect large empty regions and slightly scale up orbs (respecting tier ratios) to fill them better.

### Mobile UX

The header wraps on small screens and becomes hard to navigate. The `+ add` and `make your own` buttons could collapse into a hamburger menu on narrow viewports.

### Token visible in deployed source

The GitHub PAT is visible in the deployed HTML source — anyone who views source can find it. This is unavoidable for a static site without a backend. The blast radius is limited to your Gist data since the token only has `gist` scope. Rotate it from GitHub Settings any time if needed.

### Image uploads stored as base64

Large images will inflate the Gist file significantly. Prefer external image URLs or Iconify SVG URLs for large images; base64 upload is best for small icons and SVGs.

### CORS proxy dependency

Color extraction for external image URLs falls back to wsrv.nl if the image server doesn't serve CORS headers. wsrv.nl is an image-specific proxy that handles binary content and adds `Access-Control-Allow-Origin: *`. Note: corsproxy.io was tried first but blocks binary/image content on its free plan (returns 403). If wsrv.nl is unavailable, extraction falls back to the emoji glyph colors instead.

---

## Tech used

| Thing | Version | What it does |
|---|---|---|
| Vanilla HTML/CSS/JS | — | Everything — no framework |
| Roboto | Google Fonts | Display font for names and UI |
| Roboto Mono | Google Fonts | Monospace font for labels and admin UI |
| Pixi.js | 7.3.2 | WebGL rendering for circles, glows, and dots at 60fps |
| Custom physics engine | — | Gravity, mass, position-based collision solver, velocity damping |
| Canvas API | native | Offscreen 48×48 pixel sampling for color extraction |
| CSS custom properties | native | `--r` drives all responsive orb sizing via `calc()` |
| ResizeObserver API | native | Responsive relayout on window resize |
| requestAnimationFrame | native | 60fps render loop via Pixi's ticker |
| localStorage API | native | Visitor token + username persistence (with private-browsing fallback) |
| URLSearchParams API | native | Multi-user routing via `?id=GIST_ID` |
| Iconify API | public | 200k+ open-source icons; no key required |
| wsrv.nl | public | Image proxy fallback for external image color extraction (supports binary, adds CORS headers) |
| GitHub Gist API | v3 | Cross-device JSON persistence |
| GitHub Pages | — | Static hosting |
| GitHub Actions | — | CI/CD pipeline for token injection and deployment |
| GitHub Secrets | — | Encrypted storage for the Gist PAT |
| Roboto | Google Fonts | Display font for names and UI (weights 400–800) |
| Roboto Mono | Google Fonts | Monospace font for labels and admin UI (weight 400) |

---

## Customizing your skills

Skills are defined in `DEFAULT_SKILLS` inside the script. These seed your Gist on first load if `pocketful.json` doesn't exist yet. Each object:

```js
{
  id: 1,
  name: "Your skill name",      // shown inside the orb (keep it short)
  desc: "Longer description",   // shown in the detail popup
  glyph: "⚛️",                  // emoji (used if img is null)
  img: null,                    // base64 data URL, external URL, or Iconify SVG URL
  cat: "Quantum",               // category (legacy, not shown on canvas)
  achieved: true,               // true = achieved, false = in progress
  xp: 120,                      // tier * 80, used for XP bar
  tier: 2                       // 1–5, controls orb size
}
```

Tier reference:

| Tier | Scale | Relative size |
|---|---|---|
| 1 | 0.48 | Tiny |
| 2 | 0.78 | Small |
| 3 | 1.15 | Medium |
| 4 | 1.68 | Large |
| 5 | 2.35 | XL |

## Changing the admin password

```js
const ADMIN_PASSWORD = 'pocketful2025';  // change this
```

The password is stored in plain text in the HTML source. It is visible to anyone who views source on the deployed site. Its only purpose is to prevent accidental edits — not to protect against a determined adversary.
