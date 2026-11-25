# Thumbnail carousel

Source: https://splidejs.com/tutorials/thumbnail-carousel/

The thumbnail carousel tutorial converted to React: a main carousel controlled by a carousel of clickable thumbnails. It uses two Splide features: `sync()` links two carousels so that moving one moves the other, and the `isNavigation` option makes a carousel's slides clickable (and keyboard-operable) navigation buttons. (The tutorial intro spells it `isNaviation`. The option is `isNavigation`.) In React the two carousels are linked from a `useEffect`, after both have mounted. Every component here was run in headless Chrome with React 19 and `@splidejs/react-splide` 0.7.12, in production and in development with StrictMode.

For a gallery with plain (non-Splide) thumbnails driven by `go()` and events, see `references/tutorials/gallery.md`.

Contents:
- Install and imports
- 1. Thumbnail carousel markup
- 2. Fixed slide width instead of `perPage`
- 3. Fit the images to the thumbnails
- 4. Active state, mobile size, centering
- 5. Make the thumbnails clickable: `isNavigation`
- 6. Main carousel
- 7. Sync in React (timing, what `isNavigation` does, refresh issue, labelling)
- Complete component
- Variations (vertical thumbnails, dynamic slides, labels, several galleries)

## Install and imports

```bash
npm install @splidejs/react-splide
```

```tsx
import { useEffect, useRef } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

Types need one tsconfig line under the default Vite and Next.js module resolution, see `references/typescript.md`. In the Next.js App Router the component must be a client component, see `references/nextjs.md`.

## 1. Thumbnail carousel markup

Start with the thumbnail carousel. Its label should say what selecting a thumbnail does and which carousel it controls. Pass it as `options.label`: an `aria-label` prop is removed in development by StrictMode's destroy and remount, `options.label` is not.

```tsx
const thumbsOptions: Options = {
  label: 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
};

<Splide tag="section" className="thumbnail-carousel" options={ thumbsOptions }>
  <SplideSlide>
    <img src="https://picsum.photos/id/1015/200/120" alt="Fjord between steep cliffs" />
  </SplideSlide>
  <SplideSlide>
    <img src="https://picsum.photos/id/1018/200/120" alt="Green hills and sea cliffs" />
  </SplideSlide>
  <SplideSlide>
    <img src="https://picsum.photos/id/1016/200/120" alt="Red desert canyon at sunset" />
  </SplideSlide>
</Splide>
```

(The tutorial uses `thumbnail01.jpg` with an empty `alt` as placeholders.)

## 2. Fixed slide width instead of `perPage`

A thumbnail carousel shows several slides per page. `perPage` is hard to tune for every screen size, even with breakpoints, so fix each slide's width with `fixedWidth`. The number of visible thumbnails then follows the window size:

```tsx
const thumbsOptions: Options = {
  label     : 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
  fixedWidth: 100,
  gap       : 10,
  rewind    : true,
  pagination: false,
};
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

If not, crop each thumbnail with `object-fit`. Internet Explorer does not support it. For it to work, the slides need an explicit height, so add `fixedHeight: 60` to the options, then apply the property to the thumbnails:

```css
.splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

When the page has a main carousel too, scope these rules (`.thumbnail-carousel .splide__slide img`) so they don't also crop the main images.

## 4. Active state, mobile size, centering

At this point:

- the user cannot tell which slide is selected,
- `100px` is probably too wide for mobile devices,
- the active slide should be centered when possible.

**Active-thumbnail CSS.** A theme CSS (`@splidejs/react-splide/css`, `/css/skyblue`, `/css/sea-green`) already emphasizes the active navigation slide with a border. These rules are in the 4.1.x default theme, applied through the `splide__track--nav` class that Splide adds to the track when `isNavigation` is on (the skyblue and sea-green themes use their own colour and also dim inactive thumbnails to 0.7):

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

Without a theme (`@splidejs/react-splide/css/core` only), style the `is-active` status class yourself:

```css
.thumbnail-carousel .splide__slide {
  opacity: 0.6;
}

