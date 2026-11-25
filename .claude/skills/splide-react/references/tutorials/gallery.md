# Gallery

Source: https://splidejs.com/tutorials/gallery/

The gallery tutorial converted to React: a main Splide carousel controlled through the basic APIs (`go()`, `index`, the `mounted` and `move` events) from a plain list of thumbnails. The thumbnails are not a Splide instance. Clicking a thumbnail moves the carousel, and when the carousel moves, React state marks the active thumbnail. Every component here was run in headless Chrome with React 19 and `@splidejs/react-splide` 0.7.12, in production and in development with StrictMode.

If there are too many thumbnails to fit in one row, use a second Splide carousel with `isNavigation` and `sync()` instead. See `references/tutorials/thumbnail-carousel.md`.

Contents:
- Install and imports
- 1. Markup and CSS
- 2. Main carousel
- 3. Thumbnails: click to navigate
- 4. Highlight the active thumbnail
- 5. Initial active state
- 6. Whole code, and the `destroy` caveat
- 7. Accessibility
- Complete component
- Variations (destroy-compatible extension, loop, several per page, styling)

## Install and imports

```bash
npm install @splidejs/react-splide
```

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

Types need one tsconfig line under the default Vite and Next.js module resolution, see `references/typescript.md`. In the Next.js App Router the component must be a client component, see `references/nextjs.md`.

## 1. Markup and CSS

The carousel is a `<Splide>` with one `<SplideSlide>` per image, and the thumbnails are an ordinary `<ul>` rendered by the same component. Describe the images with `alt`. The tutorial uses `image1.jpg` and `thumbnail1.jpg` with an empty `alt` as placeholders. Real URLs and alt text are filled in here.

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

Pagination is disabled because the thumbnails take over that job. Label the carousel with `options.label`: an `aria-label` prop is removed in development by StrictMode's destroy and remount.

```tsx
const options: Options = {
  label     : 'My Awesome Gallery',
  pagination: false,
};
```

## 3. Thumbnails: click to navigate

Keep a ref to the `<Splide>` component and call its `go()` method from each thumbnail's `onClick`. `go()` accepts a slide index:

```tsx
const splideRef = useRef<Splide>( null );

<Splide ref={ splideRef } options={ options }>...</Splide>

<li className="thumbnail">
  <button type="button" onClick={ () => splideRef.current?.go( index ) }>...</button>
</li>
```

`splideRef.current.go( control )` calls the core `go()` on the instance, so `splideRef.current?.splide?.go( index )` is equivalent. These are ordinary React DOM handlers on your own elements, not Splide event props, so they always see the latest render.

## 4. Highlight the active thumbnail

Keep the active index in state and update it from the `onMove` event prop, which fires every time the carousel moves:

```tsx
const [ activeIndex, setActiveIndex ] = useState( 0 );

<Splide onMove={ ( splide, newIndex ) => setActiveIndex( newIndex ) }>...</Splide>

<li className={ index === activeIndex ? 'thumbnail is-active' : 'thumbnail' }>
```

Event props are called with the core instance first, then the core event's arguments, so `onMove` receives `( splide, newIndex, prevIndex, destIndex )`. `splide.index` is already updated when `move` fires (the Controller sets the index before it emits `move`). In a `loop` carousel, `newIndex` and `splide.index` are the normalized index (0 to length - 1), so they map directly to the thumbnails.

React Splide binds event props once, when the carousel mounts, and never re-reads them. The handler above is safe because it only calls a state setter, which React keeps stable. A handler that reads props or state sees the values from the first render. Use the `splide` argument, a functional update (`setActiveIndex( i => ... )`) or a ref for anything else. See `references/reactivity.md`.

Style the selected thumbnail with the class:

```css
.thumbnail {
  /* ... */
  opacity: 0.3;
}

.thumbnail.is-active {
  opacity: 1;
}
```

## 5. Initial active state

In vanilla Splide nothing is active until the first move, so the tutorial also listens to `mounted`. In React, initial state (`useState( 0 )`) already marks thumbnail 1. Use `onMounted` anyway when the start index can differ (the `start` option) or when you need data that only exists after mount, such as the slide IDs for `aria-controls` (section 7). Event props are bound before `mount()`, so `onMounted` does fire. (In vanilla, a `mounted` listener added after `mount()` never fires.)

## 6. Whole code, and the `destroy` caveat

