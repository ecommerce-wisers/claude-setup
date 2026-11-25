# Thumbnail Carousel

Source: https://splidejs.com/tutorials/thumbnail-carousel/

A main carousel controlled by a carousel of clickable thumbnails. It uses two Splide features: `Splide#sync()` links two carousels so that moving one moves the other, and the `isNavigation` option makes a carousel's slides clickable (and keyboard-operable) navigation buttons. (The tutorial intro spells it `isNaviation`. The option is `isNavigation`.) Everything uses the global `Splide` from the CDN script:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

For a gallery with plain (non-Splide) thumbnails driven by `go()` and events, see `gallery.md`.

## 1. Thumbnail carousel markup

Start with the thumbnail carousel. Its `aria-label` should say what selecting a thumbnail does and which carousel it controls:

```html
<section
  id="thumbnail-carousel"
  class="splide"
  aria-label="The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel."
>
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1015/200/120" alt="Fjord between steep cliffs">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1018/200/120" alt="Green hills and sea cliffs">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1016/200/120" alt="Red desert canyon at sunset">
      </li>
    </ul>
  </div>
</section>
```

(The tutorial uses `thumbnail01.jpg` with an empty `alt` as placeholders.)

## 2. Fixed slide width instead of `perPage`

A thumbnail carousel shows several slides per page. `perPage` is hard to tune for every screen size, even with breakpoints, so fix each slide's width with `fixedWidth`. The number of visible thumbnails then follows the window size:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#thumbnail-carousel', {
    fixedWidth: 100,
    gap       : 10,
    rewind    : true,
    pagination: false,
  } ).mount();
} );
```

`fixedWidth` overrides `perPage`. Numbers are px. Strings may use any CSS unit.

## 3. Fit the images to the thumbnails

If all thumbnails have the same aspect ratio, this CSS is enough:

```css
.splide__slide img {
  width: 100%;
  height: auto;
}
```

If not, crop each thumbnail with `object-fit`. Internet Explorer does not support it. For it to work, the slides need an explicit height, so set `fixedHeight`:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#thumbnail-carousel', {
    fixedWidth : 100,
    fixedHeight: 60,
    gap        : 10,
    rewind     : true,
    pagination : false,
  } ).mount();
} );
```

Then apply the property to the thumbnails:

```css
.splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

When the page has a main carousel too, scope these rules (`#thumbnail-carousel .splide__slide img`) so they don't also crop the main images.

## 4. Remaining problems: active state, mobile size, centering

At this point:

- the user cannot tell which slide is selected,
- `100px` is probably too wide for mobile devices,
- the active slide should be centered when possible.

**Active-thumbnail CSS.** A theme CSS (`splide.min.css`, `splide-skyblue.min.css`, `splide-sea-green.min.css`) already emphasizes the active navigation slide with a border. These rules are in the 4.1.x theme, applied through the `splide__track--nav` class that Splide adds to the track when `isNavigation` is on:

```css
/* already in the theme CSS */
.splide__track--nav > .splide__list > .splide__slide {
  border: 3px solid transparent;
  cursor: pointer;
}

.splide__track--nav > .splide__list > .splide__slide.is-active {
  border: 3px solid #000;
}
```

Without a theme (for example `splide-core.min.css` only), style the `is-active` status class yourself:

```css
.splide__slide {
  opacity: 0.6;
}

.splide__slide.is-active {
  opacity: 1;
}
```

(Scope these to `#thumbnail-carousel` when the main carousel is on the same page. Otherwise its slides are dimmed too.)

**Size and centering.** `breakpoints` handles the second problem and `focus: 'center'` handles the third. If you are not using `object-fit`, `fixedHeight` is not necessary.

```js
new Splide( '#thumbnail-carousel', {
  fixedWidth : 100,
  fixedHeight: 60,
  gap        : 10,
  rewind     : true,
  pagination : false,
  focus      : 'center',
  breakpoints: {
    600: {
      fixedWidth : 60,
      fixedHeight: 44,
    },
  },
} ).mount();
```

`breakpoints` keys are max-widths by default (`mediaQuery: 'max'`), so the 60x44 size applies at 600px and below. With `type: 'slide'` and the default `trimSpace: true`, `focus: 'center'` centers the active thumbnail without leaving empty space at either end.

## 5. Make the thumbnails clickable: `isNavigation`

```js
new Splide( '#thumbnail-carousel', {
  fixedWidth  : 100,
  fixedHeight : 60,
  gap         : 10,
  rewind      : true,
  pagination  : false,
  isNavigation: true,
  breakpoints : {
    600: {
      fixedWidth : 60,
      fixedHeight: 44,
    },
  },
} ).mount();
```

