# Pagination

Source: https://splidejs.com/guides/pagination/

Also checked: https://splidejs.com/components/pagination/, https://splidejs.com/guides/events/, https://splidejs.com/guides/accessibility/, https://splidejs.com/guides/options/, and the v4 source (`components/Pagination/Pagination.ts`, `components/Controller/Controller.ts`, `components/Slides/Slides.ts`, `components/Slides/Slide.ts`). The 4.1.4 dist is identical to 4.1.3 apart from the version banner.

## Contents

- [Setup](#setup)
- [Options](#options)
- [Generated markup and ARIA](#generated-markup-and-aria)
- [Adding classes](#adding-classes)
- [Placeholder: placing the pagination elsewhere](#placeholder-placing-the-pagination-elsewhere)
- [How many dots (page count)](#how-many-dots-page-count)
- [No paging: one dot per slide](#no-paging-one-dot-per-slide)
- [Direction](#direction)
- [Keyboard](#keyboard)
- [Numbered pagination](#numbered-pagination)
- [Custom pagination with `pagination:mounted`](#custom-pagination-with-paginationmounted)
- [`pagination:updated`](#paginationupdated)
- [Pagination component](#pagination-component)
- [Pagination outside the root](#pagination-outside-the-root)
- [Styling](#styling)
- [Pitfalls](#pitfalls)

## Setup

The examples use the global `Splide` and string event names. No ESM import is needed.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

## Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `pagination` | `boolean` | `true` | Creates the pagination (indicator dots). It must be `false` when `isNavigation` is used, or the roles and ARIA attributes conflict. With `false` and a placeholder in the HTML, the placeholder gets inline `display: none`. The pagination is rebuilt on `updated`, so breakpoints can switch it. |
| `paginationDirection` | `'ltr' \| 'rtl' \| 'ttb'` | `undefined`, which falls back to `direction` | Sets the pagination direction. It affects appearance (`splide__pagination--{dir}`), the keyboard shortcuts and `aria-orientation`. |
| `paginationKeyboard` | `boolean` | `true` | Enables keyboard shortcuts while a dot has focus. The W3C Carousel Design Pattern requires this. |
| `classes.pagination` | `string` | `'splide__pagination'` | Class of the `<ul>` that Splide creates. It is not applied to a placeholder. |
| `classes.page` | `string` | `'splide__pagination__page'` | Class of each dot `<button>`. |
| `i18n.select` | `string` | `'Select a slide to show'` | `aria-label` of the `<ul>`. |
| `i18n.slideX` | `string` | `'Go to slide %s'` | Dot label when `focus` is set or `perPage` is 1. |
| `i18n.pageX` | `string` | `'Go to page %s'` | Dot label when `perPage > 1` and `focus` is not set. |
| `focus` | `number \| 'center'` | `undefined` | Any defined value, `0` included, switches to one dot per slide ([No paging](#no-paging-one-dot-per-slide)). |
| `omitEnd` | `boolean` | `false` | Since v4.1.0. Omits dots that would only change the active slide without moving the carousel, and disables the next arrow at the end. Docs say it needs `type: 'slide'` (or unset) and `focus`. Source says: without `focus` it still trims the end index, so the next arrow is disabled at the end, but the dot count ignores it (tested: 9 `autoWidth` slides gave 9 dots without `focus`, 6 with `focus: 0`). |

## Generated markup and ARIA

The docs show this generated HTML:

```html
<ul class="splide__pagination splide__pagination--ltr" role="tablist" aria-label="Select a slide to show">
  <li role="presentation">
    <button
      class="splide__pagination__page is-active"
      type="button"
      role="tab"
      aria-controls="splide01-slide01"
      aria-label="Go to slide 1"
      aria-selected="true">
    </button>
  </li>
  <li role="presentation">
    <button
      class="splide__pagination__page"
      type="button"
      role="tab"
      aria-controls="splide01-slide02"
      aria-label="Go to slide 2"
      tabindex="-1"
    >
    </button>
  </li>
  ......
</ul>
```

| Element | Attributes (from source) |
|---|---|
| `<ul>` | `class="splide__pagination splide__pagination--{dir}"`, `role="tablist"`, `aria-label` = `i18n.select`. `aria-orientation="vertical"` only when the pagination direction is `ttb`; otherwise the attribute is absent. |
| `<li>` | `role="presentation"` |
| `<button>` (each dot) | `class="splide__pagination__page"`, `type="button"`, `role="tab"`, `aria-controls` = space-separated ids of all slides in that page, `aria-label` = `pageX` or `slideX` with the 1-based number. |
| Active dot | Adds class `is-active` and `aria-selected="true"`, and has no `tabindex`, so it is in the Tab order. |
| Inactive dots | `tabindex="-1"` and no `aria-selected` attribute (roving tabindex). |
| Slides | `role="tabpanel"` while pagination is enabled, otherwise `role="group"`. Ids are `{rootId}-slide01`, `{rootId}-slide02`, and so on. |

- With `perPage: 3` and no `focus`, the first dot's `aria-controls` is `splide01-slide01 splide01-slide02 splide01-slide03` and its label is "Go to page 1". The docs example shows the single-slide case.
- Without a placeholder, Splide appends the `<ul>` as the last child of the track's parent element. That is usually the root, or your wrapper if the track is nested inside one.
- Docs say (accessibility guide): `aria-selected` is "`false` for other dots". Source says: the attribute is removed from inactive dots, and only the active dot gets `aria-selected="true"`.
- Docs say (accessibility guide): the pagination `aria-label` is "Select slide to show". Source says: the default `i18n.select` is "Select a slide to show".
- Slides read `pagination` when they are created. Switching `pagination` at runtime does not change slide roles until a `refresh()`.

## Adding classes

The `classes` option adds your classes to the list and button elements:

```js
new Splide( '.splide', {
  classes: {
    pagination: 'splide__pagination your-class-pagination',
    page      : 'splide__pagination__page your-class-page',
  },
} );
```

The CSS targets the `splide__`-prefixed classes, so list the default class together with yours, never yours alone. The modifier `splide__pagination--{dir}` always uses the default prefix. Splide finds a placeholder by the hard-coded class `splide__pagination`, not by `classes.pagination`.

## Placeholder: placing the pagination elsewhere

By default, Splide appends the pagination to the parent of the track. To choose the position, add the placeholder `<ul class="splide__pagination"></ul>`:

```html
<section class="splide" aria-label="Placeholder Example">
  <ul class="splide__pagination"></ul>

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
    </ul>
  </div>

  <!-- default location -->
</section>
```

Never put the placeholder inside the track element (structure restriction).

From the source:

- The placeholder is found with `root.querySelector( '.splide__pagination' )`, excluding matches that belong to a nested `.splide`. It must be inside this root, at any depth, but not in the track or list. A `<ul>` outside the root is ignored; see [Pagination outside the root](#pagination-outside-the-root).
- Splide fills it with `<li><button>` items and adds `role`, `aria-label`, `aria-orientation` and the direction modifier class.
- On destroy or rebuild, only the `<li>` children are removed. The placeholder element stays.
- Use a `<ul>` (or `<ol>`), because Splide inserts `<li>` children.

## How many dots (page count)

From `Pagination.ts` and `Controller.ts`:

```js
// number of dots
var hasFocus = options.focus !== undefined || options.isNavigation;
var count    = hasFocus ? Controller.getEnd() + 1 : Math.ceil( splide.length / options.perPage );
```

- `splide.length` counts real slides; clones are excluded.
- `perMove` does not change the number of dots. Dots follow `perPage` (or `focus`). Nine slides with `perPage: 3, perMove: 1` still give 3 dots.
- Clicking dot `i` calls `go( '>' + i )`. That moves to slide index `clamp( perPage * i, 0, endIndex )`, or to index `i` when `focus` is set.
- End index without `focus`: `length - perPage` (`length - 1` in a loop with `perMove`), so the last dot of a partial page aligns the last slides to the end. With `focus`: `length - 1`, or fewer with `omitEnd`.
- The active dot is `Controller.toPage( splide.index )`. Without `focus` that is `Math.floor( ( index >= endIndex ? length - 1 : index ) / perPage )`; with `focus` it is `Math.min( index, endIndex )`.
- The pagination is rebuilt on `updated` (breakpoints, `splide.options = ...`), on `refresh` (including `add()` and `remove()`), and when a resize changes the end index (`autoWidth` / `fixedWidth`). `pagination:mounted` fires again after each rebuild.

| Slides | Options | Dots |
|---|---|---|
| 9 | `perPage: 3` | 3 |
| 10 | `perPage: 3` | 4 (the last goes to index 7) |
| 9 | `perPage: 3, perMove: 1` | 3 |
| 9 | `perPage: 3, focus: 0` | 9 |
| 9 | `perPage: 3, focus: 0, omitEnd: true` | 7 |

The core and theme CSS hide the pagination when all slides fit: `.splide:not(.is-overflow) .splide__pagination { display: none; }` (v4.1+). See `overflow.md`.

## No paging: one dot per slide

When you set `perPage`, Splide generates a button for each page, not for each slide. To turn off this paging, so that the carousel moves one by one and shows every dot, set `focus` to `0`. You don't have to set `perMove` in this case.

```js
var splide = new Splide( '.splide', {
  perPage: 3,
  focus  : 0,
} );

splide.mount();
```

That example has 9 dots for 9 slides. Near the end, the last 2 dots only change the active slide without sliding the carousel. Since v4.1, `omitEnd` omits those dots, and also disables the next arrow once the carousel can go no further:

```js
var splide = new Splide( '.splide', {
  perPage: 3,
  focus  : 0,
  omitEnd: true,
} );

splide.mount();
```

`omitEnd` also works with `fixedWidth` and `autoWidth`, as long as `focus` is set too.

## Direction

By default, the pagination direction follows the `direction` option. With `direction: 'ttb'`, Splide treats the pagination as vertical regardless of how it looks: it sets `aria-orientation="vertical"` on the list and uses Up/Down to move focus.

To use horizontal pagination in a vertical carousel, or the reverse, override it with `paginationDirection`:

```js
new Splide( '.splide', {
  direction          : 'ttb',
  paginationDirection: 'ltr',
} );
```

## Keyboard

With `paginationKeyboard: true` (the default), these keys work while a dot has focus (source plus the accessibility guide):

| Key | Action |
|---|---|
| Right | Horizontal: focus the next dot and show its page (the previous dot in RTL) |
| Left | Horizontal: focus the previous dot (the next dot in RTL) |
| Down / Up | Vertical (`ttb`): focus the next / previous dot |
| Home | First dot |
| End | Last dot |
| Space / Enter | Native button activation (click). Shows the slide if it isn't active. |

- Arrow keys wrap around: from the last dot, Right goes to the first.
- Moving focus activates the matching page ("selection follows focus"). With `waitForTransition: true`, the selection may not follow focus during a transition.
- The global `keyboard` option is separate and `false` by default in v4. Only the Tabs-pattern shortcuts above are on by default.

## Numbered pagination

Without JavaScript, a CSS counter adds numbers:

```css
.splide__pagination {
  counter-reset: pagination-num;
}

.splide__pagination__page:before {
  counter-increment: pagination-num;
  content: counter( pagination-num );
}
```

## Custom pagination with `pagination:mounted`

Listen to `pagination:mounted` to customize each dot. Register the listener before `mount()`. This example adds page numbers (the CSS approach above is simpler):

```js
var splide = new Splide( '.splide' );

splide.on( 'pagination:mounted', function ( data ) {
  // You can add your class to the UL element
  data.list.classList.add( 'splide__pagination--custom' );

  // `items` contains all dot items
  data.items.forEach( function ( item ) {
    item.button.textContent = String( item.page + 1 );
  } );
} );

splide.mount();
```

Handler arguments:

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | `{ list, items }` |
| `item` | `PaginationItem` | The item for the active slide (`Pagination.getAt( splide.index )`). |

`PaginationData`:

| Property | Type | Description |
|---|---|---|
| `list` | `HTMLUListElement` | The UL element that contains the pagination items. |
| `items` | `PaginationItem[]` | An array of `PaginationItem` objects. |

`PaginationItem`:

| Property | Type | Description |
|---|---|---|
| `li` | `HTMLLIElement` | The list item element. |
| `button` | `HTMLButtonElement` | The button element for the dot. |
| `page` | `number` | The page index (0-based). |

Notes:

- Splide writes `aria-label` and `aria-controls` once, when it creates the buttons. It updates only `is-active`, `aria-selected` and `tabindex` afterwards. So changes you make in `pagination:mounted` (text, classes, a custom `aria-label`) last until the next rebuild, and the event fires again after each rebuild.
- The slides in a page are `splide.Components.Slides.getIn( item.page )`. Use that to build thumbnail or title dots:

```js
var splide = new Splide( '#gallery', { perPage: 1 } );

splide.on( 'pagination:mounted', function ( data ) {
  data.items.forEach( function ( item ) {
    var Slide = splide.Components.Slides.getIn( item.page )[ 0 ];
    var title = Slide && Slide.slide.getAttribute( 'data-title' );

    if ( title ) {
      item.button.textContent = title;          // visible text
      item.button.setAttribute( 'aria-label', title );
    }
  } );
} );

splide.mount();
```

## `pagination:updated`

Fired whenever the pagination status is updated: on `move` (at the start of the transition), `scroll` and `scrolled`, and once during mount before `pagination:mounted`.

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | `{ list, items }` |
| `prev` | `PaginationItem` | The previous item. |
| `curr` | `PaginationItem` | The current item. |

```js
var counter = document.querySelector( '.slide-counter' );
var splide  = new Splide( '.splide', { perPage: 2 } );

splide.on( 'pagination:updated', function ( data, prev, curr ) {
  if ( curr ) {
    counter.textContent = ( curr.page + 1 ) + ' / ' + data.items.length;
  }
} );

splide.mount();
```

## Pagination component

`splide.Components.Pagination` is available after `mount()`.

| Member | Signature | Description |
|---|---|---|
| `items` | `PaginationItem[]` | The data for each page (`li`, `button`, `page`). It is empty when pagination is disabled. |
| `getAt()` | `getAt( index: number ): PaginationItem \| undefined` | Returns the pagination item for a slide index. It converts the slide index to a page index, so with `perPage: 3`, `getAt( 0 )`, `getAt( 1 )` and `getAt( 2 )` return the same item. Returns `undefined` if not available. |

## Pagination outside the root

Splide never finds a pagination element outside the root. If you need dots outside it, set `pagination: false` and build them with the same page math. The recipe below uses a simple button group. It does not copy the full Tabs pattern (tablist, roving tabindex, arrow keys).

```html
<section id="gallery" class="splide" aria-label="Gallery">
  <div class="splide__track"><ul class="splide__list">...</ul></div>
</section>
<div id="gallery-dots" class="gallery-dots"></div>

<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    var splide = new Splide( '#gallery', { pagination: false, perPage: 3, gap: '1rem' } );
    var nav    = document.getElementById( 'gallery-dots' );
    var count  = 0;

    splide.mount();

    var Controller = splide.Components.Controller;
    var trackId    = splide.Components.Elements.track.id;

    function pageCount() {
      return Controller.hasFocus()
        ? Controller.getEnd() + 1
        : Math.ceil( splide.length / splide.options.perPage );
    }

    function build() {
      var i18n = splide.options.i18n;
      var useSlideLabel = Controller.hasFocus() || ! ( splide.options.perPage > 1 ); // same rule as Pagination.ts

      count = pageCount();
      nav.textContent = '';

      for ( var i = 0; i < count; i++ ) {
        var button = document.createElement( 'button' );
        button.type = 'button';
        button.className = 'gallery-dot';
        button.setAttribute( 'aria-controls', trackId );
        button.setAttribute( 'aria-label', ( useSlideLabel ? i18n.slideX : i18n.pageX ).replace( '%s', i + 1 ) );
        button.addEventListener( 'click', splide.go.bind( splide, '>' + i ) );
        nav.appendChild( button );
      }

      sync();
    }

    function sync() {
      var page = Controller.toPage( splide.index );

      Array.prototype.forEach.call( nav.children, function ( button, i ) {
        var active = i === page;
        button.classList.toggle( 'is-active', active );
        active ? button.setAttribute( 'aria-current', 'true' ) : button.removeAttribute( 'aria-current' );
      } );
    }

    build();
    splide.on( 'move', sync );            // splide.index is already the new index here
    splide.on( 'updated refresh', build );
    splide.on( 'resized', function () { if ( pageCount() !== count ) build(); } );
  } );
</script>
```

The handlers are registered after `mount()` so they run after Controller has recalculated on `updated` / `refresh`.

## Styling

### Core CSS (`splide-core.min.css`, verbatim without the `-ms-` prefixed declarations)

```css
.splide__pagination{align-items:center;display:flex;flex-wrap:wrap;justify-content:center;margin:0;pointer-events:none}
.splide__pagination li{display:inline-block;line-height:1;list-style-type:none;margin:0;pointer-events:auto}
.splide:not(.is-overflow) .splide__pagination{display:none}
```

The `<ul>` has `pointer-events: none` and only the `<li>` items receive clicks. Keep that in mind if you put a background or a click target on the list.

### Theme CSS (`splide.min.css` 4.1.4, verbatim without the `-ms-` prefixed declarations)

```css
.splide__pagination{bottom:.5em;left:0;padding:0 1em;position:absolute;right:0;z-index:1}
.splide__pagination__page{background:#ccc;border:0;border-radius:50%;display:inline-block;height:8px;margin:3px;opacity:.7;padding:0;position:relative;transition:transform .2s linear;width:8px}
.splide__pagination__page.is-active{background:#fff;transform:scale(1.4);z-index:1}
.splide__pagination__page:hover{cursor:pointer;opacity:.9}
.splide__pagination__page:focus-visible{outline:3px solid #0bf;outline-offset:3px}
.splide.is-focus-in .splide__pagination__page:focus{outline:3px solid #0bf;outline-offset:3px}
.splide__pagination--ttb{bottom:0;display:flex;flex-direction:column;left:auto;padding:1em 0;right:.5em;top:0}
```

### Common overrides

```css
/* Theme: move the dots below the slides instead of overlaying them */
.splide__pagination { position: static; padding: 1rem 0 0; }
.splide__pagination__page { background: #bbb; }
.splide__pagination__page.is-active { background: #222; }

/* Core-only starter dots */
.splide__pagination { gap: .5rem; padding: 1rem 0 0; }
.splide__pagination__page {
  width: 10px; height: 10px; padding: 0; border: 0; border-radius: 50%;
  background: #bbb; cursor: pointer;
}
.splide__pagination__page.is-active { background: #222; }
.splide__pagination__page:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__pagination__page:focus { outline: 3px solid #0bf; outline-offset: 3px; }

/* Numbered buttons (with the CSS counter or the pagination:mounted text) */
.splide__pagination__page { width: auto; height: auto; min-width: 2em; padding: .25em .5em; border-radius: 4px; transform: none; }
```

`is-focus-in` is added to the root when the carousel gets focus from the keyboard. It is not removed immediately when focus leaves, so pair it with `:focus`.

## Pitfalls

- Register `pagination:mounted` before `mount()`, or the first build is missed. It refires after each rebuild, so put customizations there and not after `mount()`.
- `pagination` must be `false` on an `isNavigation` (thumbnail) carousel.
- Expecting `perMove` to add dots: only `perPage` and `focus` change the count.
- A placeholder outside the root, or inside the track, is ignored or breaks the structure.
- The theme positions the pagination absolutely over the slides (`bottom: .5em`). Override `position` to put it below.
- Dots disappear when all slides fit (`.splide:not(.is-overflow)` rule in v4.1 CSS). This is intended. Override it if you need dots anyway.
