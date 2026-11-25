# Auto width

Scope: slides sized by their own content in React Splide (`autoWidth`, and `autoHeight` for vertical carousels), the end-of-list behaviour with `focus` and `omitEnd`, slide sizes that change after mount, and lazy loading.

Checked against the core 4.1.4 source (`components/Layout/Layout.ts`, `components/Controller/Controller.ts`, `components/Clones/Clones.ts`, `components/LazyLoad/LazyLoad.ts`, `components/Slides/Slide.ts`), React Splide 0.7.12 (`src/js/components/Splide/Splide.tsx`), https://splidejs.com/guides/auto-width/ and https://splidejs.com/guides/options/. Every example was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [Setup](#setup)
- [Options](#options)
- [What Splide does with autoWidth](#what-splide-does-with-autowidth)
- [Setting slide widths](#setting-slide-widths)
- [Navigation on the last page (`focus` + `omitEnd`)](#navigation-on-the-last-page-focus--omitend)
- [Sizes that change after mount](#sizes-that-change-after-mount)
- [Auto height (vertical carousels)](#auto-height-vertical-carousels)
- [Combining with lazyLoad](#combining-with-lazyload)
- [Pitfalls](#pitfalls)

## Setup

```sh
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The types need one tsconfig line under `moduleResolution: "bundler"`; see `references/typescript.md`.

## Options

Pass these through the `options` prop.

| Option | Type | Default | Notes |
|---|---|---|---|
| `autoWidth` | `boolean` | `false` | If `true`, each slide's width is determined by the slide itself. Leave out `perPage` and `perMove`, or set them to `1`. With `lazyLoad`, make sure each slide has an explicit width. |
| `autoHeight` | `boolean` | `false` | If `true`, each slide's height is determined by the slide itself. This is the vertical (`direction: 'ttb'`) counterpart of `autoWidth`. Leave out `perPage` and `perMove`, or set them to `1`. With `lazyLoad`, make sure each slide has an explicit height. |
| `focus` | `number \| 'center'` | `undefined` | Common with `autoWidth`: `'center'` for a centred loop, and `0` together with `omitEnd`. |
| `omitEnd` | `boolean` | `false` | Since v4.1.0. Disables the next arrow on the last page and omits redundant dots. Docs say it needs `type: 'slide'` (or unset) and `focus`. Source: without `focus` the next arrow is still disabled at the end, but the dots are not omitted. |
| `perPage` / `perMove` | `number` | `1` / `undefined` | Must stay `1` / unset with `autoWidth` or `autoHeight`. |

## What Splide does with autoWidth

From `Layout.ts`:

```js
// slide width
options.autoWidth ? null : unit( options.fixedWidth ) || ( vertical ? '' : cssSlideSize() )
// slide height
unit( options.fixedHeight ) || ( vertical ? ( options.autoHeight ? null : cssSlideSize() ) : cssHeight() )
```

- With `autoWidth: true`, Splide writes no `width` on slides. `fixedWidth` and the `perPage`-based `calc()` width are ignored. Your CSS or `style` prop, or the content's intrinsic width, decides.
- Splide still writes the gap (`margin-right` on each slide; `margin-left` in RTL, `margin-bottom` in TTB). It also writes the height (`height`, `heightRatio` or `fixedHeight`, on the slide or on `.splide__slide__container`) and the track padding. These are inline styles on the `<li>`, added next to whatever your `style` prop set; React leaves them alone as long as your `style` values don't change.
- `autoHeight` matters only in vertical carousels. Horizontal carousels never use it. `fixedHeight` takes precedence over `autoHeight`.
- Loop carousels: by default the number of clones on each side equals the number of slides (`Splide.length`) when `autoWidth` is on, or `autoHeight` in TTB. Override it with `clones`. Clones are copies of the rendered `<li>` made by Splide; React doesn't know about them, so React event handlers in slide content don't work in clones (tested, see [Combining with lazyLoad](#combining-with-lazyload)).
- Why `perPage` and `perMove` must be `1`: Controller uses `perPage` for the end index (`length - perPage`), for how far `go( '>' )` moves, and for the number of pagination dots (`ceil( length / perPage )`). LazyLoad `'nearby'` uses it for the preload range. With `autoWidth`, `perPage` no longer describes what is on screen, so values other than 1 give wrong ends, page jumps and dots.

## Setting slide widths

Set each slide's width with CSS, or with the `style` prop of `<SplideSlide>` (it passes straight to the `<li>`). If the content of each slide has explicit dimensions, such as an image with `width` and `height` attributes, you don't have to.

This example shows the docs' centred loop and a `slide`-type carousel that stops cleanly at the end:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const centred: Options = {
  label    : 'Centred loop',
  type     : 'loop',
  height   : '10rem',
  focus    : 'center',
  autoWidth: true,
};

const endAware: Options = {
  label    : 'Tags',
  height   : '10rem',
  autoWidth: true,
  focus    : 0,
  omitEnd  : true, // disables next on the last page and drops redundant dots
};

const widths = [ 150, 250, 200, 300, 180, 220, 260 ];

export default function Example() {
  return (
    <>
      <Splide id="centred" options={ centred }>
        { widths.map( ( width, i ) => (
          <SplideSlide key={ i } style={ { width } }>Slide { i + 1 }</SplideSlide>
        ) ) }
      </Splide>

      <Splide id="end-aware" options={ endAware }>
        { widths.map( ( width, i ) => (
          <SplideSlide key={ i } style={ { width } }>Slide { i + 1 }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested: the first two slides of the loop measure 150 and 250px, the loop has 14 clones (7 per side, `Splide.length`), and Splide wrote no width of its own. In the second carousel the next arrow became disabled before the last slide was reached, and fewer dots than slides were rendered.

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

```tsx
<SplideSlide className="is-wide">...</SplideSlide>
<SplideSlide><img src="a.jpg" width={ 320 } height={ 200 } alt="..." /></SplideSlide>
```

- Slides are flex items in `.splide__list` with `flex-shrink: 0` (core CSS), so they keep the width you give them.
- Sizes are measured from the DOM (`getBoundingClientRect`). Content without dimensions, such as images with no `width`/`height` or late web fonts, can measure wrong at mount. Splide re-measures on window `resize` and `load`, but only re-lays out when the root's own size changes. If slide sizes change after mount, call `splide.refresh()` (next section).
- `gap` and `padding` work as usual with `autoWidth`.

## Navigation on the last page (`focus` + `omitEnd`)

With `autoWidth` on a `slide`-type carousel, the next arrow can just change the active slide on the last page without moving the carousel. The last few pagination dots behave the same way.

Since v4.1, `focus: 0` and `omitEnd: true` disable the arrow when the carousel reaches the last page, and omit the redundant dots, as in the second carousel above. The docs demo for this result also sets `height: '10rem'`. Remember that `omitEnd` needs `type: 'slide'` (or unset) plus `focus` for the dots; the arrow is disabled even without `focus`.

## Sizes that change after mount

React Splide calls `splide.refresh()` only when the list's child elements change (slides added, removed or reordered). A change inside a slide keeps the same `<li>`, so Splide doesn't notice that the sizes changed. Refresh it yourself in an effect, after React has updated the DOM:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label    : 'Chips',
  autoWidth: true,
  gap      : '.5rem',
  focus    : 0,
  omitEnd  : true,
};

const chips = [ 'All', 'Shoes', 'Bags', 'Accessories', 'Sale', 'New in', 'Gifts', 'Brands' ];

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ large, setLarge ] = useState( false );

  // Splide doesn't see size changes inside slides: re-measure after React has updated the DOM
  useEffect( () => {
    ref.current?.splide?.refresh();
  }, [ large ] );

  return (
    <>
      <button type="button" className="size" onClick={ () => setLarge( ! large ) }>
        { large ? 'Smaller' : 'Larger' } chips
      </button>

      <Splide ref={ ref } options={ options }>
        { chips.map( chip => (
          <SplideSlide key={ chip }>
            {/* Size the content, not the <li>: refresh() restores the <li>'s style from when Splide first saw it */}
            <span className="chip" style={ { display: 'block', padding: large ? '1rem 2rem' : '.5rem 1rem' } }>
              { chip }
            </span>
          </SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested at a 700px viewport: with the small chips everything fits (no `is-overflow`, next disabled, 1 dot). After "Larger chips" the effect's `refresh()` gave `is-overflow`, an enabled next arrow and more dots. With the `refresh()` line commented out, the carousel kept the old state (no `is-overflow`, next disabled, 1 dot) although the chips no longer fitted.

Size the content, not the `<li>`, when the size comes from state. `refresh()` rebuilds the Slide objects, and each one first restores the `<li>`'s `style` attribute to what it was when Splide first saw the slide. Tested: a `<SplideSlide style={ { width } }>` changed from 150px to 300px by React went back to `width: 150px` after `refresh()`, and React didn't put it back (the prop hadn't changed since React's last render). A className on the `<li>`, or a style on an element inside it, is not affected.

The effect also runs once right after mount. That extra `refresh()` is harmless.

## Auto height (vertical carousels)

Splide also supports `autoHeight` for vertical carousels. The docs example sets per-slide heights:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label     : 'Vertical auto height',
  type      : 'loop',
  direction : 'ttb',
  height    : '20rem', // required: a vertical carousel needs height or heightRatio
  focus     : 'center',
  autoHeight: true,
};

const heights = [ 100, 150, 80, 120, 90 ];

export default function Example() {
  return (
    <Splide options={ options }>
      { heights.map( ( height, i ) => (
        <SplideSlide key={ i } style={ { height } }>Slide { i + 1 }</SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Tested: the slides measure 100, 150, 80, 120 and 90px. (The docs demo also sets `breakpoints: { 640: { height: '16rem' } }`.)

A vertical carousel needs `height` or `heightRatio`. Without either, `mount()` throws `Error: [splide] height or heightRatio is missing.`, because Layout asserts it to size the track (tested; `fixedHeight` alone does not satisfy the check). In React the error is thrown from `componentDidMount`, and without an error boundary React unmounts the whole tree.

## Combining with lazyLoad

The docs say: if you use `autoWidth` and `lazyLoad` together, make sure each slide has an explicit width (an explicit height for `autoHeight`).

Why, from `LazyLoad.ts`:

- At mount, an `<img data-splide-lazy="...">` without a `src` is hidden with `display: none` until it loads, and a `span.splide__spinner` is inserted next to it. A slide whose width comes only from that image is therefore 0 px wide, and the layout, positions and visible slides are all wrong.
- Fix 1: set the width on the slide (a className, or a `style` that never changes), as in the docs.
- Fix 2: give the image a placeholder `src` (for example a tiny data URI) plus `width`/`height` attributes. An image that already has a `src` is not hidden, so its attributes keep the space.
- After each image loads, Splide emits `lazyload:loaded` (`img`, `Slide`) and an internal `resize`. That resize only re-lays out if the root size changed, so if widths change on load, call `splide.refresh()` in `onLazyLoadLoaded` (capital L in Load; `onLazyloadLoaded` doesn't exist: React logs "Unknown event handler property" in development and ignores it).
- `lazyLoad: 'nearby'` loads images within `perPage * ( preloadPages + 1 ) - 1` slides of the active index. With `autoWidth` (`perPage: 1`, `preloadPages: 1`) that is only plus or minus 1 slide, even if many narrow slides are visible. Raise `preloadPages`, for example to `4`, or use `lazyLoad: 'sequential'`.

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label       : 'Photos',
  type        : 'loop',
  autoWidth   : true,
  gap         : '1rem',
  lazyLoad    : 'nearby',
  preloadPages: 4, // 'nearby' covers perPage * ( preloadPages + 1 ) - 1 slides
};

// 1x1 transparent GIF: an image that already has a src is not hidden while it waits
const PLACEHOLDER = 'data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==';

// Data URIs stand in for real photo URLs here
const photos = [ 300, 200, 250 ].map( ( width, i ) => ( {
  id : i + 1,
  width,
  src: `data:image/svg+xml,${ encodeURIComponent( `<svg xmlns="http://www.w3.org/2000/svg" width="${ width }" height="200"><rect width="100%" height="100%" fill="#89a"/></svg>` ) }`,
} ) );

export default function Example() {
  const [ loaded, setLoaded ] = useState( 0 );
  const [ likes, setLikes ]   = useState( 0 );

  return (
    <>
      <Splide
        options={ options }
        onLazyLoadLoaded={ splide => {
          setLoaded( n => n + 1 );
          splide.refresh(); // only needed if the image changes the slide size
        } }
      >
        { photos.map( photo => (
          <SplideSlide key={ photo.id }>
            <img src={ PLACEHOLDER } data-splide-lazy={ photo.src } width={ photo.width } height={ 200 } alt={ `Photo ${ photo.id }` } />
            <button type="button" className="like" onClick={ () => setLikes( n => n + 1 ) }>Like</button>
          </SplideSlide>
        ) ) }
      </Splide>

      <p className="status">{ loaded } images loaded, { likes } likes</p>
    </>
  );
}
```

Tested: every real slide's image ended up with its photo `src` and no `data-splide-lazy`, the first slide measured 300px, and `onLazyLoadLoaded` ran. Re-renders (the status line updates on every load) didn't put the placeholder back: React only writes `src` when the prop changes, so keep the placeholder constant. The Like button worked in a real slide, but a click on the same button in a clone did nothing: clones are copies Splide makes with `cloneNode()`, outside React's tree, so React handlers inside slide content don't run there.

## Pitfalls

- Setting `perPage` or `perMove` above 1 with `autoWidth` or `autoHeight` gives a wrong end index, multi-slide jumps and wrong dot counts.
- Slides with no width, such as empty slides or unsized lazy images, collapse to 0 px.
- On a `slide`-type carousel, next on the last page only changes the active slide. Use `focus: 0, omitEnd: true` (v4.1+).
- `fixedWidth` does nothing while `autoWidth` is `true`.
- A vertical carousel with `autoHeight` still needs `height` or `heightRatio`.
- Content that changes size after mount needs `ref.current?.splide?.refresh()` in an effect. React Splide refreshes only when the slide elements themselves change.
- `refresh()` restores each `<li>`'s original `style` attribute. Don't drive slide sizes from state through the `<SplideSlide>` `style` prop; size an inner element or switch classes.
- Switching `autoWidth` on at runtime (a changed `options` prop or `breakpoints`): Splide skips the width property rather than clearing it, so the `calc()` width written by the previous layout stays inline on the slides. Use `autoWidth` at every size, or refresh. `refresh()` rebuilds the Slide objects, which first restore each slide's original `style` attribute. Register the refresh with `splide.on( 'updated', ... )` in an effect, or call it in an effect that depends on the option you changed.
- If all slides together are narrower than the carousel, see `references/overflow.md` (`is-overflow`, the `overflow` event, centring).
