# Next.js with React Splide

App Router and Pages Router setup, server rendering, images and the development-only StrictMode issue. Tested with Next.js 16.4 (Turbopack), React 19.3 and React Splide 0.7.12.

Contents: [Client component](#client-component) · [TypeScript](#typescript) · [CSS](#css) · [Server rendering and first paint](#server-rendering-and-first-paint) · [Images](#images) · [Labels in development](#labels-in-development) · [Pages Router](#pages-router) · [Things you don't need](#things-you-dont-need)

## Client component

React Splide is a class component and its build has no `'use client'` directive. A Server Component that renders `<Splide>` fails the build:

```
Error: Failed to collect page data for /
  [cause]: TypeError: Class extends value undefined is not a constructor or null
```

Put the carousel in its own client component and render that from the server page. Pass it serializable data (strings, numbers, plain objects), not functions.

```tsx
// app/components/ProductCarousel.tsx
'use client';

import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = { perPage: 3, gap: '1rem', label: 'Featured products' };

export default function ProductCarousel( { items }: { items: { id: string; title: string }[] } ) {
  return (
    <Splide tag="section" options={ options }>
      { items.map( item => (
        <SplideSlide key={ item.id }>{ item.title }</SplideSlide>
      ) ) }
    </Splide>
  );
}
```

```tsx
// app/page.tsx (a Server Component)
import ProductCarousel from './components/ProductCarousel';

export default async function Page() {
  const items = await getFeatured();   // fetch on the server
  return <ProductCarousel items={ items } />;
}
```

Only the file with `'use client'` and what it imports become client code; the page stays a Server Component.

## TypeScript

Next.js projects use `"moduleResolution": "bundler"`, where React Splide's types don't resolve, and `next build` stops with `TS7016: Could not find a declaration file for module '@splidejs/react-splide'`. Add `"resolvePackageJsonExports": false` to `compilerOptions`. Don't use a `paths` entry for it: Next.js applies `paths` at runtime too, and the page then crashes at prerender. Details and the alternative: `references/typescript.md`.

## CSS

Import the theme once, either in the client component (as above) or in `app/layout.tsx`. Both work in the App Router. The import paths are `@splidejs/react-splide/css`, `@splidejs/react-splide/css/core` and `@splidejs/react-splide/css/<theme>` (see `references/themes.md`).

## Server rendering and first paint

The carousel is server-rendered as plain markup (`<section class="splide"><div class="splide__track"><ul class="splide__list"><li class="splide__slide">...`), and Splide mounts after hydration. There are no hydration warnings (tested in development), because Splide changes the DOM only after React has hydrated it.

But the core CSS hides every carousel until it mounts:

```css
.splide { position: relative; visibility: hidden; }
.splide.is-initialized, .splide.is-rendered { visibility: visible; }
```

So the server HTML is invisible until the JavaScript has loaded and run. For a hero carousel that holds the page's largest image, that delays Largest Contentful Paint. To show the first slides straight away, add CSS that lays out the unmounted markup the way Splide will:

```css
/* perPage: 3, gap: 1rem */
.product-carousel:not( .is-initialized ) { visibility: visible; }
.product-carousel:not( .is-initialized ) .splide__list { gap: 1rem; }
.product-carousel:not( .is-initialized ) .splide__slide { width: calc( ( 100% - 2rem ) / 3 ); }
```

```tsx
<Splide tag="section" options={ options } className="product-carousel">
```

The width formula is `( 100% - ( perPage - 1 ) × gap ) / perPage`; repeat the rules in media queries if `breakpoints` change `perPage` or `gap`. Tested with the default `slide` type: with JavaScript off, the server HTML shows the first three slides at exactly the positions and widths Splide gives them after mounting (no layout shift). The core CSS already makes the list a flex row and clips the track, so slides past the first page stay hidden. With `type: 'loop'` the mounted position differed from the server layout in the same test, so check a loop carousel before relying on this.

## Images

`next/image` with `fill` works inside a slide when the slide has a height (from `heightRatio`, `height` or `fixedHeight`) and the image sits in a `position: relative` box:

```tsx
'use client';

import Image from 'next/image';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function Gallery( { images }: { images: { src: string; alt: string }[] } ) {
  return (
    <Splide options={ { heightRatio: 0.5, label: 'Gallery' } }>
      { images.map( ( image, i ) => (
        <SplideSlide key={ image.src }>
          <div style={ { position: 'relative', width: '100%', height: '100%' } }>
            <Image src={ image.src } alt={ image.alt } fill sizes="100vw" style={ { objectFit: 'cover' } } priority={ i === 0 } />
          </div>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Tested: both images fill the 784 × 392 slide. `priority` on the first image keeps it from lazy loading (the others get `loading="lazy"`). Use Next.js's lazy loading rather than Splide's `lazyLoad` with `next/image`; Splide's version needs `data-splide-lazy` on a plain `<img>`.

## Labels in development

The App Router runs React's StrictMode in development, which mounts each component, unmounts it and mounts it again. Splide's `destroy()` removes an `aria-label`, `aria-labelledby` or `role` that was passed as a prop, and React doesn't put it back, so in `next dev` the carousel ends up with no label (tested), and a `role="group"` prop becomes `role="region"`. Production builds are fine, but accessibility checks in development will flag it. Pass them through the options instead, which survive the remount:

```tsx
<Splide options={ { label: 'Featured products' } }>        // not aria-label="..."
<Splide options={ { labelledby: 'featured-heading' } }>    // not aria-labelledby="..."
<Splide options={ { role: 'group', label: 'Logos' } }>     // not role="group"
```

The other development-only effects of StrictMode (mount-time props firing twice, lazy-load spinners) are in `references/reactivity.md`.

## Pages Router

The same component works without `'use client'`. Importing `@splidejs/react-splide/css` from a page or component file is allowed, because Next.js accepts CSS imports from `node_modules` anywhere (tested: a page that imports the CSS itself builds and renders). `pages/_app.tsx` also works if you'd rather load it once.

## Things you don't need

- **`next/dynamic` with `ssr: false`.** Server rendering works, so there's no reason to skip it. In a Server Component, Next.js 16 rejects it with "`ssr: false` is not allowed with `next/dynamic` in Server Components".
- **`useEffect` guards for `window`.** React Splide creates the core instance in `componentDidMount`, which only runs in the browser.
- **A `key` that changes on route change.** Navigating away unmounts the component, which calls `splide.destroy()`; a carousel in a shared layout keeps its instance across navigations.