The thumbnail carousel must **not** have pagination. Otherwise the accessibility tree gets confusing: thumbnails are buttons that control another carousel, and pagination would add a second set of tab controls. Keep `pagination: false`.

To ignore small mouse drags, so that a slightly shaky click still counts as a click, use `dragMinThreshold`. A number only sets the touch threshold (the mouse threshold stays 0). Pass an object to set the mouse threshold: `dragMinThreshold: { mouse: 4, touch: 10 }`.

Correction: the tutorial's step 4 code includes `focus: 'center'`, but its step 5 code and its final code leave it out. The demo on the tutorial page uses it. Add it back if you want the active thumbnail centered. The complete page below includes it.

## 6. Main carousel

The tutorial uses a fade carousel as the main carousel. Give it a different ID, such as `main-carousel` (or a class), so the script can grab it. Give it its own `aria-label`. The thumbnail carousel's label refers to it by name ("Beautiful Gallery").

```html
<section id="main-carousel" class="splide" aria-label="Beautiful Gallery">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1015/1200/675" alt="Deep blue fjord between steep rocky cliffs">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1018/1200/675" alt="Green hills and sea cliffs under a cloudy sky">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1016/1200/675" alt="Red desert canyon lit by the setting sun">
      </li>
    </ul>
  </div>
</section>
```

```js
var main = new Splide( '#main-carousel', {
  type      : 'fade',
  rewind    : true,
  pagination: false,
  arrows    : false,
} );
```

This disables arrows and pagination. To enable them, delete those two options. `fade` cannot loop, so `rewind: true` lets it wrap from the last slide to the first. (The tutorial's demo also used `heightRatio: 0.5625` and `cover: true`. They are not in its code.)

## 7. Sync

Sync the main carousel to the thumbnails with `Splide#sync()`. **Call it before `mount()`.**

```js
var main       = new Splide( '#main-carousel', { ... } );
var thumbnails = new Splide( '#thumbnail-carousel', { ... } );

main.sync( thumbnails );
main.mount();
thumbnails.mount();
```

The tutorial's whole script:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  var main = new Splide( '#main-carousel', {
    type      : 'fade',
    rewind    : true,
    pagination: false,
    arrows    : false,
  } );

  var thumbnails = new Splide( '#thumbnail-carousel', {
    fixedWidth  : 100,
    fixedHeight : 60,
    gap         : 10,
    rewind      : true,
    pagination  : false,
    isNavigation: true,
    breakpoints : {
      600: {
        fixedWidth : 60,
        fixedHeight: 44,
      },
    },
  } );

  main.sync( thumbnails );
  main.mount();
  thumbnails.mount();
} );
```

### `sync()` ordering rule

These orderings were verified against 4.1.4 in headless Chrome:

| Order | Result |
|---|---|
| `main.sync( thumbs ); main.mount(); thumbs.mount();` | Correct. Use this. |
| `thumbs.sync( main ); main.mount(); thumbs.mount();` | Correct. It does not matter which instance calls `sync()`. |
| `main.sync( thumbs ); thumbs.mount(); main.mount();` | Syncing works, but the thumbnails' navigation a11y is not applied: no `role="button"`, no "Go to slide N" label, no `aria-controls`. The console shows `Cannot read properties of undefined (reading 'getAt')` once per slide. The error is raised inside an event handler, so `mount()` itself does not throw. |
| `main.mount(); main.sync( thumbs ); thumbs.mount();` | Throws `Cannot read properties of undefined (reading 'remount')`. Syncing does not work. |
| `main.mount(); thumbs.mount(); main.sync( thumbs );` | Works in 4.1.x, because `sync()` remounts the Sync component when the caller is already mounted. This behavior is undocumented, so don't rely on it. |

Rule: call `sync()` before any `mount()`, then mount the main carousel **before** the `isNavigation` carousel. The navigation carousel reads the main carousel's slide IDs for `aria-controls` while it mounts.

How syncing works (`components/Sync`): each instance listens to the other's `move` event and calls `go()` on it. A loop target receives the destination index, and any other target receives the normalized index. Syncing is two-way, so dragging the main carousel also moves the thumbnails. More than two carousels can be chained by calling `sync()` several times on the same primary.

### What `isNavigation` does

Verified from `components/Sync`, `components/Slides/Slide.ts` and `components/Elements`, and confirmed in the browser:

- The root gets `splide--nav` and the track gets `splide__track--nav`. The theme's active-border CSS hooks on that class.
- Clicking a slide calls `go( slide.index )`. Pressing **Enter** or **Space** on a focused slide does the same.
- `slideFocus` defaults to `true` when `isNavigation` is on. Every thumbnail slide then gets `role="button"` and `tabindex="0"` (`-1` while hidden), and `aria-roledescription` is removed.
- Each thumbnail gets `aria-label="Go to slide N"` (from `i18n.slideX`) and `aria-controls="main-carousel-slide0N"`. Splide generates the main slide IDs as `<root id>-slide01`, `-slide02`, and so on.
- The active thumbnail gets the `is-active` class and `aria-current="true"`.
- With `direction: 'ttb'`, the list gets `aria-orientation="vertical"`.
- The main carousel's slides get `role="group"` (because `pagination: false`), `aria-roledescription="slide"` and `aria-label="1 of 6"`.

Known issue in 4.1.x (reproduced on 4.1.4): `thumbnails.refresh()`, `thumbnails.add()`, `thumbnails.remove()` and a breakpoint that changes `direction` all rebuild the slide components. Splide does not re-run the navigation setup afterwards, so the thumbnails fall back to `role="group"` and `aria-label="1 of 6"` and lose `aria-controls`. Click, Enter/Space, `is-active` and `aria-current` keep working. Workaround: re-emit the navigation event after every refresh, and register the handler **after** `mount()` so that it runs after Splide has rebuilt the slides:

```js
main.sync( thumbnails );
main.mount();
thumbnails.mount();