The complete component is under [Complete component](#complete-component). It is the tutorial's whole code plus the accessibility tips.

**Caveat: `destroy` option.** This gallery does not work with the `destroy` option, for example `breakpoints: { 640: { destroy: true } }`. Splide discards every handler registered with `splide.on()` when it is destroyed, and React Splide registers the event props with `splide.on()` only once, in `componentDidMount`. Verified on the same component with that breakpoint: after a breakpoint destroy and remount, clicking a thumbnail still moves the carousel (it is a React DOM handler), but `onMove` never fires again, so the active thumbnail stays stuck. The tutorial says an extension is needed for this case. A tested one is under [Variations](#variations).

## 7. Accessibility

The tutorial strongly recommends:

- `tabindex="0"` on the thumbnails, for keyboard users.
- An `aria-label` saying what clicking each thumbnail does (`aria-label="Go to slide 1"`).
- `aria-current` on the active thumbnail.
- `aria-controls` linking each thumbnail to its slide.

How the complete component implements them:

- Each thumbnail's content is a `<button type="button">`. A button is focusable without `tabIndex` and activates on Enter and Space without a key handler, and the `<ul>`/`<li>` list semantics stay valid. A `role="button"` on the `<li>` itself would break the list (list items cannot carry that role), so an `<li>` with `tabIndex={ 0 }` would also need a role change and an `onKeyDown` handler. The button is simpler.
- `aria-label="Go to slide N"`, `aria-current="true"` (only on the active one) and `aria-controls` go on the button, all rendered from state.
- Splide generates the slide IDs on mount as `<root id>-slide01`, `-slide02`, and so on. Without an `id` prop the root ID is generated too (`splide01`), so the slide IDs are only known after mount. The `onMounted` handler reads them with `splide.Components.Slides.get( true )` (`true` excludes loop clones) and stores them in state.

## Complete component

Save the two files next to each other and render `<Gallery />`.

```tsx
// Gallery.tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './Gallery.css';

const images = [
  { id: 1043, alt: 'Granite cliff rising above a pine forest and a river', thumbAlt: 'Granite cliff above a pine forest' },
  { id: 1044, alt: 'Person standing beside a rushing river in a misty forest', thumbAlt: 'Person beside a misty river' },
  { id: 1039, alt: 'Waterfall flowing down a mossy, forested hillside', thumbAlt: 'Waterfall on a mossy hillside' },
  { id: 15, alt: 'Waterfall pouring into a rocky gorge', thumbAlt: 'Waterfall in a rocky gorge' },
];

const options: Options = {
  label     : 'My Awesome Gallery',
  pagination: false, // the thumbnails take over that job
};

export default function Gallery() {
  const splideRef = useRef<Splide>( null );
  const [ activeIndex, setActiveIndex ] = useState( 0 );
  const [ slideIds, setSlideIds ] = useState<string[]>( [] );

  return (
    <>
      <Splide
        ref={ splideRef }
        tag="section"
        className="main-carousel"
        options={ options }
        onMounted={ splide => {
          // Splide generates the slide IDs on mount ("splide01-slide01", ...). Read them for aria-controls.
          setSlideIds( splide.Components.Slides.get( true ).map( Slide => Slide.slide.id ) );
          setActiveIndex( splide.index );
        } }
        onMove={ ( splide, newIndex ) => setActiveIndex( newIndex ) }
      >
        { images.map( image => (
          <SplideSlide key={ image.id }>
            <img src={ `https://picsum.photos/id/${ image.id }/1200/675` } alt={ image.alt } />
          </SplideSlide>
        ) ) }
      </Splide>

      <ul className="thumbnails">
        { images.map( ( image, index ) => (
          <li key={ image.id } className={ index === activeIndex ? 'thumbnail is-active' : 'thumbnail' }>
            <button
              type="button"
              aria-label={ `Go to slide ${ index + 1 }` }
              aria-controls={ slideIds[ index ] }
              aria-current={ index === activeIndex ? 'true' : undefined }
              onClick={ () => splideRef.current?.go( index ) }
            >
              <img src={ `https://picsum.photos/id/${ image.id }/140/140` } alt={ image.thumbAlt } />
            </button>
          </li>
        ) ) }
      </ul>
    </>
  );
}
```

```css
/* Gallery.css */
body {
  margin: 0;
  padding: 2rem 1rem;
  font-family: system-ui, sans-serif;
}

.main-carousel {
  max-width: 960px;
  margin: 0 auto;
}

