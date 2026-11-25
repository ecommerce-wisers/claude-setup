# Official extensions: CDN reference

Every URL below was checked against jsDelivr (HTTP 200, `access-control-allow-origin: *`, immutable caching) and the npm registry `latest` tags on 2026-11-24. Each extension's latest version is the one listed. No extension has had a release since 2022: Auto Scroll 0.5.3 is from 2022-09-06, Video 0.8.0 from 2022-09-05, and Intersection 0.2.0, Grid 0.4.1 and URL Hash 0.3.0 from April 2022. All five were loaded together with core 4.1.4 in headless Chrome, as classic scripts, as ES modules and through an import map.

## Table

Script URLs are relative to `https://cdn.jsdelivr.net/npm/`.

| Extension | Package @ version | Classic script (UMD) | CSS | UMD global | ESM build | ESM self-contained |
|---|---|---|---|---|---|---|
| Auto Scroll | `@splidejs/splide-extension-auto-scroll@0.5.3` | `@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js` | none | `window.splide.Extensions.AutoScroll` | `.../dist/js/splide-extension-auto-scroll.esm.js` (exports `AutoScroll`) | yes |
| Intersection | `@splidejs/splide-extension-intersection@0.2.0` | `@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js` | none | `window.splide.Extensions.Intersection` | `.../dist/js/splide-extension-intersection.esm.js` (exports `Intersection`) | yes |
| Video | `@splidejs/splide-extension-video@0.8.0` | `@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.min.js` | `@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css` (required) | `window.splide.Extensions.Video` | `.../dist/js/splide-extension-video.esm.js` (exports `Video`) | yes (`@vimeo/player` 2.17.1 is bundled; the YouTube IFrame API loads at runtime from `www.youtube.com`) |
| Grid | `@splidejs/splide-extension-grid@0.4.1` | `@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js` | none | `window.splide.Extensions.Grid` | `.../dist/js/splide-extension-grid.esm.js` (exports `Grid`) | yes |
| URL Hash | `@splidejs/splide-extension-url-hash@0.3.0` | `@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.min.js` | none | `window.splide.Extensions.URLHash` | `.../dist/js/splide-extension-url-hash.esm.js` (exports `URLHash`) | yes |

Full URLs, ready to paste:

```html
<!-- core (always first) -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>

<!-- extensions: include only the ones you use -->
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.min.js"></script>
```

Other files in each package's `dist/js/`: an unminified classic script `<name>.js` (with a source map, except Grid, which ships `<name>.min.js.map`) and `<name>.cjs.js` (CommonJS, for Node and bundlers, not for `<script>`). The bare package URL (for example `https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1`) serves the `.min.js`, but always write the full path.

## Compatibility with core 4.1.4

- None of the five packages declares `peerDependencies`. Each lists `@splidejs/splide` only as a devDependency: Auto Scroll `^4.0.17`, Intersection `^4.0.0`, Video `^4.0.8`, Grid `^4.0.1`, URL Hash `^4.0.0`. All five are v4 extensions. The releases built against Splide 3.x were older lines: Auto Scroll ≤ 0.3.7, Intersection 0.1.x, Video ≤ 0.6.8, Grid 0.3.x, URL Hash 0.2.x. Don't use those with v4, or these with v3.
- Every core API they call exists in 4.1.4: `Splide.event.bus`, `Move.getLimit/toIndex/translate`, `Controller.setIndex/getEnd`, `Components.Live`, `Keyboard.disable`, `LazyLoad.check`, `Direction.resolve` and `Elements.toggle`.
- The only runtime dependency is `@vimeo/player` (Video), which is bundled into the dist file.

## How the classic scripts register

All five UMD files end with the same code (shown for Grid):

```js
if (typeof window !== "undefined") {
  window.splide = window.splide || {};
  window.splide.Extensions = window.splide.Extensions || {};
  window.splide.Extensions.Grid = Grid;
}
```

