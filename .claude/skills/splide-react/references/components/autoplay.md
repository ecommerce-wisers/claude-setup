# Autoplay component (React)

`splide.Components.Autoplay` as reached from React Splide: `play()`, `pause()`, `isPaused()`, the toggle button and progress bar in JSX, and the autoplay event props.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [Examples](#examples) · [Built-in toggle button and progress bar](#built-in-toggle-button-and-progress-bar) · [Per-slide interval](#per-slide-interval) · [When autoplay is false](#when-autoplay-is-false) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/autoplay/
Source file: `src/js/components/Autoplay/Autoplay.ts` (plus `src/js/components/Autoplay/constants.ts`, and the timer in `src/js/constructors/RequestInterval/RequestInterval.ts`) in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: runs autoplay. It drives the optional progress bar and play/pause toggle button, and pauses on hover and focus.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Autoplay;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.Autoplay.pause() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()`, which cancels the timer, and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Autoplay` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

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
- Clears the hovered, focused and stopped flags, updates the toggle button, and emits `autoplay:play` (React prop `onAutoplayPlay`).
- Each time the interval completes, the component calls `splide.go( '>' )`. In a `slide` carousel without `rewind`, `go('>')` does nothing at the last page, but the timer keeps running.

### pause()

Docs say `pause(): void`; source says `pause( stop?: boolean ): void` with `stop` defaulting to `true` (the TypeScript interface still declares `pause(): void`, so `pause( false )` is a type error in TypeScript). The `stop` parameter is (undocumented, from source).

- `stop = true` (what you get with `pause()`): autoplay stays paused until `play()` is called or the toggle button is clicked. Hover and focus changes do not resume it.
- `stop = false`: a temporary pause, the same kind that hover and focus use. The next `mouseleave` or `focusout` resumes autoplay.
- Updates the toggle button. If the timer was running, it pauses the timer and emits `autoplay:pause` (React prop `onAutoplayPause`). If it was already paused, no event is emitted.

### isPaused()

`isPaused(): boolean`

Returns the timer's paused flag. It is `true` before the first `play()`, after `pause()`, and while paused by hover or focus. It is not React state: reading it during render gives a value that does not update. Mirror it into state from `onAutoplayPlay` / `onAutoplayPause`, as below.

## Examples

External play/pause buttons, a status line and a custom progress bar:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function AutoplayControlsExample() {
  const ref = useRef<Splide>( null );
  const bar = useRef<HTMLDivElement>( null );
  const [ playing, setPlaying ] = useState( false );

  // Read the component when it is needed: it does not exist during the first render.
  const getAutoplay = () => ref.current?.splide?.Components.Autoplay;

  const toggle = () => {
    const Autoplay = getAutoplay();

    if ( Autoplay ) {
      Autoplay.isPaused() ? Autoplay.play() : Autoplay.pause();
    }
  };

  return (
    <>
      <Splide
        ref={ ref }
        options={ { type: 'loop', autoplay: true, interval: 3000, label: 'Autoplay controls' } }
        onAutoplayPlay={ () => setPlaying( true ) }
        onAutoplayPause={ () => setPlaying( false ) }
        onAutoplayPlaying={ ( splide, rate ) => {
          // Fires on every animation frame: write to the DOM, not to state.
          if ( bar.current ) {
            bar.current.style.width = `${ rate * 100 }%`;
          }
        } }
      >
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <div className="my-progress">
        <div className="my-progress-bar" ref={ bar } style={ { height: 4, background: 'teal' } } />
      </div>

      <button type="button" id="play" onClick={ () => getAutoplay()?.play() }>Play</button>
      <button type="button" id="pause" onClick={ () => getAutoplay()?.pause() }>Pause</button>
      <button type="button" id="toggle" onClick={ toggle }>Toggle</button>
      <p id="status">{ playing ? 'Playing' : 'Paused' }</p>
    </>
  );
}
```

Tested: the status starts at `Playing` (the `onAutoplayPlay` prop is bound before mount, so it sees the first `play()`), the buttons switch it, and the bar stops while paused.

- The setter from `useState` never changes, so calling it from an event prop is safe even though event props are captured at mount (see `references/reactivity.md`).
- The bar's `style` prop sets only `height` and `background`, so React never touches the `width` that the handler writes.

Start paused and let the user start it: `options={ { autoplay: 'pause' } }`. A `.splide__toggle` button or `Autoplay.play()` starts it.

## Built-in toggle button and progress bar

Both are optional and are found inside the root by class, so render them with `hasTrack={ false }`:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function AutoplayMarkupExample() {
  return (
    <Splide hasTrack={ false } options={ { type: 'loop', autoplay: true, interval: 3000, label: 'Autoplay' } }>
      <SplideTrack>
        <SplideSlide data-splide-interval="8000">Slide 1 (8 s)</SplideSlide>
        <SplideSlide>Slide 2</SplideSlide>
        <SplideSlide>Slide 3</SplideSlide>
      </SplideTrack>

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

Tested: the toggle gets `is-active`, `aria-label="Pause autoplay"` and `aria-controls="<root id>-track"`; the bar width grows; a click stops autoplay and switches the label to `Start autoplay`.

- Progress bar (`Elements.bar`, `.splide__progress__bar`): on every animation frame, its inline `width` is set to `rate * 100 + '%'`. Do not give it a `style` prop with `width`, or React and Splide fight over it.
- Toggle (`Elements.toggle`, `.splide__toggle`): gets `aria-controls` set to the track id. It has class `is-active` while autoplay is not stopped, and `aria-label` set to `i18n.pause` while active or `i18n.play` while stopped. A click calls `play()` if stopped, otherwise `pause( true )`. The core CSS shows `.splide__toggle__pause` when the toggle has `is-active`, and `.splide__toggle__play` otherwise. Keep its `className` constant: a `className` change makes React rewrite the attribute and drop `is-active`.
- `is-active` on the toggle reflects the stopped flag. A temporary hover or focus pause does not remove it.
- The toggle is taken from `Elements.toggle` once, when the component is created (from source). Keep it rendered; a toggle that React adds later is ignored until the instance is mounted again (a new `key`, or a `destroy` breakpoint that stops matching). The bar is read from `Elements.bar` on every frame, and Elements collects it again on each `refresh`.

## Per-slide interval

Add `data-splide-interval="<ms>"` to a slide. `<SplideSlide>` passes attributes straight to the `<li>`. On each `move`, the timer duration is set to the destination slide's attribute value, or `options.interval` when it is missing or invalid.

```tsx
<SplideSlide data-splide-interval="8000">...</SplideSlide>
```

## When autoplay is false

`mount()` does nothing when `options.autoplay` is falsy. `play()` still starts the timer and advances slides. The rest is off, because it is only set up in `mount()`: hover and focus pausing, the toggle button binding, progress rewinding on move, and per-slide `data-splide-interval`.

## Events emitted / listened

Emitted:

| Event | React prop | Handler arguments (prop) | When |
|---|---|---|---|
| `autoplay:play` | `onAutoplayPlay` | `( splide )` | `play()` actually started the timer. |
| `autoplay:pause` | `onAutoplayPause` | `( splide )` | A running timer was paused (by `pause()`, hover, focus or the toggle). |
| `autoplay:playing` | `onAutoplayPlaying` | `( splide, rate )` | Every animation frame while running. `rate` goes from 0 to 1. Also fires with `0` when progress is rewound (on `move`, `scroll`, `refresh`), even while paused. |

Unmounting destroys the instance and cancels the timer without emitting `autoplay:pause`.

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

- `autoplay` is read once, when the component is created during mount. Breakpoint values that match at mount time apply. Changing `autoplay` later through the `options` prop or a breakpoint change does not start or stop autoplay (tested: switching the prop from `true` to `false` left it running). Call `play()` / `pause()` instead, or remount with a `key`.
- `pauseOnHover` and `pauseOnFocus` are read once in `mount()`.
- `interval` is read at creation, and again on every `move`. A new `interval` from the `options` prop therefore takes effect from the next move.
- `resetProgress` is read on each `play()`.
