# Carousel progress

Source: https://splidejs.com/tutorials/carousel-progress/

The carousel progress tutorial converted to React: a progress bar that shows how far through the slides the carousel is. The bar is a pair of `<div>`s inside the Splide root, and its width comes from `splide.index` and `Controller#getEnd()` on every move, either through React state or written straight to the DOM through a ref. Every component here was run in headless Chrome with React 19 and `@splidejs/react-splide` 0.7.12, in production and in development with StrictMode.

This bar shows the carousel **position**. It is not the same thing as the autoplay timer bar (`.splide__progress > .splide__progress__bar`), which Splide's Autoplay component fills while it waits for the next slide (see `references/autoplay.md`). Keep the custom class names below. Splide treats an element with the class `splide__progress__bar` as the autoplay bar, and the theme styles it.

Contents:
- Install and imports
- 1. Markup and CSS (`hasTrack={ false }`)
- 2. Updating the bar (state, stale handlers, ref-driven version)
- 3. Tips from the tutorial
- Complete component
- Variations (accessible slider, responsive `perPage`, transition speed, styling)

## Install and imports

```bash
npm install @splidejs/react-splide
```

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

Types need one tsconfig line under the default Vite and Next.js module resolution, see `references/typescript.md`. In the Next.js App Router the component must be a client component, see `references/nextjs.md`.

## 1. Markup and CSS

The bar must sit inside the Splide root, after the track. By default `<Splide>` wraps all its children in `<SplideTrack>`, so every child would end up inside `.splide__list` next to the slides. Set `hasTrack={ false }` and render `<SplideTrack>` yourself, then put the bar after it:

```tsx
<Splide tag="section" hasTrack={ false } options={ { label: 'Numbered slides with a progress bar' } }>
  <SplideTrack>
    <SplideSlide>Slide 01</SplideSlide>
    {/* ... */}
  </SplideTrack>

  <div className="my-carousel-progress">
    <div className="my-carousel-progress-bar" />
  </div>
</Splide>
```

(The tutorial's `aria-label="..."` is a placeholder. Give the carousel a real label, through `options.label`: an `aria-label` prop is removed in development by StrictMode's destroy and remount.)

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

Custom elements inside the root (outside the track) are allowed. Never put extra elements between `.splide__track` and `.splide__list`: `<SplideTrack>` renders both, so pass `className` or other attributes to `<SplideTrack>` instead of wrapping the list.

## 2. Updating the bar

The progress rate is `( current + 1 ) / ( end + 1 )`, where `current` is `splide.index` and `end` is `splide.Components.Controller.getEnd()`, the last index the carousel can move to. Recompute it on `mounted` (initial width) and on every `move`:

```tsx
const [ rate, setRate ] = useState( 0 );

function updateRate( splide: SplideInstance ) {
  const end = splide.Components.Controller.getEnd() + 1;
  setRate( Math.min( ( splide.index + 1 ) / end, 1 ) );
}

<Splide hasTrack={ false } onMounted={ updateRate } onMove={ updateRate }>
  ...
  <div className="my-carousel-progress-bar" style={ { width: `${ 100 * rate }%` } } />
</Splide>
```

`SplideInstance` is `NonNullable<Splide[ 'splide' ]>`, the core instance type, taken from the component so that nothing is imported from `@splidejs/splide`. Event props are bound before `mount()`, so `onMounted` fires and sets the initial width.

Why `getEnd()` and not `splide.length`: with `perPage: 3` and 9 slides, the last reachable index is 6. Dividing by `getEnd() + 1` makes the bar reach 100% on the last page. `Math.min( ..., 1 )` caps the rate in cases where the index can pass the end index, for example in a `loop` carousel with `perPage > 1`.

Verified against the 4.1.x source and in headless Chrome:

- `splide.root`, `splide.index` and `splide.Components.Controller.getEnd()` all exist on the instance that event props receive.
- `splide.index` is already updated when `move` fires.
- With 9 slides and default options, the width is 11.1111% initially, 22.2222% after the next arrow, 55.5556% after `go( 4 )` and 100% after `go( 8 )`. With `perPage: 3`, `getEnd()` returns 6.

