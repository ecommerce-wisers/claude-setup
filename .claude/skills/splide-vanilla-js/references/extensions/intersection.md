# Intersection extension

Source: https://splidejs.com/extensions/intersection/
Package: @splidejs/splide-extension-intersection@0.2.0 (peer: none declared; built against @splidejs/splide ^4.0.0, its devDependency; checked with 4.1.4)

Also checked: the 0.2.0 dist (`splide-extension-intersection.esm.js`, `dist/types/types/options.d.ts`). A headless Chrome run with core 4.1.4 confirmed the global, the event order and the AutoScroll handoff.

The extension watches the carousel root with an `IntersectionObserver` and turns Autoplay, Keyboard, AutoScroll and Video on or off when the carousel enters or leaves the viewport. It also emits events, so you can react to intersection changes yourself.

## Load (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
```

- Load core first. The extension ships no CSS.
- The script sets `window.splide.Extensions.Intersection` and adds to the shared object (`window.splide.Extensions = window.splide.Extensions || {}`), so other extensions stay.
- To control AutoScroll or Video, load those scripts too. The order between extension scripts does not matter.
- Files in `dist/js/`: `splide-extension-intersection.min.js` (classic, 3.4 KB), `splide-extension-intersection.js` (unminified classic + map), `splide-extension-intersection.esm.js` (ES module), `splide-extension-intersection.cjs.js` (CommonJS).

## Mount

All loaded extensions:

```js
new Splide( '#splide', options ).mount( window.splide.Extensions );
```

Selective:

```js
new Splide( '#splide', options ).mount( { Intersection: window.splide.Extensions.Intersection } );

// together with the components it controls (keep these exact keys):
new Splide( '#splide', options ).mount( {
  Intersection: window.splide.Extensions.Intersection,
  AutoScroll  : window.splide.Extensions.AutoScroll,
  Video       : window.splide.Extensions.Video,
} );
```

ES module. The build has no imports, so it needs no import map:

```html
<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { Intersection } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.esm.js';

  new Splide( '#splide', options ).mount( { Intersection } );
