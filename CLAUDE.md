# CLAUDE.md — Pocketful of Somethings

This document is the complete project brief for Claude Code. Read it fully before doing anything. It covers what the app is, every architectural decision made and why, the current state of the code, known issues, and what to work on next.

---

## What this is

**Pocketful of Somethings** is a personal canvas app where you collect skills, goals, and achievements as orbs on a dark space-like canvas. The name comes from the idea of carrying a pocketful of things you've gathered along a non-linear life journey — like a Pokédex for your life, but without sequence or order. Anything can be achieved at any time, in any order.

**Tagline:** *collect anything. carry everything.*

**Core design philosophy:**
- Non-linear — no sequence, no hierarchy, no progress bar that implies a finish line
- Collectible — each skill/goal is a drop you collect, like a loot drop or a Pokémon catch
- Vision board — the canvas is a collage, not a list or grid
- Personal — it's yours, intimate, carried with you
- Shareable — others can view your pocketful, and be inspired to make their own

**Who built it:** Ashnah Khalid Khan, Cloud Operations Lead at Astera Software, Fulbright applicant targeting Robotics & Mechatronics with a focus on Quantum Machine Learning applied to BCI systems. The default skills in the app are her actual achievements and goals.

---

## Tech stack — constraints are intentional

**Hosting:** GitHub Pages only. No server. Static files only.

**Storage:** GitHub Gist (JSON). One `pocketful.json` file per user.

**Frontend:** Single `index.html` file. No build step, no bundler, no framework, no npm.

**CDN libraries loaded at runtime:**
- Pixi.js 7.3.2 — WebGL rendering for circles, glows, dots
- Google Fonts — Syne (display) + DM Mono (monospace)

**Native browser APIs used:**
- Canvas API — offscreen 48×48 pixel sampling for color extraction
- CSS custom properties — `--r` drives all responsive orb sizing via `calc()`
- ResizeObserver API — responsive relayout on window resize
- requestAnimationFrame — 60fps render loop via Pixi's ticker
- localStorage API — visitor token + username persistence (with private-browsing fallback)
- URLSearchParams API — multi-user routing via `?id=GIST_ID`

**External services:**
- GitHub Gist API v3 — cross-device JSON persistence
- GitHub Pages — static hosting
- GitHub Actions — CI/CD token injection and deployment
- GitHub Secrets — encrypted storage for the Gist PAT
- Iconify API (public) — 200k+ open-source icons, no key required
- wsrv.nl (public) — image proxy fallback for CORS-blocked external images

**Standards and principles followed:**
- Position-based dynamics (PBD) — physics collision resolution technique
- XSS prevention — DOM `textContent`/`createElement` for user-sourced data, never `innerHTML`
- URL protocol validation — reject `javascript:` and non-http(s) at input boundary
- CORS proxy fallback pattern — try direct, then proxy, for external image color extraction
- Single responsibility per function — draw, extract, tick, render are separate concerns
- Named constants for all magic numbers (no inline literals)

**Why these constraints exist:** The owner consciously chose to stay within GitHub Pages + Gist for simplicity, portability, and zero infrastructure to manage. Do not suggest moving to Vercel, Supabase, or any backend service. The tech stack is a deliberate choice, not a limitation to work around.

---

## File structure

```
repo/
├── index.html    ← entire app, ~1162 lines
├── README.md     ← human-readable project documentation
└── CLAUDE.md     ← this file (AI project brief)
```

Everything lives in `index.html`. The script is organised into clearly labelled sections with comment banners. Maintain this organisation when editing.

---

## Architecture overview

### Rendering: hybrid Pixi + DOM

The canvas uses two layers stacked on top of each other:

```
z-index 3: .orb-hit divs       ← invisible, handle click/hover
z-index 2: #label-layer        ← DOM divs for emoji + text labels  
z-index 1: #pixi-canvas        ← WebGL canvas, pointer-events: none
```

