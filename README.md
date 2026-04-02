# Pocketful of Somethings

*collect anything. carry everything.*

A personal canvas for whatever you're collecting in life — skills, goals, achievements, books, places, anything. Each something is an orb on a dark canvas, gathered as you go, in any order, at any time. No sequence. No hierarchy. Just yours.

Built as a single `index.html` file. No framework, no build step, no backend. Hosted on GitHub Pages. Data persisted cross-device via a GitHub Gist.

---

## What it looks like

- A dark canvas filled with circles of varying sizes — each one a skill or goal
- **Achieved orbs**: fully colored background (derived from the icon), glowing ring and status dot in the icon's most vibrant color
- **In-progress orbs**: transparent with a dashed outline
- Status dots sit on the circumference of each orb, angled dynamically based on the orb's position on canvas
- Circles pack tightly using a greedy neighbor-hugging algorithm that rescales and repacks on every window resize
- XP bar and achieved/in-progress legend in the header
- Click any orb to see its full detail in a popup

---

## Features

- Circle packing that fills the canvas and rescales responsively
- Dominant color extraction from each emoji/image (canvas pixel sampling) for background fill and glow
- Admin mode with password protection (secret triple-click trigger)
- Add, edit, delete orbs — admin only
- Toggle achieved/in-progress status — admin only
- Icon support: emoji picker, file upload, or image URL
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
└── CLAUDE.md             ← AI project brief for Claude Code
```

---

## Step-by-step: how to recreate this from scratch

### Step 1 — Concept and canvas

The starting point was a vision board / collectible card aesthetic — skills and goals scattered like planets, no grid, no sequence. Anything can be achieved at any time.

**Design decisions made:**
- Dark space background (`#07070f`)
- Circles only, no square cards
- Icons are the hero — emoji or custom image
- Text (name) lives inside the circle
- Two states: achieved (colored, glowing) and in-progress (dashed outline, transparent)
- No categories visible on canvas — just the icon and name

---

### Step 2 — Circle packing algorithm

The layout uses a **stochastic greedy packer**. It is not mathematically optimal (that would require simulated annealing or Apollonian filling) but produces natural-looking results.

**How it works:**

1. Sort circles largest-first — big circles are hardest to place, so they go first
2. For the first circle, place it near the canvas centre
3. For each subsequent circle:
   - Pick a random already-placed circle as a reference
   - Pick a random angle
   - Place the new circle at exactly `ref.radius + new.radius + gap` along that angle — this guarantees the two are touching with just the gap between them
   - If valid (no collision, in bounds), accept it
   - Repeat up to 10,000 times with increasing slack if needed
   - Fall back to purely random placement if all attempts fail
4. Collision check: `distance(A, B) < rA + rB + gap(rA, rB)`

**Gap function** — not a fixed value but computed per pair:

```js
function computeGap(rA, rB) {
  const protA = dotRadius(rA); // dot protrusion from orb A
  const protB = dotRadius(rB); // dot protrusion from orb B
  return Math.max(5, protA + protB + 3);
}
```

This ensures status dots never overlap adjacent orbs.

**Radii computation** — each skill has a significance tier (1–5). Radii are computed from canvas area so they always fill the space:

```js
function computeRadii(W, H) {
  const area = W * H;
  const totalSq = SKILLS.reduce((a, s) => a + TIER_SCALE[s.tier] ** 2, 0);
  const unit = Math.sqrt(area * 0.88 / (totalSq * Math.PI));
  return SKILLS.map(s => Math.max(20, Math.round(unit * TIER_SCALE[s.tier])));
}
```

Tier scales: `{ 1: 0.48, 2: 0.78, 3: 1.15, 4: 1.68, 5: 2.35 }`

**Responsiveness** — a `ResizeObserver` on the canvas fires on every size change, cancels any pending animation frame, and re-runs the full pack + render:

```js
const ro = new ResizeObserver(() => {
  cancelAnimationFrame(raf);
  raf = requestAnimationFrame(render);
});
ro.observe(canvas);
```

---

### Step 3 — Dominant color extraction

Each orb derives two colors from its icon (emoji or image) by drawing it onto an offscreen canvas and sampling pixels:

**Background fill** — darkened average of all non-transparent pixels:

```js
const darken = 0.35;
const bgHex = '#' + [avgR * darken, avgG * darken, avgB * darken]
  .map(v => Math.round(Math.min(255, v)).toString(16).padStart(2, '0'))
  .join('');
```

**Glow color** — the most vibrant pixel, scored by `saturation × brightness`:

```js
const max = Math.max(r, g, b), min = Math.min(r, g, b);
const sat = max === 0 ? 0 : (max - min) / max;
const score = sat * (max / 255);
if (score > bestSat) { bestSat = score; glowR = r; glowG = g; glowB = b; }
```

Results are cached by skill ID so re-renders don't re-sample.

For emojis, the emoji is drawn via canvas `fillText` using a serif font. For images, an `<img>` is drawn onto the canvas (requires `crossOrigin = 'anonymous'` for external URLs).