</script>
```

The handlers look the components up by name: `Components.Autoplay`, `Components.Keyboard`, `Components.AutoScroll` and `Components.Video`. Register AutoScroll and Video under exactly those keys, or Intersection cannot find them.

## Markup

The extension needs no special markup. It observes the root element (`splide.root`).

## Usage

Pass the `intersection` option and define `inView`. This example starts autoplay when the carousel enters the viewport:

```js
new Splide( '#splide', {
  autoplay: 'pause',
  intersection: {
    inView: {
      autoplay: true,
    },
  },
} );
```

To pause autoplay again when the carousel leaves the viewport, add `outView`:

```js
new Splide( '#splide', {
  autoplay: 'pause',
  intersection: {
    inView: {
      autoplay: true,
    },
    outView: {
      autoplay: false,
    },
  },
} );
```

## Options

All options go under `intersection`. The docs list no defaults. When an option is not set, the extension passes `undefined` to `IntersectionObserver`, which uses its own defaults.

| Option | Type | Default | Description |
|---|---|---|---|
| `inView` | `IntersectionViewOptions` | not set | Options applied when the carousel enters, or is in, the viewport. |
| `outView` | `IntersectionViewOptions` | not set | Options applied when the carousel leaves the viewport. It takes the same properties as `inView`. |
| `root` | `Element \| Document \| null` | not set (the browser viewport) | The element used as the viewport. See MDN "Intersection Observer API: creating an intersection observer". |
| `rootMargin` | `string` | not set (`'0px'`) | Margin around the root element, in CSS margin syntax, for example `'200px'` or `'0px 0px -20% 0px'`. |
| `threshold` | `number \| number[]` | not set (`0`) | A threshold, or an array of thresholds, as the fraction of the root that must be visible before the observer callback runs. The docs page shows this signature as `gap: { row?, col? }` by mistake; the typings say `number \| number[]`. |
| `once` | `boolean` | not set (off) | If `true`, the extension stops observing once the carousel becomes visible in the viewport. `outView` is never applied after that. |

`IntersectionViewOptions` (for `inView` and `outView`):

| Property | Type | Description (`true` / `false`) |
|---|---|---|
| `autoplay` | `boolean` | Starts or pauses autoplay. `true` calls `Components.Autoplay.play()`, but only if the core `autoplay` option is truthy. Use `autoplay: 'pause'` so autoplay is on but waits. `false` calls `Components.Autoplay.pause()`. |
| `keyboard` | `boolean` | Enables or disables keyboard shortcuts through `Components.Keyboard.disable( false / true )`. Keyboard is off by default in v4, so also set the core `keyboard` option (`'global'` listens on `window`; `true`/`'focused'` listens on the root). |
| `autoScroll` | `boolean` | Starts or pauses auto scroll through `Components.AutoScroll.play()` / `.pause()`. If AutoScroll is not mounted, nothing happens. |
| `video` | `boolean` | Plays or pauses the active video. `true` calls `Components.Video.play()` (the video in the active slide); `false` calls `Components.Video.pause()` (all videos). If Video is not mounted, nothing happens. |

A property that is missing or `undefined` is left alone. Any other key, for example a typo like `autoScoll`, throws a `TypeError` inside the observer callback.

Another example with `rootMargin`: enable the arrow keys only while the carousel is within 200 px of the viewport:

```js
new Splide( '#splide', {
  keyboard: 'global', // the docs example omits this; without it there are no shortcuts to toggle
  intersection: {
    rootMargin: '200px',
    inView: {
      keyboard: true,
    },
    outView: {
      keyboard: false,
    },
  },
} );
```

## Methods

The extension has no public methods. `splide.Components.Intersection` exposes only the lifecycle functions `mount()` and `destroy()`. `destroy()` disconnects the observer, and `splide.destroy()` calls it for you.

## Events

Register handlers with `splide.on()` before `mount()`. The observer's first callback fires right after mounting.

| Event | Arguments | Fired |
|---|---|---|
| `intersection` | `entry: IntersectionObserverEntry` | Whenever the carousel crosses the threshold, in either direction. |
| `intersection:in` | `entry: IntersectionObserverEntry` | Whenever the carousel crosses the threshold and intersects the viewport (`entry.isIntersecting === true`). |
| `intersection:out` | `entry: IntersectionObserverEntry` | Whenever the carousel leaves the viewport (`entry.isIntersecting === false`). |

For each observer callback, the order is: the `inView`/`outView` options are applied, then `intersection:in` or `intersection:out` fires, then `intersection`. With `once: true`, the observer disconnects after the first `intersection:in`. The `intersection` event still fires for that entry.

```js
splide.on( 'intersection', function ( entry ) {
  if ( entry.isIntersecting ) {
    if ( entry.intersectionRatio >= 0.5 ) {
      // do something
    }
  }
} );

splide.on( 'intersection:in', function ( entry ) {
  console.log( 'in', entry.target );
} );

