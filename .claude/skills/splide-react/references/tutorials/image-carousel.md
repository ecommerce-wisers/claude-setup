# Image carousel

Source: https://splidejs.com/tutorials/image-carousel/

The image carousel tutorial converted to React: a typical image carousel, with and without text, and two ways to normalize image dimensions (CSS `object-fit` + `heightRatio`, or the legacy `cover` option). It also covers a card carousel and a fullscreen carousel. Every component here was run in headless Chrome with React 19 and `@splidejs/react-splide` 0.7.12, in production and in development with StrictMode.

Contents:
- Install and imports
- 1. Basic image carousel
- 2. Cropping images (`object-fit`, `cover`)
- 3. Images and text (card carousel)
- 4. Fullscreen carousel
- Complete component
- Variations (lazy loading, breakpoints, labels, captions, reactivity)

## Install and imports

```bash
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

Importing the CSS once (in the app entry or root layout) is enough. The components below import it themselves so that each file works on its own. React Splide's types need one tsconfig line under the default Vite and Next.js module resolution, see `references/typescript.md`. In the Next.js App Router, a file that renders `<Splide>` must be a client component (`'use client'`), see `references/nextjs.md`.

## 1. Basic image carousel

`<Splide>` renders the root `.splide` with `.splide__track` > `.splide__list` inside it, and each `<SplideSlide>` renders an `li.splide__slide`. Put one image in each slide. Describe the carousel with a label and every image with `alt`, because screen reader users depend on them. The tutorial uses `image01.jpg` and an empty `alt` as placeholders. The snippets below fill in real URLs and alt text.

```tsx
<Splide tag="section" options={ { label: 'Beautiful Images' } }>
  <SplideSlide>
    <img src="https://picsum.photos/id/1015/1200/600" alt="Deep blue fjord between steep rocky cliffs" />
  </SplideSlide>
  <SplideSlide>
    <img src="https://picsum.photos/id/1018/1200/600" alt="Green hills and sea cliffs under a cloudy sky" />
  </SplideSlide>
  <SplideSlide>
    <img src="https://picsum.photos/id/1016/1200/600" alt="Red desert canyon lit by the setting sun" />
  </SplideSlide>
</Splide>
```

Label the carousel with `options.label` (or `options.labelledby`), not an `aria-label` prop. In development, StrictMode mounts, destroys and remounts the carousel, and Splide's `destroy()` removes the `aria-label` attribute that React rendered. React does not put it back. `options.label` is applied again on every mount. Production behaves the same with either, but the option is the safe default.

`tag="section"` renders a `<section>`. Splide adds `role="region"` only when the root is not a `<section>` (`components/Elements/Elements.ts:211`), because a labelled `<section>` is already a region. It always adds `aria-roledescription="carousel"`.

Images are usually smaller or larger than the slides, so make each image take the slide's width:

```css
.splide__slide img {
  width: 100%;
  height: auto;
}
```

There is no "wait for the DOM" step in React. React Splide creates the core instance and calls `mount()` in `componentDidMount`, after the elements exist. Until then the core CSS keeps `.splide { visibility: hidden }`, so server-rendered markup stays invisible until the component mounts on the client.

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

```tsx
const options: Options = {
  label      : 'Beautiful Images',
  heightRatio: 0.5,
};

<Splide tag="section" options={ options }>...</Splide>
```

With `heightRatio`, Splide sets the slide height to the carousel (list) width multiplied by the ratio, not the slide width. With one slide per page, a 1000px-wide slide becomes 500px tall (`1000 * 0.5`). With `perPage: 2` the slides are 500px wide and still 500px tall. The images then cover their slides. The other ways to give slides a height are `height` (a fixed CSS length, not `%`) and `fixedHeight`.

Verified: in the 4.x source (`Layout`), `heightRatio` is calculated from the width of the list element and applied to every slide (or to `.splide__slide__container` if there is one). It is recalculated on resize. In the browser, a 960px carousel with `perPage: 2` and `heightRatio: 0.5` has 480x480 slides.

### `cover` option (legacy)

If you still need to support IE, the `cover` option turns each slide's `<img>` into the slide's background image (`background: center/cover no-repeat url(...)`) and hides the `<img>`:

```tsx
const options: Options = {
  label      : 'Beautiful Images',
  cover      : true,
  heightRatio: 0.5,
};
```

Always set `height`, `heightRatio` or `fixedHeight` along with it, so the slide has an explicit height. The author said he plans to deprecate `cover` once IE is retired. It still exists and works in 4.1.4 (the core version React Splide bundles behaves the same). Prefer `object-fit` in new code.

Source detail (`components/Cover`): only an `<img>` that is a **direct child** of the slide (or of `.splide__slide__container`) is used. The background is applied to the container when one exists. Cover also re-runs after a lazy-loaded image finishes loading.

## 3. Images and text (card carousel)

Slides can contain any content, so a card carousel is just an image plus a text block:

```tsx
<SplideSlide>
  <img src="https://picsum.photos/id/10/800/500" alt="Evergreen forest on a hillside above a blue sea" />
  <div>Description 01</div>
