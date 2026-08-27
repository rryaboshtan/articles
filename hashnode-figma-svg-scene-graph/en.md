---
title: 'Why Figma SVG Icons Disappear in the Browser'
slug: figma-svg-empty-shifted-clipped
description: 'A field note on the Figma SVG exports that look empty, land in a corner, or lose their strokes in the browser.'
---

# Why Figma SVG Icons Disappear in the Browser

This usually starts with a perfectly ordinary message: “Can you export this icon from Figma?” Someone copies it, opens the file in a browser, and gets nothing. Or a tiny mark in the middle of a huge white rectangle. Or a line with one end missing.

Then we blame CSS. I have done that too. The path is often fine. The awkward part is everything Figma wrapped around it: a Frame, a clip, a transform that remembers the icon's position on the page, and a `viewBox` that belongs to the artboard rather than the glyph. I normally keep the XML and a browser preview next to each other; [getsvgeditor.com](https://getsvgeditor.com) works fine for this.

Figma is a layout tool. SVG is a document with its own coordinate system. `viewBox` sets that system, clips cut it down, and only then does the browser put it in the viewport. The exporter records how the layer was assembled on the canvas. It does not know that you wanted a portable 24px glyph.

That difference explains most of the “broken SVG” tickets I see. CSS cannot repair a wrong coordinate system. React cannot repair it either. Renaming `clip-path` to `clipPath` is necessary in JSX, but it does not make an empty clip useful.

My first rule is simple: if the `d` attribute exists, do not start by restyling the component. Read the XML. Empty usually means `clipPath` or `mask`; a shifted icon usually has a nested `transform`; the postage stamp usually lives in an artboard-sized Frame. A missing hairline is often `overflow: hidden`, not a stroke that needs to be thicker. If the file works as `<img src>` but fails when inlined, check duplicate `id`s.

Forget size tokens for a minute. The useful question is less impressive: does this document describe an icon, or is it still describing the page it came from?

![The path data is fine; clipPath, translate, and viewBox from Figma make the preview look broken](./images/01-path-fine-scene-lies.svg)

## The file contains more than the drawing

On the canvas you selected a 24×24 component. The SVG you downloaded is closer to a snapshot of how Figma _composed_ that component: a Frame, maybe a clip matching the Frame, groups that remember where the instance sat on the page, `defs` for filters you cannot see at 24px, and a root `viewBox` that still thinks in artboard coordinates.

Here is the kind of “simple” chevron that turns up in a code review (ids
shortened):

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

Nothing in that file is corrupt. The path is a chevron. The rest says where Figma thought the chevron lived. Put it in a 24px button and you get a blank or a speck. Put it in a hero and you get a tiny mark in the middle of a billboard. The renderer is not being clever; it is following the document.

Illustrator and Sketch emit the same kind of file (clip groups, artboards, `Layer_1`). The examples use Figma’s names because that is the usual source.

### Three ways to get the same-looking file

The three export paths are easy to treat as interchangeable. They are not.

**Copy as SVG** (right-click, or the shortcut) serializes the current selection as Figma currently draws it. If the selection is an instance sitting on a page, you often get page-space `translate` and a `viewBox` that still belongs to a parent Frame. If the selection is the 24×24 component Frame itself, Copy is frequently the cleanest of the three — until “Clip content” is on, in which case you still get a `clipPath`.

**Export → SVG** on the right sidebar uses the export settings on that layer: suffix, constraint (`1x` / `2x` / `3x`), and whether ids are included. SVG does not get sharper at 2×. A `2x` SVG export is still vectors; what sometimes changes is whether Figma bakes a larger `width`/`height` onto the root. Teams then rasterize _that_ number and wonder why the PNG is 48px when the button is 24. That is a size-contract problem, not the subject of this article — but the exporter put the number there because someone asked for 2× on a vector format.

**Plugins** (SVGO-in-Figma, icon-pipeline plugins, “flatten to outline”) can be better or worse than native export. A plugin that outlines strokes and unions boolean shapes before serializing will save you a week. A plugin that runs a default SVGO preset including `removeViewBox` will give you the 300×150 banner. Inspect the plugin output the same way you inspect native export. Do not trust a checkbox labeled “optimize.”

