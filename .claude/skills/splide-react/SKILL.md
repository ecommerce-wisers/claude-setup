---
name: splide-react
description: "Build, configure or debug Splide v4 carousels in React or Next.js with @splidejs/react-splide: components, props, event props, refs, sync, extensions."
---

# Splide v4 — React (`@splidejs/react-splide`)

Reference for React Splide. Core Splide options, events and component APIs are identical to vanilla Splide v4 — this skill covers the React layer plus the options you'll use most. Source: https://splidejs.com/integration/react-splide/

## Workflow

1. Install `@splidejs/react-splide` (it bundles `@splidejs/splide`; extensions are separate packages).
2. Import one CSS file once (app entry / layout).
3. In Next.js App Router, put the carousel in a `'use client'` component.
4. Keep `options` stable (`const` outside the component or `useMemo`) — the prop is reactive and a new object each render triggers updates.
5. Use `hasTrack={false}` + `<SplideTrack>` when arrows/pagination/toggle must sit outside the track.
6. Access the core instance through `ref.current.splide` after mount.
7. Run the accessibility checklist at the end.

## Install & CSS

```bash
npm install @splidejs/react-splide
```

```js
import '@splidejs/react-splide/css';            // default theme
// '@splidejs/react-splide/css/skyblue' | '/css/sea-green' | '/css/core'
```

## Basic

```tsx
'use client'; // Next.js App Router only
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/splide';
import '@splidejs/react-splide/css';

const options: Options = {
  perPage: 3,
  gap: '1rem',
  rewind: true,
  breakpoints: { 1024: { perPage: 2 }, 640: { perPage: 1 } },
};

export default function ProductCarousel( { products } ) {
  return (
    <Splide tag="section" options={ options } aria-label="Featured products">
      { products.map( p => (
        <SplideSlide key={ p.id }>
          <img src={ p.image } alt={ p.title } />
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

## Props (`<Splide>`)

| Prop | Type | Default | Notes |
|---|---|---|---|
| `options` | `Options` | — | Reactive; don't change readonly options (e.g. `type`) at runtime |
| `hasTrack` | boolean | `true` | `false` → render `<SplideTrack>` yourself |
| `tag` | `'div' \| 'section' \| 'header' \| 'footer' \| 'nav'` | `'div'` | `'section'` recommended |
| `extensions` | `Record<string, ComponentConstructor>` | — | e.g. `{ AutoScroll }` |
| `transition` | `ComponentConstructor` | — | custom transition |
| `aria-label` / `aria-labelledby` | string | — | always set one |
| other HTML attrs | — | — | `className`, `id`, etc. go to root |

Exports: `Splide`, `SplideSlide`, `SplideTrack`.

## Event props

Rule: `on` + PascalCase of the event, colons removed. Handler receives the **Splide instance first**, then the original event args.

| Core event | Prop | Handler args |
|---|---|---|
| `mounted` / `ready` | `onMounted` / `onReady` | `(splide)` |
| `move` / `moved` | `onMove` / `onMoved` | `(splide, newIndex, prevIndex, destIndex)` |
| `active` / `inactive` | `onActive` / `onInactive` | `(splide, Slide)` |
| `visible` / `hidden` | `onVisible` / `onHidden` | `(splide, Slide)` |
| `click` | `onClick` | `(splide, Slide, e)` |
| `updated` | `onUpdated` | `(splide, options)` |
| `resize` / `resized` / `refresh` / `destroy` | `onResize` / `onResized` / `onRefresh` / `onDestroy` | `(splide)` |
| `drag` / `dragging` / `dragged` | `onDrag` / `onDragging` / `onDragged` | `(splide)` |
| `scroll` / `scrolled` | `onScroll` / `onScrolled` | `(splide)` |
| `overflow` | `onOverflow` | `(splide, isOverflow)` |
| `arrows:mounted` / `arrows:updated` | `onArrowsMounted` / `onArrowsUpdated` | `(splide, prev, next, ...)` |
| `pagination:mounted` / `pagination:updated` | `onPaginationMounted` / `onPaginationUpdated` | `(splide, data, ...)` |
| `navigation:mounted` | `onNavigationMounted` | `(splide, splides)` |
| `autoplay:play` / `autoplay:pause` / `autoplay:playing` | `onAutoplayPlay` / `onAutoplayPause` / `onAutoplayPlaying` | `(splide, rate?)` |
| `lazyload:loaded` | `onLazyloadLoaded` | `(splide, img, Slide)` |

Full list: `src/js/types/events.ts` in the react-splide repo. `*:mounted` props are bound before mount automatically.

```tsx
<Splide
  onMoved={ ( splide, newIndex ) => setActive( newIndex ) }
  onPaginationMounted={ ( splide, data ) => {
    data.items.forEach( item => { item.button.textContent = String( item.page + 1 ); } );
  } }
/>
```

## Ref & instance API

```tsx
import { useRef, useEffect } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';

const ref = useRef<Splide>( null );

useEffect( () => {
  const splide = ref.current?.splide;  // core Splide instance
  if ( ! splide ) return;
  console.log( splide.length, splide.index );
}, [] );

<Splide ref={ ref }>...</Splide>

// Controls
ref.current?.splide?.go( '>' );   // '<', '+1', '-1', 2, '>2'
ref.current?.splide?.Components.Autoplay?.pause();
```

Prefer event props over `splide.on()` in effects; if you do call `on()`, use a namespace and `off()` in the cleanup.

## Custom structure (arrows / autoplay UI outside track)

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';

<Splide hasTrack={ false } tag="section" options={ { type: 'loop', autoplay: true } } aria-label="Hero">
  <div className="custom-wrapper">
    <SplideTrack>
      <SplideSlide>...</SplideSlide>
    </SplideTrack>

    <div className="splide__arrows">
      <button className="splide__arrow splide__arrow--prev">Prev</button>
      <button className="splide__arrow splide__arrow--next">Next</button>
    </div>

    <div className="splide__progress"><div className="splide__progress__bar" /></div>

    <button className="splide__toggle" type="button">
      <span className="splide__toggle__play">Play</span>
      <span className="splide__toggle__pause">Pause</span>
    </button>
  </div>
</Splide>
```