**Pixi.js** (`orbContainer`, `glowContainer`) renders:
- Filled circles (achieved orbs, background color from icon extraction)
- Glow rings (achieved orbs, vibrant color from icon extraction)
  - Solid ring: `lineStyle(2.5, glowColor, 0.65)` at `r + 1.5`
  - Soft outer glow: 3 layers at `r+3`, `r+6`, `r+9` with decreasing opacity
- Dashed outline circles (pending orbs, drawn as arc segments)
- Status dots (on circumference, color-matched to glow/accent)

**DOM label layer** renders:
- `.orb-label` divs: emoji + name text, `pointer-events: none`
- `.orb-hit` divs: sized to match orb, `pointer-events: all`, handles clicks

**Why hybrid:** Pixi renders circles at 60fps with GPU acceleration and real glow effects. But emoji rendering in WebGL (via Pixi sprites) is unreliable cross-platform — different OS render emoji differently. DOM text stays crisp and consistent everywhere. Hit areas are DOM because Pixi hit detection is more complex than needed.

### Layout: custom 2D physics engine

D3 has been fully removed. The layout is driven by `physicsTick()`, called every frame by the Pixi ticker (60fps). Key functions:

- `initPhysics(w, h)` — builds `simNodes`, phases spawning (achieved first, pending queued)
- `spawnNode(s, r, w)` — creates a node above the canvas at random x
- `outerR(n)` — returns the visual boundary radius used for collision and boundary checks
- `physicsTick()` — advances one frame: gravity → boundary → collision solver → re-clamp

**Physics constants:**
- `GRAVITY = 0.30` — px/frame² downward acceleration
- `ACHIEVED_MASS = 2.0` — heavier, falls faster, settles lower
- `PENDING_MASS = 1.0` — lighter, arrives later, lands higher
- `DAMPING = 0.92` — velocity decay per frame (fluid, not frozen)
- `RESTITUTION = 0.04` — near-zero bounce
- `SOLVER_ITERS = 12` — collision resolution passes per frame
- `COLLISION_MARGIN = 2` — px clearance between outer visual edges
- `PENDING_SPAWN_DELAY = 100` — frames before pending orbs drop (achieved settle first)
- `CANVAS_FILL_RATIO = 0.62` — fraction of canvas area the orb set fills (physical random packing max ~64%)

**Phased spawning:** On first load, achieved orbs drop immediately. Pending orbs are held in `pendingQueue` and released after 100 frames (~1.7s at 60fps), ensuring achieved orbs settle at the bottom before pending orbs arrive and land on top. On resize or `refreshCanvas()`, existing nodes keep their positions (clamped to new bounds); only newly added orbs drop from above.

**Collision solver:** Position-based dynamics — for every overlapping pair, push them apart along the line between centres, proportional to inverse mass. Impulse transferred along the contact normal gives the "sliding-past" liquid feel.

**Visual outer edge values:**
- **Achieved orbs:** `r + 9` (glow extends to `r+9` via 3 soft layers)
- **Pending orbs:** `r + 1` (half the 1.5px dashed stroke width)

### Color extraction

`extractColors(src, isEmoji, id, cb)`:
1. Draws emoji/image onto a 48×48 offscreen canvas
2. Samples all non-transparent pixels
3. **Background fill:** darkened average (`darken = 0.36`) of all pixel colors
4. **Glow color:** pixel with highest `saturation × brightness` score (most vibrant)
5. Results cached in `colorCache` by skill ID

Color extraction is async. The callback fires after extraction and triggers a `simulation.alpha(0.1).restart()` nudge to repaint (no longer used — now just calls `repaintOrb` directly since physics runs continuously). Cache is invalidated on edit (`delete colorCache[s.id]`).

### Status dots

`dotRadius(r)` = `Math.min(10, Math.max(4, r * 0.12))` — scales with orb size.

`dotOffset(r, id, cx, cy, W, H)`:
- Base angle: seeded from `id` for stable unique position per orb
- Bias angle: toward canvas centre (so dot never gets clipped at edges)
- Blended 45% seeded / 55% toward-centre
- Dot placed exactly on circumference: `cx + cos(angle) * r`, `cy + sin(angle) * r`
- Updates on every render tick — responsive to position changes

