# Arrows

Scope: the previous and next arrows in React Splide. Generated, relocated and custom JSX arrows, buttons outside the carousel, styling, disabled state, ARIA, events and the Arrows component.

Checked against the core 4.1.4 source (`components/Arrows/Arrows.ts`, `components/Arrows/path.ts`, `components/Elements/Elements.ts`, `components/Controller/Controller.ts`), React Splide 0.7.12 (`src/js/components/Splide/Splide.tsx`), https://splidejs.com/guides/arrows/ and https://splidejs.com/integration/react-splide/. Every example was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [Setup](#setup)
- [Options](#options)
- [Generated markup](#generated-markup)
- [Changing the SVG path and adding classes](#changing-the-svg-path-and-adding-classes)
- [Placeholder: choosing the position](#placeholder-choosing-the-position)
- [Custom arrows (your own JSX)](#custom-arrows-your-own-jsx)
- [Placement rules](#placement-rules)
- [Styling](#styling)
- [Disabled state](#disabled-state)
- [ARIA attributes and i18n labels](#aria-attributes-and-i18n-labels)
- [Events](#events)
- [Arrows component](#arrows-component)
- [Buttons outside the root](#buttons-outside-the-root)
- [Pitfalls](#pitfalls)

## Setup

```sh
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';            // default theme, with arrow styles
// import '@splidejs/react-splide/css/core';    // core only: no arrow styles at all
```

The types need one tsconfig line under `moduleResolution: "bundler"` (the Vite and Next.js default); see `references/typescript.md`. In the Next.js App Router the file that renders `<Splide>` must be a client component; see `references/nextjs.md`.

Arrows are on by default. Without any arrow markup, Splide generates them and inserts them before the track.

## Options

Pass these through the `options` prop.

| Option | Type | Default | Notes |
|---|---|---|---|
| `arrows` | `boolean` | `true` | Creates the arrows. It must stay `true` when you supply arrows in JSX. With `false`, Splide generates nothing. If your JSX supplies prev/next buttons, it hides their `.splide__arrows` wrapper with inline `display: none`; an empty placeholder is left as it is. A change at runtime (a new `options` prop value or a `breakpoints` match) remounts the arrows. |
| `arrowPath` | `string` | built-in path (below) | Replaces the `d` attribute of the generated `<path>`, for example `'m7.61 0.807-2.12...'`. The SVG is 40×40. Splide uses it only for arrows it generates. |
| `classes.arrows` | `string` | `'splide__arrows'` | Class of the wrapper that Splide creates. It is not applied to a placeholder you supply. |
| `classes.arrow` | `string` | `'splide__arrow'` | Class of each generated button. |
| `classes.prev` | `string` | `'splide__arrow--prev'` | Class of the generated previous button. |
| `classes.next` | `string` | `'splide__arrow--next'` | Class of the generated next button. |
| `i18n.prev` / `i18n.next` | `string` | `'Previous slide'` / `'Next slide'` | `aria-label` values. |
| `i18n.last` / `i18n.first` | `string` | `'Go to last slide'` / `'Go to first slide'` | `aria-label` used when the move wraps around (see [ARIA](#aria-attributes-and-i18n-labels)). |
| `direction` | `'ltr' \| 'rtl' \| 'ttb'` | `'ltr'` | Adds the modifier class `splide__arrows--{direction}` to the wrapper. Themes use it to position and rotate the arrows. |

In v4, `arrows` is boolean only. The v3 value `arrows: 'slider'` no longer exists, and the v3 React props for controls (`renderControls` and friends) were removed in React Splide 0.7.

## Generated markup

This is the exact output from the source. Shown here: the first slide of a `slide`-type carousel with root id `splide01`.

```html
<div class="splide__arrows splide__arrows--ltr">
  <button class="splide__arrow splide__arrow--prev" type="button" disabled
          aria-label="Previous slide" aria-controls="splide01-track">
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 40 40" width="40" height="40" focusable="false">
      <path d="m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z"></path>
    </svg>
  </button>
  <button class="splide__arrow splide__arrow--next" type="button"
          aria-label="Next slide" aria-controls="splide01-track">
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 40 40" width="40" height="40" focusable="false">
      <path d="m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z"></path>
    </svg>
  </button>
</div>
```

Facts from the source:

- SVG: `viewBox="0 0 40 40"`, `width="40"`, `height="40"`, `focusable="false"` (stops IE from focusing the SVG), namespace `http://www.w3.org/2000/svg`.
- Both buttons draw the same right-pointing chevron. The themes flip the previous arrow with `.splide__arrow--prev svg { transform: scaleX(-1); }`. With core CSS you have to flip it yourself.
- Without a placeholder, the wrapper is inserted right before the track element, as its previous sibling in the track's parent.
- With a placeholder (`.splide__arrows` already in the JSX), the two buttons are appended inside it.
- The wrapper modifier `splide__arrows--{ltr|rtl|ttb}` always uses the default prefix, whatever `classes.arrows` is. It is removed on destroy.
- IDs: if the root has no `id`, Splide assigns `splide` plus a two-digit counter (`splide01`, `splide02`, ...). The track gets `{rootId}-track` unless it already has an id. `aria-controls` points at the track id. Pass `id` to `<Splide>` for predictable ids.

React notes:

- Generated arrows are DOM nodes that Splide creates outside React's tree. React doesn't remove them on re-render: tested with a caption that re-rendered on every move, and the buttons in the placeholder stayed.
- On unmount, `componentWillUnmount` calls `splide.destroy()`, which removes the generated buttons. Under StrictMode (development) the carousel is mounted, destroyed and mounted again, so the buttons are created twice; only the second set remains.

## Changing the SVG path and adding classes

`arrowPath` replaces the path, and the path must be drawn for a 40×40 viewBox. The path is filled by default (themes set `.splide__arrow svg { fill: #000; }`), so an open, stroke-only path draws nothing unless you style the stroke. The docs site's own demos use `'m13.5 7.01 13 13m-13 13 13-13'` with `fill: none; stroke: currentColor;`.

The `classes` option adds your classes to the wrapper and buttons that Splide generates. The CSS targets the `splide__`-prefixed classes, so list the default class together with yours, never yours alone.

- `classes` is deep-merged with the defaults, so you only pass the keys you change.
- `classes` affects only elements Splide creates. Splide finds JSX-provided arrows by the hard-coded default classes (`splide__arrows`, `splide__arrow--prev`, `splide__arrow--next`). Renaming a class in `classes` does not change where Splide looks.

Both appear in the placeholder example below (`arrowPath` plus `classes.arrows` and `classes.arrow`).

## Placeholder: choosing the position

By default, Splide inserts the arrows before the track. To choose the position yourself, render `hasTrack={ false }`, your own `<SplideTrack>`, and an empty placeholder `<div className="splide__arrows" />` where the arrows should go. Splide fills it with generated buttons.

To centre the arrows on the slides when the carousel has other content (pagination, a caption, a toggle button), put the placeholder and the track inside your own relatively positioned element. The arrows' `top: 50%` is then measured against that element instead of the whole root.

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/arrows-placeholder.css';

const options: Options = {
  label    : 'Gallery',
  height   : '12rem',
  arrowPath: 'm13.5 7.01 13 13m-13 13 13-13',
  classes  : {
    arrows: 'splide__arrows gallery-arrows',
    arrow : 'splide__arrow gallery-arrow',
  },
};

const slides = [ 'Slide 1', 'Slide 2', 'Slide 3', 'Slide 4' ];

export default function Example() {
  const [ index, setIndex ] = useState( 0 );

  return (
    <Splide
      tag="section"
      options={ options }
      hasTrack={ false }
      onMove={ ( splide, newIndex ) => setIndex( newIndex ) }
    >
      {/* Arrows are centred on the slides, not on the whole root */}
      <div className="gallery-stage">
        <div className="splide__arrows" />
        <SplideTrack>
          { slides.map( text => <SplideSlide key={ text }>{ text }</SplideSlide> ) }
        </SplideTrack>
      </div>

      <p className="gallery-caption">{ slides[ index ] } of { slides.length }</p>
    </Splide>
  );
}
```

`css/arrows-placeholder.css`:

```css
.gallery-stage { position: relative; }
.gallery-arrow svg { fill: none; stroke: currentColor; stroke-width: 3; }
```

Tested: both generated buttons are inside the placeholder with `class="splide__arrow gallery-arrow splide__arrow--prev"` and the custom path, the placeholder gets `splide__arrows--ltr` but not `gallery-arrows` (that class is only for a wrapper Splide creates), and the caption re-renders on every move without disturbing the buttons.

- Never put the placeholder inside `<SplideTrack>` (structure restriction). With the default `hasTrack`, every child of `<Splide>` is rendered inside the `<ul class="splide__list">`, so any arrows, placeholder or other control needs `hasTrack={ false }`.
- On destroy, Splide removes the generated buttons but leaves the placeholder.

## Custom arrows (your own JSX)

To use an icon, an image or text, render the buttons yourself inside a `.splide__arrows` wrapper. This example puts them in a header next to the heading, translates the labels, hides a disabled arrow with CSS, and shows a "View all" link once the carousel reaches the end:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './css/arrows-custom.css';

const options: Options = {
  labelledby: 'products-heading',
  perPage   : 2,
  gap       : '1rem',
  height    : '10rem',
  i18n      : {
    prev : 'Previous product',
    next : 'Next product',
    first: 'Go to first product',
    last : 'Go to last product',
  },
};

const products = [ 'Lamp', 'Chair', 'Desk', 'Shelf', 'Rug', 'Sofa' ];

export default function Example() {
  const [ atEnd, setAtEnd ] = useState( false );

  return (
    <Splide
      tag="section"
      options={ options }
      hasTrack={ false }
      onArrowsUpdated={ ( splide, prev, next, prevIndex, nextIndex ) => setAtEnd( nextIndex < 0 ) }
    >
      <div className="carousel-header">
        <h2 id="products-heading">Products</h2>
        <div className="splide__arrows">
          <button type="button" className="splide__arrow splide__arrow--prev">Prev</button>
          <button type="button" className="splide__arrow splide__arrow--next">Next</button>
        </div>
      </div>

      <SplideTrack>
        { products.map( name => (
          <SplideSlide key={ name }>{ name }</SplideSlide>
        ) ) }
      </SplideTrack>

      { atEnd && <a className="view-all" href="#all-products">View all products</a> }
    </Splide>
  );
}
```

`css/arrows-custom.css`:

```css
.carousel-header { display: flex; align-items: center; justify-content: space-between; }

/* Undo the theme's absolute positioning so the arrows sit in the header */
.carousel-header .splide__arrow { position: static; transform: none; width: auto; height: auto; padding: .25rem .75rem; border-radius: 4px; }
.carousel-header .splide__arrows { display: flex; gap: .5rem; }

/* Hide a disabled arrow instead of dimming it. Splide sets the disabled property. */
.carousel-header .splide__arrow:disabled { visibility: hidden; }
```

Tested: Splide labelled the buttons "Previous product" / "Next product", set `aria-controls` to the track id, disabled the previous button at the start (hidden by the CSS), and after two clicks disabled the next button and the link appeared.

Requirements and behaviour from the source:

- Both `.splide__arrow--prev` and `.splide__arrow--next` must exist. If either is missing, Splide generates a complete pair of its own (appended to your `.splide__arrows` if present), and your single button is left unmanaged.
- The buttons must sit inside an element with the class `splide__arrows`. `Arrows.ts` sets `display` on that wrapper, so if prev/next exist with no `.splide__arrows` wrapper, `mount()` throws `TypeError: Cannot read properties of undefined (reading 'style')`. In React it is thrown from `componentDidMount`, and without an error boundary React unmounts the whole tree (tested).
- Add `type="button"` yourself. Splide does not add it to buttons you supply (the docs example omits it), and a button without it submits any surrounding `<form>`.
- Splide overwrites `aria-label` on every update with the i18n strings, and sets `disabled` itself. Don't pass `aria-label` or `disabled` props to these buttons: change labels with the `i18n` option and let Splide own the state. Your `className` and children are left alone.
- `arrowPath` and `classes.*` have no effect on buttons you supply.
- On destroy (unmount, and the StrictMode remount in development), Splide removes `role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation` and `aria-roledescription` from your buttons, and leaves the elements in place. The next mount sets them again.
- `arrows: false` hides your wrapper (`display: none`) instead of removing it.
- The heading labels the carousel through `options.labelledby`, not an `aria-labelledby` prop: under StrictMode the prop is stripped by the first `destroy()` and not restored (see `references/reactivity.md`).

## Placement rules

Splide finds `.splide__arrows`, `.splide__arrow--prev` and `.splide__arrow--next` with `root.querySelector()`. It ignores a match whose closest `.splide` is not this root, which keeps nested sliders separate.

| Location | Works? |
|---|---|
| Direct child of the root (`hasTrack={ false }`), before or after `<SplideTrack>` | Yes |
| Nested anywhere inside the root, for example in a header `div` next to a heading | Yes |
| Inside `<SplideTrack>` or `<SplideSlide>`, or as a child of `<Splide>` with the default `hasTrack` (which lands in the list) | No. This breaks the structure restriction. |
| Inside a nested carousel's root | No. It belongs to that carousel. |
| Outside the root | Not detected. Use [buttons outside the root](#buttons-outside-the-root), or make a larger element the root. |

The track does not have to be a direct child of the root. To get arrows "outside" the slider visually, make `<Splide>` the outer wrapper (heading, arrows, track, footer) with `hasTrack={ false }`, and put `.splide__arrows` wherever you need it inside. Splide still handles state and ARIA for you.

## Styling

### Theme CSS (`@splidejs/react-splide/css`, verbatim rules without the `-ms-` prefixed declarations)

The CSS files are byte-identical to core 4.1.4's `splide.min.css`.

```css
.splide__arrow{align-items:center;background:#ccc;border:0;border-radius:50%;cursor:pointer;display:flex;height:2em;justify-content:center;opacity:.7;padding:0;position:absolute;top:50%;transform:translateY(-50%);width:2em;z-index:1}
.splide__arrow svg{fill:#000;height:1.2em;width:1.2em}
.splide__arrow:hover:not(:disabled){opacity:.9}
.splide__arrow:disabled{opacity:.3}
.splide__arrow:focus-visible{outline:3px solid #0bf;outline-offset:3px}
.splide__arrow--prev{left:1em}
.splide__arrow--prev svg{transform:scaleX(-1)}
.splide__arrow--next{right:1em}
.splide.is-focus-in .splide__arrow:focus{outline:3px solid #0bf;outline-offset:3px}
.splide__arrows--rtl .splide__arrow--prev{left:auto;right:1em}
.splide__arrows--rtl .splide__arrow--prev svg{transform:scaleX(1)}
.splide__arrows--rtl .splide__arrow--next{left:1em;right:auto}
.splide__arrows--rtl .splide__arrow--next svg{transform:scaleX(-1)}
.splide__arrows--ttb .splide__arrow{left:50%;transform:translate(-50%)}
.splide__arrows--ttb .splide__arrow--prev{top:1em}
.splide__arrows--ttb .splide__arrow--prev svg{transform:rotate(-90deg)}
.splide__arrows--ttb .splide__arrow--next{bottom:1em;top:auto}
.splide__arrows--ttb .splide__arrow--next svg{transform:rotate(90deg)}
```

The other themes are `@splidejs/react-splide/css/default` (the same as `/css`), `/css/sea-green` and `/css/skyblue`.

### Core CSS only (`@splidejs/react-splide/css/core`)

Core CSS has no arrow rules at all. Generated arrows render as plain 40×40 SVG buttons in normal flow. Core does set `.splide { position: relative; }`, so absolutely positioned arrows are placed against the root. A starting point that mirrors the theme:

```css
.splide__arrow {
  position: absolute; top: 50%; transform: translateY(-50%); z-index: 1;
  display: flex; align-items: center; justify-content: center;
  width: 2.5rem; height: 2.5rem; padding: 0;
  border: 0; border-radius: 50%; background: #fff; color: #111; cursor: pointer;
}
.splide__arrow svg { width: 1.2em; height: 1.2em; fill: currentColor; }
.splide__arrow--prev { left: 1rem; }
.splide__arrow--prev svg { transform: scaleX(-1); }   /* the default path points right */
.splide__arrow--next { right: 1rem; }
.splide__arrow:disabled { opacity: .3; cursor: default; }

/* Focus: :focus-visible plus the is-focus-in fallback (added to the root on keyboard focus) */
.splide__arrow:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__arrow:focus { outline: 3px solid #0bf; outline-offset: 3px; }

/* RTL and vertical carousels */
.splide__arrows--rtl .splide__arrow--prev { left: auto; right: 1rem; }
.splide__arrows--rtl .splide__arrow--prev svg { transform: scaleX(1); }
.splide__arrows--rtl .splide__arrow--next { left: 1rem; right: auto; }
.splide__arrows--rtl .splide__arrow--next svg { transform: scaleX(-1); }
.splide__arrows--ttb .splide__arrow { left: 50%; transform: translate(-50%); }
.splide__arrows--ttb .splide__arrow--prev { top: 1rem; }
.splide__arrows--ttb .splide__arrow--prev svg { transform: rotate(-90deg); }
.splide__arrows--ttb .splide__arrow--next { top: auto; bottom: 1rem; }
.splide__arrows--ttb .splide__arrow--next svg { transform: rotate(90deg); }
```

`is-focus-in` is not removed the moment focus leaves the carousel, so always pair it with `:focus`.

Other styling notes:

- Arrows outside the slide area: use negative offsets (`.splide__arrow--prev { left: -3rem; }`) and leave room with padding or margin on a wrapper. The arrows are not inside `.splide__track` (`overflow: hidden`), so the track does not clip them. Only an ancestor with `overflow: hidden` would.
- Hide arrows when all slides fit: `.splide:not(.is-overflow) .splide__arrows { display: none; }` (v4.1+, see `references/overflow.md`).
- Hide instead of dim: `.splide__arrow:disabled { visibility: hidden; }`, as in the custom arrows example. The theme sets `display: flex` on the buttons, so use `visibility` (or a class), not the `hidden` attribute.

## Disabled state

Splide sets the buttons' `disabled` property itself:

```js
prev.disabled = Controller.getPrev() < 0;
next.disabled = Controller.getNext() < 0;
```

- The state is recalculated on mount and on the `mounted`, `moved`, `refresh` and `scrolled` events. It is also recalculated on an internal event that fires after a resize changes the end index. It is not recalculated on `move`, so the state changes when the transition finishes.
- `type: 'loop'` never disables either arrow.
- `rewind: true` never disables them when there are enough slides, because a move past the end wraps around.
- In `type: 'slide'` without `rewind`, prev is disabled at the first position and next at the end index.
- With `autoWidth` or `fixedWidth`, next can stay enabled on the last page even when pressing it only moves the active slide. Add `focus: 0, omitEnd: true` (v4.1+) to disable it once the carousel can't move further.
- In `type: 'slide'`, if there are no more slides than `perPage` and `focus` is not set, both arrows are disabled.
- The theme dims disabled arrows (`.splide__arrow:disabled { opacity: .3 }`). Core CSS has no disabled style.

## ARIA attributes and i18n labels

| Attribute | Value |
|---|---|
| `aria-controls` | The track id, for example `splide01-track`. |
| `aria-label` (prev) | `i18n.prev` ("Previous slide"). It becomes `i18n.last` ("Go to last slide") when the previous move would wrap to the end. |
| `aria-label` (next) | `i18n.next` ("Next slide"). It becomes `i18n.first` ("Go to first slide") when the next move would wrap to the start. |
| `disabled` | Set when there is no slide to go to. |

The label logic in the source:

```js
const prevLabel = prevIndex > -1 && index < prevIndex ? i18n.last  : i18n.prev;
const nextLabel = nextIndex > -1 && index > nextIndex ? i18n.first : i18n.next;
```

So with `rewind: true`, and in `loop` mode, the prev arrow at index 0 reads "Go to last slide" and the next arrow at the last index reads "Go to first slide".

Docs say (i18n guide table): `first` is the "aria-label of a previous arrow" and `last` is the "aria-label of a next arrow". Source says: `last` is used on the previous arrow and `first` on the next arrow.

Translate the labels, or make them match the content, with `options.i18n` (as in the custom arrows example). For every carousel in the app, share an options object (`export const baseOptions: Options = { i18n: { ... } }`) and spread it into each `options` prop. Setting `Splide.defaults` on `@splidejs/splide` has no effect, because React Splide bundles its own copy of core.

## Events

React Splide has an event prop for both arrow events. Each handler gets the core instance first, then the core arguments.

| Prop | Core event | Arguments | Fired |
|---|---|---|---|
| `onArrowsMounted` | `arrows:mounted` | `splide`, `prev: HTMLButtonElement`, `next: HTMLButtonElement` | Right after the arrows are mounted. It fires only when `arrows` is `true` and both buttons exist. It fires again after every remount, which happens on each `updated` event (a breakpoint change or a changed `options` prop). |
| `onArrowsUpdated` | `arrows:updated` | `splide`, `prev`, `next`, `prevIndex: number`, `nextIndex: number` | Whenever the arrow status (disabled, labels) is updated. `prevIndex` and `nextIndex` are the indexes the prev and next arrows would go to (`Controller.getPrev()` / `Controller.getNext()`), or `-1` when that arrow is disabled. |

- Docs say `prevIndex` is "An old index" and `nextIndex` is "A new index". Source says they are the destinations of the prev and next arrows (`getPrev()` / `getNext()`).
- On mount, the first `arrows:updated` fires before `arrows:mounted`. Another `arrows:updated` follows on the core `mounted` event.
- React Splide binds the event props in `componentDidMount`, before `splide.mount()`, so `onArrowsMounted` does fire for the first mount.
- The props are read once, at mount. A handler keeps the props and state of the first render; calling a `setState` function is fine (it never changes), as in `setAtEnd( nextIndex < 0 )` above. For other values use a ref (`references/reactivity.md`).
- `onArrowsMounted` and `onArrowsUpdated` don't fire with `arrows: false`. For your own external buttons, see below.

## Arrows component

`ref.current?.splide?.Components.Arrows` is available after mount (in effects and handlers, not during render).

| Property | Type | Description |
|---|---|---|
| `arrows` | `{ prev?: HTMLButtonElement, next?: HTMLButtonElement }` | The previous and next buttons, whether you supplied them in JSX or Splide generated them. It is empty only when `arrows` is `false` at mount and the JSX has no buttons. JSX buttons stay in it even with `arrows: false`, and after a runtime switch to `arrows: false` it still holds the removed generated buttons (the object is never cleared). |

`splide.Components.Elements` also exposes `arrows` (the wrapper), `prev` and `next` as they were found in the markup at mount time. Generated buttons appear only in `Arrows.arrows`.

## Buttons outside the root

Splide never finds buttons outside the root. Set `arrows: false` and drive the carousel through the ref:

| Call | Moves to |
|---|---|
| `ref.current?.go( '<' )` | Previous page or destination. Respects `perPage`, `perMove` and `focus`. |
| `ref.current?.go( '>' )` | Next page or destination. |
| `ref.current?.go( '-1' )` / `ref.current?.go( '+1' )` | Previous or next slide by index. |
| `ref.current?.go( '>2' )` | Page index 2, the third page (page indexes start at 0). With `perPage: 3` that is slide index 6. |

`ref.current?.go()` is the component's own method; it calls `ref.current.splide?.go()`, so the two are the same.

To keep the buttons accessible, give them `aria-controls` (the track id), a label, and a `disabled` state that follows the carousel. The recipe below copies the built-in logic into React state:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label  : 'Products',
  arrows : false,
  perPage: 3,
  gap    : '1rem',
  height : '8rem',
};

type NavState = {
  trackId     : string;
  prevDisabled: boolean;
  nextDisabled: boolean;
  prevLabel   : string;
  nextLabel   : string;
};

const products = [ 'Lamp', 'Chair', 'Desk', 'Shelf', 'Rug', 'Sofa' ];

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ nav, setNav ] = useState<NavState>();

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( ! splide ) {
      return;
    }

    const { Controller, Elements } = splide.Components;
    // Namespaced, so off() removes only these handlers.
    const events = 'moved.nav scrolled.nav refresh.nav resized.nav updated.nav';

    function update() {
      const i18n      = splide!.options.i18n!;
      const index     = splide!.index;
      const prevIndex = Controller.getPrev(); // -1 when there is nowhere to go
      const nextIndex = Controller.getNext();

      setNav( {
        trackId     : Elements.track.id,
        prevDisabled: prevIndex < 0,
        nextDisabled: nextIndex < 0,
        // Same labels as the built-in arrows
        prevLabel: prevIndex > -1 && index < prevIndex ? i18n.last : i18n.prev,
        nextLabel: nextIndex > -1 && index > nextIndex ? i18n.first : i18n.next,
      } );
    }

    update();
    // Registered after mount, so these run after Controller has recalculated on updated / refresh.
    splide.on( events, update );

    return () => {
      splide.off( events );
    };
  }, [] );

  return (
    <>
      <div className="product-controls">
        <button
          type="button"
          onClick={ () => ref.current?.go( '<' ) }
          disabled={ nav?.prevDisabled }
          aria-label={ nav?.prevLabel }
          aria-controls={ nav?.trackId }
        >
          Previous
        </button>
        <button
          type="button"
          onClick={ () => ref.current?.go( '>' ) }
          disabled={ nav?.nextDisabled }
          aria-label={ nav?.nextLabel }
          aria-controls={ nav?.trackId }
        >
          Next
        </button>
      </div>

      <Splide ref={ ref } tag="section" options={ options }>
        { products.map( name => <SplideSlide key={ name }>{ name }</SplideSlide> ) }
      </Splide>
    </>
  );
}
```

Tested: at the start "Previous" is disabled and both buttons have `aria-controls` ending in `-track`; after "Next" the carousel shows "Shelf" first and "Next" is disabled; after "Previous" it is back at "Lamp".

Notes:

- The listeners are registered with `splide.on()` in an effect, after mount, on purpose. Event props are bound before `mount()`, so an `onUpdated` or `onRefresh` handler runs before Controller has recalculated its end index. Tested: after `perPage` changed from 3 to 2, an `onUpdated` handler read `Controller.getEnd()` as 3, the old value; the real end index was 4. `moved` and `scrolled` have no such problem, so `onMoved` alone would be fine for a fixed layout.
- Use a namespace (`moved.nav`) and remove only that namespace in the cleanup. `splide.off( 'moved' )` without a namespace also removes React Splide's handler for an `onMoved` prop, because the props are registered with `splide.on()` too (tested).
- The effect runs once with the instance that exists after mount. Under StrictMode the first instance is destroyed (which drops its listeners) and the effect runs again for the second one.
- Built-in arrows call `Controller.go( control, true )`, where the second argument allows a "move" to the current index (used to correct the position). `splide.go()` does not pass it. For exactly the same behaviour, call `ref.current?.splide?.Components.Controller.go( '>', true )`.
- The root is a `region` landmark by default, and buttons outside it sit outside that landmark. Where you can, prefer a larger root containing a `.splide__arrows` placeholder ([Placement rules](#placement-rules)); Splide then manages everything.
- Splide's live region (the track's `aria-live`) announces slide changes, so you don't need extra announcements for external buttons.

## Pitfalls

- With the default `hasTrack`, children of `<Splide>` go into the slide list. Arrows and placeholders need `hasTrack={ false }` and your own `<SplideTrack>`.
- Custom prev/next buttons without a `.splide__arrows` wrapper make `mount()` throw from `componentDidMount`.
- Custom arrows must be `<button type="button">`. `<a>` and `<div>` can't be disabled and break focus handling.
- An `aria-label` or `disabled` prop on a custom arrow is overwritten. Use `i18n`, and style `:disabled`.
- With core CSS, the prev arrow points right until you add `transform: scaleX(-1)` to its SVG.
- `classes` only renames generated elements. Splide always finds JSX arrows by the default classes.
- Theme arrows are positioned against the root (`top: 50%`). Pagination or other content inside the root shifts the centre; wrap the track and placeholder in a `position: relative` element.
- Event props are captured at mount and run before core's own handlers on `updated` and `refresh`. Read Controller state in handlers registered with `splide.on()` after mount.
- `Splide.defaults` from `@splidejs/splide` doesn't reach `<Splide>`. Share an options object for app-wide labels.
