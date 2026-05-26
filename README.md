# Tipogravity

A physics playground for typography lovers.

Random words from the world of type design — technical terms and surnames of legendary type designers — appear letter by letter, then collapse under gravity. Drag the letters around, watch them rearrange as gravity flips every 8 seconds. Click `?` to learn about the current word.

**Live:** [typogravity.vercel.app](https://typogravity.vercel.app)
**Author:** [Sandro Stefanelli](https://sandrostefanelli.com)

---

## Stack

Single-file vanilla project — no build step, no bundler, no dependencies to install.

- `index.html` — markup, styles and script all in one file
- [Matter.js](https://brm.io/matter-js/) — 2D rigid-body physics engine (loaded via CDN)
- [Geist](https://vercel.com/font) — font (loaded via Google Fonts)

To run locally, just open the file in a browser. That's it.

```bash
open index.html
# or simply double-click the file
```

---

## Project structure

```
.
├── index.html       # the whole app
└── README.md        # this file
```

Currently the file is named `sandro-graphic-physics.html` — rename it to `index.html` before deploying, so the server serves it as the root page.

---

## How it works

The app is structured in four logical layers, all inside `index.html`:

### 1. UI layer (CSS + HTML)
- Fixed-position controls in top-left (title) and top-right (Reset + Info buttons)
- Bottom-center: physics preset pills (default / heavy / floaty / snappy) and grip slider (0.1 / 0.2 / 0.35)
- A full-screen credits overlay opened by the `?` button — Vignelli-inspired typographic layout, two columns in landscape, stacked in portrait

### 2. Physics engine (Matter.js)
- An `Engine` with `positionIterations: 10`, `velocityIterations: 8` for stable collisions
- Four invisible walls (left, right, floor, and conditional ceiling)
- The ceiling is only added **after** the word has fully collapsed — this way letters can enter from above, but once everything is settled, gravity can flip without letting them escape

### 3. Word lifecycle
Every reset cycle goes through these states:

```
pickWord() → layoutWord() → typewriter compose → pause →
collapseWord() → close ceiling → start gravity loop
```

- **`pickWord()`** — picks a random word (7–13 letters) from `WORD_POOL`
- **`layoutWord()`** — calculates each letter's final position with wrap-to-newline if the word doesn't fit
- **Typewriter** — letters appear one by one every 180ms as **static** bodies (they don't fall yet)
- **`collapseWord()`** — turns all bodies dynamic at once with a small random impulse → they fall
- **Gravity loop** — every 8 seconds gravity flips between `y: 1` (down) and `y: -1` (up)

### 4. Drag system with force capping
Matter's `MouseConstraint` is a spring: the more you pull beyond what the letter can follow, the more force accumulates → letters jitter and shoot out.

The fix: **before every physics tick**, we measure the distance between the cursor and the letter's anchor point. If it exceeds `fontSize × gripFactor`, we virtually move the cursor closer — the spring never tenses beyond that threshold. The drag stays attached but no force accumulates.

```js
Events.on(engine, "beforeUpdate", () => {
  // if cursor-to-letter distance > threshold → clamp it
});
```

This is what makes the drag feel "soft" — the letter resists when blocked, instead of trembling and flying off.

---

## Tweaking the physics

All physics behavior lives in the **`PRESETS`** object near the top of the script:

```js
const PRESETS = {
  floaty: {
    density: 0.0008,         // mass — lower = lighter
    friction: 0.3,           // surface friction
    frictionAir: 0.07,       // viscosity of the "air"
    restitution: 0.4,        // bounciness
    mouseStiffness: 0.08,    // how rigid the drag spring is
    mouseDamping: 0.4,       // dampens drag oscillations — key for smoothness
    timeScale: 0.7,          // global simulation speed (1 = normal)
    collapseImpulse: 0.6,    // randomness of the collapse
  },
  // ...
};
```

Add your own preset by adding a new entry to `PRESETS` and a button to the `.presets` div. The default preset is set by:

```js
let currentPreset = "floaty";
```

And the drag's force cap:

```js
let gripFactor = 0.2;  // lower = softer grip, less force accumulated
```

---

## Tweaking the words and dictionary

Two arrays/objects to edit, near the top of the script:

- **`WORD_POOL`** — the list of words that can appear. Add or remove freely. Length is filtered to 7–13 letters at runtime
- **`DICTIONARY`** — keyed by uppercase word, each entry has `{ type, def }`. The `type` line is shown in italics, the `def` line is the description

Example:
```js
DICTIONARY.FRUTIGER = {
  type: "Adrian Frutiger · 1928–2015 · Switzerland",
  def: "Designer of Univers, Frutiger, and Avenir..."
};
```

If a word is in `WORD_POOL` but not in `DICTIONARY`, the info panel will show "No definition available."

---

## Tweaking visual style

- **Background color** — search `#0a0a0a` and replace with your color. It appears in the body, the canvas, and the credits overlay
- **Letter colors** — the `COLORS` array near the top: a list of hex colors used cyclically for each letter
- **Font size** — search `baseSize * 0.45` and change the multiplier (40%–50% of the shorter screen edge is the sweet spot)
- **Spacing between letters / lines** — variables `gap` and `lineHeightFactor` inside `layoutWord()`
- **Typewriter speed** — the `180` in `setTimeout(typeNext, 180)` inside `startAutoDrop()`
- **Gravity flip interval** — the `8000` in `startGravityLoop()`

---

## Deploy to Vercel

Three options, from simplest to most automated.

### Option 1 — Drag and drop (no Git required)

1. Rename the file to `index.html`
2. Go to [vercel.com/new](https://vercel.com/new)
3. Drag the file (or the folder containing it) into the page
4. Vercel deploys it and gives you a URL like `tipogravity-xyz.vercel.app`
5. (Optional) In project settings → Domains, you can add a custom domain like `tipogravity.sandrostefanelli.com`

### Option 2 — Vercel CLI

```bash
npm i -g vercel
cd /path/to/tipogravity
vercel
```

Follow the prompts. Subsequent deploys are just `vercel --prod`.

### Option 3 — GitHub integration (for auto-deploys)

1. Create a GitHub repo, push the file
2. On Vercel: New Project → Import Git Repository
3. Every push to `main` will trigger a new deploy automatically

---

## Browser support

Tested on:
- Chrome / Edge / Brave (desktop and mobile)
- Safari (macOS and iOS)
- Firefox

Touch dragging works on iOS and Android. The interface uses `100dvh` and `env(safe-area-inset-*)` for proper rendering on devices with notches.

---

## Known compromises

- **Collision boxes are rectangular**, not glyph-shaped. With dense letters like `T` or `L`, you may notice a slight visual overlap during collisions — the physics body is smaller than the visible glyph. Glyph-shaped collisions would require building polygonal hulls per letter and would be much heavier
- **Onboarding hints** only appear once, after the first word collapses. There's no persistent memory across page reloads
- **No sound** — by design, but could be added with Web Audio API on collisions

---

## Credits

Built by [Sandro Stefanelli](https://sandrostefanelli.com) with [Claude](https://claude.ai) (Anthropic).

Type designer biographies and term definitions written by hand. Geist is a free font by Vercel. Matter.js is by Liam Brummitt, MIT licensed.

---

## License

MIT — do whatever you want, just keep the credits.
