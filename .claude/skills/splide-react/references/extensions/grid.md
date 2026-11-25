# Grid extension (React)

Rows and columns with `@splidejs/splide-extension-grid` in React Splide: markup, generated DOM, options, breakpoints, and why a grid carousel needs a `key` when its slides change.

Source: https://splidejs.com/extensions/grid/
Package: @splidejs/splide-extension-grid@0.4.1 (peer: none declared; built against @splidejs/splide ^4.0.1, its devDependency; tested with React Splide 0.7.12 and React 19)

Contents: [Install](#install-and-mount) · [Markup](#markup) · [Options](#options) · [Breakpoints](#breakpoints-and-option-changes) · [Methods and events](#methods-and-events) · [Changing the slides](#changing-the-slides) · [Examples](#examples) · [Caveats](#caveats)

The extension creates rows and columns in the carousel. You render a flat list of `<SplideSlide>`s. The extension groups them into generated slides ("pages"), and each page holds a rows × cols grid of your original slides ("cells"). Each page can have its own dimensions.

## Install and mount

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-grid
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { Grid } from '@splidejs/splide-extension-grid';
import '@splidejs/react-splide/css';

<Splide options={ { label: 'Products', height: '20rem', grid: { rows: 2, cols: 2 } } } extensions={ { Grid } }>
  ...
</Splide>
```

The package ships no CSS (it sets the layout with inline styles) and exports only `Grid`. Importing it types the `grid` option.

## Markup

Render ordinary slides. No grid markup is needed:

```tsx
<Splide tag="section" id="products" options={ options } extensions={ { Grid } }>
  { items.map( item => <SplideSlide key={ item.id }>{ item.name }</SplideSlide> ) }
</Splide>
```

Generated DOM with `rows: 2, cols: 2, gap: { row: '4px', col: '4px' }` (captured in Chrome, abbreviated):

```html
<section id="splide" class="splide splide--grid ...">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide is-active is-visible" id="splide-slide01" style="width: calc(100%); height: 10rem;">
        <ul class="splide__slide__row" style="height: calc(50% - 2px); display: flex; margin: 0px 0px 4px; padding: 0px;">
          <li class="splide__slide splide__slide--col" id="splide-slide01-col01" style="width: calc(50% - 2px); margin-right: 4px;">01</li>
          <li class="splide__slide splide__slide--col" id="splide-slide01-col02" style="width: calc(50% - 2px);">02</li>
        </ul>
        <ul class="splide__slide__row" style="height: calc(50% - 2px); display: flex; margin: 0px; padding: 0px;">
          <li class="splide__slide splide__slide--col" id="splide-slide01-col03" ...>03</li>
          <li class="splide__slide splide__slide--col" id="splide-slide01-col04" ...>04</li>
        </ul>
      </li>
      <li class="splide__slide" id="splide-slide02">...05, 06...</li>
    </ul>
  </div>
</section>
```

- Each page uses the same tag as your slides. A row is a `<ul>` when the slides are `<li>` (as `<SplideSlide>` renders), otherwise a `<div>`.
- Cells keep `splide__slide` and get `splide__slide--col`, plus an id `<page id>-colNN`. The root gets `splide--grid`.
- Row height is `calc(100%/rows - rowGap*(rows-1)/rows)` and the row has `display: flex`. Every row except the last gets `margin-bottom: rowGap`.
- Cell width is `calc(100%/cols - colGap*(cols-1)/cols)`. Every cell except the last in a row gets `margin-right: colGap` (`margin-left` in RTL).
- Original slides are **moved**, not cloned, and their order is kept. Your `<SplideSlide>` elements become the cells, so their attributes, class names and React event handlers (`onClick`) stay on them (tested).
- **Inline styles don't survive.** Grid sets each cell's `width` and `margin-right`, and when it rebuilds or is destroyed it removes the whole `style` attribute of every cell and row. A `style` prop on `<SplideSlide>` disappears after a breakpoint change or a layout switch in production, and right away in development (StrictMode's remount), and React doesn't put it back (tested). Style an inner element, as in the examples, or use a class.

## Options

Pass the `grid` option in the `options` prop:

```tsx
options={ {
  grid: {
    rows: 2,
    cols: 2,
    gap : {
      row: '1rem',
      col: '1.5rem',
    },
  },
} }
```

| Option | Type | Default | Description |
|---|---|---|---|
| `rows` | `number` | `1` | The number of rows. |
| `cols` | `number` | `1` | The number of columns. |
| `dimensions` | `[ number, number ][]` | `[]` | Dimensions (rows, cols) for each page, as an array. With `[ [ 1, 1 ], [ 2, 2 ] ]` the first slide (page) is 1×1 and all the following ones are 2×2: the last entry repeats. When this option is set, the extension ignores `rows` and `cols`. A `0` or missing number counts as `1` when the slides are grouped, but the layout divides by the raw value, so those cells get no width (or the rows no height). Checked: `[ [ 2, 0 ] ]` grouped two cells per page, and the cells had no `width` style. Always give both numbers. |
| `gap` | `{ row?: number \| string, col?: number \| string }` | `{}` | Gaps between rows and between columns. CSS relative units are accepted; a number means px. |

The grid is built only when `grid` is truthy and `rows > 1`, `cols > 1` or `dimensions` is a non-empty array. `grid: false` turns it off. The docs page shows `grid {` in one sample, which is a typo for `grid: {`.

## Breakpoints and option changes

You might need fewer cells, or no grid at all, on mobile devices. The `grid` option responds to breakpoints. Set it to `false` to turn grid mode off:

```tsx
options={ {
  grid: {
    rows: 3,
    cols: 3,
  },
  breakpoints: {
    800: {
      grid: {
        rows: 2,
        cols: 2,
      },
    },
    600: {
      grid: false,
    },
  },
} }
```

On every `updated` event (breakpoint change, or a changed `options` prop, which React Splide applies with `splide.options = options`), the extension rebuilds. It first restores the original slides, then regroups them and calls `splide.refresh()`. With `grid: false` it restores the flat slides and refreshes. So switching the layout from state works without a remount (second example). A new object is deep-merged into the current `grid` value, so with `dimensions` set, new `rows`/`cols` are still ignored.

## Methods and events

The extension has no public methods. `splide.Components.Grid` exposes only the lifecycle functions `mount()` and `destroy()`. To switch the grid at runtime, change the option.

The extension emits no events of its own. After building, it calls `splide.refresh()`, so the core `refresh` event fires (`onRefresh`). Core events (`active`, `move`, `click`, `visible`, ...) and `Slide` objects refer to the generated pages, not to the cells. A cell's own `onClick` on `<SplideSlide>` still works.

`splide.length` counts pages, but read it at the right moment. React Splide binds event props before `mount()`, so in `onRefresh` it is still the number of original slides. Tested with 9 items in a 2 × 2 grid: `onResized` saw 9 and then 3, `onRefresh` saw 9, `onMounted` saw 3. Use `onMounted` for the first value and `onResized` (which fires again after every rebuild) for breakpoint changes, as in the first example.

## Changing the slides

Grid moves your `<SplideSlide>` elements out of the `<ul class="splide__list">` into the generated rows. React still believes they are children of the list, so when the slide list changes, React's DOM operations fail:

- **Inserting or removing** a slide anywhere but the end throws `NotFoundError: Failed to execute 'insertBefore' on 'Node': The node before which the new node is to be inserted is not a child of this node.` (or the `removeChild` equivalent), and React unmounts the whole tree (tested).
- **Appending** a slide doesn't throw, but the new slide lands directly in the list and becomes a page of its own instead of a cell (tested: 6 cells on two pages, then a third page with the 7th item).

Remount the carousel whenever the list changes, with a `key` built from the items. On unmount, Grid's `destroy()` puts the original slides back before React removes the tree, and the new instance groups the new list (second example). Unmounting and StrictMode's development remount work without a key, because Grid restores the DOM in `destroy()`.

## Examples

### Mixed dimensions (official demo) with a page count

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import { Grid } from '@splidejs/splide-extension-grid';
import '@splidejs/react-splide/css';

const options: Options = {
  label  : 'Gallery',
  type   : 'loop',
  height : '14rem', // rows are % of the page height, so a height is required
  perPage: 2,
  perMove: 1,
  grid   : {
    // You can define rows/cols instead of dimensions.
    dimensions: [ [ 1, 1 ], [ 2, 2 ], [ 2, 1 ], [ 1, 2 ], [ 2, 2 ], [ 3, 2 ] ],
    gap       : {
      row: '6px',
      col: '6px',
    },
  },
  breakpoints: {
    640: {
      height    : '8rem',
      perPage   : 1,
      pagination: false,
      grid      : {
        dimensions: [ [ 2, 2 ], [ 1, 1 ], [ 2, 1 ], [ 1, 2 ], [ 2, 2 ] ],
      },
    },
  },
};

const items = Array.from( { length: 19 }, ( _, i ) => String( i + 1 ).padStart( 2, '0' ) );

export default function Gallery() {
  const [ pages, setPages ] = useState( 0 );

  return (
    <>
      <Splide
        id="gallery"
        options={ options }
        extensions={ { Grid } }
        // splide.length counts pages. `resized` fires again after every Grid rebuild (at mount
        // and on breakpoint changes), so the last value wins. In `refresh` it is still the item count.
        onResized={ splide => setPages( splide.length ) }
      >
        { items.map( item => (
          <SplideSlide key={ item }>
            { /* Style an inner element: Grid removes the cells' style attribute when it rebuilds. */ }
            <div className="cell" style={ { height: '100%', background: '#eee', display: 'flex', alignItems: 'center', justifyContent: 'center' } }>
              { item }
            </div>
          </SplideSlide>
        ) ) }
      </Splide>
      <p id="pages">{ items.length } items on { pages } pages</p>
    </>
  );
}
```

The demo's 19 items fill 1+4+2+2+4+6 = 19 cells, so it builds 6 pages. The code sample on the docs page leaves `pagination: false` out of the `640` breakpoint, but the live demo's options include it, so it is included here. Tested in production and development: "19 items on 6 pages" at 1280 px, 7 pages at 600 px, and 6 again after widening.

### Product grid with a changing list and a layout switch

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { Grid } from '@splidejs/splide-extension-grid';
import '@splidejs/react-splide/css';

const extensions = { Grid };

export default function ProductGrid() {
  const [ products, setProducts ] = useState( [ 'P1', 'P2', 'P3', 'P4', 'P5', 'P6', 'P7' ] );
  const [ asGrid, setAsGrid ] = useState( true );

  return (
    <>
      <p>
        <button id="add" type="button" onClick={ () => setProducts( list => [ `N${ list.length + 1 }`, ...list ] ) }>
          Add a product first
        </button>
        <button id="remove" type="button" onClick={ () => setProducts( list => list.slice( 1 ) ) }>
          Remove the first product
        </button>
        <button id="layout" type="button" onClick={ () => setAsGrid( value => ! value ) }>
          { asGrid ? 'Show as a row' : 'Show as a grid' }
        </button>
      </p>

      <Splide
        // Grid moves the <SplideSlide> elements into generated pages, so React can no longer
        // insert or remove them. Remount with a new key whenever the list changes.
        key={ products.join( '|' ) }
        id="products"
        options={ {
          label : 'Products',
          height: '16rem',
          gap   : '1rem',
          // Changing `grid` through the options prop is fine: Grid rebuilds on `updated`.
          grid  : asGrid ? { rows: 2, cols: 3, gap: { row: '1rem', col: '1rem' } } : false,
          perPage: asGrid ? 1 : 3,
        } }
        extensions={ extensions }
      >
        { products.map( product => (
          <SplideSlide key={ product }>
            <div className="product" style={ { height: '100%', background: '#eef', display: 'grid', placeItems: 'center' } }>
              { product }
            </div>
          </SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested in production and development: adding at the front, removing twice, switching to a flat row and back, and adding while flat all produce the expected pages and cells with no errors. The same component without the `key` crashes on "Add a product first". The remount resets the position to the first page; pass `options.start` if you need to keep it.

## Caveats

- **Height.** Row heights are percentages of the page (slide) height. Set `height` (the official demo uses `'14rem'`), `heightRatio` or `fixedHeight`, or the rows collapse to their content.
- **Everything counts pages.** `splide.length`, `splide.index`, `perPage`, `perMove`, `start`, `go()`, pagination, arrows, and `active`/`visible` events all refer to generated pages. The official demo uses `perPage: 2`, meaning two pages side by side. In Chrome, the demo builds 6 pages from 19 items above 640px and 7 pages below.
- **No `style` prop on `<SplideSlide>`.** Grid strips the cells' `style` attribute on every rebuild and on destroy (see [Markup](#markup)).
- **Slide-list changes need a `key`** (see [Changing the slides](#changing-the-slides)). `splide.add()` and `splide.remove()` act on pages: an added slide becomes its own page, not a cell.
- **`cover: true`** makes each cell's `<img>` (or the `<img>` inside the cell's `.splide__slide__container`) a `center/cover` background and hides the image. Grid restores this when it is destroyed.
- **`slideFocus`** gives every cell `tabindex="0"` while its page is visible and removes it when the page is hidden.
- **Other extensions.**
  - URL Hash: put `data-splide-hash` on the cells and it never matches. URL Hash also resolves the start index before Grid groups the slides.
  - Video: it depends on the key order in `extensions`. With `{ Grid, Video }`, Video sees only the generated pages, so `data-splide-*` attributes on cells are ignored. With `{ Video, Grid }`, Video creates players on the original slides before Grid moves them, so clicking a cell plays its video (checked), but `Video.play( index )` doesn't reach them.
  - Pass only the extensions a grid carousel needs.
- **`direction: 'ttb'`.** The column gap goes through `Direction.resolve( 'marginRight' )`, which in TTB becomes `margin-bottom`, so cells in a row get a bottom margin instead of a horizontal gap. Use `gap.col: 0` and space the cells with your own CSS.
- `type: 'loop'` works: the clones are copies of whole pages.
- `lazyLoad` works, because the images inside cells belong to the page slide.
