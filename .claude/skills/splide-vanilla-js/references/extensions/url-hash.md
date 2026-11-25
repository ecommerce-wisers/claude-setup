# URL Hash Navigation extension

Source: https://splidejs.com/extensions/url-hash/
Package: @splidejs/splide-extension-url-hash@0.3.0 (peer: none declared; built against @splidejs/splide ^4.0.0, its devDependency; checked with 4.1.4)

Also checked: the 0.3.0 dist (`splide-extension-url-hash.esm.js`). A headless Chrome run with core 4.1.4 confirmed the start index from the hash, the hash updates, the history entries and `hashchange` navigation.

The extension ties the carousel to the URL hash. It parses the hash to choose the start index, and it updates the hash every time the carousel moves. It also follows later hash changes, such as links and the Back/Forward buttons.

## Load (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.min.js"></script>
```

- Load core first. The extension ships no CSS.
- The script sets `window.splide.Extensions.URLHash` (capital `URL`) and adds to the shared object, so other extensions stay.
- Files in `dist/js/`: `splide-extension-url-hash.min.js` (classic, 2.8 KB), `splide-extension-url-hash.js` (unminified classic + map), `splide-extension-url-hash.esm.js` (ES module), `splide-extension-url-hash.cjs.js` (CommonJS).

## Mount

All loaded extensions:

```js
new Splide( '#splide' ).mount( window.splide.Extensions );
```

Selective. Prefer this form: URL Hash rewrites the URL fragment on every carousel it is mounted on (see [Caveats](#caveats)):

```js
new Splide( '#splide' ).mount( { URLHash: window.splide.Extensions.URLHash } );
```

ES module. The build has no imports, so it needs no import map:

```html
<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { URLHash } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.esm.js';

  new Splide( '#splide' ).mount( { URLHash } );
</script>
```

## Markup

Give each slide its hash value with the `data-splide-hash` attribute. Write the value without `#`.

```html
<div class="splide">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" data-splide-hash="slide01"></li>
      <li class="splide__slide" data-splide-hash="slide02"></li>
      <li class="splide__slide" data-splide-hash="slide03"></li>
    </ul>
  </div>
</div>
```

| Attribute | On | Value |
|---|---|---|
| `data-splide-hash` | `.splide__slide` (real slides; clones are ignored) | The fragment for that slide, without `#`. It is matched exactly and case-sensitively against `location.hash` with its first `#` removed. |

## Options

The extension has no options of its own. The core `start` option is the fallback when the URL hash matches no slide.

## How it works