---

## Data model

Each skill/goal is an object:

```js
{
  id: 1,                        // unique integer, auto-incremented
  name: 'QWorld cert',          // short name shown inside orb (max 32 chars)
  desc: 'Completed quantum...',  // longer description shown in detail popup
  glyph: '⚛️',                  // emoji (used when img is null)
  img: null,                    // base64 data URL or external image URL, or null
  cat: 'Quantum',               // category (legacy, not shown on canvas)
  achieved: true,               // true = achieved, false = in progress
  xp: 120,                      // XP value (tier * 80), used for XP bar
  tier: 2                       // 1–5, controls orb size via TIER_SCALE
}
```

`TIER_SCALE = {1:0.48, 2:0.78, 3:1.15, 4:1.68, 5:2.35}` — multipliers for radius computation.

Radii computed from canvas area so orbs always fill the space proportionally, regardless of orb count:
```js
function computeRadius(s, W, H) {
  const area = W * H;
  const totalSq = SKILLS.reduce((a,x) => a + TIER_SCALE[x.tier]**2, 0);
  const unit = Math.sqrt(area * CANVAS_FILL_RATIO / (totalSq * Math.PI));
  return Math.max(22, Math.round(unit * TIER_SCALE[s.tier]));
}
```

`CANVAS_FILL_RATIO = 0.62` — set below the physical random circle packing maximum (~64%) to allow the physics engine to resolve all collisions without persistent overlap.

---

## Multi-user / URL system

The app supports multiple users without any backend:

**Owner view** (no URL params): loads `OWNER_GIST_ID` with `OWNER_TOKEN`

**Visitor view** (`?id=GIST_ID`): loads that Gist using token from `localStorage.getItem('pocketful_token_GIST_ID')`

Gist reading: two-step to avoid CORS — fetch authenticated API to get `raw_url`, then fetch `raw_url` directly (public CDN, no CORS block). Fallback: `gist.githubusercontent.com/USERNAME/GIST_ID/raw/pocketful.json`.

Gist writing: authenticated PATCH to `api.github.com/gists/GIST_ID`. Only happens in admin mode.

**Onboarding flow** (`openOnboarding()`): 3-step modal — enter Gist ID, enter token, enter username. On finish, validates token, creates `pocketful.json` if missing, stores token in localStorage, redirects to `?id=GIST_ID`.

---

## Admin mode

**Entry:** Triple-click the invisible 44×44px zone at bottom-left of canvas. Small dots flash to confirm each click. On third click, password prompt appears.

**Password:** `pocketful2025` (change `ADMIN_PASSWORD` constant)

**What admin mode unlocks:**
- `+ add` button in header
- `admin` indicator + `exit ✕` button in header
- Edit / toggle achieved / remove buttons at bottom of detail popup

**Exit:** Click `exit ✕` in header.

**Admin mode is purely UI-gated.** There is no server-side auth. The token in `OWNER_TOKEN` is what actually controls write access to the Gist. A visitor using `?id=GIST_ID` can enter admin mode with the same password but can only write to their own Gist (their token from localStorage).

---

## Key config constants (top of script)

```js
const ADMIN_PASSWORD = 'pocketful2025';
const OWNER_GIST_ID  = 'e019b7babf70f4dd72c07d19e2996fcf';
const OWNER_TOKEN    = 'PASTE_YOUR_NEW_TOKEN_HERE';  // ← owner must fill this in
const GIST_FILE      = 'pocketful.json';
const OWNER_USERNAME = 'AshnahKhalidKhan';
```

`OWNER_TOKEN` must be a GitHub Personal Access Token with only `gist` scope. The owner was told to revoke any previously shared token and generate a fresh one. This value is intentionally left as a placeholder in the repo — the owner pastes their real token locally before deploying.

---

## Visual design system

