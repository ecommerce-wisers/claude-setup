# Image Carousel

Source: https://splidejs.com/tutorials/image-carousel/

Builds a typical image carousel, with and without text, and shows how to normalize image dimensions (CSS `object-fit` + `heightRatio`, or the legacy `cover` option). It also covers a card carousel and a fullscreen carousel. Everything here uses the global `Splide` from the CDN script. The tutorial needs no npm, imports or build step.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
```

## 1. Basic image carousel

Create the basic structure (root `.splide` > `.splide__track` > `.splide__list` > `.splide__slide`) and put one image in each slide. Describe the carousel with `aria-label` (or `aria-labelledby`) and every image with `alt`, because screen reader users depend on them. The tutorial uses `image01.jpg` and an empty `alt` as placeholders. The snippets below fill in real URLs and alt text.

```html
<section id="image-carousel" class="splide" aria-label="Beautiful Images">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1015/1200/600" alt="Deep blue fjord between steep rocky cliffs">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1018/1200/600" alt="Green hills and sea cliffs under a cloudy sky">
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/1016/1200/600" alt="Red desert canyon lit by the setting sun">
      </li>
    </ul>
  </div>
</section>
```

Images are usually smaller or larger than the slides, so make each image take the slide's width:

```css
.splide__slide img {
  width: 100%;
  height: auto;
}
```

Initialize Splide only once the DOM is ready. If the script runs in `<head>`, wait for `DOMContentLoaded`:

```html
<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    new Splide( '#image-carousel' ).mount();
  } );
</script>
```

If the script sits at the end of `<body>`, after the markup and after the Splide `<script>` tag, you can mount directly:

```html
<script>
  new Splide( '#image-carousel' ).mount();
</script>
```

If the Splide `<script>` tag has `defer`, an inline init script runs before it and throws `Splide is not defined`. In that case, always wrap the init in `DOMContentLoaded`.

## 2. Cropping images

Source images often have different aspect ratios. It is better to resize or crop them beforehand, with [sharp](https://sharp.pixelplumbing.com/) or an image editor. When that is not possible, crop them on the client side.

### `object-fit` (recommended)

All modern browsers support `object-fit`, which decides how an image fits its parent element. (Internet Explorer does not support it, and IE is end-of-life.)

```css
.splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

For this CSS to work, each slide needs an explicit height. There are several ways to set one. The tutorial uses the `heightRatio` option:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#image-carousel', {
    heightRatio: 0.5,
  } ).mount();
} );
```

With `heightRatio`, Splide sets the slide height to the carousel (list) width multiplied by the ratio, not the slide width. With one slide per page, a 1000px-wide slide becomes 500px tall (`1000 * 0.5`). With `perPage: 2` the slides are 500px wide and still 500px tall. The images then cover their slides. The other ways to give slides a height are `height` (a fixed CSS length, not `%`) and `fixedHeight`.

Verified: in the 4.x source (`Layout`), `heightRatio` is calculated from the width of the list element and applied to every slide (or to `.splide__slide__container` if there is one). It is recalculated on resize.

### `cover` option (legacy)

If you still need to support IE, the `cover` option turns each slide's `<img>` into the slide's background image (`background: center/cover no-repeat url(...)`) and hides the `<img>`:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#image-carousel', {
    cover      : true,
    heightRatio: 0.5,
  } ).mount();
} );
```

Always set `height`, `heightRatio` or `fixedHeight` along with it, so the slide has an explicit height. The author said he plans to deprecate `cover` once IE is retired. It still exists and works in 4.1.4. Prefer `object-fit` in new code.

Source detail (`components/Cover`): only an `<img>` that is a **direct child** of the slide (or of `.splide__slide__container`) is used. The background is applied to the container when one exists. Cover also re-runs after a lazy-loaded image finishes loading.

## 3. Images and text (card carousel)

Slides can contain any content, so a card carousel is just an image plus a text block:

```html
<section id="card-carousel" class="splide" aria-label="Beautiful Images">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">
        <img src="https://picsum.photos/id/10/800/500" alt="Evergreen forest on a hillside above a blue sea">
        <div>
          Description 01
        </div>
      </li>
      <li class="splide__slide">
        <img src="https://picsum.photos/id/15/800/500" alt="Waterfall pouring into a rocky gorge">
        <div>
          Description 02
        </div>
      </li>
    </ul>
  </div>
</section>
```

