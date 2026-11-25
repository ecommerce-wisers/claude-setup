# Controller component

Source: https://splidejs.com/components/controller/
Source file: `src/js/components/Controller/Controller.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: turns a move request (index, `'+1'`, `'>'`, and so on) into a valid slide index, keeps the current and previous index, and passes the move to the Move component.

Access: `splide.Components.Controller` (filled in by `splide.mount()`).

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `go( control, allowSameIndex?, callback? )` | `void` | Moves the carousel by a control pattern. `splide.go( control )` is an alias that passes only `control`. |
| `scroll( destination, duration?, snap?, callback? )` | `void` | Scrolls to a pixel position with the Scroll component (requestAnimationFrame, not a CSS transition), then updates the current/previous index. |
| `getNext( destination? )` | `number` | Index that `'>'` would go to, or `-1`. |
| `getPrev( destination? )` | `number` | Index that `'<'` would go to, or `-1`. |
| `getEnd()` | `number` | The last index the carousel can go to. |
| `setIndex( index )` | `void` | Sets the internal current index (old one becomes previous). Nothing moves. |
| `getIndex( prev? )` | `number` | Current index, or the previous index when `prev` is `true`. `splide.index` returns `getIndex()`. |
| `toIndex( page )` | `number` | Page index to slide index. |
| `toPage( index )` | `number` | Slide index to page index. |
| `toDest( position )` | `number` | (undocumented, from source) Pixel position to the closest destination index. |
| `hasFocus()` | `boolean` | (undocumented, from source) `true` if the `focus` option is set or `isNavigation` is `true`. |
| `isBusy()` | `boolean` | `true` while the carousel is moving/scrolling **and** `waitForTransition` is truthy. |
| `getAdjacent( prev, destination? )` | `number` | (undocumented, from source; marked `@internal`) Shared implementation of `getPrev` (`prev = true`) and `getNext` (`prev = false`). |

Lifecycle member `mount()` is also on the object. Splide calls it.

### go()

Docs say `go( control: number | string, allowSameIndex?: boolean ): void` (the params table lists `callback`); source says `go( control: number | string, allowSameIndex?: boolean, callback?: AnyFunction ): void`.

| Control | Meaning |
|---|---|
| `i` (number) | Go to slide `i`. Non-loop: clamped to `0..getEnd()`. Loop: used as given (may be a clone index). |
| `'+i'` | Add `i` to the current index. `'+'` alone means `'+1'`. |
| `'-i'` | Subtract `i` from the current index. `'-'` alone means `'-1'`. |
| `'>'` | Next page (`getNext( true )`). Respects `perPage`/`perMove`. |
| `'<'` | Previous page (`getPrev( true )`). |
| `'>i'` | Go to page `i` (`toIndex( i )`). |

Params:

| Param | Description |
|---|---|
| `control` | A control pattern from the table above. |
| `allowSameIndex` | Optional. If `true`, moves (and fires `move`/`moved`) even when the target equals the current index. Arrows and pagination pass `true`. |
| `callback` | Optional. Called after the transition ends, right after `moved` is emitted. |

Behaviour from source:

- Does nothing while `isBusy()` is `true`.
- Does nothing if the resolved index is `-1` (nowhere to go) or equals the current index without `allowSameIndex`.
- `'<i'` is not a pattern: the number is ignored and it behaves like `'<'`.
- In loop mode, the destination can be a clone index (`dest`), and the real index is `dest` wrapped into `0..length-1`. Both are passed to `move`/`moved` handlers as `( index, prev, dest )`.

```js
const splide = new Splide( '.splide', { perPage: 3 } ).mount();
const Controller = splide.Components.Controller;

