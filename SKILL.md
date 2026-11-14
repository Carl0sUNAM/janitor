---
name: "janitorai-bio-html"
description: "Build rich JanitorAI bot bios (descriptions) in inline-styled raw HTML — scene selectors, phone/chat mockups, page-turn books, audio buttons, textures, animations. Use when the user wants a bot bio beyond what the rich-text editor allows, mentions \"the html method\", unrestricted bio, details/summary layouts, or scene selection in a bot bio. Companion to css-janitorai (which covers profile CSS)."
---

# JanitorAI Unrestricted Bot Bio (inline HTML)

Bot **bios** are a different system from **profile CSS** (`.pp-cc-*`, covered by `css-janitorai`).
This skill covers designing bios that use raw HTML + inline styles.

## 1. How the platform behaves (verified)

**The editor strips everything.** The create/edit Bio box is a TipTap editor whose schema only has:
- Nodes: `paragraph, heading, blockquote, bulletList, orderedList, listItem, codeBlock, horizontalRule, hardBreak, image, youtube`
- Marks: `bold, italic, underline, strike, code, link, highlight, textStyle` (+ font-size, color, text-align)

Anything typed, pasted, put through the **"Source code"** dialog, or imported from a Tavern PNG is normalized
to `<p>`/`<span>`. `div`, `details`, `summary`, `audio`, `background`, flex, `aspect-ratio` are all lost.