### Stale handlers

React Splide binds event props once, in `componentDidMount`, and never re-reads them. A handler that reads props or state keeps seeing the values from the first render, and a handler prop added later is ignored. `updateRate` is safe because it reads everything from the `splide` argument and only calls a state setter, which React keeps stable. If the bar needs other values (a total from props, a colour from state), read them through a ref that you update on every render (`const latest = useRef( value ); latest.current = value;`). See `references/reactivity.md`.

### Ref-driven version (no re-renders)

`setRate` re-renders the component, and with it every `<SplideSlide>`, on each move. React Splide then compares the options and the slide elements, finds no change and does nothing, so this is cheap for a few slides. For many slides, or to keep React out of it, write the width straight to the DOM through a ref:

```tsx
import { useRef } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './CarouselProgress.css';

type SplideInstance = NonNullable<Splide[ 'splide' ]>;

const slides = [ '01', '02', '03', '04', '05', '06', '07', '08', '09' ];

const options: Options = {
  label  : 'Numbered slides with a progress bar',
  perPage: 3,
};

export default function RefProgressCarousel() {
  const barRef = useRef<HTMLDivElement>( null );

  // Writes to the DOM directly: no React re-render on every move.
  // barRef is the same object on every render, so this handler cannot go stale.
  function updateBar( splide: SplideInstance ) {
    const bar = barRef.current;

    if ( bar ) {
      const end  = splide.Components.Controller.getEnd() + 1;
      const rate = Math.min( ( splide.index + 1 ) / end, 1 );

      bar.style.width = `${ 100 * rate }%`;
      bar.style.transitionDuration = `${ splide.options.speed }ms`; // follows reducedMotion (speed 0)
    }
  }

  return (
    <Splide
      tag="section"
      hasTrack={ false }
      options={ options }
      onMounted={ updateBar }
      onMove={ updateBar }
    >
      <SplideTrack>
        { slides.map( slide => (
          <SplideSlide key={ slide }>Slide { slide }</SplideSlide>
        ) ) }
      </SplideTrack>

      <div className="my-carousel-progress" aria-hidden="true">
        <div className="my-carousel-progress-bar" ref={ barRef } />
      </div>
    </Splide>
  );
}
```

It uses the CSS from the complete component. A ref object is the same on every render, so reading `barRef.current` in a handler bound at mount is always current. This version also follows the live speed (see Variations) and marks the bar `aria-hidden`, because it is decorative. Verified: with `perPage: 3` the bar starts at 14.2857% (1 of 7), reaches 100% after `go( 6 )`, and has `transition-duration: 400ms`.

## 3. Tips from the tutorial

