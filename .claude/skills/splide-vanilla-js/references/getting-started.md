# Getting Started

Source: https://splidejs.com/guides/getting-started/
Checked against: `src/js/core/Splide/Splide.ts` (v4.1.3 source) and the v4.1.4 `dist/` files. The 4.1.3 and 4.1.4 `splide.esm.js` builds are identical except for the version banner.

## Contents

- [1. Load the files from the CDN](#1-load-the-files-from-the-cdn)
- [2. What each dist file is for](#2-what-each-dist-file-is-for)
- [3. Minimal markup](#3-minimal-markup)
- [4. Create and mount](#4-create-and-mount)
- [5. Script loading order](#5-script-loading-order)
- [6. Self-contained page (works from disk)](#6-self-contained-page-works-from-disk)
- [7. Multiple carousels](#7-multiple-carousels)
- [8. Options in the `data-splide` attribute](#8-options-in-the-data-splide-attribute)
- [9. `Splide.defaults`](#9-splidedefaults)
- [10. ESM from the CDN](#10-esm-from-the-cdn)
- [11. Docs snippets converted to no-bundler form](#11-docs-snippets-converted-to-no-bundler-form)

---

## 1. Load the files from the CDN

The docs list four ways to install: NPM, hosting the files yourself, CDN (jsDelivr), and the React/Vue/Svelte integration packages. This skill uses only the CDN or self-hosted copies of the same files.

Pin the exact version. The docs recommend a specific version number instead of `latest` in production, to avoid unexpected breakage from later updates.

### Default theme

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

### Other themes

Load exactly one theme file. Each one is complete: core styles plus arrows, pagination, progress bar, toggle and focus styles.

```html
<!-- Sky Blue -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-skyblue.min.css">

<!-- Sea Green -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-sea-green.min.css">

<!-- Default, explicit file (byte-identical to dist/css/splide.min.css) -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-default.min.css">
```

### Core only

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide-core.min.css">
```

`splide-core.min.css` contains only the layout and state rules. It has no arrow, pagination or progress-bar appearance, so you don't have to override theme styles. Use it when you style the controls yourself. Otherwise use `splide.min.css` or a `splide-[theme].min.css` file. See `themes.md` for every rule in each file.

### CSS files

| File (under `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/`) | Bytes | Contents |
|---|---|---|
| `splide.min.css` | 5,002 | All styles. Same content as `themes/splide-default.min.css`. |
| `themes/splide-default.min.css` | 5,002 | All styles, default theme |
| `themes/splide-skyblue.min.css` | 5,137 | All styles, Sky Blue theme |
| `themes/splide-sea-green.min.css` | 5,354 | All styles, Sea Green theme |
| `splide-core.min.css` | 1,964 | Core styles only (no arrows, pagination or progress-bar appearance) |

`dist/css` contains only minified CSS. There are no unminified CSS files.

### Subresource Integrity (optional)

These hashes were computed from the files jsDelivr serves for `@4.1.4`, which match the npm package byte for byte. Add `crossorigin="anonymous"` together with `integrity`.

| File | `integrity` |
|---|---|
| `dist/js/splide.min.js` | `sha384-RbYB5yr9jD1p+2OHlV3KzOBLAY48CbFsJ87bXEFYINGgOlQJNE4cEjTUf+Q2fygb` |
| `dist/css/splide.min.css` (and `themes/splide-default.min.css`) | `sha384-RQYEYzAnHyG0swTqlleCCsKHN9pUKyYMflAk6KEjvY6KhE5MLlvjwZkai5538g2T` |
| `dist/css/splide-core.min.css` | `sha384-UUMoBPww5zPuXpRJLHFTN+9x39lTnFOA9kj9q4HOVqv+aZH6G7jZwnXkrE5c18lX` |
| `dist/css/themes/splide-skyblue.min.css` | `sha384-gtXLXASD1+sb9IByi6Xkap/HiroMkSyPc+goLCSY+m/urO4jR9wozqmbqEeT5RHV` |
| `dist/css/themes/splide-sea-green.min.css` | `sha384-Sv6CQreqQHwlgh9O2mw4o19JFpNFBTYZkz0idi3mYpVfz3oLtUASkQXB5zT+T4df` |

```html
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css"
  integrity="sha384-RQYEYzAnHyG0swTqlleCCsKHN9pUKyYMflAk6KEjvY6KhE5MLlvjwZkai5538g2T"
  crossorigin="anonymous">
<script
  src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"
  integrity="sha384-RbYB5yr9jD1p+2OHlV3KzOBLAY48CbFsJ87bXEFYINGgOlQJNE4cEjTUf+Q2fygb"
  crossorigin="anonymous"></script>
```

Never use SRI with jsDelivr's auto-minified files (for example `splide.esm.min.js`, see section 10). jsDelivr generates them on demand and warns against SRI for them.

### Self-hosting

The docs' "Hosting Files" option is to download the package (GitHub releases), go to `dist/js`, and load `splide.min.js` with a `<script>` tag:

```html
<script src="path-to-the-file/splide.min.js"></script>
<link rel="stylesheet" href="path-to-the-file/splide.min.css">
```

Copy the same files that the CDN serves (`dist/js/splide.min.js` and one CSS file). Examples: a Shopify `assets/` folder with `{{ 'splide.min.js' | asset_url }}`, or WordPress `wp_enqueue_script` / `wp_enqueue_style`.

This skill already bundles the two most common files in `files/` at the skill root: `files/splide.min.js` (`dist/js/splide.min.js`) and `files/splide.min.css` (`dist/css/splide.min.css`, default theme). Copy those into the project instead of downloading them. Any other file (`splide-core.min.css`, the other themes, `splide.esm.js`, extensions) still has to come from the CDN URLs in this page and in `extensions/cdn.md`.

### Integrations (out of scope here)

React Splide, Vue Splide and Svelte Splide exist as separate packages. The latest Vue Splide supports only Vue 3. Vue 2 needs the old version 0.3.5, which also bundles an outdated Splide.

---

## 2. What each dist file is for

All paths are under `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/`. Sizes are from the 4.1.4 package.

| File | Format | Exposes | Bytes | Use |
|---|---|---|---|---|
| `js/splide.min.js` | UMD, minified | `window.Splide` | 29,803 (12,933 gzipped) | **Production classic `<script>`.** This is the package's `jsdelivr` entry. |
| `js/splide.js` | UMD, unminified | `window.Splide` | 85,483 | Debugging: readable stack traces and stepping |
| `js/splide.min.js.map` | Source map | — | 265,200 | Browser devtools map `splide.min.js` to source |
| `js/splide.min.js.gz` | Pre-gzipped `splide.min.js` | — | 12,933 | For self-hosted servers that serve pre-compressed files |
| `js/splide.esm.js` | ES module, unminified, no imports | default + named exports (below) | 97,011 | `<script type="module">` when you need `EventInterface`, constants, etc. |
| `js/splide.cjs.js` | CommonJS | `module.exports` | 98,899 | Node / bundlers using `require`. **Not usable in a browser `<script>`.** |
| `js/splide-renderer.min.js` | UMD, minified | `window.SplideRenderer` | 11,929 | Generates static carousel HTML (server-side or build-time pre-rendering). Not needed for normal pages. |
| `js/utils/splide-utils.esm.js`, `.cjs.js` | ESM / CJS | Internal DOM/array/object helpers | 7,271 / 8,463 | Package subpath `@splidejs/splide/src/js/utils`. Not needed for carousels. |
| `types/index.d.ts` | TypeScript declarations | — | — | Editor hints only |

### What the classic script exposes

`splide.min.js` and `splide.js` expose **only** `window.Splide`, with two static members:

| Member | Value |
|---|---|
| `Splide.defaults` | `{}` by default. Options merged into every instance created afterwards (section 9). |
| `Splide.STATES` | `{ CREATED: 1, MOUNTED: 2, IDLE: 3, MOVING: 4, SCROLLING: 5, DRAGGING: 6, DESTROYED: 7 }`. Use with `splide.state.is( Splide.STATES.IDLE )`. |

The classic build does **not** expose `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, `DEFAULTS`, `CLASSES`, `STATUS_CLASSES`, `EVENT_*`, `CLASS_*`, `SLIDE`/`LOOP`/`FADE` or `LTR`/`RTL`/`TTB`. Use string literals instead (`'move'`, `'is-active'`, `'loop'`, `'ttb'`), or use the ESM build (section 10).

UMD caveat: the UMD wrapper checks `module.exports` first, then AMD `define.amd`, and only then sets `window.Splide`. If the page already has an AMD loader (RequireJS, as in Magento 2), the script registers an anonymous AMD module and **does not** create `window.Splide`. Load it through the AMD loader there, or use the ESM build.

### `splide.esm.js` exports

`Splide` (also the default export), `SplideRenderer`, `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, `DEFAULTS`, `CLASSES`, `STATUS_CLASSES`, `SLIDE`, `LOOP`, `FADE`, `LTR`, `RTL`, `TTB`, and:

- `CLASS_ROOT`, `CLASS_TRACK`, `CLASS_LIST`, `CLASS_SLIDE`, `CLASS_CLONE`, `CLASS_CONTAINER`, `CLASS_ARROWS`, `CLASS_ARROW`, `CLASS_ARROW_PREV`, `CLASS_ARROW_NEXT`, `CLASS_PAGINATION`, `CLASS_PAGINATION_PAGE`, `CLASS_PROGRESS`, `CLASS_PROGRESS_BAR`, `CLASS_TOGGLE`, `CLASS_TOGGLE_PLAY`, `CLASS_TOGGLE_PAUSE`, `CLASS_SPINNER`, `CLASS_SR`, `CLASS_INITIALIZED`, `CLASS_ACTIVE`, `CLASS_PREV`, `CLASS_NEXT`, `CLASS_VISIBLE`, `CLASS_LOADING`, `CLASS_FOCUS_IN`, `CLASS_OVERFLOW`
- `EVENT_MOUNTED`, `EVENT_READY`, `EVENT_MOVE`, `EVENT_MOVED`, `EVENT_CLICK`, `EVENT_ACTIVE`, `EVENT_INACTIVE`, `EVENT_VISIBLE`, `EVENT_HIDDEN`, `EVENT_REFRESH`, `EVENT_UPDATED`, `EVENT_RESIZE`, `EVENT_RESIZED`, `EVENT_DRAG`, `EVENT_DRAGGING`, `EVENT_DRAGGED`, `EVENT_SCROLL`, `EVENT_SCROLLED`, `EVENT_OVERFLOW`, `EVENT_DESTROY`, `EVENT_ARROWS_MOUNTED`, `EVENT_ARROWS_UPDATED`, `EVENT_PAGINATION_MOUNTED`, `EVENT_PAGINATION_UPDATED`, `EVENT_NAVIGATION_MOUNTED`, `EVENT_AUTOPLAY_PLAY`, `EVENT_AUTOPLAY_PLAYING`, `EVENT_AUTOPLAY_PAUSE`, `EVENT_LAZYLOAD_LOADED`, `EVENT_SLIDE_KEYDOWN` (`'sk'`), `EVENT_SHIFTED` (`'sh'`), `EVENT_END_INDEX_CHANGED` (`'ei'`)

The last three have short internal string values since v4.1.0. The string names `'slide:keydown'` and `'shifted'` no longer fire. See `version4.md`.

### `splide-renderer.min.js` (undocumented in the guides)

From `src/js/renderer/SplideRenderer/SplideRenderer.ts`:

- `new SplideRenderer( contents, options?, config?, defaults? )`
  - `contents`: an array of HTML strings, or `{ html?, styles?, attrs? }` objects. At least one is required.
  - `options`: Splide options.
  - `config`: `{ id?, rootClass?, listTag = 'ul', slideTag = 'li', arrows?, hidden?, slider?, beforeSlider?, afterSlider?, beforeTrack?, afterTrack? }`.
  - `defaults`: pass `Splide.defaults` if you use it.
- `.html()` returns the carousel markup with an inline `<style>`. The root gets `is-rendered` (which the CSS makes visible) unless `config.hidden` is set.
- `SplideRenderer.clean( splide )` removes the pre-rendered clones and removes the inline `<style>` on `mounted`. Call it before `splide.mount()`.

---

## 3. Minimal markup

Two basic structures, depending on whether the carousel belongs to the main content.

Main content (banners, gallery, cards, product lists): use `<section>`.

```html
<section class="splide" aria-label="Splide Basic HTML Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</section>
```

Unrelated to the main content, or decoration only: use `<div>` with `role="group"`.

```html
<div class="splide" role="group" aria-label="Splide Basic HTML Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</div>
```

With a visible heading, use `aria-labelledby` instead of `aria-label`:

```html
<section class="splide" aria-labelledby="carousel-heading">
  <h2 id="carousel-heading">Splide Basic HTML Example</h2>

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</section>
```

- Put any content inside `.splide__slide`, such as images and text.
- `<div>` can replace `<section>`, `<ul>` and `<li>`.
- Always give the root a concise label.

Full rules, optional elements, data attributes and the generated ARIA are in `structure.md`.

---

## 4. Create and mount

```js
new Splide( target, options? ).mount( Extensions?, Transition? );
```

| Parameter | Type | Description |
|---|---|---|
| `target` | `string \| HTMLElement` | A CSS selector, resolved with `document.querySelector` (first match only), or the root element itself |
| `options` | `object` | Optional. See `options.md`. |

`mount()` returns the instance, so `const splide = new Splide( '.splide' ).mount();` works.

Always call `mount()`. Without it nothing appears in the browser. The root stays `visibility: hidden` until `mount()` adds `is-initialized`.

With the classic script, `Splide` is a global:

```html
<script>
  new Splide( '.splide' ).mount();
</script>
```

The target element must exist when the constructor runs. If the script is in `<head>`, wait for `DOMContentLoaded`:

```html
<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    var splide = new Splide( '.splide' );
    splide.mount();
  } );
</script>
```

Register `mounted`, `ready`, `arrows:mounted`, `pagination:mounted` and `navigation:mounted` listeners **between** the constructor and `mount()`. Listeners added after `mount()` never fire for these.

```js
var splide = new Splide( '.splide' );
splide.on( 'pagination:mounted', function ( data ) { /* ... */ } );
splide.mount();
```

### Errors thrown by the constructor and `mount()`

All assertion errors start with `[splide]`.

| Message | Cause |
|---|---|
| `[splide] null is invalid.` | The selector matched nothing, often because the script ran before the element was parsed |
| `[splide] Invalid JSON` | The `data-splide` attribute holds invalid JSON. An empty `data-splide=""` also throws. |
| `[splide] A track/list element is missing.` | No `.splide__track` in the root, or `.splide__list` is not a **direct child** of the track |
| `[splide] Already mounted!` | `mount()` was called twice without `destroy()` in between |
| `[splide] height or heightRatio is missing.` | `direction: 'ttb'` without `height` or `heightRatio` |
| `ReferenceError: Splide is not defined` | Init code ran before the core script. See section 5. |
| `TypeError: Cannot read properties of undefined (reading 'style')` | Custom `.splide__arrow--prev`/`--next` buttons without a `.splide__arrows` wrapper. See `structure.md`. |

---

## 5. Script loading order

Inline scripts ignore `defer`. Deferred external scripts run in document order, after parsing and **before** `DOMContentLoaded`.

| Pattern | Result |
|---|---|
| Core `<script src>` and the inline init both at the end of `<body>`, core first | Works |
| Core with `defer` in `<head>`, inline init wrapped in `DOMContentLoaded` | Works. The deferred core has run by the time `DOMContentLoaded` fires. |
| Core with `defer`, inline init **not** wrapped | `ReferenceError: Splide is not defined`. The inline script runs before the deferred core. |
| Core with `defer`, then your own init file with `defer` after it | Works. Deferred scripts keep their order, and the DOM is parsed. |
| Core in `<head>` without `defer`, inline init in `<head>` not wrapped | `[splide] null is invalid.` The markup isn't parsed yet. |
| Core with `async` and separate init code | Unreliable. `async` scripts run in any order. Don't use it. |
| Extensions | Load each extension script **after** the core, using the same `defer` choice |
| `<script type="module">` | Deferred automatically. No `DOMContentLoaded` wrapper needed. |

If your code may run after `DOMContentLoaded` has already fired (scripts injected later, a tag manager, a theme-editor reload), guard it:

```js
function onReady( fn ) {
  if ( document.readyState !== 'loading' ) {
    fn();
  } else {
    document.addEventListener( 'DOMContentLoaded', fn );
  }
}

onReady( function () {
  new Splide( '.splide' ).mount();
} );
```

`window.addEventListener( 'load', ... )` also works, but it waits for every image.

---

## 6. Self-contained page (works from disk)

This page uses classic scripts only, so it works when opened with `file://` and when served over http(s). Tested by loading it from `file://` in headless Chrome.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Splide demo</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js" defer></script>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem; }
    .splide__slide {
      display: flex; align-items: center; justify-content: center;
      height: 12rem; background: #e8edf3; font-size: 2rem;
    }
  </style>
</head>
<body>
  <section class="splide" aria-labelledby="demo-heading">
    <h2 id="demo-heading">Featured items</h2>
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">01</li>
        <li class="splide__slide">02</li>
        <li class="splide__slide">03</li>
        <li class="splide__slide">04</li>
        <li class="splide__slide">05</li>
      </ul>
    </div>
  </section>

  <script>
    document.addEventListener( 'DOMContentLoaded', function () {
      new Splide( '.splide', {
        perPage    : 3,
        gap        : '1rem',
        breakpoints: {
          640: { perPage: 1 },
        },
      } ).mount();
    } );
  </script>
</body>
</html>
```

Module scripts on `file://`: an inline `<script type="module">` that imports the full jsDelivr URL works from disk, because jsDelivr sends `access-control-allow-origin: *` (tested in headless Chrome). A module loaded from a **local file** (`<script type="module" src="app.js">`) is blocked on `file://`. Serve the folder over http(s) for that, or keep the code inline.

---

## 7. Multiple carousels

Each Splide instance controls exactly one carousel. A selector string goes through `document.querySelector`, so even a class selector applies the carousel to the first matching element only. Create one instance per carousel.

By ID:

```js
new Splide( '#slider1' ).mount();
new Splide( '#slider2' ).mount();
new Splide( '#slider3' ).mount();
```

With a loop (docs version):

```js
var elms = document.getElementsByClassName( 'splide' );

for ( var i = 0; i < elms.length; i++ ) {
  new Splide( elms[ i ] ).mount();
}
```

The same with `querySelectorAll`, keeping the instances for later use:

```js
var splides = [];

document.querySelectorAll( '.splide' ).forEach( function ( el ) {
  splides.push( new Splide( el, { perPage: 2 } ).mount() );
} );
```

Notes:

- Per-carousel options: use different constructor options per element, or a `data-splide` attribute on each root (section 8).
- Nested carousels (a `.splide` inside a slide of another `.splide`): `querySelectorAll( '.splide' )` returns both, and each gets its own instance. Splide ignores elements that belong to a nested carousel. It looks up track, arrows, pagination, progress bar and toggle only where `closest( '.splide' )` is the instance's own root.
- Each root without an `id` gets `splide01`, `splide02`, and so on, in creation order. Track, list and slide IDs derive from it. See `structure.md`.
- Mount the main carousel and the thumbnails as separate instances. Call `main.sync( thumbs )` before both `mount()` calls.

---

## 8. Options in the `data-splide` attribute

Put options as JSON on the root:

```html
<section class="splide" aria-label="Products"
  data-splide='{"type":"loop","perPage":3,"gap":"1rem","breakpoints":{"640":{"perPage":1}}}'>
  ...
</section>
```

```js
new Splide( '.splide' ).mount(); // reads data-splide automatically
```

### How the constructor builds the options (`Splide.ts`)

```js
options = merge(
  { label: root.getAttribute( 'aria-label' ) || '', labelledby: root.getAttribute( 'aria-labelledby' ) || '' },
  DEFAULTS,
  Splide.defaults,
  options || {}
);
merge( options, JSON.parse( root.getAttribute( 'data-splide' ) ) ); // throws "[splide] Invalid JSON" on failure
this._o = Object.create( merge( {}, options ) );
```

Precedence, lowest to highest:

1. `label` / `labelledby` taken from the root's existing `aria-label` / `aria-labelledby`
2. `DEFAULTS` (the built-in defaults)
3. `Splide.defaults`
4. Constructor `options`
5. `data-splide` JSON (**wins over constructor options**)

Verified: with `Splide.defaults = { perPage: 2, speed: 900 }`, constructor `{ perPage: 5, speed: 100 }` and `data-splide='{"perPage":3}'`, the result is `perPage: 3` and `speed: 100`.

Merge semantics (`utils/object/merge`):

- Plain objects merge **deeply**: `i18n`, `classes`, `breakpoints`, `padding`, `reducedMotion`. For example, `data-splide='{"i18n":{"prev":"Back"}}'` changes only `prev` and keeps the other strings.
- Arrays are copied, not merged.
- Other values are replaced.
- The merged result becomes the **prototype** of `splide.options`. Breakpoint options are later set as own properties on top and removed when the breakpoint stops matching. That is why a key missing from the base options falls back to its default.

Rules and caveats:

- Write strict JSON: double-quoted keys and strings. Wrap the attribute value in single quotes in HTML, or escape quotes as `&quot;`.
- A missing attribute is fine: `JSON.parse( null )` returns `null`, which merges as nothing. An **empty** attribute (`data-splide=""`) or invalid JSON throws `[splide] Invalid JSON`.
- JSON cannot hold functions or `undefined`, so `easingFunc` and similar options must go in the constructor.
- Breakpoint keys are strings in JSON (`"640"`). That works, since Splide reads them with `Object.keys` anyway.
- The attribute is read only once, in the constructor. Changing it later does nothing. Use `splide.options = { ... }` instead.
- Server-side templates: escape the JSON for an HTML attribute. In PHP, `htmlspecialchars( json_encode( $opts ), ENT_QUOTES )`. In Liquid, build the attribute from `| json` inside single quotes, or escape it.

---

## 9. `Splide.defaults`

A static object, `{}` by default, deep-merged into the options of every instance **constructed afterwards**. Its precedence is above the built-in defaults and below constructor options and `data-splide`.

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  Splide.defaults = {
    gap : '1rem',
    i18n: {
      prev: 'Vorherige Folie',
      next: 'Nächste Folie',
    },
  };

  document.addEventListener( 'DOMContentLoaded', function () {
    document.querySelectorAll( '.splide' ).forEach( function ( el ) {
      new Splide( el ).mount(); // each gets gap + German arrow labels
    } );
  } );
</script>
```

- Set it **before** `new Splide()`. Instances that already exist are not affected. (Verified: setting it after the constructor leaves `options.perPage` at `1`.)
- Assigning `Splide.defaults = { ... }` replaces earlier defaults. To add keys, use `Object.assign( Splide.defaults, { ... } )`, or nested assignment such as `Splide.defaults.i18n = { ... }`.
- The classic and ESM builds each have their own `Splide` class, so each has its own `defaults`. Use one build per page.
- For `SplideRenderer`, pass `Splide.defaults` as the 4th constructor argument.

---

## 10. ESM from the CDN

Use a module script when you need what the classic build doesn't expose: `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, `DEFAULTS`, `CLASSES`, `STATUS_CLASSES`, the `EVENT_*` / `CLASS_*` constants, or `SLIDE` / `LOOP` / `FADE` / `LTR` / `RTL` / `TTB`. `splide.esm.js` has no `import` statements of its own, so the browser loads it directly. No bundler is needed.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">

<section id="hero" class="splide" aria-label="Hero">...</section>

<script type="module">
  import { Splide, EventInterface, EVENT_MOVE, CLASS_ACTIVE } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function Logger( Splide, Components, options ) {
    const { on } = EventInterface( Splide ); // handlers removed automatically on destroy

    return {
      mount() {
        on( EVENT_MOVE, index => console.log( 'move to', index, CLASS_ACTIVE ) );
      },
    };
  }

  new Splide( '#hero' ).mount( { Logger } );
</script>
```

Default import also works: `import Splide from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';`.

Notes:

- Module scripts are deferred, so no `DOMContentLoaded` wrapper is needed.
- The ESM build does **not** create `window.Splide` (verified). Code in a classic script can't see that class. Keep all Splide code inside modules, or set `window.Splide = Splide` yourself.
- Don't load both builds on one page. They define separate `Splide` classes, with separate `defaults` and separate ID counters (two roots could both get `splide01`).
- `file://`: an inline module that imports the full CDN URL works from disk in Chrome (tested). A module loaded from a local file is blocked on `file://`, so serve the page over http(s) for that.
- `splide.esm.js` is unminified (97 KB). jsDelivr can minify it on the fly at `.../dist/js/splide.esm.min.js` (about 49 KB, made by jsDelivr, not part of the package). Don't put an SRI `integrity` on that URL.
- Extensions and their CDN builds are covered in `extensions/*.md`.

---

## 11. Docs snippets converted to no-bundler form

| Docs (bundler) | No-bundler equivalent |
|---|---|
| `$ npm install @splidejs/splide` | Drop it. Use the CDN tags in section 1. |
| `import Splide from '@splidejs/splide';` | `<script src=".../dist/js/splide.min.js">`, then use the global `Splide`. Or use the ESM URL from section 10. |
| `import '@splidejs/splide/css';` | `<link rel="stylesheet" href=".../dist/css/splide.min.css">` |
| `import '@splidejs/splide/css/skyblue';` | `<link rel="stylesheet" href=".../dist/css/themes/splide-skyblue.min.css">` |
| `import '@splidejs/splide/css/sea-green';` | `<link rel="stylesheet" href=".../dist/css/themes/splide-sea-green.min.css">` |
| `import '@splidejs/splide/css/core';` | `<link rel="stylesheet" href=".../dist/css/splide-core.min.css">` |
| `import { EventInterface } from '@splidejs/splide';` | ESM from the CDN URL (section 10). Inside an extension with the classic script, use `Splide.on( 'move.myNs', fn )` and `Splide.off( 'move.myNs' )`. |
| `new Splide( '.splide' ).mount();` | Same. Wrap it in `DOMContentLoaded` if the core script is deferred or the code is in `<head>`. |

Here `...` stands for `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4`. The bundler package maps `@splidejs/splide/css` to `dist/css/splide.min.css`, `/css/core` to `dist/css/splide-core.min.css`, and `/css/<name>` to `dist/css/themes/splide-<name>.min.css`.
