# Direction component

Source: https://splidejs.com/components/direction/
Source file: `src/js/components/Direction/Direction.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: hides the differences between the `ltr`, `rtl` and `ttb` directions by translating property names and signs.

Access: `splide.Components.Direction` (filled in by `splide.mount()`).

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

```js
const splide = new Splide( '.splide', { direction: 'ttb', height: '20rem' } ).mount();
const { resolve } = splide.Components.Direction;

resolve( 'marginRight' );                // 'marginBottom'
resolve( 'width' );                      // 'height'
resolve( 'left', false, 'rtl' );         // 'right' (explicit direction)

// Size of the list along the slider axis, in any direction:
const size = splide.Components.Elements.list.getBoundingClientRect()[ resolve( 'width' ) ];
```

### orient()

`orient( value: number ): number`

Returns `value * ( options.direction === 'rtl' ? 1 : -1 )`. Example: `orient( 100 )` is `-100` in `ltr` (and `ttb`) and `100` in `rtl`. This is why Move positions are negative when an `ltr` carousel moves forward.

| Param | Description |
|---|---|
| `value` | A value to orient. |

Returns the oriented value.

Both methods read `options.direction` each time they are called, so they follow direction changes made through options updates or breakpoints.

## Events emitted / listened

None. The component neither emits nor listens to events.

## Options read

| Option | Use |
|---|---|
| `direction` | `'ltr'` (default), `'rtl'` or `'ttb'`. |
