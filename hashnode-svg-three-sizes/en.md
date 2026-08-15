---
title: "SVG viewBox vs width/height vs CSS: The Three Sizes That Break Icons"
seo_title: "SVG viewBox vs width/height vs CSS: Fix Icon Size Bugs"
slug: svg-viewbox-vs-width-height-css
description: "Why Figma SVGs look the wrong size: viewBox is the contract, width/height are defaults, CSS/props set size. Retina PNG math + a 60s audit."
locale: en
canonical: https://getsvgeditor.com/blog/svg-viewbox-vs-width-height-css
keywords:
  - SVG viewBox
  - SVG width height
  - SVG CSS size
  - React SVG icon size
  - Figma SVG export size
  - retina PNG from SVG
  - SVG icon sizing
  - preserveAspectRatio
og_image: ./images/01-three-sizes.svg
published: true
---

# SVG viewBox vs width/height vs CSS: The Three Sizes That Break Icons

You export a 24×24 icon from Figma. In the browser it shows up tiny, huge, or sharp — but trapped inside a soft PNG. The file is almost never “broken.” It’s carrying **three different sizes at once**, and your tools listened to the wrong one.

Most advice boils down to “don’t touch the `viewBox`.” That’s necessary, but not enough. If you run a design system, wrap icons in React, or ship PNGs for email, you need to know which size is the **contract** — and which two are just defaults.

![Three SVG size signals: viewBox is the contract, width/height are defaults, CSS/props set the on-screen size](./images/01-three-sizes.svg)

## SVG viewBox vs width/height vs CSS — three sizes

A typical design-tool export looks harmless:

```xml
<svg
  xmlns="http://www.w3.org/2000/svg"
  width="24"
  height="24"
  viewBox="0 0 24 24"
>
  <path d="…" />
</svg>
```

That’s **three independent signals**:

| Signal | What it’s for | Who cares |
| --- | --- | --- |
| **`viewBox`** | The coordinate system: “this drawing lives in a 24×24 space.” | Renderers, `preserveAspectRatio`, any serious icon pipeline |
| **`width` / `height` attributes** | The default on-screen size when nothing else overrides it | The browser (if CSS is silent), some PNG exporters, naïve copy-paste |
| **CSS / props** (`className`, `width={32}`) | The size **this use** should have in the UI | React, layout, design tokens |

When they disagree, you get the usual bugs:

- Fine as `<img src="icon.svg">`, wrong as inline JSX
- `<Icon width={16} />` seems to do nothing (CSS or attribute order wins)
- Retina PNG looks soft because you rasterized the **attribute** size, not the CSS size you actually show
- `viewBox="0 0 24 24"` while paths were drawn in `0…100` → empty padding or a clipped glyph

**Rule of thumb:** `viewBox` is the contract. Attributes are a fallback. CSS and props are the on-screen size.

### How the browser picks the size

Inline SVG and `<img src="*.svg">` do **not** follow the same rules.

**Inline SVG** in HTML or JSX is a real DOM tree. Layout checks CSS first, then presentation attributes on the root `<svg>`.

**`<img>` and `background-image`** treat the file like an image. The browser looks for an intrinsic size: absolute root `width`/`height`, otherwise aspect ratio from `viewBox`, otherwise the infamous fallback — **300×150**.

For icon bugs, the order is usually:

1. CSS, flex, or grid sets the **box on screen**.
2. If CSS is silent, root `width`/`height` become the intrinsic size (bare `24` in HTML means `24px`).
3. If those are missing too, `viewBox` only gives an aspect ratio — and if there’s still no concrete size, many engines paint a **300×150** box. That’s how a “tiny icon” becomes a banner after someone “cleaned up” the attributes.
4. `viewBox` still decides **what** is drawn inside the box. CSS and attributes only decide **how large** that box is.

`viewBox` units are not CSS pixels. They’re the drawing’s own coordinate space. `stroke-width="1.5"` means one and a half of those units. Scale the icon from 24 to 48 CSS pixels and the stroke looks twice as thick on screen — unless you set `vector-effect="non-scaling-stroke"`.

That projection is what this whole article is about.

---

## How SVG sizing works: coordinates, projection, pixels

Think of SVG as a canvas API with XML syntax:

1. **Coordinates** — numbers in `d`, `cx`, `x`, and so on live in `viewBox` units.
2. **Projection** — the renderer stretches that box onto a layout box or a pixel grid when you export.
3. **Pixels** — only appear when something asks for a bitmap: PNG, `<canvas>`, a screenshot, or a flattened PDF.

