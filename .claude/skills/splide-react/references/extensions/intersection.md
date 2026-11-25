# Intersection extension (React)

Viewport-driven behavior with `@splidejs/splide-extension-intersection` in React Splide: options, events, a safe custom root, and React-specific behavior.

Source: https://splidejs.com/extensions/intersection/
Package: @splidejs/splide-extension-intersection@0.2.0 (peer: none declared; built against @splidejs/splide ^4.0.0, its devDependency; tested with React Splide 0.7.12 and React 19)

Contents: [Install](#install-and-mount) · [Usage](#usage) · [Options](#options) · [Methods](#methods) · [Events](#events) · [Examples](#examples) · [React notes](#react-notes) · [Caveats](#caveats)

The extension watches the carousel root with an `IntersectionObserver` and turns Autoplay, Keyboard, AutoScroll and Video on or off when the carousel enters or leaves the viewport. It also emits events, so you can react to intersection changes yourself.

## Install and mount

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-intersection
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';

<Splide options={ options } extensions={ { Intersection } }>...</Splide>

// together with the components it controls (keep these exact keys):
<Splide options={ options } extensions={ { Intersection, AutoScroll, Video } }>...</Splide>
```

- The package ships no CSS and exports only `Intersection` (no types besides the augmentation of `Options`).
- The handlers look the components up by name: `Components.Autoplay`, `Components.Keyboard`, `Components.AutoScroll` and `Components.Video`. Register AutoScroll and Video under exactly those keys, or Intersection cannot find them. The order of the keys doesn't matter for this.
- The extension needs no markup. It observes the root element (`splide.root`).

## Usage

Pass the `intersection` option and define `inView`. This starts autoplay when the carousel enters the viewport:

```tsx
<Splide
  options={ {
    autoplay    : 'pause',
    intersection: {
      inView: {
        autoplay: true,
      },
    },
  } }
  extensions={ { Intersection } }
>
```

To pause autoplay again when the carousel leaves the viewport, add `outView`:

```tsx
intersection: {
  inView : { autoplay: true },
  outView: { autoplay: false },
},
```

## Options

All options go under `intersection`. The docs list no defaults. When an option is not set, the extension passes `undefined` to `IntersectionObserver`, which uses its own defaults. The options are read once, at mount: a changed `options` prop doesn't reach the observer (remount with a `key` to change them).

| Option | Type | Default | Description |
|---|---|---|---|
| `inView` | `IntersectionViewOptions` | not set | Options applied when the carousel enters, or is in, the viewport. |
| `outView` | `IntersectionViewOptions` | not set | Options applied when the carousel leaves the viewport. It takes the same properties as `inView`. |
| `root` | `Element \| Document \| null` | not set (the browser viewport) | The element used as the viewport. See MDN "Intersection Observer API: creating an intersection observer". **Don't pass it through the `options` prop**: React Splide deep-copies the options, and copying a React-rendered element recurses into React's internals ("Maximum call stack size exceeded", the whole app unmounts). Core's own option merge would also turn the element into a plain object. Use the wrapper in [Custom root](#custom-root-scrolling-container). |
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

A property that is missing or `undefined` is left alone. Any other key, for example a typo like `autoScoll`, throws a `TypeError` inside the observer callback. In TypeScript this typo is a compile error once the package is imported.

Enable the arrow keys only while the carousel is within 200 px of the viewport:

```tsx
options={ {
  keyboard: 'global', // the docs example omits this; without it there are no shortcuts to toggle
  intersection: {
    rootMargin: '200px',
    inView    : { keyboard: true },
    outView   : { keyboard: false },
  },
} }
```

## Methods

The extension has no public methods. `splide.Components.Intersection` exposes only the lifecycle functions `mount()` and `destroy()`. `destroy()` disconnects the observer, and React Splide's unmount (`splide.destroy()`) calls it for you.

## Events

There are no event props for these events. Register them with `splide.on()` in `onMounted`. It runs inside `mount()`, before the observer's first callback, which is asynchronous and fires right after mounting.

| Event | Arguments | Fired |
|---|---|---|
| `intersection` | `entry: IntersectionObserverEntry` | Whenever the carousel crosses the threshold, in either direction. |
| `intersection:in` | `entry: IntersectionObserverEntry` | Whenever the carousel crosses the threshold and intersects the viewport (`entry.isIntersecting === true`). |
| `intersection:out` | `entry: IntersectionObserverEntry` | Whenever the carousel leaves the viewport (`entry.isIntersecting === false`). |

For each observer callback, the order is: the `inView`/`outView` options are applied, then `intersection:in` or `intersection:out` fires, then `intersection`. With `once: true`, the observer disconnects after the first `intersection:in`. The `intersection` event still fires for that entry.

```tsx
<Splide
  extensions={ { Intersection } }
  onMounted={ splide => {
    splide.on( 'intersection', entry => setRatio( entry.intersectionRatio ) );
    splide.on( 'intersection:in', entry => console.log( 'in', entry.target ) );
    splide.on( 'intersection:out', entry => console.log( 'out', entry.target ) );
  } }
>
```

- `entry.target` is the carousel root. Other useful fields are `intersectionRatio`, `boundingClientRect`, `intersectionRect`, `rootBounds` and `time`.
- The handlers are typed (`entry` is `IntersectionObserverEntry`) because the package augments core's `EventMap`.
- `onMounted` is captured at mount. Calling a state setter is safe; for other changing values read a ref (`references/reactivity.md`).
- What `inView`/`outView` start and stop has core event props: `onAutoplayPlay` and `onAutoplayPause` follow autoplay (used below).

## Examples

### Autoplay only while visible

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';

const reduce = typeof window !== 'undefined'
  && window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

const slides = [ 'Slide 01', 'Slide 02', 'Slide 03' ];

export default function Promotions() {
  const [ playing, setPlaying ] = useState( false );
  const [ ratio, setRatio ] = useState( 0 );

  return (
    <>
      <div style={ { height: '120vh' } }>Scroll down</div>

      <Splide
        tag="section"
        id="promo"
        hasTrack={ false }
        options={ {
          label       : 'Promotions',
          type        : 'loop',
          height      : '12rem',
          autoplay    : 'pause',   // autoplay on, but waiting
          interval    : 4000,
          intersection: {
            threshold: 0.5,        // at least half visible
            inView   : { autoplay: ! reduce },
            outView  : { autoplay: false },
          },
        } }
        extensions={ { Intersection } }
        onMounted={ splide => {
          // Registered inside mount(), before the observer's first (asynchronous) callback.
          splide.on( 'intersection', entry => setRatio( entry.intersectionRatio ) );
        } }
        onAutoplayPlay={ () => setPlaying( true ) }
        onAutoplayPause={ () => setPlaying( false ) }
      >
        <SplideTrack>
          { slides.map( slide => (
            <SplideSlide key={ slide } style={ { background: '#ddd', display: 'grid', placeItems: 'center' } }>
              { slide }
            </SplideSlide>
          ) ) }
        </SplideTrack>
        <div className="splide__progress">
          <div className="splide__progress__bar" />
        </div>
        <button className="splide__toggle" type="button">
          <span className="splide__toggle__play">Play</span>
          <span className="splide__toggle__pause">Pause</span>
        </button>
      </Splide>

      <p id="status">
        { playing ? 'Autoplay running' : 'Autoplay paused' }, { Math.round( ratio * 100 ) }% visible
      </p>
      <div style={ { height: '120vh' } } />
    </>
  );
}
```

`inView: { autoplay: ! reduce }` matters. Under reduced motion, core `reducedMotion` sets `autoplay` to `'pause'`, which is truthy, so `inView: { autoplay: true }` would start autoplay anyway. Tested in production and development: paused and 0% at load, running and 100% when scrolled into view, paused again when scrolled away.

### Auto Scroll only while visible

```tsx
<Splide
  options={ {
    label       : 'Partner logos',
    type        : 'loop',
    drag        : 'free',
    perPage     : 3,
    autoScroll  : { speed: 1, autoStart: false },
    intersection: {
      inView : { autoScroll: true },
      outView: { autoScroll: false },
    },
  } }
  extensions={ { AutoScroll, Intersection } }
>
```

`outView` calls `AutoScroll.pause()`, which marks the scroller as stopped, so hovering does not restart it off-screen. `inView` calls `play()`. The complete tested component, with a threshold, the toggle button and reduced motion, is the combined example in `references/extensions/packages.md`.

### Play the active video when visible, once

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css';
import { SafeVideo } from './SafeVideo';

// Keep the keys `Video` and `Intersection`: Intersection looks Video up by name.
const extensions = { Video: SafeVideo, Intersection };

export default function Showreel() {
  const [ status, setStatus ] = useState( 'Not started' );

  return (
    <>
      <div style={ { height: '120vh' } }>Scroll down</div>
      <Splide
        id="showreel"
        options={ {
          label       : 'Showreel',
          heightRatio : 0.5625,
          video       : { mute: true }, // browsers block unmuted playback without a user gesture
          intersection: {
            once  : true,
            inView: { video: true },
          },
        } }
        extensions={ extensions }
        onMounted={ splide => {
          splide.on( 'video:play', () => setStatus( 'Playing' ) );
          splide.on( 'video:pause', () => setStatus( 'Paused' ) );
        } }
      >
        <SplideSlide data-splide-html-video="/01.mp4">Clip 1</SplideSlide>
        <SplideSlide data-splide-html-video="/01.mp4">Clip 2</SplideSlide>
      </Splide>
      <p id="status">{ status }</p>
      <div style={ { height: '120vh' } } />
    </>
  );
}
```

`SafeVideo` is the Video extension wrapped to survive unmounting; its code and the reason are in `references/extensions/video.md`. Load the Video CSS as shown. Tested: the clip starts when scrolled into view, and keeps playing after scrolling away (`once`).

### Custom root (scrolling container)

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';

type ExtensionConstructor = NonNullable<SplideProps[ 'extensions' ]>[ string ];

// Never put an element in `options`: React Splide deep-copies the options, and copying a
// React-managed element recurses into React's internals ("Maximum call stack size exceeded").
// Hand the element to Intersection through a wrapper instead.
function intersectionWithRoot( root: Element ): ExtensionConstructor {
  return ( splide, components, options ) => Intersection( splide, components, {
    ...options,
    intersection: { ...options.intersection, root },
  } );
}

export default function PanelCarousel() {
  const [ panel, setPanel ] = useState<HTMLDivElement | null>( null );
  const [ playing, setPlaying ] = useState( false );

  return (
    <>
      <div ref={ setPanel } id="scroll-panel" style={ { height: 300, overflow: 'auto' } }>
        <div style={ { height: 600 } }>Scroll inside this panel</div>

        { /* Render the carousel once the panel exists; extensions are read only at mount. */ }
        { panel && (
          <Splide
            id="inside-panel"
            options={ {
              label       : 'Inside the panel',
              autoplay    : 'pause',
              intersection: {
                threshold: [ 0, 0.25, 0.5, 0.75, 1 ],
                inView   : { autoplay: true },
                outView  : { autoplay: false },
              },
            } }
            extensions={ { Intersection: intersectionWithRoot( panel ) } }
            onAutoplayPlay={ () => setPlaying( true ) }
            onAutoplayPause={ () => setPlaying( false ) }
          >
            <SplideSlide>Slide 01</SplideSlide>
            <SplideSlide>Slide 02</SplideSlide>
          </Splide>
        ) }
      </div>
      <p id="status">{ playing ? 'Autoplay running' : 'Autoplay paused' }</p>
    </>
  );
}
```

- The root must be an ancestor of the carousel. A callback ref (`ref={ setPanel }`) stores the element in state, and the carousel renders on the next pass. A `useRef` would still be `null` when `<Splide>` mounts, because React attaches a parent's ref after its children mount (tested: `null` in the extension constructor, also on StrictMode's second mount).
- The wrapper passes Intersection a copy of the options with `root` set. Intersection reads `options.intersection` only when it is created, so this is the only place the element needs to exist.
- Tested in production and development: `entry.rootBounds.height` was 300 (the panel, not the 800 px window), autoplay started after scrolling the panel and paused after scrolling back.
- With an array of thresholds, the callback, and the `inView` options with it, runs at each crossing while the carousel stays visible.

