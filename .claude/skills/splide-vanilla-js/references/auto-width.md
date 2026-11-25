# Auto Width

Source: https://splidejs.com/guides/auto-width/

Also checked: https://splidejs.com/guides/options/ (`autoWidth`, `autoHeight`, `omitEnd`, `lazyLoad`) and the v4 source (`components/Layout/Layout.ts`, `components/Controller/Controller.ts`, `components/Clones/Clones.ts`, `components/LazyLoad/LazyLoad.ts`, `components/Slides/Slide.ts`). The 4.1.4 dist is identical to 4.1.3 apart from the version banner.

## Contents

- [Setup](#setup)
- [Options](#options)
- [What Splide does with autoWidth](#what-splide-does-with-autowidth)
- [Setting slide widths](#setting-slide-widths)
- [Navigation on the last page (`focus` + `omitEnd`)](#navigation-on-the-last-page-focus--omitend)
- [Auto height (vertical carousels)](#auto-height-vertical-carousels)
- [Combining with lazyLoad](#combining-with-lazyload)
- [Pitfalls](#pitfalls)

## Setup

The examples use the global `Splide`. No ESM import is needed.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

## Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `autoWidth` | `boolean` | `false` | If `true`, each slide's width is determined by the slide itself. Leave out `perPage` and `perMove`, or set them to `1`. With `lazyLoad`, make sure each slide has an explicit width. |
| `autoHeight` | `boolean` | `false` | If `true`, each slide's height is determined by the slide itself. This is the vertical (`direction: 'ttb'`) counterpart of `autoWidth`. Leave out `perPage` and `perMove`, or set them to `1`. With `lazyLoad`, make sure each slide has an explicit height. |
| `focus` | `number \| 'center'` | `undefined` | Common with `autoWidth`: `'center'` for a centred loop, and `0` together with `omitEnd`. |
| `omitEnd` | `boolean` | `false` | Since v4.1.0. Disables the next arrow on the last page and omits redundant dots. Needs `type: 'slide'` (or unset) and `focus`. |
| `perPage` / `perMove` | `number` | `1` / `undefined` | Must stay `1` / unset with `autoWidth` or `autoHeight`. |

## What Splide does with autoWidth

From `Layout.ts`:

```js
// slide width
options.autoWidth ? null : unit( options.fixedWidth ) || ( vertical ? '' : cssSlideSize() )
// slide height
unit( options.fixedHeight ) || ( vertical ? ( options.autoHeight ? null : cssSlideSize() ) : cssHeight() )
```

- With `autoWidth: true`, Splide writes no `width` on slides. `fixedWidth` and the `perPage`-based `calc()` width are ignored. Your CSS or inline width, or the content's intrinsic width, decides.
- Splide still writes the gap (`margin-right` on each slide; `margin-left` in RTL, `margin-bottom` in TTB). It also writes the height (`height`, `heightRatio` or `fixedHeight`, on the slide or on `.splide__slide__container`) and the track padding.
- `autoHeight` matters only in vertical carousels. Horizontal carousels never use it. `fixedHeight` takes precedence over `autoHeight`.
- Loop carousels: by default the number of clones on each side equals the number of slides (`Splide.length`) when `autoWidth` is on, or `autoHeight` in TTB. Override it with `clones`.
- Why `perPage` and `perMove` must be `1`: Controller uses `perPage` for the end index (`length - perPage`), for how far `go( '>' )` moves, and for the number of pagination dots (`ceil( length / perPage )`). LazyLoad `'nearby'` uses it for the preload range. With `autoWidth`, `perPage` no longer describes what is on screen, so values other than 1 give wrong ends, page jumps and dots.

## Setting slide widths

Basically, you set each slide's width with CSS:

```html
<div class="splide">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" style="width: 150px">Slide 01</li>
      <li class="splide__slide" style="width: 250px">Slide 02</li>
      <li class="splide__slide" style="width: 200px">Slide 03</li>
    </ul>
  </div>
</div>
```

If the content of each slide has explicit dimensions, such as an image with a `width` attribute, you don't have to.

The docs' first demo is a centred loop:

```js
var splide = new Splide( '.splide', {
  type     : 'loop',
  height   : '10rem',
  focus    : 'center',
  autoWidth: true,
} );

splide.mount();
```

Other ways to provide widths:

```css
/* Classes instead of inline styles */
.splide__slide.is-narrow { width: 150px; }
.splide__slide.is-wide   { width: 250px; }

/* Any CSS unit works. % is relative to the list (the visible area). */
.splide__slide.is-half   { width: 50%; }

/* Content-driven: image with explicit dimensions (width/height attributes or CSS) */
.splide__slide img { display: block; height: 10rem; width: auto; }
```

```html
<li class="splide__slide"><img src="a.jpg" width="320" height="200" alt="..."></li>
```

- Slides are flex items in `.splide__list` with `flex-shrink: 0` (core CSS), so they keep the width you give them.
- Sizes are measured from the DOM (`getBoundingClientRect`). Content without dimensions, such as images with no `width`/`height` or late web fonts, can measure wrong at mount. Splide re-measures on window `resize` and `load`, but only re-lays out when the root's own size changes. If slide sizes change after mount, call `splide.refresh()`.
- `gap` and `padding` work as usual with `autoWidth`.

## Navigation on the last page (`focus` + `omitEnd`)

With `autoWidth` on a `slide`-type carousel, the next arrow can just change the active slide on the last page without moving the carousel. The last few pagination dots behave the same way. Try pressing next at the end of this one:

```js
var splide = new Splide( '.splide', {
  height   : '10rem',
  autoWidth: true,
} );

splide.mount();
```

Since v4.1, `focus` and `omitEnd` disable the arrow when the carousel reaches the last page, and omit the redundant dots:

```js
new Splide( '.splide', {
  autoWidth: true,
  focus    : 0,
  omitEnd  : true,
} );
```

The docs demo for this result also sets `height: '10rem'`. Remember that `omitEnd` needs `type: 'slide'` (or unset) plus `focus`.

## Auto height (vertical carousels)

Splide also supports `autoHeight` for vertical carousels. The docs example sets per-slide heights (for example `style="height: 120px"`):

```js
var splide = new Splide( '.splide', {
  type      : 'loop',
  direction : 'ttb',
  height    : '20rem',
  focus     : 'center',
  autoHeight: true,
} );

splide.mount();
```

```html
<div class="splide" aria-label="Vertical auto height">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" style="height: 100px">Slide 01</li>
      <li class="splide__slide" style="height: 150px">Slide 02</li>
      <li class="splide__slide" style="height: 80px">Slide 03</li>
    </ul>
  </div>
</div>
```

(The docs demo also sets `breakpoints: { 640: { height: '16rem' } }`.)

A vertical carousel needs `height` or `heightRatio`. Without either, `mount()` throws `Error: [splide] height or heightRatio is missing.`, because Layout asserts it to size the track. `fixedHeight` alone does not satisfy the check.

## Combining with lazyLoad

The docs say: if you use `autoWidth` and `lazyLoad` together, make sure each slide has an explicit width (an explicit height for `autoHeight`).

Why, from `LazyLoad.ts`:

- At mount, an `<img data-splide-lazy="...">` without a `src` is hidden with `display: none` until it loads, and a `span.splide__spinner` is inserted next to it. A slide whose width comes only from that image is therefore 0 px wide, and the layout, positions and visible slides are all wrong.
- Fix 1: set the width on the slide (inline or with a CSS class), as in the docs.
- Fix 2: give the image a placeholder `src` (for example a tiny data URI) plus `width`/`height` attributes. An image that already has a `src` is not hidden, so its attributes keep the space:

```html
<li class="splide__slide">
  <img
    src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=="
    data-splide-lazy="photo-1.jpg"
    width="300" height="200"
    alt="Photo 1"
  >
</li>
```

- After each image loads, Splide emits `lazyload:loaded` (`img`, `Slide`) and an internal `resize`. That resize only re-lays out if the root size changed, so if widths change on load, call `splide.refresh()` in `lazyload:loaded`.
- `lazyLoad: 'nearby'` loads images within `perPage * ( preloadPages + 1 ) - 1` slides of the active index. With `autoWidth` (`perPage: 1`, `preloadPages: 1`) that is only plus or minus 1 slide, even if many narrow slides are visible. Raise `preloadPages`, for example to `4`, or use `lazyLoad: 'sequential'`.

```js
var splide = new Splide( '.splide', {
  autoWidth   : true,
  gap         : '1rem',
  lazyLoad    : 'nearby',
  preloadPages: 4,
} );

splide.on( 'lazyload:loaded', function () {
  splide.refresh(); // only needed if the image changes the slide size
} );

splide.mount();
```

## Pitfalls

- Setting `perPage` or `perMove` above 1 with `autoWidth` or `autoHeight` gives a wrong end index, multi-slide jumps and wrong dot counts.
- Slides with no width, such as empty slides or unsized lazy images, collapse to 0 px.
- On a `slide`-type carousel, next on the last page only changes the active slide. Use `focus: 0, omitEnd: true` (v4.1+).
- `fixedWidth` does nothing while `autoWidth` is `true`.
- A vertical carousel with `autoHeight` still needs `height` or `heightRatio`.
- Switching `autoWidth` on at runtime through `breakpoints` (source): Splide skips the width property rather than clearing it, so the `calc()` width written by the previous layout stays inline on the slides. Use `autoWidth` at every size, or refresh. `refresh()` rebuilds the Slide objects, which first restore each slide's original `style` attribute:

```js
splide.on( 'updated', function ( options ) {
  if ( options.autoWidth ) {
    splide.refresh(); // restores original inline styles, then lays out without a width
  }
} );
```

- If all slides together are narrower than the carousel, see `overflow.md` (`is-overflow`, the `overflow` event, centring).