### The Frame is usually the culprit

Figma's layer types do not map cleanly onto SVG. This is where the extra markup usually comes from.

A **Group** is a selection convenience. Export often becomes a `<g>` with no clip and no extra box. Groups are the least surprising parent.

A **Frame** is a layout box. It has a size, optional Auto Layout, optional fill, and a **Clip content** toggle. When Clip content is on (the default for many UI frames), children that stick out are hidden on the canvas — and the exporter writes `clip-path="url(#clip0_…)"` with a `<rect>` the size of the Frame. That clip is not an artistic mask. It is the Frame’s `overflow: hidden`. For a 24px icon whose strokes sit on the pixel grid, that is usually why hairlines vanish.

A **Section** (the pink organizer) is not a design component. Exporting a section is how you accidentally ship `viewBox="0 0 2400 1600"` with twelve icons inside it.

**Auto Layout** frames add padding as empty space in the box, not as path inset. Export the parent and the `viewBox` includes that padding, so the glyph looks small. Export the vector child and you may lose the intended 24 grid. For icons I tend to use a separate 20 or 24px Frame whose only job is holding the glyph. Clip content stays off unless the crop is intentional. An inset of two units is enough for many strokes; the exact number is less important than not drawing on the edge by accident.

**Absolute position** children inside Auto Layout are the other surprise. On the canvas they sit where you dropped them. In SVG they often become an extra `translate` on a nested group, while siblings stay in flow coordinates. The file matches Figma and looks surprising in a button.

### Which component did you actually export?

Exporting an **instance** sometimes includes the instance’s position on the current page. Exporting the **main component** (the source in the components page) more often yields a box that matches the component Frame. If a designer “just copied the icon from the screen we were looking at,” you are looking at instance-on-page coordinates. Ask for export from the component file, or enter the instance and copy only when the selection is the 24 Frame, not the card that contains it.

Variants (Default / Hover / Open) are separate documents. Hover that uses a thicker stroke is a different path, not a CSS `:hover` you get for free from one SVG. Inspect the variant you actually exported; do not assume the default file contains the hover geometry.

### Keep the ink; question the wrappers

The `path`, `circle`, `rect`, and `line` elements that make up the glyph are the part worth keeping. A root `viewBox` that matches those shapes, with a little optical padding, is the useful size contract.

Most of the surrounding structure is negotiable. A `translate` that only puts a Frame back where it was on the page can be flattened. So can a `clipPath` that is just a copy of the Frame rectangle, provided the icon does not need that crop. UI icons rarely need an opacity `mask` or a blur filter either. I remove those when they are leftovers rather than effects the design actually relies on.

There are a few things not to remove blindly. A gradient may be part of the artwork, and its id must be unique when several SVGs are inlined. An embedded `<image>` is a raster image, not a vector version of it. Text needs the font to remain stable; otherwise outline it. Names such as `Layer_1` and `Frame 214` are editor labels, so I usually strip them from UI icons. A hard-coded `fill="#1E1E1E"` is a product choice, not geometry: replace it for theming, but leave brand marks alone.

This distinction matters because otherwise a discussion about theming or component size starts before anyone has established what the file contains. The advice here is for a **UI icon** that accidentally behaved like a page.

## Start with the symptom, then find its node

Do not start with “make `stroke-width` 1.5.” Look at what the file _does_ on screen, then search the markup for the node that could cause that picture. The preview tells you which symptom to investigate; the source tells you where it comes from.

![Empty, shifted, clipped, and tiny-in-a-field SVG previews map to clipPath, translate, overflow, and artboard viewBox](./images/02-symptom-map.svg)

For a completely empty preview, search for `clip-path`, `mask`, and path coordinates that sit outside `viewBox`; changing `fill` will not help. If the shape is visible but sits in a corner, inspect `translate` and `matrix` on the wrapper before touching CSS positioning. When a stroke disappears at the edge, check the Frame clip and root overflow before making the stroke thicker.

