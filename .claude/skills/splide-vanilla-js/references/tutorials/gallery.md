# Gallery

Source: https://splidejs.com/tutorials/gallery/

Controls a carousel through the basic APIs (`Splide#go()`, `Splide#index`, the `mounted` and `move` events) to build a gallery. A main Splide carousel sits above a plain `<ul>` of thumbnails. The thumbnails are not a Splide instance. Clicking a thumbnail moves the carousel, and when the carousel moves, the active thumbnail changes. Everything uses the global `Splide` from the CDN script:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

If there are too many thumbnails to fit in one row, use a second Splide carousel with `isNavigation` and `sync()` instead. See `thumbnail-carousel.md`.

## 1. HTML and CSS

Markup for the carousel and the thumbnails. Describe the images with `alt`. The tutorial uses `image1.jpg` and `thumbnail1.jpg` with an empty `alt` as placeholders. Real URLs and alt text are filled in here.

```html
<section id="main-carousel" class="splide" aria-label="My Awesome Gallery">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1043/1200/675" alt="Granite cliff rising above a pine forest and a river">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1044/1200/675" alt="Person standing beside a rushing river in a misty forest">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1039/1200/675" alt="Waterfall flowing down a mossy, forested hillside">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/15/1200/675" alt="Waterfall pouring into a rocky gorge">
      </li>
    </ul>
  </div>
</section>

<ul id="thumbnails" class="thumbnails">
  <li class="thumbnail">
    <img src="https://picsum.photos/id/1043/140/140" alt="Granite cliff above a pine forest">
  </li>
  <li class="thumbnail">
    <img src="https://picsum.photos/id/1044/140/140" alt="Person beside a misty river">
  </li>
  <li class="thumbnail">
    <img src="https://picsum.photos/id/1039/140/140" alt="Waterfall on a mossy hillside">
  </li>
  <li class="thumbnail">
    <img src="https://picsum.photos/id/15/140/140" alt="Waterfall in a rocky gorge">
  </li>
</ul>
```

Style the thumbnails however you like. The tutorial uses:

```css
.thumbnails {
  display: flex;
  margin: 1rem auto 0;
  padding: 0;
  justify-content: center;
}

.thumbnail {
  width: 70px;
  height: 70px;
  overflow: hidden;
  list-style: none;
  margin: 0 0.2rem;
  cursor: pointer;
}

.thumbnail img {
  width: 100%;
  height: auto;
}
```

`height: auto` with a 70x70 box crops only when the image is taller than it is wide. Use square thumbnails, or switch to `height: 100%; object-fit: cover;`.

## 2. Main carousel

Apply Splide to the main carousel. Pagination is disabled because the thumbnails take over that job:

```js
var splide = new Splide( '#main-carousel', {
  pagination: false,
} );

splide.mount();
```

## 3. Thumbnails: click to navigate

To make the thumbnails clickable, listen to their `click` events:

```js
// Collects LI elements:
var thumbnails = document.getElementsByClassName( 'thumbnail' );

for ( var i = 0; i < thumbnails.length; i++ ) {
  initThumbnail( thumbnails[ i ], i );
}

// The function to initialize each thumbnail.
function initThumbnail( thumbnail, index ) {
  thumbnail.addEventListener( 'click', function () {
    // move the carousel
  } );
}
```

When the user clicks a thumbnail, the main carousel should move to the matching slide. `Splide#go()` accepts a slide index:

```js
var splide = new Splide( ... );

...

function initThumbnail( thumbnail, index ) {
  thumbnail.addEventListener( 'click', function () {
    splide.go( index );
  } );
}
```

## 4. Highlight the active thumbnail

The user still can't tell which thumbnail is selected. Add an `is-active` class to the active thumbnail and toggle it on the `move` event, which fires every time the carousel moves:

```js
var thumbnails = document.getElementsByClassName( 'thumbnail' );
var current; // Keeps the current thumbnail

...

splide.on( 'move', function () {
  if ( current ) {
    current.classList.remove( 'is-active' );
  }

  // Splide#index returns the latest slide index:
  var thumbnail = thumbnails[ splide.index ];

  if ( thumbnail ) {
    thumbnail.classList.add( 'is-active' );
    current = thumbnail;
  }
} );
```