Correction: the tutorial's snippet gives this section `id="image-carousel"` but mounts `#card-carousel`, and it closes the `<section>` with `</div>`. Use one id for both (as above) and close with `</section>`.

Show 2 cards on desktop and 1 on mobile:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#card-carousel', {
    perPage    : 2,
    breakpoints: {
      640: {
        perPage: 1,
      },
    },
  } ).mount();
} );
```

`breakpoints` keys are max-widths by default (`mediaQuery: 'max'`), so `640` applies at viewport widths of 640px and below.

To use the `cover` option in a card carousel, wrap the image in a slide container. Otherwise the whole card, text included, becomes the background box. The container must be a direct child of the slide. Splide applies `height`/`heightRatio`/`fixedHeight` to the container instead of the slide (see the structure guide, https://splidejs.com/guides/structure/#slide-container):

```html
<li class="splide__slide">
  <div class="splide__slide__container">
    <img src="https://picsum.photos/id/10/800/500" alt="Evergreen forest on a hillside above a blue sea">
  </div>
  <div>
    Description 01
  </div>
</li>
```

## 4. Fullscreen carousel

The `width` and `height` options accept CSS relative units, so `'100vw'` and `'100vh'` make the carousel fill the screen:

```js
document.addEventListener( 'DOMContentLoaded', function () {
  new Splide( '#fullscreen-carousel', {
    width : '100vw',
    height: '100vh',
  } ).mount();
} );
```

`width` is applied as `max-width` on the root, and `height` is applied to every slide. Combine it with the `object-fit: cover` CSS (or `cover: true`) so the images fill the slides. See the fullscreen variation below.

## Complete page

Contains the cropped image carousel (`object-fit` + `heightRatio`) and the card carousel (2 per page, 1 at 640px and below). Save it as `.html` and open it in a browser.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Image Carousel - Splide</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <style>
    body {
      margin: 0;
      padding: 2rem 1rem;
      font-family: system-ui, sans-serif;
    }

    .demo {
      max-width: 960px;
      margin: 0 auto 3rem;
    }

    /* Image carousel: crop every image to the slide box (slide height comes from heightRatio) */
    #image-carousel .splide__slide img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      display: block;
    }

    /* Card carousel: image fills the card width, text block below */
    #card-carousel .splide__slide img {
      width: 100%;
      height: auto;
      display: block;
    }

    #card-carousel .splide__slide div {
      padding: 0.75rem 0;
      line-height: 1.5;
    }
  </style>
</head>
<body>
  <!-- 1. Image carousel with client-side cropping (object-fit + heightRatio) -->
  <section id="image-carousel" class="splide demo" aria-label="Mountain and coast landscapes">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1015/1200/600" alt="Deep blue fjord between steep rocky cliffs">
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1018/1200/600" alt="Green hills and sea cliffs under a cloudy sky">
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/1016/1200/600" alt="Red desert canyon lit by the setting sun">
        </li>
      </ul>
    </div>
  </section>

  <!-- 2. Card carousel: image + text, 2 per page, 1 per page at 640px and below -->
  <section id="card-carousel" class="splide demo" aria-label="Forest and waterfall cards">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide">
          <img src="https://picsum.photos/id/10/800/500" alt="Evergreen forest on a hillside above a blue sea">
          <div>
            Description 01: an evergreen forest running down to the sea.
          </div>
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/15/800/500" alt="Waterfall pouring into a rocky gorge">
          <div>
            Description 02: a waterfall dropping into a rocky gorge.
          </div>
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/17/800/500" alt="Gravel path through a green meadow toward tall pine trees">
          <div>
            Description 03: a gravel path leading into the pines.
          </div>
        </li>
        <li class="splide__slide">
          <img src="https://picsum.photos/id/28/800/500" alt="Dense green forest around a rocky stream bed">
          <div>
            Description 04: a rocky stream bed in a green forest.
          </div>
        </li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script>
    // Scripts sit at the end of <body>, so the DOM is ready and DOMContentLoaded is not needed.
    new Splide( '#image-carousel', {
      heightRatio: 0.5, // slide height = slide width * 0.5
    } ).mount();

    new Splide( '#card-carousel', {
      perPage    : 2,
      gap        : '1rem', // not in the tutorial; keeps the two cards apart
      breakpoints: {
        640: {
          perPage: 1,
        },
      },
    } ).mount();
  </script>
</body>
</html>
```

