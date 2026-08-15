---
title: "Figma SVG Looks Empty, Shifted, or Clipped: Inspect the Scene"
slug: figma-svg-empty-shifted-clipped
description: "Figma SVG path data is usually fine. Empty, shifted, or clipped icons come from Clip content, nested translate, and an artboard viewBox. Inspect source next to preview."
---

# Figma SVG Looks Empty, Shifted, or Clipped: Inspect the Scene

You copy an icon from Figma. The path looks complete in the layers panel. In the browser it is gone, stuck in the wrong corner, or chewing its own stroke. The ticket says “broken SVG.” The file is almost never missing geometry. Figma exported a **scene graph** — frames, clips, and page-space transforms — and the browser is drawing that scene faithfully.

Figma’s canvas is a layout engine. SVG is a document: user space defined by `viewBox`, then a clip stack, then a viewport. The exporter serializes *how the canvas composed the layer*, not the 24px glyph you meant. CSS cannot invent a crop. Neither can React. CamelCasing `clip-path` still paints an empty box.

**Short version:** if the `d` attribute is there, stop restyling the component. Open the XML. Empty usually means `clipPath` or `mask`. Shifted usually means nested `transform`. A postage stamp in a white field usually means you exported a Frame the size of the artboard. Hairlines that vanish at the box edge are often `overflow: hidden` plus a Frame clip, not a thin `stroke-width`. A file that looks perfect as `<img src>` and dies when inlined is a different failure — duplicate `id`s. That is the [cleanup and inlining pass](https://getsvgeditor.com/blog/figma-svg-inline-ids-svgo), not this article.

Size tokens and embedding methods only make sense **after** the file is honest. This piece is the first inspection: does the document still describe a page?

![The path data is fine; clipPath, translate, and viewBox from Figma make the preview look broken](./images/01-path-fine-scene-lies.svg)

## Figma does not export an icon. It exports a scene

On the canvas you selected a 24×24 component. The SVG you downloaded is closer to a snapshot of how Figma *composed* that component: a Frame, maybe a clip matching the Frame, groups that remember where the instance sat on the page, `defs` for filters you cannot see at 24px, and a root `viewBox` that still thinks in artboard coordinates.

A typical “simple” chevron arrives looking like this (ids shortened):

```xml
<svg width="1440" height="900" viewBox="0 0 1440 900" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g clip-path="url(#clip0_12_8)">
    <g transform="translate(712 438)">
      <path d="M4 2L12 10L4 18" stroke="#1E1E1E" stroke-width="2"/>
    </g>
  </g>
  <defs>
    <clipPath id="clip0_12_8">
      <rect width="1440" height="900" fill="white"/>
    </clipPath>
  </defs>
</svg>
```

Nothing in that file is “corrupt.” The path is a chevron. The rest is **where Figma thought the chevron lived**. Drop this into a 24px button and you get a blank or a speck. Drop it into a hero and you get a tiny mark in the middle of a billboard. The renderer did what the document asked.

Illustrator and Sketch emit the same kind of file (clip groups, artboards, `Layer_1`). The examples use Figma’s names because that is the usual source.

### Copy as SVG versus Export versus a plugin

These are not the same pipeline, and they do not fail the same way.

**Copy as SVG** (right-click, or the shortcut) serializes the current selection as Figma currently draws it. If the selection is an instance sitting on a page, you often get page-space `translate` and a `viewBox` that still belongs to a parent Frame. If the selection is the 24×24 component Frame itself, Copy is frequently the cleanest of the three — until “Clip content” is on, in which case you still get a `clipPath`.

**Export → SVG** on the right sidebar uses the export settings on that layer: suffix, constraint (`1x` / `2x` / `3x`), and whether ids are included. SVG does not get sharper at 2×. A `2x` SVG export is still vectors; what sometimes changes is whether Figma bakes a larger `width`/`height` onto the root. Teams then rasterize *that* number and wonder why the PNG is 48px when the button is 24. That is a size-contract problem, not the subject of this article — but the exporter put the number there because someone asked for 2× on a vector format.

**Plugins** (SVGO-in-Figma, icon-pipeline plugins, “flatten to outline”) can be better or worse than native export. A plugin that outlines strokes and unions boolean shapes before serializing will save you a week. A plugin that runs a default SVGO preset including `removeViewBox` will give you the 300×150 banner the next article in this series already covers. Inspect the plugin output the same way you inspect native export. Do not trust a checkbox labeled “optimize.”

### Frames, groups, sections, and “Clip content”

Figma’s layer types do not map 1:1 onto SVG.

A **Group** is a selection convenience. Export often becomes a `<g>` with no clip and no extra box. Groups are the least surprising parent.

A **Frame** is a layout box. It has a size, optional Auto Layout, optional fill, and a **Clip content** toggle. When Clip content is on (the default for many UI frames), children that stick out are hidden on the canvas — and the exporter writes `clip-path="url(#clip0_…)"` with a `<rect>` the size of the Frame. That clip is not an artistic mask. It is the Frame’s `overflow: hidden`. For a 24px icon whose strokes sit on the pixel grid, that is usually why hairlines vanish.

A **Section** (the pink organizer) is not a design component. Exporting a section is how you accidentally ship `viewBox="0 0 2400 1600"` with twelve icons inside it.

**Auto Layout** frames add padding as empty space in the box, not as path inset. Export the Auto Layout parent and the `viewBox` includes padding; the glyph looks small. Export the vector child and you may lose the intended 24 grid. The design-system recipe is: a dedicated **icon Frame** at 20 or 24 whose only job is the glyph, Clip content off unless you truly need it, vectors inset by 2 units if strokes need optical padding.

**Absolute position** children inside Auto Layout are the other surprise. On the canvas they sit where you dropped them. In SVG they often become an extra `translate` on a nested group, while siblings stay in flow coordinates. The file matches Figma and looks surprising in a button.

### Component instance versus main component

Exporting an **instance** sometimes includes the instance’s position on the current page. Exporting the **main component** (the source in the components page) more often yields a box that matches the component Frame. If a designer “just copied the icon from the screen we were looking at,” you are looking at instance-on-page coordinates. Ask for export from the component file, or enter the instance and copy only when the selection is the 24 Frame, not the card that contains it.

Variants (Default / Hover / Open) are separate documents. Hover that uses a thicker stroke is a different path, not a CSS `:hover` you get for free from one SVG. Inspect the variant you actually exported; do not assume the default file contains the hover geometry.

### What to treat as ink versus machinery

| In the file | Treat as |
| --- | --- |
| `path` / `circle` / `rect` / `line` that *draw* the glyph | Ink. Keep, then simplify. |
| Root `viewBox` that matches the ink bounds (plus optical pad) | Contract. Keep or recompute once. |
| `g transform="translate…"` that only recenters a Frame on a page | Machinery. Flatten or drop. |
| `clipPath` that is a copy of the Frame rect | Machinery. Drop if the glyph is fully inside. |
| `mask` from opacity tricks or boolean leftovers | Usually machinery for UI glyphs. |
| `filter` / blur / drop shadow at icon size | Usually machinery. Rarely worth the `fe*` subtree. |
| `linearGradient` / `radialGradient` that the icon actually uses | Ink, but ids must be unique if you inline. |
| `<image href="data:image/png…">` | Not a vector. Wrong format or a flattened effect. |
| `<text>` / `<tspan>` | Ink only if you ship the font; otherwise outline or it will reflow. |
| `id="Layer_1"`, `<title>Frame 214` | Editor labels. Strip for UI icons. |
| `fill="#1E1E1E"` on a UI glyph | Product decision, sitting in the same export. Rewrite for theming; leave brand marks. |

If you skip this split, theming and `size` debates are noise: you are still arguing about a page, not a glyph. Illustrations may keep a real clip; wordmarks may keep hex. This article assumes a **UI icon** that behaved like a page.

## Match the preview to the node

Do not start with “make `stroke-width` 1.5.” Look at what the file *does* on screen, then search the markup for the node that would cause that picture. The preview names the bug. The source names the node. You need both at once.

![Empty, shifted, clipped, and tiny-in-a-field SVG previews map to clipPath, translate, overflow, and artboard viewBox](./images/02-symptom-map.svg)

| What you see | Look for first | Do not start with |
| --- | --- | --- |
| Completely empty | `clip-path`, `mask`, path numbers far outside `viewBox` | `fill`, theme tokens |
| Visible, but in a corner or off-center | `transform="translate("` / `matrix(` on a wrapper `<g>` | CSS `margin` / `left` |
| Shape there, strokes eaten on the edge | Frame `clipPath`, root overflow | Thicker `stroke-width` |
| Correct drawing, huge empty padding | `viewBox` is the Frame or page; ink is a small `getBBox()` | Scaling the component up |
| Soft, photographic | `<image href="data:image/png">` or `filter` | PNG compression |
| One bar of a plus/minus missing | Stroke flush to `x=0`/`24` + `overflow` | Deleting the path |

Paste source and preview together ([SVG Editor](https://getsvgeditor.com) if you have no local file). Search `clip-path`, `mask=`, `translate`, `matrix(`. DevTools last: `$0.getAttribute('viewBox')`, `$0.querySelector('path').getBBox()`. No hit target usually means the clip already ate pointer-events. Inline-only breakage belongs in [ids / SVGO](https://getsvgeditor.com/blog/figma-svg-inline-ids-svgo), not this pass.

## Empty: the clip is usually the Frame

Figma Frames clip their children when **Clip content** is enabled. Export keeps that as:

```xml
<g clip-path="url(#clip0_12_8)">
  <!-- glyph -->
</g>
<defs>
  <clipPath id="clip0_12_8">
    <rect width="24" height="24" fill="white"/>
  </clipPath>
</defs>
```

Two facts about SVG clips are easy to mix up with masks.

**`clipPath` (default `clipPathUnits="userSpaceOnUse"`) uses the fill geometry of its children, not their stroke and not `fill` color.** Figma’s `fill="white"` on the clip rect is convention. Ignore it.

**`mask` uses luminance (`mask-type: luminance`: white reveals) or alpha.** Layer opacity and some booleans become a mask. Delete it and the glyph “fills in”: that was a knockout, not a missing `d`.

SVG strokes are centered (`stroke-alignment` is not in browsers). A 1.5-unit stroke on `x=24` puts 0.75 units outside the Frame. The clip throws that half away. So does the HTML user-agent rule `svg:not(:root) { overflow: hidden }` — **inline** SVG clips; a standalone `.svg` root is often `overflow: visible`, which is why the file “looks fine” and JSX eats the hairline. Teams then bump `stroke-width` or nudge 0.5px on one icon forever.

![A Frame clipPath eats strokes on the box edge; removing the unused clip shows the same path with a full stroke](./images/04-frame-clip.svg)

**What to do:** if the clip rect equals the `viewBox` and you are not actually masking a picture-in-picture, delete the `clip-path` attribute and the unused `clipPath` in `defs`. If a stroke still kisses the boundary, inset the drawing (optical padding on the 24 grid — Material and most icon sets already do this) or set `overflow="visible"` on the root **on purpose**. Do not widen the stroke until you have ruled out clipping.

Nested clips happen when an icon Frame sits in a card Frame and someone exported the card, or when a vector inside a 16×16 nested Frame is itself clipped. You will see `clip-path` on more than one `<g>`. Delete from the outside in, with the preview visible. The first deletion that restores the glyph is the Frame. The inner one might be load-bearing (a circular crop on an avatar). Stop when you can name why each remaining clip exists.

`clipPathUnits="objectBoundingBox"` is rare in Figma output and common in hand-written SVG. If you ever see it, the clip rect’s `width="1"` means 100% of the referencing element, not 1 user unit. Do not delete that clip thinking the numbers look tiny.

`clip-rule` (`nonzero` / `evenodd`) on a clipPath child can punch holes in the clip itself. If a “simple” Frame clip is a path instead of a rect, boolean leftovers may have landed in the clip, not in the glyph. Look at the clipPath contents, not only at the drawing.

## Shifted: nested translate is page coordinates

Figma remembers where the instance sat. Export wraps the ink:

```xml
<svg viewBox="0 0 1440 900" width="1440" height="900">
  <g transform="translate(712.0001 438.499)">
    <path d="M0 0h16v16H0z"/>
  </g>
</svg>
```

The path numbers are local (`0…16`). The **document** is still the desktop. Inline that in a button and the 16×16 square is translated into the middle of a 1440-wide coordinate system — or clipped away entirely if a Frame clip is also present. Fractional translates (`438.499`) are Figma’s subpixel layout, not a corrupt file. They still have to go.

![Artboard viewBox plus translate versus a flattened 24 viewBox that matches the ink](./images/03-nested-translate.svg)

**What to do:** flatten once. Either:

1. Re-export the **component / 24 Frame**, not a selection sitting on a page, or  
2. Bake the translate into the path (flatten in Figma, or a one-off rewrite), then set `viewBox` to the real ink bounds.

Do not compensate in CSS with `transform: translate(-712px, …)` or `left: -712px`. That number will be different on the next icon, and it will fight `viewBox` scaling.

### Reading `matrix`, rotation, and flips

Nested `matrix(a b c d e f)` is the same story with rotation or scale baked in. In SVG the matrix maps user coordinates: `x' = ax + cy + e`, `y' = bx + dy + f`.

| Matrix | Meaning |
| --- | --- |
| `1 0 0 1 tx ty` | Pure translate. Flatten or drop. |
| `-1 0 0 1 tx ty` or `1 0 0 -1 …` | Flip on an axis. Keep the flip in the path or keep a single group, but do not also keep a page translate. |
| `0 -1 1 0 tx ty` (and cousins) | 90° rotation. Either outline in Figma so the path is already rotated, or keep one `transform` on the glyph group — not three nested ones. |
| `sx 0 0 sy 0 0` with `sx` ≠ 1 | Scale that belongs in CSS, not in the icon, unless the drawing is deliberately non-uniform. |

Concatenated functions (`translate() rotate()`) apply **right-to-left**; nested `<g>` elements apply inside-out (innermost first). Figma encodes rotate-about-center as translate → rotate → translate. That is valid SVG; flatten in the file so `d` already sits in icon space.

CSS `transform-origin` does not rewrite the SVG `transform` attribute. `getCTM()` is user space → viewport; `getScreenCTM()` includes CSS transforms on ancestors. A large CTM translate after you “tightened `viewBox`” means the glyph is still in page space — and the new viewBox cropped it. That is the empty preview after a fake cleanup.

## Tiny in a field: you exported the artboard

`viewBox="0 0 1440 900"` with a 24px glyph is not a sizing bug in the CSS sense. The contract says the drawing is a whole page. The browser is correct. Tightening `width`/`height` on the `<svg>` just scales the **page**, so the icon stays a speck. `preserveAspectRatio="xMidYMid meet"` will letterbox it politely. `none` will stretch the empty page. Neither gets you the chevron.

How it happens: Export is on a parent Frame; Copy as SVG from a selection that still inherits the parent box; a Section selected; or a component instance copied from a dashboard mock with the parent still in the selection stack.

How you confirm: compare `viewBox` to the ink.

```js
const svg = document.querySelector("svg");
const ink = svg.querySelector("path, circle, rect, line, polyline, polygon");
console.log("viewBox", svg.getAttribute("viewBox"));
console.log("getBBox", ink.getBBox());
console.log("svg getBBox", svg.getBBox());
```

If `ink.getBBox()` is roughly `16×16` and `viewBox` is `1440×900`, the file is lying about what it is. `svg.getBBox()` can be the large box if a clip rect or empty group still spans the Frame — another reason to read the XML, not only the numbers.

Default `getBBox()` is the **object** bounding box: geometry only — no stroke, markers, clip, mask, or filter (SVG 2: `getBBox({ stroke: true })` where implemented). A line `(2,12)→(22,12)` with `stroke-width="2"` reports height 0. Rebuild `viewBox` from that and you clip the stroke; add half stroke plus miter spike (`stroke-miterlimit`). Filter regions (`x="-10%" width="120%"`) paint larger than bbox.

Recompute from ink (padded) or re-export the icon Frame. Tight box vs CSS is [viewBox vs width/height vs CSS](https://getsvgeditor.com/blog/svg-viewbox-vs-width-height-css) — but not while the contract still describes a desktop.

No `viewBox`: `<img>` and `background-image` are replaced elements. HTML’s fallback intrinsic size is **300×150**. The glyph was not empty; the size mapping was gone. Restore `viewBox` from ink. Do not invent `0 0 24 24` if bbox is `0 0 20 20`.

## Clipped hairline: overflow, not stroke

Same geometry, different host: standalone SVG `overflow` on `:root` is typically visible; **inline** `svg:not(:root)` is `hidden` in the HTML user-agent stylesheet. A 1.5 stroke on `x=0`/`24` is half outside the viewport box. Check, in order:

1. Frame `clipPath` (Clip content).  
2. Nested clip on an inner Frame.  
3. Computed `overflow` on the `<svg>` (inline vs file).  
4. Figma inside/outside stroke (SVG has center only — see the inlining article).  
5. `shape-rendering="crispEdges"` — snaps to the pixel grid; hairlines vanish on odd device pixels.

Optical padding inside a 24 grid is cheaper than a special-case stroke on one icon. A practical grid: glyph in 20×20, centered in 24, so a 2-unit stroke never sits on the root edge. If the set is 16px (small toolbar), the pad is 1.5–2 and the stroke is 1.25–1.5 in user units — still not flush to 0.

`overflow="visible"` is a valid fix when the set is already drawn to the box and you cannot re-pad eighty icons this week. It is a product decision: visible overflow can paint into adjacent buttons. Prefer padding when you can.

## Export settings, hidden layers, and why Figma’s preview is not the browser

The right-sidebar **Export** panel has a few checkboxes that change the document you inspect. They are easy to miss because the canvas looks the same.

**Include “id” attribute.** On: you get `id="Layer_1"` / node-based ids on groups and sometimes on paths. Off: fewer collisions, harder to tell which group was which Frame. For a design-system icon that will be inlined, ids off (or prefix in CI) is the default worth arguing for. For an illustration you will only ever load as `<img>`, ids are mostly harmless.

**Outline text.** On: `<text>` becomes paths. File size goes up; reflow goes away; the mark is immutable. For icons and wordmarks in UI, this should be on. For a diagram that must remain selectable text, leave it off and ship fonts — that is not an icon pipeline.

There is no “outline stroke” checkbox in the export panel. Stroke outlining is a canvas operation you do *before* export. If the team wanted filled silhouettes and you only ticked export settings, you still have strokes.

**Suffix and scale (`1x` / `2x` / `@2x`).** For PNG this is density. For SVG it is a misunderstanding. You may still get `width="48"` on a 24-grid drawing. That number is a default presentation size, not extra resolution. If someone then rasterizes using that 48, they are not wrong — they followed the attribute. The fix is a 1× SVG with an honest `viewBox`, then rasterize at CSS size × DPR when you actually need a bitmap.

Hidden layers (`visible: false` in Figma) sometimes still serialize, sometimes not, depending on Copy versus Export and whether the layer is inside the selected Frame. An `opacity="0"` path is worse: it is in the file, it can expand `getBBox()`, it can still be announced if it has a title, and `removeHiddenElems` may or may not drop it. If the preview looks empty but `d` is huge, search `opacity="0"` and `display="none"`. Designers leave “old” vectors under the new ones.

**Constraints** (left/right/top/bottom/scale) affect how a child resizes on the canvas. They do not have a 1:1 SVG attribute. What you get is whatever transform and box Figma computed *at export time*. A child with “scale” constraints, exported from a stretched instance, is a differently proportioned path than the main component. Inspect the instance you actually shipped, not the library thumbnail.

**Why Figma looks right.** Figma’s renderer is not a browser. Inside/outside strokes, background blur, missing fonts (Figma has the file), blend modes against the grey canvas, and Clip content as a *canvas* feature all look correct there. The SVG is a lossy projection into a different engine. “But it looks fine in Figma” is not evidence that the XML is an icon document. It is evidence that the canvas scene is fine. Those are different artifacts. Paste the XML into a browser preview; that is the only preview that counts for shipping.

## Two more exports you will actually see

The chevron at the top is the page-translate case. These two show up just as often.

### Auto Layout padding mistaken for a 24 grid

A designer puts a 16×16 vector in an Auto Layout Frame with 4 padding and a 24×24 min size. The canvas shows a correct icon button. Export of that Frame:

```xml
<svg viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g transform="translate(4 4)">
    <path d="M0 0h16v16H0z" stroke="#1E1E1E"/>
  </g>
</svg>
```

This one is *almost* honest: `viewBox` is 24, the glyph is 16, padding is a translate. For a **button** that includes padding, keep it — but then the component’s `size={24}` is the whole control, not the glyph, and you will double-pad if the CSS button also has padding. For an **icon set**, you want the vector on the 24 grid with optical inset in the path, not a layout translate. Re-export the vector Frame, or flatten the 4,4 into the `d` and decide whether `viewBox` stays 24 (with padding inside) or becomes `0 0 16 16` (CSS provides the rest). Mixing both conventions in one sprite is how 16 and 24 icons end up looking like different optical weights.

### Nested Frame with Clip content on the inner 16

```xml
<svg viewBox="0 0 24 24" fill="none">
  <g clip-path="url(#clip0_1_1)">
    <g transform="translate(4 4)">
      <g clip-path="url(#clip1_1_1)">
        <path d="M1 8h14" stroke="#1E1E1E" stroke-width="2"/>
      </g>
    </g>
  </g>
  <defs>
    <clipPath id="clip0_1_1"><rect width="24" height="24"/></clipPath>
    <clipPath id="clip1_1_1"><rect width="16" height="16"/></clipPath>
  </defs>
</svg>
```

The line `y=8` with stroke 2 in the inner 16 box is fine. If the path were `y=0`, the inner clip eats it. People delete the outer clip, the icon still looks broken, and they stop. Delete **both** Frame clips if neither is a real crop, then pad. Two clips is not “more correct.” It is two Frames with Clip content on.

When you inline two such files, `clip0_1_1` collides even if you “already removed one clip” in the first icon and not the second. Prefix or strip ids after the clips are gone so empty `defs` can be dropped.

## Inspect the scene, then stop

![Paste SVG, read the preview, fix clip and transform once before sizing or converting](./images/05-inspect-source-preview.svg)

1. **Paste the raw export.** XML, not JSX. `clip-path` is easier to grep than `clipPath={}`.
2. **Name the picture.** Empty / shifted / clipped / padded. Use the table above.
3. **Delete one node at a time** with the preview visible. Restored after dropping `clip-path`? That was the Frame. Jumped into the box after removing `translate`? Flatten that group.
4. **Compare `viewBox` to `getBBox()`.** An order of magnitude off means you still have a page. Off by a stroke width means pad the viewBox (stroke is not in bbox).
5. **Stop when the preview is the glyph**, on a tight `viewBox`, with no Frame clip.

The editor on [getsvgeditor.com](https://getsvgeditor.com) is built for that loop: source on one side, preview on the other, nothing uploaded. You are not converting yet. You are asking whether the document still describes a page.

Cleaned-up chevron from the start of this article:

```xml
<!-- after: icon space, no Frame clip, no page translate -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" aria-hidden="true">
  <path d="M8 6L16 12L8 18" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
</svg>
```

The `d` values changed because flatten moved the chevron from local `4…18` inside `translate(712 438)` onto a 24 grid. That rewrite belongs in Figma (or one careful edit), not at every place the icon is used.

### Diagnosis audit

Before anyone “fixes” stroke in CSS:

1. Does `viewBox` describe the **glyph**, or a Frame / page / Section?
2. Is Clip content sitting in the file as `clip-path` / `mask` whose rect equals the Frame?
3. Is there `translate` / `matrix` that only stores page position?
4. Does `getBBox()` roughly match `viewBox` (stroke is not in bbox)?
5. Are strokes flush to the box edge (overflow / clip)?
6. Did you export the icon Frame — or a selection on the desktop?
7. Is Figma’s canvas the thing that “looks right,” while the XML still describes a page?

If 1–4 fail, do not discuss `currentColor`, SVGO presets, or React props yet.

When the scene is gone, the next failures are collisions, leftover `.st0`, inside/outside strokes, and a blind SVGO run. That is a separate pass: [Figma SVG that breaks only when inlined](https://getsvgeditor.com/blog/figma-svg-inline-ids-svgo). On-screen size is [viewBox vs width/height vs CSS](https://getsvgeditor.com/blog/svg-viewbox-vs-width-height-css). How it lands on the page is [inline vs img vs background vs sprite](https://getsvgeditor.com/blog/inline-svg-vs-img-vs-background-vs-sprite). Convert only after the file is honest — [SVG → React](https://getsvgeditor.com/svg-to-react) and [SVG → PNG](https://getsvgeditor.com/svg-to-png) will not invent a crop that is not in the markup.

Treat “looks broken” as a cue to **read the XML next to a preview**, not as a CSS mystery and not as a reason to thicken the stroke. Once the scene is gone, the path was almost always fine.
