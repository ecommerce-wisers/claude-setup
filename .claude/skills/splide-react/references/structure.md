# Structure

Scope: what `<Splide>`, `<SplideTrack>` and `<SplideSlide>` render, `hasTrack`, where your own elements go in JSX, every data attribute Splide reads, and every attribute and ARIA value Splide adds. Checked against React Splide 0.7.12 (`src/js/components/`) and the core 4.1.4 source (`components/Elements`, `Slides/Slide`, `Arrows`, `Pagination`, `Autoplay`, `Live`, `LazyLoad`, `Layout`, `Clones`, `Cover`, `Sync`, `constants/classes.ts`, `constants/attributes.ts`). The React examples were run in headless Chrome (React 19.3.1, production and development with StrictMode) and the rendered DOM was checked.

## Contents

- [Basic structure](#basic-structure)
- [The three components](#the-three-components)
- [Roles](#roles)
- [Element reference](#element-reference)
- [hasTrack and where your own elements go](#hastrack-and-where-your-own-elements-go)
- [Restrictions](#restrictions)
- [What Splide inserts itself](#what-splide-inserts-itself)
- [Using div instead of ul/li](#using-div-instead-of-ulli)
- [Slide container](#slide-container)
- [Arrows](#arrows)
- [Pagination](#pagination)
- [Progress bar and autoplay toggle](#progress-bar-and-autoplay-toggle)
- [Data attributes](#data-attributes)
- [IDs Splide assigns](#ids-splide-assigns)
- [Attributes Splide adds to each element](#attributes-splide-adds-to-each-element)
- [Inline styles Splide writes](#inline-styles-splide-writes)
- [What unmounting removes](#what-unmounting-removes)
- [When React and Splide both write the DOM](#when-react-and-splide-both-write-the-dom)
- [Complete example with every optional element](#complete-example-with-every-optional-element)

---

## Basic structure

Splide needs four elements: `root`, `track`, `list` and `slide`. With the default `hasTrack`, `<Splide>` renders the first three and each `<SplideSlide>` is one slide:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DefaultStructure() {
  return (
    <>
      <Splide options={ { label: 'Div root' } }>
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
        <SplideSlide>03</SplideSlide>
      </Splide>

      <Splide tag="section" id="featured" options={ { label: 'Section root' } }>
        <SplideSlide className="card">01</SplideSlide>
        <SplideSlide className="card" aria-label="Summer sale">02</SplideSlide>
        <SplideSlide className="card">03</SplideSlide>
      </Splide>

      <Splide options={ { label: 'Extensions attribute' } } extensions={ {} }>
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
      </Splide>
    </>
  );
}
```

The first carousel renders this (tested; `...` marks attributes covered below):

```html
<div class="splide splide--slide splide--ltr splide--draggable is-active is-overflow is-initialized"
  id="splide01" role="region" aria-roledescription="carousel" aria-label="Div root">
  <div class="splide__arrows splide__arrows--ltr">...generated prev/next buttons...</div>
  <div class="splide__track splide__track--slide splide__track--ltr splide__track--draggable"
    id="splide01-track" aria-live="polite" aria-atomic="true" ...>
    <ul class="splide__list" id="splide01-list" role="presentation" style="transform: ...">
      <li class="splide__slide is-active is-visible" id="splide01-slide01"
        role="tabpanel" aria-roledescription="slide" aria-label="1 of 3" style="width: calc(100%);">01</li>
      <li class="splide__slide is-next" id="splide01-slide02" ... aria-hidden="true">02</li>
      ...
    </ul>
  </div>
  <ul class="splide__pagination splide__pagination--ltr" role="tablist" aria-label="Select a slide to show">...</ul>
</div>
```

Give the root a concise label. Use `options.label`, or `options.labelledby` with the ID of a visible heading. The `aria-label` / `aria-labelledby` props also reach the root, but in development with StrictMode Splide's `destroy()` strips them during the remount and React doesn't put them back (tested). `options.label` and `options.labelledby` both default to the root's existing attributes.

About the word "carousel" in the label: W3C suggests leaving it out, because Splide sets `aria-roledescription="carousel"` and some screen readers announce "label" + "description". That behavior isn't required by the spec, though, and some screen readers ignore the description. The docs' author thinks including "carousel" is acceptable. Some users may hear it twice, which is a small cost compared with hearing nothing.

---

## The three components

| Component | Renders | Props |
|---|---|---|
| `<Splide>` | `<{tag} class="splide {className}">`, then `<SplideTrack>{ children }</SplideTrack>`, or the children as they are with `hasTrack={ false }` | `options`, `tag` (`'div'` default, `'section'`, `'header'`, `'footer'`, `'nav'`), `hasTrack` (default `true`), `extensions`, `transition`, the 28 event props (`references/events.md`), `className`, `id`, `aria-*`, `data-*`, `style` and other HTML attributes |
| `<SplideTrack>` | `<div class="splide__track {className}"><ul class="splide__list">{ children }</ul></div>` | Any `<div>` attribute (`className`, `id`, ...). The props go on the track `<div>`. The `<ul>` can't be changed. |
| `<SplideSlide>` | `<li class="splide__slide {className}">{ children }</li>` | Any `<li>` attribute, passed straight through (`data-splide-interval`, `data-splide-hash`, `aria-label`, `style`, ...) |

From `Splide.render()`:

- Only `options` and the event props are removed before the remaining props are spread onto the root. So `extensions` and `transition` also end up on the root as attributes. Tested: `extensions={ {} }` rendered `extensions="[object Object]"`. Harmless, but visible in the DOM.
- The prop type is `SplideHTMLAttributes`, which omits React's DOM event handlers. `onClick` on `<Splide>` is Splide's slide `click` event (`( splide, Slide, e )`), not a DOM click, and handlers like `onKeyDown` aren't typed. Put DOM listeners on a wrapper element or on the slide content.
- The root element is created by React with `ref={ this.splideRef }` (a `RefObject<HTMLDivElement>` on the component instance). The core instance is `ref.current.splide`. See `references/getting-started.md`.
- Don't change `tag` or `hasTrack` after mount. React would replace the root, or the track and list, under a live instance. Change the `key` to remount instead (`references/reactivity.md`).

---

## Roles

### Landmark role

Many assistive technologies can jump between landmarks (important page areas such as `banner` and `navigation`). A landmark works as an invisible but very useful table of contents. A `<section>` with an accessible name has an implicit `region` landmark role, so it's a better root than a `<div>`.

With `tag="section"` the root has the implicit `region` role, and Splide doesn't add one. With the default `<div>`, Splide sets `role="region"` itself (the `role` option, default `'region'`), which gives the same result. Tested: the `<div>` root got `role="region"`, the `<section>` root got no `role` attribute.

Source rule (`Elements.init()`): `role` is set only when the root has **no** `role` attribute, is **not** a `<section>`, and the `role` option is truthy. Splide never changes an existing role, because that would remove the region from the accessibility tree.

`tag="header"`, `"footer"` or `"nav"`: the docs say to provide the most appropriate landmark role with them. Splide's rule above still applies, so without your own role a `<nav>` root gets `role="region"`, which replaces its implicit `navigation` role (from source). Set the role you want with `options.role`.

### Group role

For a carousel that isn't significant or isn't directly related to the main content, use the `group` role. Set it through `options.role`, not the `role` prop:

```tsx
<Splide options={ { role: 'group', label: 'Unimportant content' } }>
```

Tested: in production, `role="group"` as a prop and `options.role: 'group'` both give `role="group"`. In development with StrictMode, the prop version ended up as `role="region"` (Splide's `destroy()` removed the prop's `role`, and the remount added the default), while `options.role` stayed `group`. `options.role` takes effect only on a non-`<section>` root without its own `role`.

---

## Element reference

| Element | Class | Required | Placement rule (from source) | Recommended tag |
|---|---|---|---|---|
| Root | `splide` | yes | The constructor target | `<section>` or `<div>` |
| Track | `splide__track` | yes | Anywhere inside the root, found with `root.querySelector( '.splide__track' )`. Only the first match is checked, and it must not belong to a nested carousel. | `<div>` |
| List | `splide__list` | yes | **Direct child** of the track | `<ul>` or `<div>` |
| Slide | `splide__slide` | yes (0 slides is allowed, but nothing shows) | **Direct children** of the list. Only children with the class count. | `<li>` or `<div>` |
| Slide container | `splide__slide__container` | no | **Direct child** of a slide | `<div>` |
| Arrows wrapper | `splide__arrows` | no (required if you write custom arrows) | Anywhere in the root **except inside the track/list** | `<div>` |
| Prev / next arrow | `splide__arrow splide__arrow--prev` / `splide__arrow splide__arrow--next` | no | Anywhere in the root outside the track. Both must exist, or Splide generates its own. In practice inside `.splide__arrows`. | `<button type="button">` |
| Pagination placeholder | `splide__pagination` | no | Anywhere in the root outside the track | empty `<ul>` |
| Progress container | `splide__progress` | no | Anywhere in the root outside the track (only the bar is looked up) | `<div>` |
| Progress bar | `splide__progress__bar` | no | Anywhere in the root outside the track | `<div>` |
| Autoplay toggle | `splide__toggle` | no | Anywhere in the root outside the track | `<button type="button">` |
| Toggle icons | `splide__toggle__play`, `splide__toggle__pause` | with the toggle | Inside `.splide__toggle` | any inline element (`<span>`, `<img>`, `<svg>`) |
| Lazy spinner | `splide__spinner` | no (auto-created) | Sibling of a lazy `<img>` (same parent) | `<span>` |

Splide looks up the arrows wrapper, prev/next arrows, pagination placeholder, progress bar and toggle anywhere in the root with `querySelector`. It keeps an element only if `element.closest( '.splide' ) === root`, so controls inside a **nested** carousel are ignored. The track lookup works the same way.

Only the **first** match is checked. If it belongs to a nested carousel, Splide treats the element as missing and doesn't look further. Verified: with a nested carousel mounted before its parent, the parent's `.splide__arrows` and `.splide__pagination` placeholders placed after the track stayed empty, and the parent generated new arrows and pagination, because the nested carousel's generated controls came first in document order. Put the parent's own controls before its track, or mount the parent first (`querySelectorAll( '.splide' )` order does this).

In React, `<SplideTrack>` provides the track and the list, `<SplideSlide>` the slides. Everything else in the table is markup you write yourself as children of `<Splide hasTrack={ false }>`.

---

## hasTrack and where your own elements go

With the default `hasTrack={ true }`, every child of `<Splide>` goes **inside the `<ul class="splide__list">`**. Only `<SplideSlide>` children belong there. Tested: an `<h2>` child became the list's first child (the carousel still mounted, with a broken layout).

With `hasTrack={ false }`, the children render directly inside the root, and you place `<SplideTrack>` yourself. The docs show that these render the same HTML:

```tsx
<Splide>
  <SplideSlide>...</SplideSlide>
</Splide>

<Splide hasTrack={ false }>
  <SplideTrack>
    <SplideSlide>...</SplideSlide>
  </SplideTrack>
</Splide>
```

| Location in JSX | Allowed? | Notes |
|---|---|---|
| Children of `<Splide hasTrack={ false }>`, before or after `<SplideTrack>` (headings, captions, counters, buttons, placeholders) | Yes | Arrows and pagination are positioned against the nearest positioned ancestor (the root by default, which has `position: relative`). Wrap the track in a `position: relative` element if arrows should center on the slides only. |
| Wrapper elements around `<SplideTrack>` | Yes | Generated arrows and pagination are inserted into the track's direct parent |
| Inside a `<SplideSlide>` (any content: images, text, links, forms, even another `<Splide>`) | Yes | Focusable elements in hidden slides get `tabindex="-1"` (`focusableNodes`) |
| Children of `<Splide>` with the default `hasTrack` that aren't `<SplideSlide>` | **No** | They land inside the `<ul>` and break layout and indexing |
| Children of `<SplideTrack>` that aren't `<SplideSlide>` | **No** | Same: they land inside the `<ul>` |
| Arrows/pagination placeholders inside `<SplideTrack>` | **No** | Docs: "do not insert the placeholder in the track element" |
| Controls outside `<Splide>` | Not picked up | Splide doesn't find them. Wire them yourself: `onClick={ () => ref.current?.go( '>' ) }`. |

Re-rendering is safe for placeholders. React doesn't touch children it didn't create, so a `<div className="splide__arrows" />` or `<ul className="splide__pagination" />` keeps the buttons Splide put inside it. Tested: after three state updates that re-rendered a heading next to them, the placeholders still held 2 arrows and 3 dots. Keep placeholders empty in JSX.

---

## Restrictions

Keep two rules:

- Don't insert any elements inside the `track` and `list` elements. With `<SplideTrack>`, that means its only children are `<SplideSlide>`s.
- Don't break the `track > list > slide` relation. The list must be a child of the track, and slides must be children of the list. `<SplideTrack>` and `<SplideSlide>` guarantee this.

The track does **not** have to be a direct child of the root, so any number of wrappers between `<Splide hasTrack={ false }>` and `<SplideTrack>` is valid. If you write the track by hand, a wrapper between the track and the list throws `[splide] A track/list element is missing.`, and so does `hasTrack={ false }` without any track (tested).

Don't let the styles of wrapper elements break the core components. For example, don't give a wrapper `overflow`, `display: flex` with shrinking, or a `transform` that changes how the track measures its width.

What happens with stray elements (from source):

- A non-slide child of the list (no `splide__slide` class) isn't counted as a slide (`splide.length` excludes it). It still sits in the flex row and breaks positions and widths.
- An extra element inside the track (beside the list) shifts the list and confuses visibility checks. Splide itself only appends a visually hidden `.splide__sr` span there for the live region.
- If the root contains several `.splide__track` elements that aren't in nested carousels, only the first is used.

---

## What Splide inserts itself

| Inserted element | Where | When |
|---|---|---|
| `<div class="splide__arrows splide__arrows--{dir}">` with two `<button>`s | Immediately **before the track** (`track.parentNode.insertBefore( wrapper, track )`). If a `.splide__arrows` placeholder exists, the buttons are appended **inside it**. | `arrows: true` and no complete custom prev/next pair |
| `<ul class="splide__pagination splide__pagination--{dir}">` | **Appended to the track's parent element** (after the track and anything after it). If a `.splide__pagination` placeholder exists, it's filled instead. | `pagination: true` |
| `<span class="splide__sr">…</span>` | Appended inside the track for about 90 ms after each `moved` / `scrolled` | `live: true` and not `isNavigation` |
| `<span class="splide__spinner">` | Appended to the lazy image's parent element, unless a `.splide__spinner` child already exists there | `lazyLoad` and the image hasn't loaded yet |
| Clone slides `.splide__slide.splide__slide--clone` | Before the first slide and after the last slide, inside the list | `type: 'loop'` |

If the track sits inside a wrapper, the generated arrows and pagination go **inside that wrapper** (the track's parent), not into the root.

These nodes are created by Splide, outside React's virtual DOM. React leaves them alone on re-render (it only updates nodes it rendered), and `componentWillUnmount` calls `splide.destroy()`, which removes the generated arrows, pagination and clones before React removes the root. Tested: the root's children were the generated `div.splide__arrows`, the track, and the generated `ul.splide__pagination`, in that order.

---

## Using div instead of ul/li

`<SplideTrack>` always renders a `<ul>` and `<SplideSlide>` always an `<li>`. For `<div>` slides, write the track yourself inside `<Splide hasTrack={ false }>`. Lookups use classes, not tag names:

```tsx
import { Splide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DivSlides() {
  return (
    <Splide hasTrack={ false } options={ { label: 'Div slides', perPage: 2 } }>
      <div className="splide__track">
        <div className="splide__list">
          <div className="splide__slide">01</div>
          <div className="splide__slide">02</div>
          <div className="splide__slide">03</div>
        </div>
      </div>
    </Splide>
  );
}
```

Tested: the `<div>` list got `role="presentation"`, and each `<div>` slide got its `aria-label` (`1 of 3`). The list gets `role="presentation"` either way, so `<ul>` list semantics are removed anyway. A `<div>` root gets `role="region"` unless you set a role. Generated pagination is always a `<ul>` with `<li>`/`<button>` (the pagination placeholder should also be a `<ul>`). With hand-written slides, React still owns the elements, so map over data with a `key` as usual. Adding or removing them triggers `refresh()` just like `<SplideSlide>` children.

---

## Slide container

A container sets a height on part of a slide while the slide's own height adapts to its content. It's especially useful with `cover` mode when slides also contain headings or descriptions. It must be a **direct child** of the slide:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const svg = ( color: string ) => 'data:image/svg+xml,' + encodeURIComponent(
  `<svg xmlns="http://www.w3.org/2000/svg" width="800" height="400"><rect width="800" height="400" fill="${ color }"/></svg>`,
);

const products = [
  { name: 'Product 1', image: svg( '#c33' ), interval: 3000 },
  { name: 'Product 2', image: svg( '#3c3' ) },
  { name: 'Product 3', image: svg( '#33c' ) },
];

export default function ProductSlides() {
  return (
    <Splide
      options={ {
        label      : 'Products',
        perPage    : 2,
        gap        : '1rem',
        lazyLoad   : 'nearby',
        cover      : true,
        heightRatio: 0.4,
      } }
    >
      { products.map( product => (
        <SplideSlide key={ product.name } data-splide-interval={ product.interval }>
          <div className="splide__slide__container">
            <img data-splide-lazy={ product.image } alt={ product.name } />
          </div>
          <p>{ product.name }</p>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Tested: `height` (from `heightRatio`) and the `cover` background went on each `.splide__slide__container`, not on the slide. Each `<img>` got `display: none` after loading, and `data-splide-lazy` was removed. The `data-splide-interval` attribute reached the first `<li>`, and was omitted from the others because their value is `undefined`. The SVG data URLs only keep the example self-contained. Use real image URLs.

Splide applies `height`, `heightRatio` or `fixedHeight` to the containers instead of the slides.

From source:

- The container is `child( slide, '.splide__slide__container' )`, so only a direct child counts, and only the first one.
- `Slide.style( prop, value, true )` and `Slides.style( ..., true )` target the container when there is one, and the slide otherwise.
- With `cover: true`, Splide takes the first `<img>` that is a **direct child** of the container (or of the slide when there's no container) and has a `src`. It sets `background: center/cover no-repeat url("<src>")` on the container (or slide) and hides the `<img>` with `display: none`. With `lazyLoad`, this happens after the image loads.
- Only slide widths go on the slides themselves (`width`, or `fixedWidth`). The gap goes on the slides as `margin-right` (`margin-left` for RTL, `margin-bottom` for TTB).
- The core CSS rule `.splide__container { box-sizing: border-box; position: relative }` does **not** match `.splide__slide__container` (dead selector). Add your own styles if you need them.

---

## Arrows

### Generated markup

```html
<div class="splide__arrows splide__arrows--ltr">
  <button
    class="splide__arrow splide__arrow--prev"
    type="button"
    aria-label="Previous slide"
    aria-controls="splide01-track"
  >
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 40 40" width="40" height="40" focusable="false">
      <path d="m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z" />
    </svg>
  </button>
  <button
    class="splide__arrow splide__arrow--next"
    type="button"
    aria-label="Next slide"
    aria-controls="splide01-track"
  >
    <svg>...</svg>
  </button>
</div>
```

- The SVG path comes from the `arrowPath` option. The SVG is 40×40.
- The wrapper class follows `direction`: `splide__arrows--ltr`, `--rtl` or `--ttb`.
- `disabled` is set when there's no slide to go to (for example the first slide in `slide` type without `rewind`).
- `aria-label`: prev = i18n `prev` ("Previous slide"), or `last` ("Go to last slide") when the previous move would wrap to the end. Next = `next` ("Next slide"), or `first` ("Go to first slide") when it would wrap to the start.
- Extra classes: the `classes` option (`arrows`, `arrow`, `prev`, `next`). Keep the `splide__` classes, because the CSS targets them.

### Placeholder (choose where generated arrows go)

By default the arrows go right before the track. Put an empty `<div className="splide__arrows" />` where you want them. To center arrows vertically on the slides while other controls are in the root, put the placeholder inside your own relative element:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function CustomStructure() {
  return (
    <Splide
      tag="section"
      hasTrack={ false }
      options={ { labelledby: 'promo-heading', autoplay: true, interval: 3000, type: 'loop' } }
    >
      <h2 id="promo-heading">Promotions</h2>

      { /* relative wrapper so arrows center on the slides, not on the heading */ }
      <div className="carousel-stage" style={ { position: 'relative' } }>
        <div className="splide__arrows" />

        <SplideTrack>
          <SplideSlide>01</SplideSlide>
          <SplideSlide>02</SplideSlide>
          <SplideSlide>03</SplideSlide>
        </SplideTrack>
      </div>

      <ul className="splide__pagination" />

      <div className="splide__progress">
        <div className="splide__progress__bar" />
      </div>

      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

Tested: the generated buttons were inserted into the placeholder inside `.carousel-stage`, the pagination placeholder got 3 dots, the toggle read `Pause autoplay` with `is-active` and `aria-controls` set to the track ID, the track's `aria-live` was `off` while autoplay played, and the progress bar width moved. After clicking the toggle: no `is-active`, `Start autoplay`, and `aria-live="polite"`.

Never put the placeholder inside `<SplideTrack>`.

### Custom arrows

For icon fonts, images or text, write the arrows yourself:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function CustomArrows() {
  return (
    <Splide hasTrack={ false } options={ { label: 'Custom arrows' } }>
      <div className="splide__arrows">
        <button className="splide__arrow splide__arrow--prev" type="button">Prev</button>
        <button className="splide__arrow splide__arrow--next" type="button">Next</button>
      </div>

      <SplideTrack>
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
        <SplideSlide>03</SplideSlide>
      </SplideTrack>
    </Splide>
  );
}
```

Tested: no extra arrows were generated, Splide set `aria-label="Previous slide"` / `"Next slide"` and `aria-controls="splide01-track"`, the prev arrow started `disabled`, and clicking Next moved to slide 2 and enabled Prev.

`.splide__arrows` can go anywhere that keeps the structure rules. Each arrow must be a `<button>`, so it can receive focus and a `disabled` attribute.

Source facts for custom arrows (verified):

- Splide uses them only when **both** `.splide__arrow--prev` and `.splide__arrow--next` exist. Otherwise it generates arrows (inside `.splide__arrows` if present).
- **The `.splide__arrows` wrapper is required.** With custom prev/next buttons and no wrapper, `mount()` throws `TypeError: Cannot read properties of undefined (reading 'style')`, because Splide sets `display` and a modifier class on the wrapper. The docs example includes the wrapper but doesn't say it's required.
- Splide sets `aria-controls` (track ID), `aria-label` (from i18n, **overwriting** your own `aria-label`) and `disabled`. Change the text through the `i18n` option (`prev`, `next`, `first`, `last`).
- Splide doesn't add `type="button"` to custom arrows. Add it yourself, so they don't submit a surrounding `<form>`.
- `.splide__arrow` isn't needed to find the arrows, but keep it. The themes style it, and drag doesn't start on elements matching `.splide__arrow` or `.splide__pagination__page`.
- `arrows: false` with custom arrows: the wrapper gets `display: none`.
- On `destroy()`, custom arrows are kept but lose `role`, `tabindex`, `disabled` and all `aria-*` attributes that Splide manages (`ALL_ATTRIBUTES`). Generated arrows are removed.

React notes:

- The wrapper requirement holds in React too. Tested: custom buttons without `.splide__arrows` threw `TypeError: Cannot read properties of undefined (reading 'style')` from `componentDidMount`.
- Don't pass `disabled` or `aria-label` props to custom arrows. Splide manages both. React writes a prop only when its value changes, so a changing prop would fight Splide's value. Change the text through `options.i18n` (`prev`, `next`, `first`, `last`).
- With nested carousels, put the outer carousel's custom arrows or placeholder **before** its `<SplideTrack>`. The inner carousel mounts first, Splide checks only the first match, and a placeholder after the track is ignored (tested; see the element reference above).

---

## Pagination

### Generated markup

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

- Extra classes: the `classes` option (`pagination`, `page`). Keep the `splide__` classes.
- By default the pagination is appended to the track's parent. Its direction class and `aria-orientation="vertical"` follow `paginationDirection`, falling back to `direction`.
- Number of dots: with `focus` set or `isNavigation`, `Controller.getEnd() + 1` (one per slide index up to the end index). Otherwise `ceil( length / perPage )` (one per page). `omitEnd` (4.1+) drops dots that would only change the active slide without moving.
- `aria-controls` lists every slide ID on that page, space-separated.
- `aria-label` uses i18n `pageX` ("Go to page %s") when there's no `focus` and `perPage > 1`, and `slideX` ("Go to slide %s") otherwise.
- Roving tabindex: only the active button lacks `tabindex="-1"`.
- Docs (Accessibility page) say inactive dots get `aria-selected="false"`. The source **removes** `aria-selected` from inactive dots (`removeAttribute`) and sets `"true"` only on the active one.

### Placeholder

Render an empty `<ul className="splide__pagination" />` where the dots should go (the placeholder example above). Splide fills it with `<li>`/`<button>` items and keeps your extra classes on it. With `pagination: false` the placeholder gets `display: none`. On destroy, only its children are removed. Never put the placeholder inside `<SplideTrack>`.

Core CSS hides `.splide__pagination` while the root lacks `is-overflow`, which happens when all slides fit (v4.1+).

---

## Progress bar and autoplay toggle

Both are plain markup you render as children of `<Splide hasTrack={ false }>` (see the placeholder example above and the complete example below):

```tsx
<div className="splide__progress">
  <div className="splide__progress__bar" />
</div>

<button className="splide__toggle" type="button">
  <span className="splide__toggle__play">Play</span>
  <span className="splide__toggle__pause">Pause</span>
</button>
```

The docs' React example writes `class=` instead of `className=` on the toggle and its spans. That still renders, but React warns about `class` in development. Use `className`.

Progress bar:

- Only `.splide__progress__bar` is looked up. `.splide__progress` is a plain wrapper for your styling.
- During autoplay, Splide sets the bar's inline `width` from `0%` to `100%` on each animation frame (`autoplay:playing` gives the same rate, 0 to 1).
- By default the progress rewinds when autoplay is interrupted. Set `resetProgress: false` to keep the elapsed time.
- The bar only moves with `autoplay`. For a "position in carousel" bar, see the Carousel Progress tutorial (`references/tutorials/carousel-progress.md`).

Autoplay toggle: if you enable autoplay, also provide a play/pause button. The accessibility page and WCAG 2.2.2 require one. The icons can be any inline elements, such as `<img>` or `<svg>`:

```tsx
<button className="splide__toggle" type="button">
  <svg className="splide__toggle__play" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
    <path d="m22 12-20 11v-22l10 5.5z" />
  </svg>
  <svg className="splide__toggle__pause" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
    <path d="m2 1v22h7v-22zm13 0v22h7v-22z" />
  </svg>
</button>
```

Which icon shows. Docs (Autoplay page) say "while autoplay is playing, `splide__toggle__play` will be visible, and while it's paused, `splide__toggle__pause` will be". **The source and CSS do the opposite**, which is the usual control convention:

- While playing, the toggle has `is-active`, so `.splide__toggle.is-active .splide__toggle__pause { display: inline }` shows the **pause** icon. `aria-label` is "Pause autoplay".
- While paused or stopped, `is-active` is absent, so `.splide__toggle__pause { display: none }` hides the pause icon and the play icon shows. `aria-label` is "Start autoplay".

Tested in React (the placeholder example above): with `autoplay: true` the toggle read `class="splide__toggle is-active"` and `aria-label="Pause autoplay"`, and after a click `class="splide__toggle"` and `aria-label="Start autoplay"`.

More source facts:

- Splide sets `aria-controls` (track ID) on the toggle and binds `click`: play if stopped, otherwise `pause()` (stop).
- The toggle only works when `autoplay` is `true` or `'pause'`. With `autoplay: false`, Splide ignores it.
- Hover or focus pauses (`pauseOnHover` / `pauseOnFocus`) don't change the toggle state. Only a stop by the button or by `Autoplay.pause()` does.
- Splide doesn't add `type="button"`. Include it.

---

## Data attributes

Every attribute Splide core reads (`DATA_ATTRIBUTE = 'data-splide'` in `constants/project.ts`, plus the component constants). In React they are ordinary JSX attributes: on `<Splide>` for the root, on `<SplideSlide>` for slides, on `<img>` for images.

| Attribute | On | Value | Read by | Notes |
|---|---|---|---|---|
| `data-splide` | root | JSON options, for example `'{"perPage":3}'` | Constructor | Wins over the `options` prop at construction, but a later change of the `options` prop re-applies every key in it. Invalid or empty JSON throws `[splide] Invalid JSON`. Read once. See `references/getting-started.md`. |
| `data-splide-interval` | slide | Milliseconds, for example `"10000"` | Autoplay | On every `move`, the interval becomes the destination slide's value, or `options.interval` if it has none (or `0`/non-numeric). The first slide's value isn't applied to the very first interval after mount, because that one uses `interval`. It applies whenever the carousel moves to that slide. |
| `data-splide-lazy` | `<img>` inside a slide | Image URL | LazyLoad | Copied to `src` when the slide comes near (`lazyLoad: 'nearby'`) or in order (`'sequential'`), then the attribute is removed |
| `data-splide-lazy-srcset` | `<img>` inside a slide | `srcset` string | LazyLoad | Copied to `srcset` with the same timing, then removed |

`data-splide` on `<Splide>` works, but prefer the `options` prop: precedence and update behaviour are in `references/getting-started.md`. A number passed to `data-splide-interval` is written as a string, and `undefined` omits the attribute (tested above).

LazyLoad details (from source): the selector is `[data-splide-lazy], [data-splide-lazy-srcset]`, matched anywhere inside a slide (not only direct children). An image **without** a `src` is hidden (`display: none`) until it loads. An image with a placeholder `src` stays visible. While it loads, the slide gets `is-loading` and a `<span class="splide__spinner">` is added to the image's parent. A spinner you provide in that parent is reused. On `load`, the spinner is removed and the image shown. On `error`, the spinner stays.

Attributes read by official extensions (not core; see `references/extensions/*.md`):

| Attribute | On | Extension |
|---|---|---|
| `data-splide-youtube`, `data-splide-vimeo`, `data-splide-html-video` | slide | Video |
| `data-splide-hash` | slide | URL Hash Navigation |

Splide also reads these existing attributes:

| Attribute | On | Effect |
|---|---|---|
| `aria-label`, `aria-labelledby` | root | Become the defaults of the `label` / `labelledby` options |
| `role` | root | If present, Splide doesn't set `role` |
| `id` | root, track, list | Kept if present (slide IDs are always overwritten) |
| `aria-label` | slide | Kept instead of the generated "1 of 5" label (not for `isNavigation` slides) |
| `style` | slide | Saved and restored on `destroy()` |

React notes for the last table: an `aria-label`, `aria-labelledby` or `role` prop on `<Splide>` works in production, but is lost in development with StrictMode (use `options.label`, `options.labelledby`, `options.role`). A slide's own `aria-label` (a prop on `<SplideSlide>`) survives, because Splide saves and restores it.

---

## IDs Splide assigns

| Element | ID | Overwrites an existing ID? |
|---|---|---|
| Root | its own `id`, else `splide01`, `splide02`, ... (a per-build counter, 2-digit padded) | No |
| Track | `{rootId}-track` | No |
| List | `{rootId}-list` | No |
| Slide | `{rootId}-slide01`, `-slide02`, ... | **Yes** |
| Clone | `{rootId}-clone01`, `-clone02`, ... | Yes (clones are new elements) |

Give each root a stable `id` if anything references the generated IDs, for example CSS or `aria-controls` in your own markup.

React notes:

- Pass `id` to `<Splide>` for a stable root ID. Tested: `<Splide id="featured">` gave `featured-track`, `featured-list` and `featured-slide01`, and didn't use up a generated number (the next carousel without an `id` got `splide02`).
- Generated IDs follow mount order, and under StrictMode the remount reuses the ID already on the element (tested: `splide01` to `splide03` in both modes).
- React Splide's bundled core has its own counter. Another copy of core on the same page (the vanilla build, or a runtime import of `@splidejs/splide`) counts from `splide01` too, so give roots explicit IDs if both are used.
- `useId()` is the easy way to make heading IDs for `options.labelledby` in a reusable component (`references/getting-started.md`).

---

## Attributes Splide adds to each element

`setAttribute( el, name, '' )` in the source **removes** the attribute, so "removed" below means Splide deletes the attribute.

### Root

| Attribute / class | Value |
|---|---|
| `id` | See IDs |
| `role` | `options.role` (default `region`), only if the root isn't a `<section>` and has no `role` |
| `aria-roledescription` | i18n `carousel` (`carousel`) |
| `aria-label` | `options.label`. Removed if empty. |
| `aria-labelledby` | `options.labelledby`. Removed if empty. |
| classes | `splide--{type}`, `splide--{direction}`, `splide--draggable` (if `drag`), `splide--nav` (if `isNavigation`), `is-active`, `is-initialized`. Also `is-overflow` and `is-focus-in` when they apply. |
| `style` | `max-width` from the `width` option |

Splide never adds `tabindex` to the root in v4. If you need it (for example with `keyboard: 'focused'`, which behaves like `true`), a `tabIndex={ 0 }` prop works in production. In development with StrictMode it is removed by `destroy()` during the remount (tested), so don't rely on it while testing there.

### Track

| Attribute / class | Value |
|---|---|
| `id` | `{rootId}-track` (if none) |
| `aria-live` | `polite`, or `off` while autoplay is playing. Only with `live: true` and not `isNavigation`. |
| `aria-atomic` | `true` (live region enabled, ^4.0.3) |
| `aria-busy` | `true` right after `moved`/`scrolled`, then `false` about 90 ms later (live region enabled, ^4.0.3) |
| classes | `splide__track--{type}`, `splide__track--{direction}`, `splide__track--draggable`, `splide__track--nav` |
| `style` | `padding-left`/`padding-right` (or `padding-top`/`padding-bottom` for TTB) from `padding`, defaulting to `0px`. `height` for TTB. |

### List

| Attribute | Value |
|---|---|
| `id` | `{rootId}-list` (if none) |
| `role` | `presentation` |
| `aria-orientation` | `vertical` when `isNavigation` and `direction: 'ttb'` |
| `style` | `transform: translateX(...)` / `translateY(...)` |

### Slides (normal carousel)

| Attribute | Value |
|---|---|
| `id` | `{rootId}-slideNN` |
| `role` | `tabpanel` if `pagination` is enabled, `group` otherwise |
| `aria-roledescription` | i18n `slide` (`slide`) |
| `aria-label` | The slide's own `aria-label` if present, else i18n `slideLabel` (`'%s of %s'`), for example `1 of 5` |
| `aria-hidden` | `true` when the slide isn't fully visible **and** (it isn't active, or it's a clone). Removed otherwise. Not updated while moving or scrolling. |
| `tabindex` on focusable descendants | `-1` inside hidden slides, removed inside visible ones. Descendants are matched by `focusableNodes`, default `'a, button, textarea, input, select, iframe'`. |
| `tabindex` on the slide | Only if `slideFocus`: `0` when visible, `-1` when hidden |
| classes | `is-active`, `is-visible`, `is-prev`, `is-next`, `is-loading` (see `references/themes.md`) |
| `style` | `width` (or `height` for TTB), `margin-right`/`-left`/`-bottom` for `gap`, `height` (on the container if present) |

Clones copy the original slide's attributes, get `splide__slide--clone` and their own ID, and take part in `aria-hidden` and `is-active` updates (`cloneStatus`).

### Slides with `isNavigation: true`

| Attribute | Value |
|---|---|
| `aria-label` | i18n `slideX` ("Go to slide %s") |
| `aria-controls` | Space-separated IDs of the matching slides in the synced carousels |
| `role` | `button` when `slideFocus` (default `true` for navigation). **Removed** when `slideFocus: false`. Docs say it falls back to `group`. |
| `aria-roledescription` | Removed when `slideFocus` is on (interactive). Kept (`slide`) when `slideFocus: false`. |
| `aria-current` | `true` on the active slide (not in the docs tables) |
| `aria-hidden` | As above |
| `tabindex` | `0` visible / `-1` hidden (from `slideFocus`) |

Enter or Space on a navigation slide works like a click: it calls `go( slideIndex )`, and the synced carousel follows.

### Arrows

| Attribute | Value |
|---|---|
| `aria-controls` | Track ID |
| `aria-label` | "Previous slide" / "Go to last slide" (prev). "Next slide" / "Go to first slide" (next). |
| `disabled` | Set when there's no slide to go to |
| `type` | `button` (generated arrows only) |

### Pagination

| Element | Attributes |
|---|---|
| `<ul class="splide__pagination">` | `role="tablist"`, `aria-label` = i18n `select` ("Select a slide to show"), `aria-orientation="vertical"` for TTB pagination |
| `<li>` | `role="presentation"` |
| `<button class="splide__pagination__page">` | `type="button"`, `role="tab"`, `aria-controls` (slide IDs), `aria-label` ("Go to slide %s" / "Go to page %s"), `tabindex="-1"` except the active one, `aria-selected="true"` + `is-active` on the active one |

The docs' accessibility table writes the pagination label as "Select slide to show". The i18n default in source is `'Select a slide to show'`.

### Toggle and progress bar

| Element | Attributes |
|---|---|
| `.splide__toggle` | `aria-controls` = track ID, `aria-label` = "Pause autoplay" while playing / "Start autoplay" while stopped, class `is-active` while playing |
| `.splide__progress__bar` | inline `width: N%` |


---

## Inline styles Splide writes

| Element | Property | From |
|---|---|---|
| root | `max-width` | `width` |
| track | `padding-left` / `padding-right` (`-top` / `-bottom` for TTB) | `padding` |
| track | `height: calc(<height> - <paddings>)` | TTB: `height` or `heightRatio` |
| list | `transform` | Position |
| slide | `width` | `fixedWidth`, or computed from `perPage` and `gap` (not set with `autoWidth`) |
| slide | `margin-right` (`margin-left` RTL, `margin-bottom` TTB) | `gap` |
| slide or container | `height` | `fixedHeight`, `height`, `heightRatio`, or (TTB) computed from `perPage` |
| container or slide | `background` | `cover` |
| `<img>` | `display: none` | `cover`, or lazy image without `src` |
| arrows wrapper, pagination placeholder | `display: none` | `arrows: false` / `pagination: false` |
| progress bar | `width` | Autoplay progress |

CSS can't override these without `!important`. Change the options instead.

A `style` prop on `<SplideSlide>` merges with these (React sets only the properties you pass), and survives destroy, because Splide saves and restores the slide's original `style`. A `style` prop on `<Splide>` or `<SplideTrack>` does not survive: `destroy()` removes the `style` attribute from the root, track and list. Tested in development with StrictMode: both were gone after the remount. Style the root and track with classes.

---

## What unmounting removes

`componentWillUnmount` calls `splide.destroy()` (`completely = true`). React then removes the elements anyway, so this matters in two cases: the StrictMode remount in development (same DOM node, new instance), and breakpoints with `destroy`, which destroy a component that stays mounted (`destroy: true` calls `destroy( false )`, `destroy: 'completely'` calls `destroy( true )`). What `destroy()` does (core source, verified in vanilla with jsdom):

- Root: removes the modifier classes and `is-active`. **Keeps `is-initialized`**, so core CSS stacks the slides as blocks. `destroy()` also removes `role`, `tabindex`, `disabled`, all managed `aria-*` attributes (including **your own** `aria-label` / `aria-labelledby` / `role`) and `style`. `destroy( false )` removes only `style` and `aria-roledescription`.
- Track and list: modifier classes, managed attributes and `style` removed. IDs stay.
- Slides: status classes and managed attributes removed. The original `style` and `aria-label` are restored. Generated IDs stay.
- Generated arrows, generated pagination and clones are removed. A pagination placeholder is emptied. Custom arrows stay, minus managed attributes.
- All Splide event handlers are removed.

Lost during the StrictMode remount, because React doesn't re-apply unchanged props (all tested): `aria-label`, `aria-labelledby`, `role` and `tabIndex` props on `<Splide>` (`role` comes back as `region` for a `<div>`), and `style` props on `<Splide>` and `<SplideTrack>`. Kept: `id`, `className`, `data-*` attributes, everything set through `options`, and `aria-label` / `style` on `<SplideSlide>`.

With `destroy: true` in a breakpoint, the root keeps `is-initialized`, its label and its role (verified in vanilla: only `aria-roledescription` and `style` go). With `destroy: 'completely'`, the label and role go too and nothing remounts it, so the slides stay as an unlabelled stack until the component remounts (for example with a new `key`).

---

## When React and Splide both write the DOM

Splide adds classes, attributes and inline styles to elements React rendered. React only writes an attribute when its prop value changes, so the two usually coexist. These are the cases where they collide:

| You change | React does | Result | Do instead |
|---|---|---|---|
| `className` on `<Splide>` | Rewrites the whole `class` attribute | Splide's classes are gone, including `is-initialized`, so core CSS hides the carousel (tested: root left as `splide dark`, `visibility: hidden`) | Keep it constant. Put changing classes on a wrapper element. |
| `className` on `<SplideSlide>` | Rewrites the slide's `class` | `is-active`, `is-visible` and friends are gone until Splide updates that slide again (tested) | Put changing classes on an element inside the slide |
| `style` on `<Splide>` / `<SplideTrack>` | Sets only the changed properties | Works, but the whole attribute is lost on destroy (above) | Use classes |
| `aria-label` / `disabled` on custom arrows | Writes the new value | Overwrites Splide's label or disabled state | Use `options.i18n`. Let Splide manage `disabled`. |
| `tag` or `hasTrack` | Replaces root or track elements | The instance points at detached elements | Change `key` to remount |

The working pattern for changing classes (tested: after toggling the theme and picking a card, the root kept all its Splide classes and the first slide kept `is-active`):

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const items = [ 'Red', 'Green', 'Blue' ];

export default function StableClassNames() {
  const [ dark, setDark ] = useState( false );
  const [ picked, setPicked ] = useState<string>();

  return (
    // Changing classes go on a wrapper and on the slide content, never on <Splide> or <SplideSlide>.
    <div className={ dark ? 'theme-dark' : 'theme-light' }>
      <Splide className="color-carousel" options={ { label: 'Colors', perPage: 2 } }>
        { items.map( item => (
          <SplideSlide key={ item }>
            <button
              type="button"
              className={ item === picked ? 'card is-picked' : 'card' }
              onClick={ () => setPicked( item ) }
            >
              { item }
            </button>
          </SplideSlide>
        ) ) }
      </Splide>

      <button type="button" className="theme-toggle" onClick={ () => setDark( value => ! value ) }>
        Toggle theme
      </button>
    </div>
  );
}
```

---

## Complete example with every optional element

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type Product = { id: string; name: string; image: string; interval?: number };

const options: Options = {
  labelledby : 'featured-heading',
  perPage    : 2,
  gap        : '1rem',
  autoplay   : true,
  interval   : 5000,
  lazyLoad   : 'nearby',
  cover      : true,
  heightRatio: 0.4,
  breakpoints: {
    640: { perPage: 1 },
  },
};

export function FeaturedProducts( { products }: { products: Product[] } ) {
  return (
    <Splide tag="section" id="featured" hasTrack={ false } options={ options }>
      <h2 id="featured-heading">Featured products</h2>

      { /* relative wrapper so arrows center on the slides, not on the heading */ }
      <div className="carousel-stage">
        <div className="splide__arrows">
          <button className="splide__arrow splide__arrow--prev" type="button">&lsaquo;</button>
          <button className="splide__arrow splide__arrow--next" type="button">&rsaquo;</button>
        </div>

        <SplideTrack>
          { products.map( product => (
            <SplideSlide key={ product.id } data-splide-interval={ product.interval }>
              <div className="splide__slide__container">
                <img data-splide-lazy={ product.image } alt={ product.name } />
              </div>
              <p>{ product.name }</p>
            </SplideSlide>
          ) ) }
        </SplideTrack>
      </div>

      <ul className="splide__pagination" />

      <div className="splide__progress">
        <div className="splide__progress__bar" />
      </div>

      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}

const svg = ( color: string ) => 'data:image/svg+xml,' + encodeURIComponent(
  `<svg xmlns="http://www.w3.org/2000/svg" width="800" height="400"><rect width="800" height="400" fill="${ color }"/></svg>`,
);

export default function Example() {
  return (
    <>
      <style>{ '.carousel-stage { position: relative; }' }</style>
      <FeaturedProducts
        products={ [
          { id: 'p1', name: 'Product 1', image: svg( '#c33' ), interval: 3000 },
          { id: 'p2', name: 'Product 2', image: svg( '#3c3' ) },
          { id: 'p3', name: 'Product 3', image: svg( '#33c' ) },
        ] }
      />
    </>
  );
}
```

The `Example` wrapper only supplies test data and the `.carousel-stage { position: relative; }` rule. Put that rule in your stylesheet.

Tested: the root kept `aria-labelledby="featured-heading"` and no `role` (a `<section>`), Splide used the two custom arrows (labelled `Previous slide` / `Next slide`) and centered them on the track, filled the pagination placeholder with 2 dots without generating a second list, labelled the toggle `Pause autoplay`, moved the progress bar, and put the `cover` background on each container. Because the pagination placeholder sits after the stage, its dots appear below the slides. With a theme, the pagination is still `position: absolute; bottom: .5em` against the root. Add `.splide__pagination { position: static; }` to keep it in normal flow below the slides (`references/themes.md`).
