# Move component (React)

`splide.Components.Move` as reached from React Splide: positions, `jump()`, `translate()`, and an instant go that keeps React-side state in sync.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [move()](#move) · [jump() and instant moves](#jump) · [translate() / shift() / cancel()](#translate) · [Positions](#toindex) · [reposition()](#reposition) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/move/
Source file: `src/js/components/Move/Move.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: moves the carousel by setting `transform: translateX(...)` (`translateY` for `ttb`) on the list element. It converts between slide indices and pixel positions, and runs transitions through the Transition component.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Move;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.Move.getPosition() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Move` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

Positions are list translate values in px. They are produced by `Direction.orient()`, so in `ltr`/`ttb` moving forward gives negative positions (slide 2 of a 300px-wide one-per-page carousel is at `-600`), and in `rtl` positive ones.

The list's `transform` is an inline style Splide writes on the `<ul>` that `<SplideTrack>` renders. React never sets a `style` on that element, so the two do not conflict.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `move( dest, index, prev, callback? )` | `void` | Runs the transition to `dest` and emits `move`/`moved`. Does not update the current index; use `Controller.go()`. |
| `jump( index )` | `void` | Instantly positions the list at slide `index` (no transition). |
| `translate( position, preventLoop? )` | `void` | Sets the list transform to `position` px. No-op for `fade`. |
| `shift( position, backwards )` | `number` | (undocumented, from source) Returns `position` shifted by whole slider lengths, used for loop wrapping. |
| `cancel()` | `void` | Stops the running transition where it is. |
| `toIndex( position )` | `number` | Closest slide index (clones included) to a position. |
| `toPosition( index, trimming? )` | `number` | Position of slide `index`. |
| `getPosition()` | `number` | Current position of the list. |
| `getLimit( max )` | `number` | Position of the first (`false`) or last reachable (`true`) index. |
| `exceededLimit( max?, position? )` | `boolean` | Whether a position is beyond the min and/or max limit. |
| `reposition()` | `void` | (undocumented, from source; marked `@internal`) Cancels scrolling, jumps to `splide.index` and updates slide states, unless the carousel is busy. |

Lifecycle member `mount()` is also on the object. Splide calls it.

### move()

Docs say `move( dest: number, index: number, prev: number ): void`; source says `move( dest: number, index: number, prev: number, callback?: AnyFunction ): void`.

| Param | Description |
|---|---|
| `dest` | Destination index, including clone indices (may be negative or `>= length` in loop mode). |
| `index` | The real slide index that becomes active. |
| `prev` | The previous index. |
| `callback` | (undocumented, from source) Optional. Called after the transition ends, after `moved`. |

Steps:

1. In loop mode, if `dest !== index` and there is room, the list is first shifted by one slider length so the move can continue seamlessly through the clones. This shift uses `translate( position, true )`, so it does not emit the internal `sh` event (tested: `go( '<' )` from slide 0 of a loop carousel emits no `sh`).
2. Sets the state to `MOVING` (`Splide.STATES.MOVING`, 4) and emits `move( index, prev, dest )` (React prop `onMove`, called as `( splide, index, prev, dest )`).
3. Calls `Components.Transition.start( index, done )`. When the transition ends, the state becomes `IDLE` (3), then `moved( index, prev, dest )` is emitted (`onMoved`), then `callback` runs.

Calling `move()` directly leaves Controller's index unchanged, so arrows, pagination, `splide.index` and any React state you keep from `onMoved` get out of sync. Use `Controller.go()`, `splide.go()` or `ref.current?.go()`.

### jump()

`jump( index: number ): void`

Calls `translate( toPosition( index, true ) )`. There is no transition, no events and no index change.

`jump()` alone does not update `splide.index`, slide classes, arrows or pagination, and no event prop fires. To go to a slide instantly with everything kept in sync, run a normal `go()` with a zero speed. The Slide transition jumps synchronously when `speed < 1`:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function MoveJumpExample() {
  const ref = useRef<Splide>( null );
  const [ position, setPosition ] = useState( 0 );

  // Go to a slide instantly, keeping index, classes, arrows and pagination in sync.
  const jumpTo = ( index: number ) => {
    const splide = ref.current?.splide;

    if ( splide ) {
      const { options } = splide; // the live options object
      const speed = options.speed;

      options.speed = 0; // direct property write: no 'updated' event, nothing remounts
      splide.go( index ); // slide/loop: move -> jump -> moved, synchronously
      options.speed = speed;
    }
  };

  return (
    <>
      <Splide
        ref={ ref }
        options={ { label: 'Move demo' } }
        onMoved={ ( splide ) => setPosition( splide.Components.Move.getPosition() ) }
      >
        { [ 1, 2, 3, 4, 5 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="jump" onClick={ () => jumpTo( 3 ) }>Jump to slide 4</button>
      <p id="position">{ position }</p>
    </>
  );
}
```

Tested: slide 4 is active in the same task as the click, the active dot is `Go to slide 4`, and `position` is `-3 * trackWidth`.

- Writing `splide.options.speed` directly is invisible to React Splide: it compares the `options` prop with its own copy, so the next render does not undo the write, and no `updated` event fires. Do not use the `options` prop for this: a prop change emits `updated` and remounts arrows and pagination.
- `fade`: `moved` fires on the next tick, not synchronously.
- If `rewindSpeed` is set and the jump rewinds (first to last page or back), set `rewindSpeed` to 0 as well, because the transition then uses it instead of `speed`.

### translate()

Docs say `translate( position: number ): void`; source says `translate( position: number, preventLoop?: boolean ): void`.

| Param | Description |
|---|---|
| `position` | Position in px. |
| `preventLoop` | (undocumented, from source) Optional. If `true`, the position is used as given. Otherwise, in loop mode, a position past the first or last index is wrapped by `shift()` (emits internal `sh`). |

Sets `list.style.transform = 'translateX(<position>px)'` (`translateY` for `ttb`). Does nothing for the `fade` type.

### shift()

(undocumented, from source) `shift( position: number, backwards: boolean ): number`

Returns `position` moved by enough whole `Layout.sliderSize()` lengths to bring it back over the limit on the `backwards` side (at least one length). It only computes; nothing moves.

### cancel()

`cancel(): void`

Freezes the list at its current visual position (`translate( getPosition(), true )`) and calls `Transition.cancel()` (clears the CSS transition, cancels any Scroll). Drag calls it on pointer-down.

### toIndex()

`toIndex( position: number ): number`

Walks the Slide objects (clones included, sorted by index) and returns the index whose trimmed position is closest to `position`. In loop mode, the result can be a clone index.

### toPosition()

`toPosition( index: number, trimming?: boolean ): number`

| Param | Description |
|---|---|
| `index` | An index to convert (clone indices allowed). |
| `trimming` | Optional. If `true` and `trimSpace` is truthy and the type is `slide`, clamps the position so no empty space shows at the edges. |

Returns `orient( totalSize( index - 1 ) - offset )`. `offset` comes from the `focus` option: with `'center'` it is `(listSize - slideSize) / 2`; with a number `n` it is `n * slideSize`; otherwise 0.

```tsx
const { Move, Controller } = splide.Components;
Controller.scroll( Move.toPosition( 4, true ), 600, false, done ); // animate to slide 5 with the Scroll component
```

The full tested component is in `references/components/scroll.md`.

### getPosition()

`getPosition(): number`

Returns the current visual position, measured with `getBoundingClientRect()` as list edge minus track edge plus the oriented start padding. Mid-transition values are therefore accurate. The example above reads it in `onMoved`.

### getLimit()

`getLimit( max: boolean ): number`

Returns `toPosition( max ? Controller.getEnd() : 0, !! options.trimSpace )`.

### exceededLimit()

`exceededLimit( max?: boolean | undefined, position?: number ): boolean`

| Param | Description |
|---|---|
| `max` | Optional. `true` tests only the max limit, `false` only the min, `undefined` both. |
| `position` | Optional. The position to test. Defaults to `getPosition()`. |

Returns `true` if the position is past the limit.

### reposition()

(undocumented, from source; marked `@internal`) `reposition(): void`

If `Controller.isBusy()` is `false`, calls `Scroll.cancel()`, `jump( splide.index )` and `Slides.update()`. The component runs it on `mounted`, `resized`, `updated` and `refresh`. You can call it after an external layout change, for example in an effect after something outside the carousel changed its size.

## Events emitted / listened

Emitted:

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `move` | `onMove` | `( splide, index, prev, dest )` | At the start of `move()`. |
| `moved` | `onMoved` | `( splide, index, prev, dest )` | After the transition ends. |
| `sh` (internal shifted event) | none | `()` | When `translate()` wraps a loop position. Slide objects update their status on it. |

State changes: `MOVING` at the start of `move()`, `IDLE` when it finishes.

Listened:

| Event | Action |
|---|---|
| `mounted`, `resized`, `updated`, `refresh` | `reposition()`. |

## Options read

| Option | Use |
|---|---|
| `type` | `fade` disables `translate()`. `loop` enables wrapping. `slide` enables trimming. |
| `trimSpace` | Trims edge space in `toPosition( i, true )` and `getLimit()` (`slide` type). |
| `focus` | Offset of the active slide (`'center'` or a number of slide widths). |
| `direction` | Axis (`translateX`/`translateY`) and sign, through Direction. |
| `waitForTransition` | Through `Controller.isBusy()` in `reposition()`. |
| `speed`, `rewindSpeed`, `easing` | Read by the Slide transition on each move, so a direct write to `splide.options` (as above) applies to the next move. |
