# Pagination component

Source: https://splidejs.com/components/pagination/
Source file: `src/js/components/Pagination/Pagination.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: generates the pagination dots (an ARIA tablist with one tab button per page) and keeps the active dot in sync.

Access: `splide.Components.Pagination` (filled in by `splide.mount()`).

## Properties

| Name | Type | Description |
|---|---|---|
| `items` | `PaginationItem[]` | One entry per page. Empty when pagination is disabled. |

`PaginationItem`:

| Property | Type | Description |
|---|---|---|
| `li` | `HTMLLIElement` | The list item element. |
| `button` | `HTMLButtonElement` | The dot button. |
| `page` | `number` | The page index. |

The `items` array keeps its identity: it is emptied and refilled on every remount (`updated`, `refresh`, end-index change). The `li`/`button` elements are recreated each time, so do not cache them. Style them from the `pagination:mounted` event instead.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `getAt( index )` | `PaginationItem \| undefined` | The item for the page that contains slide `index`. |
| `update()` | `void` | (undocumented, from source) Moves the active state from the previous index's dot to the current index's dot, then emits `pagination:updated`. |

Lifecycle members `mount()` and `destroy()` are also on the object. Splide calls them.

### getAt()

Docs say `getAt( index: number ): PaginationItem` and the return text says "or otherwise `undefined`". The source interface also says `PaginationItem`, but the implementation returns `PaginationItem | undefined`.

| Param | Description |
|---|---|
| `index` | A slide index. It is converted with `Controller.toPage( index )`. |

Returns the pagination item, or `undefined`. With `perPage: 3`, `getAt( 0 )`, `getAt( 1 )` and `getAt( 2 )` return the same item.

```js
const splide = new Splide( '.splide', { perPage: 3 } ).mount();
const item = splide.Components.Pagination.getAt( splide.index );

if ( item ) {
  item.button.focus();
}
```

### update()

(undocumented, from source) `update(): void`

- Previous dot (`getAt( Controller.getIndex( true ) )`): removes `is-active` and `aria-selected`, sets `tabindex="-1"`.
- Current dot (`getAt( splide.index )`): adds `is-active`, sets `aria-selected="true"`, removes `tabindex`.
- Emits `pagination:updated( { list, items }, prev, curr )`.

It already runs on `move`, `scroll` and `scrolled`, so you rarely need to call it.

### Generated markup

```html
<ul class="splide__pagination splide__pagination--ltr" role="tablist" aria-label="Select a slide to show">
  <li role="presentation">
    <button class="splide__pagination__page is-active" type="button" role="tab"
            aria-controls="splide01-slide01 splide01-slide02 splide01-slide03"
            aria-label="Go to page 1" aria-selected="true"></button>
  </li>
  <li role="presentation">
    <button class="splide__pagination__page" type="button" role="tab"
            aria-controls="splide01-slide04 ..." aria-label="Go to page 2" tabindex="-1"></button>
  </li>
</ul>
```

- Number of dots: `ceil( splide.length / perPage )`, or `Controller.getEnd() + 1` when `focus`/`isNavigation` is set.
- Placement: if the HTML has a `.splide__pagination` `<ul>` inside the root (`Elements.pagination`), dots are appended into it. Otherwise a new `<ul>` is appended as the last child of the track's parent element, which is usually the root.
- Class modifier: `splide__pagination--{paginationDirection || direction}`. With `ttb`, `aria-orientation="vertical"` is set.
- Label: `i18n.pageX` (`'Go to page %s'`) when `perPage > 1` and no `focus`. Otherwise `i18n.slideX` (`'Go to slide %s'`). `%s` is the 1-based page number.
- `aria-controls` lists the ids of the slides in that page (`Slides.getIn( page )`).
- On remount or destroy, a generated `<ul>` is removed. With an HTML placeholder, only its children are removed and the placeholder is kept.
- With `pagination: false`, an HTML placeholder is hidden with `display: none`.

Customising dots (for example numbering them):

```js
const splide = new Splide( '.splide' );

splide.on( 'pagination:mounted', function ( data ) {
  data.list.classList.add( 'my-pagination' );

  data.items.forEach( function ( item ) {
    item.button.textContent = String( item.page + 1 );
  } );
} );

splide.mount();
```

Register the handler before `mount()`. It fires again after each remount.

### Interaction (from source)

- Click on a dot: `Controller.go( '>' + page, true )`.
- Keyboard (when `paginationKeyboard` is `true`, the default), on a focused dot:
  - ArrowRight / ArrowLeft go to the next / previous dot, wrapping around. These keys are resolved by `paginationDirection || direction`: reversed for `rtl`, ArrowDown / ArrowUp for `ttb`.
  - Home goes to the first dot, End to the last.
  - The key moves focus to the new dot, calls `go( '>' + page )`, and the event is cancelled.

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `pagination:mounted` | `( data: { list, items }, item )` | After the dots are created. `item` is `getAt( splide.index )`. Fires again after each remount. |
| `pagination:updated` | `( data: { list, items }, prev, curr )` | After every `update()`. `prev`/`curr` are the items for the previous and current index (either may be `undefined`). |

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `updated`, `refresh`, internal `ei` (end index changed) | Full remount: dots destroyed and rebuilt. |
| Splide | `move`, `scroll`, `scrolled` | `update()` (only when pagination is enabled). |
| DOM | `click` on each dot | Go to that page. |
| DOM | `keydown` on each dot | Keyboard navigation (when `paginationKeyboard`). |

## Options read

| Option | Default | Use |
|---|---|---|
| `pagination` | `true` | Enables the dots. |
| `paginationKeyboard` | `true` | Arrow/Home/End keys on dots. |
| `paginationDirection` | (none) | Direction for the modifier class, `aria-orientation` and arrow keys. Falls back to `direction`. |
| `direction` | `'ltr'` | Fallback for the above. |
| `perPage` | `1` | Number of dots and label choice. |
| `focus`, `isNavigation` | (none) | Per-slide dots (`Controller.hasFocus()`). |
| `omitEnd` | (none) | Indirectly, through `Controller.getEnd()` / `toPage()`. It changes the dot count only when `focus`/`isNavigation` is set, because only then is the count `getEnd() + 1`. |
| `classes.pagination`, `classes.page` | `'splide__pagination'`, `'splide__pagination__page'` | Class names. |
| `i18n.select`, `i18n.pageX`, `i18n.slideX` | `'Select a slide to show'`, `'Go to page %s'`, `'Go to slide %s'` | ARIA labels. |