thumbnails.on( 'refresh', function () {
  thumbnails.emit( 'navigation:mounted', thumbnails.splides );
} );
```

Inside Splide, only the slide components listen to `navigation:mounted` (verified in the source), so re-emitting it has no other side effects. Your own `navigation:mounted` listeners will run again, though.

### Thumbnail a11y labelling

- Put a descriptive `aria-label` on the thumbnail carousel root that says what it controls, for example: "The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel."
- Put an `aria-label` (or `aria-labelledby`) on the main carousel, for example "Beautiful Gallery".
- Splide **overwrites** any `aria-label` you put on navigation slides with `i18n.slideX`. To change the wording, set it on the thumbnail instance: `i18n: { slideX: 'Show image %s' }`. The thumbnail images' `alt` text is not used as the button name, because the slide's `aria-label` takes precedence. Keep a short alt anyway.
- Keep `pagination: false` on the thumbnail carousel.

## Complete page

Save as `.html` and open it in a browser.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Thumbnail Carousel - Splide</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    body {
      margin: 0;
      padding: 2rem 1rem;
      font-family: system-ui, sans-serif;
    }

    .gallery {
      max-width: 960px;
      margin: 0 auto;
    }

    /* Main carousel: images share one aspect ratio, so width 100% / height auto is enough */
    #main-carousel .splide__slide img {
      width: 100%;
      height: auto;
      display: block;
    }

    #thumbnail-carousel {
      margin-top: 1rem;
    }

    /* Thumbnails: crop to the fixedWidth x fixedHeight box */
    #thumbnail-carousel .splide__slide img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      display: block;
    }

    /*
     * Active-thumbnail styling. splide.min.css already adds a 3px border to the active
     * navigation slide (.splide__track--nav > .splide__list > .splide__slide.is-active).
     * Without a theme CSS (splide-core.min.css only), style the is-active class yourself:
     */
    #thumbnail-carousel .splide__slide {
      opacity: 0.6;
    }

    #thumbnail-carousel .splide__slide.is-active {
      opacity: 1;
    }
  </style>
</head>
<body>
  <div class="gallery">
    <!-- Main carousel. Needs its own ID (or class) so the script can grab it. -->
    <section id="main-carousel" class="splide" aria-label="Beautiful Gallery">
      <div class="splide__track">
        <ul class="splide__list">
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1015/1200/675" alt="Deep blue fjord between steep rocky cliffs">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1018/1200/675" alt="Green hills and sea cliffs under a cloudy sky">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1016/1200/675" alt="Red desert canyon lit by the setting sun">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1036/1200/675" alt="Snow-covered campsite below jagged mountains">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1050/1200/675" alt="Turquoise sea around white chalk cliffs">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/29/1200/675" alt="Snow-covered mountain range under a blue sky">
          </li>
        </ul>
      </div>
    </section>

    <!-- Thumbnail carousel. The aria-label tells screen reader users what selecting a thumbnail does. -->
    <section
      id="thumbnail-carousel"
      class="splide"
      aria-label="The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel."
    >
      <div class="splide__track">
        <ul class="splide__list">
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1015/200/120" alt="Fjord between steep cliffs">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1018/200/120" alt="Green hills and sea cliffs">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1016/200/120" alt="Red desert canyon at sunset">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1036/200/120" alt="Snowy campsite below mountains">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/1050/200/120" alt="Chalk cliffs over a turquoise sea">
          </li>
          <li class="splide__slide">
            <img src="https://picsum.photos/id/29/200/120" alt="Snow-covered mountain range">
          </li>
        </ul>
      </div>
    </section>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script>
    var main = new Splide( '#main-carousel', {
      type      : 'fade',
      rewind    : true,
      pagination: false,
      arrows    : false,
    } );

    var thumbnails = new Splide( '#thumbnail-carousel', {
      fixedWidth  : 100,
      fixedHeight : 60,
      gap         : 10,
      rewind      : true,
      pagination  : false,
      focus       : 'center', // shown in the tutorial's breakpoints step, dropped from its final code
      isNavigation: true,
      breakpoints : {
        600: {
          fixedWidth : 60,
          fixedHeight: 44,
        },
      },
    } );

    // Order matters: sync() BEFORE mount(), then mount the main carousel first.
    main.sync( thumbnails );
    main.mount();
    thumbnails.mount();
  </script>
</body>
</html>
```