Controller.go( 2 );            // slide 2
Controller.go( '+1' );         // next slide
Controller.go( '>' );          // next page
Controller.go( '>1' );         // page 1 (slide 3 with perPage 3)
Controller.go( 0, true, function () {
  console.log( 'transition finished' ); // splide.go() cannot take a callback
} );
```

### scroll()

`scroll( destination: number, duration?: number, snap?: boolean, callback?: AnyFunction ): void`

| Param | Description |
|---|---|
| `destination` | A pixel position (the same units as `Move.getPosition()` / `Move.toPosition()`). Docs say "A position or an index to scroll to. The `useIndex` must be `true` to specify the index." Source: there is no `useIndex` parameter or option in v4; `destination` is always a position. To scroll to an index, convert it with `Move.toPosition( index, true )`. |
| `duration` | Optional. Scroll duration in ms. If omitted (or `0`), it is computed from the distance: `max( distance / 1.5, 800 )`, and `0` when the distance is under 1px. |
| `snap` | Optional. Snap to the closest slide. |
| `callback` | Optional. Called after the scroll ends. |

- Ignores `isBusy()`.
- When the scroll ends, the current index is set to the slide closest to the final position (wrapped in loop mode, capped at `getEnd()` when `omitEnd` is set). Then `callback` runs, then `scrolled` is emitted. `move`/`moved` are not emitted.
- Uses `Components.Scroll.scroll()`, which emits `scroll` at the start.

```js
const { Controller, Move } = splide.Components;
Controller.scroll( Move.toPosition( 4, true ), 600, false, function () {
  console.log( splide.index ); // 4 (or the closest reachable index)
} );
```

### getNext() / getPrev()

`getNext( destination?: boolean ): number`, `getPrev( destination?: boolean ): number`

- The step size is `perMove`, or `1` when `focus`/`isNavigation` is set, otherwise `perPage`.
- `destination = false` (default): a clone index is resolved to the real slide index. For example, in loop mode `-1` becomes `length - 1`.
- `destination = true`: the raw destination, which can be negative or `>= length` in loop mode.
- Returns `-1` when there is nowhere to go (for example a `slide` carousel at its last page without `rewind`). Exception for the `slide` type: if the carousel is not exactly at its limit position yet, it returns `0` (prev) or `getEnd()` (next).
- Docs typo: the getPrev description says "destionation".

### getEnd()

`getEnd(): number`

`slideCount - perPage`, or `slideCount - 1` when `focus`/`isNavigation` is set or the type is `loop` with `perMove`. The result is clamped to `0..slideCount-1`. With `omitEnd`, it is reduced to the first index whose position equals the last slide's position. Example: 10 slides with `perPage: 3` gives `7`.

### setIndex()

`setIndex( index: number ): void`

Docs list the argument under "Return" ("A new index to set"); it is a parameter. It updates the stored current index (the old value becomes the previous index) only if `index` differs. Nothing moves, no events fire, and slide classes are not updated. Rarely needed.

### getIndex()

Docs show the signature as `getPrev( prev?: boolean ): number`; the source signature is `getIndex( prev?: boolean ): number`.

```js
splide.Components.Controller.getIndex();      // same as splide.index
splide.Components.Controller.getIndex( true ); // previous index
```

### toIndex() / toPage()

- `toIndex( page: number ): number` returns `page * perPage` (or `page` when `focus`/`isNavigation` is set), clamped to `0..getEnd()`.
- `toPage( index: number ): number`: docs name the param `page`; source names it `index`. With `focus`/`isNavigation`, it returns `min( index, getEnd() )`. Otherwise it returns `floor( index / perPage )`, where any index `>= getEnd()` maps to the last page.

```js
const Controller = splide.Components.Controller;
const page = Controller.toPage( splide.index );
const firstSlideOfNextPage = Controller.toIndex( page + 1 );
```

### toDest()

(undocumented, from source) `toDest( position: number ): number` returns the index closest to `position` (`Move.toIndex()`), clamped to `0..getEnd()` for the `slide` type. Drag uses it to pick a target after a flick.

### hasFocus()

(undocumented, from source) `hasFocus(): boolean` returns `options.focus !== undefined || options.isNavigation`. When it is `true`, paging works per slide: pages equal slides, steps are 1, and `getEnd()` is `length - 1`.

### isBusy()

`isBusy(): boolean`

Docs (and the source JSDoc) say it returns "`true` if the carousel can move". That is inverted. The code is `state.is( [ MOVING, SCROLLING ] ) && !! options.waitForTransition`, so it returns `true` when the carousel is busy and `go()` will be ignored. `waitForTransition` has no default (docs: default `false`), so without `waitForTransition: true` this always returns `false`.

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `ei` (internal end-index-changed event) | `()` | On `resized`, if `getEnd()` changed (for example with `autoWidth`/`fixedWidth`). Arrows, Pagination and Controller itself react to it. |

`move`/`moved` are emitted by the Move component when `go()` hands the move over, and `scroll`/`scrolled` by the Scroll component.

Listened:

| Event | Action |
|---|---|
| `updated`, `refresh`, `ei` | Re-reads slide count, `perMove`, `perPage` and the end index. If the current index is now out of range, it is clamped and `Move.reposition()` is called. |
| `resized` | Emits `ei` if the end index changed. |

## Options read

| Option | Use |
|---|---|
| `start` | Initial index (default `0`). |
| `type` | `loop` wraps indices. `slide` enables the limit checks. |
| `perPage`, `perMove` | Page size and step size. |
| `focus`, `isNavigation` | Switch to per-slide paging (`hasFocus()`). |
| `rewind` | Wrap from the end to the start (non-loop). |
| `omitEnd` | Compute the end index from positions, and cap `scroll()` results. |
| `trimSpace` | `'move'` skips destinations that do not change the position (slide type). |
| `waitForTransition` | Makes `isBusy()` (and so `go()`) block during transitions. |