.thumbnail-carousel .splide__slide.is-active {
  opacity: 1;
}
```

Splide toggles `is-active` (and `aria-current`) itself on every move, so this CSS is all you need. No React state is involved.

**Size and centering.** `breakpoints` handles the second problem and `focus: 'center'` handles the third. If you are not using `object-fit`, `fixedHeight` is not necessary.

```tsx
const thumbsOptions: Options = {
  label      : 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
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
};
```

`breakpoints` keys are max-widths by default (`mediaQuery: 'max'`), so the 60x44 size applies at 600px and below. With `type: 'slide'` and the default `trimSpace: true`, `focus: 'center'` centers the active thumbnail without leaving empty space at either end.

## 5. Make the thumbnails clickable: `isNavigation`

Add `isNavigation: true` to the thumbnail options (the complete component below has the full object).

The thumbnail carousel must **not** have pagination. Otherwise the accessibility tree gets confusing: thumbnails are buttons that control another carousel, and pagination would add a second set of tab controls. Keep `pagination: false`.

To ignore small mouse drags, so that a slightly shaky click still counts as a click, use `dragMinThreshold`. A number only sets the touch threshold (the mouse threshold stays 0). Pass an object to set the mouse threshold: `dragMinThreshold: { mouse: 4, touch: 10 }`.

Correction: the tutorial's step 4 code includes `focus: 'center'`, but its step 5 code and its final code leave it out. The demo on the tutorial page uses it. Add it back if you want the active thumbnail centered. The complete component below includes it.

## 6. Main carousel

The tutorial uses a fade carousel as the main carousel. Give it its own label. The thumbnail carousel's label refers to it by name ("Beautiful Gallery").

```tsx
const mainOptions: Options = {
  label     : 'Beautiful Gallery',
  type      : 'fade',
  rewind    : true,
  pagination: false,
  arrows    : false,
};
```

This disables arrows and pagination. To enable them, delete those two options. `fade` cannot loop, so `rewind: true` lets it wrap from the last slide to the first. (The tutorial's demo also used `heightRatio: 0.5625` and `cover: true`. They are not in its code.)

## 7. Sync in React

Give each `<Splide>` a ref and call `sync()` once, in an effect:

```tsx
const mainRef   = useRef<Splide>( null );
const thumbsRef = useRef<Splide>( null );

