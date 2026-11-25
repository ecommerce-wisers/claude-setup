# Overflow

Scope: carousels with too few slides to scroll in React Splide. What counts as overflow, the `is-overflow` class, reading the `overflow` state without an `onOverflow` prop, switching arrows, pagination, drag and clones, and centring the slides.

Checked against the core 4.1.4 source (`components/Layout/Layout.ts`, `components/Clones/Clones.ts`, `components/Drag/Drag.ts`, `components/Autoplay/Autoplay.ts`, `core/Splide/Splide.ts`, `constants/classes.ts`), the 4.1.4 CSS, React Splide 0.7.12 (`src/js/components/Splide/Splide.tsx`, `src/js/constants/events.ts`), https://splidejs.com/guides/overflow/ and https://splidejs.com/components/layout/. Every example was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [Setup](#setup)
- [Enough / not enough slides](#enough--not-enough-slides)
- [What counts as overflow](#what-counts-as-overflow)
- [The `is-overflow` class (CSS only)](#the-is-overflow-class-css-only)
- [The `overflow` event: there is no `onOverflow` prop](#the-overflow-event-there-is-no-onoverflow-prop)
- [`Layout.isOverflow()`](#layoutisoverflow)
- [Toggling arrows, pagination and drag from React state](#toggling-arrows-pagination-and-drag-from-react-state)
- [Slide roles after switching pagination](#slide-roles-after-switching-pagination)
- [Loop carousels (clones) with an extension](#loop-carousels-clones-with-an-extension)
- [Centering slides](#centering-slides)
- [Pitfalls](#pitfalls)

## Setup

```sh
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The types need one tsconfig line under `moduleResolution: "bundler"`; see `references/typescript.md`.

## Enough / not enough slides

If a carousel doesn't have enough slides, you may want to hide the arrows and pagination and disable drag. Since v4.1.0, you can control this with the `is-overflow` status class or the `overflow` event.

For example, when slide widths come from `fixedWidth` (or `autoWidth`), the content can be narrower than the carousel. The docs demo disables some functions in that case and re-enables them once the viewport is resized so that the content is wide enough. Its options:

```tsx
const options: Options = {
  fixedWidth : 'min( 10rem, 20vw )',
  fixedHeight: '10rem',
  breakpoints: {
    640: { fixedHeight: '8rem' },
  },
};
```

This also helps dynamic carousels. In the docs' second demo (`perPage: 3`, `rewind: true`), navigation and drag turn on only once enough items have been added; the React version is [below](#toggling-arrows-pagination-and-drag-from-react-state).

## What counts as overflow

From `Layout.ts`:

```js
function isOverflow() {
  return Splide.is( 'fade' ) || sliderSize( true ) > listSize();
}
```

- `sliderSize( true )`: the distance from the start of the first real slide to the end of the last real slide. It includes the gaps between slides, excludes the trailing gap, and ignores clones.
- `listSize()`: the width of the `.splide__list` element, or its height in `ttb`. That is the visible area inside the track padding.
- It is always `true` for `type: 'fade'`.
- It applies to every layout, not only `fixedWidth` / `autoWidth`. With `perPage: 3`, 3 slides fit exactly and are not overflowing; 4 slides are (tested).
- It is recalculated on mount, on `updated` and `refresh` (which includes a changed `options` prop, slides added or removed in JSX, and breakpoint changes), and on window `resize` / `load` when the root's size has changed. A size change inside a slide is not noticed until one of those; see `references/auto-width.md`.

## The `is-overflow` class (CSS only)

When the slides together are wider than their parent list element, Splide adds the `is-overflow` class to the root. Combined with `:not()`, CSS can hide the arrows and centre the slides when there isn't enough content. This needs no JavaScript and no React state, and it can't flash: the class is set during mount, before the root becomes visible (`.splide` is `visibility: hidden` until `is-initialized`).

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/overflow.css';

const options: Options = {
  label      : 'Partners',
  fixedWidth : 'min( 10rem, 20vw )',
  fixedHeight: '6rem',
  gap        : '1rem',
};

export default function Example( { partners = [ 'Acme', 'Globex', 'Initech' ] }: { partners?: string[] } ) {
  return (
    <Splide tag="section" options={ options }>
      { partners.map( name => <SplideSlide key={ name }>{ name }</SplideSlide> ) }
    </Splide>
  );
}
```

`css/overflow.css`:

```css
/* Hide the arrows when there is not enough content (pagination is already hidden by the 4.1 CSS) */
.splide:not( .is-overflow ) .splide__arrows {
  display: none;
}

/* Centre the slides when there is not enough content */
.splide:not( .is-overflow ) .splide__list {
  justify-content: center;
}

/* Remove the trailing gap so centring is exact */
.splide:not( .is-overflow ) .splide__slide:last-child {
  margin: 0 !important;
}
```

Tested with 3 partners at 1280px: no `is-overflow`, the arrows and pagination are `display: none`, and the space left and right of the slides is equal.

- The class is toggled only when the state changes.
- The built-in CSS (core and themes, 4.1.x) already contains `.splide:not(.is-overflow) .splide__pagination{display:none}`. Pagination therefore hides itself when slides don't overflow. Arrows don't; add the rule above.
- Splide adds the class to the root element directly, and React doesn't remove it on re-render as long as the `className` prop of `<Splide>` stays the same. If `className` changes after mount, React rewrites the root's whole `class` attribute and drops every class Splide added, `is-initialized` included, so the core CSS (`.splide { visibility: hidden }`) hides the whole carousel. Tested: switching `className` from `is-light` to `is-dark` left `class="splide is-dark"` and `visibility: hidden`. Keep `<Splide className>` constant and put changing classes on a wrapper element.
- The class constant is `CLASS_OVERFLOW = 'is-overflow'` in the core ESM build. React Splide doesn't export it; use the string.

## The `overflow` event: there is no `onOverflow` prop

Splide emits `overflow` when the total width of the slides becomes wider or narrower than the list element.

| Event | Arguments | Since |
|---|---|---|
| `overflow` | `isOverflow: boolean` | 4.1.0 |

- Source: the stored state starts as `undefined`, so `overflow` always fires once during `mount()` with the initial state, `true` or `false`. After that it fires only on changes.
- The events guide lists no arguments for `overflow` and describes it as fired "by resizing the viewport". Source: it passes `isOverflow`, and also fires on mount, `updated` and `refresh`.
- React Splide has no `onOverflow` prop (its 28 event props don't include it). An `onOverflow` prop is a TypeScript error, and otherwise it is passed to the root element: React logs "Unknown event handler property `onOverflow`. It will be ignored." in development and the handler never runs (tested).

Three ways to get the state:

| Where | Catches the first call? | How |
|---|---|---|
| `onMounted` prop | No, it has already happened. Read `splide.Components.Layout.isOverflow()` there, then `splide.on( 'overflow', ... )` for changes. | [State example](#toggling-arrows-pagination-and-drag-from-react-state) |
| `useEffect` with `ref.current?.splide` | No. Same: read `isOverflow()`, then listen. | [`Layout.isOverflow()`](#layoutisoverflow) |
| An extension's `setup()` | Yes. `setup()` runs for every component before any of them mounts. | [Loop example](#loop-carousels-clones-with-an-extension) |

An extension's `mount()` is too late: extensions mount after the core components, Layout included, so a listener added there misses the first call (tested: the `setup()` listener received `true`, the `mount()` listener received nothing). Under StrictMode (development) two instances are created one after the other, so the extension runs twice and each instance gets its own listener.

The first call happens inside `mount()`, before Arrows, Pagination, Controller and Move have mounted. Setting `splide.options` there is safe: those components read the new values when they mount.

## `Layout.isOverflow()`

| Method | Signature | Returns |
|---|---|---|
| `isOverflow()` | `isOverflow(): boolean` | `true` if the total width of the slides is wider than the list element, otherwise `false`. Always `true` for fade carousels. Since 4.1.0. |

```tsx
useEffect( () => {
  const splide = ref.current?.splide;

  if ( splide ) {
    setOverflow( splide.Components.Layout.isOverflow() ); // the instance exists only after mount
  }
}, [] );
```

Tested: an effect like this read `true` for a carousel whose slides overflowed, in production and under StrictMode (where it ran once per instance).

Docs say (Layout component page): the signature is `getPadding( right: boolean ): number`, a copy error. Source says `isOverflow(): boolean`.

Related Layout methods: `listSize(): number` (list width, or height in vertical carousels, in px) and `sliderSize( withoutGap?: boolean ): number` (carousel size without the clones before the first slide). The docs list `sliderSize()` with no parameter; the source accepts `withoutGap`.

## Toggling arrows, pagination and drag from React state

The React way is to keep the overflow state in React and derive the options from it. React Splide sees the changed `options` prop and sets `splide.options`, which merges the values and emits `updated`; Arrows and Pagination remount and Drag re-reads `drag`. This is the docs' dynamic demo, with an "Add item" button and a status line:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const base: Options = {
  label  : 'Saved items',
  perPage: 3,
  gap    : '1rem',
  height : '8rem',
  rewind : true,
};

export default function Example() {
  const [ items, setItems ] = useState( [ 'Item 1', 'Item 2' ] );
  const [ overflow, setOverflow ] = useState<boolean>();

  // Navigation and drag only when there is something to navigate to
  const options: Options = {
    ...base,
    arrows    : overflow !== false,
    pagination: overflow !== false,
    drag      : overflow !== false,
  };

  return (
    <>
      <Splide
        options={ options }
        onMounted={ splide => {
          // There is no onOverflow prop. `overflow` has already fired during mount, so read
          // the initial state here and listen for changes from now on.
          setOverflow( splide.Components.Layout.isOverflow() );
          splide.on( 'overflow', setOverflow );
        } }
      >
        { items.map( item => <SplideSlide key={ item }>{ item }</SplideSlide> ) }
      </Splide>

      <p className="overflow-status">{ overflow ? 'Scroll for more' : `Showing all ${ items.length } items` }</p>
      <button type="button" className="add" onClick={ () => setItems( [ ...items, `Item ${ items.length + 1 }` ] ) }>
        Add item
      </button>
    </>
  );
}
```

Tested: with 2 and 3 items, no arrows, no dots, no `splide--draggable` and "Showing all N items"; after the fourth item, 2 arrows, 2 dots, draggable, `is-overflow` and "Scroll for more".

- `onMounted` is bound before `mount()` and fires after every component has mounted, so `isOverflow()` is ready there. The `overflow` listener then covers later changes (the new items arrive through React Splide's automatic `refresh()`).
- `splide.on( 'overflow', setOverflow )` passes the boolean straight to the state setter. Setters never change, so the mount-time capture of event props doesn't matter here.
- `overflow` starts `undefined`, so the first render has the controls on (`overflow !== false`); the state from `onMounted` turns them off before the browser paints.
- If you use `drag: 'free'`, restore `'free'`, not `true`: `drag: overflow !== false ? 'free' : false`.
- Autoplay is not affected, because it reads `autoplay` once at mount. Pause it yourself through the ref: `isOverflow ? Autoplay.play() : Autoplay.pause()`. Note that `play()` clears a user's earlier pause.

The vanilla pattern also works in React: call `splide.options = { arrows: isOverflow, ... }` inside the `overflow` handler, as in the loop example below. Those values are stored as base options, so they persist across breakpoint changes, and React Splide only overwrites the keys that are in your `options` prop. Don't mix the two for the same key: if the prop sets `arrows` and the handler sets it too, the next prop change wins.

## Slide roles after switching pagination

Slides take their role when they are created: `tabpanel` if pagination is on, `group` if not. After `pagination` switches to `false`, they keep `role="tabpanel"` with no tablist until the next `refresh()`. This also applies to the first, in-mount change, because Slides mounts before Layout. If the roles matter, refresh after the state changes:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const base: Options = { label: 'Saved items', perPage: 3, gap: '1rem', height: '8rem' };

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ overflow, setOverflow ] = useState<boolean>();
  const options: Options = { ...base, pagination: overflow !== false };

  // Slides take role="tabpanel" or "group" when they are created. Rebuild them after
  // pagination is switched, so they don't keep role="tabpanel" without a tablist.
  useEffect( () => {
    if ( overflow !== undefined ) {
      ref.current?.splide?.refresh();
    }
  }, [ overflow ] );

  return (
    <Splide
      ref={ ref }
      options={ options }
      onMounted={ splide => {
        setOverflow( splide.Components.Layout.isOverflow() );
        splide.on( 'overflow', setOverflow );
      } }
    >
      <SplideSlide>Item 1</SplideSlide>
      <SplideSlide>Item 2</SplideSlide>
    </Splide>
  );
}
```

Tested: both slides ended with `role="group"` and no pagination list. With the `refresh()` line commented out they kept `role="tabpanel"`.

`refresh()` also restores each slide's original `style` attribute (`references/auto-width.md`), so don't combine it with slide sizes driven by state through `<SplideSlide style>`.

## Loop carousels (clones) with an extension

Loop carousels need two more steps because of clones:

1. Update the `clones` option to generate or destroy clones.
2. Reset the carousel position with `go()`.

A small extension that listens in `setup()` does both, and also sees the first, in-mount call:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/overflow.css';

type Extension = NonNullable<SplideProps[ 'extensions' ]>[ string ];

// Registered in setup(), which runs before the core components mount, so it also sees the
// first `overflow` call made during mount. An extension's mount() runs too late for that.
const OverflowToggle: Extension = Splide => ( {
  setup() {
    Splide.on( 'overflow', ( isOverflow: boolean ) => {
      Splide.go( 0 ); // reset the loop position; a no-op during mount when start is 0

      Splide.options = {
        arrows    : isOverflow,
        pagination: isOverflow,
        drag      : isOverflow,
        clones    : isOverflow ? undefined : 0, // auto clones, or none
      };
    } );
  },
} );

const extensions = { OverflowToggle };

const options: Options = {
  label      : 'Partners',
  type       : 'loop',
  fixedWidth : 'min( 10rem, 20vw )',
  fixedHeight: '6rem',
  gap        : '1rem',
};

export default function Example() {
  const [ partners, setPartners ] = useState( [ 'Acme', 'Globex', 'Initech' ] );

  return (
    <>
      <Splide tag="section" options={ options } extensions={ extensions }>
        { partners.map( name => <SplideSlide key={ name }>{ name }</SplideSlide> ) }
      </Splide>

      <button
        type="button"
        className="add"
        onClick={ () => setPartners( [ ...partners, ...Array.from( { length: 9 }, ( _, i ) => `Partner ${ partners.length + i + 1 }` ) ] ) }
      >
        Add partners
      </button>
    </>
  );
}
```

`css/overflow.css` is the file from [the CSS section](#the-is-overflow-class-css-only).

Tested: with 3 partners, no `is-overflow`, no clones, no arrows, not draggable, and the list centred. After "Add partners" (12 slides, applied through React Splide's `refresh()`), `is-overflow`, clones, 2 arrows and draggable.

- `clones: undefined` tells Splide to work out the number of clones automatically.
- The extension constant lives outside the component: `extensions` is read once, at mount. Its type comes from `SplideProps`, so nothing is imported from `@splidejs/splide`.
- Inside an extension, `Splide` is the core instance, the same object as `ref.current.splide`.

From `Clones.ts`:

- The Clones component checks the required count on `updated` and `resize`. It calls `refresh()` when the count rises, or when it drops to 0. So `clones: 0` removes them and `clones: undefined` brings them back.
- `clones` only applies to `type: 'loop'`. Other types always have 0 clones.
- The automatic count, per side, is: with `fixedWidth`, `ceil( trackWidth / ( slideWidth + gap ) )`; with `autoWidth`, `Splide.length`; otherwise `perPage * 2`.
- During the first, in-mount `overflow` call, `Splide.go( 0 )` does nothing when `start` is 0, because the index is already 0. With a non-zero `start`, guard it so nothing moves during mount: `if ( Splide.state.is( 3 ) ) Splide.go( 0 );` (`3` is `IDLE` in `Splide.STATES`; React Splide doesn't export the constants).

For a `slide`-type carousel, drop `Splide.go( 0 )` and the `clones` line. With `trimSpace` (on by default), a non-overflowing slide carousel stays at position 0.

## Centering slides

When there aren't enough slides, you may want them centred rather than aligned left or right. There are two approaches: CSS and script.

### By CSS

The easiest approach uses the `is-overflow` status class, as in `css/overflow.css` above. Add `justify-content: center` to the list only when the root doesn't have `is-overflow`:

```css
.splide:not( .is-overflow ) .splide__list {
  justify-content: center;
}
```

If you use the `gap` option, remove the last slide's margin by overriding the inline style, because it shifts the slides off centre:

```css
.splide:not( .is-overflow ) .splide__slide:last-child {
  margin: 0 !important;
}
```

Vertical (`ttb`) lists are `display: block` in the core CSS, so `justify-content` has no effect there.

### By script

Prefer the CSS. The same result is possible in script:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label     : 'Partners',
  fixedWidth: '10rem',
  gap       : '1rem',
  height    : '6rem',
};

export default function Example() {
  return (
    <Splide
      tag="section"
      options={ options }
      onResized={ splide => {
        const { Layout, Elements, Slides } = splide.Components;
        const isOverflow = Layout.isOverflow();
        const lastSlide  = Slides.getAt( splide.length - 1 );

        if ( lastSlide ) {
          // Toggles `justify-content: center`
          Elements.list.style.justifyContent = isOverflow ? '' : 'center';

          // Remove the last margin (margin-left in RTL, margin-bottom in TTB)
          if ( ! isOverflow ) {
            lastSlide.slide.style.marginRight = '';
          }
        }
      } }
    >
      <SplideSlide>Acme</SplideSlide>
      <SplideSlide>Globex</SplideSlide>
      <SplideSlide>Initech</SplideSlide>
    </Splide>
  );
}
```

Tested: at 1280px the list got `justify-content: center`, the last slide's margin was cleared and the slides were centred exactly; at 400px (overflowing) the centring was removed and the margin was back (`1rem`); back at 1280px it was centred again.

- Check `Layout#isOverflow()` on `resized`, not `overflow`, and toggle `justifyContent = 'center'` on the list element to match. Layout rewrites the gap margin on every slide each time it re-lays out, then emits `resized`, so a margin cleared there survives and comes back by itself once the slides overflow. `overflow` fires only when the state changes, so a margin cleared there is overwritten by the next resize.
- `onResized` also fires during mount, so it covers the initial layout. The handler uses only its `splide` argument, so the mount-time capture of event props doesn't matter.
- The gap is written to `margin-right` in LTR, `margin-left` in RTL and `margin-bottom` in TTB. Clear the matching property for your direction. The CSS approach (`margin: 0 !important`) covers all three.
- Don't also pass a `style` prop with `margin` to `<SplideSlide>`; Splide owns that property.

## Pitfalls

- There is no `onOverflow` prop. Use `onMounted` plus `isOverflow()` and `splide.on( 'overflow' )`, or an extension's `setup()`.
- Registering `overflow` after mount (in `useEffect` or `onMounted`) misses the initial call. Read `Layout.isOverflow()` for the initial state.
- In an extension, `mount()` is too late for the initial call; use `setup()`.
- In a loop carousel, forgetting `clones: 0` leaves clones visible beside the real slides when there is not enough content.
- Centring by CSS without removing the last slide's margin leaves the slides off centre by one gap.
- `drag: isOverflow` turns `'free'` drag into normal drag when overflow returns.
- `autoplay: false` in the `options` prop has no runtime effect. Use `Components.Autoplay.pause()` through the ref.
- Switching `pagination` leaves slide roles stale until `refresh()`.
- A `className` on `<Splide>` that changes after mount wipes `is-initialized`, `is-overflow` and the other classes Splide added, and the carousel becomes invisible.
- A fade carousel always counts as overflowing.