![Authoring coordinates project onto a layout box and become pixels only when you rasterize](./images/02-mapping-model.svg)

If the projection is wrong, tweaking `stroke-width` won’t help. Fix `viewBox` and where it lands first.

### Example: padding inside a 24×24 frame

Design tools love this setup: frame is 24, glyph is about 16, centered.

```xml
<svg width="24" height="24" viewBox="0 0 24 24">
  <!-- glyph roughly occupies 4…20 on both axes -->
  <path d="M8 6h8v12H8z" />
</svg>
```

| What you do | What you get |
| --- | --- |
| Inline it in a 24px button | Looks fine |
| Rasterize at 24×24, then show the PNG at 16 CSS px | Soft edges |
| Drop `width`/`height`, keep `viewBox`, size with CSS or props | Sharp at any pixel size |

Icon systems want the third row: **a stable `viewBox`, no baked-in presentation size, and the instance size set where the icon is used.**

### `preserveAspectRatio` — the quiet fourth control

When the CSS box aspect ratio doesn’t match the `viewBox`, `preserveAspectRatio` decides letterboxing vs cropping vs stretching:

| Value | Behavior |
| --- | --- |
| `xMidYMid meet` (default) | Fit inside, keep proportions, may leave empty space |
| `xMidYMid slice` | Fill the box, keep proportions, may crop |
| `none` | Stretch to fill (almost always wrong for icons) |

If an icon looks squashed after `width: 32px; height: 20px`, the path isn’t broken — you changed the box and did (or didn’t) keep the aspect ratio.

---

## Why viewBox “doesn’t work”: the coordinates are wrong

```xml
<!-- paths drawn in 0…100 space; viewBox is lying -->
<svg width="24" height="24" viewBox="0 0 24 24">
  <circle cx="50" cy="50" r="40" />
</svg>
```

The circle’s center sits outside the declared box. You can “fix” size in CSS all day and still see a clipped disc. The `viewBox` contract was wrong — not the button styles.

![A lying viewBox clips art drawn in a larger coordinate space](./images/05-viewbox-lie.svg)

**How to check in 30 seconds**

These are three different rectangles — don’t mix them up:

| What you inspect | Units | What it tells you |
| --- | --- | --- |
| `viewBox` | Drawing units | The **contract** you declared |
| `el.getBBox()` | Drawing units | Where the ink actually is (ignores CSS size) |
| `el.getBoundingClientRect()` | CSS pixels | The box layout reserved on screen |

```js
const svg = document.querySelector('svg');
const ink = svg.querySelector('g, path, circle');
console.log(svg.getAttribute('viewBox'));     // declared contract
console.log(ink.getBBox());                   // real ink bounds
console.log(svg.getBoundingClientRect());     // on-screen CSS box
```

If `getBBox()` is roughly `0…100` and `viewBox` is `0 0 24 24`, the file is lying. CSS won’t save you.

**A common Figma export pattern:** the ink gets wrapped in `<g transform="translate(4 4)">` (or nested translates), while the root still claims `viewBox="0 0 24 24"`. It looked centered in the design tool; the shipped file’s contract doesn’t match the path numbers. Fix it once — recompute `viewBox` from `getBBox()`, or flatten transforms in the exporter — instead of patching `translate` in every PR.

Don’t invent `0 0 24 24` just because “icons are 24.” Measure the real bounds, then pick a team grid (20 or 24) and normalize to it.

---

## Why React breaks SVG icon size

Browsers forgive messy HTML. React makes the conflict obvious through **prop order**:

```jsx
// Component default: 24. Call site asks for 16. Who wins?
export function Icon(props) {
  return (
    <svg
      viewBox="0 0 24 24"
      width={24}
      height={24}
      fill="none"
      {...props}
    >
      <path d="…" stroke="currentColor" strokeWidth={1.5} />
    </svg>
  );
}

<Icon width={16} height={16} className="text-slate-700" />
```

With `{...props}` last, `16` wins. Move the spread earlier and the call site quietly loses. Teams can spend a week arguing about “broken icons” before someone checks attribute order.

![React SVG prop order: spread early and the call site loses; spread last and the caller wins](./images/03-react-prop-order.svg)

**Before and after — the shape of a real PR:**

