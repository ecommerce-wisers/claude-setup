# Scroll component (React)

`splide.Components.Scroll` as reached from React Splide: animated scrolling to any position, with and without updating the index.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [scroll()](#scroll) · [cancel()](#cancel) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/scroll/
Source file: `src/js/components/Scroll/Scroll.ts` (constants in `src/js/components/Scroll/constants.ts`) in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: scrolls the carousel to any pixel position with requestAnimationFrame instead of the Transition component. Free drag mode (`drag: 'free'`), `Controller.scroll()` and the `useScroll` option use it.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Scroll;
```

- In an event prop, the core instance is the first argument: `onDragged={ ( splide ) => splide.Components.Scroll.cancel() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()`, which stops a running scroll without emitting `scrolled`, and clears `ref.current.splide`. A pending scroll callback never runs, so it cannot set state on an unmounted component.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Scroll` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `scroll( destination, duration?, snap?, callback? )` | `void` | Animates the list to a position. Unlike `Controller.scroll()`, it does not update the current/previous index. |
| `cancel()` | `void` | Stops a running scroll and emits `scrolled` (and runs its callback). Does nothing if no scroll is running. |

Lifecycle members `mount()` and `destroy()` are also on the object. `destroy` stops the timer without emitting. Splide calls them.

### scroll()

Docs say `scroll( position: number, duration?: number, snap?: boolean, callback?: AnyFunction ): void` (the params table calls the first argument `destination`). The source interface is the same. The implementation takes a fifth, internal argument: `scroll( destination, duration?, snap?, onScrolled?, noConstrain?: boolean )`.

| Param | Description |
|---|---|
| `destination` | Target position in px (see Move: negative values move an `ltr` carousel forward). Docs say "A position or an index to scroll to. The `useIndex` must be `true` to specify the index." Source: there is no `useIndex` in v4; the value is always a position. Convert an index with `splide.Components.Move.toPosition( index, true )`. |
| `duration` | Optional. Duration in ms. If omitted or `0`, it is `max( abs( destination - current ) / 1.5, 800 )`. If the distance is under 1px, the duration is `0`. |
| `snap` | Optional. Replaces the destination with the position of the closest slide (`Controller.toDest()`), keeping whole slider-length offsets. Skipped for a `slide` type carousel that is currently past its limits. |
| `callback` | Optional. Called when the scroll ends, after the state is set back to `IDLE` and before `scrolled` is emitted. |
| `noConstrain` | (undocumented, from source; internal) If `true`, no bounce at the edges. Used by the bounce-back scroll itself. |

Behaviour from source:

- Stops any previous scroll first, without emitting `scrolled` for it.
- Sets the state to `SCROLLING` (`Splide.STATES.SCROLLING`, 5), emits `scroll` (`onScroll`), and starts a one-shot timer.
- On each frame, the list moves toward `from + (to - from) * easing( rate )`. The easing is `options.easingFunc( t )` if given, otherwise easeOutQuart `1 - (1 - t)^4`.
- Bounce (`slide` type only): past the first or last limit, the movement is damped by a friction that multiplies by 0.6 each frame. When the per-frame step drops under 10px, a 600 ms scroll back to the nearest limit runs instead (same callback).
- Does not change `splide.index`, so arrows and pagination are not updated (pagination updates on `scroll`/`scrolled` but uses the unchanged index), and an index kept in React state from `onMoved` stays as it was. Use `Controller.scroll()` when the index should follow.
- A `move` (for example from `go()`) stops any running scroll silently: no `scrolled` event, no callback.

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function ScrollExample() {
  const ref = useRef<Splide>( null );
  const [ status, setStatus ] = useState( '' );

  const scrollTo = ( index: number, updateIndex: boolean ) => {
    const splide = ref.current?.splide;

    if ( splide ) {
      const { Scroll, Controller, Move } = splide.Components;
      const position = Move.toPosition( index, true );
      const done = () => setStatus( `index ${ splide.index }` );

      if ( updateIndex ) {
        Controller.scroll( position, 600, false, done ); // updates splide.index
      } else {
        Scroll.scroll( position, 600, false, done ); // leaves splide.index alone
      }
    }
  };

  return (
    <>
      <Splide ref={ ref } options={ { label: 'Scroll demo' } } onScrolled={ () => setStatus( s => `${ s } scrolled` ) }>
        { [ 1, 2, 3, 4, 5 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="scroll" onClick={ () => scrollTo( 2, false ) }>Scroll to slide 3</button>
      <button type="button" id="controller" onClick={ () => scrollTo( 4, true ) }>Scroll to slide 5 and update the index</button>
      <p id="status">{ status }</p>
    </>
  );
}
```

Tested in production and under StrictMode: `Scroll.scroll()` to slide 3 ends with `index 0 scrolled`, `Controller.scroll()` to slide 5 with `index 4 scrolled`. The order shows the callback runs before `scrolled`. `onScrolled` uses a functional update because the handler is captured at mount.

### cancel()

`cancel(): void`

If a scroll is running: stops the timer, sets the state to `IDLE`, calls the pending callback and emits `scrolled`. Otherwise it does nothing. Drag calls it on pointer-down. The component calls it on `updated` and `refresh`, so in React an `options` prop change or a change in the slide children during a scroll ends it early (with `onScrolled` and the callback).

## Events emitted / listened

Emitted:

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `scroll` | `onScroll` | `( splide )` | Each `scroll()` call (including the internal bounce-back scroll). |
| `scrolled` | `onScrolled` | `( splide )` | When a scroll ends or is cancelled through `cancel()`. |

State changes: `SCROLLING` on start, `IDLE` on end.

Listened:

| Event | Action |
|---|---|
| `move` | Stops a running scroll silently (no `scrolled`, no callback). |
| `updated`, `refresh` | `cancel()` (emits `scrolled` if a scroll was running). |

## Options read

| Option | Use |
|---|---|
| `easingFunc` | Custom easing `( t ) => number` for scrolls. Default easeOutQuart. Define the function outside the component (or memoise it): React Splide compares functions in `options` by identity, so a new function on every render makes each render an options update. |
| `type` | Bounce and snap restrictions apply only to `slide`. |

`useScroll` (read by the Slide transition, not this component) makes normal `go()` transitions use `Scroll.scroll( position, speed, false, done )` instead of a CSS transition.