- **Clickable bar:** convert the clicked position into a target index, then move the carousel with `go()`.
- **Accessibility:** add ARIA attributes such as `aria-controls` and `aria-valuenow`. The WAI-ARIA slider role (https://www.w3.org/TR/wai-aria-1.2/#slider) describes what is needed.

Both are implemented and tested under [Variations](#variations).

## Complete component

Save the two files next to each other and render `<CarouselProgress />`.

```tsx
// CarouselProgress.tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import './CarouselProgress.css';

type SplideInstance = NonNullable<Splide[ 'splide' ]>;

const slides = [ '01', '02', '03', '04', '05', '06', '07', '08', '09' ];

const options: Options = {
  label: 'Numbered slides with a progress bar',
};

export default function CarouselProgress() {
  const [ rate, setRate ] = useState( 0 );

  // Reads everything from the instance argument, so it never sees stale props or state.
  function updateRate( splide: SplideInstance ) {
    const end = splide.Components.Controller.getEnd() + 1;
    setRate( Math.min( ( splide.index + 1 ) / end, 1 ) );
  }

  return (
    <Splide
      tag="section"
      hasTrack={ false }
      options={ options }
      onMounted={ updateRate }
      onMove={ updateRate }
    >
      <SplideTrack>
        { slides.map( slide => (
          <SplideSlide key={ slide }>Slide { slide }</SplideSlide>
        ) ) }
      </SplideTrack>

      {/* The progress bar: inside the root, after the track */}
      <div className="my-carousel-progress">
        <div className="my-carousel-progress-bar" style={ { width: `${ 100 * rate }%` } } />
      </div>
    </Splide>
  );
}
```

```css
/* CarouselProgress.css */
body {
  margin: 0;
  padding: 2rem 1rem;
  font-family: system-ui, sans-serif;
}

.splide {
  max-width: 960px;
  margin: 0 auto 3rem;
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
```

Verified in headless Chrome (production and development + StrictMode): the bar is a direct child of the root `<section>` (after the track), the carousel has `aria-label="Numbered slides with a progress bar"` and 9 slides labelled "1 of 9" and so on, and the widths match the list in section 2. There are no console errors or warnings.

The `.splide` rule in the CSS styles every carousel on the page. Scope it with a `className` on `<Splide>` when the page has other carousels.

## Variations

- **Clickable, keyboard-operable, accessible bar (both tutorial tips), with responsive `perPage`:** this makes the bar a WAI-ARIA `slider`. It enlarges the click area without making the bar look thicker, keeps the ARIA values in sync from React state, and handles the keys the slider pattern expects. The `perPage` button changes the `options` prop, which React Splide applies with `splide.options = ...` and Splide announces with `updated`.
  ```tsx
  import { useRef, useState } from 'react';
  import type { KeyboardEvent, MouseEvent } from 'react';
  import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
  import type { Options } from '@splidejs/react-splide';
  import '@splidejs/react-splide/css';
  import './CarouselProgress.css';
  import './SliderProgressCarousel.css';

  type SplideInstance = NonNullable<Splide[ 'splide' ]>;

  const slides = [ '01', '02', '03', '04', '05', '06', '07', '08', '09' ];

  export default function SliderProgressCarousel() {
    const splideRef = useRef<Splide>( null );
    const [ perPage, setPerPage ] = useState( 3 );
    const [ position, setPosition ] = useState( { now: 1, end: 1 } );
    const [ trackId, setTrackId ] = useState<string>();

    // Only reads the instance argument and calls a state setter: safe in handlers captured at mount.
    function update( splide: SplideInstance ) {
      const end = splide.Components.Controller.getEnd() + 1;
      setPosition( { now: Math.min( splide.index + 1, end ), end } );
    }

    function getEnd() {
      return splideRef.current?.splide?.Components.Controller.getEnd() ?? 0;
    }

    // Click: convert the clicked x position into a target index
    function onClick( e: MouseEvent<HTMLDivElement> ) {
      const rect = e.currentTarget.getBoundingClientRect();
      const rate = ( e.clientX - rect.left ) / rect.width;
      const end  = getEnd();

      splideRef.current?.go( Math.max( 0, Math.min( Math.floor( rate * ( end + 1 ) ), end ) ) );
    }

    // Keyboard support required by role="slider"
    function onKeyDown( e: KeyboardEvent<HTMLDivElement> ) {
      const map: Record<string, number | string> = { ArrowRight: '>', ArrowUp: '>', ArrowLeft: '<', ArrowDown: '<', Home: 0, End: getEnd() };

      if ( e.key in map ) {
        e.preventDefault();
        splideRef.current?.go( map[ e.key ] );
      }
    }

    return (
      <>
        <Splide
          ref={ splideRef }
          tag="section"
          hasTrack={ false }
          options={ { label: 'Numbered slides with a progress bar', perPage } }
          onMounted={ splide => {
            update( splide );
            setTrackId( splide.Components.Elements.track.id );

            // Registered during mount, after the Controller's own listeners, so the end index
            // is already recomputed when this runs. An onUpdated prop would read the old one.
            splide.on( 'updated resized', () => update( splide ) );
          } }
          onMove={ update }
        >
          <SplideTrack>
            { slides.map( slide => (
              <SplideSlide key={ slide }>Slide { slide }</SplideSlide>
            ) ) }
          </SplideTrack>

          <div
            className="my-carousel-progress"
            role="slider"
            tabIndex={ 0 }
            aria-label="Carousel position"
            aria-controls={ trackId }
            aria-valuemin={ 1 }
            aria-valuemax={ position.end }
            aria-valuenow={ position.now }
            aria-valuetext={ `Position ${ position.now } of ${ position.end }` }
            onClick={ onClick }
            onKeyDown={ onKeyDown }
          >
            <div className="my-carousel-progress-bar" style={ { width: `${ 100 * position.now / position.end }%` } } />
          </div>
        </Splide>

        <button type="button" className="per-page" onClick={ () => setPerPage( perPage === 3 ? 1 : 3 ) }>
          Show { perPage === 3 ? 1 : 3 } per page
        </button>
      </>
    );
  }
  ```
  ```css
  /* SliderProgressCarousel.css, loaded after CarouselProgress.css */
  .my-carousel-progress {
    cursor: pointer;
    padding: 6px 0;               /* 14px tall hit area ... */
    background-clip: content-box; /* ... while the grey track stays 2px tall */
  }

  .my-carousel-progress:focus-visible {
    outline: 3px solid #0bf;
  }
  ```
  Verified: with `perPage: 3` the slider starts at `aria-valuenow="1"`, `aria-valuemax="7"`, "Position 1 of 7", with `aria-controls` equal to the track ID and a 14px hit area. Clicking at 50% goes to index 3, the right edge goes to 6 (bar 100%), Home goes to 0 and ArrowRight goes to 3 (one page). Switching to 1 per page at index 3 gives "Position 4 of 9", and End then goes to 8. Switching back to 3 per page gives 7 of 7.

  `'>'`/`'<'` move the same way as the arrows: by `perMove` if set, otherwise by one page (one slide when `focus` is set). Use `'+1'`/`'-1'` to move by exactly one slide. Splide gives the track an ID on mount (`<root id>-track`, so `splide01-track` when the root has no `id`), which is why `trackId` is stored from `onMounted`.
- **Why the `updated resized` listener is added in `onMounted`, not as an `onUpdated` prop:** when breakpoints or a new `options` prop change `perPage`, the end index changes too, so the bar must be recomputed on `updated` and `resized`. Splide's handlers run in registration order. Event props are registered before `mount()`, so an `onUpdated` prop runs before the Controller has recomputed the end index and reads the stale value. A listener registered inside `onMounted` (or later, in an effect) runs after it. Verified in React: switching from `perPage: 3` to `1` gave a stale end of 6 to the `onUpdated` prop and the correct 8 to the `onMounted` listener. A listener added with `splide.on()` inside `onMounted` needs no cleanup, because `destroy()` (on unmount) removes it. Like event props, it does not come back after a breakpoint `destroy`. Use an extension for that (see `references/tutorials/gallery.md`).
- **Transition matched to the real speed:** under `prefers-reduced-motion`, Splide's default `reducedMotion` option sets `speed` to 0. To follow it, set the duration from the live options in the handler, as the ref-driven version does: `` bar.style.transitionDuration = `${ splide.options.speed }ms`; `` (or a `transitionDuration` entry in the `style` prop). A custom `speed` option needs the same.
- **Several slides per page or loop:** no code change. `getEnd()` already accounts for `perPage`, `focus` and `omitEnd`, and the `Math.min` cap handles loop indexes.
- **Thicker or styled bar:** change `height`, `background` and add `border-radius` on `.my-carousel-progress-bar` (and on `.my-carousel-progress` for the track). Put the bar above the slides by rendering the progress `<div>` before `<SplideTrack>`. It must stay inside `<Splide>`.
- **Purely decorative bar:** if you keep the plain, non-interactive bar, it repeats information that the slides already announce ("3 of 9"). Mark it `aria-hidden="true"` (as the ref-driven version does) instead of giving it partial ARIA.
- **Several carousels on one page:** each component instance has its own state or ref, so there is no selector to scope. (In vanilla, `new Splide( '.splide' )` only matches the first `.splide` on the page.)
