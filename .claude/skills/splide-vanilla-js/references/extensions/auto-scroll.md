# Auto Scroll extension

Source: https://splidejs.com/extensions/auto-scroll/
Package: @splidejs/splide-extension-auto-scroll@0.5.3 (peer: none declared; built against @splidejs/splide ^4.0.17, its devDependency; checked with 4.1.4)

Also checked: the 0.5.3 dist (`splide-extension-auto-scroll.esm.js`, `dist/types/types/options.d.ts`). A headless Chrome run with core 4.1.4 confirmed the global, the mount and play/pause.

The extension scrolls the carousel continuously, a few pixels per frame, instead of moving one slide at a time. Use it with `type: 'loop'` and `drag: 'free'`. It also works with the Intersection extension, so scrolling can start only while the carousel is in the viewport (see `intersection.md`).

## Load (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
```

- Load core first, then the extension. The extension ships no CSS.
- The script runs `window.splide = window.splide || {}; window.splide.Extensions = window.splide.Extensions || {}; window.splide.Extensions.AutoScroll = AutoScroll;`. It adds to the shared object, so loading other extensions keeps it. Note that `window.splide` (lowercase) is not the `window.Splide` class.
- If you add `defer` to core, add it to the extension too (deferred scripts run in document order), and run the init code inside `DOMContentLoaded`.
- Files in `dist/js/`: `splide-extension-auto-scroll.min.js` (classic script, 5.6 KB), `splide-extension-auto-scroll.js` (the same build unminified, with a source map; its banner wrongly says 0.5.2 but the code is 0.5.3), `splide-extension-auto-scroll.esm.js` (ES module), `splide-extension-auto-scroll.cjs.js` (CommonJS, not for browsers).

## Mount

Classic script, every loaded extension:

```js
new Splide( '#ticker', { type: 'loop', drag: 'free', autoScroll: { speed: 1 } } )
  .mount( window.splide.Extensions );
```

Classic script, only this extension. Use this form when other extensions are loaded on the page, because `mount( window.splide.Extensions )` mounts all of them (see [Caveats](#caveats)):

```js
new Splide( '#ticker', { type: 'loop', drag: 'free' } )
  .mount( { AutoScroll: window.splide.Extensions.AutoScroll } );
```

ES module. The build has no imports, so it loads straight from the CDN without an import map:

```html
<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { AutoScroll } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.esm.js';

  new Splide( '#ticker', { type: 'loop', drag: 'free' } ).mount( { AutoScroll } );
</script>
```

The ESM build sets no globals. Keep the key `AutoScroll`. The Intersection extension looks the component up as `splide.Components.AutoScroll`, and a key that matches a core component name (such as `Scroll`) replaces that core component.

## Markup

The extension needs no special markup. Standard Splide markup is enough.

The extension can also drive the core play/pause toggle button. This needs `useToggleButton: true`, which the docs page does not mention (it is in the typings and the source). The button must be inside the root:

```html
<section id="ticker" class="splide" aria-label="Partner logos">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
    </ul>
  </div>
  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>
