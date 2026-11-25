# Autoplay

Scope: autoplay in React Splide. Options, how it advances, the play/pause toggle and progress bar in JSX, per-slide `data-splide-interval`, the Autoplay component, the autoplay event props and React state that mirrors them, pause on hover and focus, reduced motion, accessibility, and autoplay only while in view.

Checked against the core 4.1.4 source (`components/Autoplay/Autoplay.ts`, `components/Autoplay/constants.ts`, `constructors/RequestInterval/RequestInterval.ts`, `components/Media/Media.ts`, `components/Live/Live.ts`), React Splide 0.7.12 (`src/js/components/Splide/Splide.tsx` and the official `examples/src/js/components/AutoplayExample.tsx`), `@splidejs/splide-extension-intersection` 0.2.0 (`dist/js`), https://splidejs.com/guides/autoplay/ and https://splidejs.com/components/autoplay/. Every example was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [Setup](#setup)
- [Options](#options)
- [How autoplay advances](#how-autoplay-advances)
- [Play/pause toggle and progress bar in JSX](#playpause-toggle-and-progress-bar-in-jsx)
- [Per-slide interval](#per-slide-interval)
- [Autoplay component methods](#autoplay-component-methods)
- [Events](#events)
- [React state that mirrors autoplay](#react-state-that-mirrors-autoplay)
- [Pause on hover and focus](#pause-on-hover-and-focus)
- [Reduced motion](#reduced-motion)
- [Accessibility (WCAG 2.2.1, 2.2.2, 2.3.3)](#accessibility-wcag-221-222-233)
- [Start autoplay only when in view](#start-autoplay-only-when-in-view)
- [Pitfalls](#pitfalls)

## Setup

```sh
npm install @splidejs/react-splide
npm install @splidejs/splide-extension-intersection   # only for "autoplay only in view"
```

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The types need one tsconfig line under `moduleResolution: "bundler"`; see `references/typescript.md`. Autoplay itself is one option: `options={ { type: 'loop', autoplay: true, label: '...' } }`. The rest of this page is about doing it accessibly.

## Options

Pass these through the `options` prop.

| Option | Type | Default | Notes |
|---|---|---|---|
| `autoplay` | `boolean \| 'pause'` | `false` | `true` starts autoplay on mount. `'pause'` mounts the feature without starting it; start it with a toggle button or `Autoplay.play()`. With `false`, the Autoplay component does nothing, and a toggle button or progress bar in the JSX stays inert. With the Intersection extension, autoplay can run only while the carousel is in the viewport. |
| `interval` | `number` | `5000` | Autoplay interval in milliseconds. Override it per slide with `data-splide-interval`. |
| `pauseOnHover` | `boolean` | `true` | Pauses autoplay on mouseover of the root. It should stay `true` for accessibility. |
| `pauseOnFocus` | `boolean` | `true` | Pauses autoplay while the carousel contains the focused element. It should stay `true` for accessibility. |
| `resetProgress` | `boolean` | `true` | Whether to reset the progress when autoplay is asked to start again. With `false`, the elapsed time is kept. |
| `reducedMotion` | `Options` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Merged into the options when `(prefers-reduced-motion: reduce)` matches. |
| `i18n.play` | `string` | `'Start autoplay'` | Toggle button `aria-label` while autoplay is stopped. |
| `i18n.pause` | `string` | `'Pause autoplay'` | Toggle button `aria-label` while autoplay is on. |

Runtime changes (from the source):

- `autoplay`, `pauseOnHover` and `pauseOnFocus` are read once, when the component mounts. A new `options` prop with `autoplay: false`, or a breakpoint that changes them after mount, does not start or stop autoplay. Call `Autoplay.play()` / `Autoplay.pause()` instead, or remount the carousel with a `key` (`references/reactivity.md`).
- Breakpoint values that apply at load do take effect, because breakpoints are merged before the Autoplay component is created.
- `interval` is re-read on every move, so a changed `interval` applies from the next slide.

## How autoplay advances

From `Autoplay.ts` and `RequestInterval.ts`:

- The timer runs on `requestAnimationFrame`. Each time it reaches `interval`, it calls `splide.go( '>' )`, the next page, which respects `perPage`, `perMove` and `focus`.
- In `type: 'slide'` without `rewind`, `go( '>' )` does nothing at the end while the timer keeps cycling. Use `rewind: true` or `type: 'loop'`.
- Every `move`, `scroll` and `refresh` rewinds the progress to 0. That includes moves made by arrows, pagination, drag and the API, and the `refresh()` React Splide calls when the slides change.
- `play()` only starts if there are more slides (clones included) than `perPage` (`Slides.isEnough()`). With too few slides, autoplay never starts.
- On unmount, `splide.destroy()` cancels the timer. Under StrictMode the first instance is destroyed before the second one starts, so only one timer runs.

## Play/pause toggle and progress bar in JSX

If you enable autoplay, also provide a play/pause button for accessibility; WCAG requires it. Render `hasTrack={ false }` so you can place a button with the class `splide__toggle` and a `.splide__progress__bar` inside the root, outside `<SplideTrack>`. This follows the official React example, with `type="button"` added and per-slide intervals:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/autoplay-toggle.css';

const options: Options = {
  label   : 'Highlights',
  type    : 'loop',
  autoplay: true,
  interval: 3000,
  height  : '12rem',
};

const slides = [
  { id: 1, text: 'Short one', interval: 1500 },
  { id: 2, text: 'Default interval' },
  { id: 3, text: 'Long read', interval: 8000 },
];

export default function Example() {
  return (
    <Splide tag="section" options={ options } hasTrack={ false }>
      <div className="autoplay-stage">
        <SplideTrack>
          { slides.map( slide => (
            // Read on `move` for the slide being moved to; missing or 0 falls back to `interval`
            <SplideSlide key={ slide.id } data-splide-interval={ slide.interval }>
              { slide.text }
            </SplideSlide>
          ) ) }
        </SplideTrack>
      </div>

      <div className="splide__progress">
        <div className="splide__progress__bar" />
      </div>

      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

`css/autoplay-toggle.css`:

```css
.autoplay-stage { position: relative; }
.splide__toggle {
  display: inline-flex; align-items: center; justify-content: center;
  margin-top: .5rem; padding: .25rem .75rem; border: 0; border-radius: 4px;
  background: #222; color: #fff;
}
.splide__toggle:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
```

Tested: on mount the toggle has `is-active`, `aria-label="Pause autoplay"` and `aria-controls` ending in `-track`, the Pause text is shown and Play is hidden, the bar's inline width grows, and the track's `aria-live` is `off`. After a click: no `is-active`, "Start autoplay", Play shown, `aria-live="polite"`.

- `.autoplay-stage` gives the theme's pagination (absolutely positioned at the bottom of the track's parent) a box of its own. Without it, the dots sit over the bottom of the root and can cover a toggle placed after the track (tested: a real mouse click on such a toggle didn't reach it until the track was wrapped).
- You can use other inline elements instead of `<span>`, such as `<img>` and `<svg>`, as long as they carry the `splide__toggle__play` / `splide__toggle__pause` classes.
- Leave the toggle's `aria-label`, `aria-controls` and `is-active` class to Splide; don't set them as props.

Behaviour (from the source):

| Aspect | Value |
|---|---|
| Lookup | `root.querySelector( '.splide__toggle' )`, excluding nested carousels. It must be inside the root but not inside the track or list. |
| `aria-controls` | The track id, for example `splide01-track`. Set only when `autoplay` is truthy. |
| `aria-label` | `i18n.pause` ("Pause autoplay") while autoplay is on; `i18n.play` ("Start autoplay") while it is stopped. |
| Class `is-active` | Present while autoplay is on, including while it is only paused by hover or focus. Removed when it is stopped: by `autoplay: 'pause'` at start, a toggle click, `Autoplay.pause()`, or reduced motion. With `autoplay: true` and too few slides to play, the toggle still gets `is-active` and "Pause autoplay" although nothing plays. |
| Click | If stopped, `play()`; otherwise `pause()`, which stops it until `play()` is called again. |

Visibility CSS (core and themes, verbatim):

```css
.splide__toggle.is-active .splide__toggle__play,.splide__toggle__pause{display:none}
.splide__toggle.is-active .splide__toggle__pause{display:inline}
```

While autoplay is on (`is-active`), the Pause element is shown. While it is stopped, the Play element is shown.

Docs say: "While autoplay is playing, `<span class="splide__toggle__play">` will be visible, and while it's paused, `<span class="splide__toggle__pause">` will be." Source CSS says the opposite: `__pause` shows while playing and `__play` shows while stopped.

The theme only adds `.splide__toggle { cursor: pointer; }` and focus outlines (`:focus-visible` and `.splide.is-focus-in .splide__toggle:focus`, `3px solid #0bf`). Position and size the button yourself.

Progress bar:

- Only `.splide__progress__bar` is looked up, inside the root. `.splide__progress` is a styling wrapper.
- On every animation frame, Splide sets an inline `width: {rate * 100}%` on the bar. Core CSS sets `.splide__progress__bar { width: 0; }`; the theme adds `background: #ccc; height: 3px;`. With `@splidejs/react-splide/css/core`, style the bar yourself (`height: 3px; background: #ccc;`).
- React doesn't render a `style` on the bar, so it never overwrites Splide's inline width. Don't give the bar a `style` prop.
- By default, the progress is rewound whenever autoplay is interrupted. Set `resetProgress: false` to keep the elapsed time.
- The bar tracks the autoplay timer, not the carousel position. For a position bar (slide 3 of 9), use your own class names so Autoplay doesn't drive it.

## Per-slide interval

`interval` sets one duration for every slide. `data-splide-interval` on `<SplideSlide>` overrides it for specific slides; attributes on `<SplideSlide>` pass straight to the `<li>`. In the toggle example above, `data-splide-interval={ slide.interval }` renders `data-splide-interval="1500"` on the first slide and no attribute when the value is `undefined`.

From the source:

- The attribute is read on `move`, for the slide being moved to. The value is `+attr || options.interval`, so a missing, `0` or non-numeric value falls back to `interval`.
- The first cycle after mount uses `interval`, not the start slide's attribute, because no `move` has happened yet (tested: with `interval: 3000` and `1500` on the first slide, the carousel was still on the first slide after 2.2 seconds). If the first slide needs its own duration, set `interval` to that value and give every other slide its own `data-splide-interval`.

## Autoplay component methods

`ref.current?.splide?.Components.Autoplay` is available after mount (in effects and handlers, not during render).

| Method | Signature | Description |
|---|---|---|
| `play()` | `play(): void` | Starts autoplay. Nothing happens if it is already running or there are not enough slides. Clears the hover, focus and stopped flags, so it runs even while hovered or focused until the next mouseenter or focusin. Resumes the elapsed time when `resetProgress: false`. Emits `autoplay:play`. |
| `pause()` | `pause(): void` | Pauses autoplay and keeps it paused until `play()` is called; hover and focus changes won't resume it. Emits `autoplay:pause` if it was running. |
| `isPaused()` | `isPaused(): boolean` | `true` if autoplay is paused, otherwise `false`. It is also `true` during a temporary hover or focus pause, and before autoplay first starts. |

Docs say `pause(): void`. Source says `pause( stop = true )`. `pause( false )` is the temporary pause used for hover and focus, and autoplay resumes on mouseleave or focusout.

## Events

React Splide has an event prop for each autoplay event. Each handler gets the core instance first.

| Prop | Core event | Arguments | Fired |
|---|---|---|---|
| `onAutoplayPlay` | `autoplay:play` | `splide` | When autoplay starts: every successful `play()`, including resumes after hover or focus. |
| `onAutoplayPause` | `autoplay:pause` | `splide` | Right after autoplay is paused, including hover and focus pauses. Only fired if it was running. |
| `onAutoplayPlaying` | `autoplay:playing` | `splide`, `rate: number` | On every animation frame while the timer is ticking. `rate` is the current progress from 0 to 1. It also fires once with `0` each time the progress is rewound (`move`, `scroll`, `refresh`), even while autoplay is paused or stopped. |

- The props are bound in `componentDidMount` before `splide.mount()`. With `autoplay: true`, `autoplay:play` fires during mount, and `onAutoplayPlay` catches it.
- Live region: while autoplay plays, Splide sets the track's `aria-live` to `off` (on `autoplay:play`), and back to `polite` on `autoplay:pause`. Screen readers therefore don't keep announcing slides. This needs `live: true`, the default, and is never active on an `isNavigation` carousel.

## React state that mirrors autoplay

To show the state outside the carousel (a custom toggle, a status text, a circular progress), mirror it in React state from the event props. This example has its own toggle outside the root, a conic-gradient progress ring, a pause counter, and an `onPause` callback from the parent:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/autoplay-state.css';

const options: Options = {
  label   : 'Highlights',
  type    : 'loop',
  autoplay: true,
  interval: 4000,
  height  : '10rem',
};

const slides = [ 'One', 'Two', 'Three' ];

type HeroProps = { onPause?: ( index: number ) => void };

function HeroCarousel( { onPause }: HeroProps ) {
  const ref  = useRef<Splide>( null );
  const ring = useRef<HTMLDivElement>( null );
  const [ running, setRunning ] = useState( false ); // ticking now (false during hover and focus pauses too)
  const [ stopped, setStopped ] = useState( false ); // stopped until the user presses play
  const [ pauses, setPauses ]   = useState( 0 );
  const [ trackId, setTrackId ] = useState<string>();

  // Event props are bound once, at mount: keep the latest callback in a ref.
  const onPauseRef = useRef( onPause );
  onPauseRef.current = onPause;

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( splide ) {
      setTrackId( splide.Components.Elements.track.id );
      setStopped( splide.Components.Autoplay.isPaused() ); // true for 'pause', reduced motion or too few slides
    }
  }, [] );

  function toggle() {
    const Autoplay = ref.current?.splide?.Components.Autoplay;

    if ( Autoplay ) {
      stopped ? Autoplay.play() : Autoplay.pause();
      setStopped( Autoplay.isPaused() );
    }
  }

  return (
    <>
      <Splide
        ref={ ref }
        tag="section"
        options={ options }
        onAutoplayPlay={ () => setRunning( true ) }
        onAutoplayPause={ splide => {
          setRunning( false );
          setPauses( n => n + 1 );              // functional update: never reads stale state
          onPauseRef.current?.( splide.index ); // latest prop, not the one from the first render
        } }
        onAutoplayPlaying={ ( splide, rate ) => {
          // Every animation frame: write to the DOM instead of setting state
          ring.current?.style.setProperty( '--progress', String( rate ) );
        } }
      >
        { slides.map( text => <SplideSlide key={ text }>{ text }</SplideSlide> ) }
      </Splide>

      <div className="hero-controls">
        <div ref={ ring } className="autoplay-ring" aria-hidden="true" />
        <button
          type="button"
          className={ stopped ? 'hero-toggle' : 'hero-toggle is-active' }
          aria-label={ stopped ? 'Start autoplay' : 'Pause autoplay' }
          aria-controls={ trackId }
          onClick={ toggle }
        >
          { stopped ? 'Play' : 'Pause' }
        </button>
        <span className="hero-status">{ running ? 'Playing' : 'Paused' } ({ pauses } pauses)</span>
      </div>
    </>
  );
}

export default function Example() {
  const [ section, setSection ] = useState( 'home' );
  const [ log, setLog ] = useState<string[]>( [] );

  return (
    <>
      <button type="button" className="section" onClick={ () => setSection( 'sale' ) }>Go to sale</button>
      <HeroCarousel onPause={ index => setLog( old => [ ...old, `${ section }:${ index }` ] ) } />
      <ol className="pause-log">{ log.map( ( entry, i ) => <li key={ i }>{ entry }</li> ) }</ol>
    </>
  );
}
```

`css/autoplay-state.css`:

```css
.hero-controls { display: flex; align-items: center; gap: .75rem; margin-top: .5rem; }
.autoplay-ring {
  width: 1.5rem; height: 1.5rem; border-radius: 50%;
  background: conic-gradient( #222 calc( var( --progress, 0 ) * 360deg ), #ddd 0 );
}
```

Tested: on load the toggle reads "Pause autoplay" with `aria-controls` ending in `-track`, the status is "Playing (0 pauses)" and `--progress` grows. After "Go to sale" and a hover over the carousel, the status was "Paused (1 pauses)", the toggle still read "Pause autoplay" (a hover pause is not a stop) and the log showed `sale:0`. Leaving resumed it; the toggle stopped it ("Start autoplay", "Play") and started it again.

Why it is written this way:

- **Two flags.** `running` follows the events, so it is `false` during hover and focus pauses too. `stopped` is the user's choice and drives the toggle. The built-in `stopped` flag is not public, and `isPaused()` is also `true` during hover and focus pauses, which is why the toggle keeps its own state instead of listening to `onAutoplayPause`. `isPaused()` right after mount is `true` for `autoplay: 'pause'`, reduced motion, or too few slides.
- **Stale handlers.** React Splide reads every event prop once, at mount, and never again. A handler that reads state or props directly keeps the first render's values. Tested: an `onAutoplayPause={ () => setSeen( section ) }` handler still reported `home` after `section` had changed to `sale`. Use a functional update (`setPauses( n => n + 1 )`) for state, and a ref for anything else (`onPauseRef.current = onPause` on every render). `setState` functions never change, so `setRunning( true )` needs no ref. More in `references/reactivity.md`.
- **No state per frame.** `onAutoplayPlaying` fires on every animation frame. Setting state there re-renders the component (and runs React Splide's `componentDidUpdate`) about 60 times a second. Write to the DOM through a ref instead, here as a CSS custom property.
- **The instance only exists after mount.** `ref.current?.splide` is `undefined` during the first render, so the track id and the initial `isPaused()` are read in an effect.
- **Outside the root.** Hover and focus on these external controls don't pause autoplay, because they are outside the root. Keep a built-in `.splide__toggle` inside the root when you can.

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

1. Include a `.splide__toggle` button inside the root (`hasTrack={ false }`), with `type="button"`.
2. Keep `pauseOnHover` and `pauseOnFocus` at `true`.
3. Keep manual navigation (arrows or pagination).
4. Translate `i18n.play` / `i18n.pause`, and the toggle's visible text, for non-English pages.
5. Give the toggle visible focus styles (themes include them; with core CSS add `:focus-visible` plus `.splide.is-focus-in .splide__toggle:focus`).
6. Label the carousel with `options.label` or `options.labelledby`, not an `aria-label` prop, which StrictMode strips in development (`references/reactivity.md`).

## Start autoplay only when in view

### With the Intersection extension

The extension uses `IntersectionObserver`, so IE is not supported without a polyfill. Pass it through the `extensions` prop, which is read once at mount:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';
import './css/autoplay-toggle.css'; // .autoplay-stage { position: relative; } plus toggle styles

// Read once, at mount. Keep it outside the component (or in useMemo).
const extensions = { Intersection };

const options: Options = {
  label       : 'Highlights',
  type        : 'loop',
  autoplay    : 'pause',
  intersection: {
    inView : { autoplay: true },
    outView: { autoplay: false },
  },
};

export default function Example() {
  return (
    <Splide
      tag="section"
      options={ options }
      extensions={ extensions }
      hasTrack={ false }
    >
      <div className="autoplay-stage">
        <SplideTrack>
          <SplideSlide>One</SplideSlide>
          <SplideSlide>Two</SplideSlide>
          <SplideSlide>Three</SplideSlide>
        </SplideTrack>
      </div>

      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

Tested: in view the toggle read "Pause autoplay"; scrolled out of view, "Start autoplay"; back in view, playing again. The toggle styles and `.autoplay-stage` come from `css/autoplay-toggle.css` above.

- `intersection` is not in the `Options` type as its own key. `Options` accepts any extra key, so it compiles, but it isn't type-checked.
- `extensions` is spread onto the root as an attribute (`extensions="[object Object]"`). Harmless.
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

Extension events: `intersection` (`entry: IntersectionObserverEntry`, fired whenever the carousel crosses the threshold), `intersection:in` (`entry`) and `intersection:out` (`entry`). React Splide has no props for them. Listen with `splide.on()` in an effect, or in your own extension.

Caveat (extension source, 0.2.0): `inView.autoplay: true` calls `Autoplay.play()` whenever `options.autoplay` is truthy, and `'pause'` is truthy. So it:

- restarts autoplay each time the carousel re-enters the viewport, even after the user pressed the pause toggle (tested: stopped with the toggle, scrolled out and back, and it was playing again), and
- starts autoplay for users with `prefers-reduced-motion: reduce`, whose `autoplay` was set to `'pause'`.

If that matters, use the plain `IntersectionObserver` recipe below instead of the extension.

### Without the extension (respects the user's pause and reduced motion)

```tsx
import { useEffect, useRef } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/autoplay-toggle.css'; // .autoplay-stage { position: relative; } plus toggle styles

const options: Options = {
  label   : 'Highlights',
  type    : 'loop',
  autoplay: 'pause',
  interval: 6000,
};

export default function Example() {
  const ref = useRef<Splide>( null );
  // Should autoplay run while visible? Off by default under reduced motion.
  const wanted = useRef( true );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( ! splide || ! ( 'IntersectionObserver' in window ) ) {
      return;
    }

    const { Autoplay } = splide.Components;
    wanted.current = ! window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

    const observer = new IntersectionObserver( ( [ entry ] ) => {
      if ( ! entry ) {
        return;
      }

      if ( entry.isIntersecting ) {
        wanted.current && Autoplay.play();
      } else {
        Autoplay.pause();
      }
    }, { threshold: 0.5 } );

    observer.observe( splide.root );

    return () => {
      observer.disconnect();
    };
  }, [] );

  return (
    <Splide ref={ ref } tag="section" options={ options } hasTrack={ false }>
      {/* The theme's pagination is absolute at the bottom of the track's parent: keep it off the toggle */}
      <div className="autoplay-stage">
        <SplideTrack>
          <SplideSlide>One</SplideSlide>
          <SplideSlide>Two</SplideSlide>
          <SplideSlide>Three</SplideSlide>
        </SplideTrack>
      </div>

      <button
        className="splide__toggle"
        type="button"
        onClick={ () => {
          // Splide's own click listener is on the button and runs before React's
          // delegated onClick, so isPaused() already reflects the user's choice.
          wanted.current = ! ref.current?.splide?.Components.Autoplay.isPaused();
        } }
      >
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

The toggle styles and `.autoplay-stage` come from `css/autoplay-toggle.css` above.

Tested: plays in view, pauses out of view, plays again on re-entry; after a click on the toggle it stays stopped through a scroll out and back; with `prefers-reduced-motion: reduce` emulated, it did not start.

- The effect runs after mount, with the instance that exists then. Under StrictMode the first instance's observer is disconnected by the cleanup before the second instance gets its own.
- `wanted` is a ref, not state: the observer callback is created once, and a ref always holds the current value.

## Pitfalls

- Autoplay without a `.splide__toggle` fails WCAG 2.2.2. Always add one (with `hasTrack={ false }`), or an equivalent custom button.
- In a `slide` carousel without `rewind: true`, autoplay stops visibly at the last page.
- Changing `autoplay` in the `options` prop after mount does nothing. Use `Autoplay.play()` / `pause()` through the ref, or remount with a `key`.
- The toggle and the progress bar must be inside the root. Splide does not find them outside it.
- With the default `hasTrack`, a toggle or progress bar passed as a child of `<Splide>` ends up inside the slide list.
- The first slide's `data-splide-interval` is ignored for the first cycle.
- Under reduced motion, autoplay starts paused by design. Don't force `play()` on load.
- Event props are captured at mount: a handler that reads state directly sees the first render's values. Use functional updates or refs.
- Setting state in `onAutoplayPlaying` re-renders on every animation frame. Write to the DOM through a ref.
- The Intersection extension's `inView.autoplay` overrides the user's pause and reduced motion (see above).
- The docs describe the toggle icon visibility the wrong way round. Trust the CSS: the pause icon shows while playing.