</SplideSlide>
```

Show 2 cards on desktop and 1 on mobile:

```tsx
const cardOptions: Options = {
  label      : 'Forest and waterfall cards',
  perPage    : 2,
  breakpoints: {
    640: {
      perPage: 1,
    },
  },
};
```

`breakpoints` keys are max-widths by default (`mediaQuery: 'max'`), so `640` applies at viewport widths of 640px and below.

To use the `cover` option in a card carousel, wrap the image in a slide container. Otherwise the whole card, text included, becomes the background box. The container must be a direct child of the slide. Splide applies `height`/`heightRatio`/`fixedHeight` to the container instead of the slide (see the structure guide, https://splidejs.com/guides/structure/#slide-container). Tested component:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './CoverCardCarousel.css';

const cards = [
  { src: 'https://picsum.photos/id/10/800/500', alt: 'Evergreen forest on a hillside above a blue sea', text: 'Description 01' },
  { src: 'https://picsum.photos/id/15/800/600', alt: 'Waterfall pouring into a rocky gorge', text: 'Description 02' },
  { src: 'https://picsum.photos/id/17/800/400', alt: 'Gravel path through a green meadow toward tall pine trees', text: 'Description 03' },
];

const options: Options = {
  label      : 'Forest and waterfall cards',
  perPage    : 2,
  gap        : '1rem',
  cover      : true, // legacy: prefer object-fit in new code
  heightRatio: 0.3, // applied to .splide__slide__container, not the whole card
  breakpoints: {
    640: {
      perPage    : 1,
      heightRatio: 0.6,
    },
  },
};

export default function CoverCardCarousel() {
  return (
    <Splide tag="section" className="cover-card-carousel" options={ options }>
      { cards.map( card => (
        <SplideSlide key={ card.src }>
          <div className="splide__slide__container">
            <img src={ card.src } alt={ card.alt } />
          </div>
          <div>{ card.text }</div>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

```css
/* CoverCardCarousel.css */
.cover-card-carousel {
  max-width: 960px;
  margin: 2rem auto;
}

.cover-card-carousel .splide__slide__container + div {
  padding: 0.75rem 0;
}
```

Verified: every `.splide__slide__container` gets the background and a height of `0.3 * 960 = 288px` (472px wide with the 1rem gap), the `<img>` gets `display: none`, and the slide itself gets neither.

## 4. Fullscreen carousel

The `width` and `height` options accept CSS relative units, so `'100vw'` and `'100vh'` make the carousel fill the screen:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './FullscreenCarousel.css';

const options: Options = {
  label : 'Fullscreen landscapes',
  width : '100vw',
  height: '100vh',
};

export default function FullscreenCarousel() {
  return (
    <Splide tag="section" className="fullscreen-carousel" options={ options }>
      <SplideSlide>
        <img src="https://picsum.photos/id/1015/1920/1080" alt="Deep blue fjord between steep rocky cliffs" />
      </SplideSlide>
      <SplideSlide>
        <img src="https://picsum.photos/id/1018/1920/1080" alt="Green hills and sea cliffs under a cloudy sky" />
      </SplideSlide>
    </Splide>
  );
}
```

```css
/* FullscreenCarousel.css */
body {
  margin: 0;
}

.fullscreen-carousel .splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}
```

`width` is applied as `max-width` on the root, and `height` is applied to every slide. Combine it with the `object-fit: cover` CSS (or `cover: true`) so the images fill the slides. Verified: in a 1280x800 window the slides and images are 1280x800. `100vw` includes the vertical scrollbar width. If the page scrolls, use `width: '100%'` to avoid a horizontal scrollbar.

