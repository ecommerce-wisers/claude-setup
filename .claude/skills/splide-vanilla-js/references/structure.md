# Structure

Source: https://splidejs.com/guides/structure/
Markup for the optional elements also comes from https://splidejs.com/guides/arrows/, https://splidejs.com/guides/pagination/, https://splidejs.com/guides/autoplay/ and the ARIA tables on https://splidejs.com/guides/accessibility/. Everything was checked against `components/Elements`, `Slides/Slide`, `Arrows`, `Pagination`, `Autoplay`, `Live`, `LazyLoad`, `Layout`, `Clones`, `Cover`, `Sync`, `constants/classes.ts` and `constants/attributes.ts` (v4.1.3 source). The generated output was verified by mounting the v4.1.4 `splide.min.js` in jsdom.

## Contents

- [Basic structure](#basic-structure)
- [Roles](#roles)
- [Element reference](#element-reference)
- [Restrictions](#restrictions)
- [What Splide inserts itself](#what-splide-inserts-itself)
- [Where your own elements can go](#where-your-own-elements-can-go)
- [Using div instead of ul/li](#using-div-instead-of-ulli)
- [Slide container](#slide-container)
- [Arrows](#arrows)
- [Pagination](#pagination)
- [Progress bar](#progress-bar)
- [Autoplay toggle button](#autoplay-toggle-button)
- [Data attributes](#data-attributes)
- [IDs Splide assigns](#ids-splide-assigns)
- [Attributes Splide adds to each element](#attributes-splide-adds-to-each-element)
- [Inline styles Splide writes](#inline-styles-splide-writes)
- [What `destroy()` removes](#what-destroy-removes)
- [Complete example with every optional element](#complete-example-with-every-optional-element)

---

## Basic structure

Splide needs four elements: `root`, `track`, `list` and `slide`.

```html
<section class="splide" aria-label="Basic Structure Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</section>
```

Give the root a concise label. If the carousel has a visible heading, point `aria-labelledby` at its ID instead:

```html
<section class="splide" aria-labelledby="carousel-heading">
  <h2 id="carousel-heading">Basic Structure Example</h2>

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</section>
```

`<div>` can replace `<section>`, `<ul>` and `<li>`.

The label can also come from options: `label` sets `aria-label` and `labelledby` sets `aria-labelledby`. Both default to the root's existing attributes.

About the word "carousel" in the label: W3C suggests leaving it out, because Splide sets `aria-roledescription="carousel"` and some screen readers announce "label" + "description". That behavior isn't required by the spec, though, and some screen readers ignore the description. The docs' author thinks including "carousel" is acceptable. Some users may hear it twice, which is a small cost compared with hearing nothing.

---

## Roles

### Landmark role

Many assistive technologies can jump between landmarks (important page areas such as `banner` and `navigation`). A landmark works as an invisible but very useful table of contents. A `<section>` with an accessible name has an implicit `region` landmark role, so it's a better root than a `<div>`.

If the root is a `<div>`, Splide sets `role="region"` itself (the `role` option, default `'region'`), which gives the same result.

Source rule (`Elements.init()`): `role` is set only when the root has **no** `role` attribute, is **not** a `<section>`, and the `role` option is truthy. Splide never changes an existing role, because that would remove the region from the accessibility tree.

If you know how roles affect accessibility, pick the tag and role that fit your page architecture. Otherwise, use `<div>` or `<section>`.

### Group role

For a carousel that isn't significant or isn't directly related to the main content, set `role="group"` on the root:

```html
<div class="splide" role="group" aria-label="Unimportant Content">
  <div class="splide__track">...</div>
</div>
```

The same through options: `new Splide( '.splide', { role: 'group' } )` (takes effect only on a non-`<section>` root without its own `role`).

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

---

## Restrictions

v4 markup is more flexible than in older versions. Keep only two rules:

- Don't insert any elements inside the `track` and `list` elements.
- Don't break the `track > list > slide` relation. The list must be a child of the track, and slides must be children of the list.

The track does **not** have to be a direct child of the root, so this is valid:

```html
<section class="splide" aria-label="Restriction Example">
  <div>
    <div>
      <div>
        <div class="splide__track">
          <ul class="splide__list">
            <li class="splide__slide">...</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</section>
```

This is **invalid**, because the relation becomes `track > div > list > slide`:

```html
<section class="splide" aria-label="Incorrect Structure Example">
  <div class="splide__track">
    <div>
      <ul class="splide__list">
        <li class="splide__slide">...</li>
      </ul>
    </div>
  </div>
</section>
```

Mounting that throws `[splide] A track/list element is missing.` (verified).

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

---

## Where your own elements can go

| Location | Allowed? | Notes |
|---|---|---|
| Inside the root, before or after the track (headings, captions, counters, buttons) | Yes | Arrows and pagination are positioned against the nearest positioned ancestor (the root by default, which has `position: relative`). Wrap the track in a `position: relative` element if arrows should center on the slides only. |
| Wrapper elements between root and track | Yes | Generated arrows and pagination are inserted into the track's direct parent |
| Inside a slide (any content: images, text, links, forms, even another carousel) | Yes | Focusable elements in hidden slides get `tabindex="-1"` (`focusableNodes`) |
| Directly inside the track, beside the list | **No** | Breaks the structure rules |
| Directly inside the list, beside the slides | **No** | Breaks layout and indexing |
| Between track and list (wrapping the list) | **No** | Throws `[splide] A track/list element is missing.` |
| Arrows/pagination placeholders inside the track or list | **No** | Docs: "do not insert the placeholder in the track element" |
| Controls outside the root | Not picked up | Splide doesn't find them. Wire them yourself: `btn.addEventListener( 'click', () => splide.go( '>' ) )`. |

---

## Using div instead of ul/li

`<div>` works for the root, list and slides. Lookups use classes, not tag names.

```html
<div class="splide" aria-label="Div Example">
  <div class="splide__track">
    <div class="splide__list">
      <div class="splide__slide">Slide 01</div>
      <div class="splide__slide">Slide 02</div>
    </div>
  </div>
</div>
```

The list gets `role="presentation"` either way, so `<ul>` list semantics are removed anyway. A `<div>` root gets `role="region"` unless you set a role. Generated pagination is always a `<ul>` with `<li>`/`<button>` (the pagination placeholder should also be a `<ul>`).

---

## Slide container

A container sets a height on part of a slide while the slide's own height adapts to its content. It's especially useful with `cover` mode when slides also contain headings or descriptions.

In the docs example, images are `9rem` tall and each slide is as tall as the tallest content among all slides.

Wrap the images in a `<div class="splide__slide__container">`. It must be a **direct child** of the `splide__slide` element:

```html
<section class="splide" aria-label="Slide Container Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <div class="splide__slide__container">
          <img src="..." alt="...">
        </div>
        Lorem Ipsum Dolor Sit Amet
      </li>
      <li class="splide__slide">
        <div class="splide__slide__container">
          <img src="..." alt="...">
        </div>
        Lorem Ipsum Dolor Sit Amet
      </li>
    </ul>
  </div>
</section>
```

```js
new Splide( '.splide', { cover: true, heightRatio: 0.5 } ).mount();
// or { cover: true, fixedHeight: '9rem' }
```

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

By default the arrows go right before the track. Put an empty wrapper where you want them:

```html
<section class="splide" aria-label="Placeholder Example">
  <!-- default location -->

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
    </ul>
  </div>

  <div class="splide__arrows"></div>
</section>
```

To center arrows vertically on the slides while other controls are in the root, put the placeholder inside your own relative element:

```html
<section class="splide" aria-label="Placeholder Example">
  <div style="position: relative">
    <div class="splide__arrows"></div>

    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">...</li>
      </ul>
    </div>
  </div>

  <div>
    <!-- Some other controls -->
  </div>
</section>
```

Never put the placeholder inside the track.

### Custom arrows

For icon fonts, images or text, write the arrows yourself:

```html
<section class="splide" aria-label="Custom Arrows Example">
  <div class="splide__arrows">
    <button class="splide__arrow splide__arrow--prev" type="button">
      Prev
    </button>
    <button class="splide__arrow splide__arrow--next" type="button">
      Next
    </button>
  </div>

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>
</section>
```

`.splide__arrows` can go anywhere that keeps the structure rules. Each arrow must be a `<button>`, so it can receive focus and a `disabled` attribute.

Source facts for custom arrows (verified):

- Splide uses them only when **both** `.splide__arrow--prev` and `.splide__arrow--next` exist. Otherwise it generates arrows (inside `.splide__arrows` if present).
- **The `.splide__arrows` wrapper is required.** With custom prev/next buttons and no wrapper, `mount()` throws `TypeError: Cannot read properties of undefined (reading 'style')`, because Splide sets `display` and a modifier class on the wrapper. The docs example includes the wrapper but doesn't say it's required.
- Splide sets `aria-controls` (track ID), `aria-label` (from i18n, **overwriting** your own `aria-label`) and `disabled`. Change the text through the `i18n` option (`prev`, `next`, `first`, `last`).
- Splide doesn't add `type="button"` to custom arrows. Add it yourself, so they don't submit a surrounding `<form>`.
- `.splide__arrow` isn't needed to find the arrows, but keep it. The themes style it, and drag doesn't start on elements matching `.splide__arrow` or `.splide__pagination__page`.
- `arrows: false` with custom arrows: the wrapper gets `display: none`.
- On `destroy()`, custom arrows are kept but lose `role`, `tabindex`, `disabled` and all `aria-*` attributes that Splide manages (`ALL_ATTRIBUTES`). Generated arrows are removed.

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

Splide fills the placeholder with `<li>`/`<button>` items and keeps your extra classes on it. With `pagination: false` the placeholder gets `display: none`. On destroy, only its children are removed. Never put the placeholder inside the track.

Core CSS hides `.splide__pagination` while the root lacks `is-overflow`, which happens when all slides fit (v4.1+).

---

## Progress bar

```html
<div class="splide" aria-label="Progress Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>

  <div class="splide__progress">
    <div class="splide__progress__bar">
    </div>
  </div>
</div>
```

With `splide-core.min.css`, style the bar yourself to make it visible:

```css
.splide__progress__bar {
  height: 3px;
  background: #ccc;
}
```

- Only `.splide__progress__bar` is looked up. `.splide__progress` is a plain wrapper for your styling.
- During autoplay, Splide sets the bar's inline `width` from `0%` to `100%` on each animation frame (`autoplay:playing` gives the same rate, 0 to 1).
- By default the progress rewinds when autoplay is interrupted. Set `resetProgress: false` to keep the elapsed time.
- The bar only moves with `autoplay`. For a "position in carousel" bar, see the Carousel Progress tutorial (`tutorials/`).

---

## Autoplay toggle button

If you enable autoplay, also provide a play/pause button. The accessibility page and WCAG 2.2.2 require one.

```html
<section class="splide" aria-label="Autoplay Example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
      <li class="splide__slide">Slide 03</li>
    </ul>
  </div>

  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>
```

The docs example wraps the track in a `<div class="splide__slider">`. That wrapper has no function in v4 (see `version4.md`), so it's omitted here.

The icons can be any inline elements, such as `<img>` or `<svg>`:

```html
<button class="splide__toggle" type="button">
  <svg
    class="splide__toggle__play"
    viewBox="0 0 24 24"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path d="m22 12-20 11v-22l10 5.5z"/>
  </svg>
  <svg
    class="splide__toggle__pause"
    viewBox="0 0 24 24"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path d="m2 1v22h7v-22zm13 0v22h7v-22z"/>
  </svg>
</button>
```

Which icon shows. Docs (Autoplay page) say "while autoplay is playing, `splide__toggle__play` will be visible, and while it's paused, `splide__toggle__pause` will be". **The source and CSS do the opposite**, which is the usual control convention:

- While playing, the toggle has `is-active`, so `.splide__toggle.is-active .splide__toggle__pause { display: inline }` shows the **pause** icon. `aria-label` is "Pause autoplay".
- While paused or stopped, `is-active` is absent, so `.splide__toggle__pause { display: none }` hides the pause icon and the play icon shows. `aria-label` is "Start autoplay".

Verified with jsdom: after `mount()` with `autoplay: true` the toggle reads `class="splide__toggle is-active" aria-label="Pause autoplay"`, and after `Autoplay.pause()` it reads `class="splide__toggle" aria-label="Start autoplay"`.

More source facts:

- Splide sets `aria-controls` (track ID) on the toggle and binds `click`: play if stopped, otherwise `pause()` (stop).
- The toggle only works when `autoplay` is `true` or `'pause'`. With `autoplay: false`, Splide ignores it.
- Hover or focus pauses (`pauseOnHover` / `pauseOnFocus`) don't change the toggle state. Only a stop by the button or by `Autoplay.pause()` does.
- Splide doesn't add `type="button"`. Include it.

---

## Data attributes

Every attribute Splide core reads (`DATA_ATTRIBUTE = 'data-splide'` in `constants/project.ts`, plus the component constants):

| Attribute | On | Value | Read by | Notes |
|---|---|---|---|---|
| `data-splide` | root | JSON options, for example `'{"perPage":3}'` | Constructor | Overrides constructor options and `Splide.defaults`. Invalid or empty JSON throws `[splide] Invalid JSON`. Read once. See `getting-started.md`. |
| `data-splide-interval` | slide | Milliseconds, for example `"10000"` | Autoplay | On every `move`, the interval becomes the destination slide's value, or `options.interval` if it has none (or `0`/non-numeric). The first slide's value isn't applied to the very first interval after mount, because that one uses `interval`. It applies whenever the carousel moves to that slide. |
| `data-splide-lazy` | `<img>` inside a slide | Image URL | LazyLoad | Copied to `src` when the slide comes near (`lazyLoad: 'nearby'`) or in order (`'sequential'`), then the attribute is removed |
| `data-splide-lazy-srcset` | `<img>` inside a slide | `srcset` string | LazyLoad | Copied to `srcset` with the same timing, then removed |

```html
<li class="splide__slide" data-splide-interval="1000"></li>
<li class="splide__slide" data-splide-interval="10000"></li>
```

```html
<li class="splide__slide">
  <img data-splide-lazy="path/to/image.jpg" data-splide-lazy-srcset="path/to/image-800.jpg 800w, path/to/image-1600.jpg 1600w" alt="...">
</li>
```

LazyLoad details (from source): the selector is `[data-splide-lazy], [data-splide-lazy-srcset]`, matched anywhere inside a slide (not only direct children). An image **without** a `src` is hidden (`display: none`) until it loads. An image with a placeholder `src` stays visible. While it loads, the slide gets `is-loading` and a `<span class="splide__spinner">` is added to the image's parent. A spinner you provide in that parent is reused. On `load`, the spinner is removed and the image shown. On `error`, the spinner stays.

Attributes read by official extensions (not core; see `extensions/*.md`):

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

Splide never adds `tabindex` to the root in v4. Add `tabindex="0"` yourself if you need it (for example with `keyboard: 'focused'`, which behaves like `true`).

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
| classes | `is-active`, `is-visible`, `is-prev`, `is-next`, `is-loading` (see `themes.md`) |
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

---

## What `destroy()` removes

`splide.destroy()` (`completely = true`) and `splide.destroy( false )` (can remount through breakpoints), verified with jsdom:

- Root: removes the modifier classes and `is-active`. **Keeps `is-initialized`**, so core CSS stacks the slides as blocks. `destroy()` also removes `role`, `tabindex`, `disabled`, all managed `aria-*` attributes (including **your own** `aria-label` / `aria-labelledby` / `role`) and `style`. `destroy( false )` removes only `style` and `aria-roledescription`.
- Track and list: modifier classes, managed attributes and `style` removed. IDs stay.
- Slides: status classes and managed attributes removed. The original `style` and `aria-label` are restored. Generated IDs stay.
- Generated arrows, generated pagination and clones are removed. A pagination placeholder is emptied. Custom arrows stay, minus managed attributes.
- All Splide event handlers are removed.

Re-add the root label after a complete destroy if the markup stays on the page (for example with `destroy: 'completely'` in a breakpoint, use `role="group"` / `aria-label` again, or remount). `destroy: true` in a breakpoint calls `destroy( false )`, which keeps the label and role (verified: `aria-label` and `role="region"` stayed, `aria-roledescription` was removed).

---

## Complete example with every optional element

```html
<section id="featured" class="splide" aria-labelledby="featured-heading">
  <h2 id="featured-heading">Featured products</h2>

  <!-- relative wrapper so arrows center on the slides, not on the heading -->
  <div class="carousel-stage" style="position: relative">
    <div class="splide__arrows">
      <button class="splide__arrow splide__arrow--prev" type="button">&lsaquo;</button>
      <button class="splide__arrow splide__arrow--next" type="button">&rsaquo;</button>
    </div>

    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide" data-splide-interval="3000">
          <div class="splide__slide__container">
            <img data-splide-lazy="img/01.jpg" alt="Product 1">
          </div>
          <p>Product 1</p>
        </li>
        <li class="splide__slide">
          <div class="splide__slide__container">
            <img data-splide-lazy="img/02.jpg" alt="Product 2">
          </div>
          <p>Product 2</p>
        </li>
        <li class="splide__slide">
          <div class="splide__slide__container">
            <img data-splide-lazy="img/03.jpg" alt="Product 3">
          </div>
          <p>Product 3</p>
        </li>
      </ul>
    </div>
  </div>

  <ul class="splide__pagination"></ul>

  <div class="splide__progress">
    <div class="splide__progress__bar"></div>
  </div>

  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Play</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>

<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  new Splide( '#featured', {
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
  } ).mount();
</script>
```

Here the generated pagination fills the placeholder, so its dots appear after the stage rather than over the slides. With a theme, the pagination is still `position: absolute; bottom: .5em` against the root. Add `.splide__pagination { position: static; }` to keep it in normal flow below the slides.
