# Autoplay

Source: https://splidejs.com/guides/autoplay/

Also checked: https://splidejs.com/components/autoplay/, https://splidejs.com/guides/options/, https://splidejs.com/guides/events/, https://splidejs.com/guides/accessibility/, https://splidejs.com/extensions/intersection/, the v4 source (`components/Autoplay/Autoplay.ts`, `components/Autoplay/constants.ts`, `constructors/RequestInterval/RequestInterval.ts`, `components/Media/Media.ts`, `components/Live/Live.ts`), the 4.1.4 dist CSS, and the `@splidejs/splide-extension-intersection@0.2.0` dist script.

## Contents

- [Setup](#setup)
- [Options](#options)
- [How autoplay advances](#how-autoplay-advances)
- [Play/pause toggle button](#playpause-toggle-button)
- [Progress bar](#progress-bar)
- [Per-slide interval](#per-slide-interval)
- [Autoplay component methods](#autoplay-component-methods)
- [Events](#events)
- [Pause on hover and focus](#pause-on-hover-and-focus)
- [Reduced motion](#reduced-motion)
- [Accessibility (WCAG 2.2.1, 2.2.2, 2.3.3)](#accessibility-wcag-221-222-233)
- [Start autoplay only when in view](#start-autoplay-only-when-in-view)
- [Recipes](#recipes)
- [Pitfalls](#pitfalls)

## Setup

The examples use the global `Splide` and string event names. No ESM import is needed.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

```js
var splide = new Splide( '.splide', {
  type    : 'loop',
  perPage : 3,
  autoplay: true,
} );

splide.mount();
```

## Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `autoplay` | `boolean \| 'pause'` | `false` | `true` starts autoplay on mount. `'pause'` mounts the feature without starting it; start it with a toggle button or `Autoplay.play()`. With `false`, the Autoplay component does nothing, and a toggle button or progress bar in the HTML stays inert. With the Intersection extension, autoplay can run only while the carousel is in the viewport. |
| `interval` | `number` | `5000` | Autoplay interval in milliseconds. Override it per slide with `data-splide-interval`. |
| `pauseOnHover` | `boolean` | `true` | Pauses autoplay on mouseover of the root. It should stay `true` for accessibility. |
| `pauseOnFocus` | `boolean` | `true` | Pauses autoplay while the carousel contains the focused element. It should stay `true` for accessibility. |
| `resetProgress` | `boolean` | `true` | Whether to reset the progress when autoplay is asked to start again. With `false`, the elapsed time is kept. |
| `reducedMotion` | `Options` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Merged into the options when `(prefers-reduced-motion: reduce)` matches. |
| `i18n.play` | `string` | `'Start autoplay'` | Toggle button `aria-label` while autoplay is stopped. |
| `i18n.pause` | `string` | `'Pause autoplay'` | Toggle button `aria-label` while autoplay is on. |

Runtime changes (from the source):

- `autoplay`, `pauseOnHover` and `pauseOnFocus` are read once, when the component mounts. `splide.options = { autoplay: false }`, or a breakpoint that changes them after mount, does not start or stop autoplay. Call `Autoplay.play()` / `Autoplay.pause()` instead.
- Breakpoint values that apply at load do take effect, because breakpoints are merged before the Autoplay component is created.
- `interval` is re-read on every move, so a changed `interval` applies from the next slide.

## How autoplay advances

From `Autoplay.ts` and `RequestInterval.ts`:

- The timer runs on `requestAnimationFrame`. Each time it reaches `interval`, it calls `splide.go( '>' )`, the next page, which respects `perPage`, `perMove` and `focus`.
- In `type: 'slide'` without `rewind`, `go( '>' )` does nothing at the end while the timer keeps cycling. Use `rewind: true` or `type: 'loop'`.
- Every `move`, `scroll` and `refresh` rewinds the progress to 0. That includes moves made by arrows, pagination, drag and the API.
- `play()` only starts if there are more slides (clones included) than `perPage` (`Slides.isEnough()`). With too few slides, autoplay never starts.

## Play/pause toggle button

If you enable autoplay, also provide a play/pause button for accessibility; WCAG requires it. To mount one, put a button with the class `splide__toggle` in the carousel:

```html
<section class="splide">
  <div class="splide__slider">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Slide 01</li>
        <li class="splide__slide">Slide 02</li>
        <li class="splide__slide">Slide 03</li>
      </ul>
    </div>
  </div>

  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>
```

```js
new Splide( '.splide', {
  type    : 'loop',
  autoplay: 'pause',
  perPage : 3,
} );
```

You can use other inline elements instead of `<span>`, such as `<img>` and `<svg>`:

```html
<button class="splide__toggle" type="button">
  <svg
    class="splide__toggle__play"
    viewBox="0 0 24 24"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path d="m22 12-20 11v-22l10 5.5z"/>
  </svg>
  <svg
    class="splide__toggle__pause"
    viewBox="0 0 24 24"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path d="m2 1v22h7v-22zm13 0v22h7v-22z"/>
  </svg>
</button>
```

The docs example wraps the track in `<div class="splide__slider">`, which is a v3 wrapper. v4 doesn't need it, but it is harmless.

Behaviour (from the source):

| Aspect | Value |
|---|---|
| Lookup | `root.querySelector( '.splide__toggle' )`, excluding nested carousels. It must be inside the root but not inside the track or list. |
| `aria-controls` | The track id, for example `splide01-track`. Set only when `autoplay` is truthy. |
| `aria-label` | `i18n.pause` ("Pause autoplay") while autoplay is on; `i18n.play` ("Start autoplay") while it is stopped. |
| Class `is-active` | Present while autoplay is on, including while it is only paused by hover or focus. Removed when it is stopped: by `autoplay: 'pause'` at start, a toggle click, `Autoplay.pause()`, or reduced motion. |
| Click | If stopped, `play()`; otherwise `pause()`, which stops it until `play()` is called again. |

Visibility CSS (core and themes, verbatim):

```css
.splide__toggle.is-active .splide__toggle__play,.splide__toggle__pause{display:none}
.splide__toggle.is-active .splide__toggle__pause{display:inline}
```

While autoplay is on (`is-active`), the Pause element is shown. While it is stopped, the Play element is shown.

Docs say: "While autoplay is playing, `<span class="splide__toggle__play">` will be visible, and while it's paused, `<span class="splide__toggle__pause">` will be." Source CSS says the opposite: `__pause` shows while playing and `__play` shows while stopped.

The theme only adds `.splide__toggle { cursor: pointer; }` and focus outlines (`:focus-visible` and `.splide.is-focus-in .splide__toggle:focus`, `3px solid #0bf`). Position and size the button yourself, for example:

```css
.splide__toggle {
  position: absolute; right: 1rem; bottom: 1rem; z-index: 1;
  display: inline-flex; align-items: center; justify-content: center;
  width: 2.5rem; height: 2.5rem; padding: 0; border: 0; border-radius: 50%;
  background: rgba(0, 0, 0, .6); color: #fff;
}
.splide__toggle svg { width: 1rem; height: 1rem; fill: currentColor; }
.splide__toggle:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
```

## Progress bar

To display the progress bar, add this HTML:

```html
<div class="splide">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>

  <div class="splide__progress">
    <div class="splide__progress__bar">
    </div>
  </div>
</div>
```

With `splide-core.min.css`, style the bar yourself to make it visible:

```css
.splide__progress__bar {
  height: 3px;
  background: #ccc;
}
```

- Only `.splide__progress__bar` is looked up, inside the root. `.splide__progress` is a styling wrapper.
- On every animation frame, Splide sets an inline `width: {rate * 100}%` on the bar. Core CSS sets `.splide__progress__bar { width: 0; }`; the theme adds `background: #ccc; height: 3px;`.
- By default, the progress is rewound whenever autoplay is interrupted. Set `resetProgress: false` to keep the elapsed time.
- The bar tracks the autoplay timer, not the carousel position. For a position bar (slide 3 of 9), use your own class names so Autoplay doesn't drive it (see the "Carousel Progress" tutorial and the SKILL.md progress recipe).

## Per-slide interval

`interval` sets one duration for every slide. `data-splide-interval` overrides it for specific slides:

```html
<li class="splide__slide" data-splide-interval="1000"></li>
<li class="splide__slide" data-splide-interval="10000"></li>
```

From the source:

- The attribute is read on `move`, for the slide being moved to. The value is `+attr || options.interval`, so a missing, `0` or non-numeric value falls back to `interval`.
- The first cycle after mount uses `interval`, not the start slide's attribute, because no `move` has happened yet. If the first slide needs its own duration, set `interval` to that value and give every other slide its own `data-splide-interval`.

## Autoplay component methods

`splide.Components.Autoplay` is available after `mount()`.

| Method | Signature | Description |
|---|---|---|
| `play()` | `play(): void` | Starts autoplay. Nothing happens if it is already running or there are not enough slides. Clears the hover, focus and stopped flags, so it runs even while hovered or focused until the next mouseenter or focusin. Resumes the elapsed time when `resetProgress: false`. Emits `autoplay:play`. |
| `pause()` | `pause(): void` | Pauses autoplay and keeps it paused until `play()` is called; hover and focus changes won't resume it. Emits `autoplay:pause` if it was running. |
| `isPaused()` | `isPaused(): boolean` | `true` if autoplay is paused, otherwise `false`. It is also `true` during a temporary hover or focus pause, and before autoplay first starts. |

Docs say `pause(): void`. Source says `pause( stop = true )`. `pause( false )` is the temporary pause used for hover and focus, and autoplay resumes on mouseleave or focusout.

`destroy()` cancels the timer.

## Events

| Event | Arguments | Fired |
|---|---|---|
| `autoplay:play` | none | When autoplay starts: every successful `play()`, including resumes after hover or focus. |
| `autoplay:pause` | none | Right after autoplay is paused, including hover and focus pauses. Only fired if it was running. |
| `autoplay:playing` | `rate: number` | On every animation frame while the timer is ticking. `rate` is the current progress from 0 to 1. |

```js
var splide = new Splide( '.splide' );

splide.on( 'autoplay:playing', function ( rate ) {
  console.log( rate ); // 0-1
} );

splide.mount();
```

Live region: while autoplay plays, Splide sets the track's `aria-live` to `off` (on `autoplay:play`), and back to `polite` on `autoplay:pause`. Screen readers therefore don't keep announcing slides. This needs `live: true`, the default, and is never active on an `isNavigation` carousel.

## Pause on hover and focus

- `pauseOnHover` listens to `mouseenter` / `mouseleave` on the root, and `pauseOnFocus` to `focusin` / `focusout` on the root. Both are temporary pauses: autoplay resumes when the pointer leaves or focus leaves, unless it was stopped.
- Clicking an arrow or a dot usually focuses that button. With `pauseOnFocus`, autoplay then stays paused until focus leaves the carousel. This is intended, per W3C.
- Controls outside the root (custom buttons) don't count as hover or focus inside the carousel.

## Reduced motion

When Splide detects `(prefers-reduced-motion: reduce)`, it merges `reducedMotion`, which defaults to `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }`:

- `speed` and `rewindSpeed` become `0`, so the active slide appears without a transition.
- `autoplay` becomes `'pause'`, so autoplay doesn't start after page load. The toggle shows its Play state ("Start autoplay"), and the user can still start it.
- Transitions made by drag and swipe keep their normal speed. They count as essential motion that tells the user what is happening.

You can emulate the preference in Chrome DevTools: Rendering tab, "Emulate CSS media feature prefers-reduced-motion".

Don't rely only on autoplay to change slides unless the carousel is purely decorative. Users with reduced motion would otherwise have no way to rotate it. Keep arrows or pagination.

From the source: `options.reducedMotion` is deep-merged with the default, so passing `reducedMotion: { autoplay: true }` keeps `speed: 0` and `rewindSpeed: 0`. Because Autoplay reads `autoplay` once at mount, a preference change after page load does not start or stop autoplay.

## Accessibility (WCAG 2.2.1, 2.2.2, 2.3.3)

- WCAG 2.2.2 Pause, Stop, Hide: "For any auto-updating information that starts automatically and is presented in parallel with other content, there is a mechanism for the user to pause, stop, or hide it."
- WCAG 2.2.1 Timing Adjustable: "The user is allowed to turn off the time limit before encountering it."
- W3C says a carousel with autoplay must have a button to pause and resume it, should pause when focus enters the carousel, and should pause while the mouse hovers over it.
- Splide's `.splide__toggle` implements the first, using the Toggle Button pattern. `pauseOnHover` and `pauseOnFocus` default to `true` and cover the other two.
- WCAG 2.3.3 Animation from Interactions is covered by `reducedMotion`, above.

Checklist:

1. Include a `.splide__toggle` button inside the root, with `type="button"`.
2. Keep `pauseOnHover` and `pauseOnFocus` at `true`.
3. Keep manual navigation (arrows or pagination).
4. Translate `i18n.play` / `i18n.pause`, and the toggle's visible text, for non-English pages.
5. Give the toggle visible focus styles (themes include them; with core CSS add `:focus-visible` plus `.splide.is-focus-in .splide__toggle:focus`).

## Start autoplay only when in view

Use the Intersection extension to start autoplay only while the carousel is in the viewport. It uses `IntersectionObserver`, so IE is not supported without a polyfill. Script-tag install:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js"></script>
<script>
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
  } ).mount( window.splide.Extensions );
</script>
```

- The UMD script registers `window.splide.Extensions.Intersection`. Note the lowercase `splide` namespace, which is separate from the `Splide` constructor. `mount( window.splide.Extensions )` mounts every extension script loaded on the page. `mount( { Intersection: window.splide.Extensions.Intersection } )` mounts only this one.
- Leave out `outView` if autoplay should keep running after the carousel leaves the viewport.

| `intersection` key | Type | Description |
|---|---|---|
| `inView` | `{ autoplay?, keyboard?, autoScroll?, video?: boolean }` | Applied when the carousel enters or is in the viewport. `autoplay` starts or pauses autoplay. |
| `outView` | same | Applied when the carousel leaves the viewport. |
| `root` | `Element \| Document \| null` | The viewport element for `IntersectionObserver`. |
| `rootMargin` | `string` | Margin around the root, for example `'200px'`. |
| `threshold` | `number \| number[]` | `IntersectionObserver` threshold(s). |
| `once` | `boolean` | Stop observing once the carousel has become visible. |

Docs say (Intersection page): `threshold` has the type `gap: { row?: number | string, col?: number | string }`, which is a copy error. Source says it is passed straight to `IntersectionObserver` as `threshold`.

Extension events: `intersection` (`entry: IntersectionObserverEntry`, fired whenever the carousel crosses the threshold), `intersection:in` (`entry`) and `intersection:out` (`entry`).

Caveat (extension source, 0.2.0): `inView.autoplay: true` calls `Autoplay.play()` whenever `options.autoplay` is truthy, and `'pause'` is truthy. So it:

- restarts autoplay each time the carousel re-enters the viewport, even after the user pressed the pause toggle, and
- starts autoplay for users with `prefers-reduced-motion: reduce`, whose `autoplay` was set to `'pause'`.

If that matters, use the plain `IntersectionObserver` recipe below instead of the extension.

## Recipes

### In-view autoplay that respects the user's pause and reduced motion (no extension)

```html
<section id="hero" class="splide" aria-label="Highlights">
  <div class="splide__track"><ul class="splide__list">...</ul></div>
  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>

<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    var splide = new Splide( '#hero', { type: 'loop', autoplay: 'pause', interval: 6000 } );
    splide.mount();

    var Autoplay = splide.Components.Autoplay;
    var toggle   = splide.Components.Elements.toggle;
    // Should autoplay run while visible? Off by default under reduced motion.
    var wanted   = ! window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

    if ( toggle ) {
      // Splide's own click handler was bound during mount() and runs first,
      // so isPaused() already reflects the user's choice here.
      toggle.addEventListener( 'click', function () {
        wanted = ! Autoplay.isPaused();
      } );
    }

    if ( 'IntersectionObserver' in window ) {
      var observer = new IntersectionObserver( function ( entries ) {
        var entry = entries[ 0 ];
        if ( ! entry ) return;

        if ( entry.isIntersecting ) {
          if ( wanted ) Autoplay.play();
        } else {
          Autoplay.pause();
        }
      }, { threshold: 0.5 } );

      observer.observe( splide.root );
      splide.on( 'destroy', function () { observer.disconnect(); } );
    }
  } );
</script>
```

### Custom toggle outside the root

```js
var splide = new Splide( '#hero', { type: 'loop', autoplay: true } );
var button = document.querySelector( '.hero-toggle' ); // <button type="button" class="hero-toggle">

splide.mount();

var Autoplay = splide.Components.Autoplay;
var stopped  = Autoplay.isPaused(); // true for 'pause', reduced motion, or too few slides

function render() {
  var i18n = splide.options.i18n;
  button.setAttribute( 'aria-label', stopped ? i18n.play : i18n.pause );
  button.setAttribute( 'aria-controls', splide.Components.Elements.track.id );
  button.classList.toggle( 'is-active', ! stopped );
}

button.addEventListener( 'click', function () {
  stopped ? Autoplay.play() : Autoplay.pause();
  stopped = Autoplay.isPaused();
  render();
} );

render();
```

The built-in `stopped` flag is not public, and `isPaused()` is also `true` during hover and focus pauses. That is why the recipe keeps its own flag instead of listening to `autoplay:pause`.

### Circular or text progress from `autoplay:playing`

```js
var ring = document.querySelector( '.autoplay-ring' );
var splide = new Splide( '.splide', { type: 'loop', autoplay: true } );

splide.on( 'autoplay:playing', function ( rate ) {
  ring.style.setProperty( '--progress', rate ); // use in CSS, e.g. conic-gradient(... calc(var(--progress) * 360deg) ...)
} );

splide.mount();
```

## Pitfalls

- Autoplay without a `.splide__toggle` fails WCAG 2.2.2. Always add one, or an equivalent custom button.
- In a `slide` carousel without `rewind: true`, autoplay stops visibly at the last page.
- `splide.options = { autoplay: false }` and runtime breakpoint changes don't stop autoplay. Use `Autoplay.pause()`.
- The toggle and the progress bar must be inside the root. Splide does not find them outside it.
- The first slide's `data-splide-interval` is ignored for the first cycle.
- Under reduced motion, autoplay starts paused by design. Don't force `play()` on load.
- The Intersection extension's `inView.autoplay` overrides the user's pause and reduced motion (see above).
- The docs describe the toggle icon visibility the wrong way round. Trust the CSS: the pause icon shows while playing.