Keep `splide__` class names — Splide finds elements by them. Never put extra wrappers between `SplideTrack`'s track and list.

## Thumbnails (sync)

```tsx
const mainRef = useRef<Splide>( null );
const thumbsRef = useRef<Splide>( null );

useEffect( () => {
  if ( mainRef.current && thumbsRef.current?.splide ) {
    mainRef.current.sync( thumbsRef.current.splide );
  }
}, [] );

const mainOptions: Options = { type: 'fade', rewind: true, pagination: false, arrows: false };
const thumbOptions: Options = {
  fixedWidth: 100, fixedHeight: 60, gap: 10, rewind: true,
  pagination: false, isNavigation: true,
  breakpoints: { 600: { fixedWidth: 60, fixedHeight: 44 } },
};

<Splide ref={ mainRef } options={ mainOptions } aria-label="Product images">{ slides }</Splide>
<Splide ref={ thumbsRef } options={ thumbOptions } aria-label="Thumbnails — selecting one changes the main image">{ thumbs }</Splide>
```

```css
.thumbs .splide__slide { opacity: .6; }
.thumbs .splide__slide.is-active { opacity: 1; }
```

## Extensions

```bash
npm install @splidejs/splide-extension-auto-scroll
```

```tsx
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';

<Splide
  extensions={ { AutoScroll } }
  options={ { type: 'loop', drag: 'free', focus: 'center', perPage: 4,
    autoScroll: { speed: 1, pauseOnHover: true } } }
  aria-label="Logos"
>...</Splide>
```

| Package | Export | Options key |
|---|---|---|
| `@splidejs/splide-extension-auto-scroll` | `AutoScroll` | `autoScroll: { speed:1, autoStart:true, rewind, pauseOnHover:true, pauseOnFocus:true }` |
| `@splidejs/splide-extension-intersection` | `Intersection` | `intersection: { inView, outView, root, rootMargin, threshold, once }` |
| `@splidejs/splide-extension-video` (+ its CSS) | `Video` | `video: { autoplay, mute, loop, volume, hideControls, host, playerOptions }`; `<SplideSlide data-splide-youtube="...">` |
| `@splidejs/splide-extension-grid` | `Grid` | `grid: { rows, cols, dimensions, gap:{row,col} }` |
| `@splidejs/splide-extension-url-hash` | `URLHash` | `<SplideSlide data-splide-hash="name">` |

Custom extension/transition: same function signature as vanilla — `( Splide, Components, options ) => ({ mount, destroy })`; transition must return `start( index, done )` + `cancel()`. Pass via `extensions` / `transition` props.

## Key options (shared with core)

`type` (`slide`/`loop`/`fade`) · `perPage` · `perMove` · `gap` · `padding` · `focus` (`'center'`) · `trimSpace` · `omitEnd` · `rewind` · `speed` (400) · `height` / `heightRatio` / `fixedWidth` / `fixedHeight` · `autoWidth` · `direction` (`ttb` needs height) · `arrows` · `pagination` · `drag` (`'free'` + `snap`) · `autoplay` (`'pause'`) · `interval` (5000) · `pauseOnHover` · `lazyLoad` (`'nearby'` + `data-splide-lazy`) · `keyboard` (false) · `wheel` · `isNavigation` · `mediaQuery` (`'min'` mobile-first) · `breakpoints` · `reducedMotion` · `classes` · `i18n` · `label`.

## Dynamic slides & reactivity

- Changing children (add/remove `SplideSlide`) is handled — React Splide refreshes the carousel. Always use stable `key`s.
- Changing `options` merges new values (like `splide.options = {...}`); memoize to avoid needless refreshes.
- Readonly options (e.g. `type`) can't change after mount — remount with a different `key` on `<Splide>` instead.
- Hide controls when slides don't fill the view: CSS `.splide:not(.is-overflow) .splide__arrows { display:none }` or `onOverflow`.

## Next.js notes

- Splide touches `window`/DOM → client component (`'use client'`), or `dynamic( () => import( './Carousel' ), { ssr: false } )` in Pages Router if hydration warnings appear.
- Import the CSS in `app/layout.tsx` (or `_app.tsx`) — global CSS can't be imported from arbitrary components in Pages Router.
- With `next/image`, give images fixed dimensions or use `fill` inside a slide with explicit height (`fixedHeight`/`heightRatio`).

## Accessibility checklist

- `aria-label`/`aria-labelledby` on every `<Splide>`; `tag="section"` for main content, `role="group"` for decorative.
- Autoplay → render `splide__toggle` (needs `hasTrack={false}`) for WCAG 2.2.2.
- Translate `i18n` strings for non-English UIs (`prev`, `next`, `first`, `last`, `slideX`, `pageX`, `play`, `pause`, `carousel`, `select`, `slide`, `slideLabel`).
- Focus styles: themes include them; with core CSS use `.splide.is-focus-in .splide__arrow:focus`.

## Pitfalls

- Inline `options={{...}}` object recreated each render → repeated updates; hoist or `useMemo`.
- `ref.current.splide` is undefined until mounted — guard in effects.
- Navigation (thumbnail) carousel must have `pagination: false`.
- Wrong CSS path: use `@splidejs/react-splide/css`, not the core package path, unless core is installed separately.
- Extension options without passing the extension in `extensions` do nothing.