---

### Step 4 — Status dots

Each orb has a small dot on its circumference indicating achieved (vibrant glow color) or in-progress (accent purple).

**Size** — proportional to orb radius, clamped:

```js
function dotRadius(r) {
  return Math.round(Math.min(10, Math.max(4, r * 0.12)));
}
```

**Position** — placed on the circumference at an angle that blends a seeded random angle (unique per skill) with a bias toward the canvas centre (so dots never get clipped at edges):

```js
function dotOffset(r, id, cx, cy, W, H) {
  const seeded = ((id * 2654435761) >>> 0) / 4294967296;
  const baseAngle = seeded * Math.PI * 2;
  const toCentreAngle = Math.atan2(H / 2 - cy, W / 2 - cx);
  const blend = 0.55;
  const ax = Math.cos(baseAngle) * (1 - blend) + Math.cos(toCentreAngle) * blend;
  const ay = Math.sin(baseAngle) * (1 - blend) + Math.sin(toCentreAngle) * blend;
  const angle = Math.atan2(ay, ax);
  const dr = dotRadius(r);
  const ex = r + Math.cos(angle) * r;
  const ey = r - Math.sin(angle) * r;
  return { top: Math.round(ey - dr), right: Math.round((2 * r) - ex - dr) };
}
```

The dot position updates on every render, so it adapts as orbs repack on resize.

---

### Step 5 — Admin mode (secret entry)

There is no visible admin button. Entry is through a **secret triple-click** on an invisible 40×40px zone in the bottom-left corner of the canvas:

```html
<div id="secret-trigger" style="position:absolute;bottom:0;left:0;width:40px;height:40px;z-index:50;cursor:default;"></div>
```

Three small dots briefly flash at the bottom-left to confirm each click. On the third click, a password prompt appears.

**Password check:**

```js
const ADMIN_PASSWORD = 'your-password-here';

function checkPw() {
  if (document.getElementById('pw-input').value === ADMIN_PASSWORD) {
    enterAdmin();
  } else {
    // show error
  }
}
```

**Admin mode unlocks:**
- `+ add drop` button in the header
- `admin` indicator and `exit ✕` button in the header
- Clicking any orb shows edit / toggle / remove actions at the bottom of the detail popup

**Exiting admin mode** — click the `exit ✕` button in the header, or refresh the page.

---

### Step 6 — Adding and editing skills

The add/edit form collects:
- **Name** (max 32 chars)
- **Description**
- **Significance** (tier 1–5, controls orb size)
- **Icon** — one of three modes:
  - Emoji picker (38 emoji options)
  - File upload (converts to base64 data URL, stored in the skill object)
  - Image URL (external link, loaded at render time)
- **Status** — Achieved or In progress

On save, the skill is added to (or updated in) `SKILLS`, the canvas re-renders and repacks, and the data is saved to the Gist.

---

### Step 7 — GitHub Gist for cross-device sync

All skill data is stored as a JSON array in a GitHub Gist. The app reads on load and writes on every change.

#### 7a — Create the Gist

1. Go to [gist.github.com](https://gist.github.com)
2. Create a **secret** gist
3. Filename: `pocketful.json`
4. Content: `[]`
5. Click **Create secret gist**
6. Copy the Gist ID from the URL: `https://gist.github.com/YOUR_USERNAME/GIST_ID_HERE`

#### 7b — Create a Personal Access Token

1. Go to github.com → **Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Name: anything (e.g. `pocketful`)
4. Expiration: your preference
5. Scopes: tick **only `gist`**
6. Click Generate — **copy the token immediately**, GitHub won't show it again

#### 7c — Wire into the HTML

At the top of the `<script>` block, set these three constants:

```js
const GIST_ID   = 'your-gist-id-here';
const GIST_FILE = 'pocketful.json';
const GH_TOKEN  = 'your-personal-access-token-here';
```

**Security note:** The token is visible in your HTML source. Since GitHub Pages repos are public, anyone who views your source can find it. The token only has `gist` scope — the worst someone can do is edit your skill data. Rotate the token any time from GitHub settings if needed.

#### 7d — How reading works

GitHub's API (`api.github.com`) blocks unauthenticated browser requests due to CORS. The fix is a two-step read:

1. Fetch the gist metadata with the token to get the `raw_url` of the file
2. Fetch that `raw_url` directly — it's a public CDN URL with no CORS restrictions

Fallback: if the API fetch fails, try `gist.githubusercontent.com/USERNAME/GIST_ID/raw/FILENAME` directly.

```js
async function loadFromGist() {
  const metaRes = await fetch(`https://api.github.com/gists/${GIST_ID}`, {
    headers: { 'Authorization': `token ${GH_TOKEN}` }
  });
  const gist = await metaRes.json();
  const rawUrl = gist.files[GIST_FILE].raw_url;
  const rawRes = await fetch(rawUrl); // no CORS block
  const data = JSON.parse(await rawRes.text());
  SKILLS = data;
}
```

If the gist file is empty (`[]`), the app seeds it with the default skills and saves back immediately.

#### 7e — How writing works

Every mutation (add, edit, delete, toggle) calls `saveToGist()`:

```js
async function saveToGist() {
  await fetch(`https://api.github.com/gists/${GIST_ID}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `token ${GH_TOKEN}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      files: {
        [GIST_FILE]: { content: JSON.stringify(SKILLS, null, 2) }
      }
    })
  });
}
```

A `saved ✓` indicator appears bottom-right on success. `sync failed` in red if the request fails.

---

### Step 8 — Deploy to GitHub Pages with GitHub Actions

Rather than committing your token directly into `index.html` (which GitHub will immediately detect and revoke), the recommended approach is to use a GitHub Actions pipeline. The pipeline injects the token at deploy time from a GitHub Secret — so your token never touches your repo, and you never need to manage it locally.

#### 8a — Store your token as a GitHub Secret

1. Go to your repo → **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS`
4. Value: your `ghp_...` Personal Access Token
5. Click **Add secret**

