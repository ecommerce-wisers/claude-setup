# Direction component (React)

`splide.Components.Direction` as reached from React Splide: `resolve()` and `orient()` for direction-independent code.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [resolve()](#resolve) · [orient()](#orient) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/direction/
Source file: `src/js/components/Direction/Direction.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: hides the differences between the `ltr`, `rtl` and `ttb` directions by translating property names and signs.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Direction;
```

- In an event prop, the core instance is the first argument: `onResized={ ( splide ) => splide.Components.Direction.resolve( 'width' ) }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true`. A remount creates a new Direction object, so read `splide.Components.Direction` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `resolve( prop, axisOnly?, direction? )` | `string` | Translates a property name (CSS property, rect key, axis letter or key name) for the current or given direction. |
| `orient( value )` | `number` | Gives a value the sign of the current direction: `-value` for `ltr` and `ttb`, `value` for `rtl`. |

The component has no `mount()`/`destroy()`.

### resolve()

Docs say `resolve( prop: string, axisOnly?: boolean ): string`; source says `resolve( prop: string, axisOnly?: boolean, direction?: 'ltr' | 'rtl' | 'ttb' ): string`.

| Param | Description |
|---|---|
| `prop` | The property name to translate. |
| `axisOnly` | Optional. If `true`, `rtl` is treated like `ltr` (only the axis changes, not the side). Has no effect for `ttb`. |
| `direction` | (undocumented, from source) Optional. The direction to resolve for. Defaults to the current `options.direction`. Pagination uses it with `paginationDirection`. |

Returns the resolved property name. Names that are not in the map and contain no `width`, `left` or `right` (any case) are returned unchanged.

How it works: an exact match in the internal map is used first. Otherwise, the first `width`/`left`/`right` substring (case-insensitive) is replaced, and the replacement is capitalised when it is not at the start. Resulting values:

| `prop` | `ltr` | `rtl` | `ttb` |
|---|---|---|---|
| `width` | `width` | `width` | `height` |
| `autoWidth` | `autoWidth` | `autoWidth` | `autoHeight` |
| `fixedWidth` | `fixedWidth` | `fixedWidth` | `fixedHeight` |
| `scrollWidth` | `scrollWidth` | `scrollWidth` | `scrollHeight` |
| `left` | `left` | `right` | `top` |
| `right` | `right` | `left` | `bottom` |
| `marginRight` | `marginRight` | `marginLeft` | `marginBottom` |
| `paddingLeft` | `paddingLeft` | `paddingRight` | `paddingTop` |
| `paddingRight` | `paddingRight` | `paddingLeft` | `paddingBottom` |
| `x` | `x` | `x` | `y` |
| `X` | `X` | `X` | `Y` |
| `Y` | `Y` | `Y` | `X` |
| `ArrowLeft` | `ArrowLeft` | `ArrowRight` | `ArrowUp` |
| `ArrowRight` | `ArrowRight` | `ArrowLeft` | `ArrowDown` |

With `axisOnly = true`, every `rtl` column value equals the `ltr` one.

The docs list these accepted names: `marginRight`, `width`, `autoWidth`, `fixedWidth`, `paddingLeft`, `paddingRight`, `left`, `right`, `x`, `X`, `Y`, `ArrowLeft`, `ArrowRight` (the docs list `left` twice).

The return type is `string`. To index a `DOMRect` or `CSSStyleDeclaration` under `strict`, cast it to the keys you expect, as below.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DirectionExample() {
  const ref = useRef<Splide>( null );
  const [ resolved, setResolved ] = useState( '' );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( splide ) {
      const { resolve, orient } = splide.Components.Direction;
      const rect = splide.Components.Elements.list.getBoundingClientRect();
      const size = rect[ resolve( 'width' ) as 'width' | 'height' ]; // list size along the slider axis

      setResolved( [
        resolve( 'marginRight' ),     // 'marginBottom'
        resolve( 'width' ),           // 'height'
        resolve( 'left', false, 'rtl' ), // 'right' (explicit direction)
        orient( 100 ),                // -100
        size > 0,
      ].join( ' ' ) );
    }
  }, [] );

  return (
    <>
      <Splide ref={ ref } options={ { direction: 'ttb', height: '20rem', label: 'Vertical carousel' } }>
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <p id="resolved">{ resolved }</p>
    </>
  );
}
```

Tested: the paragraph reads `marginBottom height right -100 true`.

### orient()

`orient( value: number ): number`

Returns `value * ( options.direction === 'rtl' ? 1 : -1 )`. Example: `orient( 100 )` is `-100` in `ltr` (and `ttb`) and `100` in `rtl`. This is why Move positions are negative when an `ltr` carousel moves forward.

| Param | Description |
|---|---|
| `value` | A value to orient. |

Returns the oriented value.

Both methods read `options.direction` each time they are called, so they follow direction changes made through the `options` prop or breakpoints. A value you computed earlier (like `resolved` in the example) does not: compute it again after the change, for example in an `onUpdated` handler or an effect that depends on the direction.

## Events emitted / listened

None. The component neither emits nor listens to events. When a breakpoint changes `direction`, the Media component calls `splide.refresh()`. A change through the `options` prop only emits `updated` (both from source).

## Options read

| Option | Use |
|---|---|
| `direction` | `'ltr'` (default), `'rtl'` or `'ttb'`. `'ttb'` needs `height` or `heightRatio`. |