```

While it scrolls, the button has `is-active` and `aria-label="Pause auto scroll"`. While it is paused, `is-active` is removed and the label is `"Start auto scroll"`. The core CSS (`splide.min.css`/`splide-core.min.css`) shows `.splide__toggle__play` when the button is not active and `.splide__toggle__pause` when it is. The extension updates the button's class and label whenever a `.splide__toggle` exists, but it binds the click only when `useToggleButton` is `true`. Don't turn on core `autoplay` on the same carousel, because core Autoplay also binds `.splide__toggle`.

## Options

Pass the options as `autoScroll` in the Splide options:

```js
new Splide( '.splide', {
  autoScroll: {
    speed: 2,
  },
} );
```

| Option | Type | Default | Description |
|---|---|---|---|
| `speed` | `number` | `1` | Scroll speed in pixels per frame. Since 0.5.0 a negative value scrolls the other way. `0` does not stop it: the source uses `speed \|\| 1`, so `0` scrolls at 1. To stop, call `pause()`. |
| `autoStart` | `boolean` | `true` | Whether scrolling starts after initialization. If `document.readyState` is not yet `'complete'`, it waits for the window `load` event. |
| `rewind` | `boolean` | not set (off) | Whether the carousel rewinds when it reaches the end. This only matters for `type: 'slide'`, because a loop has no end. At the end it calls `go( 0 )`, or `go( Controller.getEnd() )` when `speed` is negative, then scrolling resumes. Without it, scrolling stops at the end. This is separate from the core `rewind` option. |
| `pauseOnHover` | `boolean` | `true` | Whether scrolling pauses while the cursor is over the carousel (`mouseenter`/`mouseleave` on the root). |
| `pauseOnFocus` | `boolean` | `true` | Whether scrolling pauses while the carousel contains an active element (`focusin`/`focusout` on the root). |
| `useToggleButton` | `boolean` | not set (off) | Not on the docs page (typings: "use the Autoplay toggle button for controlling auto scroll"). Clicking `.splide__toggle` calls `play()` or `pause()`. |

Related settings outside the `autoScroll` object:

- `autoScroll: false` disables the extension for that carousel. Setting it at runtime (`splide.options = { autoScroll: false }`, or inside `breakpoints`) destroys the scroller. Setting it back to an object mounts it again.
- `splide.options = { autoScroll: { speed: 3 } }` merges into the current values on the `updated` event, and the new speed applies right away. The hover, focus and toggle listeners are bound only when the scroller mounts (at `mount()`, or when it is turned back on after `autoScroll: false`), so changing `pauseOnHover`, `pauseOnFocus` or `useToggleButton` while it runs has no effect.
- i18n keys (in the core `i18n` option) for the toggle's `aria-label`: `startScroll` (default `"Start auto scroll"`) and `pauseScroll` (default `"Pause auto scroll"`).

## Methods

The component is at `splide.Components.AutoScroll`, under the key you passed to `mount()`:

| Method | Signature | Description |
|---|---|---|
| `play()` | `play(): void` | Starts scrolling, or resumes it. It also clears the hover and focus pause flags and disables the ARIA live region while scrolling (`Components.Live.disable( true )`). When the scroller is not mounted (`type: 'fade'`, or `autoScroll: false`), it throws a `TypeError` (checked), so guard the call. |
| `pause()` | `pause(): void` | Pauses auto scroll and marks it as stopped. After a manual `pause()`, leaving the carousel with the mouse or ending a drag does not resume it; only `play()` does. It also turns the live region back on. |
| `isPaused()` | `isPaused(): boolean` | Returns `true` if auto scroll is paused, otherwise `false`. It also returns `true` when the scroller never mounted (`type: 'fade'` or `autoScroll: false`). |

```js
const ticker = new Splide( '#ticker', { type: 'loop', drag: 'free' } )
  .mount( { AutoScroll: window.splide.Extensions.AutoScroll } );

const { AutoScroll } = ticker.Components;
pauseBtn.addEventListener( 'click', () => AutoScroll.pause() );
playBtn.addEventListener( 'click', () => AutoScroll.play() );
```

## Events

The extension emits no events of its own and does not emit `autoplay:*`. It listens to core events:

- `move`, `drag` and `scroll` pause scrolling while an arrow click, a drag or a `go()` runs. `moved`, `dragged` and `scrolled` resume it, unless `pause()` was called or the pointer or focus is still inside.
- `updated` merges new `autoScroll` options (see above).

While it scrolls, it updates the index whenever a different slide reaches the active position. That updates `is-active` and the pagination, fires the core `active`/`inactive`/`visible`/`hidden` events, and runs the lazy loader when `lazyLoad: 'nearby'`. It does not fire `move` or `moved` for these index changes. Since 0.5.3 the arrows' disabled state is updated at most once every 500 ms.

## Examples

### Infinite logo ticker

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Auto Scroll ticker</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .splide__slide { display: flex; align-items: center; justify-content: center; height: 6rem; background: #f2f2f2; }
  </style>
</head>
<body>
  <section id="ticker" class="splide" aria-label="Partner logos">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Logo 01</li>
        <li class="splide__slide">Logo 02</li>
        <li class="splide__slide">Logo 03</li>
        <li class="splide__slide">Logo 04</li>
        <li class="splide__slide">Logo 05</li>
        <li class="splide__slide">Logo 06</li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
  <script>
    new Splide( '#ticker', {
      type      : 'loop',
      drag      : 'free',
      focus     : 'center',
      perPage   : 3,
      gap       : '1rem',
      arrows    : false,
      pagination: false,
      autoScroll: {
        speed: 1,
      },
    } ).mount( { AutoScroll: window.splide.Extensions.AutoScroll } );
  </script>
</body>
</html>
```