- Each file **adds** its key to the shared `window.splide.Extensions` object. Loading several extensions merges them, in any order (checked: `Object.keys( window.splide.Extensions )` gave `AutoScroll,Intersection,Video,Grid,URLHash`).
- `window.splide` (lowercase) is a namespace object. It is not `window.Splide`, the core class. Core only sets `window.Splide`, so `window.splide` is `undefined` until an extension script runs.
- **Never create a global named `splide`.** A top-level `var splide = new Splide(...)` (or `function splide`, or `window.splide = ...`) in a classic script replaces the namespace. Checked: afterwards `window.splide` was the instance and `window.splide.Extensions` was `undefined`, so `mount( { X: window.splide.Extensions.X } )` threw. Top-level `const`/`let` do not create window properties and are safe, as is any other name or a function scope. The official docs examples use `const splide`.
- The wrapper is `(function (factory) { typeof define === 'function' && define.amd ? define(factory) : factory(); })(...)`. If an **AMD loader** (RequireJS, as used by Magento 2 and some themes) is on the page, the factory goes to `define()` and **no global is set**. Core `splide.min.js` behaves the same way, so `window.Splide` is also missing. Checked: with a stub `define.amd`, both globals were undefined. Workarounds:
  - Use the ES module builds (below).
  - Or hide `define` while the classic scripts load (checked working):

    ```html
    <script>window.__splideDefine = window.define; window.define = undefined;</script>
    <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js"></script>
    <script>window.define = window.__splideDefine; delete window.__splideDefine;</script>
    ```

- Global name leaks from the classic scripts (checked in Chrome):
  - `splide-extension-video.min.js` declares top-level `St`, `qt` and `e`, assigns `t`, and its bundled Vimeo library sets `VimeoPlayerResizeEmbeds_` and `VimeoSeoMetadataAppended`.
  - Core `splide.min.js` itself leaks `r`, `Jt`, `n` and `t`.
  - The other four extension scripts leak nothing besides `window.splide`.
  - The ESM builds run in module scope and leak nothing; only the Vimeo flags remain when Video is used.
- `defer`: if core has `defer`, give every extension script `defer` too (deferred scripts run in document order), and init in `DOMContentLoaded`.

## Mount forms

```js
// 1. Every loaded extension
new Splide( '#a', options ).mount( window.splide.Extensions );

// 2. Selective: pick the ones this carousel needs (recommended)
new Splide( '#a', options ).mount( {
  AutoScroll  : window.splide.Extensions.AutoScroll,
  Intersection: window.splide.Extensions.Intersection,
} );
```

Use the selective form whenever a page has more than one carousel or loads more than one extension. Two extensions act on any carousel they are mounted on, even without options (checked):

- **AutoScroll** starts scrolling unless `autoScroll: false` (or `type: 'fade'`).
- **URLHash** clears the URL fragment when the active slide has no `data-splide-hash`, so `page.html#section2` became `page.html` at mount.

Intersection observes but does nothing without `inView`/`outView`. Video only touches slides with `data-splide-*` video attributes. Grid only acts with a `grid` option.

Keep the keys `AutoScroll`, `Intersection`, `Video`, `Grid` and `URLHash`. The keys become `splide.Components.<Key>`. Intersection finds AutoScroll and Video by these exact names, and a key that matches a core component (`Scroll`, `Slides`, `Layout`, ...) replaces it.

## Combined example: core + Auto Scroll + Intersection

A logo ticker that scrolls only while at least 25% of it is on screen. It pauses on hover and focus, has a toggle button, and respects reduced motion.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Splide extensions from the CDN</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .spacer { height: 110vh; }
    #ticker .splide__slide { height: 6rem; display: grid; place-items: center; background: #f1f3f5; }
  </style>