## Complete component

Contains the cropped image carousel (`object-fit` + `heightRatio`) and the card carousel (2 per page, 1 at 640px and below). Save the two files next to each other and render `<ImageCarouselPage />`.

```tsx
// ImageCarousel.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './ImageCarousel.css';

const images = [
  { src: 'https://picsum.photos/id/1015/1200/600', alt: 'Deep blue fjord between steep rocky cliffs' },
  { src: 'https://picsum.photos/id/1018/1200/600', alt: 'Green hills and sea cliffs under a cloudy sky' },
  { src: 'https://picsum.photos/id/1016/1200/600', alt: 'Red desert canyon lit by the setting sun' },
];

const cards = [
  { src: 'https://picsum.photos/id/10/800/500', alt: 'Evergreen forest on a hillside above a blue sea', text: 'Description 01: an evergreen forest running down to the sea.' },
  { src: 'https://picsum.photos/id/15/800/500', alt: 'Waterfall pouring into a rocky gorge', text: 'Description 02: a waterfall dropping into a rocky gorge.' },
  { src: 'https://picsum.photos/id/17/800/500', alt: 'Gravel path through a green meadow toward tall pine trees', text: 'Description 03: a gravel path leading into the pines.' },
  { src: 'https://picsum.photos/id/28/800/500', alt: 'Dense green forest around a rocky stream bed', text: 'Description 04: a rocky stream bed in a green forest.' },
];

// Module-level options: the same object on every render, so React Splide never sees a change.
const imageOptions: Options = {
  label      : 'Mountain and coast landscapes', // options.label survives StrictMode, an aria-label prop does not
  heightRatio: 0.5, // slide height = carousel width * 0.5
};

const cardOptions: Options = {
  label      : 'Forest and waterfall cards',
  perPage    : 2,
  gap        : '1rem', // not in the tutorial; keeps the two cards apart
  breakpoints: {
    640: {
      perPage: 1,
    },
  },
};

// 1. Image carousel with client-side cropping (object-fit + heightRatio)
export function ImageCarousel() {
  return (
    <Splide tag="section" className="demo image-carousel" options={ imageOptions }>
      { images.map( image => (
        <SplideSlide key={ image.src }>
          <img src={ image.src } alt={ image.alt } />
        </SplideSlide>
      ) ) }
    </Splide>
  );
}

// 2. Card carousel: image + text, 2 per page, 1 per page at 640px and below
export function CardCarousel() {
  return (
    <Splide tag="section" className="demo card-carousel" options={ cardOptions }>
      { cards.map( card => (
        <SplideSlide key={ card.src }>
          <img src={ card.src } alt={ card.alt } />
          <div>{ card.text }</div>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}

export default function ImageCarouselPage() {
  return (
    <>
      <ImageCarousel />
      <CardCarousel />
    </>
  );
}
```

```css
/* ImageCarousel.css */
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
.image-carousel .splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

/* Card carousel: image fills the card width, text block below */
.card-carousel .splide__slide img {
  width: 100%;
  height: auto;
  display: block;
}

.card-carousel .splide__slide div {
  padding: 0.75rem 0;
  line-height: 1.5;
}
```

Verified in headless Chrome (production and development + StrictMode). At a 1280px window, the image carousel slides are 960x480, the root is a `<section>` with `aria-label="Mountain and coast landscapes"` and `aria-roledescription="carousel"`, and the first slide has `role="tabpanel"`, `aria-roledescription="slide"` and `aria-label="1 of 3"`. The next arrow moves to slide 2. The card carousel shows 2 cards (2 pages). At 600px it shows 1 card per page (4 pages) and the image slides are 568x284. There are no console errors or warnings.

The option objects live at module level. React Splide compares `options` deeply on every update and only calls `splide.options = ...` when the content changes, so an inline `options={ { ... } }` object also works. Functions inside options are compared by identity and do cause an update on every render. See `references/reactivity.md`.

## Variations

