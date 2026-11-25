# Layout component (React)

`splide.Components.Layout` as reached from React Splide: size measurements, `isOverflow()`, and the `overflow` event (which has no event prop).

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [isOverflow() and the overflow event](#isoverflow) · [resize()](#resize) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/layout/
Source file: `src/js/components/Layout/Layout.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: applies sizes to the carousel elements from the current options (root `max-width`, track padding and height, slide width/height/gap) and measures them.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Layout;
```

- In an event prop, the core instance is the first argument: `onResized={ ( splide ) => splide.Components.Layout.listSize() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Layout` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

All sizes are in pixels from `getBoundingClientRect()`, measured along the slider axis: width for `ltr`/`rtl`, height for `ttb`. They are measurements, not React state: read them when you need them, or copy them into state from `onResized`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `listSize()` | `number` | Width (or height for `ttb`) of the list element. |
| `slideSize( index?, withoutGap? )` | `number` | Size of the slide at `index` (default 0), plus the gap unless `withoutGap`. |
| `sliderSize( withoutGap? )` | `number` | Total size of the real slides (clones excluded). |
| `totalSize( index, withoutGap? )` | `number` | Distance from the list's start edge to the end edge of slide `index`, plus the gap unless `withoutGap`. Includes head clones. |
| `getPadding( right )` | `number` | Computed track padding in px for the start (`false`) or end (`true`) side. |
| `isOverflow()` | `boolean` | `true` if the slides are larger than the list (always `true` for `fade`). Since 4.1.0. |
| `resize( force? )` | `void` | (undocumented, from source; marked `@internal`) Re-applies sizes when the root size changed (or always with `force`), then emits `resized` and possibly `overflow`. |

Lifecycle member `mount()` is also on the object. Splide calls it.

### listSize()

`listSize(): number`

Returns `list.getBoundingClientRect()[ resolve( 'width' ) ]`. This is the visible area inside the track padding.

### slideSize()

Docs say `slideSize( index: number, withoutGap?: boolean ): number` and mark `index` optional. The implementation is `slideSize( index?: number, withoutGap?: boolean )`, and `index` defaults to `0`. The TypeScript interface keeps `index` required, so pass `0` explicitly in TypeScript.

| Param | Description |
|---|---|
| `index` | Optional. A slide index (clone indices work too). |
| `withoutGap` | Optional. If `true`, the gap is not added. |

Returns the size in px, or `0` if there is no slide at that index. The gap is read from the computed `margin-right` (`margin-bottom` for `ttb`, `margin-left` for `rtl`) of slide 0.

### sliderSize()

Docs say `sliderSize(): number`; source says `sliderSize( withoutGap?: boolean ): number`.

Computed as `totalSize( length - 1 ) - totalSize( 0 ) + slideSize( 0, withoutGap )`: the size of all real slides with their gaps. With `withoutGap`, the trailing gap is left out.

### totalSize()

Docs say `totalSize( index?: number, withoutGap?: boolean ): number` with "If omitted, uses the last index"; the source JSDoc and the TypeScript interface say the same. The implementation looks the index up with `Slides.getAt( index )`, which matches nothing for `undefined`, so an omitted index returns `0` (tested). Always pass an index, for example `Layout.totalSize( splide.length - 1 )`.

| Param | Description |
|---|---|
| `index` | A slide index (clone indices allowed). |
| `withoutGap` | Optional. If `true`, the trailing gap is not added. |

Returns `|slideRect.right - listRect.left| + gap` (direction-resolved), or `0` if no slide matches.

### getPadding()

`getPadding( right: boolean ): number`

| Param | Description |
|---|---|
| `right` | `true` for `paddingRight` (`paddingBottom` for `ttb`, `paddingLeft` for `rtl`), `false` for the start side. |

Returns the computed padding of the track in px.

### isOverflow()

Docs show the signature as `getPadding( right: boolean ): number` (a copy error); source says `isOverflow(): boolean`.

Returns `splide.is( 'fade' ) || sliderSize( true ) > listSize()`. Since 4.1.0. The result is also reflected by the root class `is-overflow` and the `overflow` event.

There is no `onOverflow` prop. The first `overflow` event fires during mount, before any effect of yours runs, so read `isOverflow()` once after mount and listen with `splide.on()` for later changes. Then drive the options from state:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function LayoutOverflowExample() {
  const ref = useRef<Splide>( null );
  const [ count, setCount ] = useState( 2 );
  const [ overflow, setOverflow ] = useState( true );
  const [ sizes, setSizes ] = useState( '' );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( splide ) {
      const { Layout } = splide.Components;

      // The first 'overflow' event fired during mount, before this effect. Read the current value,
      // then follow later changes.
      setOverflow( Layout.isOverflow() );
      splide.on( 'overflow', isOverflow => setOverflow( isOverflow ) );
      splide.on( 'resized', () => {
        setSizes( `${ Layout.listSize() } ${ Layout.slideSize( 0 ) } ${ Layout.sliderSize() }` );
      } );
      setSizes( `${ Layout.listSize() } ${ Layout.slideSize( 0 ) } ${ Layout.sliderSize() }` );
    }
  }, [] );

  return (
    <>
      <Splide
        ref={ ref }
        options={ {
          perPage   : 3,
          gap       : '1rem',
          arrows    : overflow, // hide controls when everything already fits
          pagination: overflow,
          drag      : overflow,
          label     : 'Layout demo',
        } }
      >
        { Array.from( { length: count }, ( _, i ) => <SplideSlide key={ i }>Slide { i + 1 }</SplideSlide> ) }
      </Splide>

      <button type="button" id="add" onClick={ () => setCount( c => c + 2 ) }>Add two slides</button>
      <p id="overflow">{ String( overflow ) }</p>
      <p id="sizes">{ sizes }</p>
    </>
  );
}
```

Tested in production and under StrictMode: with 2 slides and `perPage: 3` there are no arrows, no dots and no `is-overflow`; after adding two slides (React Splide refreshes), `overflow` turns `true` and two arrows and two dots appear.

- The carousel renders once with the initial `overflow` value (`true` here) before the effect corrects it. Pick the initial state that matches your usual content.
- The `Layout` captured in the effect belongs to the current mount. A `destroy` breakpoint that stops matching mounts new components; read `splide.Components.Layout` inside the listener if you use one.
- Do not set `splide.options` from inside the `overflow` listener in React. React Splide compares the `options` prop with its own copy, not with the live options, so the next `options` prop change merges the prop's values back over yours (from source). Keep the values in state and pass them through the prop, as above.
- To catch the first `overflow` event itself, register in an extension's `setup()` (or its constructor body): built-in components mount before extensions, so a listener added in the extension's `mount()` misses it (tested). See `references/custom-extensions.md`.

## resize()

(undocumented, from source; marked `@internal`) `resize( force?: boolean ): void`

The component runs it on every `resize` event. Unless `force` is `true`, it does nothing when the root's width and height have not changed since the last run. Otherwise it:

1. Sets the track `height` (only for `ttb`: `calc(height - paddingStart - paddingEnd)`. For `ttb`, `height` or `heightRatio` is required; otherwise it throws `[splide] height or heightRatio is missing.`).
2. Sets each slide's gap margin (`marginRight`, resolved by direction) to `gap`.
3. Sets each slide's `width`: nothing with `autoWidth`, else `fixedWidth`, else (horizontal) `calc((100% + gap)/perPage - gap)`.
4. Sets the `height` of each slide's container, or the slide itself when there is no container: `fixedHeight`, else for `ttb` the per-page size (nothing with `autoHeight`), else for horizontal `height` or `listWidth * heightRatio`.
5. Emits `resized`. If the overflow state changed, toggles `is-overflow` on the root and emits `overflow( isOverflow )`.

To force a re-layout after changing CSS that Splide cannot see, call `splide.refresh()` (full refresh) or `splide.emit( 'resize' )` (size check only). In React, a change of slide content that alters sizes (with `autoWidth`, for example) keeps the same `<li>` elements, so React Splide does not refresh: call `ref.current?.splide?.refresh()` in an effect on that content (see `references/reactivity.md`).

## Events emitted / listened

Emitted:

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `resize` | `onResize` | `( splide )` | Window `resize` or `load`, throttled to one per animation frame. LazyLoad and `Slides.add()` also emit it. |
| `resized` | `onResized` | `( splide )` | After sizes were re-applied by `resize()`. |
| `overflow` | none | core: `( overflow: boolean )` | When `isOverflow()` changes: on first mount (initial `undefined` to boolean) and later. Listen with `splide.on( 'overflow', ... )`. |

Listened:

| Source | Event | Action |
|---|---|---|
| DOM window | `resize`, `load` | Emits `resize` (throttled). |
| Splide | `updated`, `refresh` | Re-applies root `max-width` and track padding, then `resize( true )`. In React: an `options` prop change, or a change in the slide children. |
| Splide | `resize` | `resize()` (skipped if the root size is unchanged). |

Only window `resize` and `load` are watched (from source). A container that changes size without a window resize (a sidebar opening, a parent with a CSS transition) is not noticed: call `splide.emit( 'resize' )` from your own `ResizeObserver` or from the code that changes the layout.

## Options read

| Option | Use |
|---|---|
| `width` | Root `max-width`. |
| `height`, `heightRatio` | Slide height (horizontal), track height (`ttb`). |
| `fixedWidth`, `fixedHeight` | Fixed slide width/height. |
| `autoWidth`, `autoHeight` | Leave slide width (horizontal) / height (`ttb`) to CSS. |
| `perPage` | Slide size `calc((100% + gap)/perPage - gap)`. |
| `gap` | Slide margin on the end side. |
| `padding` | Track padding: a number/string for both sides, or `{ left, right }` / `{ top, bottom }`. |
| `direction` | Axis and side resolution; `ttb` enables vertical sizing. |
| `type` | `fade` makes `isOverflow()` always `true`. |

Numbers are converted to `px`. Strings (`'10rem'`, `'50%'`) are used as given. All of these are re-read on every `updated` and `refresh`, so they can change through the `options` prop.