.main-carousel .splide__slide img {
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

/* Button reset: the button inside each thumbnail provides keyboard access */
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

/* While the carousel is destroyed (640px and below), the slides stack, so hide the thumbnails */
@media (max-width: 640px) {
  .thumbnails.destroyable {
    display: none;
  }
}
```

The last rule is for the `destroy` variation below. It does nothing here.

Verified in headless Chrome (production and development + StrictMode). Thumbnail 1 starts active (`is-active`, `aria-current="true"`, opacity 1), and the others are at 0.3 without `aria-current`. Each button's `aria-controls` equals its slide's generated ID (`splide01-slide01` to `04`), and the labels read "Go to slide N". There is no pagination. Clicking thumbnail 3 moves to index 2 and moves the active state. `go( '>' )` on the instance activates thumbnail 4. Enter on the focused first button moves back to index 0. There are no console errors or warnings.

## Variations

- **Compatible with the `destroy` option (extension):** extension components are rebuilt and their `mount()` runs again every time Splide remounts after a breakpoint destroy, so a listener registered there comes back. Event props do not. Pass the extension through the `extensions` prop. React Splide reads that prop once, at mount (it is passed to `splide.mount()` and Splide keeps it for later remounts), so later values are ignored. `useMemo` only avoids building a new object on every render. The extension closes over the two state setters, which are stable, so it cannot go stale. The thumbnails are React elements with React handlers, so unlike the vanilla extension there are no DOM listeners to remove in `destroy()`.
  ```tsx
  import { useMemo, useRef, useState } from 'react';
  import { Splide, SplideSlide } from '@splidejs/react-splide';
  import type { Options, SplideProps } from '@splidejs/react-splide';
  import '@splidejs/react-splide/css';
  import './Gallery.css';

  type Extension = NonNullable<SplideProps[ 'extensions' ]>[ string ];

  const images = [
    { id: 1043, alt: 'Granite cliff rising above a pine forest and a river', thumbAlt: 'Granite cliff above a pine forest' },
    { id: 1044, alt: 'Person standing beside a rushing river in a misty forest', thumbAlt: 'Person beside a misty river' },
    { id: 1039, alt: 'Waterfall flowing down a mossy, forested hillside', thumbAlt: 'Waterfall on a mossy hillside' },
    { id: 15, alt: 'Waterfall pouring into a rocky gorge', thumbAlt: 'Waterfall in a rocky gorge' },
  ];

  const options: Options = {
    label      : 'My Awesome Gallery',
    pagination : false,
    breakpoints: {
      640: { destroy: true },
    },
  };

  export default function DestroyableGallery() {
    const splideRef = useRef<Splide>( null );
    const [ activeIndex, setActiveIndex ] = useState( 0 );
    const [ slideIds, setSlideIds ] = useState<string[]>( [] );

    // Splide rebuilds extension components on every mount, including the remount after a
    // breakpoint destroy, so this listener comes back. Event props (onMove, ...) do not.
    // React Splide reads `extensions` once, at mount: useMemo only avoids a new object per render.
    const extensions = useMemo<Record<string, Extension>>( () => ( {
      GalleryThumbnails: Splide => ( {
        mount() {
          Splide.on( 'mounted move', () => {
            setActiveIndex( Splide.index );
            setSlideIds( Splide.Components.Slides.get( true ).map( Slide => Slide.slide.id ) );
          } );
        },
      } ),
    } ), [] );

    return (
      <>
        <Splide
          ref={ splideRef }
          tag="section"
          className="main-carousel"
          options={ options }
          extensions={ extensions }
        >
          { images.map( image => (
            <SplideSlide key={ image.id }>
              <img src={ `https://picsum.photos/id/${ image.id }/1200/675` } alt={ image.alt } />
            </SplideSlide>
          ) ) }
        </Splide>

        <ul className="thumbnails destroyable">
          { images.map( ( image, index ) => (
            <li key={ image.id } className={ index === activeIndex ? 'thumbnail is-active' : 'thumbnail' }>
              <button
                type="button"
                aria-label={ `Go to slide ${ index + 1 }` }
                aria-controls={ slideIds[ index ] }
                aria-current={ index === activeIndex ? 'true' : undefined }
                onClick={ () => splideRef.current?.go( index ) }
              >
                <img src={ `https://picsum.photos/id/${ image.id }/140/140` } alt={ image.thumbAlt } />
              </button>
            </li>
          ) ) }
        </ul>
      </>
    );
  }
  ```
  It uses `Gallery.css` from the complete component, whose last rule hides the thumbnails at 640px and below: while the carousel is destroyed, the slides render as a plain stacked list. The `Extension` type is derived from `SplideProps`, so the component does not import types from `@splidejs/splide`. Verified: thumbnail 1 starts active, at 600px the carousel is destroyed and the thumbnails are hidden, back at 1280px it remounts, and clicking thumbnail 3 moves to index 2 and activates thumbnail 3. `aria-controls` still matches the slide IDs. Component `mount()` runs before Splide emits `mounted`, so the `mounted` listener registered inside it fires. React Splide also spreads `extensions` onto the root element as an attribute (`extensions="[object Object]"`). It is harmless.
- **Loop or fade main carousel:** `type: 'loop'` or `type: 'fade'` (with `rewind: true`) work unchanged, because `newIndex` is always the real slide index.
- **Several slides per page:** with `perPage: 3` and no `focus`, indexes past `Controller.getEnd()` are clamped. With 6 slides, `go( 5 )` lands on index 3, so clicking thumbnail 6 highlights thumbnail 4. Add `focus: 0` (or `focus: 'center'`) so that every index is reachable. The end index becomes `length - 1` and `trimSpace` still stops the track at the edges.
- **Different active style:** replace the opacity rules with `.thumbnail.is-active { outline: 2px solid #000; outline-offset: -2px; }`, or a border with `box-sizing: border-box`.
- **Lazy loading:** add `lazyLoad: 'nearby'` to the main options and render `data-splide-lazy={ url }` instead of `src` on the main images. The thumbnails are ordinary images (add `loading="lazy"` if needed).
- **Many thumbnails:** a flex row overflows on small screens. Allow wrapping (`flex-wrap: wrap`), or make the thumbnails a Splide `isNavigation` carousel (see `references/tutorials/thumbnail-carousel.md`).
- **Several galleries on one page:** render the component several times. Each instance renders its own thumbnails and keeps its own ref and state, and Splide generates unique root and slide IDs, so nothing is shared. If you pass `id` props, keep them unique.