`current` holds the current thumbnail element, so `is-active` can be removed when the selection changes.

Verified: `splide.index` is already updated when `move` fires (the Controller sets the index before it emits `move`). The handler also receives `( newIndex, prevIndex, destIndex )`. In a `loop` carousel, `splide.index` is the normalized index (0 to length - 1), so it maps directly to the thumbnails.

Style the selected thumbnail with the class:

```css
.thumbnail {
  ...
  opacity: 0.3;
}

.thumbnail.is-active {
  opacity: 1;
}
```

## 5. Initial active state

The class is only added when the carousel moves, so nothing is active at first. Also listen to `mounted`, which fires right after the carousel is initialized:

```js
splide.on( 'mounted move', function () {
  ...
} );
```

Register it **before** `splide.mount()`. A `mounted` listener added after `mount()` never fires.

## 6. Whole code

```js
var splide = new Splide( '#main-carousel', {
  pagination: false,
} );

var thumbnails = document.getElementsByClassName( 'thumbnail' );
var current;

for ( var i = 0; i < thumbnails.length; i++ ) {
  initThumbnail( thumbnails[ i ], i );
}

function initThumbnail( thumbnail, index ) {
  thumbnail.addEventListener( 'click', function () {
    splide.go( index );
  } );
}

splide.on( 'mounted move', function () {
  var thumbnail = thumbnails[ splide.index ];

  if ( thumbnail ) {
    if ( current ) {
      current.classList.remove( 'is-active' );
    }

    thumbnail.classList.add( 'is-active' );
    current = thumbnail;
  }
} );

splide.mount();
```

The tutorial links a CodeSandbox of this code: https://codesandbox.io/s/gallery-hkzhm

**Caveat: `destroy` option.** This gallery does not work with the `destroy` option, for example `breakpoints: { 640: { destroy: true } }`, because Splide discards every handler registered with `splide.on()` when it is destroyed. Verified on 4.1.4: after a breakpoint destroy and remount, clicking a thumbnail still moves the carousel, because that is a plain DOM listener. However, the `mounted move` handler is gone, so the active thumbnail never updates again. The tutorial says an extension is needed for this case. A tested one is under [Variations](#variations).

## 7. Tips: accessibility

The tutorial strongly recommends:

- `tabindex="0"` on the thumbnails, for keyboard users.
- An `aria-label` saying what clicking each thumbnail does (`aria-label="Go to slide 1"`).
- `aria-current` on the active thumbnail.
- `aria-controls` linking each thumbnail to its slide.

How the complete page implements them:

- Each thumbnail's content is wrapped in a `<button type="button">`. A button is focusable without `tabindex` and activates on Enter and Space without a keydown handler. Its click bubbles to the `<li>`, so the tutorial's `click` listener is unchanged, and the `<ul>`/`<li>` list semantics stay valid. A `role="button"` on the `<li>` itself would break the list.
- `aria-label="Go to slide N"`, `aria-current="true"` and `aria-controls` go on the button.
- Splide generates the slide IDs on mount as `<root id>-slide01`, `-slide02`, and so on (`main-carousel-slide01`). Read them in a `mounted` handler with `splide.Components.Slides.getAt( i ).slide.id`.

To skip the inner button, use `<div class="thumbnail">` items in a `<div>` container. List items cannot carry `role="button"`. Add `tabindex="0"`, `role="button"` and the aria attributes to each thumbnail, plus a key handler, because `tabindex` alone does not make Enter or Space activate it:

```js
function initThumbnail( thumbnail, index ) {
  thumbnail.setAttribute( 'tabindex', '0' );
  thumbnail.setAttribute( 'role', 'button' );
  thumbnail.setAttribute( 'aria-label', 'Go to slide ' + ( index + 1 ) );

  thumbnail.addEventListener( 'click', function () {
    splide.go( index );
  } );

  thumbnail.addEventListener( 'keydown', function ( e ) {
    if ( e.key === 'Enter' || e.key === ' ' ) {
      e.preventDefault();
      splide.go( index );
    }
  } );
}
```

The button approach used in the complete page is simpler.

## Complete page

Save as `.html` and open it in a browser.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Gallery - Splide</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    body {
      margin: 0;
      padding: 2rem 1rem;
      font-family: system-ui, sans-serif;
    }

    #main-carousel {
      max-width: 960px;
      margin: 0 auto;
    }

    #main-carousel .splide__slide img {
      width: 100%;
      height: auto;
      display: block;
    }

    .thumbnails {
      display: flex;
      margin: 1rem auto 0;
      padding: 0;
      justify-content: center;
    }

    .thumbnail {
      width: 70px;
      height: 70px;
      overflow: hidden;
      list-style: none;
      margin: 0 0.2rem;
      cursor: pointer;
      opacity: 0.3;
    }

    .thumbnail.is-active {
      opacity: 1;
    }

    .thumbnail img {
      width: 100%;
      height: auto;
    }

    /* Button reset: the button inside each thumbnail provides keyboard access (see Tips) */
    .thumbnail button {
      display: block;
      width: 100%;
      height: 100%;
      padding: 0;
      border: 0;
      background: none;
      cursor: pointer;
    }

    .thumbnail button:focus-visible {
      outline: 3px solid #0bf;
      outline-offset: -3px;
    }
  </style>