**Colors (CSS variables):**
```
--bg:      #07070f   (near-black background)
--surface: #0f0f1a   (modal backgrounds)
--border:  rgba(255,255,255,0.08)
--text:    rgba(255,255,255,0.85)
--muted:   rgba(255,255,255,0.35)
--faint:   rgba(255,255,255,0.1)
--done:    #1D9E75   (achieved green)
--accent:  #7F77DD   (purple accent)
```

**Fonts:**
- `Syne` — display/UI font (weights 400, 500, 600, 700, 800)
- `DM Mono` — monospace for labels, admin UI, XP display (weights 300, 400, 500)

**Orb visual states:**
- **Achieved:** filled circle (darkened dominant color from icon), solid glow ring + soft outer glow (vibrant color from icon), filled status dot with glow
- **Pending:** transparent background, dashed white outline (`rgba(255,255,255,0.22)`), purple status dot, no fill

**Do not** add gradients, drop shadows, blur effects, or heavy animations. The aesthetic is flat, dark, minimal — the glow effects are the only exception and they're subtle.

---

## Script sections (maintain this organisation)

```
CONFIG                    — constants
MODE DETECTION            — URL params, localStorage, visitor/owner
STATE                     — let variables (incl. physicsFrame, pendingQueue)
PIXI.JS SETUP             — app, containers, ticker
COLOR EXTRACTION          — extractColors(), hexN(), hexToNum()
DOT HELPERS               — dotRadius(), dotOffset()
SHARED HELPERS            — calcXP, getIconSrc, findSkill, buildPillsHTML
PHYSICS ENGINE            — computeRadius(), outerR(), spawnNode(),
                            initPhysics(), physicsTick()
PIXI RENDER FRAME         — renderFrame(), createPixiOrb(), updatePixiOrb(),
                            createDomLabel(), updateDomLabel(), destroyOrb()
INIT / RESIZE             — initCanvas(), ResizeObserver
XP                        — updateXP()
GIST SYNC                 — showSyncStatus(), loadFromGist(), saveToGist(),
                            refreshCanvas(), applySkillsData()
SECRET KNOCK              — triple-click zone, checkPw(), exitAdmin()
DETAIL MODAL              — openDetail(), adminEditFromDetail(), etc.
ADD / EDIT                — resetForm(), openAddModal(), openEditModal(),
                            saveSkill()
DELETE                    — openDeleteConfirm(), confirmDelete()
TOGGLE                    — toggleAchieved()
ICON FORM HELPERS         — switchIconTab(), buildEmojiGrid(), pickEmoji(),
                            handleFileUpload(), handleUrlInput(), selectStatus()
ONBOARDING                — obStep vars, OB_STEPS, openOnboarding(),
                            renderObStep(), obBack(), obNext(), obFinish()
MODAL UTILS               — openModal(), closeModal(), overlay click handlers
BOOT                      — buildEmojiGrid(), loadFromGist().then(initCanvas)
```

---

## What has been tried and reverted

Keep this section in mind to avoid repeating failed approaches:

| Attempt | What happened | Why reverted |
|---|---|---|
| Stochastic greedy packer | Original layout algorithm, good enough visually | Replaced by D3 for animation |
| Fixed GAP constant for collision | Too rigid, didn't account for different orb sizes | Replaced by `computeGap(rA, rB)` function |
| D3 collision with high iterations + slow decay | Still probabilistic, orbs still overlapped | Reverted after failing to eliminate overlaps |
| setInterval-based constraint solver | Introduced rendering artifacts, `simulation.stop()` calls broke | Fully reverted, D3 restored |
| D3 entirely (force simulation) | Could not support gravity, mass, or hard collision guarantee | Replaced with custom physics engine |
| CANVAS_FILL_RATIO = 0.88 with physics solver | Random circle packing max ~64% — solver could not resolve overlaps in real-time | Reduced to 0.62 |
| Spawn pending orbs with y-offset above canvas | Volume of 16 pending orbs still filled the bottom regardless of spawn height | Replaced with phased spawning (pendingQueue, PENDING_SPAWN_DELAY) |
| .on('tick', renderFrame) on D3 simulation | D3 timer uses rAF which doesn't fire for inactive/background iframes | Replaced with app.ticker.add() (Pixi ticker, always fires for visible tabs) |
| `overflow: hidden` on `.drop` | Clipped status dots out of existence | Removed |
| Status dots as fixed `top: 7px; right: 7px` | Not responsive to orb size or position | Replaced with `dotOffset()` math |
| Category field in add form | Removed — not shown on canvas, added visual noise | Gone |

