# Layout component

Source: https://splidejs.com/components/layout/
Source file: `src/js/components/Layout/Layout.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: applies sizes to the carousel elements from the current options (root `max-width`, track padding and height, slide width/height/gap) and measures them.

Access: `splide.Components.Layout` (filled in by `splide.mount()`).

All sizes are in pixels from `getBoundingClientRect()`, measured along the slider axis: width for `ltr`/`rtl`, height for `ttb`.

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

Docs say `slideSize( index: number, withoutGap?: boolean ): number` and mark `index` optional. The implementation is `slideSize( index?: number, withoutGap?: boolean )`, and `index` defaults to `0`.

| Param | Description |
|---|---|
| `index` | Optional. A slide index (clone indices work too). |
| `withoutGap` | Optional. If `true`, the gap is not added. |

Returns the size in px, or `0` if there is no slide at that index. The gap is read from the computed `margin-right` (`margin-bottom` for `ttb`, `margin-left` for `rtl`) of slide 0.

### sliderSize()

Docs say `sliderSize(): number`; source says `sliderSize( withoutGap?: boolean ): number`.

Computed as `totalSize( length - 1 ) - totalSize( 0 ) + slideSize( 0, withoutGap )`: the size of all real slides with their gaps. With `withoutGap`, the trailing gap is left out.

### totalSize()

Docs say `totalSize( index?: number, withoutGap?: boolean ): number` with "If omitted, uses the last index"; the source JSDoc says the same. The implementation looks the index up with `Slides.getAt( index )`, which matches nothing for `undefined`, so an omitted index returns `0`. Always pass an index, for example `Layout.totalSize( splide.length - 1 )`.

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

```js
const splide = new Splide( '.splide', { perPage: 3, gap: '1rem' } );

splide.on( 'overflow', function ( isOverflow ) {
  // Hide controls when everything already fits.
  splide.options = { arrows: isOverflow, pagination: isOverflow, drag: isOverflow };
} );

splide.mount();

const Layout = splide.Components.Layout;
console.log( Layout.listSize(), Layout.slideSize( 0 ), Layout.sliderSize(), Layout.isOverflow() );
```

### resize()

(undocumented, from source; marked `@internal`) `resize( force?: boolean ): void`

The component runs it on every `resize` event. Unless `force` is `true`, it does nothing when the root's width and height have not changed since the last run. Otherwise it:

1. Sets the track `height` (only for `ttb`: `calc(height - paddingStart - paddingEnd)`. For `ttb`, `height` or `heightRatio` is required; otherwise it throws `[splide] height or heightRatio is missing.`).
2. Sets each slide's gap margin (`marginRight`, resolved by direction) to `gap`.
3. Sets each slide's `width`: nothing with `autoWidth`, else `fixedWidth`, else (horizontal) `calc((100% + gap)/perPage - gap)`.
4. Sets the `height` of each slide's container, or the slide itself when there is no container: `fixedHeight`, else for `ttb` the per-page size (nothing with `autoHeight`), else for horizontal `height` or `listWidth * heightRatio`.
5. Emits `resized`. If the overflow state changed, toggles `is-overflow` on the root and emits `overflow( isOverflow )`.

To force a re-layout after changing CSS that Splide cannot see, call `splide.refresh()` (full refresh) or `splide.emit( 'resize' )` (size check only).

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `resize` | `()` | Window `resize` or `load`, throttled to one per animation frame. LazyLoad and `Slides.add()` also emit it. |
| `resized` | `()` | After sizes were re-applied by `resize()`. |
| `overflow` | `( overflow: boolean )` | When `isOverflow()` changes: on first mount (initial `undefined` to boolean) and later. |

Listened:

| Source | Event | Action |
|---|---|---|
| DOM window | `resize`, `load` | Emits `resize` (throttled). |
| Splide | `updated`, `refresh` | Re-applies root `max-width` and track padding, then `resize( true )`. |
| Splide | `resize` | `resize()` (skipped if the root size is unchanged). |

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

Numbers are converted to `px`. Strings (`'10rem'`, `'50%'`) are used as given.