splide.on( 'intersection:out', function ( entry ) {
  console.log( 'out', entry.target );
} );
```

`entry.target` is the carousel root. Other useful fields are `intersectionRatio`, `boundingClientRect`, `intersectionRect`, `rootBounds` and `time`.

## Examples

### Autoplay only while visible

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Intersection autoplay</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    .spacer { height: 120vh; }
    .splide__slide { height: 12rem; background: #ddd; display: grid; place-items: center; }
  </style>
</head>
<body>
  <div class="spacer">Scroll down</div>

  <section id="promo" class="splide" aria-label="Promotions">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Slide 01</li>
        <li class="splide__slide">Slide 02</li>
        <li class="splide__slide">Slide 03</li>
      </ul>
    </div>
    <div class="splide__progress"><div class="splide__progress__bar"></div></div>
    <button class="splide__toggle" type="button">
      <span class="splide__toggle__play">Play</span>
      <span class="splide__toggle__pause">Pause</span>
    </button>
  </section>

  <div class="spacer"></div>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
  <script>
    var reduce = window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

    // Don't name a global `var splide`: it would overwrite window.splide (the extension namespace).
    var promo = new Splide( '#promo', {
      type        : 'loop',
      autoplay    : 'pause',      // autoplay on, but waiting
      interval    : 4000,
      intersection: {
        threshold: 0.5,           // at least half visible
        inView   : { autoplay: ! reduce },
        outView  : { autoplay: false },
      },
    } );

    promo.on( 'intersection:in', function ( entry ) {
      console.log( 'visible', entry.intersectionRatio );
    } );

    promo.mount( { Intersection: window.splide.Extensions.Intersection } );
  </script>
</body>
</html>
```

`inView: { autoplay: ! reduce }` matters. Under reduced motion, core `reducedMotion` sets `autoplay` to `'pause'`, which is truthy, so `inView: { autoplay: true }` would start autoplay anyway.

### Auto Scroll only while visible

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

`outView` calls `AutoScroll.pause()`, which marks the scroller as stopped, so hovering does not restart it off-screen. `inView` calls `play()`.

### Play the active video when visible, once

```js
new Splide( '#videos', {
  heightRatio : 0.5625,
  video       : { mute: true },  // browsers block unmuted playback without a user gesture
  intersection: {
    once  : true,
    inView: { video: true },
  },
} ).mount( {
  Video       : window.splide.Extensions.Video,
  Intersection: window.splide.Extensions.Intersection,
} );
```

Load the Video CSS and JS from `video.md` for this example.

### Custom root (scrolling container)

```js
new Splide( '#inside-panel', {
  autoplay    : 'pause',
  intersection: {
    root     : document.querySelector( '#scroll-panel' ), // must be an ancestor of the carousel
    threshold: [ 0, 0.25, 0.5, 0.75, 1 ],
    inView   : { autoplay: true },
    outView  : { autoplay: false },
  },
} ).mount( { Intersection: window.splide.Extensions.Intersection } );
```

With an array of thresholds, the callback, and the `inView` options with it, runs at each crossing while the carousel stays visible.

## Caveats

- IE is not supported, because the extension uses Intersection Observer (install a polyfill if needed). Without `window.IntersectionObserver` it silently does nothing and emits no events.
- The observer reports the current state immediately after `mount()`. A carousel that starts off-screen gets `outView` applied and `intersection:out` fired on page load.
- `inView: { autoplay: true }` needs the core `autoplay` option to be truthy. With `autoplay: false` nothing starts. With `autoplay: true`, autoplay starts at mount and is paused only when the first `outView` applies. Use `autoplay: 'pause'`.
- `keyboard` toggling needs the core `keyboard` option. It is off by default in v4.
- The `intersection` options are read once, when the extension mounts. Changing them through `splide.options = {...}` or `breakpoints` has no effect.
- An unknown key inside `inView` or `outView` throws a `TypeError` in the observer callback.
- When `once: true` is set, nothing is paused when the carousel later scrolls out of view.
- It ignores `prefers-reduced-motion`. Guard `autoplay`, `autoScroll` and `video` yourself, as in the first example.
- Mounting `window.splide.Extensions` mounts every loaded extension on that carousel. AutoScroll then starts scrolling and URL Hash rewrites the URL fragment even when you didn't configure them. Prefer the selective form.
- Don't write `var splide = new Splide(...)` at the top level of a classic script. That global replaces the `window.splide` namespace, and `window.splide.Extensions.Intersection` then throws (verified). Use `const`/`let`, another name, or a function scope.
- With an AMD loader on the page (RequireJS), the UMD wrapper calls `define()` instead of setting the global. See `cdn.md`.