Verified in headless Chrome against 4.1.4 from the CDN. At a 1200px window, the image carousel slides are 960x480 and the card carousel shows 2 cards (2 pages). At 600px it shows 1 card per page (4 pages). There are no console errors.

## Variations

- **Different crop ratio:** change `heightRatio` (for example `0.5625` for 16:9). To give slides a fixed height instead, use `height: '400px'`, or `fixedHeight` when combined with `fixedWidth`. `height` does not accept `%`.
- **Per-breakpoint height:** `breakpoints: { 640: { heightRatio: 0.75 } }` makes slides taller on phones.
- **Lazy loading:** put the URL in `data-splide-lazy` (and `data-splide-lazy-srcset` for `srcset`) and leave out `src`. Then set `lazyLoad: 'nearby'` (loads the visible slides plus `preloadPages`, default 1, pages around them) or `lazyLoad: 'sequential'` (loads one image after another). Splide adds a `.splide__spinner` while each image loads. The core CSS styles it, and every theme includes the core CSS. Give slides a height (`heightRatio`, `height`, `fixedHeight`) so they do not collapse before loading. Keep `alt` on the `<img>`.
  ```html
  <li class="splide__slide">
    <img data-splide-lazy="https://picsum.photos/id/1015/1200/600" alt="Deep blue fjord between steep rocky cliffs">
  </li>
  ```
  ```js
  new Splide( '#image-carousel', { heightRatio: 0.5, lazyLoad: 'nearby' } ).mount();
  ```
- **Responsive images:** the normal `srcset`/`sizes` attributes work on slide images. With `lazyLoad`, use `data-splide-lazy-srcset`.
- **Card carousel spacing and peeking:** `gap: '1rem'` separates the cards. `padding: { left: '2rem', right: '2rem' }` (or `padding: '10%'`) shows part of the neighbouring cards. Both go in `breakpoints` too.
- **Mobile-first breakpoints:** `mediaQuery: 'min'` with `perPage: 1, breakpoints: { 641: { perPage: 2 } }` gives the same result, written mobile-first. Use `641`, not `640`: `(min-width: 640px)` also matches at exactly 640px, so that width would show 2 cards instead of 1.
- **Looping and autoplay:** `type: 'loop'` (`rewind` has no effect in loop), `autoplay: true`, `interval: 5000`. Add a play/pause toggle when autoplaying (see `references/autoplay.md`).
- **Fullscreen page:** remove the body margin and crop the images:
  ```html
  <style>
    body { margin: 0; }
    #fullscreen-carousel .splide__slide img { width: 100%; height: 100%; object-fit: cover; display: block; }
  </style>
  <section id="fullscreen-carousel" class="splide" aria-label="Fullscreen landscapes">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide"><img src="https://picsum.photos/id/1015/1920/1080" alt="Deep blue fjord between steep rocky cliffs"></li>
        <li class="splide__slide"><img src="https://picsum.photos/id/1018/1920/1080" alt="Green hills and sea cliffs under a cloudy sky"></li>
      </ul>
    </div>
  </section>
  <script>
    new Splide( '#fullscreen-carousel', { width: '100vw', height: '100vh' } ).mount();
  </script>
  ```
  `100vw` includes the vertical scrollbar width. If the page scrolls, use `width: '100%'` to avoid a horizontal scrollbar.
- **Accessibility labels:** label the carousel with `aria-label` on the root or the `label` option (`labelledby` for `aria-labelledby`). Splide gives each slide `role="tabpanel"` with pagination (`"group"` without it), `aria-roledescription="slide"` and `aria-label="1 of 3"`. If a slide already has an `aria-label`, Splide keeps it. Translate the generated strings with the `i18n` option (`slideLabel: '%s of %s'`, `prev`, `next`, `pageX`, ...). Decorative images get `alt=""`. Informative images need real alt text.
- **Text-only overlay captions:** with `object-fit` (not `cover`), put an absolutely positioned caption inside the slide (`.splide__slide { position: relative }` is already set by the core CSS).
