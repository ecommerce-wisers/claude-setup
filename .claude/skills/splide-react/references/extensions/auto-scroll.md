# Auto Scroll extension (React)

Continuous scrolling with `@splidejs/splide-extension-auto-scroll` in React Splide: setup, options, methods, the toggle button, and React-specific behavior.

Source: https://splidejs.com/extensions/auto-scroll/
Package: @splidejs/splide-extension-auto-scroll@0.5.3 (peer: none declared; built against @splidejs/splide ^4.0.17, its devDependency; tested with React Splide 0.7.12 and React 19)

Contents: [Install](#install-and-mount) · [Markup](#markup-and-the-toggle-button) · [Options](#options) · [Methods](#methods) · [Events](#events) · [Examples](#examples) · [React notes](#react-notes) · [Caveats](#caveats)

The extension scrolls the carousel continuously, a few pixels per frame, instead of moving one slide at a time. Use it with `type: 'loop'` and `drag: 'free'`. It also works with the Intersection extension, so scrolling can start only while the carousel is in the viewport (see `references/extensions/intersection.md`).

## Install and mount

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-auto-scroll
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import '@splidejs/react-splide/css';

<Splide options={ { label: 'Partner logos', type: 'loop', drag: 'free', autoScroll: { speed: 1 } } } extensions={ { AutoScroll } }>
  ...
</Splide>
```

- The package ships no CSS. The named export is `AutoScroll`; the type `AutoScrollComponent` is exported too. Package details: `references/extensions/packages.md`.
- Keep the key `AutoScroll`. The Intersection extension looks the component up as `splide.Components.AutoScroll`, and a key that matches a core component name (such as `Scroll`) replaces that core component.
- `extensions` is read once, at mount.

## Markup and the toggle button

The extension needs no special markup.

It can also drive the core play/pause toggle button. This needs `useToggleButton: true`, which the docs page does not mention (it is in the typings and the source). The button must be inside the root, so render the track yourself with `hasTrack={ false }` and `<SplideTrack>`:

```tsx
<Splide hasTrack={ false } options={ options } extensions={ { AutoScroll } }>
  <SplideTrack>
    <SplideSlide>...</SplideSlide>
  </SplideTrack>
  <button className="splide__toggle" type="button">
    <span className="splide__toggle__play">Play</span>
    <span className="splide__toggle__pause">Pause</span>
  </button>
</Splide>
```

While it scrolls, the button has `is-active` and `aria-label="Pause auto scroll"`. While it is paused, `is-active` is removed and the label is `"Start auto scroll"`. The core CSS (`@splidejs/react-splide/css` or `/css/core`) shows `.splide__toggle__play` when the button is not active and `.splide__toggle__pause` when it is. The extension updates the button's class and label whenever a `.splide__toggle` exists, but it binds the click only when `useToggleButton` is `true`. Don't turn on core `autoplay` on the same carousel, because core Autoplay also binds `.splide__toggle`.

An `onClick` on the button runs after the extension's own click listener (React handles events at the root), so it can read the new state with `isPaused()` (see the second example).

## Options

Pass the options as `autoScroll` in the `options` prop:

```tsx
<Splide options={ { autoScroll: { speed: 2 } } } extensions={ { AutoScroll } }>
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

- `autoScroll: false` disables the extension for that carousel. Changing the prop to `autoScroll: false` (or setting it inside `breakpoints`) destroys the scroller. Changing it back to an object mounts it again (tested through state).
- A changed `options` prop reaches the extension: React Splide sets `splide.options = options`, the extension merges the new `autoScroll` values on `updated`, and a new `speed` applies right away (tested). The hover, focus and toggle listeners are bound only when the scroller mounts (at mount, or when it is turned back on after `autoScroll: false`), so changing `pauseOnHover`, `pauseOnFocus` or `useToggleButton` while it runs has no effect.
- i18n keys (in the core `i18n` option) for the toggle's `aria-label`: `startScroll` (default `"Start auto scroll"`) and `pauseScroll` (default `"Pause auto scroll"`).
- TypeScript checks these options once the package is imported (`speed: 'fast'` is an error). See `references/extensions/packages.md` for deriving an `AutoScrollOptions` type.

## Methods

The component is at `splide.Components.AutoScroll`, under the key you passed in `extensions`. In React, reach it through a ref (`ref.current?.splide?.Components.AutoScroll`) or the instance passed to an event prop. It is `undefined` before mount and after unmount.

| Method | Signature | Description |
|---|---|---|
| `play()` | `play(): void` | Starts scrolling, or resumes it. It also clears the hover and focus pause flags and disables the ARIA live region while scrolling (`Components.Live.disable( true )`). When the scroller is not mounted (`type: 'fade'`, or `autoScroll: false`), it throws a `TypeError` (checked), so guard the call. |
| `pause()` | `pause(): void` | Pauses auto scroll and marks it as stopped. After a manual `pause()`, leaving the carousel with the mouse or ending a drag does not resume it; only `play()` does. It also turns the live region back on. |
| `isPaused()` | `isPaused(): boolean` | Returns `true` if auto scroll is paused, otherwise `false`. It also returns `true` when the scroller never mounted (`type: 'fade'` or `autoScroll: false`). |

```tsx
const ref = useRef<Splide>( null );

function pause(): void {
  ref.current?.splide?.Components.AutoScroll?.pause();
}
```

## Events

The extension emits no events of its own and does not emit `autoplay:*`, so there is nothing to subscribe to and no prop to pass. When React state must follow the scroller, read `isPaused()` right after you call a method or after the toggle button's click (see below). It listens to core events:

- `move`, `drag` and `scroll` pause scrolling while an arrow click, a drag or a `go()` runs. `moved`, `dragged` and `scrolled` resume it, unless `pause()` was called or the pointer or focus is still inside.
- `updated` merges new `autoScroll` options (see above).

While it scrolls, it updates the index whenever a different slide reaches the active position. That updates `is-active` and the pagination, fires the core `active`/`inactive`/`visible`/`hidden` events (so `onActive` and friends fire), and runs the lazy loader when `lazyLoad: 'nearby'`. It does not fire `move` or `moved` for these index changes, so `onMove`/`onMoved` stay silent. Since 0.5.3 the arrows' disabled state is updated at most once every 500 ms.

## Examples

### Infinite logo ticker

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import '@splidejs/react-splide/css';

const logos = [ 'Logo 01', 'Logo 02', 'Logo 03', 'Logo 04', 'Logo 05', 'Logo 06' ];

export default function LogoTicker() {
  return (
    <Splide
      tag="section"
      id="ticker"
      options={ {
        label     : 'Partner logos',
        height    : '6rem', // Splide overwrites inline slide heights, so size slides with options or CSS
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
      } }
      extensions={ { AutoScroll } }
    >
      { logos.map( logo => (
        <SplideSlide key={ logo } style={ { display: 'flex', alignItems: 'center', justifyContent: 'center', background: '#f2f2f2' } }>
          { logo }
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

The official demo uses `type: 'loop'`, `drag: 'free'`, `focus: 'center'`, `perPage: 3` and `autoScroll: { speed: 1 }`. A second demo uses `speed: -1` to scroll the other way. Tested: it scrolls, hovering pauses it, and leaving resumes it.

### Reverse direction, toggle button, own controls, translated labels, reduced motion

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import '@splidejs/react-splide/css';

// The extension ignores the reducedMotion option, so check the preference yourself.
const reduce = typeof window !== 'undefined'
  && window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

const extensions = { AutoScroll };
const items = [ 'Item 1', 'Item 2', 'Item 3', 'Item 4', 'Item 5' ];

export default function NewsTicker() {
  const ref = useRef<Splide>( null );
  const [ speed, setSpeed ] = useState( -0.5 );
  const [ enabled, setEnabled ] = useState( true );
  const [ paused, setPaused ] = useState( reduce );

  const options: Options = {
    label     : 'Neuigkeiten',
    height    : '8rem',
    type      : 'loop',
    drag      : 'free',
    perPage   : 2,
    gap       : '1rem',
    pagination: false,
    i18n      : {
      startScroll: 'Automatisches Scrollen starten',
      pauseScroll: 'Automatisches Scrollen pausieren',
    },
    autoScroll: enabled
      ? {
        speed,                  // negative: slides travel left to right (LTR); fractions are allowed
        autoStart      : ! reduce,
        pauseOnHover   : true,  // listeners are bound at mount; changing these later does nothing
        pauseOnFocus   : true,
        useToggleButton: true,
      }
      : false,                  // destroys the scroller; an object mounts it again
  };

  // Reads the state after the extension's own click handler (or a method call) ran.
  function sync(): void {
    const scroller = ref.current?.splide?.Components.AutoScroll;
    setPaused( scroller ? scroller.isPaused() : true );
  }

  function togglePlay(): void {
    const scroller = ref.current?.splide?.Components.AutoScroll;
    if ( ! scroller || ! enabled ) {
      return; // play() throws while the scroller is not mounted
    }
    if ( scroller.isPaused() ) {
      scroller.play();
    } else {
      scroller.pause();
    }
    sync();
  }

  function toggleEnabled(): void {
    setEnabled( value => ! value );
    setPaused( enabled || reduce );
  }

  return (
    <>
      <Splide ref={ ref } tag="section" id="news" hasTrack={ false } options={ options } extensions={ extensions }>
        <SplideTrack>
          { items.map( item => (
            <SplideSlide key={ item } style={ { background: '#e8eef7', display: 'grid', placeItems: 'center' } }>
              { item }
            </SplideSlide>
          ) ) }
        </SplideTrack>
        <button className="splide__toggle" type="button" onClick={ sync }>
          <span className="splide__toggle__play">Start</span>
          <span className="splide__toggle__pause">Pause</span>
        </button>
      </Splide>

      <p>
        <button id="play" type="button" onClick={ togglePlay } disabled={ ! enabled }>
          { paused ? 'Play' : 'Pause' }
        </button>
        <button id="faster" type="button" onClick={ () => setSpeed( value => value * 2 ) }>Faster</button>
        <button id="enabled" type="button" onClick={ toggleEnabled }>
          { enabled ? 'Turn auto scroll off' : 'Turn auto scroll on' }
        </button>
      </p>
      <p id="state">{ paused ? 'paused' : 'scrolling' } at speed { speed }</p>
    </>
  );
}
```

Tested in production and development: the German labels appear on the toggle, the own button pauses and resumes, the toggle button's state reaches React through `onClick`, "Faster" changes the speed through the `options` prop without a remount, and turning auto scroll off and on again stops and restarts it. The `paused` state reflects your own calls and the toggle; hovering or focusing the carousel pauses it without telling React.

### Start only when visible (with Intersection)

```tsx
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import { Intersection } from '@splidejs/splide-extension-intersection';

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

The full tested component (threshold, toggle button, reduced motion, visibility state) is the combined example in `references/extensions/packages.md`.

## React notes

- **SSR.** Importing the extension on the server is safe, and it runs only after mount. Guard browser-only reads such as `matchMedia` with `typeof window !== 'undefined'`, as above. The options are not rendered into the markup, so a server value that differs from the client value causes no hydration mismatch.
- **`autoStart` and `load`.** If `<Splide>` mounts before the window `load` event (a server-rendered page whose images are still loading), scrolling starts only after `load`. After client-side navigation the document is already complete, so it starts at mount.
- **StrictMode.** In development the carousel mounts, is destroyed and mounts again. The scroller restarts cleanly (both examples pass in development).
- **No re-render needed for speed.** Changing `speed` through state updates the running scroller. You don't need a `key`.
- **Slide sizes.** Splide sets each slide's inline `width` and `height` during layout, which overwrites `width` or `height` in `<SplideSlide style>`. Use the `height`, `fixedHeight` or `heightRatio` options, or a CSS class.
- **State that follows the scroller.** There are no events, so store what you need right after calling `play()`/`pause()`, and read `isPaused()` in an `onClick` on the toggle button.

## Caveats

- Giving a carousel the extension turns auto scroll on even without an `autoScroll` option. The scroller starts unless `autoScroll: false` or `type: 'fade'`. If several carousels share a page, pass `AutoScroll` only to the ones that should scroll.
- It does nothing with `type: 'fade'`. With `type: 'slide'` it stops at the end unless `autoScroll.rewind` is `true`. Use `type: 'loop'` for an endless ticker.
- `speed: 0` still scrolls (it is treated as `1`). Fractional values such as `0.5` work.
- `autoStart` waits for `window` `load`, so on a page with heavy images scrolling starts only after they finish loading.
- Moving the carousel (arrows, pagination, drag, `go()`) pauses scrolling and then resumes it. A manual `pause()` stays paused until `play()`.
- `play()` throws while the scroller is not mounted (`type: 'fade'`, `autoScroll: false`). Check before calling it.
- It ignores the core `reducedMotion` option and `prefers-reduced-motion`. Check `matchMedia( '(prefers-reduced-motion: reduce)' )` yourself and set `autoStart: false`, or `autoScroll: false`.
- Accessibility: content that moves for more than 5 seconds needs a way to pause it (WCAG 2.2.2). Provide the `.splide__toggle` button with `useToggleButton: true`, or your own buttons that call `pause()`/`play()`. The default `pauseOnHover`/`pauseOnFocus` help mouse and keyboard users, but not touch users.