1. **Before mount** (the extension's `setup()`): it reads `location.hash`. If it equals a slide's `data-splide-hash`, the start index becomes that slide's index and overrides `start`. Otherwise the index is `options.start || 0`.
2. **On every `active` event**, when a slide becomes active:
   - If the slide has `data-splide-hash`, the extension sets `location.hash = value`. This **adds a browser history entry**, and Back/Forward step through the slides.
   - If the slide has no hash, it calls `history.replaceState( null, null, ' ' )`, which removes the fragment from the URL (path and query stay) without adding a history entry.
3. **On `window` `hashchange`**: if the new hash matches a slide, it calls `splide.go( index )`. Hashes that match nothing are ignored. So `<a href="#slide03">` links anywhere on the page move the carousel.
4. **On mount**: the first `active` event writes the initial slide's hash, which adds one history entry. Exception: if an extension mounted before URL Hash calls `Slides.update()` during its own mount, that first `active` fires before URL Hash listens, so no hash is written at mount. Video does this. Extensions mount in the key order of the object passed to `mount()`, and for `window.splide.Extensions` that is the order the scripts loaded.

Checked in Chrome. Opening `page.html#c` starts on the slide with `data-splide-hash="c"`. Moving from `#a` to `#b` to `#c` increases `history.length` by one each time. Setting `location.hash = 'b'` moves the carousel to that slide.

## Methods

The extension has no public methods. `splide.Components.URLHash` has only the lifecycle functions `setup()` and `mount()`. The `hashchange` listener is removed when the carousel is destroyed.

## Events

The extension emits no events of its own. It listens to the core `active` event and the window `hashchange` event. Use the core `move`/`moved`/`active` events for your own logic.

## Examples

### Linkable slides (official demo settings)

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>URL hash carousel</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .splide__slide { background: #eee; display: grid; place-items: center; }
  </style>
</head>
<body>
  <nav>
    <a href="#slide-01">First</a>
    <a href="#slide-05">Fifth</a>
    <a href="#slide-09">Last</a>
  </nav>

  <section id="hash-carousel" class="splide" aria-label="Linkable slides">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide" data-splide-hash="slide-01">01</li>
        <li class="splide__slide" data-splide-hash="slide-02">02</li>
        <li class="splide__slide" data-splide-hash="slide-03">03</li>
        <li class="splide__slide" data-splide-hash="slide-04">04</li>
        <li class="splide__slide" data-splide-hash="slide-05">05</li>
        <li class="splide__slide" data-splide-hash="slide-06">06</li>
        <li class="splide__slide" data-splide-hash="slide-07">07</li>
        <li class="splide__slide" data-splide-hash="slide-08">08</li>
        <li class="splide__slide" data-splide-hash="slide-09">09</li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.min.js"></script>
  <script>
    new Splide( '#hash-carousel', {
      perPage    : 3,
      perMove    : 1,
      height     : '9rem',
      focus      : 'center',
      trimSpace  : false,
      breakpoints: {
        600: {
          perPage: 2,
          height : '6rem',
        },
      },
    } ).mount( { URLHash: window.splide.Extensions.URLHash } );
  </script>
</body>
</html>
```

The docs code sample ends with `splide.mount( { URLHash );`, which is a syntax error. The correct call is `mount( { URLHash } )`. With `perPage > 1`, keep `perMove: 1`, `focus: 'center'` and `trimSpace: false` as in the demo. Otherwise the end index is clamped and the last slides can never become active, so their hashes never appear and `go()` to them lands on an earlier slide.

### Replace history instead of pushing (wrapper extension)

The extension always pushes a history entry. To keep Back for leaving the page, use a small custom extension instead of URL Hash:

```js
function HashReplace( Splide, Components ) {
  function indexOf( hash ) {
    var slides = Components.Elements.slides;
    for ( var i = 0; i < slides.length; i++ ) {
      if ( slides[ i ].getAttribute( 'data-splide-hash' ) === hash ) return i;
    }
    return -1;
  }
  function setup() {
    var i = indexOf( location.hash.slice( 1 ) );
    if ( i > -1 ) Components.Controller.setIndex( i );
  }
  function onHashChange() {
    var i = indexOf( location.hash.slice( 1 ) );
    if ( i > -1 ) Splide.go( i );
  }
  function mount() {
    Splide.on( 'active.hashReplace', function ( Slide ) {
      var hash = Slide.slide.getAttribute( 'data-splide-hash' );
      if ( hash ) history.replaceState( null, '', '#' + hash );
    } );
    window.addEventListener( 'hashchange', onHashChange );
  }
  function destroy() {
    Splide.off( 'active.hashReplace' );
    window.removeEventListener( 'hashchange', onHashChange );
  }
  return { setup: setup, mount: mount, destroy: destroy };
}

new Splide( '#hash-carousel' ).mount( { HashReplace: HashReplace } );
```

`replaceState` does not fire `hashchange`, so the carousel doesn't react to its own updates. Checked in Chrome: `history.length` did not grow while moving, and setting `location.hash` still moved the carousel.

## Caveats

- **History.** Every slide change adds a history entry, and mounting usually adds one too, so Back steps through slides before it leaves the page. Use the wrapper above if you don't want that.
- **Slides without a hash clear the fragment.** URL Hash mounted on a carousel whose active slide has no `data-splide-hash` removes the existing fragment at mount and on every move. Checked: `page.html#section2` became `page.html`. Mount it only on the carousel that uses hashes, never on every carousel through `mount( window.splide.Extensions )`.
- **One carousel per page.** Each instance writes `location.hash` on its own moves, and an instance whose active slide has no hash clears the fragment. Values must be unique across the page.
- **Page jumps.** If a hash value equals the `id` of an element on the page, the browser scrolls to that element whenever the hash is set. Don't reuse element ids (Splide's slide ids look like `<root id>-slide01`).
- **Encoding.** `location.hash` returns percent-encoded text, so values with spaces or non-ASCII characters (`café`) never match after a reload. Use `[A-Za-z0-9_-]`.
- **`go()` right after `mount()`.** The mount-time hash write queues a `hashchange`. With the default `speed: 400`, that event arrives before the new slide's `active` (which fires on `moved`) and sends the carousel back. In Chrome, `splide.go( 2 )` right after `mount()` ended at index 0; with `speed: 0` it stayed at 2. Choose the initial slide with the URL hash or `start`, or call `go()` later (after a timeout or a user action).
- **Grid.** It doesn't work with the Grid extension. Grid's generated pages have no `data-splide-hash`, and URL Hash resolves the hash against the slides before Grid groups them.
- Works with `type: 'loop'`; `Elements.slides` holds only the real slides, so indexes match.
- Don't write `var splide = new Splide(...)` at the top level of a classic script. That global replaces the `window.splide` namespace, so `window.splide.Extensions` becomes `undefined` (verified). Use `const`/`let`, another name, or a function scope.
- With an AMD loader on the page (RequireJS), the UMD wrapper calls `define()` instead of setting the global. See `cdn.md`.