---

## What to work on next (priority order)

### 1. Space coverage

With `CANVAS_FILL_RATIO = 0.62`, orbs fill about 62% of canvas area. There is often empty space near the top and corners after settling. Options:
- Detect large empty regions and slightly scale up orbs (respecting tier ratios)
- Add a weak upward force to "float" the pile and spread it more vertically
- Tune `CANVAS_FILL_RATIO` upward carefully (keep below 0.64 to avoid persistent solver overlap)

### 2. Mobile UX

Header wraps on small screens and is hard to navigate. The `+ add` button and `make your own` button could collapse into a menu on narrow viewports.

### 3. Token security note

The `OWNER_TOKEN` in the HTML is visible to anyone who views source. This is acceptable for the owner's personal use case but worth noting as a limitation. One improvement: hash-compare the token on the client side rather than storing it in plain text — though this only marginally improves security for a static site.

---

## What NOT to change

- The hosting approach (GitHub Pages + Gist only)
- The single-file architecture (`index.html`)
- The font choices (Syne + DM Mono)
- The dark space aesthetic
- The admin triple-click entry mechanism
- The non-linear, no-sequence philosophy of the app
- The multi-user URL system (`?id=GIST_ID`)
- The name: **Pocketful of Somethings**

---

## Running locally

No build step needed. Just open `index.html` in a browser. For Gist sync to work, you need a valid `OWNER_TOKEN` set. Without it, the app falls back to `DEFAULT_SKILLS` and shows "offline — changes won't sync".

For local development with live reload:
```bash
npx serve .
# or
python3 -m http.server 8080
```

---

## Owner context (useful for personalising suggestions)

Ashnah is a software engineer and Cloud Operations Lead at Astera Software with a Computer Science background. She is actively applying for a Fulbright Master's scholarship targeting Robotics and Mechatronics Engineering with a focus on Quantum Machine Learning applied to brain-computer interface systems. She has QWorld quantum computing certifications and experience judging/mentoring at the World Robot Olympiad Pakistan, and participated in the NYUAD International Hackathon for Social Good.

The default 14 skills in `DEFAULT_SKILLS` are her real achievements and goals — do not change them without being asked.

---

## Deployment pipeline

The app deploys via GitHub Actions on every push to `main`. The workflow lives at `.github/workflows/deploy.yml`.

**How the token injection works:**
- `index.html` is committed with `OWNER_TOKEN = 'PASTE_YOUR_NEW_TOKEN_HERE'` — the placeholder is safe to commit
- The Actions workflow runs `sed` to replace the placeholder with the real token from GitHub Secrets
- The injected file is uploaded directly to GitHub Pages infrastructure — never committed back to `main`
- `main` branch never contains the real token at any point

**GitHub secrets required:**
- `INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS` — GitHub Personal Access Token with only `gist` scope

To add/rotate the token:
1. Go to repo → Settings → Secrets and variables → Actions → `INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS`
2. Update the value with the new token
3. Push any commit to `main` to trigger a redeploy with the new token

**GitHub Pages settings required:**
- Repo → Settings → Pages → Source: **GitHub Actions** (not "Deploy from a branch")

**File structure with workflow:**
```
repo/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── index.html         ← always has placeholder token, safe to commit
├── README.md
└── CLAUDE.md
```

**Important:** The deployed `index.html` on GitHub Pages contains the real token in plain text — anyone who views source can find it. This is unavoidable for a static site without a backend. The token only has `gist` scope so the blast radius is limited to your Gist data. Rotate it from GitHub settings any time if needed.