```jsx
// BEFORE — fine in Storybook at default 24, “broken” in a dense toolbar
export function Chevron(props) {
  return (
    <svg viewBox="0 0 24 24" {...props} width={24} height={24}>
      <path d="…" stroke="currentColor" strokeWidth={1.5} />
    </svg>
  );
}
// <Chevron width={16} /> still paints at 24 — the spread lost

// AFTER — one size API, call site wins, viewBox left alone
export function Chevron({ size = 24, ...props }) {
  return (
    <svg viewBox="0 0 24 24" width={size} height={size} aria-hidden="true" {...props}>
      <path d="…" stroke="currentColor" strokeWidth={1.5} />
    </svg>
  );
}
```

Watch **types in React Native** too. Web often forgives `width="24"` as a string; `react-native-svg` wants `width={24}` as a number. A converter that emits quotes “breaks size” only on mobile — another fake “SVG is broken” ticket.

### A clean default for UI icons

```jsx
export function Icon({ size = 24, ...props }) {
  return (
    <svg
      viewBox="0 0 24 24"
      width={size}
      height={size}
      fill="none"
      aria-hidden="true"
      {...props}
    >
      <path d="…" stroke="currentColor" strokeWidth={1.5} />
    </svg>
  );
}
```

Leave `viewBox` alone. Expose one size API. Don’t also hard-code a second size in CSS and a third in the PNG pipeline.

**CSS trap:** if your stylesheet has `.icon { width: 24px !important; }`, props will look broken even with perfect JSX order. On-screen size needs one owner.