</head>
<body>
  <section id="main-carousel" class="splide" aria-label="My Awesome Gallery">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1043/1200/675" alt="Granite cliff rising above a pine forest and a river">
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1044/1200/675" alt="Person standing beside a rushing river in a misty forest">
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1039/1200/675" alt="Waterfall flowing down a mossy, forested hillside">
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/15/1200/675" alt="Waterfall pouring into a rocky gorge">
        </li>
      </ul>
    </div>
  </section>

  <ul id="thumbnails" class="thumbnails">
    <li class="thumbnail">
      <button type="button"><img src="https://picsum.photos/id/1043/140/140" alt="Granite cliff above a pine forest"></button>
    </li>
    <li class="thumbnail">
      <button type="button"><img src="https://picsum.photos/id/1044/140/140" alt="Person beside a misty river"></button>
    </li>
    <li class="thumbnail">
      <button type="button"><img src="https://picsum.photos/id/1039/140/140" alt="Waterfall on a mossy hillside"></button>
    </li>
    <li class="thumbnail">
      <button type="button"><img src="https://picsum.photos/id/15/140/140" alt="Waterfall in a rocky gorge"></button>
    </li>
  </ul>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script>
    var splide = new Splide( '#main-carousel', {
      pagination: false,
    } );

    var thumbnails = document.getElementsByClassName( 'thumbnail' );
    var current;

    for ( var i = 0; i < thumbnails.length; i++ ) {
      initThumbnail( thumbnails[ i ], i );
    }

    function initThumbnail( thumbnail, index ) {
      // Tips (a11y): tell the user what clicking the thumbnail does.
      // The button makes it focusable and activatable by Enter/Space without extra key handlers.
      thumbnail.querySelector( 'button' ).setAttribute( 'aria-label', 'Go to slide ' + ( index + 1 ) );

      thumbnail.addEventListener( 'click', function () {
        splide.go( index );
      } );
    }

    // Tips (a11y): link each thumbnail to its slide. Slide IDs ("main-carousel-slide01", ...)
    // are generated by Splide on mount, so read them in a "mounted" handler.
    splide.on( 'mounted', function () {
      for ( var i = 0; i < thumbnails.length; i++ ) {
        var Slide = splide.Components.Slides.getAt( i );

        if ( Slide ) {
          thumbnails[ i ].querySelector( 'button' ).setAttribute( 'aria-controls', Slide.slide.id );
        }
      }
    } );

    // "mounted" sets the initial active thumbnail; "move" fires on every move.
    splide.on( 'mounted move', function () {
      var thumbnail = thumbnails[ splide.index ];

      if ( thumbnail ) {
        if ( current ) {
          current.classList.remove( 'is-active' );
          current.querySelector( 'button' ).removeAttribute( 'aria-current' ); // Tips (a11y)
        }

        thumbnail.classList.add( 'is-active' );
        thumbnail.querySelector( 'button' ).setAttribute( 'aria-current', 'true' ); // Tips (a11y)
        current = thumbnail;
      }
    } );

    splide.mount();
  </script>