The official demo uses `type: 'loop'`, `drag: 'free'`, `focus: 'center'`, `perPage: 3` and `autoScroll: { speed: 1 }`. A second demo uses `speed: -1` to scroll the other way.

### Reverse direction, toggle button, translated labels, reduced motion

```html
<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Auto Scroll toggle</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .splide__slide { height: 8rem; background: #e8eef7; display: grid; place-items: center; }
    .splide__toggle { margin-top: .5rem; }
  </style>
</head>
<body>
  <section id="news" class="splide" aria-label="Neuigkeiten">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Item 1</li>
        <li class="splide__slide">Item 2</li>
        <li class="splide__slide">Item 3</li>
        <li class="splide__slide">Item 4</li>
        <li class="splide__slide">Item 5</li>
      </ul>
    </div>
    <button class="splide__toggle" type="button">
      <span class="splide__toggle__play">Start</span>
      <span class="splide__toggle__pause">Pause</span>
    </button>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
  <script>
    // The extension ignores the reducedMotion option, so check the preference yourself.
    var reduce = window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

    new Splide( '#news', {
      type      : 'loop',
      drag      : 'free',
      perPage   : 2,
      gap       : '1rem',
      pagination: false,
      i18n      : {
        startScroll: 'Automatisches Scrollen starten',
        pauseScroll: 'Automatisches Scrollen pausieren',
      },
      autoScroll: {
        speed          : -0.5,   // negative: slides travel left to right (LTR); fractions are allowed
        autoStart      : ! reduce,
        pauseOnHover   : true,
        pauseOnFocus   : true,
        useToggleButton: true,
      },
    } ).mount( { AutoScroll: window.splide.Extensions.AutoScroll } );
  </script>
</body>
</html>
```

### Start only when visible (with Intersection)

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
<script>
  new Splide( '#ticker', {
    type        : 'loop',
    drag        : 'free',
    perPage     : 3,
    autoScroll  : { speed: 1, autoStart: false },
    intersection: {
      inView : { autoScroll: true },
      outView: { autoScroll: false },
    },
  } ).mount( {
    AutoScroll  : window.splide.Extensions.AutoScroll,
    Intersection: window.splide.Extensions.Intersection,
  } );
</script>
```

## Caveats

- Mounting with `window.splide.Extensions` turns auto scroll on for that carousel even without an `autoScroll` option. The scroller starts unless `autoScroll: false` or `type: 'fade'` (verified). If several carousels share a page, mount AutoScroll only on the ones that should scroll.
- It does nothing with `type: 'fade'`. With `type: 'slide'` it stops at the end unless `autoScroll.rewind` is `true`. Use `type: 'loop'` for an endless ticker.
- `speed: 0` still scrolls (it is treated as `1`). Fractional values such as `0.5` work.
- `autoStart` waits for `window` `load`, so on a page with heavy images scrolling starts only after they finish loading.
- Moving the carousel (arrows, pagination, drag, `go()`) pauses scrolling and then resumes it. A manual `pause()` stays paused until `play()`.
- It ignores the core `reducedMotion` option and `prefers-reduced-motion`. Check `matchMedia( '(prefers-reduced-motion: reduce)' )` yourself and set `autoStart: false`, or `autoScroll: false`.
- Accessibility: content that moves for more than 5 seconds needs a way to pause it (WCAG 2.2.2). Provide the `.splide__toggle` button with `useToggleButton: true`, or your own buttons that call `pause()`/`play()`. The default `pauseOnHover`/`pauseOnFocus` help mouse and keyboard users, but not touch users.
- Don't write `var splide = new Splide(...)` at the top level of a classic script. That global replaces the `window.splide` namespace, so `window.splide.Extensions` becomes `undefined` (verified). Use `const`/`let`, another name, or a function scope.
- With an AMD loader on the page (RequireJS, e.g. Magento), the UMD wrapper calls `define()` and sets no global. See `cdn.md` for the workaround, or use the ESM build.