Verified in headless Chrome against 4.1.4 from the CDN. Thumbnails are 100x60 with `role="button"`, `aria-label="Go to slide 1"`, `aria-controls="main-carousel-slide01"` and `aria-current="true"` on the active one. Clicking thumbnail 4 moves the main carousel to index 3. `main.go( 5 )` moves the thumbnails to 5. Enter on a thumbnail navigates. There are no console errors.

## Variations

- **Main carousel controls:** delete `arrows: false` and/or `pagination: false` from the main options to get arrows and dots. Keep pagination off on the thumbnails.
- **Looping main carousel:** use `type: 'loop'` on the main instead of `'fade'` (drop `rewind`, which has no effect in loop). Syncing handles loop indexes. The thumbnails can stay `type: 'slide'` with `rewind: true`.
- **Cropped main images:** add `heightRatio: 0.5625` to the main options plus `#main-carousel .splide__slide img { width: 100%; height: 100%; object-fit: cover; }`. Alternatively, use `cover: true` with `heightRatio` (the tutorial demo did this).
- **Vertical thumbnails beside the main carousel:** `direction: 'ttb'` needs a `height` (or `heightRatio`). Verified working with a loop main:
  ```css
  .gallery { display: flex; gap: 10px; }
  #main-carousel { flex: 1; min-width: 0; }
  ```
  ```js
  var thumbnails = new Splide( '#thumbnail-carousel', {
    direction   : 'ttb',
    height      : '400px',
    fixedWidth  : 100,
    fixedHeight : 60,
    gap         : 10,
    rewind      : true,
    pagination  : false,
    arrows      : false,
    focus       : 'center',
    isNavigation: true,
    breakpoints : {
      600: { direction: 'ltr', height: undefined, fixedWidth: 60, fixedHeight: 44 },
    },
  } );

  main.sync( thumbnails );
  main.mount();
  thumbnails.mount();

  // Changing direction in a breakpoint calls refresh(), which drops the navigation ARIA (see "Known issue")
  thumbnails.on( 'refresh', function () {
    thumbnails.emit( 'navigation:mounted', thumbnails.splides );
  } );
  ```
  ```css
  @media (max-width: 600px) {
    .gallery { flex-direction: column; }
  }
  ```
  Verified: at 600px and below the thumbnails switch to a horizontal 60x44 strip, and above it they return to a 100x60 vertical strip 400px tall. The workaround keeps "Go to slide N" and `role="button"` through both switches. One leftover: after the refresh, the list's `aria-orientation="vertical"` is not set again. This is minor, because the list has `role="presentation"`.
- **No theme CSS:** load `splide-core.min.css`. Add the `is-active` opacity rules from step 4, or copy the theme's border rules shown there, plus a focus style such as `#thumbnail-carousel .splide__slide:focus-visible { outline: 3px solid #0bf; outline-offset: -3px; }`.
- **Clicks ignored after tiny drags:** `dragMinThreshold: { mouse: 4, touch: 10 }` on the thumbnails.
- **Lazy loading the main images:** add `lazyLoad: 'nearby'` to the main options and use `data-splide-lazy="URL"` instead of `src`. Thumbnails are small, so they are usually loaded eagerly.
- **Translated or custom labels:** `i18n: { slideX: 'Bild %s anzeigen' }` on the thumbnails changes "Go to slide %s". On the main carousel, `i18n: { slideLabel: '%s von %s', carousel: 'Karussell', slide: 'Folie' }`. Translate both root `aria-label`s too.
- **Several galleries on one page:** give every pair unique IDs (slide IDs and `aria-controls` derive from the root ID), and init each pair with its own `sync()`/`mount()` sequence. If a root has no ID, Splide assigns `splide01`, `splide02`, and so on.
- **Breakpoint `destroy`:** `sync()` links are kept on a breakpoint destroy (`destroy: true`) and restored on remount. This was verified by destroying and remounting a synced main carousel: clicks and `go()` still sync both ways. `destroy: 'completely'` or `splide.destroy()` (completely by default) removes the links.