</head>
<body>
  <div class="spacer">Scroll down</div>

  <section id="ticker" class="splide" aria-label="Our partners">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Partner 01</li>
        <li class="splide__slide">Partner 02</li>
        <li class="splide__slide">Partner 03</li>
        <li class="splide__slide">Partner 04</li>
        <li class="splide__slide">Partner 05</li>
        <li class="splide__slide">Partner 06</li>
      </ul>
    </div>
    <button class="splide__toggle" type="button">
      <span class="splide__toggle__play">Play</span>
      <span class="splide__toggle__pause">Pause</span>
    </button>
  </section>

  <div class="spacer"></div>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
  <script>
    var reduce = window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

    var ticker = new Splide( '#ticker', {
      type      : 'loop',
      drag      : 'free',
      focus     : 'center',
      perPage   : 4,
      gap       : '1rem',
      arrows    : false,
      pagination: false,
      breakpoints: {
        640: { perPage: 2 },
      },
      autoScroll: {
        speed          : 1,
        autoStart      : false,  // Intersection starts it
        useToggleButton: true,
      },
      intersection: {
        threshold: 0.25,
        inView   : { autoScroll: ! reduce },
        outView  : { autoScroll: false },
      },
    } );

    ticker.on( 'intersection', function ( entry ) {
      console.log( 'ticker visible:', entry.isIntersecting );
    } );

    ticker.mount( {
      AutoScroll  : window.splide.Extensions.AutoScroll,
      Intersection: window.splide.Extensions.Intersection,
    } );
  </script>
</body>
</html>
```

## ES modules without a build step

Every extension's `.esm.js` is self-contained: it has **no `import` statements**, so it does not import `@splidejs/splide`. You can import it by full URL from a `<script type="module">` with no import map. Core's `splide.esm.js` is also self-contained and exports both `Splide` (named) and a default export.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css">

<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { Grid } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.esm.js';
  import { Video } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.esm.js';

  new Splide( '#grid', { height: '20rem', grid: { rows: 2, cols: 2 } } ).mount( { Grid } );
  new Splide( '#videos', { heightRatio: 0.5625 } ).mount( { Video } );
</script>
```

- The ESM builds set no `window.splide` and no `window.Splide`.
- The extensions do not depend on which core build created the instance. An ESM extension can be mounted on an instance of the classic `window.Splide` (checked), because extensions are plain functions that get the instance passed in.
- Module scripts are deferred automatically, so they run after the document is parsed.

### Import map (only if the project already uses one)

No import map is needed: the default is the full URLs above, with no bare `@splidejs/...` specifiers. Use this pattern only when a project already maps its modules with an import map and wants the docs' `import ... from '@splidejs/...'` style. Put it before any module script (checked working in Chrome):

```html
<script type="importmap">
{
  "imports": {
    "@splidejs/splide": "https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js",
    "@splidejs/splide-extension-auto-scroll": "https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.esm.js",
    "@splidejs/splide-extension-intersection": "https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.esm.js",
    "@splidejs/splide-extension-video": "https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.esm.js",
    "@splidejs/splide-extension-grid": "https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.esm.js",
    "@splidejs/splide-extension-url-hash": "https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.esm.js"
  }
}
</script>
<script type="module">
  import Splide from '@splidejs/splide';
  import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
  import { Intersection } from '@splidejs/splide-extension-intersection';

  new Splide( '#ticker', {
    type: 'loop', drag: 'free', perPage: 3,
    autoScroll  : { autoStart: false },
    intersection: { inView: { autoScroll: true }, outView: { autoScroll: false } },
  } ).mount( { AutoScroll, Intersection } );
</script>
```

The import map does not load CSS. The Video CSS still needs its `<link>`.

## Self-hosting

To avoid a third-party CDN (Shopify `assets/`, WordPress `wp_enqueue_script`), copy the same files from the URLs above:

- `dist/js/<name>.min.js` for every extension;
- `dist/css/splide-extension-video.min.css` for Video.

The files are self-contained and reference no other files. Keep the load order: core first, then the extensions, and register the extensions with each carousel's `mount()` call.

The skill's `files/` folder bundles only core (`splide.min.js`, `splide.min.css`). No extension file is bundled, so download each extension you self-host from the URLs above.