Large empty margins usually mean that `viewBox` describes the Frame or the page while `getBBox()` describes a small glyph. A soft, photographic result usually means `<image href="data:image/png">` or a filter has entered the export. If one bar of a plus or minus is missing, look for a stroke sitting on `x=0` or `x=24` together with overflow clipping. Deleting the path would only hide the symptom.

Paste source and preview together ([SVG Editor](https://getsvgeditor.com) if
you have no local file). I search `clip-path`, `mask=`, `translate`, and
`matrix(` before opening DevTools. If I do open it, these are the two values I
care about:

```js
$0.getAttribute('viewBox')
$0.querySelector('path').getBBox()
```

No hit target often means the clip has already eaten the pointer events. A
problem that appears only after inlining belongs to the IDs/SVGO pass, not this
one.

## When the preview is empty, look for a clip

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

If the clip rect equals the `viewBox` and you are not actually masking a picture-in-picture, delete the `clip-path` attribute and the unused `clipPath` in `defs`. If a stroke still kisses the boundary, inset the drawing. Setting `overflow="visible"` on the root is also possible, but do it knowingly: the stroke can then paint into a neighbouring button. Widening the stroke is not the fix for a clip.

Nested clips happen when an icon Frame sits in a card Frame and someone exported the card, or when a vector inside a 16×16 nested Frame is itself clipped. You will see `clip-path` on more than one `<g>`. Delete from the outside in, with the preview visible. The first deletion that restores the glyph is the Frame. The inner one might be load-bearing (a circular crop on an avatar). Stop when you can name why each remaining clip exists.

`clipPathUnits="objectBoundingBox"` is rare in Figma output and common in hand-written SVG. If you ever see it, the clip rect’s `width="1"` means 100% of the referencing element, not 1 user unit. Do not delete that clip thinking the numbers look tiny.

`clip-rule` (`nonzero` / `evenodd`) on a clipPath child can punch holes in the clip itself. If a “simple” Frame clip is a path instead of a rect, boolean leftovers may have landed in the clip, not in the glyph. Look at the clipPath contents, not only at the drawing.

## When the icon moves, check the coordinate system

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

The clean fix is to flatten this once. Re-export the **component / 24 Frame** instead of a selection sitting on a page, or bake the translate into the path (flatten in Figma or do a one-off rewrite) and set `viewBox` to the real ink bounds.

Do not compensate in CSS with `transform: translate(-712px, …)` or `left: -712px`. That number will be different on the next icon, and it will fight `viewBox` scaling.

### Reading `matrix`, rotation, and flips

Nested `matrix(a b c d e f)` is the same story with rotation or scale baked in. In SVG the matrix maps user coordinates: `x' = ax + cy + e`, `y' = bx + dy + f`.

The identity matrix with a translation, `1 0 0 1 tx ty`, is just a translate; flatten or remove it. A `-1` in the first or fourth position usually means a flip. Keep that flip either in the path or in one group, not together with a page translate. A matrix such as `0 -1 1 0 tx ty` is a 90-degree rotation; outline it in Figma or keep one transform on the glyph group. A scale such as `sx 0 0 sy 0 0` generally belongs in CSS unless the non-uniform drawing is intentional.

Concatenated functions (`translate() rotate()`) apply **right-to-left**; nested `<g>` elements apply inside-out (innermost first). Figma encodes rotate-about-center as translate → rotate → translate. That is valid SVG; flatten in the file so `d` already sits in icon space.

CSS `transform-origin` does not rewrite the SVG `transform` attribute. `getCTM()` is user space → viewport; `getScreenCTM()` includes CSS transforms on ancestors. A large CTM translate after you “tightened `viewBox`” means the glyph is still in page space — and the new viewBox cropped it. That is the empty preview after a fake cleanup.

## When the icon is tiny, the artboard came along

`viewBox="0 0 1440 900"` with a 24px glyph is not a sizing bug in the CSS sense. The contract says the drawing is a whole page. The browser is correct. Tightening `width`/`height` on the `<svg>` just scales the **page**, so the icon stays a speck. `preserveAspectRatio="xMidYMid meet"` will letterbox it politely. `none` will stretch the empty page. Neither gets you the chevron.

This happens when Export is attached to a parent Frame, when Copy as SVG still inherits that parent's box, when a Section is selected, or when somebody copies an instance from a dashboard mock with the parent still in the selection stack. The quickest confirmation is to compare `viewBox` with the ink.

```js
const svg = document.querySelector('svg');
const ink = svg.querySelector('path, circle, rect, line, polyline, polygon');
console.log('viewBox', svg.getAttribute('viewBox'));
console.log('getBBox', ink.getBBox());
console.log('svg getBBox', svg.getBBox());
```

If `ink.getBBox()` is roughly `16×16` and `viewBox` is `1440×900`, the file is lying about what it is. `svg.getBBox()` can be the large box if a clip rect or empty group still spans the Frame — another reason to read the XML, not only the numbers.

Default `getBBox()` is the **object** bounding box: geometry only — no stroke, markers, clip, mask, or filter (SVG 2: `getBBox({ stroke: true })` where implemented). A line `(2,12)→(22,12)` with `stroke-width="2"` reports height 0. Rebuild `viewBox` from that and you clip the stroke; add half stroke plus miter spike (`stroke-miterlimit`). Filter regions (`x="-10%" width="120%"`) paint larger than bbox.

Recompute from ink (padded) or re-export the icon Frame. Tight box vs CSS is [viewBox vs width/height vs CSS](https://roman-riaboshtan.hashnode.dev/svg-viewbox-vs-width-height-css) — but not while the contract still describes a desktop.

No `viewBox`: `<img>` and `background-image` are replaced elements. HTML’s fallback intrinsic size is **300×150**. The glyph was not empty; the size mapping was gone. Restore `viewBox` from ink. Do not invent `0 0 24 24` if bbox is `0 0 20 20`.

## When a hairline disappears, check the box

Same geometry, different host: standalone SVG `overflow` on `:root` is typically visible; **inline** `svg:not(:root)` is `hidden` in the HTML user-agent stylesheet. A 1.5 stroke on `x=0`/`24` is half outside the viewport box. I check the Frame `clipPath`, then any clip on an inner Frame, then the computed `overflow` on the `<svg>`. After that I look at Figma's inside/outside stroke setting (SVG has center alignment only) and finally `shape-rendering="crispEdges"`, which can make a hairline disappear on an odd device pixel.

Optical padding inside a 24 grid is cheaper than a special-case stroke on one icon. A practical grid: glyph in 20×20, centered in 24, so a 2-unit stroke never sits on the root edge. If the set is 16px (small toolbar), the pad is 1.5–2 and the stroke is 1.25–1.5 in user units — still not flush to 0.

`overflow="visible"` is a valid fix when the set is already drawn to the box and you cannot re-pad eighty icons this week. It is a product decision: visible overflow can paint into adjacent buttons. Prefer padding when you can.

## The export panel does not tell the whole story

The right-sidebar **Export** panel has a few checkboxes that change the document you inspect. They are easy to miss because the canvas looks the same.

**Include “id” attribute.** On: you get `id="Layer_1"` / node-based ids on groups and sometimes on paths. Off: fewer collisions, harder to tell which group was which Frame. For a design-system icon that will be inlined, ids off (or prefix in CI) is the default worth arguing for. For an illustration you will only ever load as `<img>`, ids are mostly harmless.

**Outline text.** On: `<text>` becomes paths. File size goes up; reflow goes away; the mark is immutable. For icons and wordmarks in UI, this should be on. For a diagram that must remain selectable text, leave it off and ship fonts — that is not an icon pipeline.

There is no “outline stroke” checkbox in the export panel. Stroke outlining is a canvas operation you do _before_ export. If the team wanted filled silhouettes and you only ticked export settings, you still have strokes.

**Suffix and scale (`1x` / `2x` / `@2x`).** For PNG this is density. For SVG it is a misunderstanding. You may still get `width="48"` on a 24-grid drawing. That number is a default presentation size, not extra resolution. If someone then rasterizes using that 48, they are not wrong — they followed the attribute. The fix is a 1× SVG with an honest `viewBox`, then rasterize at CSS size × DPR when you actually need a bitmap.

Hidden layers (`visible: false` in Figma) sometimes still serialize, sometimes not, depending on Copy versus Export and whether the layer is inside the selected Frame. An `opacity="0"` path is worse: it is in the file, it can expand `getBBox()`, it can still be announced if it has a title, and `removeHiddenElems` may or may not drop it. If the preview looks empty but `d` is huge, search `opacity="0"` and `display="none"`. Designers leave “old” vectors under the new ones.

**Constraints** (left/right/top/bottom/scale) affect how a child resizes on the canvas. They do not have a 1:1 SVG attribute. What you get is whatever transform and box Figma computed _at export time_. A child with “scale” constraints, exported from a stretched instance, is a differently proportioned path than the main component. Inspect the instance you actually shipped, not the library thumbnail.

**Why Figma looks right.** Figma’s renderer is not a browser. Inside/outside strokes, background blur, missing fonts (Figma has the file), blend modes against the grey canvas, and Clip content as a _canvas_ feature all look correct there. The SVG is a lossy projection into a different engine. “But it looks fine in Figma” is not evidence that the XML is an icon document. It is evidence that the canvas scene is fine. Those are different artifacts. Paste the XML into a browser preview; that is the only preview that counts for shipping.

## Two less obvious cases

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

This one is _almost_ honest: `viewBox` is 24, the glyph is 16, padding is a translate. For a **button** that includes padding, keep it — but then the component’s `size={24}` is the whole control, not the glyph, and you will double-pad if the CSS button also has padding. For an **icon set**, you want the vector on the 24 grid with optical inset in the path, not a layout translate. Re-export the vector Frame, or flatten the 4,4 into the `d` and decide whether `viewBox` stays 24 (with padding inside) or becomes `0 0 16 16` (CSS provides the rest). Mixing both conventions in one sprite is how 16 and 24 icons end up looking like different optical weights.

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

## What I actually do when this happens

![Paste SVG, read the preview, fix clip and transform once before sizing or converting](./images/05-inspect-source-preview.svg)

Paste the raw export first, not JSX; `clip-path` is easier to search for than `clipPath={}`. Give the symptom a name — empty, shifted, clipped, or padded — and use that to choose where to look. Keep the preview visible while removing one node at a time. If dropping `clip-path` restores the glyph, you found the Frame. If removing `translate` moves it into place, flatten that group.

Then compare `viewBox` with `getBBox()`. A difference of an order of magnitude means the file probably still contains a page. A difference of roughly one stroke width means the box needs padding, since stroke is not included in the default bbox. Stop once the preview shows the glyph in a tight viewBox without an unnecessary Frame clip.

The editor on [getsvgeditor.com](https://getsvgeditor.com) is handy here because the source and preview stay visible together. At this point I am not “optimizing” anything. I am just checking whether the file still describes a page.

Cleaned-up chevron from the start of this article:

```xml
<!-- after: icon space, no Frame clip, no page translate -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" aria-hidden="true">
  <path d="M8 6L16 12L8 18" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
</svg>
```

The `d` values changed because flatten moved the chevron from local `4…18` inside `translate(712 438)` onto a 24 grid. That rewrite belongs in Figma (or one careful edit), not at every place the icon is used.

### Before changing the CSS

Ask what `viewBox` actually describes: the glyph, a Frame, a page, or a Section. Look for a Frame-sized `clip-path` or `mask`, and for a `translate`/`matrix` that only records page position. Check that `getBBox()` is in the same general range as the viewBox, leaving room for the stroke. Make sure the strokes are not flush with the edge and that the exported selection was the icon Frame rather than the desktop around it.

Only after that would I touch `currentColor`, an SVGO preset, or React props. When an SVG looks broken, reading the XML beside a browser preview is usually faster than guessing at CSS. In this particular failure mode, the path is very often the part that was right all along.