useEffect( () => {
  const thumbs = thumbsRef.current?.splide;

  if ( mainRef.current && thumbs ) {
    mainRef.current.sync( thumbs );
  }
}, [] );
```

`mainRef.current.sync( splide )` is a method of the React component that calls the core `sync()` on its instance. Pass the other carousel's **core instance** (`thumbsRef.current.splide`), not the component. `mainRef.current.splide?.sync( thumbs )` does the same.

### Timing: why this works although `sync()` comes after `mount()`

In vanilla Splide the rule is "call `sync()` before any `mount()`, then mount the main carousel first". React cannot follow it: each `<Splide>` creates and mounts its core instance in its own `componentDidMount`, and children mount before the parent's effects run. So when the effect calls `sync()`, both carousels are already mounted.

It works anyway because `sync()` on a mounted instance remounts the Sync component of both carousels (`core/Splide/Splide.ts`, `sync()`: `if ( this.state.is( IDLE ) ) { this._C.Sync.remount(); splide.Components.Sync.remount(); }`). The remount re-applies the navigation setup, including `aria-controls`, which now finds the main carousel's slide IDs. The vanilla reference calls this path undocumented. It is also what the official React example does (`examples/src/js/components/ThumbnailsExample.tsx` in the react-splide repo, from `componentDidMount`). Verified in production and under StrictMode: thumbnails get `role="button"`, "Go to slide N" and `aria-controls`, and clicks and keys navigate.

Consequences:
- The order of the two `<Splide>` elements in the JSX does not matter. The vertical variation below renders the thumbnails first and was tested.
- Call `sync()` once per pair of instances. Every call adds another link. With an empty dependency array the effect runs once per mount. Under StrictMode, React destroys and recreates both core instances before the effect runs again, so each fresh instance is linked once (tested: `splides.length` is 1 on both).
- There is no `unsync()`. Unmounting a `<Splide>` calls `destroy()`, which removes its links.
- Don't put `sync()` in an `onMounted` handler of one carousel: the other carousel may not exist yet.

How syncing works (`components/Sync`): each instance listens to the other's `move` event and calls `go()` on it. A loop target receives the destination index, and any other target receives the normalized index. Syncing is two-way, so dragging the main carousel also moves the thumbnails. More than two carousels can be chained by calling `sync()` several times on the same primary.

### What `isNavigation` does

Verified from `components/Sync`, `components/Slides/Slide.ts` and `components/Elements`, and confirmed in the browser:

- The root gets `splide--nav` and the track gets `splide__track--nav`. The theme's active-border CSS hooks on that class.
- Clicking a slide calls `go( slide.index )`. Pressing **Enter** or **Space** on a focused slide does the same.
- `slideFocus` defaults to `true` when `isNavigation` is on. Every thumbnail slide then gets `role="button"` and `tabindex="0"` (`-1` while hidden), and `aria-roledescription` is removed.
- Each thumbnail gets `aria-label="Go to slide N"` (from `i18n.slideX`) and `aria-controls` with the matching main slide ID. Splide generates the main slide IDs as `<root id>-slide01`, `-slide02`, and so on. Without an `id` prop, the root ID is generated too (`splide01`, `splide02`, ...).
- The active thumbnail gets the `is-active` class and `aria-current="true"`.
- With `direction: 'ttb'`, the list gets `aria-orientation="vertical"`.
- The main carousel's slides get `role="group"` (because `pagination: false`), `aria-roledescription="slide"` and `aria-label="1 of 6"`.

### Known issue: `refresh()` drops the navigation ARIA

In 4.1.x, `refresh()`, `add()`, `remove()` and a breakpoint that changes `direction` all rebuild the slide components. Splide does not re-run the navigation setup afterwards, so the thumbnails fall back to `role="group"` and `aria-label="1 of 6"` and lose `aria-controls`. Click, Enter/Space, `is-active` and `aria-current` keep working.

This matters more in React: **React Splide calls `refresh()` whenever the rendered slides change** (`componentDidUpdate` compares the list's children). Adding or removing a thumbnail through state therefore triggers the issue. Workaround: re-emit the navigation event after every refresh, from a handler registered **after** mount, so that it runs after Splide has rebuilt the slides. The effect that calls `sync()` is the right place:

```tsx
useEffect( () => {
  const thumbs = thumbsRef.current?.splide;

  if ( ! mainRef.current || ! thumbs ) {
    return;
  }

  mainRef.current.sync( thumbs );

  thumbs.on( 'refresh.nav', () => {
    thumbs.emit( 'navigation:mounted', thumbs.splides );
  } );

  return () => {
    thumbs.off( 'refresh.nav' );
  };
}, [] );
```

Tested with a component that adds a slide through state (full code under Variations): without the handler, all four thumbnails become `group` / "1 of 4" with no `aria-controls`; with it, they keep `button` / "Go to slide N" / `aria-controls`. An `onRefresh` prop does **not** work: event props are bound before `mount()`, so the prop runs before Splide's own refresh listener rebuilds the slides (tested: same broken result as no handler). The `.nav` namespace lets the cleanup remove only this listener. A plain `thumbs.off( 'refresh' )` would also remove an `onRefresh` prop handler, because React Splide binds props with the same `splide.on()`.

Inside Splide, only the slide components listen to `navigation:mounted` (verified in the source), so re-emitting it has no other side effects. An `onNavigationMounted` prop will run again, though.

### Thumbnail a11y labelling

- Give the thumbnail carousel a descriptive `label` that says what it controls, for example: "The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel."
- Give the main carousel a `label` (or `labelledby`), for example "Beautiful Gallery".
- Splide **overwrites** any `aria-label` you put on navigation slides with `i18n.slideX`. To change the wording, set it in the thumbnail options: `i18n: { slideX: 'Show image %s' }`. The thumbnail images' `alt` text is not used as the button name, because the slide's `aria-label` takes precedence. Keep a short alt anyway.
- Keep `pagination: false` on the thumbnail carousel.

## Complete component

Save the two files next to each other and render `<ThumbnailCarousel />`.

```tsx
// ThumbnailCarousel.tsx
import { useEffect, useRef } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './ThumbnailCarousel.css';

const images = [
  { id: 1015, alt: 'Deep blue fjord between steep rocky cliffs', thumbAlt: 'Fjord between steep cliffs' },
  { id: 1018, alt: 'Green hills and sea cliffs under a cloudy sky', thumbAlt: 'Green hills and sea cliffs' },
  { id: 1016, alt: 'Red desert canyon lit by the setting sun', thumbAlt: 'Red desert canyon at sunset' },
  { id: 1036, alt: 'Snow-covered campsite below jagged mountains', thumbAlt: 'Snowy campsite below mountains' },
  { id: 1050, alt: 'Turquoise sea around white chalk cliffs', thumbAlt: 'Chalk cliffs over a turquoise sea' },
  { id: 29, alt: 'Snow-covered mountain range under a blue sky', thumbAlt: 'Snow-covered mountain range' },
];

