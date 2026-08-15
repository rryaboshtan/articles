---
title: "Figma SVG Looks Fine as a File and Breaks When Inlined"
seo_title: "Figma SVG Inline Bugs: IDs, SVGO, Strokes"
slug: figma-svg-inline-ids-svgo
description: "Figma SVG that previews fine then dies in React: colliding clip0_ ids, .st0 styles, inside/outside strokes, and SVGO presets that remove viewBox or fill=none."
locale: en
canonical: https://getsvgeditor.com/blog/figma-svg-inline-ids-svgo
keywords:
  - Figma SVG inline
  - SVG id collision clipPath
  - SVGO removeViewBox
  - Figma SVG currentColor
  - Figma outline stroke SVG
  - SVG fill-rule evenodd
og_image: ./images/05-inspect-source-preview.svg
published: true
---

# Figma SVG Looks Fine as a File and Breaks When Inlined

You cleaned the Frame clip. The chevron shows in a tab. You drop two icons into React and one vanishes, a gradient leaks, or SVGO “optimizes” a line icon into a black fill. The ticket says the framework is broken. The documents were never written to share a page.

An SVG opened as a file, or loaded with `<img src>`, is a **separate document**. Its `id="clip0_12_8"` lives there. Inlined in HTML or JSX, every icon shares the page’s ID namespace and CSS. Figma’s exporter does not know you will paste forty of these next to each other. SVGO does not know that `clip0_12_8` was a Frame. The canvas stroke model is richer than SVG’s, so “it looks right in Figma” still ships an approximation.

