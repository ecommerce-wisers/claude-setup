# Pagination component (React)

`splide.Components.Pagination` as reached from React Splide: the dot items, `getAt()`, `update()`, the pagination event props, and a pagination placeholder in JSX.

Contents: [Access from React](#access-from-react) · [Properties](#properties) · [Methods](#methods) · [Generated markup](#generated-markup) · [Customising dots](#customising-dots) · [Placeholder in JSX](#placeholder-in-jsx) · [Interaction](#interaction-from-source) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/pagination/
Source file: `src/js/components/Pagination/Pagination.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: generates the pagination dots (an ARIA tablist with one tab button per page) and keeps the active dot in sync.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Pagination;
```

- In an event prop, the core instance is the first argument, and the pagination events pass the data directly: `onPaginationMounted={ ( splide, data, item ) => ... }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Pagination` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

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

The `items` array keeps its identity: it is emptied and refilled on every remount (`updated`, `refresh`, end-index change). In React that is every `options` prop change that is not deep-equal, and every change in the `<SplideSlide>` children. The `li`/`button` elements are recreated each time, so do not cache them, and do not keep `items` in React state (the array is mutated in place, so React sees no change). Style them from the `onPaginationMounted` prop instead.

The dots are DOM elements Splide creates. You cannot render them as JSX. To get JSX dots, set `pagination: false` and render your own buttons that call `ref.current?.go( '>' + page )`, with the active page kept in state from `onMove`.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `getAt( index )` | `PaginationItem \| undefined` | The item for the page that contains slide `index`. |
| `update()` | `void` | (undocumented, from source) Moves the active state from the previous index's dot to the current index's dot, then emits `pagination:updated`. |

Lifecycle members `mount()` and `destroy()` are also on the object. Splide calls them.

### getAt()

Docs say `getAt( index: number ): PaginationItem` and the return text says "or otherwise `undefined`". The source interface also says `PaginationItem`, but the implementation returns `PaginationItem | undefined`. TypeScript therefore does not force a check: test the result before using it, as below.

| Param | Description |
|---|---|
| `index` | A slide index. It is converted with `Controller.toPage( index )`. |

Returns the pagination item, or `undefined`. With `perPage: 3`, `getAt( 0 )`, `getAt( 1 )` and `getAt( 2 )` return the same item.

```tsx
const focusCurrentDot = () => {
  const splide = ref.current?.splide;
  const item = splide?.Components.Pagination.getAt( splide.index );

  if ( item ) {
    item.button.focus();
  }
};
```

(Part of the tested placeholder example below.)

### update()

(undocumented, from source) `update(): void`

- Previous dot (`getAt( Controller.getIndex( true ) )`): removes `is-active` and `aria-selected`, sets `tabindex="-1"`.
- Current dot (`getAt( splide.index )`): adds `is-active`, sets `aria-selected="true"`, removes `tabindex`.
- Emits `pagination:updated( { list, items }, prev, curr )` (`onPaginationUpdated`).

It already runs on `move`, `scroll` and `scrolled`, so you rarely need to call it.

## Generated markup

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
- Placement: if the markup has a `.splide__pagination` `<ul>` inside the root (`Elements.pagination`), dots are appended into it. Otherwise a new `<ul>` is appended as the last child of the track's parent element, which is usually the root.
- Class modifier: `splide__pagination--{paginationDirection || direction}`. With `ttb`, `aria-orientation="vertical"` is set.
- Label: `i18n.pageX` (`'Go to page %s'`) when `perPage > 1` and no `focus`. Otherwise `i18n.slideX` (`'Go to slide %s'`). `%s` is the 1-based page number.
- `aria-controls` lists the ids of the slides in that page (`Slides.getIn( page )`).
- On remount or destroy, a generated `<ul>` is removed. With a placeholder, only its children are removed and the placeholder is kept.
- With `pagination: false`, a placeholder is hidden with `display: none`.

## Customising dots

Number the dots from `onPaginationMounted`:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function NumberedPaginationExample() {
  const [ perPage, setPerPage ] = useState( 1 );
  const [ current, setCurrent ] = useState( '' );

  return (
    <>
      <Splide
        options={ { perPage, label: 'Numbered dots' } }
        onPaginationMounted={ ( splide, data ) => {
          // Runs at mount and after every remount (options change, refresh), on new buttons.
          data.list.classList.add( 'my-pagination' );
          data.items.forEach( item => {
            item.button.textContent = String( item.page + 1 );
          } );
        } }
        onPaginationUpdated={ ( splide, data, prev, curr ) => setCurrent( curr ? String( curr.page + 1 ) : '' ) }
      >
        { [ 1, 2, 3, 4, 5, 6 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="per-page" onClick={ () => setPerPage( 2 ) }>Two per page</button>
      <p id="current">{ current }</p>
    </>
  );
}
```

Tested: six dots numbered `1` to `6`; after the `perPage` change, three dots numbered `1` to `3` with the label `Go to page 1`; clicking the third dot sets `current` to `3`.

Event props are bound before `splide.mount()`, so `onPaginationMounted` sees the first mount. Like every event prop, it is captured at mount: if the dot text depends on state, read it through a ref (see `references/reactivity.md`).

## Placeholder in JSX

To place the dots somewhere specific inside the root, render an empty `<ul className="splide__pagination">` with `hasTrack={ false }`:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function PaginationPlaceholderExample() {
  const ref = useRef<Splide>( null );
  const [ renders, setRenders ] = useState( 0 );

  const focusCurrentDot = () => {
    const splide = ref.current?.splide;
    const item = splide?.Components.Pagination.getAt( splide.index );

    if ( item ) {
      item.button.focus();
    }
  };

  return (
    <Splide ref={ ref } hasTrack={ false } options={ { perPage: 3, label: 'Placeholder dots' } }>
      <SplideTrack>
        { [ 1, 2, 3, 4, 5, 6, 7 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </SplideTrack>

      <div className="my-controls">
        { /* Leave it empty: Splide appends the dots. React does not touch children it did not render. */ }
        <ul className="splide__pagination" />
        <button type="button" id="rerender" onClick={ () => setRenders( n => n + 1 ) }>Re-render ({ renders })</button>
        <button type="button" id="focus-dot" onClick={ focusCurrentDot }>Focus current dot</button>
      </div>
    </Splide>
  );
}
```

Tested: three dots inside the placeholder, no second list, the dots survive re-renders, and the focus button focuses `Go to page 1`.

- Keep the `<ul>` empty in JSX and its `className` constant (Splide adds `splide__pagination--{direction}`).
- Keep it rendered. The Pagination component keeps the placeholder element it found when it was created, even across remounts (from source). If React replaces the element, the dots keep going into the old, detached one until the instance is mounted again (a new `key`).

## Interaction (from source)

- Click on a dot: `Controller.go( '>' + page, true )`.
- Keyboard (when `paginationKeyboard` is `true`, the default), on a focused dot:
  - ArrowRight / ArrowLeft go to the next / previous dot, wrapping around. These keys are resolved by `paginationDirection || direction`: reversed for `rtl`, ArrowDown / ArrowUp for `ttb`.
  - Home goes to the first dot, End to the last.
  - The key moves focus to the new dot, calls `go( '>' + page )`, and the event is cancelled.

## Events emitted / listened

Emitted:

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `pagination:mounted` | `onPaginationMounted` | `( splide, data: { list, items }, item )` | After the dots are created. `item` is `getAt( splide.index )`. Fires again after each remount. |
| `pagination:updated` | `onPaginationUpdated` | `( splide, data: { list, items }, prev, curr )` | After every `update()`. `prev`/`curr` are the items for the previous and current index (either may be `undefined`). |

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
| `pagination` | `true` | Enables the dots. Re-read on every remount, so it can change through the `options` prop. |
| `paginationKeyboard` | `true` | Arrow/Home/End keys on dots. |
| `paginationDirection` | (none) | Direction for the modifier class, `aria-orientation` and arrow keys. Falls back to `direction`. |
| `direction` | `'ltr'` | Fallback for the above. |
| `perPage` | `1` | Number of dots and label choice. |
| `focus`, `isNavigation` | (none) | Per-slide dots (`Controller.hasFocus()`). |
| `omitEnd` | (none) | Indirectly, through `Controller.getEnd()` / `toPage()`. It changes the dot count only when `focus`/`isNavigation` is set, because only then is the count `getEnd() + 1`. |
| `classes.pagination`, `classes.page` | `'splide__pagination'`, `'splide__pagination__page'` | Class names. |
| `i18n.select`, `i18n.pageX`, `i18n.slideX` | `'Select a slide to show'`, `'Go to page %s'`, `'Go to slide %s'` | ARIA labels. |
