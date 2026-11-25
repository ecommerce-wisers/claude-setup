# Overflow

Source: https://splidejs.com/guides/overflow/

Also checked: https://splidejs.com/components/layout/, https://splidejs.com/guides/events/, and the v4 source (`components/Layout/Layout.ts`, `components/Clones/Clones.ts`, `components/Drag/Drag.ts`, `components/Autoplay/Autoplay.ts`, `constants/classes.ts`), plus the 4.1.4 dist CSS. The 4.1.4 dist JS is identical to 4.1.3 apart from the version banner.

## Contents

- [Setup](#setup)
- [Enough / not enough slides](#enough--not-enough-slides)
- [What counts as overflow](#what-counts-as-overflow)
- [The `is-overflow` class](#the-is-overflow-class)
- [The `overflow` event](#the-overflow-event)
- [`Layout.isOverflow()`](#layoutisoverflow)
- [Toggling arrows, pagination and drag](#toggling-arrows-pagination-and-drag)
- [Loop carousels (clones)](#loop-carousels-clones)
- [Centering slides](#centering-slides)
- [Full recipe (CSS + JS, no build step)](#full-recipe-css--js-no-build-step)
- [Pitfalls](#pitfalls)

## Setup

The examples use the global `Splide` and string event names. No ESM import is needed.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

The docs examples call `new Splide()` with no target. Here they use `new Splide( '.splide', ... )`, because the constructor needs a selector or an element.

## Enough / not enough slides

If a carousel doesn't have enough slides, you may want to hide the arrows and pagination and disable drag. Since v4.1.0, you can control this with the `is-overflow` status class or the `overflow` event.

For example, when slide widths come from `fixedWidth` (or `autoWidth`), the content can be narrower than the carousel. The docs demo disables some functions in that case and re-enables them once the viewport is resized so that the content is wide enough. Its options:

```js
new Splide( '.splide', {
  fixedWidth : 'min( 10rem, 20vw )',
  fixedHeight: '10rem',
  breakpoints: {
    640: { fixedHeight: '8rem' },
  },
} );
```

This also helps dynamic carousels. In the docs' second demo (`perPage: 3`, `rewind: true`), navigation and drag turn on only once enough items have been added.

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
- It applies to every layout, not only `fixedWidth` / `autoWidth`. With `perPage: 3`, 3 slides fit exactly and are not overflowing; 4 slides are.
- It is recalculated on mount, on `updated` and `refresh` (which includes `add()`, `remove()` and breakpoint changes), and on window `resize` / `load` when the root's size has changed.

## The `is-overflow` class

When the slides together are wider than their parent list element, Splide adds the `is-overflow` class to the root. Combined with `:not()`, it can hide the arrows when there isn't enough content:

```css
.splide:not(.is-overflow) .splide__arrows {
  display: none;
}
```

- The class is toggled only when the state changes.
- The built-in CSS (core and themes, 4.1.x) already contains `.splide:not(.is-overflow) .splide__pagination{display:none}`. Pagination therefore hides itself when slides don't overflow. Arrows don't; add the rule above.
- The class constant is `CLASS_OVERFLOW = 'is-overflow'`. It is exported only from the ESM build; with the global build, use the string.

## The `overflow` event

Splide emits `overflow` when the total width of the slides becomes wider or narrower than the list element. Call `on()` before `mount()` and check the callback argument:

```js
var splide = new Splide( '.splide' );

splide.on( 'overflow', function ( isOverflow ) {
  if ( isOverflow ) {
    // Enough slides
  } else {
    // Not enough slides
  }
} );

splide.mount();
```

| Event | Arguments | Since |
|---|---|---|
| `overflow` | `isOverflow: boolean` | 4.1.0 |

- Source: the stored state starts as `undefined`, so `overflow` always fires once during `mount()` with the initial state, `true` or `false`. After that it fires only on changes.
- The events guide lists no arguments for `overflow` and describes it as fired "by resizing the viewport". Source: it passes `isOverflow`, and also fires on mount, `updated` and `refresh`.
- The first call happens inside `mount()`, before Arrows, Pagination, Controller and Move have mounted. Setting `splide.options` there is safe: those components read the new values when they mount.
- In an extension, register the listener in `setup()`. Extensions mount after the core components, so a listener added in the extension's `mount()` misses the first call (tested).

## `Layout.isOverflow()`

| Method | Signature | Returns |
|---|---|---|
| `isOverflow()` | `isOverflow(): boolean` | `true` if the total width of the slides is wider than the list element, otherwise `false`. Always `true` for fade carousels. Since 4.1.0. |

```js
splide.Components.Layout.isOverflow();
```

Docs say (Layout component page): the signature is `getPadding( right: boolean ): number`, a copy error. Source says `isOverflow(): boolean`.

Related Layout methods: `listSize(): number` (list width, or height in vertical carousels, in px) and `sliderSize( withoutGap?: boolean ): number` (carousel size without the clones before the first slide). The docs list `sliderSize()` with no parameter; the source accepts `withoutGap`.

## Toggling arrows, pagination and drag

The `overflow` event and `options` together can switch navigation and drag:

```js
var splide = new Splide( '.splide' );

splide.on( 'overflow', function ( isOverflow ) {
  splide.options = {
    arrows    : isOverflow,
    pagination: isOverflow,
    drag      : isOverflow,
  };
} );

splide.mount();
```

From the source:

- `splide.options = {...}` merges the values into the current and base options and emits `updated`. Arrows and Pagination remount. Arrows that Splide generated are removed; arrows you supplied in HTML get `display: none` on their `.splide__arrows` wrapper. Drag re-reads `drag`, and the `splide--draggable` / `splide__track--draggable` classes update.
- If you use `drag: 'free'`, restore `'free'`, not `true`: `drag: isOverflow ? 'free' : false`.
- Autoplay is not affected, because it reads `autoplay` once at mount. Pause it yourself: `isOverflow ? splide.Components.Autoplay.play() : splide.Components.Autoplay.pause();`. Note that `play()` clears a user's earlier pause.
- The options are stored as base options, so they persist across breakpoint changes.
- Slides take their role when they are created: `tabpanel` if pagination is on, `group` if not. After the handler sets `pagination: false`, they keep `role="tabpanel"` with no tablist until the next `refresh()`. This also applies to the in-mount call, because Slides mounts before Layout. If the roles matter, call `splide.refresh()` outside the handler, for example right after `mount()` when `splide.Components.Layout.isOverflow()` is `false`. That rebuilds the Slide objects with the current `pagination` value.

## Loop carousels (clones)

Loop carousels need two more steps because of clones:

1. Update the `clones` option to generate or destroy clones.
2. Reset the carousel position with `go()`.

```js
var splide = new Splide( '.splide', { type: 'loop' } );

splide.on( 'overflow', function ( isOverflow ) {
  // Reset the carousel position
  splide.go( 0 );

  splide.options = {
    arrows    : isOverflow,
    pagination: isOverflow,
    drag      : isOverflow,
    clones    : isOverflow ? undefined : 0, // Toggle clones
  };
} );

splide.mount();
```

`clones: undefined` tells Splide to work out the number of clones automatically.

From `Clones.ts`:

- The Clones component checks the required count on `updated` and `resize`. It calls `refresh()` when the count rises, or when it drops to 0. So `clones: 0` removes them and `clones: undefined` brings them back.
- `clones` only applies to `type: 'loop'`. Other types always have 0 clones.
- The automatic count, per side, is: with `fixedWidth`, `ceil( trackWidth / ( slideWidth + gap ) )`; with `autoWidth`, `Splide.length`; otherwise `perPage * 2`.
- During the first, in-mount `overflow` call, `splide.go( 0 )` does nothing when `start` is 0, because the index is already 0. With a non-zero `start`, guard it so nothing moves during mount: `if ( splide.state.is( Splide.STATES.IDLE ) ) splide.go( 0 );`.

## Centering slides

When there aren't enough slides, you may want them centred rather than aligned left or right. There are two approaches: CSS and script.

### By CSS

The easiest approach uses the `is-overflow` status class. Add `justify-content: center` to the list only when the root doesn't have `is-overflow`:

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

### By script

The same result is possible with script. The important points:

- Check `Layout#isOverflow()` on the `resized` event, not `overflow`, and toggle `justifyContent = 'center'` on the list element to match.
- If you use `gap`, remove the last slide's margin only when `isOverflow()` returns `false`.

```js
var splide     = new Splide( '.splide' );
var Components = splide.Components;

splide.on( 'resized', function() {
  var isOverflow = Components.Layout.isOverflow();
  var list       = Components.Elements.list;
  var lastSlide  = Components.Slides.getAt( splide.length - 1 );

  if ( lastSlide ) {
    // Toggles `justify-content: center`
    list.style.justifyContent = isOverflow ? '' : 'center';

    // Remove the last margin
    if ( ! isOverflow ) {
      lastSlide.slide.style.marginRight = '';
    }
  }
} );

splide.mount();
```

Why `resized`: Layout rewrites the gap margin on every slide each time it re-lays out, then emits `resized`. Clearing the last margin in that handler survives, and the margin comes back automatically once the slides overflow again. `overflow` fires only when the state changes, so a margin cleared there is overwritten by the next resize.

From the source: the gap is written to `margin-right` in LTR, `margin-left` in RTL and `margin-bottom` in TTB. Clear the matching property for your direction. The CSS approach (`margin: 0 !important`) covers all three.

## Full recipe (CSS + JS, no build step)

This combines the page's techniques. When the slides don't fill the carousel, it hides the arrows (pagination is hidden by the built-in CSS), disables drag, removes the clones, and centres the slides:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    /* Hide controls when there is not enough content */
    .splide:not( .is-overflow ) .splide__arrows,
    .splide:not( .is-overflow ) .splide__pagination { /* pagination rule is already in 4.1 CSS */
      display: none;
    }

    /* Center slides when there is not enough content */
    .splide:not( .is-overflow ) .splide__list {
      justify-content: center;
    }

    /* Remove the trailing gap so centring is exact */
    .splide:not( .is-overflow ) .splide__slide:last-child {
      margin: 0 !important;
    }

    .splide__slide img { width: 100%; height: 100%; object-fit: contain; }
  </style>
</head>
<body>
  <section id="partners" class="splide" aria-label="Partners">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide"><img src="logo-1.svg" alt="Partner 1"></li>
        <li class="splide__slide"><img src="logo-2.svg" alt="Partner 2"></li>
        <li class="splide__slide"><img src="logo-3.svg" alt="Partner 3"></li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script>
    document.addEventListener( 'DOMContentLoaded', function () {
      var splide = new Splide( '#partners', {
        type       : 'loop',
        fixedWidth : 'min( 10rem, 20vw )',
        fixedHeight: '6rem',
        gap        : '1rem',
        breakpoints: {
          640: { fixedHeight: '4rem' },
        },
      } );

      // Must be registered before mount(): it also fires once during mount.
      splide.on( 'overflow', function ( isOverflow ) {
        // Reset the position (loop). A no-op during mount when start is 0.
        splide.go( 0 );

        splide.options = {
          arrows    : isOverflow,
          pagination: isOverflow,
          drag      : isOverflow,              // use isOverflow ? 'free' : false for free drag
          clones    : isOverflow ? undefined : 0, // loop only: auto clones / none
        };
      } );

      splide.mount();
    } );
  </script>
</body>
</html>
```

For a `slide`-type carousel, drop `splide.go( 0 )` and the `clones` line. With `trimSpace` (on by default), a non-overflowing slide carousel stays at position 0.

## Pitfalls

- Registering `overflow` after `mount()` misses the initial state.
- In a loop carousel, forgetting `clones: 0` leaves clones visible beside the real slides when there is not enough content.
- Centring by CSS without removing the last slide's margin leaves the slides off centre by one gap.
- Clearing the last margin in an `overflow` handler doesn't stick. Use `resized`, or the CSS `!important` rule.
- `drag: isOverflow` turns `'free'` drag into normal drag when overflow returns.
- `splide.options = { autoplay: false }` has no runtime effect. Use `Components.Autoplay.pause()`.
- A fade carousel always counts as overflowing.