**Short version:** if it dies only when inlined, search duplicate ids and `.st0` first, not React. Prefix or strip ids. Delete editor `<title>` and generic stylesheets. Do not run a default SVGO preset on a file you have not looked at. Outline text and decide stroke alignment **in Figma** for the whole set. If the preview is still empty or shifted *as a single file*, that is not this article — go back to [inspecting the scene graph](https://getsvgeditor.com/blog/figma-svg-empty-shifted-clipped).

This piece is the shipping pass: collisions, leftover machinery, strokes, booleans, filters, and a recipe that scales past one heroic cleanup.

![Paste SVG, read the preview, then clean ids and transforms before converting](./images/05-inspect-source-preview.svg)

## The page is one ID namespace

The same two files **inlined** in HTML or JSX share the page’s ID namespace and the page’s CSS. The second `clip-path="url(#clip0_12_8)"` points at the first icon’s `clipPath`. Depending on order, one glyph vanishes, both crop to the wrong rect, or a gradient from icon A fills icon B. Class names `.st0` / `.st1` from Illustrator-style sheets restyle whichever elements matched last.

That is why a designer’s “it previews fine” and a developer’s “the React component is empty” can both be true. Preview was a single document. The app is a pile of them.

`url(#clip0_12_8)` is a **document** fragment (`getElementById`), not scoped to the `<svg>`. Tree order: the first matching id wins. Nested `<svg>` does not create a scope; a shadow root or `<img>` does. Do not “fix” with wrapper `<svg>` unless you also isolate (custom element shadow, or keep `<img>`).

**What to do:** no ids unless a paint/clip is real — then prefix (`icon-chevron-grad`, not `paint0_linear_12_8`). SVGO `prefixIds` in CI. One-off: rename by hand before a second icon lands.

## IDs, titles, and styles that only explode later

**Figma id pattern.** `clip0_12_8`, `paint0_linear_12_8`, `filter0_d_12_8` — the numbers are node ids from the file. They are stable for that node and **not unique across files**. Two Figma files can emit the same `clip0_1_2`. Treat every export as colliding until prefixed.

**`<title>Layer 1</title>` / `Frame 214` / the component name.** Screen readers may announce it when the SVG is inline and not `aria-hidden`. For UI icons the glyph should be `aria-hidden="true"` (and `focusable="false"` for old IE/Edge habits) on the `<svg>`, and the **button** should have the accessible name. Delete the Figma title. Do not keep it as fake alt text. For a standalone meaningful figure (`<img>` of a diagram), write real `alt` on the image; do not hope Figma’s Frame name is a sentence.

**`<desc>`** is uncommon in Figma export and useful only if a human wrote it. Generated junk goes out with `<title>`.

**`<style>` with `.st0 { fill: #1E1E1E }`.** Harmless in an `<img>`. Hostile when two inlined SVGs share `.st0`. Prefer presentation attributes on the elements, or a single `currentColor` / CSS variable on the paths. If you must keep a stylesheet, scope it — which SVG cannot do well without unique class prefixes. Prefixing is more work than deleting `.st0`.

**Presentation attributes versus CSS.** `fill="#1E1E1E"` on the path is easy to grep. `style="fill:#1E1E1E"` is slightly worse. A class plus `<style>` is the worst for inlining. Inspection is grepping all three.

**`fill="#1E1E1E"` on every path.** The preview looks right on a white canvas and wrong in dark mode. That is a product decision sitting in the same export as `Layer_1`. UI glyphs should leave color to the component (`currentColor` or `fill="currentColor"` / `stroke="currentColor"`). Brand marks should not. Do not blindly rewrite a logo. Do not skip rewrite on a 24px nav icon because “it looks correct in Figma’s light canvas.”

SVG initial `fill` is **black**, not none. Presentation attributes have specificity 0 — a page `.st0` beats `fill="#1E1E1E"` on the element. `currentColor` is the used value of `color`. Root `fill="none"` on line icons is load-bearing; SVGO `removeUnknownsAndDefaults` treating it as redundant fills the chevron. Inspect after SVGO.

`<img>` still clips Frame `clipPath` inside its own document. Isolation is not a substitute for dropping a useless Frame clip.

## Strokes: the exporter is not the canvas

Figma’s stroke model is richer than SVG’s. The exporter approximates. When the approximation is wrong, the file looks “close” in Figma and wrong in the browser — with the path still present.

**Alignment.** Figma: center / inside / outside. SVG 1.1/`stroke` is center only (`stroke-alignment` is not interoperable). Inside/outside usually **outline to a filled path**; leftover center stroke shifts optical weight. Flush-inside 24 icons then overflow or look thin. One set: all outlined, or all center plus inset — never mixed.

**Scale stroke** (Figma diamond): visual width constant under canvas resize. SVG stroke scales with the `viewBox` → viewport CTM. `vector-effect="non-scaling-stroke"` is user-space; skip it on a 16/24/32 icon. Outline or accept scaling.

**Caps/joins** export. Miter spikes use `stroke-miterlimit` (SVG default 4). A sharp corner can exceed the viewBox and clip — overflow, not a missing `d`. Round joins for UI.

**Dashes** (`stroke-dasharray`) are user units; rhythm changes at 16 vs 24. Prefer circle glyphs for UI dots.

**Open path + `fill` not `none`:** SVG fills the chord. Figma may have shown stroke-only. Grep every path `fill`, not only the root.

## Vector networks, booleans, and fill-rule

Figma’s vector tool is a **network** (vertices with more than two edges). SVG paths are not. Export walks the network into one or more `d` attributes. A “single” shape in Figma can become three paths, or a compound path with a hole.

Boolean operations:

- **Union** should become one silhouette. If it does not, flatten before export.  
- **Subtract** becomes a hole. In SVG that is either a compound path (`d` with two subpaths) and `fill-rule="evenodd"`, or `nonzero` with opposite winding. If winding is wrong, the hole fills in.  
- **Intersect / exclude** are the usual source of leftover extra paths and evenodd surprises.

Default `fill-rule` is **`nonzero`**. `evenodd` on a triangle is boolean debris. Keep `evenodd` for a real hole (compound `d` with two subpaths). `mergePaths` concatenates `d` — it is not CSG union; wrong rule punches or fills holes. Overlapping fills without boolean paint twice; `opacity="0.5"` makes the overlap obvious.

## Text, fonts, and the reflow that looks like a clip

If the export still contains `<text>`, the browser will use a font. Figma used Inter / SF / a custom file. The browser uses whatever is installed or loaded. Metrics differ. Letters clip against the Frame `clipPath`, wrap, or overflow. This is not a `viewBox` bug.

For **icons**, outline text in Figma before export (“Outline stroke” is the wrong command; you want to flatten / outline the text object so it becomes paths). For **wordmarks**, either outline (immutable, no live text) or ship the font with the SVG and accept reflow — almost never what a toolbar wants.

`<tspan>` with absolute `x`/`y` is Figma placing each line. Fine until someone deletes a `clipPath` and the hidden overflow line becomes visible. Read the text nodes.

Missing `xmlns="http://www.w3.org/2000/svg"` on a fragment you inlined into HTML is usually fine (HTML’s SVG integration). The same fragment as a `.svg` file may not render. Inspection includes: is this a file or a snippet?

## Filters, blend modes, and the fake vector

If the file is heavy for a 24 icon, search `filter`, `feGaussianBlur`, `feOffset`, `feColorMatrix`, `feBlend`, `style="mix-blend-mode`, and `<image`.

Drop shadows in Figma become filter graphs whose **filter region** is larger than the glyph (`x="-20%" width="140%"` and similar). The painted result — and sometimes the exported `viewBox` — expands to fit the blur. The icon looks padded or soft. At 16–24px UI size, a CSS `filter: drop-shadow(...)` on the `<svg>` (or no shadow) is almost always better than shipping `feOffset` + `feGaussianBlur`. Inner shadows are worse: they rasterize poorly at small sizes.

**Layer blur** is a filter. **Background blur** on the canvas is not a real SVG `feGaussianBlur` of the page behind the icon; Figma cannot pack your dashboard into the SVG. Export either drops it or approximates with a translucent fill. If the mock depended on glass, the SVG will not.

**Blend modes** (`multiply`, `overlay`) become `style="mix-blend-mode:…"` or `feBlend`. They blend with whatever is behind the SVG in the browser, which is not Figma’s canvas. An icon that looked correct on a grey frame will look wrong on a white button. Flatten blends in Figma for UI glyphs.

“Looks like SVG, paints like PNG”: `<image href="data:image/png;base64,…">` (or `xlink:href`) inside the `<svg>`. That is a flattened bitmap — a rasterized blur, a photo fill, an imported PNG in a Frame, or an accidental screenshot. No `viewBox` hygiene will make it sharp at 2×. The file can still be 80 KB for one icon. Re-draw as vectors or export a real PNG on purpose and treat it as a raster asset.

`xlink:href` versus `href`: SVG 2 prefers `href`. Browsers still accept `xlink:href` with the namespace. If you strip `xmlns:xlink` and leave `xlink:href`, the image dies. Either keep the namespace or rewrite to `href`.

## Gradients and paints

Figma linear and radial fills become `<linearGradient>` / `<radialGradient>` in `defs`, referenced as `fill="url(#paint0_linear_12_8)"`. Inspection checklist:

- The gradient id will collide when inlined — prefix.  
- Default `gradientUnits` is **`objectBoundingBox`** (0–1 of the referencing shape). `userSpaceOnUse` shares the glyph’s user space — a leftover page `viewBox` shifts the paint when you tighten the box.  
- `gradientTransform` is a matrix; flatten with the rest.  
- `stroke="url(#…)"` dies if you rewrite stroke to `currentColor` — right for nav, wrong for a logo.

Image fills on a shape become `fill="url(#pattern…)"` or an `<image>`. That is the fake-vector case again.

## What to strip, what to keep, what to re-export

A useful cleanup is boring. It is not “minify everything.”

**Strip for UI icons**

- Unused `clipPath` / `mask` that duplicate the Frame (Clip content).  
- Wrapper groups whose only job is `translate` onto a page.  
- `id="Layer_1"`, editor `<title>` / `<desc>`, comments.  
- `width`/`height` on the root *once* you size with CSS or a `size` prop (keep them if the file must have an intrinsic size as `<img>` with no CSS).  
- Filter subtrees for shadows you will not ship.  
- `xmlns:xlink` if nothing uses `xlink:href`.  
- Internal `<style>` with generic classes.  
- `version="1.1"` and editor namespaces (`xmlns:figma`, Inkscape leftovers if the file visited another tool).  
- Twelve decimal places on path commands. Two or three are enough at 24 units; more bloats diffs.

**Keep**

- An honest `viewBox`.  
- The path data (or a simplified union of it).  
- `fill="none"` on the root when the icon is strokes.  
- Real holes (`evenodd` or a compound path you can explain).  
- Unique ids **only** if gradients or clips are actually needed.  
- `stroke-linecap` / `stroke-linejoin` that are part of the drawing.

**Re-export instead of hand-editing** when:

- The `viewBox` is page-sized and there are five nested groups.  
- Text is still `<text>` and you needed outlines.  
- Strokes should be fills for a silhouette icon (outline stroke, then union).  
- Inside/outside strokes must match the canvas and you do not want to maintain a matrix of approximations.  
- You selected the wrong Frame, a Section, or an instance-on-page.  
- Blend modes and background blur were part of the look.

Hand-flattening one chevron is fine. Hand-flattening a 40-icon set is how the next hire “fixes” `translate` in CSS. Push the work back to Figma: icon Frame, Clip content off, flatten booleans, outline text, export the component.

## SVGO: useful, not sentient

SVGO is good at numeric precision, editor namespaces, default attributes, and collapsing empty groups. It is **bad** at knowing that `clip0_12_8` was a Frame. Configure it so the **contract** survives. Do not run a blind preset on a file you have not looked at.

Plugins that regularly hurt icon exports:

| Plugin / behavior | What goes wrong |
| --- | --- |
| `removeViewBox` | `<img>` / background fall toward 300×150. |
| `cleanupIds` without prefix, then inlining | Collisions; empty icons. |
| `removeUselessDefs` before you drop the Frame clip reference | Orphan vs missing clip — order matters. |
| `mergePaths` | Wrong holes; evenodd debris. |
| `convertShapeToPath` | Fine until you wanted a `cx`/`r` to grep; usually OK. |
| `removeUnknownsAndDefaults` dropping `fill="none"` | Initial fill is black → filled line icons. |
| `convertPathData` (`floatPrecision`, `utilizeAbsolute`) | 24-grid corners go off-pixel; “soft” icon. |
| `removeHiddenElems` | Can drop `opacity="0"` debug layers you wanted — usually OK — and can drop overflow you still needed. |
| A plugin that rewrites `viewBox` from `getBBox()` **before** you drop page `translate` | Crops to the wrong box; empty preview. |

A plugin that rewrites `viewBox` from `getBBox()` is useful *after* page translate is gone and *after* you add stroke padding. Before that, it will happily crop to the clip rect or to a zero-height stroked line.

Run SVGO in CI on the **cleaned** file, not as a substitute for inspection. Fail the build on:

- `data:image/png` in UI icon folders  
- `viewBox="0 0 1440` (or any viewBox width over, say, 128) in those folders  
- `translate(` with numbers over 32 in UI icons  
- raw `clip0_` / `paint0_` ids if you inline without prefixing  
- `<text` in icon folders  
- `Layer_1`

Those greps are crude and extremely effective. They encode the recipe so inspection is not heroic.

## One shipping pass: source, preview, then CI

1. **Paste the raw export**, then paste it **twice** on one page. If the second copy dies, you have an id or `.st0` collision.  
2. **Grep** `clip0_`, `paint0_`, `<style`, `<title`, `<text`, `filter`, `<image`.  
3. **Prefix or strip ids** before a second icon lands.  
4. **Inspect after SVGO**, not only before: did `viewBox` and `fill="none"` survive?  
5. **Stop when two copies still look like the glyph**, with no generic stylesheet and no editor title.  
6. **Only then** pick size at the call site and pick an embedding method.

The editor on [getsvgeditor.com](https://getsvgeditor.com): source beside preview, nothing uploaded. Paste **twice** — that is the id test. React/PNG tabs come after. SVGR camelCases `clip-path` → `clipPath`; converting a Frame clip you should have deleted still yields an empty component. [SVG → React](https://getsvgeditor.com/svg-to-react) / [SVG → PNG](https://getsvgeditor.com/svg-to-png) need an honest `viewBox`. Empty as a **single** file → [scene graph](https://getsvgeditor.com/blog/figma-svg-empty-shifted-clipped), not `prefixIds`.

Size: [viewBox vs CSS](https://getsvgeditor.com/blog/svg-viewbox-vs-width-height-css). Embed: [inline vs img vs sprite](https://getsvgeditor.com/blog/inline-svg-vs-img-vs-background-vs-sprite).

## Shipping audit (two copies on one page)

1. Second paste dies or restyles the first → id / `.st0`.  
2. `clip0_` / `paint0_` unprefixed while you inline.  
3. `<title>Layer` / `Frame 214` still in the tree.  
4. `<style>` with generic classes.  
5. UI glyph still `#1E1E1E` / `#111` (not a logo).  
6. `<text>` that needed outlining; `<image>` / `filter` on a 24 icon.  
7. SVGO already ran: `fill="none"` and `viewBox` still there.  
8. `fill-rule="evenodd"` on a shape with no hole.

Items about empty/shifted **as one file** belong in the [scene article](https://getsvgeditor.com/blog/figma-svg-empty-shifted-clipped). Do not start `currentColor` until that pass is green.

## Agree on an export recipe, not a heroic cleanup

Inspection does not scale if every icon is a unique archaeology project. Write the recipe once so “looks broken” means the same five minutes for everyone.

| Step | Decision |
| --- | --- |
| **In Figma** | A dedicated 20 or 24 **component Frame**, not the page. Clip content **off** unless you need a real crop. Boolean union until one silhouette. Outline text. Outline stroke only when the system wants fills, and then for the whole set. Center strokes with optical padding, or all outlined — do not mix. Export the main component, not an instance on a mock. SVG at 1×; do not use 2× as a sharpness trick. |
| **In the file** | Honest `viewBox`, no Frame clip, no page `translate`, no `Layer_1`, no generic CSS classes, prefixed ids only when paints are real. |
| **In CI** | SVGO with `viewBox` and `fill="none"` preserved; prefix ids if you inline; fail on leftover `clip0_`, huge viewBoxes, embedded PNG, `<text` in icon folders. |
| **In the app** | Size at the call site. Embed by job. Color via `currentColor` for UI glyphs. Do not compensate transforms in CSS. |

Untrusted uploads: sanitize, preview as `<img>` or a sandbox — never raw markup in a privileged DOM. You trust the designer’s file, not its leftover ids.

The exporter will still emit colliding `clip0_` and a center-stroke approximation. Treat “broke in React” as **paste it twice**, not as a JSX bug. When ids are unique and `fill="none"` survived SVGO, the path was almost always fine.