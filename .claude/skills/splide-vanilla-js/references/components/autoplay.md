# Autoplay component

Source: https://splidejs.com/components/autoplay/
Source file: `src/js/components/Autoplay/Autoplay.ts` (plus `src/js/components/Autoplay/constants.ts`, and the timer in `src/js/constructors/RequestInterval/RequestInterval.ts`), Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: runs autoplay. It drives the optional progress bar and play/pause toggle button, and pauses on hover and focus.

Access: `splide.Components.Autoplay` (filled in by `splide.mount()`).

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `play()` | `void` | Starts autoplay and clears all pause flags. Does nothing if autoplay is already running or there are not enough slides. |
| `pause()` | `void` | Pauses autoplay. Docs: `pause(): void`. Source implementation: `pause( stop = true )`, see below. |
| `isPaused()` | `boolean` | `true` if autoplay is paused (or never started), otherwise `false`. |

Lifecycle members `mount()` and `destroy()` are also on the object. `destroy` cancels the timer. Splide calls them; do not call them yourself.

### play()

`play(): void`

- Runs only when `isPaused()` is `true` **and** `Components.Slides.isEnough()` is `true` (more slides than `perPage`, counting clones). Otherwise it silently does nothing.
- Starts the requestAnimationFrame timer. With `resetProgress: true` (default), progress restarts from 0. With `resetProgress: false`, it resumes from where it was paused.
- Clears the hovered, focused and stopped flags, updates the toggle button, and emits `autoplay:play`.
- Each time the interval completes, the component calls `splide.go( '>' )`. In a `slide` carousel without `rewind`, `go('>')` does nothing at the last page, but the timer keeps running.

### pause()

Docs say `pause(): void`; source says `pause( stop?: boolean ): void` with `stop` defaulting to `true` (the TypeScript interface still declares `pause(): void`). The `stop` parameter is (undocumented, from source).

- `stop = true` (what you get with `pause()`): autoplay stays paused until `play()` is called or the toggle button is clicked. Hover and focus changes do not resume it.
- `stop = false`: a temporary pause, the same kind that hover and focus use. The next `mouseleave` or `focusout` resumes autoplay.
- Updates the toggle button. If the timer was running, it pauses the timer and emits `autoplay:pause`. If it was already paused, no event is emitted.

### isPaused()

`isPaused(): boolean`

Returns the timer's paused flag. It is `true` before the first `play()`, after `pause()`, and while paused by hover or focus.

### Examples

External play/pause buttons:

```js
const splide = new Splide( '.splide', {
  type    : 'loop',
  autoplay: true,
  interval: 3000,
} ).mount();

const Autoplay = splide.Components.Autoplay;

document.querySelector( '#play' ).addEventListener( 'click', function () {
  Autoplay.play();
} );

document.querySelector( '#pause' ).addEventListener( 'click', function () {
  Autoplay.pause(); // stays paused until play()
} );

document.querySelector( '#toggle' ).addEventListener( 'click', function () {
  Autoplay.isPaused() ? Autoplay.play() : Autoplay.pause();
} );
```

Custom progress bar driven by the event:

```js
const bar = document.querySelector( '.my-progress-bar' );

splide.on( 'autoplay:playing', function ( rate ) {
  bar.style.width = String( rate * 100 ) + '%';
} );
```

Start paused and let the user start it:

```js
new Splide( '.splide', { autoplay: 'pause' } ).mount(); // a .splide__toggle button or Autoplay.play() starts it
```

### Built-in toggle button and progress bar

Both are optional and are found inside the root by class:

```html
<section class="splide" aria-label="...">
  <div class="splide__track">
    <ul class="splide__list">...</ul>
  </div>

  <div class="splide__progress">
    <div class="splide__progress__bar"></div>
  </div>

  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>
```

- Progress bar (`Elements.bar`, `.splide__progress__bar`): on every animation frame, its inline `width` is set to `rate * 100 + '%'`.
- Toggle (`Elements.toggle`, `.splide__toggle`): gets `aria-controls` set to the track id. It has class `is-active` while autoplay is not stopped, and `aria-label` set to `i18n.pause` while active or `i18n.play` while stopped. A click calls `play()` if stopped, otherwise `pause( true )`. The core CSS shows `.splide__toggle__pause` when the toggle has `is-active`, and `.splide__toggle__play` otherwise.
- `is-active` on the toggle reflects the stopped flag. A temporary hover or focus pause does not remove it.

### Per-slide interval

Add `data-splide-interval="<ms>"` to a slide element. On each `move`, the timer duration is set to the destination slide's attribute value, or `options.interval` when it is missing or invalid.

```html
<li class="splide__slide" data-splide-interval="8000">...</li>
```

### When `autoplay` is false

`mount()` does nothing when `options.autoplay` is falsy. `play()` still starts the timer and advances slides. The rest is off, because it is only set up in `mount()`: hover and focus pausing, the toggle button binding, progress rewinding on move, and per-slide `data-splide-interval`.

## Events emitted / listened

Emitted:

| Event | Handler arguments | When |
|---|---|---|
| `autoplay:play` | `()` | `play()` actually started the timer. |
| `autoplay:pause` | `()` | A running timer was paused (by `pause()`, hover, focus or the toggle). |
| `autoplay:playing` | `( rate )` | Every animation frame while running. `rate` goes from 0 to 1. Also fires with `0` when progress is rewound (on `move`, `scroll`, `refresh`), even while paused. |

Listened (only when `options.autoplay` is truthy):

| Source | Event | Action |
|---|---|---|
| DOM root | `mouseenter` / `mouseleave` | Temporary pause / resume, when `pauseOnHover` is set. |
| DOM root | `focusin` / `focusout` | Temporary pause / resume, when `pauseOnFocus` is set. |
| DOM toggle | `click` | `play()` or `pause( true )`. |
| Splide | `move`, `scroll`, `refresh` | Rewinds the progress to 0, so the next slide gets a full interval. |
| Splide | `move` | Sets the interval from the destination slide's `data-splide-interval`. |

The Live component listens to `autoplay:play` / `autoplay:pause` to set the track's `aria-live` to `off` / `polite`.

## Options read

| Option | Default | Use |
|---|---|---|
| `autoplay` | (none, falsy) | `true` starts on mount. `'pause'` mounts but starts stopped. `false` disables `mount()`. |
| `interval` | `5000` | Interval in ms. Overridden per slide by `data-splide-interval`. |
| `pauseOnHover` | `true` | Pause while the pointer is over the root. |
| `pauseOnFocus` | `true` | Pause while focus is inside the root. |
| `resetProgress` | `true` | `true`: `play()` restarts progress. `false`: resumes it. |
| `i18n.play`, `i18n.pause` | `'Start autoplay'`, `'Pause autoplay'` | Toggle button `aria-label`. |
| `reducedMotion.autoplay` | `'pause'` | Merged into options by the Media component when `(prefers-reduced-motion: reduce)` matches, so autoplay starts stopped for those users. |
| `perPage` | `1` | Indirectly, through `Slides.isEnough()`. |

When each option is read:

- `autoplay` is read once, when the component is created during `mount()`. Breakpoint values that match at mount time apply. Changing `autoplay` later through `splide.options = { ... }` or a breakpoint change does not start or stop autoplay; call `play()` / `pause()` instead.
- `pauseOnHover` and `pauseOnFocus` are read once in `mount()`.
- `interval` is read at creation, and again on every `move`.
- `resetProgress` is read on each `play()`.
