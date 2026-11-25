# Carousel Progress

Source: https://splidejs.com/tutorials/carousel-progress/

A few lines of code add a progress bar that shows how far through the slides the carousel is. The bar is a pair of `<div>`s inside the Splide root, and its width is updated from `Splide#index` and `Controller#getEnd()` on every move. Everything uses the global `Splide` from the CDN script:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

This bar shows the carousel **position**. It is not the same thing as the autoplay timer bar (`.splide__progress > .splide__progress__bar`), which Splide's Autoplay component fills while it waits for the next slide (see `references/autoplay.md`). Keep the custom class names below. Splide treats an element named `splide__progress__bar` as the autoplay bar, and the theme styles it.

## 1. HTML and CSS

Create the bar with two `<div>` tags placed inside the Splide root, after the track. (The tutorial's `aria-label="..."` is a placeholder. Give it a real label.)

```html
<section class="splide" aria-label="...">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      ...
    </ul>
  </div>

  <!-- Add the progress bar element -->
  <div class="my-carousel-progress">
    <div class="my-carousel-progress-bar"></div>
  </div>
</section>
```

```css
.my-carousel-progress {
  background: #ccc;
}

.my-carousel-progress-bar {
  background: greenyellow;
  height: 2px;
  transition: width 400ms ease;
  width: 0;
}
```

Use the carousel speed (`options.speed`, default `400` ms) as the transition duration too. The carousel and the bar then look synchronized.

Custom elements inside the root (outside the track) are allowed. Never put extra elements between `.splide__track` and `.splide__list`.

## 2. JavaScript

The progress rate is `( current + 1 ) / ( end + 1 )`, where `current` is `Splide#index` and `end` is `Controller#getEnd()`, the last index the carousel can move to. Update the bar width every time the carousel moves:

```js
var splide = new Splide( '.splide' );
var bar    = splide.root.querySelector( '.my-carousel-progress-bar' );

// Updates the bar width whenever the carousel moves:
splide.on( 'mounted move', function () {
  var end  = splide.Components.Controller.getEnd() + 1;
  var rate = Math.min( ( splide.index + 1 ) / end, 1 );
  bar.style.width = String( 100 * rate ) + '%';
} );

splide.mount();
```

The code also listens to `mounted` to set the initial width. Register both events **before** `mount()`, or `mounted` never fires.

Why `getEnd()` and not `splide.length`: with `perPage: 3` and 9 slides, the last reachable index is 6. Dividing by `getEnd() + 1` makes the bar reach 100% on the last page. `Math.min( ..., 1 )` caps the rate in cases where the index can pass the end index, for example in a `loop` carousel with `perPage > 1`.

Verified against the 4.1.x source and in headless Chrome on 4.1.4:

- `splide.root`, `splide.index` and `splide.Components.Controller.getEnd()` all exist on the global build.
- `splide.index` is already updated when `move` fires.
- With 9 slides and default options, the width is 11.1111% initially, 55.5556% after `go( 4 )` and 100% after `go( 8 )`. With `perPage: 3`, `getEnd()` returns 6.

## 3. Tips from the tutorial

- **Clickable bar:** convert the clicked position into a target index, then move the carousel with `Splide#go()`.
- **Accessibility:** add ARIA attributes such as `aria-controls` and `aria-valuenow`. The WAI-ARIA slider role (https://www.w3.org/TR/wai-aria-1.2/#slider) describes what is needed.

Both are implemented and tested under [Variations](#variations).

## Complete page

Save as `.html` and open it in a browser.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Carousel Progress - Splide</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    body {
      margin: 0;
      padding: 2rem 1rem;
      font-family: system-ui, sans-serif;
    }

    .splide {
      max-width: 960px;
      margin: 0 auto;
    }

    /* Demo slide styling (not part of the tutorial) */
    .splide__slide {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 10rem;
      background: #f2f2f2;
      font-size: 2rem;
    }

    /* Progress bar (from the tutorial) */
    .my-carousel-progress {
      background: #ccc;
    }

    .my-carousel-progress-bar {
      background: greenyellow;
      height: 2px;
      transition: width 400ms ease; /* same as options.speed (default 400) */
      width: 0;
    }
  </style>
</head>
<body>
  <section class="splide" aria-label="Numbered slides with a progress bar">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">Slide 01</li>
        <li class="splide__slide">Slide 02</li>
        <li class="splide__slide">Slide 03</li>
        <li class="splide__slide">Slide 04</li>
        <li class="splide__slide">Slide 05</li>
        <li class="splide__slide">Slide 06</li>
        <li class="splide__slide">Slide 07</li>
        <li class="splide__slide">Slide 08</li>
        <li class="splide__slide">Slide 09</li>
      </ul>
    </div>

    <!-- Add the progress bar element -->
    <div class="my-carousel-progress">
      <div class="my-carousel-progress-bar"></div>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script>
    var splide = new Splide( '.splide' );
    var bar    = splide.root.querySelector( '.my-carousel-progress-bar' );

    // Updates the bar width whenever the carousel moves:
    splide.on( 'mounted move', function () {
      var end  = splide.Components.Controller.getEnd() + 1;
      var rate = Math.min( ( splide.index + 1 ) / end, 1 );
      bar.style.width = String( 100 * rate ) + '%';
    } );

    splide.mount();
  </script>
</body>
</html>
```

`new Splide( '.splide' )` only matches the **first** `.splide` on the page. With several carousels, create one instance per root element, and query each bar from that instance's `splide.root`.

## Variations

- **Clickable, keyboard-operable, accessible bar (both tutorial tips):** this makes the bar a WAI-ARIA `slider`. It enlarges the click area without making the bar look thicker, keeps the ARIA values in sync, and handles the keys the slider pattern expects. Verified in headless Chrome: clicking at 50% goes to index 4 of 9, the right edge goes to 8, Home goes to 0, ArrowRight goes to the next slide, and End goes to the end index. `aria-valuenow`/`aria-valuemax`/`aria-valuetext` and `aria-controls` (`splide01-track`) are updated.
  ```html
  <div class="my-carousel-progress" role="slider" tabindex="0" aria-label="Carousel position" aria-valuemin="1">
    <div class="my-carousel-progress-bar"></div>
  </div>
  ```
  ```css
  .my-carousel-progress {
    background: #ccc;
    cursor: pointer;
    padding: 6px 0;               /* 14px tall hit area ... */
    background-clip: content-box; /* ... while the grey track stays 2px tall */
  }

  .my-carousel-progress:focus-visible {
    outline: 3px solid #0bf;
  }
  ```
  ```js
  var splide   = new Splide( '.splide' );
  var progress = splide.root.querySelector( '.my-carousel-progress' );
  var bar      = splide.root.querySelector( '.my-carousel-progress-bar' );

  function getEnd() {
    return splide.Components.Controller.getEnd();
  }

  function updateBar() {
    var end = getEnd() + 1;
    var now = Math.min( splide.index + 1, end );

    bar.style.width = String( 100 * now / end ) + '%';
    progress.setAttribute( 'aria-valuemax', String( end ) );
    progress.setAttribute( 'aria-valuenow', String( now ) );
    progress.setAttribute( 'aria-valuetext', 'Position ' + now + ' of ' + end );
  }

  splide.on( 'mounted move', updateBar );

  // Click: convert the clicked x position into a target index
  progress.addEventListener( 'click', function ( e ) {
    var rect = progress.getBoundingClientRect();
    var rate = ( e.clientX - rect.left ) / rect.width;
    var end  = getEnd();

    splide.go( Math.max( 0, Math.min( Math.floor( rate * ( end + 1 ) ), end ) ) );
  } );

  // Keyboard support required by role="slider"
  progress.addEventListener( 'keydown', function ( e ) {
    var map = { ArrowRight: '>', ArrowUp: '>', ArrowLeft: '<', ArrowDown: '<', Home: 0, End: getEnd() };

    if ( e.key in map ) {
      e.preventDefault();
      splide.go( map[ e.key ] );
    }
  } );

  splide.mount();
  splide.on( 'updated resized', updateBar ); // registered after mount(): see next item
  progress.setAttribute( 'aria-controls', splide.Components.Elements.track.id );
  ```
  `'>'`/`'<'` move the same way as the arrows: by `perMove` if set, otherwise by one page (one slide when `focus` is set). Use `'+1'`/`'-1'` to move by exactly one slide. Splide gives the track an ID on mount (`<root id>-track`, or `splide01-track` when the root has no ID).
- **Responsive `perPage` / `fixedWidth`:** when breakpoints change `perPage`, the end index changes too. Recompute the bar on `updated` (breakpoint or `splide.options = {...}` change) and `resized`, but register that listener **after** `mount()`. Handlers run in registration order. A handler registered before `mount()` runs before the Controller has recomputed the end index, and reads the stale value. Verified: switching from `perPage: 3` to `1` gave a stale end of 6 to the early handler and the correct 8 to the late one.
  ```js
  splide.on( 'mounted move', updateBar );
  splide.mount();
  splide.on( 'updated resized', updateBar );
  ```
- **Transition matched to the real speed:** under `prefers-reduced-motion`, Splide's default `reducedMotion` option sets `speed` to 0. To follow it, set the duration from the live options in the handler: `bar.style.transitionDuration = splide.options.speed + 'ms';`. A custom `speed` option needs the same.
- **Several slides per page or loop:** no code change. `getEnd()` already accounts for `perPage`, `focus` and `omitEnd`, and the `Math.min` cap handles loop indexes.
- **Thicker or styled bar:** change `height`, `background` and add `border-radius` on `.my-carousel-progress-bar` (and on `.my-carousel-progress` for the track). Put the bar above the slides by moving the progress `<div>` before `.splide__track`. It must stay inside the root so that `splide.root.querySelector` finds it.
- **Purely decorative bar:** if you keep the plain, non-interactive bar, it repeats information that the slides already announce ("3 of 9"). Mark it `aria-hidden="true"` instead of giving it partial ARIA.