React Native follows the same idea with numeric props (`width={24}` on `<Svg>`). Same contract: `viewBox` in the file, size at the call site. If you need JSX quickly, paste into the [SVG → React converter](https://getsvgeditor.com/svg-to-react) — the React Native tab sits next to it, so you can check numeric props against the preview.

### Tips for SVGR / design systems

When you generate components from `.svg` files:

- Keep `viewBox`.
- Prefer replacing fixed `width`/`height` with a `size` prop (or overridable `width`/`height`).
- Make sure `{...props}` can override size and accessibility attributes.
- Theme UI icons with `currentColor`, not baked-in `#000`.

---

## Why PNG from SVG looks soft on retina

PNG doesn’t care about your design tokens. It only knows **how many pixels you asked the rasterizer for**.

The failure mode that still shows up in handoffs:

1. SVG `width`/`height` attributes say 24×24  
2. You export a 24×24 PNG  
3. You show it at 48 CSS pixels on a 2× screen **without** a 2× asset  
4. Someone files “SVG looks bad”

The SVG was fine. You baked the wrong pixel count.

![PNG export size = CSS size × device pixel ratio](./images/04-retina-pixel-budget.svg)

### CSS size → DPR → PNG

| CSS size | Screen density | PNG you should export |
| --- | --- | --- |
| 24×24 | 1× | 24×24 |
| 24×24 | 2× | 48×48 |
| 16×16 | 2× | 32×32 |
| 24×24 | 3× | 72×72 |

Never upscale a 16px PNG in CSS and call it retina.

### Serving retina PNGs the right way

```html
<img
  src="/icons/check-24.png"
  srcset="/icons/check-24.png 1x, /icons/check-48.png 2x, /icons/check-72.png 3x"
  width="24"
  height="24"
  alt=""
/>
```

`width`/`height` (or CSS) keep the **layout slot** at 24 CSS pixels. `srcset` supplies enough device pixels. Mixing those two up is how “sharp SVG → soft PNG” tickets start.

### Before you export SVG to PNG

1. `viewBox` matches the real path bounds — no surprise crop  
2. Decide the **CSS display size** first (email, CMS, whatever)  
3. Multiply by the density you care about, then export  
4. Prefer transparency; flatten to white only if the host requires it  
5. If the host can take SVG (most modern web UI), you often don’t need PNG  

When an export looks soft but the paths are fine, stop guessing: paste the markup, nudge `width`/`height` against the preview, then download PNG. In the [SVG → PNG converter](https://getsvgeditor.com/svg-to-png) export is **2×** the SVG size — the usual retina handoff: show at 24 CSS px, file is 48 px.

---

## Inline SVG vs img vs React component

| Approach | Size control | Theming | Best when |
| --- | --- | --- | --- |
| `<img src="*.svg">` | CSS on the image box | Weak (`currentColor` won’t theme the file) | Static logos, CMS content |
| Inline SVG | CSS + attributes | Strong | One-off art on a page |
| React / RN component | props + tokens | Strong | Design-system icons, repeated UI |

Size bugs show up in all three. Components just make the contract explicit: **`viewBox` in the file, size at the call site**.

For a deeper comparison of embedding methods (inline vs `<img>` vs background vs sprite), that’s a separate guide — same topic, different decision.

---

## Edge cases

These are the “we already keep viewBox” bugs that still ship.

### 1. No size at all → 300×150 billboard

No CSS size, no root `width`/`height`, only a `viewBox` — many engines fall back to SVG’s default intrinsic size (often **300×150**). Your 24 icon becomes a banner. Set size with CSS or a `size` prop, or keep sensible attributes when the file isn’t a component.

### 2. SVGO removed your defaults (and sometimes more)

Optimizers may strip `width`/`height` (fine for icon systems) or rewrite `viewBox`. Configure SVGO so the **contract** survives. Removing presentation size is fine. Removing or inventing `viewBox` is not.

### 3. Nested transforms from Figma

Exports can wrap the art in `<g transform="translate(...)">` while the root box doesn’t match the ink. Normalize once (union, flatten transforms, “Outline stroke” policy) instead of editing paths in every PR.

### 4. `vector-effect="non-scaling-stroke"`

When icons scale from 16→32, hairlines get thicker unless you opt into non-scaling strokes. Pick one rule for the whole set: strokes scale with the icon (default), or optical weight stays roughly constant. Don’t mix both strategies in one icon set.

### 5. Hard-coded `#000` + wrong size = “the icon is broken”

Wrong color and wrong size often land as one ticket. For UI icons, prefer `currentColor` or CSS variables so theming and sizing stay separate.

### 6. Email clients

Many email clients are hostile to inline SVG. Plan on PNG (usually 2×) with a 1× CSS/`width` slot in the template. Don’t expect `currentColor` magic in HTML email.

### 7. `width="100%"` without a definite height

Percentages resolve against the containing block. An SVG with `width="100%"` and no height / no CSS aspect ratio can collapse, stretch via `preserveAspectRatio="none"`, or hit the 300×150 trap — depending on context. For icons: absolute defaults or a `size` prop, not percentages.

### 8. `overflow` and “missing” strokes

Root SVG defaults to `overflow: hidden` in many user-agent stylesheets. Art that sits flush against the `viewBox` edge can clip antialiased strokes when scaled. Either leave optical padding inside the 24 grid, or set `overflow="visible"` on purpose — don’t widen `stroke-width` until you’ve ruled out clipping.

---

## 60-second SVG icon audit

Run this before you merge or before you rasterize:

1. **Honest root `viewBox`?** Don’t invent `0 0 24 24` if the art is `0 0 20 20`.  
2. **Do `width`/`height` duplicate a size CSS already sets?** Remove them or align to the default token.  
3. **In JSX, can callers override size and `aria-*`?** Check `{...props}` order.  
4. **Hard-coded `#000` on UI icons?** Prefer `currentColor`.  
5. **Need PNG?** CSS size → × DPR → rasterize. Never upscale a tiny bitmap.  
6. **Is `preserveAspectRatio` fighting a non-square box?** Check before blaming paths.  
7. **No CSS and no attributes?** Watch for the 300×150 default.  
8. **Do `getBBox()` and `viewBox` agree?** If not, fix the contract before the button CSS.  
9. **Hairline clipped at the frame edge?** Check `overflow` / optical padding — not only `stroke-width`.  

---

## Agree on sizing once as a team

Write it down so you stop rediscovering it in every PR:

| Layer | Decision |
| --- | --- |
| **Authoring** | One square `viewBox` — 24 *or* 20, pick one |
| **Web** | Components with `size` + `currentColor`; no per-instance path edits |
| **Email / older hosts** | PNG at 2×, template sized to the 1× CSS slot |
| **Pipeline** | SVGR (or similar) for a steady stream of `.svg` files; paste-to-JSX when you need one component and a live preview now |

Three sizes will always exist. Production stays calm only when **one** of them is allowed to win — and that should be the size at the call site, not a surprise baked into the file.

### When the next icon “looks broken”

Don’t start with `stroke-width`. Ask which signal won:

1. Is the `viewBox` honest?  
2. Are attributes fighting CSS or props?  
3. If you need a bitmap — did you export **CSS size × screen density**, not the SVG attribute size?  

Paste the markup, check the contract against a live preview, set the size where the icon is used, and rasterize only when the host can’t take SVG.

Keep the contract in the asset. Keep the size argument at the call site. The postage-stamp icons and soft PNGs usually disappear after that.