Your token is now encrypted and stored by GitHub. It is never visible in logs, never in your code, and only accessible to Actions workflows running on your repo.

#### 8b — Add the deployment workflow

Create this file at exactly `.github/workflows/deploy.yml` in your repo:

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

#### 8c — Configure GitHub Pages to use GitHub Actions

1. Go to your repo → **Settings → Pages**
2. Under **Source**, select **GitHub Actions** (not "Deploy from a branch")
3. Save

#### 8d — Push and deploy

Commit everything to `main` and push:

```bash
git add .
git commit -m "initial commit"
git push origin main
```

GitHub Actions will run automatically. You can watch it under the **Actions** tab. On success, your app will be live at `https://YOUR_USERNAME.github.io/YOUR_REPO_NAME`.

**How it works end to end:**
- Your committed `index.html` always contains `PASTE_YOUR_NEW_TOKEN_HERE` — safe to push, safe to view in source
- The Actions runner checks out your code, runs `sed` to replace the placeholder with the real token from GitHub Secrets
- The modified file is uploaded directly to GitHub Pages infrastructure — never committed back to `main`
- `main` stays clean forever

**Rotating your token:**
1. Generate a new PAT at github.com → Settings → Developer settings → Personal access tokens
2. Go to repo → Settings → Secrets and variables → Actions → `INDEXDOTHTML_OWNER_TOKEN_GITHUB_PERSONAL_ACCESS_TOKEN_GISTS` → Update
3. Push any commit to trigger a redeploy with the new token

**Note:** The deployed `index.html` on GitHub Pages contains the real token in plain text — anyone who views source on your live site can find it. This is unavoidable for a static site without a backend. The token only has `gist` scope, so the worst someone can do is edit your skill data.

No build step. No dependencies to install. No server. The fonts (Syne, DM Mono) load from Google Fonts via a CSS `@import`.

---

## Customizing your skills

The default skills are defined in `DEFAULT_SKILLS` inside the script. Edit them directly to change the initial data that gets seeded into your Gist on first load. Each skill object has this shape:

```js
{
  id: 1,                        // unique integer
  name: "Your skill name",      // shown inside the orb
  desc: "Longer description",   // shown in the detail popup
  glyph: "⚛️",                  // emoji (used if img is null)
  img: null,                    // base64 data URL or external image URL
  cat: "Quantum",               // category (legacy, not shown on canvas)
  achieved: true,               // true = achieved, false = in progress
  xp: 120,                      // used for XP bar calculation
  tier: 2                       // 1–5, controls orb size
}
```

---

## Changing the admin password

Find this line in the script and change the value:

```js
const ADMIN_PASSWORD = 'pocketful2025';
```

---

## Known limitations and future improvements

- **Packing algorithm** — the current greedy packer leaves some gaps, especially near edges. A future improvement would use force-directed relaxation or Apollonian gap filling for truly optimal coverage.
- **Token in source** — the GitHub PAT is visible in the HTML. Acceptable for personal use; not suitable for a shared/team context without a proper backend.
- **Image uploads** — uploaded images are stored as base64 in the Gist. Large images will make the Gist file very large and slow to load. Prefer image URLs for photos.
- **Emoji color extraction** — emoji rendering varies across operating systems, so the extracted dominant color may differ slightly between devices.

---

## Tech used

| Thing | What it does |
|---|---|
| Vanilla HTML/CSS/JS | Everything — no framework |
| Syne (Google Fonts) | Display font for names and UI |
| DM Mono (Google Fonts) | Monospace font for labels and admin UI |
| Canvas API | Offscreen pixel sampling for color extraction |
| ResizeObserver API | Responsive repacking on window resize |
| GitHub Gist API | Cross-device JSON persistence |
| GitHub Pages | Static hosting |
| GitHub Actions | CI/CD pipeline for token injection and deployment |
| GitHub Secrets | Encrypted storage for the Gist PAT |
| Pixi.js 7.3.2 | WebGL rendering for circles, glows, and dots |
| D3.js 7.8.5 | Force simulation for orb layout |
