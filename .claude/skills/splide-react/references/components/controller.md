# Controller component (React)

`splide.Components.Controller` as reached from React Splide: `go()` with a callback, `scroll()`, index and page conversion, and `isBusy()`.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [go()](#go) · [scroll()](#scroll) · [getNext() / getPrev()](#getnext--getprev) · [getEnd()](#getend) · [Index helpers](#setindex) · [isBusy()](#isbusy) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/controller/
Source file: `src/js/components/Controller/Controller.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: turns a move request (index, `'+1'`, `'>'`, and so on) into a valid slide index, keeps the current and previous index, and passes the move to the Move component.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Controller;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.Controller.getEnd() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Controller` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

The React component itself has `go( control )` (`ref.current?.go( '>' )`), which calls `splide.go( control )`, the alias of `Controller.go( control )`. Neither alias takes `allowSameIndex` or a callback; use `Controller.go()` for those.

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

None of these values are React state. `splide.index` or `getEnd()` read during render are stale (and `undefined` in the first render). Keep the index in state from `onMove` or `onMoved`, as below.

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
- In loop mode, the destination can be a clone index (`dest`), and the real index is `dest` wrapped into `0..length-1`. Both are passed to `move`/`moved` handlers: core `( index, prev, dest )`, React props `onMove` / `onMoved` as `( splide, index, prev, dest )`.

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function ControllerExample() {
  const ref = useRef<Splide>( null );
  const [ index, setIndex ] = useState( 0 );
  const [ message, setMessage ] = useState( '' );

  const getController = () => ref.current?.splide?.Components.Controller;

  const showPage = () => {
    const Controller = getController();

    if ( Controller ) {
      const page = Controller.toPage( Controller.getIndex() );
      setMessage( `page ${ page }, next page starts at ${ Controller.toIndex( page + 1 ) }, end ${ Controller.getEnd() }` );
    }
  };

  return (
    <>
      <Splide
        ref={ ref }
        options={ { perPage: 3, speed: 200, label: 'Controller demo' } }
        onMoved={ ( splide, newIndex ) => setIndex( newIndex ) }
      >
        { [ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="plus" onClick={ () => getController()?.go( '+1' ) }>Next slide</button>
      <button type="button" id="next-page" onClick={ () => getController()?.go( '>' ) }>Next page</button>
      <button type="button" id="page-1" onClick={ () => getController()?.go( '>1' ) }>Page 1</button>
      <button
        type="button"
        id="first"
        onClick={ () => getController()?.go( 0, true, () => setMessage( 'transition finished' ) ) }
      >
        First, with a callback
      </button>
      <button type="button" id="info" onClick={ showPage }>Page info</button>

      <p id="index">{ index }</p>
      <p id="message">{ message }</p>
    </>
  );
}
```

Tested: `'+1'` from 0 gives 1, `'>'` from 1 gives 3 (it snaps to the next page), page info at index 3 reads `page 1, next page starts at 6, end 7`, the callback runs after the move to 0, and `'>1'` gives 3.

### scroll()

`scroll( destination: number, duration?: number, snap?: boolean, callback?: AnyFunction ): void`

| Param | Description |
|---|---|
| `destination` | A pixel position (the same units as `Move.getPosition()` / `Move.toPosition()`). Docs say "A position or an index to scroll to. The `useIndex` must be `true` to specify the index." Source: there is no `useIndex` parameter or option in v4; `destination` is always a position. To scroll to an index, convert it with `Move.toPosition( index, true )`. |
| `duration` | Optional. Scroll duration in ms. If omitted (or `0`), it is computed from the distance: `max( distance / 1.5, 800 )`, and `0` when the distance is under 1px. |
| `snap` | Optional. Snap to the closest slide. |
| `callback` | Optional. Called after the scroll ends. |

- Ignores `isBusy()`.
- When the scroll ends, the current index is set to the slide closest to the final position (wrapped in loop mode, capped at `getEnd()` when `omitEnd` is set). Then `callback` runs, then `scrolled` is emitted (`onScrolled`). `move`/`moved` are not emitted, so an `onMoved` handler that keeps the index in state does not see it: update state in the callback or in `onScrolled`.
- Uses `Components.Scroll.scroll()`, which emits `scroll` at the start.

```tsx
const { Controller, Move } = splide.Components;
Controller.scroll( Move.toPosition( 4, true ), 600, false, () => setStatus( `index ${ splide.index }` ) );
```

The full tested component is in `references/components/scroll.md` (`index 4` after the scroll).

### getNext() / getPrev()

`getNext( destination?: boolean ): number`, `getPrev( destination?: boolean ): number`

- The step size is `perMove`, or `1` when `focus`/`isNavigation` is set, otherwise `perPage`.
- `destination = false` (default): a clone index is resolved to the real slide index. For example, in loop mode `-1` becomes `length - 1`.
- `destination = true`: the raw destination, which can be negative or `>= length` in loop mode.
- Returns `-1` when there is nowhere to go (for example a `slide` carousel at its last page without `rewind`). Exception for the `slide` type: if the carousel is not exactly at its limit position yet, it returns `0` (prev) or `getEnd()` (next).
- Docs typo: the getPrev description says "destionation".

Use them to disable your own buttons, from `onMoved` (or `onArrowsUpdated`, which already passes `prevIndex` and `nextIndex`):

```tsx
onArrowsUpdated={ ( splide, prev, next, prevIndex, nextIndex ) => setCanGoNext( nextIndex > -1 ) }
```

### getEnd()

`getEnd(): number`

`slideCount - perPage`, or `slideCount - 1` when `focus`/`isNavigation` is set or the type is `loop` with `perMove`. The result is clamped to `0..slideCount-1`. With `omitEnd`, it is reduced to the first index whose position equals the last slide's position. Example: 10 slides with `perPage: 3` gives `7` (tested above).

### setIndex()

`setIndex( index: number ): void`

Docs list the argument under "Return" ("A new index to set"); it is a parameter. It updates the stored current index (the old value becomes the previous index) only if `index` differs. Nothing moves, no events fire, and slide classes are not updated. Rarely needed.

### getIndex()

Docs show the signature as `getPrev( prev?: boolean ): number`; the source signature is `getIndex( prev?: boolean ): number`.

```tsx
splide.Components.Controller.getIndex();      // same as splide.index
splide.Components.Controller.getIndex( true ); // previous index
```

### toIndex() / toPage()

- `toIndex( page: number ): number` returns `page * perPage` (or `page` when `focus`/`isNavigation` is set), clamped to `0..getEnd()`.
- `toPage( index: number ): number`: docs name the param `page`; source names it `index`. With `focus`/`isNavigation`, it returns `min( index, getEnd() )`. Otherwise it returns `floor( index / perPage )`, where any index `>= getEnd()` maps to the last page.

The `showPage` function in the example above uses both.

### toDest()

(undocumented, from source) `toDest( position: number ): number` returns the index closest to `position` (`Move.toIndex()`), clamped to `0..getEnd()` for the `slide` type. Drag uses it to pick a target after a flick.

### hasFocus()

(undocumented, from source) `hasFocus(): boolean` returns `options.focus !== undefined || options.isNavigation`. When it is `true`, paging works per slide: pages equal slides, steps are 1, and `getEnd()` is `length - 1`.

### isBusy()

`isBusy(): boolean`

Docs (and the source JSDoc) say it returns "`true` if the carousel can move". That is inverted. The code is `state.is( [ MOVING, SCROLLING ] ) && !! options.waitForTransition`, so it returns `true` when the carousel is busy and `go()` will be ignored. `waitForTransition` has no default (docs: default `false`), so without `waitForTransition: true` this always returns `false`.

## Events emitted / listened

Emitted:

| Event | React prop | Arguments | When |
|---|---|---|---|
| `ei` (internal end-index-changed event) | none | `()` | On `resized`, if `getEnd()` changed (for example with `autoWidth`/`fixedWidth`). Arrows, Pagination and Controller itself react to it. Listen with `splide.on( 'ei', ... )` if you need it. |

`move`/`moved` are emitted by the Move component when `go()` hands the move over (`onMove`, `onMoved`), and `scroll`/`scrolled` by the Scroll component (`onScroll`, `onScrolled`).

Listened:

| Event | Action |
|---|---|
| `updated`, `refresh`, `ei` | Re-reads slide count, `perMove`, `perPage` and the end index. If the current index is now out of range, it is clamped and `Move.reposition()` is called. |
| `resized` | Emits `ei` if the end index changed. |

In React, `updated` comes from an `options` prop change and `refresh` from a change in the `<SplideSlide>` children. Removing slides so that the current index is past the new end therefore clamps the index without a `move` event (from source), so an index kept in state from `onMoved` can go stale. Read `splide.index` again after the change (for example in a `useEffect` on the slide list).

## Options read

| Option | Use |
|---|---|
| `start` | Initial index (default `0`). Read at creation only; changing it in the `options` prop later does not move the carousel. |
| `type` | `loop` wraps indices. `slide` enables the limit checks. |
| `perPage`, `perMove` | Page size and step size. Re-read on every `updated` and `refresh`. |
| `focus`, `isNavigation` | Switch to per-slide paging (`hasFocus()`). |
| `rewind` | Wrap from the end to the start (non-loop). |
| `omitEnd` | Compute the end index from positions, and cap `scroll()` results. Read once, at creation. |
| `trimSpace` | `'move'` skips destinations that do not change the position (slide type). |
| `waitForTransition` | Makes `isBusy()` (and so `go()`) block during transitions. |
