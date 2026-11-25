# Move component

Source: https://splidejs.com/components/move/
Source file: `src/js/components/Move/Move.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: moves the carousel by setting `transform: translateX(...)` (`translateY` for `ttb`) on the list element. It converts between slide indices and pixel positions, and runs transitions through the Transition component.

Access: `splide.Components.Move` (filled in by `splide.mount()`).

Positions are list translate values in px. They are produced by `Direction.orient()`, so in `ltr`/`ttb` moving forward gives negative positions (slide 2 of a 300px-wide one-per-page carousel is at `-600`), and in `rtl` positive ones.

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
2. Sets the state to `MOVING` (`Splide.STATES.MOVING`, 4) and emits `move( index, prev, dest )`.
3. Calls `Components.Transition.start( index, done )`. When the transition ends, the state becomes `IDLE` (3), then `moved( index, prev, dest )` is emitted, then `callback` runs.

Calling `move()` directly leaves Controller's index unchanged, so arrows, pagination and `splide.index` get out of sync. Use `Controller.go()` or `splide.go()`.

### jump()

`jump( index: number ): void`

Calls `translate( toPosition( index, true ) )`. There is no transition, no events and no index change.

`jump()` alone does not update `splide.index`, slide classes, arrows or pagination. To go to a slide instantly with everything kept in sync, run a normal `go()` with a zero speed. The Slide transition jumps synchronously when `speed < 1`:

```js
const options = splide.options; // the live options object
const speed   = options.speed;

options.speed = 0;  // direct property write: no 'updated' event, nothing remounts
splide.go( 3 );     // slide/loop: move -> jump -> moved, synchronous (fade: moved fires on the next tick)
options.speed = speed;
```

If `rewindSpeed` is set and the jump rewinds (first to last page or back), set `rewindSpeed` to 0 as well, because the transition then uses it instead of `speed`.

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

```js
const { Move, Controller } = splide.Components;
Controller.scroll( Move.toPosition( 5, true ) ); // animate to slide 5 with the Scroll component
```

### getPosition()

`getPosition(): number`

Returns the current visual position, measured with `getBoundingClientRect()` as list edge minus track edge plus the oriented start padding. Mid-transition values are therefore accurate.

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

If `Controller.isBusy()` is `false`, calls `Scroll.cancel()`, `jump( splide.index )` and `Slides.update()`. The component runs it on `mounted`, `resized`, `updated` and `refresh`. You can call it after an external layout change.

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `move` | `( index, prev, dest )` | At the start of `move()`. |
| `moved` | `( index, prev, dest )` | After the transition ends. |
| `sh` (internal shifted event) | `()` | When `translate()` wraps a loop position. Slide objects update their status on it. |

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
