# Drag component (React)

`splide.Components.Drag` as reached from React Splide: `disable()`, `isDragging()`, the drag event props, and how to keep drag locked across option updates.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [disable()](#disable) · [Locking drag from state](#locking-drag-from-state) · [isDragging()](#isdragging) · [Drag behaviour](#drag-behaviour-from-source) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/drag/
Source file: `src/js/components/Drag/Drag.ts` (constants in `src/js/components/Drag/constants.ts`) in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: watches mouse and touch input on the track, moves the carousel while the user drags, and decides where to go on release (flick).

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Drag;
```

- In an event prop, the core instance is the first argument: `onMove={ ( splide ) => splide.Components.Drag.isDragging() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects (and a new, enabled Drag), so read `splide.Components.Drag` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `disable( disabled )` | `void` | `true` disables dragging, `false` enables it. Reset from `options.drag` on every `mounted`/`updated` event. |
| `isDragging()` | `boolean` | (undocumented, from source) `true` while the user is actually dragging the carousel. |

Lifecycle member `mount()` is also on the object. Splide calls it.

### disable()

`disable( disabled: boolean ): void`. The docs params table names the argument `value`; the interface names it `disabled`.

| Param | Description |
|---|---|
| `disabled` | `true` to disable the component, `false` to enable it. |

Behaviour from source:

- While disabled, `mousedown`/`touchstart` on the track are ignored, and the click blocker that suppresses clicks after a drag is off.
- The listeners stay bound. `drag: false` also mounts the component and only sets the disabled flag.
- The flag is overwritten by `disable( ! options.drag )` on every `mounted` and `updated` event. In React every `options` prop change that is not deep-equal emits `updated`, and so do breakpoint changes. Either undoes a manual `disable()`.

## Locking drag from state

Persistent lock: put it in the options. React Splide sets `splide.options`, the `updated` event runs `disable( ! options.drag )`, and the root loses `splide--draggable`:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DragOptionExample() {
  const [ zoomed, setZoomed ] = useState( false );
  const [ drags, setDrags ] = useState( 0 );

  return (
    <>
      <Splide
        options={ { drag: ! zoomed, height: 150, label: 'Drag option demo' } }
        onDrag={ () => setDrags( n => n + 1 ) }
      >
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="zoom" onClick={ () => setZoomed( z => ! z ) }>{ zoomed ? 'Zoom out' : 'Zoom in' }</button>
      <p id="drags">{ drags }</p>
    </>
  );
}
```

Tested with a real mouse drag: no `drag` event while zoomed, one after zooming out.

Manual lock with `disable()`: re-apply it after every `updated` event. The listener must run after the Drag component's own `updated` listener, so add it with `splide.on()` in an effect. An `onUpdated` prop does not work here: event props are bound before `splide.mount()`, so they run before the component's listener, which then resets the flag (tested: the carousel was draggable again after an options change).

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DragLockExample() {
  const ref = useRef<Splide>( null );
  const [ locked, setLocked ] = useState( false );
  const [ gap, setGap ] = useState( 0 );
  const [ drags, setDrags ] = useState( 0 );
  const [ source, setSource ] = useState( '' );

  const lockedRef = useRef( locked );
  lockedRef.current = locked;

  // Apply the flag whenever it changes.
  useEffect( () => {
    ref.current?.splide?.Components.Drag.disable( locked );
  }, [ locked ] );

  // Every 'updated' event resets the flag from options.drag. Re-apply it from a listener
  // added after mount, so it runs after the Drag component's own listener.
  useEffect( () => {
    const splide = ref.current?.splide;

    splide?.on( 'updated', () => {
      if ( lockedRef.current ) {
        splide.Components.Drag.disable( true );
      }
    } );
  }, [] );

  return (
    <>
      <Splide
        ref={ ref }
        options={ { gap, height: 150, label: 'Drag demo' } }
        onDrag={ () => setDrags( n => n + 1 ) }
        onMove={ ( splide ) => setSource( splide.Components.Drag.isDragging() ? 'drag' : 'other' ) }
      >
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="lock" onClick={ () => setLocked( l => ! l ) }>{ locked ? 'Unlock' : 'Lock' }</button>
      <button type="button" id="gap" onClick={ () => setGap( g => g + 10 ) }>Change gap</button>
      <button type="button" id="next" onClick={ () => ref.current?.go( '>' ) }>Next</button>
      <p id="drags">{ drags }</p>
      <p id="source">{ source }</p>
    </>
  );
}
```

Tested with real mouse drags, in production and under StrictMode: a drag counts and `source` reads `drag`; the Next button gives `other`; after Lock and Change gap, a drag does nothing; after Unlock it counts again.

- The listener reads `lockedRef.current`, because a function registered once keeps the values of the render it was created in (see `references/reactivity.md`).
- No cleanup is needed for unmount: `destroy()` removes every listener. Don't call `splide.off( 'updated' )` as a cleanup: `off()` removes every handler added with `splide.on()` for that event on the instance, including the ones React Splide added for the event props (tested with `off( 'moved' )`: `onMoved` stopped firing).
- Under StrictMode the carousel mounts, is destroyed and mounts again, and the effects run again with the second instance, so the listener ends up on the live one.

## isDragging()

(undocumented, from source) `isDragging(): boolean`

Returns the internal `dragging` flag:

- It becomes `true` once the pointer has moved past `dragMinThreshold` along the slider axis, and further along that axis than across it.
- It is also `true` right after pointer-down if the carousel was already moving or scrolling (the user grabbed a moving carousel).
- It goes back to `false` on pointer-up.
- It is `undefined` (falsy) before the first pointer-down.

The release move is started synchronously before the flag is cleared. Inside a `move` handler (`onMove`), or in free mode a `scroll` handler (`onScroll`), `isDragging()` therefore tells whether that move came from a drag. By the time `moved` fires, or a `click` handler runs, it is already `false`. The `source` state in the example above uses this.

## Drag behaviour (from source)

- Start: `touchstart`/`mousedown` on the track. For mouse, only the primary button (`e.button === 0`). Move and up listeners go on the track for touch and on `window` for mouse. All pointer listeners use `{ passive: false, capture: true }`.
- Ignored targets: `.splide__pagination__page`, `.splide__arrow`, and elements matching the `noDrag` selector. The match uses `Element.matches()` on the event target itself, not its ancestors, so use a selector that also covers descendants, e.g. `noDrag: '.no-drag, .no-drag *'`.
- If `Controller.isBusy()` is `true` on pointer-down (only possible with `waitForTransition: true`), the event is cancelled with `preventDefault()` and `stopPropagation()`, and no drag starts.
- On pointer-down, any running transition (`Move.cancel()`) and scroll (`Scroll.cancel()`) are stopped where they are.
- The first pointer-move sets the state to `DRAGGING` (`Splide.STATES.DRAGGING`, 6) and emits `drag`, even before the threshold is reached.
- While dragging, the list follows the pointer and `dragging` is emitted on each move. Past the edges of a `slide` type carousel, the movement is divided by 5 (friction).
- Release: if the state was `DRAGGING`, it is set to `IDLE` and `dragged` is emitted. If a drag actually happened, the destination is computed:
  - Velocity is taken over the last 200 ms. A pause of 200 ms or more while holding gives 0 velocity. Velocity is also 0 past the edges, except for `loop`.
  - Destination is `position + sign( v ) * min( |v| * flickPower, listSize * flickMaxPages )`. `flickPower` defaults to 600 and `flickMaxPages` to 1. There is no page cap in free mode.
  - `drag: 'free'`: `Controller.scroll( destination, 0, options.snap )`.
  - `fade` type: `go( '<' / '>' )` when `rewind` and `rewindByDrag` are set, otherwise `go( '-' / '+' )`.
  - `slide` type past an edge with `rewind` and `rewindByDrag`: `go( '>' )` past the end, `go( '<' )` past the start.
  - Otherwise: `Controller.go( Controller.toDest( destination ), true )`.
  - The release move ignores `reducedMotion` (it temporarily removes the reduced-motion overrides), so the snap animation still plays for those users.
- After a real drag, `click` events inside the track are cancelled in the capture phase, so links and buttons in slides do not fire on release. Propagation stops at the track, so a React `onClick` inside a slide does not fire after a drag either, while an `onClickCapture` handler still does (tested).
- Native `dragstart` (image/link dragging) on the track is always prevented.

## Events emitted / listened

Emitted:

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `drag` | `onDrag` | `( splide )` | First pointer-move after pointer-down (state becomes `DRAGGING`). |
| `dragging` | `onDragging` | `( splide )` | Every pointer-move while the carousel follows the pointer. Fires often: avoid a state update on each call. |
| `dragged` | `onDragged` | `( splide )` | Pointer-up after a `drag` (state returns to `IDLE`). |

The release then calls `Controller.go()` or `Controller.scroll()`, which leads to `move`/`moved` (`onMove`, `onMoved`) or `scroll`/`scrolled` (`onScroll`, `onScrolled`).

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `mounted`, `updated` | `disable( ! options.drag )`, and re-reads free mode (`drag === 'free'`). |
| DOM track | `touchstart`, `mousedown` | Start of a drag. |
| DOM track or window | `touchmove`, `mousemove` | Drag update. |
| DOM track or window | `touchend`, `touchcancel`, `mouseup`, `click` | Drag end. |
| DOM track | `click` (capture) | Cancelled after a drag. |
| DOM track | `dragstart` | Always prevented. |

## Options read

| Option | Default | Use |
|---|---|---|
| `drag` | `true` | `false` disables, `true` enables, `'free'` is free-drag mode (scroll with momentum). Re-read on every `updated`, so it can change through the `options` prop. |
| `snap` | (none) | In free mode, snap to the closest slide after the flick. |
| `dragMinThreshold` | `10` | Number: touch threshold in px (mouse threshold 0). Object `{ mouse, touch }`: separate thresholds (missing touch falls back to 10, missing mouse to 0). |
| `flickPower` | `600` | Velocity multiplier for the flick distance. |
| `flickMaxPages` | `1` | Max flick distance in list widths (not used in free mode). |
| `noDrag` | (none) | Selector of targets that cannot start a drag. |
| `rewind`, `rewindByDrag` | (none) | Rewind when dragging past the edges (`slide` and `fade`). |
| `type` | `'slide'` | Friction and limit behaviour (`slide`), wrapping (`loop`), step behaviour (`fade`). |
| `direction` | `'ltr'` | Axis and sign (through Direction). |
| `waitForTransition` | (none) | Through `Controller.isBusy()`. |
| `reducedMotion` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Temporarily removed for the release move. |
