# Grid extension

Source: https://splidejs.com/extensions/grid/
Package: @splidejs/splide-extension-grid@0.4.1 (peer: none declared; built against @splidejs/splide ^4.0.1, its devDependency; checked with 4.1.4)

Also checked: the 0.4.1 dist (`splide-extension-grid.esm.js`, `dist/types/types/options.d.ts`). A headless Chrome run with core 4.1.4 confirmed the global, the generated DOM, `splide.length` and the ESM build.

The extension creates rows and columns in the carousel. You write a flat list of slides. The extension groups them into generated slides ("pages"), and each page holds a rows × cols grid of your original slides ("cells"). Each page can have its own dimensions.

## Load (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js"></script>
```

- Load core first. The extension ships no CSS; it sets the layout with inline styles.
- The script sets `window.splide.Extensions.Grid` and adds to the shared object, so other extensions stay.
- Files in `dist/js/`: `splide-extension-grid.min.js` (classic, 6.8 KB, + `.min.js.map`), `splide-extension-grid.js` (unminified classic), `splide-extension-grid.esm.js` (ES module), `splide-extension-grid.cjs.js` (CommonJS).

## Mount

All loaded extensions:

```js
new Splide( '#splide', { grid: { rows: 2, cols: 2 } } ).mount( window.splide.Extensions );
```

Selective:

```js
new Splide( '#splide', { grid: { rows: 2, cols: 2 } } ).mount( { Grid: window.splide.Extensions.Grid } );
```

ES module. The build has no imports, so it needs no import map:

```html
<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { Grid } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.esm.js';

  new Splide( '#splide', { grid: { rows: 2, cols: 2 } } ).mount( { Grid } );
</script>
```

## Markup

Write ordinary slides. No grid markup is needed:

```html
<section id="splide" class="splide" aria-label="Products">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">01</li>
      <li class="splide__slide">02</li>
      <li class="splide__slide">03</li>
      <li class="splide__slide">04</li>
      <li class="splide__slide">05</li>
      <li class="splide__slide">06</li>
    </ul>
  </div>
</section>
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

- Each page uses the same tag as your slides. A row is a `<ul>` when the slides are `<li>`, otherwise a `<div>`.
- Cells keep `splide__slide` and get `splide__slide--col`, plus an id `<page id>-colNN`. The root gets `splide--grid`.
- Row height is `calc(100%/rows - rowGap*(rows-1)/rows)` and the row has `display: flex`. Every row except the last gets `margin-bottom: rowGap`.
- Cell width is `calc(100%/cols - colGap*(cols-1)/cols)`. Every cell except the last in a row gets `margin-right: colGap` (`margin-left` in RTL).
- Original slides are moved, not cloned, and their order is kept.

## Options

Pass the `grid` option to the constructor:

```js
new Splide( '#splide', {
  grid: {
    rows: 2,
    cols: 2,
    gap : {
      row: '1rem',
      col: '1.5rem',
    },
  },
} );
```

| Option | Type | Default | Description |
|---|---|---|---|
| `rows` | `number` | `1` | The number of rows. |
| `cols` | `number` | `1` | The number of columns. |
| `dimensions` | `[ number, number ][]` | `[]` | Dimensions (rows, cols) for each page, as an array. With `[ [ 1, 1 ], [ 2, 2 ] ]` the first slide (page) is 1×1 and all the following ones are 2×2: the last entry repeats. When this option is set, the extension ignores `rows` and `cols`. A `0` or missing number counts as `1`. |
| `gap` | `{ row?: number \| string, col?: number \| string }` | `{}` | Gaps between rows and between columns. CSS relative units are accepted; a number means px. |

The grid is built only when `grid` is truthy and `rows > 1`, `cols > 1` or `dimensions` is a non-empty array. `grid: false` turns it off.

