# Arrows

Source: https://splidejs.com/guides/arrows/

Also checked: https://splidejs.com/components/arrows/, https://splidejs.com/guides/events/, https://splidejs.com/guides/i18n/, https://splidejs.com/guides/accessibility/, and the v4 source (`components/Arrows/Arrows.ts`, `components/Arrows/path.ts`, `components/Elements/Elements.ts`, `components/Controller/Controller.ts`). The 4.1.4 dist is identical to 4.1.3 apart from the version banner.

## Contents

- [Setup](#setup)
- [Options](#options)
- [Generated markup](#generated-markup)
- [Changing the SVG path](#changing-the-svg-path)
- [Adding classes](#adding-classes)
- [Placeholder: choosing the position](#placeholder-choosing-the-position)
- [Custom arrows (your own markup)](#custom-arrows-your-own-markup)
- [Placement rules](#placement-rules)
- [Styling](#styling)
- [Disabled state](#disabled-state)
- [ARIA attributes and i18n labels](#aria-attributes-and-i18n-labels)
- [Events](#events)
- [Arrows component](#arrows-component)
- [Fully custom buttons outside the root](#fully-custom-buttons-outside-the-root)
- [Pitfalls](#pitfalls)

## Setup

Every example here uses the global `Splide` from the UMD build and string event names. None of them needs an ESM import.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<!-- or core only (no arrow styles at all): .../dist/css/splide-core.min.css -->
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

## Options

| Option | Type | Default | Notes |
|---|---|---|---|
| `arrows` | `boolean` | `true` | Creates the arrows. It must stay `true` when you supply arrows in HTML. With `false`, Splide generates nothing. If your HTML supplies prev/next buttons, it hides their `.splide__arrows` wrapper with inline `display: none`; an empty placeholder is left as it is. A change at runtime (through `splide.options = {...}` or `breakpoints`) remounts the arrows. |
| `arrowPath` | `string` | built-in path (below) | Replaces the `d` attribute of the generated `<path>`, for example `'m7.61 0.807-2.12...'`. The SVG is 40×40. Splide uses it only for arrows it generates. |
| `classes.arrows` | `string` | `'splide__arrows'` | Class of the wrapper that Splide creates. It is not applied to a placeholder you supply. |
| `classes.arrow` | `string` | `'splide__arrow'` | Class of each generated button. |
| `classes.prev` | `string` | `'splide__arrow--prev'` | Class of the generated previous button. |
| `classes.next` | `string` | `'splide__arrow--next'` | Class of the generated next button. |
| `i18n.prev` / `i18n.next` | `string` | `'Previous slide'` / `'Next slide'` | `aria-label` values. |
| `i18n.last` / `i18n.first` | `string` | `'Go to last slide'` / `'Go to first slide'` | `aria-label` used when the move wraps around (see [ARIA](#aria-attributes-and-i18n-labels)). |
| `direction` | `'ltr' \| 'rtl' \| 'ttb'` | `'ltr'` | Adds the modifier class `splide__arrows--{direction}` to the wrapper. Themes use it to position and rotate the arrows. |

In v4, `arrows` is boolean only. The v3 value `arrows: 'slider'` no longer exists.

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

The docs show it without the SVG details:

```html
<div class="splide__arrows splide__arrows--ltr">
  <button
    class="splide__arrow splide__arrow--prev"
    type="button"
    aria-label="Previous slide"
    aria-controls="splide01-track"
  >
    <svg>...</svg>
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

Facts from the source:

- SVG: `viewBox="0 0 40 40"`, `width="40"`, `height="40"`, `focusable="false"` (stops IE from focusing the SVG), namespace `http://www.w3.org/2000/svg`.
- Both buttons draw the same right-pointing chevron. The themes flip the previous arrow with `.splide__arrow--prev svg { transform: scaleX(-1); }`. With core CSS you have to flip it yourself.
- Without a placeholder, the wrapper is inserted right before the track element, as its previous sibling in the track's parent.
- With a placeholder (`.splide__arrows` already in the HTML), the two buttons are appended inside it.
- The wrapper modifier `splide__arrows--{ltr|rtl|ttb}` always uses the default prefix, whatever `classes.arrows` is. It is removed on destroy.
- IDs: if the root has no `id`, Splide assigns `splide` plus a two-digit counter (`splide01`, `splide02`, ...). The track gets `{rootId}-track` unless it already has an id. `aria-controls` points at the track id.

## Changing the SVG path

By default Splide draws the arrows with the SVG above. `arrowPath` replaces the path, and the path must be drawn for a 40×40 viewBox.

```js
new Splide( '.splide', {
  arrowPath: 'm15.5 0.932-4.3 4.38...',
} ).mount();
```

The path is filled by default (themes set `.splide__arrow svg { fill: #000; }`). An open, stroke-only path draws nothing unless you style the stroke. The docs site's own demos use this one:

```js
new Splide( '.splide', { arrowPath: 'm13.5 7.01 13 13m-13 13 13-13' } ).mount();
```

```css
.splide__arrow svg { fill: none; stroke: currentColor; stroke-width: 3; }
```

## Adding classes

The `classes` option adds your classes to the wrapper and buttons that Splide generates:

```js
new Splide( '.splide', {
  classes: {
    arrows: 'splide__arrows your-class-arrows',
    arrow : 'splide__arrow your-class-arrow',
    prev  : 'splide__arrow--prev your-class-prev',
    next  : 'splide__arrow--next your-class-next',
  },
} );
```

The CSS targets the `splide__`-prefixed classes, so list the default class together with yours, never yours alone.

- `classes` is deep-merged with the defaults, so you only pass the keys you change.
- `classes` affects only elements Splide creates. Splide finds HTML-provided arrows by the hard-coded default classes (`splide__arrows`, `splide__arrow--prev`, `splide__arrow--next`). Renaming a class in `classes` does not change where Splide looks.

## Placeholder: choosing the position

By default, Splide inserts the arrows before the track. To choose the position yourself, add the placeholder `<div class="splide__arrows"></div>`:

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

To center the arrows vertically on the slides when the carousel has other controls (pagination, a toggle button, captions), put the placeholder and the track inside your own relatively positioned element. The arrows' `top: 50%` is then measured against that element instead of the whole root:

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

Never put the placeholder inside the track element (structure restriction). On destroy, Splide removes the generated buttons but leaves the placeholder.

## Custom arrows (your own markup)

To use an icon font, an image or text, write the arrows yourself:

```html
<section class="splide" aria-label="Custom Arrows Example">
  <div class="splide__arrows">
    <button class="splide__arrow splide__arrow--prev">
      Prev
    </button>
    <button class="splide__arrow splide__arrow--next">
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

You can put `splide__arrows` anywhere that keeps the HTML structure valid. Each arrow must be a `<button>` so it can receive focus and take a `disabled` attribute.

Requirements and behaviour from the source:

- Both `.splide__arrow--prev` and `.splide__arrow--next` must exist. If either is missing, Splide generates a complete pair of its own (appended to your `.splide__arrows` if present), and your single button is left unmanaged.
- The buttons must sit inside an element with the class `splide__arrows`. `Arrows.ts` sets `display` on that wrapper, so if prev/next exist with no `.splide__arrows` wrapper, `mount()` throws a TypeError.
- Add `type="button"` yourself. Splide does not add it to buttons you supply (the docs example omits it), and a button without it submits any surrounding `<form>`.
- Splide overwrites `aria-label` on every update with the i18n strings, so an `aria-label` in your HTML is replaced. Change labels with the `i18n` option instead.
- `arrowPath` and `classes.*` have no effect on buttons you supply.
- On destroy, Splide removes `role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation` and `aria-roledescription` from your buttons, and leaves the elements in place.
- `arrows: false` hides your wrapper (`display: none`) instead of removing it.

## Placement rules

Splide finds `.splide__arrows`, `.splide__arrow--prev` and `.splide__arrow--next` with `root.querySelector()`. It ignores a match whose closest `.splide` is not this root, which keeps nested sliders separate.

| Location | Works? |
|---|---|
| Direct child of the root, before or after the track | Yes |
| Nested anywhere inside the root, for example in a header `div` next to a heading | Yes |
| Inside `.splide__track` or `.splide__list` | No. This breaks the structure restriction. |
| Inside a nested carousel's root | No. It belongs to that carousel. |
| Outside the root | Not detected. Use [fully custom buttons](#fully-custom-buttons-outside-the-root), or make a larger element the root. |

The track does not have to be a direct child of the root. To get arrows "outside" the slider visually, make the root the outer wrapper (heading, arrows, track, footer) and put `.splide__arrows` wherever you need it inside. Splide still handles state and ARIA for you.

## Styling

### Theme CSS (`splide.min.css` 4.1.4, verbatim rules)

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

The other themes are `dist/css/themes/splide-default.min.css` (the same as `splide.min.css`), `splide-sea-green.min.css` and `splide-skyblue.min.css`.

### Core CSS only (`splide-core.min.css`)

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
- Hide arrows when all slides fit: `.splide:not(.is-overflow) .splide__arrows { display: none; }` (v4.1+, see `overflow.md`).

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

Translate the labels, or make them match the content:

```js
new Splide( '.splide', {
  i18n: {
    prev : 'Previous product',
    next : 'Next product',
    first: 'Go to first product',
    last : 'Go to last product',
  },
} ).mount();

// Or for every carousel on the page (set before creating instances):
Splide.defaults = {
  i18n: { prev: 'Vorheriges Bild', next: 'Nächstes Bild', first: 'Zum ersten Bild', last: 'Zum letzten Bild' },
};
```

## Events

Register listeners before `mount()`, or the `*:mounted` events are missed.

| Event | Arguments | Fired |
|---|---|---|
| `arrows:mounted` | `prev: HTMLButtonElement`, `next: HTMLButtonElement` | Right after the arrows are mounted. It fires only when `arrows` is `true` and both buttons exist. It fires again after every remount, which happens on each `updated` event (a breakpoint change or `splide.options = ...`). |
| `arrows:updated` | `prev: HTMLButtonElement`, `next: HTMLButtonElement`, `prevIndex: number`, `nextIndex: number` | Whenever the arrow status (disabled, labels) is updated. `prevIndex` and `nextIndex` are the indexes the prev and next arrows would go to (`Controller.getPrev()` / `Controller.getNext()`), or `-1` when that arrow is disabled. |

- Docs say `prevIndex` is "An old index" and `nextIndex` is "A new index". Source says they are the destinations of the prev and next arrows (`getPrev()` / `getNext()`).
- On mount, the first `arrows:updated` fires before `arrows:mounted`. Another `arrows:updated` follows on the core `mounted` event.

```js
var splide = new Splide( '.splide' );

splide.on( 'arrows:mounted', function ( prev, next ) {
  prev.classList.add( 'my-prev' );
  next.classList.add( 'my-next' );
} );

// Example: hide instead of dim. The theme sets display:flex, so toggle a class, not the hidden attribute.
splide.on( 'arrows:updated', function ( prev, next, prevIndex, nextIndex ) {
  prev.classList.toggle( 'is-hidden', prevIndex < 0 );
  next.classList.toggle( 'is-hidden', nextIndex < 0 );
} );

splide.mount();
```

```css
.splide__arrow.is-hidden { visibility: hidden; }
```

## Arrows component

`splide.Components.Arrows` is available after `mount()`.

| Property | Type | Description |
|---|---|---|
| `arrows` | `{ prev?: HTMLButtonElement, next?: HTMLButtonElement }` | The previous and next buttons, whether you supplied them in HTML or Splide generated them. The object is empty when arrows are disabled. |

`splide.Components.Elements` also exposes `arrows` (the wrapper), `prev` and `next` as they were found in the HTML at mount time. Generated buttons appear only in `Arrows.arrows`.

## Fully custom buttons outside the root

Splide never finds buttons outside the root. Set `arrows: false` and drive the carousel with `splide.go()`:

| Call | Moves to |
|---|---|
| `splide.go( '<' )` | Previous page or destination. Respects `perPage`, `perMove` and `focus`. |
| `splide.go( '>' )` | Next page or destination. |
| `splide.go( '-1' )` / `splide.go( '+1' )` | Previous or next slide by index. |
| `splide.go( '>2' )` | Page 2. |

To keep the buttons accessible, give them `aria-controls` (the track id), a label, and a `disabled` state that follows the carousel. The recipe below copies the built-in logic:

```html
<div class="product-controls">
  <button type="button" class="js-prev">Previous</button>
  <button type="button" class="js-next">Next</button>
</div>

<section id="products" class="splide" aria-label="Products">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
    </ul>
  </div>
</section>

<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    var splide  = new Splide( '#products', { arrows: false, perPage: 3, gap: '1rem' } );
    var prevBtn = document.querySelector( '.js-prev' );
    var nextBtn = document.querySelector( '.js-next' );

    prevBtn.addEventListener( 'click', function () { splide.go( '<' ); } );
    nextBtn.addEventListener( 'click', function () { splide.go( '>' ); } );

    splide.mount();

    var Controller = splide.Components.Controller;
    var trackId    = splide.Components.Elements.track.id; // e.g. "products-track"

    prevBtn.setAttribute( 'aria-controls', trackId );
    nextBtn.setAttribute( 'aria-controls', trackId );

    function update() {
      var i18n      = splide.options.i18n;
      var index     = splide.index;
      var prevIndex = Controller.getPrev(); // -1 when there is nowhere to go
      var nextIndex = Controller.getNext();

      prevBtn.disabled = prevIndex < 0;
      nextBtn.disabled = nextIndex < 0;

      // Same labels as the built-in arrows. Skip these two lines if the visible text is enough.
      prevBtn.setAttribute( 'aria-label', prevIndex > -1 && index < prevIndex ? i18n.last : i18n.prev );
      nextBtn.setAttribute( 'aria-label', nextIndex > -1 && index > nextIndex ? i18n.first : i18n.next );
    }

    update();
    // Registered after mount(), so these handlers run after Controller has
    // recalculated its end index on `updated` / `refresh`.
    splide.on( 'moved scrolled refresh resized updated', update );
  } );
</script>
```

Notes:

- Built-in arrows call `Controller.go( control, true )`, where the second argument allows a "move" to the current index (used to correct the position). `splide.go()` does not pass it. For exactly the same behaviour, call `splide.Components.Controller.go( '>', true )`.
- Splide removes its own event handlers on `destroy()`. It does not remove listeners you added to external buttons, so remove those yourself.
- The root is a `region` landmark by default, and buttons outside it sit outside that landmark. Where you can, prefer a larger root containing a `.splide__arrows` placeholder ([Placement rules](#placement-rules)); Splide then manages everything.
- Splide's live region (the track's `aria-live`) announces slide changes, so you don't need extra announcements for external buttons.

## Pitfalls

- Register `arrows:mounted` before `mount()`.
- Custom prev/next buttons without a `.splide__arrows` wrapper make `mount()` throw.
- Custom arrows must be `<button>` elements. `<a>` and `<div>` can't be disabled and break focus handling.
- An `aria-label` in your HTML is overwritten. Use `i18n`.
- With core CSS, the prev arrow points right until you add `transform: scaleX(-1)` to its SVG.
- `classes` only renames generated elements. Splide always finds HTML arrows by the default classes.
- Theme arrows are positioned against the root (`top: 50%`). Pagination or other controls inside the root shift the centre; wrap the track and placeholder in a `position: relative` element.