- **Different crop ratio:** change `heightRatio` (for example `0.5625` for 16:9). To give slides a fixed height instead, use `height: '400px'`, or `fixedHeight` when combined with `fixedWidth`. `height` does not accept `%`.
- **Per-breakpoint height:** `breakpoints: { 640: { heightRatio: 0.75 } }` makes slides taller on phones.
- **Lazy loading:** put the URL in `data-splide-lazy` (and `data-splide-lazy-srcset` for `srcset`) and leave out `src`. Then set `lazyLoad: 'nearby'` (loads the visible slides plus `preloadPages`, default 1, pages around them) or `lazyLoad: 'sequential'` (loads one image after another). Splide adds a `.splide__spinner` while each image loads. The core CSS styles it, and every theme includes the core CSS. Give slides a height (`heightRatio`, `height`, `fixedHeight`) so they do not collapse before loading. Keep `alt` on the `<img>`. `data-*` attributes pass straight through JSX. The event prop is `onLazyLoadLoaded` (capital L in Load), called as `( splide, img, Slide )`. `onLazyloadLoaded` does not exist and React ignores it.
  ```tsx
  import { Splide, SplideSlide } from '@splidejs/react-splide';
  import type { Options } from '@splidejs/react-splide';
  import '@splidejs/react-splide/css';
  import './ImageCarousel.css';

  const images = [
    { src: 'https://picsum.photos/id/1015/1200/600', alt: 'Deep blue fjord between steep rocky cliffs' },
    { src: 'https://picsum.photos/id/1018/1200/600', alt: 'Green hills and sea cliffs under a cloudy sky' },
    { src: 'https://picsum.photos/id/1016/1200/600', alt: 'Red desert canyon lit by the setting sun' },
    { src: 'https://picsum.photos/id/1036/1200/600', alt: 'Snow-covered campsite below jagged mountains' },
  ];

  const options: Options = {
    label      : 'Mountain and coast landscapes',
    heightRatio: 0.5,
    lazyLoad   : 'nearby',
  };

  export default function LazyImageCarousel() {
    return (
      <Splide
        tag="section"
        className="demo image-carousel"
        options={ options }
        onLazyLoadLoaded={ ( splide, img ) => console.log( 'loaded', img.src ) }
      >
        { images.map( image => (
          <SplideSlide key={ image.src }>
            <img data-splide-lazy={ image.src } alt={ image.alt } />
          </SplideSlide>
        ) ) }
      </Splide>
    );
  }
  ```
  Verified: with `perPage: 1` only slides 1 and 2 load at first. Slides 3 and 4 keep `data-splide-lazy`, a hidden `<img>` and a spinner until the carousel moves, then load and fire `onLazyLoadLoaded`. Development-only issue: StrictMode destroys and remounts the carousel while the first images are still loading. Those images still appear, but their spinner stays on top of them and `onLazyLoadLoaded` does not fire for them (seen in every `--dev` run). Production is not affected.
- **Responsive images:** the normal `srcSet`/`sizes` props work on slide images. With `lazyLoad`, use `data-splide-lazy-srcset`.
- **Card carousel spacing and peeking:** `gap: '1rem'` separates the cards. `padding: { left: '2rem', right: '2rem' }` (or `padding: '10%'`) shows part of the neighbouring cards. Both go in `breakpoints` too.
- **Mobile-first breakpoints:** `mediaQuery: 'min'` with `perPage: 1, breakpoints: { 641: { perPage: 2 } }` gives the same result, written mobile-first. Use `641`, not `640`: `(min-width: 640px)` also matches at exactly 640px, so that width would show 2 cards instead of 1.
- **Looping and autoplay:** `type: 'loop'` (`rewind` has no effect in loop), `autoplay: true`, `interval: 5000`. Add a play/pause toggle when autoplaying (see `references/autoplay.md`).
- **Accessibility labels:** label the carousel with the `label` option (`labelledby` for `aria-labelledby`). Splide gives each slide `role="tabpanel"` with pagination (`"group"` without it), `aria-roledescription="slide"` and `aria-label="1 of 3"`. If a slide already has an `aria-label` (`<SplideSlide aria-label="...">`), Splide keeps it. Translate the generated strings with the `i18n` option (`slideLabel: '%s of %s'`, `prev`, `next`, `pageX`, ...). Decorative images get `alt=""`. Informative images need real alt text.
- **Text-only overlay captions:** with `object-fit` (not `cover`), put an absolutely positioned caption inside the slide (`.splide__slide { position: relative }` is already set by the core CSS).
- **Slides from data:** render the slides with `map()` and a stable `key`. When the list of slides changes, React Splide calls `splide.refresh()` after the update, so added and removed slides are picked up without remounting.