const mainOptions: Options = {
  label     : 'Beautiful Gallery',
  type      : 'fade',
  rewind    : true,
  pagination: false,
  arrows    : false,
};

const thumbsOptions: Options = {
  label       : 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
  fixedWidth  : 100,
  fixedHeight : 60,
  gap         : 10,
  rewind      : true,
  pagination  : false, // never give the navigation carousel pagination
  focus       : 'center', // shown in the tutorial's breakpoints step, dropped from its final code
  isNavigation: true,
  breakpoints : {
    600: {
      fixedWidth : 60,
      fixedHeight: 44,
    },
  },
};

export default function ThumbnailCarousel() {
  const mainRef   = useRef<Splide>( null );
  const thumbsRef = useRef<Splide>( null );

  useEffect( () => {
    // Both carousels are mounted by now (child componentDidMount runs before the parent's effects).
    const thumbs = thumbsRef.current?.splide;

    if ( mainRef.current && thumbs ) {
      mainRef.current.sync( thumbs );
    }
  }, [] );

  return (
    <div className="gallery">
      <Splide ref={ mainRef } tag="section" className="main-carousel" options={ mainOptions }>
        { images.map( image => (
          <SplideSlide key={ image.id }>
            <img src={ `https://picsum.photos/id/${ image.id }/1200/675` } alt={ image.alt } />
          </SplideSlide>
        ) ) }
      </Splide>

      <Splide ref={ thumbsRef } tag="section" className="thumbnail-carousel" options={ thumbsOptions }>
        { images.map( image => (
          <SplideSlide key={ image.id }>
            <img src={ `https://picsum.photos/id/${ image.id }/200/120` } alt={ image.thumbAlt } />
          </SplideSlide>
        ) ) }
      </Splide>
    </div>
  );
}
```

```css
/* ThumbnailCarousel.css */
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
.main-carousel .splide__slide img {
  width: 100%;
  height: auto;
  display: block;
}

.thumbnail-carousel {
  margin-top: 1rem;
}

/* Thumbnails: crop to the fixedWidth x fixedHeight box */
.thumbnail-carousel .splide__slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

/*
 * Active-thumbnail styling. The default theme already adds a 3px border to the active
 * navigation slide (.splide__track--nav > .splide__list > .splide__slide.is-active).
 * Without a theme (@splidejs/react-splide/css/core), style the is-active class yourself:
 */
.thumbnail-carousel .splide__slide {
  opacity: 0.6;
}

