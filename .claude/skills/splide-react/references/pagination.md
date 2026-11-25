# Pagination

Scope: the pagination (indicator dots) in React Splide. Options, generated markup and ARIA, page count, keyboard, numbered and custom dots through `onPaginationMounted` / `onPaginationUpdated`, dots rendered by React outside the carousel, and styling.

Checked against the core 4.1.4 source (`components/Pagination/Pagination.ts`, `components/Controller/Controller.ts`, `components/Slides/Slides.ts`, `components/Slides/Slide.ts`), React Splide 0.7.12 (`src/js/components/Splide/Splide.tsx`, `src/js/constants/events.ts`), https://splidejs.com/guides/pagination/ and https://splidejs.com/components/pagination/. Every example was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [Setup](#setup)
- [Options](#options)
- [Generated markup and ARIA](#generated-markup-and-aria)
- [Adding classes](#adding-classes)
- [Placeholder: placing the pagination elsewhere](#placeholder-placing-the-pagination-elsewhere)
- [How many dots (page count)](#how-many-dots-page-count)
- [No paging: one dot per slide](#no-paging-one-dot-per-slide)
- [Direction](#direction)
- [Keyboard](#keyboard)
- [Numbered pagination](#numbered-pagination)
- [Custom pagination with `onPaginationMounted`](#custom-pagination-with-onpaginationmounted)
- [`onPaginationUpdated`](#onpaginationupdated)
- [Pagination component](#pagination-component)
- [Pagination outside the root (rendered by React)](#pagination-outside-the-root-rendered-by-react)
- [Styling](#styling)
- [Pitfalls](#pitfalls)

## Setup

```sh
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The types need one tsconfig line under `moduleResolution: "bundler"`; see `references/typescript.md`. Pagination is on by default: Splide generates the dots and appends them after the track.

## Options

Pass these through the `options` prop.

| Option | Type | Default | Notes |
|---|---|---|---|
| `pagination` | `boolean` | `true` | Creates the pagination (indicator dots). It must be `false` when `isNavigation` is used, or the roles and ARIA attributes conflict. With `false` and a placeholder in the JSX, the placeholder gets inline `display: none`. The pagination is rebuilt on `updated`, so a changed `options` prop or a breakpoint can switch it. |
| `paginationDirection` | `'ltr' \| 'rtl' \| 'ttb'` | `undefined`, which falls back to `direction` | Sets the pagination direction. It affects appearance (`splide__pagination--{dir}`), the keyboard shortcuts and `aria-orientation`. |
| `paginationKeyboard` | `boolean` | `true` | Enables keyboard shortcuts while a dot has focus. The W3C Carousel Design Pattern requires this. |
| `classes.pagination` | `string` | `'splide__pagination'` | Class of the `<ul>` that Splide creates. It is not applied to a placeholder. |
| `classes.page` | `string` | `'splide__pagination__page'` | Class of each dot `<button>`. |
| `i18n.select` | `string` | `'Select a slide to show'` | `aria-label` of the `<ul>`. |
| `i18n.slideX` | `string` | `'Go to slide %s'` | Dot label when `focus` is set or `perPage` is 1. |
| `i18n.pageX` | `string` | `'Go to page %s'` | Dot label when `perPage > 1` and `focus` is not set. |
| `focus` | `number \| 'center'` | `undefined` | Any defined value, `0` included, switches to one dot per slide ([No paging](#no-paging-one-dot-per-slide)). |
| `omitEnd` | `boolean` | `false` | Since v4.1.0. Omits dots that would only change the active slide without moving the carousel, and disables the next arrow at the end. Docs say it needs `type: 'slide'` (or unset) and `focus`. Source says: without `focus` it still trims the end index, so the next arrow is disabled at the end, but the dot count ignores it. |

## Generated markup and ARIA

The docs show this generated HTML:

```html
<ul class="splide__pagination splide__pagination--ltr" role="tablist" aria-label="Select a slide to show">
  <li role="presentation">
    <button
      class="splide__pagination__page is-active"
      type="button"
      role="tab"
      aria-controls="splide01-slide01"
      aria-label="Go to slide 1"
      aria-selected="true">
    </button>
  </li>
  <li role="presentation">
    <button
      class="splide__pagination__page"
      type="button"
      role="tab"
      aria-controls="splide01-slide02"
      aria-label="Go to slide 2"
      tabindex="-1"
    >
    </button>
  </li>
  ......
</ul>
```

| Element | Attributes (from source) |
|---|---|
| `<ul>` | `class="splide__pagination splide__pagination--{dir}"`, `role="tablist"`, `aria-label` = `i18n.select`. `aria-orientation="vertical"` only when the pagination direction is `ttb`; otherwise the attribute is absent. |
| `<li>` | `role="presentation"` |
| `<button>` (each dot) | `class="splide__pagination__page"`, `type="button"`, `role="tab"`, `aria-controls` = space-separated ids of all slides in that page, `aria-label` = `pageX` or `slideX` with the 1-based number. |
| Active dot | Adds class `is-active` and `aria-selected="true"`, and has no `tabindex`, so it is in the Tab order. |
| Inactive dots | `tabindex="-1"` and no `aria-selected` attribute (roving tabindex). |
| Slides | `role="tabpanel"` while pagination is enabled, otherwise `role="group"`. Ids are `{rootId}-slide01`, `{rootId}-slide02`, and so on. |

- With `perPage: 3` and no `focus`, the first dot's `aria-controls` is `splide01-slide01 splide01-slide02 splide01-slide03` and its label is "Go to page 1". The docs example shows the single-slide case.
- Without a placeholder, Splide appends the `<ul>` as the last child of the track's parent element. That is the root with the default `hasTrack`, or your wrapper if `<SplideTrack>` is nested inside one.
- Docs say (accessibility guide): `aria-selected` is "`false` for other dots". Source says: the attribute is removed from inactive dots, and only the active dot gets `aria-selected="true"`.
- Docs say (accessibility guide): the pagination `aria-label` is "Select slide to show". Source says: the default `i18n.select` is "Select a slide to show".
- Slides read `pagination` when they are created. Switching `pagination` at runtime does not change slide roles until a `refresh()` (`references/overflow.md` has a tested React fix).
- The dots are DOM nodes that Splide creates outside React's tree. React doesn't touch them on re-render, and `splide.destroy()` (on unmount) removes them.

## Adding classes

The `classes` option adds your classes to the list and button elements:

```tsx
const options: Options = {
  classes: {
    pagination: 'splide__pagination your-class-pagination',
    page      : 'splide__pagination__page your-class-page',
  },
};
```

The CSS targets the `splide__`-prefixed classes, so list the default class together with yours, never yours alone. The modifier `splide__pagination--{dir}` always uses the default prefix. Splide finds a placeholder by the hard-coded class `splide__pagination`, not by `classes.pagination`.

## Placeholder: placing the pagination elsewhere

By default, Splide appends the pagination to the parent of the track. To choose the position, render `hasTrack={ false }` and an empty placeholder `<ul className="splide__pagination" />`. This one sits in a bar above the slides, next to a caption that React re-renders on every move:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/pagination-placeholder.css';

const options: Options = { label: 'Placeholder example', height: '10rem' };

const slides = [ 'One', 'Two', 'Three' ];

export default function Example() {
  const [ index, setIndex ] = useState( 0 );

  return (
    <Splide options={ options } hasTrack={ false } onMove={ ( splide, newIndex ) => setIndex( newIndex ) }>
      <div className="carousel-top">
        <p>{ slides[ index ] }</p>
        {/* Leave it empty: Splide fills it with <li><button> items */}
        <ul className="splide__pagination" />
      </div>

      <SplideTrack>
        { slides.map( text => <SplideSlide key={ text }>{ text }</SplideSlide> ) }
      </SplideTrack>
    </Splide>
  );
}
```

`css/pagination-placeholder.css`:

```css
.carousel-top { display: flex; align-items: center; justify-content: space-between; }
.carousel-top .splide__pagination { position: static; padding: 0; }
.carousel-top .splide__pagination__page { background: #bbb; }
.carousel-top .splide__pagination__page.is-active { background: #222; }
```

Tested: Splide filled the placeholder with 3 items and added `splide__pagination--ltr`, created no second list, and the items survived the caption's re-render after a click on the second dot.

Never put the placeholder inside `<SplideTrack>` (structure restriction). With the default `hasTrack`, every child of `<Splide>` lands in the slide list, so a placeholder needs `hasTrack={ false }`.

From the source:

- The placeholder is found with `root.querySelector( '.splide__pagination' )`, excluding matches that belong to a nested `.splide`. It must be inside this root, at any depth, but not in the track or list. A `<ul>` outside the root is ignored; see [Pagination outside the root](#pagination-outside-the-root-rendered-by-react).
- Splide fills it with `<li><button>` items and adds `role`, `aria-label`, `aria-orientation` and the direction modifier class.
- On destroy or rebuild, only the `<li>` children are removed. The placeholder element stays.
- Use a `<ul>` (or `<ol>`), because Splide inserts `<li>` children. Leave it empty in JSX: React must not render children into it.

## How many dots (page count)

From `Pagination.ts` and `Controller.ts`:

```js
// number of dots
var hasFocus = options.focus !== undefined || options.isNavigation;
var count    = hasFocus ? Controller.getEnd() + 1 : Math.ceil( splide.length / options.perPage );
```

- `splide.length` counts real slides; clones are excluded.
- `perMove` does not change the number of dots. Dots follow `perPage` (or `focus`). Nine slides with `perPage: 3, perMove: 1` still give 3 dots.
- Clicking dot `i` calls `go( '>' + i )`. That moves to slide index `clamp( perPage * i, 0, endIndex )`, or to index `i` when `focus` is set.
- End index without `focus`: `length - perPage` (`length - 1` in a loop with `perMove`), so the last dot of a partial page aligns the last slides to the end. With `focus`: `length - 1`, or fewer with `omitEnd`.
- The active dot is `Controller.toPage( splide.index )`. Without `focus` that is `Math.floor( ( index >= endIndex ? length - 1 : index ) / perPage )`; with `focus` it is `Math.min( index, endIndex )`.
- The pagination is rebuilt on `updated` (breakpoints, a changed `options` prop), on `refresh` (including slides added or removed in JSX, which makes React Splide call `refresh()`), and when a resize changes the end index (`autoWidth` / `fixedWidth`). `onPaginationMounted` fires again after each rebuild.

| Slides | Options | Dots |
|---|---|---|
| 9 | `perPage: 3` | 3 |
| 10 | `perPage: 3` | 4 (the last goes to index 7) |
| 9 | `perPage: 3, perMove: 1` | 3 |
| 9 | `perPage: 3, focus: 0` | 9 |
| 9 | `perPage: 3, focus: 0, omitEnd: true` | 7 |

The core and theme CSS hide the pagination when all slides fit: `.splide:not(.is-overflow) .splide__pagination { display: none; }` (v4.1+). See `references/overflow.md`.

## No paging: one dot per slide

When you set `perPage`, Splide generates a button for each page, not for each slide. To turn off this paging, so that the carousel moves one by one and shows every dot, set `focus` to `0`. You don't have to set `perMove` in this case.

```tsx
<Splide options={ { label: 'One dot per slide', perPage: 3, focus: 0 } }>
```

That gives 9 dots for 9 slides. Near the end, the last 2 dots only change the active slide without sliding the carousel. Since v4.1, `omitEnd` omits those dots, and also disables the next arrow once the carousel can go no further:

```tsx
<Splide options={ { label: 'One dot per slide', perPage: 3, focus: 0, omitEnd: true } }>
```

`omitEnd` also works with `fixedWidth` and `autoWidth`, as long as `focus` is set too (without `focus` the dots are not omitted). A tested `autoWidth` example is in `references/auto-width.md`.

## Direction

By default, the pagination direction follows the `direction` option. With `direction: 'ttb'`, Splide treats the pagination as vertical regardless of how it looks: it sets `aria-orientation="vertical"` on the list and uses Up/Down to move focus.

To use horizontal pagination in a vertical carousel, or the reverse, override it with `paginationDirection`:

```tsx
const options: Options = {
  direction          : 'ttb',
  height             : '20rem',
  paginationDirection: 'ltr',
};
```

## Keyboard

With `paginationKeyboard: true` (the default), these keys work while a dot has focus (source plus the accessibility guide):

| Key | Action |
|---|---|
| Right | Horizontal: focus the next dot and show its page (the previous dot in RTL) |
| Left | Horizontal: focus the previous dot (the next dot in RTL) |
| Down / Up | Vertical (`ttb`): focus the next / previous dot |
| Home | First dot |
| End | Last dot |
| Space / Enter | Native button activation (click). Shows the slide if it isn't active. |

- Arrow keys wrap around: from the last dot, Right goes to the first.
- Moving focus activates the matching page ("selection follows focus"). With `waitForTransition: true`, the selection may not follow focus during a transition.
- The global `keyboard` option is separate and `false` by default in v4. Only the Tabs-pattern shortcuts above are on by default.
- These are native listeners on Splide's buttons. Don't add React `onKeyDown` to `<Splide>`: the prop types omit DOM handlers, and Splide already handles the keys.

## Numbered pagination

Without JavaScript, a CSS counter adds numbers:

```css
.splide__pagination {
  counter-reset: pagination-num;
}

.splide__pagination__page:before {
  counter-increment: pagination-num;
  content: counter( pagination-num );
}
```

## Custom pagination with `onPaginationMounted`

`onPaginationMounted` is the React prop for `pagination:mounted`. React Splide binds it before `mount()`, so it fires for the first build, and again after every rebuild. This example writes page numbers into the dots (the CSS counter above is simpler) and lets the user switch `perPage`, which rebuilds the pagination:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/pagination-numbered.css';

const slides = [ 1, 2, 3, 4, 5, 6, 7, 8, 9 ];

export default function Example() {
  const [ perPage, setPerPage ] = useState( 3 );
  const options: Options = { label: 'Numbered pagination', perPage, gap: '1rem', height: '8rem' };

  return (
    <>
      <button type="button" className="per-page" onClick={ () => setPerPage( perPage === 3 ? 1 : 3 ) }>
        Show { perPage === 3 ? 1 : 3 } per page
      </button>

      <Splide
        options={ options }
        onPaginationMounted={ ( splide, data ) => {
          // Runs after every rebuild (updated, refresh, end index change), not only on mount
          data.list.classList.add( 'splide__pagination--numbered' );

          data.items.forEach( item => {
            item.button.textContent = String( item.page + 1 );
          } );
        } }
      >
        { slides.map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
    </>
  );
}
```

`css/pagination-numbered.css`:

```css
/* Below the slides, as numbered buttons */
.splide__pagination--numbered { position: static; padding: 1rem 0 0; gap: .25rem; }
.splide__pagination--numbered .splide__pagination__page {
  width: auto; height: auto; min-width: 2em; padding: .25em .5em;
  border-radius: 4px; transform: none; background: #ddd; opacity: 1;
}
.splide__pagination--numbered .splide__pagination__page.is-active { background: #222; color: #fff; }
```

Tested: the dots read 1, 2, 3 ("Go to page 1" and so on). After the button switched `perPage` to 1, the pagination was rebuilt with 9 dots, labelled "Go to slide 1" to "Go to slide 9", and the handler numbered them and re-added the class.

Handler arguments (after the `splide` instance):

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | `{ list, items }` |
| `item` | `PaginationItem` | The item for the active slide (`Pagination.getAt( splide.index )`). |

`PaginationData`:

| Property | Type | Description |
|---|---|---|
| `list` | `HTMLUListElement` | The UL element that contains the pagination items. |
| `items` | `PaginationItem[]` | An array of `PaginationItem` objects. |

`PaginationItem`:

| Property | Type | Description |
|---|---|---|
| `li` | `HTMLLIElement` | The list item element. |
| `button` | `HTMLButtonElement` | The button element for the dot. |
| `page` | `number` | The page index (0-based). |

Notes:

- Splide writes `aria-label` and `aria-controls` once, when it creates the buttons. It updates only `is-active`, `aria-selected` and `tabindex` afterwards. So changes you make in `onPaginationMounted` (text, classes, a custom `aria-label`) last until the next rebuild, and the event fires again after each rebuild.
- The handler is captured at mount. It only uses its arguments here, so that's fine; to read props or state that change, use a ref (`references/reactivity.md`).
- These buttons belong to Splide, not React. Change them with DOM calls in the handler; don't try to render into them.
- The slides in a page are `splide.Components.Slides.getIn( item.page )`. Attributes on `<SplideSlide>` pass straight to the `<li>`, so `data-title` can carry a label for title dots:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/pagination-titles.css';

const options: Options = { label: 'Destinations', height: '10rem' };

const places = [ 'Lisbon', 'Kyoto', 'Oaxaca', 'Tromso' ];

export default function Example() {
  const [ counter, setCounter ] = useState( '' );

  return (
    <Splide
      tag="section"
      options={ options }
      hasTrack={ false }
      onPaginationMounted={ ( splide, data ) => {
        data.items.forEach( item => {
          const Slide = splide.Components.Slides.getIn( item.page )[ 0 ];
          const title = Slide && Slide.slide.getAttribute( 'data-title' );

          if ( title ) {
            item.button.textContent = title; // visible text
            item.button.setAttribute( 'aria-label', title );
          }
        } );
      } }
      onPaginationUpdated={ ( splide, data, prev, curr ) => {
        if ( curr ) {
          setCounter( `${ curr.page + 1 } / ${ data.items.length }` );
        }
      } }
    >
      <SplideTrack>
        { places.map( place => (
          <SplideSlide key={ place } data-title={ place }>{ place }</SplideSlide>
        ) ) }
      </SplideTrack>

      {/* Outside the track: with the default hasTrack, every child would land in the list */}
      <p className="slide-counter" aria-hidden="true">{ counter }</p>
    </Splide>
  );
}
```

`css/pagination-titles.css`:

```css
/* Text buttons instead of dots */
.splide__pagination__page { width: auto; height: auto; min-width: 2em; padding: .25em .5em; border-radius: 4px; transform: none; }
```

Tested: the dots read and are labelled "Lisbon", "Kyoto", "Oaxaca", "Tromso", and the counter changed from "1 / 4" to "3 / 4" after a click on the third one. Without the CSS the 8px theme dots overlap and the text of one button covers the next.

## `onPaginationUpdated`

`onPaginationUpdated` is the prop for `pagination:updated`. It fires whenever the pagination status is updated: on `move` (at the start of the transition), `scroll` and `scrolled`, once during each build before `pagination:mounted`.

| Argument | Type | Description |
|---|---|---|
| `splide` | `Splide` | The core instance (React Splide adds it). |
| `data` | `PaginationData` | `{ list, items }` |
| `prev` | `PaginationItem` | The previous item. |
| `curr` | `PaginationItem` | The current item. |

The title example above uses it for the "3 / 4" counter. Calling a `setState` function from it is fine, because the setter never changes.

## Pagination component

`ref.current?.splide?.Components.Pagination` is available after mount (in effects and handlers, not during render).

| Member | Signature | Description |
|---|---|---|
| `items` | `PaginationItem[]` | The data for each page (`li`, `button`, `page`). It is empty when pagination is disabled. |
| `getAt()` | `getAt( index: number ): PaginationItem \| undefined` | Returns the pagination item for a slide index. It converts the slide index to a page index, so with `perPage: 3`, `getAt( 0 )`, `getAt( 1 )` and `getAt( 2 )` return the same item. Returns `undefined` if not available. |

## Pagination outside the root (rendered by React)

Splide never finds a pagination element outside the root. If you need dots outside it, set `pagination: false` and render them with React, using the same page math. The recipe below uses a simple button group. It does not copy the full Tabs pattern (tablist, roving tabindex, arrow keys).

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label     : 'Gallery',
  pagination: false,
  perPage   : 3,
  gap       : '1rem',
  height    : '8rem',
  breakpoints: {
    640: { perPage: 1 },
  },
};

const photos = [ 1, 2, 3, 4, 5, 6, 7, 8, 9 ];

type Dots = { labels: string[], trackId: string };

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ dots, setDots ] = useState<Dots>( { labels: [], trackId: '' } );
  const [ page, setPage ] = useState( 0 );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( ! splide ) {
      return;
    }

    const { Controller, Elements } = splide.Components;
    const events = 'updated.dots refresh.dots resized.dots';

    function build() {
      const { i18n, perPage = 1 } = splide!.options;
      const count = Controller.hasFocus() ? Controller.getEnd() + 1 : Math.ceil( splide!.length / perPage );
      const text  = Controller.hasFocus() || perPage < 2 ? i18n!.slideX : i18n!.pageX; // same rule as Pagination.ts
      const labels = Array.from( { length: count }, ( _, i ) => text.replace( '%s', String( i + 1 ) ) );

      // Keep the old state object when nothing changed, so resizes don't re-render
      setDots( old => old.labels.join() === labels.join() ? old : { labels, trackId: Elements.track.id } );
      setPage( Controller.toPage( splide!.index ) );
    }

    build();
    splide.on( events, build ); // after mount: runs after Controller has recalculated

    return () => {
      splide.off( events );
    };
  }, [] );

  return (
    <>
      <Splide
        ref={ ref }
        tag="section"
        options={ options }
        onMove={ ( splide, index ) => setPage( splide.Components.Controller.toPage( index ) ) }
      >
        { photos.map( n => <SplideSlide key={ n }>Photo { n }</SplideSlide> ) }
      </Splide>

      <div className="gallery-dots">
        { dots.labels.map( ( label, i ) => (
          <button
            key={ i }
            type="button"
            className={ i === page ? 'gallery-dot is-active' : 'gallery-dot' }
            aria-label={ label }
            aria-controls={ dots.trackId }
            aria-current={ i === page || undefined }
            onClick={ () => ref.current?.go( `>${ i }` ) }
          />
        ) ) }
      </div>
    </>
  );
}
```

Tested: 3 dots labelled "Go to page 1" to "Go to page 3" with `aria-controls` ending in `-track` and no built-in pagination; a click on the third dot moved to "Photo 7" and marked that dot `aria-current`; at a 500px viewport the breakpoint switched to `perPage: 1` and the effect rebuilt 9 dots labelled "Go to slide …" with the seventh active.

- `onMove` keeps the active dot in sync: `splide.index` is already the new index when `move` fires.
- The dot count is read after mount, in an effect with handlers registered through `splide.on()`. Event props are bound before `mount()`, so an `onUpdated` or `onRefresh` prop runs before Controller has recalculated (tested with `onUpdated`: it read the old end index). The namespace (`updated.dots`) lets the cleanup remove only these handlers.
- `ref.current?.go( '>' + i )` is the component's own `go()`, which calls `splide.go()`.

## Styling

### Core CSS (`@splidejs/react-splide/css/core`, verbatim without the `-ms-` prefixed declarations)

```css
.splide__pagination{align-items:center;display:flex;flex-wrap:wrap;justify-content:center;margin:0;pointer-events:none}
.splide__pagination li{display:inline-block;line-height:1;list-style-type:none;margin:0;pointer-events:auto}
.splide:not(.is-overflow) .splide__pagination{display:none}
```

The `<ul>` has `pointer-events: none` and only the `<li>` items receive clicks. Keep that in mind if you put a background or a click target on the list.

### Theme CSS (`@splidejs/react-splide/css`, verbatim without the `-ms-` prefixed declarations)

```css
.splide__pagination{bottom:.5em;left:0;padding:0 1em;position:absolute;right:0;z-index:1}
.splide__pagination__page{background:#ccc;border:0;border-radius:50%;display:inline-block;height:8px;margin:3px;opacity:.7;padding:0;position:relative;transition:transform .2s linear;width:8px}
.splide__pagination__page.is-active{background:#fff;transform:scale(1.4);z-index:1}
.splide__pagination__page:hover{cursor:pointer;opacity:.9}
.splide__pagination__page:focus-visible{outline:3px solid #0bf;outline-offset:3px}
.splide.is-focus-in .splide__pagination__page:focus{outline:3px solid #0bf;outline-offset:3px}
.splide__pagination--ttb{bottom:0;display:flex;flex-direction:column;left:auto;padding:1em 0;right:.5em;top:0}
```

### Common overrides

```css
/* Theme: move the dots below the slides instead of overlaying them */
.splide__pagination { position: static; padding: 1rem 0 0; }
.splide__pagination__page { background: #bbb; }
.splide__pagination__page.is-active { background: #222; }

/* Core-only starter dots */
.splide__pagination { gap: .5rem; padding: 1rem 0 0; }
.splide__pagination__page {
  width: 10px; height: 10px; padding: 0; border: 0; border-radius: 50%;
  background: #bbb; cursor: pointer;
}
.splide__pagination__page.is-active { background: #222; }
.splide__pagination__page:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__pagination__page:focus { outline: 3px solid #0bf; outline-offset: 3px; }

/* Numbered buttons (with the CSS counter or the onPaginationMounted text) */
.splide__pagination__page { width: auto; height: auto; min-width: 2em; padding: .25em .5em; border-radius: 4px; transform: none; }
```

`is-focus-in` is added to the root when the carousel gets focus from the keyboard. It is not removed immediately when focus leaves, so pair it with `:focus`.

## Pitfalls

- `onPaginationMounted` refires after each rebuild (a changed `options` prop, a breakpoint, added or removed slides). Put customizations there, not in an effect after mount.
- Children of `<Splide>` with the default `hasTrack` go into the slide list. A placeholder or a counter next to the dots needs `hasTrack={ false }` and your own `<SplideTrack>`.
- `pagination` must be `false` on an `isNavigation` (thumbnail) carousel.
- Expecting `perMove` to add dots: only `perPage` and `focus` change the count.
- A placeholder outside the root, or inside the track, is ignored or breaks the structure.
- The theme positions the pagination absolutely over the slides (`bottom: .5em`), against the track's parent. Override `position` to put it below, and keep other controls (such as an autoplay toggle) out of that strip.
- Dots disappear when all slides fit (`.splide:not(.is-overflow)` rule in v4.1 CSS). This is intended. Override it if you need dots anyway.
- Event props run before core's own handlers on `updated` and `refresh`, so Controller values read there are stale. Register with `splide.on()` after mount.