```js
{
  grid: {   // the docs page shows `grid {` here, which is a typo
    gap: {
      row: '1rem',
      col: '1rem',
    },
  },
}
```

## Breakpoints

You might need fewer cells, or no grid at all, on mobile devices. The `grid` option responds to [breakpoints](https://splidejs.com/guides/options/#breakpoints). Set it to `false` to turn grid mode off:

```js
new Splide( '#splide', {
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
} );
```

On every `updated` event (breakpoint change or `splide.options = {...}`), the extension rebuilds. It first restores the original slides, then regroups them and calls `splide.refresh()`. With `grid: false` it restores the flat slides and refreshes.

## Methods

The extension has no public methods. `splide.Components.Grid` exposes only the lifecycle functions `mount()` and `destroy()`. To switch the grid at runtime, change the option:

```js
splide.options = { grid: false };                 // flat slides again
splide.options = { grid: { rows: 2, cols: 3 } };  // rebuild (deep-merged into the current grid object)
```

## Events

The extension emits no events of its own. After building, it calls `splide.refresh()`, so the core `refresh` event fires. Core events (`active`, `move`, `click`, `visible`, ...) and `Slide` objects refer to the generated pages, not to the cells.

## Examples

### Mixed dimensions (official demo)

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Grid carousel</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .splide__slide--col { background: #eee; display: flex; align-items: center; justify-content: center; }
    .splide__slide--col img { width: 100%; height: 100%; object-fit: cover; }
  </style>
</head>
<body>
  <section id="gallery" class="splide" aria-label="Gallery">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">01</li>
        <li class="splide__slide">02</li>
        <li class="splide__slide">03</li>
        <li class="splide__slide">04</li>
        <li class="splide__slide">05</li>
        <li class="splide__slide">06</li>
        <li class="splide__slide">07</li>
        <li class="splide__slide">08</li>
        <li class="splide__slide">09</li>
        <li class="splide__slide">10</li>
        <li class="splide__slide">11</li>
        <li class="splide__slide">12</li>
        <li class="splide__slide">13</li>
        <li class="splide__slide">14</li>
        <li class="splide__slide">15</li>
        <li class="splide__slide">16</li>
        <li class="splide__slide">17</li>
        <li class="splide__slide">18</li>
        <li class="splide__slide">19</li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js"></script>
  <script>
    new Splide( '#gallery', {
      type   : 'loop',
      height : '14rem',   // rows are % of the page height, so a height is required
      perPage: 2,
      perMove: 1,
      grid   : {
        // You can define rows/cols instead of dimensions.
        dimensions: [ [ 1, 1 ], [ 2, 2 ], [ 2, 1 ], [ 1, 2 ], [ 2, 2 ], [ 3, 2 ] ],
        gap: {
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
    } ).mount( { Grid: window.splide.Extensions.Grid } );
  </script>
</body>
</html>
```

The demo's 19 items fill 1+4+2+2+4+6 = 19 cells, so it builds 6 pages. The code sample on the docs page leaves `pagination: false` out of the `640` breakpoint, but the live demo's options include it, so it is included here.

### 2 × 3 product grid that turns into a plain carousel on mobile

```html
<section id="products" class="splide" aria-label="Products">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide"><div class="splide__slide__container"><img src="p1.jpg" alt="Product 1"></div></li>
      <li class="splide__slide"><div class="splide__slide__container"><img src="p2.jpg" alt="Product 2"></div></li>
      <!-- ... -->
    </ul>
  </div>
</section>

<script>
  var products = new Splide( '#products', {
    heightRatio: 0.6,
    cover      : true,         // each cell's <img> (or its container's) becomes a background
    slideFocus : true,         // cells get tabindex="0" while their page is visible
    grid       : { rows: 2, cols: 3, gap: { row: '1rem', col: '1rem' } },
    breakpoints: {
      768: { grid: false, perPage: 2, gap: '1rem' },
    },
  } );

  products.mount( { Grid: window.splide.Extensions.Grid } );
  console.log( 'pages:', products.length ); // pages, not products
</script>
```

## Caveats

- **Height.** Row heights are percentages of the page (slide) height. Set `height` (the official demo uses `'14rem'`), `heightRatio` or `fixedHeight`, or the rows collapse to their content.
- **Everything counts pages.** `splide.length`, `splide.index`, `perPage`, `perMove`, `start`, `go()`, pagination, arrows, and `active`/`visible` events all refer to generated pages. The official demo uses `perPage: 2`, meaning two pages side by side. In Chrome, the demo builds 6 pages from 19 items above 640px and 7 pages below.
- In a `refresh` handler that you registered before `mount()`, `splide.length` still holds the old count, because it runs before core re-collects the slides. Read it after `mount()` returns, or in a later event such as `move`.
- **`cover: true`** makes each cell's `<img>` (or the `<img>` inside the cell's `.splide__slide__container`) a `center/cover` background and hides the image. Grid restores this when it is destroyed.
- **`slideFocus`** gives every cell `tabindex="0"` while its page is visible and removes it when the page is hidden.
- **`add()`/`remove()`** act on pages. A slide inserted with `splide.add()` becomes its own page, not a cell. To change the cells, destroy the carousel, edit the markup and mount again, or toggle `grid` off and on.
- **Other extensions.**
  - URL Hash: put `data-splide-hash` on the cells and it never matches. URL Hash also resolves the start index before Grid groups the slides.
  - Video: `data-splide-*` attributes on cells are ignored, because both extensions read attributes from real slides only.
  - Prefer the selective mount, so neither runs on a grid carousel.
- **`direction: 'ttb'`.** The column gap goes through `Direction.resolve( 'marginRight' )`, which in TTB becomes `margin-bottom`, so cells in a row get a bottom margin instead of a horizontal gap. Use `gap.col: 0` and space the cells with your own CSS.
- `type: 'loop'` works: the clones are copies of whole pages.
- `lazyLoad` works, because the images inside cells belong to the page slide.
- Don't write `var splide = new Splide(...)` at the top level of a classic script. That global replaces the `window.splide` namespace, so `window.splide.Extensions` becomes `undefined` (verified). Use `const`/`let`, another name, or a function scope.
- With an AMD loader on the page (RequireJS), the UMD wrapper calls `define()` instead of setting the global. See `cdn.md`.