## React notes

- **Event timing.** `onMounted` runs inside `mount()`, so handlers registered there see every `intersection*` event, including the first one. A `useEffect` that reads `ref.current?.splide` caught the first event in tests too, but it runs after paint and is not guaranteed to win the race.
- **StrictMode.** In development the carousel mounts, is destroyed (which disconnects the observer) and mounts again. The second instance observes again and reports the current state, and `onMounted` registers the handlers on it (all examples pass in development).
- **SSR.** Nothing runs on the server. On the client, the first observer callback reports the real state right after mount, so start your React state with the "not visible" value and let the event correct it.
- **Option changes.** The `intersection` options are read once. A new `threshold` or `inView` from state has no effect until the carousel remounts (`key`). The core options that `inView`/`outView` toggle (`autoplay`, `keyboard`) are still read live by the handlers.
- **Elements in options.** Never put an element (or a ref's `current`) in the `options` prop; see the `root` row above.

## Caveats

- IE is not supported, because the extension uses Intersection Observer (install a polyfill if needed). Without `window.IntersectionObserver` it silently does nothing and emits no events.
- The observer reports the current state immediately after mount. A carousel that starts off-screen gets `outView` applied and `intersection:out` fired on page load.
- `inView: { autoplay: true }` needs the core `autoplay` option to be truthy. With `autoplay: false` nothing starts. With `autoplay: true`, autoplay starts at mount and is paused only when the first `outView` applies. Use `autoplay: 'pause'`.
- `keyboard` toggling needs the core `keyboard` option. It is off by default in v4.
- The `intersection` options are read once, when the extension mounts. Changing them through the `options` prop or `breakpoints` has no effect.
- An unknown key inside `inView` or `outView` throws a `TypeError` in the observer callback.
- When `once: true` is set, nothing is paused when the carousel later scrolls out of view.
- `inView: { autoScroll: true }` on a carousel where the scroller is not mounted (`type: 'fade'`, `autoScroll: false`) makes `AutoScroll.play()` throw inside the observer callback.
- It ignores `prefers-reduced-motion`. Guard `autoplay`, `autoScroll` and `video` yourself, as in the first example.
- Give each carousel only the extensions it needs. A shared object with every extension would start AutoScroll and let URL Hash rewrite the URL fragment on carousels that didn't configure them.