</body>
</html>
```

Verified in headless Chrome against 4.1.4 from the CDN. Thumbnail 1 starts active (`aria-current="true"`, opacity 1), and the others are at 0.3. The `aria-controls` values are `main-carousel-slide01` to `04`. Clicking thumbnail 3 moves to index 2 and moves the active state. `splide.go( '>' )` activates thumbnail 4. There are no console errors.

## Variations

- **Compatible with the `destroy` option (extension):** extension components are rebuilt and their `mount()` runs again every time Splide remounts after a breakpoint destroy. Move the gallery logic into one. Handlers registered with `Splide.on()` are removed by Splide on destroy. DOM listeners must be removed in `destroy()`, or they pile up. Verified: active state and clicks keep working after destroy and remount.
  ```js
  function ThumbnailsExtension( Splide, Components ) {
    var thumbnails = document.getElementsByClassName( 'thumbnail' );
    var listeners  = [];
    var current;

    function mount() {
      for ( var i = 0; i < thumbnails.length; i++ ) {
        bind( thumbnails[ i ], i );
      }

      Splide.on( 'mounted move', update ); // removed automatically when Splide is destroyed
    }

    function bind( thumbnail, index ) {
      function onClick() {
        Splide.go( index );
      }

      thumbnail.addEventListener( 'click', onClick );
      listeners.push( [ thumbnail, onClick ] );
    }

    function update() {
      var thumbnail = thumbnails[ Splide.index ];

      if ( thumbnail ) {
        if ( current ) {
          current.classList.remove( 'is-active' );
        }

        thumbnail.classList.add( 'is-active' );
        current = thumbnail;
      }
    }

    function destroy() {
      listeners.forEach( function ( entry ) {
        entry[ 0 ].removeEventListener( 'click', entry[ 1 ] );
      } );

      listeners = [];

      if ( current ) {
        current.classList.remove( 'is-active' );
      }
    }

    return { mount: mount, destroy: destroy };
  }

  var splide = new Splide( '#main-carousel', {
    pagination : false,
    breakpoints: {
      640: { destroy: true },
    },
  } );

  splide.mount( { ThumbnailsExtension: ThumbnailsExtension } );
  ```
  Component `mount()` runs before Splide emits `mounted`, so the `mounted` listener registered inside it fires. While the carousel is destroyed, the slides render as a plain stacked list, so you may want to hide the thumbnails in the same media query: `@media (max-width: 640px) { .thumbnails { display: none; } }`. Add the a11y attributes from the complete page in `bind()` and `update()` as needed.
- **Loop or fade main carousel:** `type: 'loop'` or `type: 'fade'` (with `rewind: true`) work unchanged, because `splide.index` is always the real slide index.
- **Several slides per page:** with `perPage: 3` and no `focus`, indexes past `Controller.getEnd()` are clamped. With 6 slides, `go( 5 )` lands on index 3, so clicking thumbnail 6 highlights thumbnail 4. Add `focus: 0` (or `focus: 'center'`) so that every index is reachable. The end index becomes `length - 1` and `trimSpace` still stops the track at the edges. Verified: with `focus: 0`, `go( 5 )` lands on index 5.
- **Different active style:** replace the opacity rules with `.thumbnail.is-active { outline: 2px solid #000; outline-offset: -2px; }`, or a border with `box-sizing: border-box`.
- **Lazy loading:** add `lazyLoad: 'nearby'` to the main options and use `data-splide-lazy` on the main `<img>` tags. The thumbnails are ordinary images (add `loading="lazy"` if needed).
- **Many thumbnails:** a flex row overflows on small screens. Allow wrapping (`flex-wrap: wrap`), or make the thumbnails a Splide `isNavigation` carousel (see `thumbnail-carousel.md`).
- **Scoped selectors:** with more than one gallery on a page, query the thumbnails relative to a wrapper (`wrapper.querySelectorAll( '.thumbnail' )`) instead of `document.getElementsByClassName`, and give each main carousel its own ID.