**The bio renderer is more permissive.** Bios saved with raw HTML (sent to the API directly — "the html method",
see *JAI Unrestricted Bio Guide by DelightfulTempe*: https://rentry.co/jai-bio-guide-by-delightfultempe)
render with the features below. Audits: 2026-10-30 — 7 raw-HTML bios from one creator + 2 of the user's own test uploads (Flandre bio, animation test); 2026-11-01 — "Yandere Superhero Is After You | Kaisa" (35 details, ~1500 styled elements); 2026-11-02 — "No im not horny" (16 details, ~1000 styled elements, borrowed site classes + vw breakpoints). 2026-11-11 — "Your Goth Girlfriend Streamer | Madison Thomas" (4 details, ~190 styled elements, 2 `<marquee>`); 2026-11-11 — "She moved on too fast. | Iris" (36 details, ~990 elements, full iPhone mockup); 2026-11-13 — "Under Your Skin 🤍 Lethe" (40 details, label/checkbox skip buttons, `sonner-fade-in` first-open reveals on every card body). All checked in the rendered DOM.

**Renderer behavior:** the inline `style` is parsed and re-serialized (React style object). Duplicate declarations
collapse (last one wins); shorthands come back as longhands (e.g. `animation:jump 1.2s ease-in-out infinite` → `animation: 1.2s ease-in-out 0s infinite normal none running jump`); `calc()` inside `clamp()/min()/max()` is folded into scientific notation (`-7.58992e+07px + 9.9999e+06vw`). Images get wrapped in a `span` and given
`class="chakra-image"`, `node`, `draggable`; your `style` is kept.
**Exception seen 2026-11-11:** on the Iris bio the `style` attributes came back **exactly as written** (`display:block;margin:0`, `var()` and `calc()` untouched), not re-serialized. Either the renderer changed or that bio was saved another way. Don't depend on either behavior; when auditing, read `getComputedStyle` to confirm what applies.

### Elements
| Works | Notes |
|---|---|
| `div`, `span`, `p`, `b`, `strong`, `i`, `em`, `s`, `br`, `hr`, `blockquote`, `h1`, `h2`, `h3`, `section`, `article`, `figure`, `figcaption`, `center` | structure (semantic tags verified 2026-11-01) |
| `a href` | external with `target="_blank" rel="noopener noreferrer"`; internal `janitorai.com/characters/...` links work too (cross-promo "THEIR BOT →"). Can wrap **block content** or **only an image** → clickable cards / image buttons. Linking to the full-size `ella.janitorai.com` image works as "open full art" |
| `details` / `summary` | all interactivity; nestable (verified, 3 levels deep in the Flandre bio); up to 35 per bio seen. **`name="…"` works** (exclusive accordion: opening one closes the others in the group) and **`open` is kept** (start a panel open) — both verified 2026-11-01 |
| `audio controls` | `preload="none"` and **`loop`** kept (verified 2026-11-02); `style="width:100%"` full-width player |
| `marquee` | **renders and scrolls** (verified 2026-11-11); `loop` and `width="20"` attributes kept. See §2 *Marquee* |
| `label for` + `input type="checkbox"` | **both survive** (verified 2026-11-13). `for` gets the same `user-content-` prefix as `id`, so they still match. Clicking the label focuses the input → the page scrolls to it. See §2 *Skip buttons* |
| `img` | `style="width:100%;display:block"`; `filter`, `mix-blend-mode`, negative margins, `object-fit`/`object-position` all kept; `loading="lazy"`, `decoding`, `alt` kept |
| attributes | `id`, `name` (both get a **`user-content-` prefix** — sanitizer clobber protection; fine for details groups since every name gets the same prefix; `#anchor` links untested, they'd probably need `#user-content-ID`; for jump-to-section use the verified label/checkbox trick, §2 *Skip buttons*), `role`, `aria-label`, `aria-hidden`, `tabindex` kept. **`title`** kept → native hover tooltip (seen on `summary`, verified 2026-11-02). **`class` is kept unprefixed** (verified 2026-11-02) — useless for your own names (no `<style>`), but lets you borrow the site's own classes, see §2 *Borrowed site classes* |

### Inline style properties confirmed
- **Layout:** `display` (block/inline-block/flex/**grid**/none), `grid-template-rows`, `justify-self`, `justify-items`, `align-content`, `row-gap`/`column-gap`, `isolation:isolate`, `flex` shorthand (`1 1 280px`, `0 0 78%`, `0 0 auto`), `flex-direction`, `flex-wrap`, **`order`** (e.g. `-1` to move an item first in a grid/flex), `gap`, `justify-content`, `align-items`, `align-self`, `width`, `height`, `max-width`, `min-width` (incl. `0`), `min-height`, **`max-height`**, `margin*` (incl. negative / % like `margin:-10% 0 -8% -14px`), `padding*`, `box-sizing`, `aspect-ratio`, `overflow:hidden`, `overflow-x`/**`overflow-y`** separately, `clear`, `vertical-align`, `white-space:nowrap`
- **Math functions:** `min()`, `max()`, `clamp()`, `calc()` with `%`, `px`, `vw`, `cqw`, nested deeply — kept (re-serialized). Enables viewport breakpoints, see §2 *Viewport breakpoint (clamp switch)*
- **Grid:** `display:grid;grid-template-columns:minmax(0,1fr)` + children with `grid-column:1/2;grid-row:1/2` → **layers stacked on top of each other** (substitute for `position:absolute`). `repeat(auto-fit,minmax(min(100%,240px),1fr))` works for responsive card grids.
- **Scrolling:** `overflow-x:auto`, `scroll-snap-type:x mandatory`, `scroll-snap-align:center`, `overscroll-behavior-x:contain`, `scrollbar-width:thin`, `scrollbar-color` → swipeable carousels
- **Vertical paging (verified 2026-11-11):** `overflow-y:auto;scroll-snap-type:y mandatory;overscroll-behavior-y:contain;scrollbar-width:none;touch-action:pan-y` on a fixed-height box, pages with `scroll-snap-align:start;scroll-snap-stop:always` (stops exactly one page per swipe, no skipping) → "swipe up to unlock" lock screen → home screen
- **Paint:** `background` shorthand + longhands (`-color`, `-image`, `-position`, `-size`, `-repeat`), `background-blend-mode`, `linear-` / `radial-` / `repeating-linear-gradient` (incl. sized ellipses like `radial-gradient(ellipse 42% 460px at 50% 0,…)`), gradients layered over `url()`, **`url('data:image/svg+xml,...')` inline SVG backgrounds** (URL-encoded — `#` MUST be `%23`)
- **Text-clip:** `background-clip:text` + `-webkit-background-clip:text` + `color:transparent` → gradient text (verified 2026-10-30)
- **Borders/shadows:** `border*`, `border-radius` (incl. irregular `3% 6% 2% 5% / 2% 3% 2% 4%` for torn-paper shapes), `outline` + **negative `outline-offset`** (double frame), `box-shadow` incl. multiple `inset` / negative spread, glow shadows
- **Effects:** `transform` (`rotate()`, `scale()`, `translateZ(0)`) + **`transform-origin`** (e.g. `0 100%` + `rotate(-45deg)` for hinged/angled bars, verified 2026-11-02), `filter` (`drop-shadow()`, `blur()`, `brightness()`, `contrast()`, `sepia()`, `saturate()`, `hue-rotate()`, `invert()`, long chains of 8 steps), `mix-blend-mode` (e.g. `lighten`/`multiply` on an img), `opacity`, `text-shadow`
- **Animation:** inline `animation` shorthand is kept (verified 2026-10-30) — but only with **@keyframes the site already loads** (see §2 *Animations*)
- **Clipping:** **`clip-path:polygon(…)` works** (verified 2026-11-01, e.g. `polygon(100% 0,100% 40%,0 100%)` for a diagonal corner) — replaces the SVG-strip trick for simple torn/cut shapes
- **Interaction:** **`pointer-events:none`** works → decorative overlay layers no longer block clicks on summaries below them
- **Container queries:** **`container-type:inline-size`** on a wrapper + `cqw` units in children (`font-size:clamp(10px,3.833cqw,23px)`, `padding:clamp(8px,calc(3cqw - 1px),19px)`) → sizes scale with the bio column, not the window (verified 2026-11-01)
- **CSS custom properties:** inline `--name:value` is kept (verified 2026-11-02) and **inherits to children, so your own inline `var()` reads it** (verified 2026-11-11): `--phone-height:clamp(560px,78svh,640px)` on a wrapper, then `height:var(--phone-height)`, `margin-top:calc(-1 * var(--phone-height) + 28px)`, `height:calc(var(--phone-height) - 69px)` deeper down. Lets one value drive many sizes. Also feeds `var()` in the site's own CSS to recolor a borrowed class (`--text-secondary`, `--text-primary`, `--colour-white-a10`, `--bg-dark`, `--border-color`)
- **Backdrop blur:** **`backdrop-filter:blur(18px)`** works (computed value verified 2026-11-11) → frosted-glass cards (`background:#202020bb;border:1px solid #ffffff30`) over a background image
- **Containment:** `contain:paint` kept (with `isolation:isolate` + `overflow:hidden` on a clipping frame, e.g. a phone bezel)
- **Color scheme:** `color-scheme:dark` kept → native controls (audio player, scrollbars) render dark
- **Mobile tap:** `-webkit-tap-highlight-color:transparent` kept (seen on summaries; removes the tap flash on phones, verified 2026-11-11)
- **Units:** `rem` works (font-size, padding, letter-spacing, margins) alongside px/%/vw/cqw. **`svh`** works too (small-viewport height — stable on phones when the browser bar hides; e.g. `clamp(560px,78svh,640px)`, verified 2026-11-11)
- **Outlined text:** `-webkit-text-stroke-width`/`-color`, `-webkit-text-fill-color:transparent`, `paint-order:stroke`
- **Text:** `overflow-wrap`, `word-break`, `font` shorthand, `font-family`, `font-size`, `font-style`, `font-weight`, `line-height`, `letter-spacing`, `text-align`, `text-transform`, `text-decoration`, `color`, `cursor`, `list-style`, **`text-overflow:ellipsis`** (with `overflow:hidden;white-space:nowrap`), **`text-wrap:balance`** (balanced headings) — last two verified 2026-11-02

### Page width (where the bio sits)
- Desktop: the bio lives in the **left column = 50% of the page** (`_box1Desktop`: `flex:0 0 50%`), minus card padding (~56px). Comments are on the right.
- A ~860px window (e.g. Chrome side panel open) gives only **~307px** for the bio; full-screen PCs give ~600–900px.
- So: outer wrapper `max-width:100%;margin:0 auto` — **never hard-cap at 640px** (it leaves dead space on big screens). Keep outer paddings small (≈12px/6px) and test at **307px and 700px**.
- No height clamp / "read more" on the bio container.

### Assets
- Images (`img` and `background:url()`): `ella.janitorai.com/media-approved/...` (every image seen uses this host).
- **Transparency survives upload**: transparent PNG → WebP on ella keeps the alpha channel (verified with 2 textures).
- Inline SVG via `data:image/svg+xml` in `background` (no upload needed for patterns/icons).
- Audio `src`: `files.catbox.moe` and `file.garden` both work.

### Fonts
Confirmed **loading** by name: `Inter`, `Poppins`, `Fraunces`, `Crimson Text`, `JetBrains Mono`, `Dancing Script`, `Rock Salt`, `Berkshire Swash`, `Jura`, **`VT323`** (pixel/terminal font, verified 2026-11-02).
Confirmed **NOT loading** (fall back silently): `Bebas Neue`, `Oswald` (2026-11-11), `Cinzel`, `Cinzel Decorative`, `Grenze Gotisch`, `Special Elite`, `Nunito`, `IM Fell English`, `Cormorant Garamond`.
Always give a fallback stack (Georgia / `Segoe Print` / `ui-monospace` / `Arial Black` etc. — system fonts are fine).
The guide lists 14 Google Fonts usable by name; custom `<link>` does not work.
Good horror/grimoire set: Fraunces (headings), Crimson Text (body), Dancing Script (character handwriting), Rock Salt (scratches), JetBrains Mono (records). Retro/terminal set: VT323 + JetBrains Mono.

### Not seen in any working bio
`<style>` blocks, `<link>`, **custom `@keyframes`** (no place to define them), authored `position`/`z-index`/`top`/`left`, `transition`, authored `:hover`,
`mask`/`-webkit-mask`, `video`, `iframe`, `svg` elements (use data-URI backgrounds instead), JavaScript (canvas, Web Audio, toggles).
(`position`, `:hover` and `:focus-within` ARE reachable indirectly through borrowed site classes — §2.)

**Caveats**
- Editing the bio later in the normal editor **reverts it** to basic formatting. Always re-upload the full HTML.
- Uploading a test bio **replaces** the real one — remind the user to re-upload the real bio afterwards.
- Minify before upload (strip comments + whitespace between tags).
- Avoid unconstrained `width`/`height`/`overflow` that could break the page for visitors. Prefer `max-width:100%` + `margin:0 auto`.
- Untested: everything in "Not seen" above. Test before relying on it.
- Not officially supported by JanitorAI; the user accepts account risk. Do NOT write or run the console/API upload step — the user does it from the guide. This skill only designs, minifies and previews the HTML.

## 2. Core techniques

### Hide the summary marker
`<summary style="list-style:none;display:block;cursor:pointer;">` — whole card becomes the click target.
(Safari may still show ▸; hiding it needs `::-webkit-details-marker`, impossible inline.)
Add `title="…"` to a summary for a hover tooltip ("Tutorial", "Game Modes").

### Image hero with readable text
```html
<div style="aspect-ratio:16/7;
  background:linear-gradient(rgba(0,0,0,.5),rgba(0,0,0,0) 26%,rgba(0,0,0,0) 50%,rgba(0,0,0,.85)),
             url(https://ella.janitorai.com/media-approved/ID.webp) center/cover;
  display:flex;flex-direction:column;justify-content:space-between;padding:8px 12px 10px;box-sizing:border-box;">
  <div><!-- top: location · time --></div>
  <div><!-- middle: tilted handwritten caption --></div>
  <div style="display:flex;justify-content:space-between;align-items:flex-end;"><!-- title + ▶ --></div>
</div>
```
Set `aspect-ratio` to the image's real ratio (e.g. `1916/821`) so no `<img>` is needed.
Side-fade variant: `background-image:linear-gradient(90deg,#171314 36%,rgba(23,19,20,.05)),url(...)` with text on the left.

### Layer stack (no position needed)
```html
<div style="aspect-ratio:1/1;display:grid;grid-template-columns:minmax(0,1fr);overflow:hidden;border-radius:4px">
  <div style="grid-column:1/2;grid-row:1/2;background:url(IMG) center/cover;filter:blur(7px) brightness(.55);transform:scale(1.15)"></div>
  <div style="grid-column:1/2;grid-row:1/2;align-self:center;text-align:center;color:#fff;transform:translateZ(0)">TEXT</div>
</div>
```
`scale(1.15)` hides blurred edges.
**Stacking gotcha (verified):** a layer with `filter`, `opacity<1` or `transform` paints ABOVE later sibling layers that have none of these. Give **every layer after a filtered/transparent one** `transform:translateZ(0)` so DOM order is respected again (fade overlays and the content layer especially).

### Swipe carousel
```html
<div style="display:flex;gap:14px;overflow-x:auto;scroll-snap-type:x mandatory;overscroll-behavior-x:contain;padding:4px 22px 14px;scrollbar-width:thin;scrollbar-color:#8c6a34 transparent">
  <div style="flex:0 0 78%;max-width:260px;scroll-snap-align:center;border-radius:6px;overflow:hidden">…</div>
</div>
```

### Double frame
`border:2px solid #8c6a34;outline:1px solid #8c6a34;outline-offset:-6px;padding:20px 14px`

### Rounded image frame
`<div style="max-width:460px;margin:0 auto;border-radius:12px;overflow:hidden;box-shadow:0 0 44px rgba(255,200,97,.16)"><img src="..." style="width:100%;display:block"></div>`
Overlapping/bleeding images: negative `%` margins on the wrapper; glowing cut-outs on dark bg: `mix-blend-mode:lighten` on the img.

### Tilted paper / note / polaroid / stamp
Wrapper `transform:rotate(-2deg);filter:drop-shadow(0 14px 24px rgba(0,0,0,.6))` around a `background:#f3ecdc` card
with `box-shadow:inset 24px 0 30px -20px rgba(0,0,0,.28)`. Stamp: `display:inline-block;transform:rotate(-12deg);border:3px solid #a3222b;color:#a3222b;border-radius:6px;opacity:.85`.

### Torn / burned parchment (border-radius + SVG strips; `clip-path` now also works, see *Cut corners*)
- Irregular `border-radius:3% 6% 2% 5% / 2% 3% 2% 4%` + `overflow:hidden` + inner glow `box-shadow:inset 0 0 70px rgba(60,40,20,.75)`.
- Torn top/bottom edge: a 14px strip with negative side margins and a URL-encoded SVG jagged polygon (filled with the outer bg color) as `background:url('data:image/svg+xml,…') top/100% 100% no-repeat`.
- Soot spots: `radial-gradient(circle at 94% 6%,rgba(15,12,10,.55),rgba(15,12,10,0) 16%)`.

### Invisible ink
Text colored exactly like the background it sits on → only readable when highlighted. Sample the real rendered bg color (screenshot pixel) — grain/hatch layers shift it.

### Responsive link cards / image links
Row `display:flex;flex-wrap:wrap;gap:12px`; card `<a href target="_blank" rel="noopener noreferrer" style="flex:1 1 280px;min-width:0;display:block;text-decoration:none;border-radius:12px;overflow:hidden">`. An `<a>` containing only an `<img>` works as an image button.

### Inline avatar chip
`<span style="display:inline-block;width:22px;height:22px;border-radius:50%;background:url(IMG) center/cover;vertical-align:middle"></span>`

### Patterns
- Striped awning: `background-color:#2f8f8a;background-image:repeating-linear-gradient(90deg,#2f8f8a 0 22px,#1b5753 22px 25px,#16120d 25px 29px),linear-gradient(180deg,transparent 0 58%,rgba(0,0,0,.25) 58% 70%,transparent 70%)`
- Paper texture: `repeating-linear-gradient(0deg,rgba(120,90,40,.035) 0 2px,transparent 2px 6px)`
- Bullet rings: `width:18px;height:18px;border-radius:50%;box-shadow:inset 3px 0 0 -1px #c4372e,inset -3px 0 0 -1px #c4372e`
- Anything else: URL-encoded `data:image/svg+xml` background.

### Image textures (uploaded, transparent) — preferred over CSS-only hatching
Upload textures as **white marks on a transparent background** (PNG → ella WebP keeps alpha). One file can then be recolored by `filter` on its own layer:
- black ink: `filter:brightness(0)`
- blood red: `filter:brightness(0) saturate(100%) invert(9%) sepia(94%) saturate(6000%) hue-rotate(358deg) brightness(88%) contrast(115%)`
- as-is (white): no filter — dust/specks on dark

Recipes (verified live on the Flandre bio):
- **A · grain behind text panels:** layer `background:url(NOISE) center/640px;filter:brightness(0);opacity:.16`, content layer `transform:translateZ(0)`.
- **B · corner hatching on the page:** layer `background:url(HATCH) center top/100% auto no-repeat,url(HATCH) center bottom/100% auto no-repeat;filter:brightness(0);opacity:.55` + a fade layer (`transform:translateZ(0)`) of parchment-colored gradients: `radial-gradient(ellipse 42% 460px at 50% 0,P 45%,transparent 100%)`, same `at 50% 100%`, and `linear-gradient(transparent 0,transparent 260px,P 620px,P calc(100% - 620px),transparent calc(100% - 260px))`. Put inset burn shadows and soot spots on that fade layer (the base shadow is hidden under the layers). Do NOT use `cover` on a tall page — it blows the hatch up into marble-like blotches.
- **C · dust on dark frames/headers (no layer needed):** `background:linear-gradient(rgba(20,17,15,.8),rgba(20,17,15,.8)),url(NOISE) center/640px,#14110f` (first listed = top).
- **D · blood mist on dark panels:** red-filtered noise layer `opacity:.85` + fade layer `radial-gradient(ellipse at 50% 100%,rgba(20,0,0,0) 25%,rgba(20,5,3,.9) 85%)`.
- Non-seamless textures: never tile, use `100% auto` placements. Noise tiles OK at ≥640px (smaller shows a quilt grid).

### Recoloring photos (e.g. coffee stain → blood)
`hue-rotate` alone fails on pale images. Strip color first, then re-add it:
- fresh: `filter:grayscale(1) brightness(.55) contrast(2.2) sepia(1) saturate(25) hue-rotate(-38deg) brightness(.7)`
- dried: `filter:grayscale(1) brightness(.4) contrast(2.4) sepia(1) saturate(18) hue-rotate(-36deg) brightness(.5)`
- strongest red with texture: grid-stack two copies — bottom with the blood-red silhouette filter above, top `filter:grayscale(1) contrast(1.8);mix-blend-mode:multiply`.
Past about -45deg the hue goes magenta. Add `mix-blend-mode:multiply` on parchment so it soaks in.

### Animations (site keyframes)
Inline `animation` works, but you can only reference @keyframes JanitorAI already ships. Present on character pages (2026-10-30; `shine`, `plus-shiny-username`, `custom-light-off`, `dot` seen in use again 2026-11-02):
| name | does | use |
|---|---|---|
| `sonner-fade-in` | opacity 0→1, scale .8→1 | entrance: `animation:sonner-fade-in .9s ease-out both`; card body "expanding" on first open (see *First-open reveal*) |
| `jump` | translateY 0→-10px→0 | bouncing "OPEN ▾" |
| `dot` | scale .8→1.1→.8 | heartbeat / pulse |
| `custom-flicker` | opacity 1→.8→1 | candle / glow flicker |
| `custom-light-off` | opacity 1→0 at 80% | blink-out; use `steps(1,end)` for a hard blink |
| `shine` | translate -100%→100% | light band sweeping over a card (grid layer + `overflow:hidden`) |
| `loading` | rotate 0→360° | slow spinning sigil (`6s linear infinite`) |
| `plus-shiny-username` | background-position → -400%/200% | moving gradient text with `background-clip:text` |
| `stopgenerate` | border-width + scale pulse | untested in bios |
Not available: `ProseMirror-cursor-blink` (only on editor pages), hashed names like `_fadeIn_1nhc3_1` (change every site build — never use).
Rules:
- Animated `transform` **overrides** an inline `transform` (rotate/scale) on the same element → animate a wrapper or child instead.
- The site can rename/remove these at any time; the effect then silently stops — design so the bio still looks right without motion.
- Keep it subtle: 2–4 looping effects max.
- Check what exists: list `@keyframes` from `document.styleSheets` on a character page (see §4).

### First-open reveal (details body) — verified 2026-11-13
Put the site keyframe on the div right after the summary:
```html
<details>
  <summary style="list-style:none;display:block;cursor:pointer">…card…</summary>
  <div style="animation:sonner-fade-in .35s ease-out">…text…</div>
</details>
```
- The 0.8→1 scale + fade reads as the text "expanding" out of the card. `.35s` for small cards, `.45s` for big panels.
- **Plays only the first time** the card opens. Chrome hides closed details content with `content-visibility:hidden` (`::details-content`), not `display:none`. The animation is created at page load but stays at 0ms until first open. After that the element never leaves the render tree, so closing and reopening doesn't replay it (checked with `getAnimations()`). Other browsers may behave differently.
- Don't put your own `transform` (tilt) on the same div, because the animation overrides it. Tilt an outer wrapper.
- If the keyframe is renamed, the text just appears without animating. Safe failure.

### Marquee (motion without keyframes) — verified 2026-11-11
`<marquee>` survives the sanitizer and actually scrolls (measured ~70px/s, Chrome's default speed). It doesn't depend on site keyframe names, so it won't silently stop on a site rebuild.
- **Tag ticker:** `<marquee loop style="width:100%"><span style="display:inline-block;padding:.38rem 1.2rem;margin-right:.8rem;border-radius:999px;background:#1f1117;border:1px solid #43232e">Angst</span>…</marquee>` → genre pills scroll across the bio.
- **Moving icon in a button:** `<marquee width="20" style="display:inline-block;color:#d6264f">▶</marquee> Play` inside an `inline-flex` pill → the ▶ keeps sliding through a 20px window.
- Untested: `scrollamount`, `scrolldelay`, `direction` (right/up/down), `behavior="alternate"`, `truespeed`, `hspace`. Test before relying on them.
- Content scrolls out of view for part of each loop → never put the only copy of important info in one. 1–2 per bio.

### Borrowed site classes (hover + position) — verified 2026-11-02, fragile
`class` survives, so an element can wear one of JanitorAI's own CSS-module classes and inherit its rules, including `position`, `:hover` and `:focus-within`, which inline styles can't do. Inline styles still win over the class, and inline `--vars` recolor anything the class paints with `var()`.

Seen in use (class names from the 2026-11-02 build):
| classes | site rule | bio use |
|---|---|---|
| `<a class="_root_l8p4e_1">` + child `<span class="_popup_l8p4e_7">` | root `display:inline-flex;position:relative`; popup `position:absolute;bottom:calc(100% + 8px);left:50%;opacity:0;visibility:hidden;z-index:2000`, shown on `:hover`/`:focus-within` of root | **hover image preview**: popup span gets inline `width`, `aspect-ratio`, `background-image:url(ART)`, `--bg-dark:#000;--border-color:#ff8a2a66` |
| `<a class="_modalClose_1nhc3_78">` | `cursor:pointer`, `color:var(--text-secondary)`; `:hover` → `background-color:var(--colour-white-a10);color:var(--text-primary)` | **hover-highlight chip/button** (e.g. hashtag links), recolored with `--text-secondary:#ff8a2a;--text-primary:#f3e9dc;--colour-white-a10:#ff8a2a2e` and inline `width/height/padding` overriding its 28–32px size |

Rules:
- Hashed names (`_xxx_l8p4e_1`) **change on site rebuilds**. Before using, confirm the class still exists on a character page (§4) and re-check after uploads.
- Plan the failure state: if the class disappears, `opacity:0;visibility:hidden` go with it and the popup becomes a visible inline box. Keep popups small, or give the popup span a fallback that looks OK inline. Hover chips just lose their hover color (safe).
- Hover doesn't exist on touch; popups also open on tap/focus (`:focus-within`) only if the `<a>` is focusable — always keep the info reachable some other way.
- Prefer this only for flourishes; the core bio should work without it.

### Viewport breakpoint (clamp switch) — verified 2026-11-02
A media-query substitute using huge coefficients so a value jumps between two extremes around a viewport width B:
```html
<!-- shown only when viewport > 760px -->
<div style="max-height:clamp(0px,calc((100vw - 760px) * 99999),99999px);overflow:hidden">DESKTOP LAYOUT</div>
<!-- shown only when viewport < 760px -->
<div style="max-height:clamp(0px,calc((760px - 100vw) * 99999),99999px);overflow:hidden">MOBILE LAYOUT</div>
```
The renderer folds it to `clamp(0px, -7.59e+07px + 9.9999e+06vw, 99999px)`; that's fine.
- Same trick picks between two sizes: `width:max(min(A, calc((100vw - Bpx)*99999)), min(C, calc((Bpx - 100vw)*99999)))` → A above B, C below (seen on popup widths/margins, B ≈ 1024px).
- It's the **window** width (`vw`), not the bio column. On desktop the bio is ~50% of the window, so pick B with that in mind; for column-relative switching try the same formula with `cqw` inside a `container-type:inline-size` wrapper (untested).
- Both versions are in the DOM (doubles the markup), and hidden content still takes no height thanks to `overflow:hidden`.
- Seen: a terminal-style "Credits" panel for wide screens vs a "tap a tab to open it" tab layout for phones.

### Skip buttons (jump to a section) — verified 2026-11-13
A `<label>` focuses a hidden checkbox placed just above the target section. Chrome scrolls a newly focused element to the **vertical center** of the viewport.
```html
<!-- button -->
<label for="go-contracts" style="cursor:pointer;display:inline-block;padding:9px 16px 8px;border-radius:99px;background:#c9a227;color:#101317;font:700 12px/1 'JetBrains Mono',monospace;letter-spacing:2px">SKIP TO THE CONTRACTS ↓</label>

<!-- anchor, right before the target section -->
<div style="display:flex;height:0;overflow:visible">
  <input type="checkbox" id="go-contracts" style="flex:0 0 auto;margin:340px auto 0;opacity:0;width:1px;height:1px">
</div>
<div><!-- target section --></div>
```
- The 0-height flex holder keeps the input out of the layout. `opacity:0` + 1px size makes it invisible.
- **`margin-top` sets where the section lands.** Because the input is centered on screen, pushing it N px below the section's start places the heading about N px above center. Measured: 340px → heading 44px from the top on a 770px-tall window. On short phone screens 340px can push the heading above the top edge, so ~200–250px is safer.
- Use a checkbox, not a text input: focusing it on a phone doesn't open the keyboard. Its checked state doesn't matter, because nothing reads it.
- Several skip buttons can sit in one row, each pointing to its own anchor.
- Only measured in Chrome. Firefox/Safari may scroll a focused element to a different position.

### Exclusive accordion (details name)
Give sibling `<details>` the same `name="group"` → only one stays open (native browser behaviour, Chrome 120+/Safari 17.2+; older browsers just allow several open). Add `open` to the one that should start expanded. Good for scene selectors, galleries, tab-like switchers.

### Column-relative sizing (container queries)
```html
<div style="container-type:inline-size;max-width:100%;margin:0 auto">
  <div style="font-size:clamp(12px,4.5cqw,27px);padding:clamp(8px,3cqw,19px)">…</div>
</div>
```
`cqw` = 1% of the wrapper's width, so headings shrink correctly in the 307px side-panel column and grow on full screen. Always clamp.

### Cut corners / angled shapes
`clip-path:polygon(…)` on any div (diagonal corner tabs, ticket notches, torn edges). Pair overlays with `pointer-events:none` so they don't eat clicks.
For hinged bars/arrows: `transform-origin:0 100%;transform:rotate(-45deg)` (and the mirrored `100% 100%` / `45deg`).

### Small text helpers
- One-line truncation: `overflow:hidden;white-space:nowrap;text-overflow:ellipsis`.
- Balanced multi-line titles: `text-wrap:balance`.
- Dark native controls: `color-scheme:dark` on the wrapper.

### Scene selector
One `<details>` per scene, stacked with `border-top:2-3px solid #000`. Summary = hero card ("01 · LABEL / Title / date · place ＋ open"); body = commentary,
mood tag, setup, "GREETING #n". Scenes start closed.

### Audio
Full player card: `<div style="max-width:520px;margin:0 auto;border-radius:14px;padding:16px;color-scheme:dark"><audio controls loop preload="none" src="https://files.catbox.moe/ID.mp3" style="width:100%"></audio></div>`.

Round button (crop trick):
```html
<span style="display:block;width:32px;height:32px;min-width:32px;border-radius:50%;overflow:hidden;">
  <audio controls src="AUDIO_URL" style="display:block;width:300px;height:32px;margin:0 0 0 -4px;"></audio>
</span>
```
The 300px player is cropped by the 32px circle so only the native play button shows.

### Nested details patterns
- **Phone chat:** each message's summary = "3:06 PM tap ▾", nested inside the previous one → messages reveal in order.
- **Book:** summary "TURN THE PAGE ›" nested repeatedly; last one "THE LAST PAGE ›".
- **Timeline:** "DEC 20, 2023 · SCROLL ▾" nested per entry.
- **Descent / stairs:** "THE STAIRS · TOP ▾" → "THE MIDDLE ▾" → "THE BOTTOM ▾", each level a darker background.
- **Reveal panel (replaces JS toggles):** "🕯 LIGHT THE CANDLE" summary → dark panel with glowing `text-shadow` secrets.

### Replacing JS features from inspiration code
| JS / CSS feature | Bio substitute |
|---|---|
| tabs / chapter switcher | stacked `<details>` |
| modal | nested `<details>` card |
| mode toggle (candle light) | `<details>` reveal panel |
| Web Audio music | round audio button + hosted mp3 (`loop` for BGM) |
| canvas hatching / soot | uploaded transparent textures (recipes above) |
| `:hover` lift | static `rotate()` tilt; hover color/popups via borrowed site classes (fragile) |
| tooltip | `title` attribute, or borrowed popup class |
| `@keyframes` | site keyframes only (table above) |
| scrolling ticker / sliding icon | `<marquee>` |
| tabs where one closes the others | `details name="group"` |
| `scrollIntoView` / "skip to section" link | `label for` + hidden checkbox (*Skip buttons*) |
| JS / CSS reveal animation on expand | `sonner-fade-in` on the details body (*First-open reveal*) |
| media queries | `container-type:inline-size` + `cqw` (column), clamp switch with `vw` (window) |
| sketchpad / anything needing JS | impossible |

### Fake UI decorations
Torii, ofuda, stamps, dividers: bordered/colored `div`s and gradient lines (`flex:1;height:1px;background:linear-gradient(90deg,transparent,#b3122e)`).

## 3. Workflow

1. Ask: which block (scene selector, phone, book, carousel, audio card, full bio), theme, number of scenes, image ratio, textures/audio available.
2. Write a **readable** version with `<!-- comments -->` and `[PLACEHOLDERS]` (list every placeholder in a header comment); inline styles only, no classes of your own (borrowed site classes only, flagged as fragile). A small Python build script that injects generated data-URI SVGs (`urllib.parse.quote`, `#`→`%23`) is handy.
3. Generate a **minified** copy:
   ```python
   import re
   s=re.sub(r'<!--.*?-->','',s,flags=re.S); s=re.sub(r'>\s+<','><',s); s=re.sub(r'\s*\n\s*',' ',s).strip()
   ```
4. Render-check with Playwright/Chromium at **307px and 700px** column widths: closed state, then all `details` opened (`d.open=true`), full-page screenshots; check `scrollWidth` for horizontal overflow. If the bio uses a clamp switch, also test window widths on both sides of each breakpoint. Substitute data URIs for placeholder textures in the preview. Fonts from Google may not load in the sandbox — that's fine. Borrowed site classes won't apply in the sandbox preview — check them live.
5. Deliver both files (`name.html`, `name.min.html`). Remind: re-upload the full HTML after every change.
6. After the user uploads, verify on the live page (Claude in Chrome): computed styles of key layers, loaded fonts, image transparency, details count, borrowed classes still matching a site rule.

## 4. Auditing live bios (to update this skill)
- Bio container on a character page: `.css-ly0sg4` (Chakra hash — still valid 2026-11-02, may change; fallback: common ancestor of all `details`).
- Rendered `id`/`name` values carry a `user-content-` prefix — strip it when comparing to source.
- Audit the **rendered DOM**, not the `/hampter/characters/<id>` API `description` (it can differ from what renders, and NSFW bots return 401 without auth).
- Tally tags, attribute names, `el.style` properties (include `--custom` props), image/audio hosts, and `document.fonts` loaded families (wait for `document.fonts.ready`).
- Tally `class` values too: anything that isn't `chakra-image`/`css-*` is author-borrowed — find its rules by walking `document.styleSheets` for selectors containing the class name.
- Look for huge coefficients (`e+0`) in `clamp()/min()/max()` values → breakpoint switches; solve for the breakpoint.
- Check a property survived: read `el.getAttribute('style')` (re-serialized) and `getComputedStyle(el)`.
- List available keyframes: iterate `document.styleSheets` → `cssRules` with `type===7` (`CSSKeyframesRule`).
- Check uploaded image alpha: load with `crossOrigin='anonymous'` into a canvas and count `alpha<10` pixels (ella allows CORS reads).
- Claude in Chrome's JS tool blocks results that look like cookies/query strings: strip `=`, `&`, `?` and URLs from the returned string (`.replace(/[=&?]/g,' ')`), and return small chunks.
- To test something new: build a tiny labeled test block (one card per feature), have the user upload it to a test bio, then inspect. Remind them to re-upload the real bio.
- Ignore renderer-added attrs (`chakra-image`, `css-*` classes, `node`, `draggable`). Add confirmed items to §1.

## 5. Ethics / scope
- Analyzing other creators' bios to learn techniques is fine; don't reuse their images, text, or hidden definitions.
- Images must be the user's own, uploaded to JanitorAI's image host.
- Public bios: keep characters presented as adults and keep sexual content out of the bio, especially for canonically young-looking characters.