.thumbnail-carousel .splide__slide.is-active {
  opacity: 1;
}
```

Verified in headless Chrome (production and development + StrictMode). Thumbnails are 100x60 with `role="button"`, `tabindex="0"`, no `aria-roledescription`, `aria-label="Go to slide 1"`, `aria-controls` equal to the first main slide's ID (`...-slide01`) and `aria-current="true"` on the active one. Main slides have `role="group"` and `aria-label="1 of 6"`. Clicking thumbnail 4 moves the main carousel to index 3 and moves `aria-current` to thumbnail 4. `go( 5 )` on the main instance moves the thumbnails to 5. Enter on thumbnail 2 navigates. At 500px the thumbnails are 60x44 and keep their navigation ARIA. Each instance has exactly one sync link. There are no console errors or warnings.

## Variations

- **Main carousel controls:** delete `arrows: false` and/or `pagination: false` from the main options to get arrows and dots. Keep pagination off on the thumbnails.
- **Looping main carousel:** use `type: 'loop'` on the main instead of `'fade'` (drop `rewind`, which has no effect in loop). Syncing handles loop indexes. The thumbnails can stay `type: 'slide'` with `rewind: true`.
- **Cropped main images:** add `heightRatio: 0.5625` to the main options plus `.main-carousel .splide__slide img { width: 100%; height: 100%; object-fit: cover; }`. Alternatively, use `cover: true` with `heightRatio` (the tutorial demo did this).
- **Vertical thumbnails beside the main carousel:** `direction: 'ttb'` needs a `height` (or `heightRatio`). The breakpoint switches back to a horizontal strip, and changing direction calls `refresh()`, so the component uses the refresh workaround. The thumbnails come first in the JSX:
  ```tsx
  import { useEffect, useRef } from 'react';
  import { Splide, SplideSlide } from '@splidejs/react-splide';
  import type { Options } from '@splidejs/react-splide';
  import '@splidejs/react-splide/css';
  import './VerticalThumbnailCarousel.css';

  const images = [
    { id: 1015, alt: 'Deep blue fjord between steep rocky cliffs' },
    { id: 1018, alt: 'Green hills and sea cliffs under a cloudy sky' },
    { id: 1016, alt: 'Red desert canyon lit by the setting sun' },
    { id: 1036, alt: 'Snow-covered campsite below jagged mountains' },
    { id: 1050, alt: 'Turquoise sea around white chalk cliffs' },
    { id: 29, alt: 'Snow-covered mountain range under a blue sky' },
  ];

  const mainOptions: Options = {
    label     : 'Beautiful Gallery',
    type      : 'loop',
    pagination: false,
  };

  const thumbsOptions: Options = {
    label       : 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
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
  };

  export default function VerticalThumbnailCarousel() {
    const mainRef   = useRef<Splide>( null );
    const thumbsRef = useRef<Splide>( null );

    useEffect( () => {
      const thumbs = thumbsRef.current?.splide;

      if ( ! mainRef.current || ! thumbs ) {
        return;
      }

      mainRef.current.sync( thumbs );

      // Changing direction in a breakpoint calls refresh(), which drops the navigation ARIA
      thumbs.on( 'refresh.nav', () => {
        thumbs.emit( 'navigation:mounted', thumbs.splides );
      } );

      return () => {
        thumbs.off( 'refresh.nav' );
      };
    }, [] );

    // The thumbnails come first in the markup (left column). In React the order does not matter,
    // because sync() runs in the effect, after both carousels have mounted.
    return (
      <div className="gallery vertical-gallery">
        <Splide ref={ thumbsRef } tag="section" className="thumbnail-carousel" options={ thumbsOptions }>
          { images.map( image => (
            <SplideSlide key={ image.id }>
              <img src={ `https://picsum.photos/id/${ image.id }/200/120` } alt={ image.alt } />
            </SplideSlide>
          ) ) }
        </Splide>

        <Splide ref={ mainRef } tag="section" className="main-carousel" options={ mainOptions }>
          { images.map( image => (
            <SplideSlide key={ image.id }>
              <img src={ `https://picsum.photos/id/${ image.id }/1200/675` } alt={ image.alt } />
            </SplideSlide>
          ) ) }
        </Splide>
      </div>
    );
  }
  ```
  ```css
  /* VerticalThumbnailCarousel.css */
  .vertical-gallery {
    display: flex;
    gap: 10px;
    max-width: 960px;
    margin: 2rem auto;
  }

  .vertical-gallery .main-carousel {
    flex: 1;
    min-width: 0;
  }

  .vertical-gallery .main-carousel .splide__slide img {
    width: 100%;
    height: auto;
    display: block;
  }

  .vertical-gallery .thumbnail-carousel .splide__slide img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  @media (max-width: 600px) {
    .vertical-gallery {
      flex-direction: column-reverse; /* thumbnails below the main carousel on phones */
    }
  }
  ```
  Verified: above 600px the thumbnails are a 100x60 vertical strip with a 400px track. At 600px and below they switch to a horizontal 60x44 strip, and back again when the window grows. The navigation ARIA (`button`, "Go to slide N", `aria-controls`) is correct on every slide through both switches. Clicking thumbnail 5 moves the loop main carousel to index 4, and the main's next arrow moves the thumbnails to 5. One leftover: after the refresh, the list's `aria-orientation="vertical"` is not set again. This is minor, because the list has `role="presentation"`.
- **Thumbnails from state (dynamic slides):** the refresh workaround in a component that adds a slide:
  ```tsx
  import { useEffect, useRef, useState } from 'react';
  import { Splide, SplideSlide } from '@splidejs/react-splide';
  import type { Options } from '@splidejs/react-splide';
  import '@splidejs/react-splide/css';
  import './ThumbnailCarousel.css';

  type Photo = { id: number; alt: string };

  const mainOptions: Options = {
    label     : 'Beautiful Gallery',
    type      : 'fade',
    rewind    : true,
    pagination: false,
    arrows    : false,
  };

  const thumbsOptions: Options = {
    label       : 'The carousel with thumbnails. Selecting a thumbnail will change the Beautiful Gallery carousel.',
    fixedWidth  : 100,
    fixedHeight : 60,
    gap         : 10,
    rewind      : true,
    pagination  : false,
    focus       : 'center',
    isNavigation: true,
  };

  export default function DynamicThumbnailCarousel() {
    const mainRef   = useRef<Splide>( null );
    const thumbsRef = useRef<Splide>( null );
    const [ images, setImages ] = useState<Photo[]>( [
      { id: 1015, alt: 'Deep blue fjord between steep rocky cliffs' },
      { id: 1018, alt: 'Green hills and sea cliffs under a cloudy sky' },
      { id: 1016, alt: 'Red desert canyon lit by the setting sun' },
    ] );

    useEffect( () => {
      const thumbs = thumbsRef.current?.splide;

      if ( ! mainRef.current || ! thumbs ) {
        return;
      }

      mainRef.current.sync( thumbs );

      // React Splide calls refresh() when the slides change. Splide then rebuilds the thumbnail
      // slides without the navigation ARIA. This handler is registered after mount, so it runs
      // after the rebuild and applies "Go to slide N", role="button" and aria-controls again.
      thumbs.on( 'refresh.nav', () => {
        thumbs.emit( 'navigation:mounted', thumbs.splides );
      } );

      return () => {
        thumbs.off( 'refresh.nav' );
      };
    }, [] );

    function addImage() {
      setImages( prev => [ ...prev, { id: 1036, alt: 'Snow-covered campsite below jagged mountains' } ] );
    }

    return (
      <div className="gallery">
        <Splide ref={ mainRef } tag="section" className="main-carousel" options={ mainOptions }>
          { images.map( image => (
            <SplideSlide key={ image.id }>
              <img src={ `https://picsum.photos/id/${ image.id }/1200/675` } alt={ image.alt } />
            </SplideSlide>
          ) ) }
        </Splide>

        <Splide ref={ thumbsRef } tag="section" className="thumbnail-carousel" options={ thumbsOptions }>
          { images.map( image => (
            <SplideSlide key={ image.id }>
              <img src={ `https://picsum.photos/id/${ image.id }/200/120` } alt={ image.alt } />
            </SplideSlide>
          ) ) }
        </Splide>

        <button type="button" className="add-image" onClick={ addImage } disabled={ images.length > 3 }>
          Add image
        </button>
      </div>
    );
  }
  ```
  It uses the CSS from the complete component. Verified: after "Add image", the four thumbnails have `button` / "Go to slide 1" to "Go to slide 4" / the matching main slide IDs, and clicking the new thumbnail moves the main carousel to index 3.
- **No theme CSS:** import `@splidejs/react-splide/css/core`. Add the `is-active` opacity rules from step 4, or copy the theme's border rules shown there, plus a focus style such as `.thumbnail-carousel .splide__slide:focus-visible { outline: 3px solid #0bf; outline-offset: -3px; }`.
- **Clicks ignored after tiny drags:** `dragMinThreshold: { mouse: 4, touch: 10 }` on the thumbnails.
- **Lazy loading the main images:** add `lazyLoad: 'nearby'` to the main options and render `data-splide-lazy={ url }` instead of `src`. Thumbnails are small, so they are usually loaded eagerly. (In development, StrictMode leaves a spinner over the first lazy images, see `references/tutorials/image-carousel.md`.)
- **Translated or custom labels:** `i18n: { slideX: 'Bild %s anzeigen' }` on the thumbnails changes "Go to slide %s". On the main carousel, `i18n: { slideLabel: '%s von %s', carousel: 'Karussell', slide: 'Folie' }`. Translate both `label` options too.
- **Several galleries on one page:** the component can be rendered many times as it is. It sets no `id`, so Splide gives each root a unique one (`splide01`, `splide02`, ...) and the slide IDs and `aria-controls` stay unique. If you pass `id` props, make them unique per instance.
- **Breakpoint `destroy`:** `sync()` links are kept on a breakpoint destroy (`destroy: true`) and restored on remount. `destroy: 'completely'` or `splide.destroy()` (completely by default, which is what unmounting does) removes the links. Event props on either `<Splide>` stop firing after a breakpoint destroy and remount, see `references/tutorials/gallery.md`.
