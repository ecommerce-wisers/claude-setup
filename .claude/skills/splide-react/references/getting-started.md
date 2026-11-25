# Getting started

Scope: install React Splide, import its CSS, render the first carousel, and understand when the core instance exists. Checked against React Splide 0.7.12 (`src/js/`, `dist/`) and the core 4.1.4 source; every example below was run in headless Chrome with React 19.3.1, in production and in development with StrictMode.

## Contents

- [1. Install](#1-install)
- [2. What the package contains](#2-what-the-package-contains)
- [3. CSS](#3-css)
- [4. First carousel](#4-first-carousel)
- [5. When the instance exists](#5-when-the-instance-exists)
- [6. Errors thrown on mount](#6-errors-thrown-on-mount)
- [7. Several carousels on one page](#7-several-carousels-on-one-page)
- [8. Shared defaults](#8-shared-defaults)
- [9. Options in the `data-splide` attribute](#9-options-in-the-data-splide-attribute)
- [10. Vite and Next.js](#10-vite-and-nextjs)
- [11. Vanilla snippets converted to React](#11-vanilla-snippets-converted-to-react)

---

## 1. Install

```sh
npm install @splidejs/react-splide
pnpm add @splidejs/react-splide
yarn add @splidejs/react-splide
```

0.7.12 (published 2022-09-30) is the latest version, and the repository hasn't changed since 2022. Pin it (`"@splidejs/react-splide": "0.7.12"`) so a lockfile refresh can't change it. History and older APIs: `references/version-notes.md`.

Package facts (from `package.json` and `dist/`):

- No `peerDependencies`, so npm, pnpm and yarn install it next to any React version without a warning. It works with React 19 (tested). It is a class component (`Splide`) plus two function components, and the examples here use hooks, so use React 16.8 or newer.
- One dependency, `@splidejs/splide` `^4.1.3`. It is installed for its TypeScript types. The runtime code doesn't import it: `dist/js/react-splide.esm.js` inlines its own copy of core 4.1.3 (identical in behaviour to 4.1.4) and imports only `react`.
- The `exports` field has no `types` condition, so TypeScript projects that use `moduleResolution: "bundler"` (the Vite and Next.js default) or `node16`/`nodenext` need one extra tsconfig line before the imports are typed. See `references/typescript.md`.
- The dist has no `'use client'` directive. In the Next.js App Router, the file that renders `<Splide>` must be a client component. See `references/nextjs.md`.

---

## 2. What the package contains

| Path | Format | Use |
|---|---|---|
| `dist/js/react-splide.esm.js` | ES module, unminified (83,934 bytes) | The `import` and `default` export condition. Components plus the inlined core. Your bundler minifies it. |
| `dist/js/react-splide.cjs.js` | CommonJS | The `require` condition (Jest, older Node tooling) |
| `dist/css/splide.min.css`, `splide-core.min.css`, `themes/splide-{default,skyblue,sea-green}.min.css` | Minified CSS | Byte-identical to the core 4.1.4 files. Imported through the subpaths in section 3. |
| `dist/types/` | `.d.ts` | Type declarations (see `references/typescript.md` for resolving them) |

`exports` maps only `.`, `./css`, `./css/core` and `./css/*`. Anything else is blocked by bundlers that honour `exports`: `import '@splidejs/react-splide/dist/css/splide.min.css'` fails in Vite with `"./dist/css/splide.min.css" is not exported` (tested). Use the short subpaths.

Named exports (there is no default export):

| Export | Kind | Renders |
|---|---|---|
| `Splide` | class component | The root element (`tag`, default `<div>`) with `class="splide"`, plus a `<SplideTrack>` around the children unless `hasTrack={ false }` |
| `SplideTrack` | function component | `<div class="splide__track"><ul class="splide__list">{ children }</ul></div>` |
| `SplideSlide` | function component | `<li class="splide__slide">` |
| `SplideProps`, `SplideEventHandlers`, `SplideEventHandlerMap`, `SplideHTMLAttributes` | types | |
| `Options` | type (re-exported from core) | Use `import type { Options } from '@splidejs/react-splide'`. Importing it from `@splidejs/splide` breaks under pnpm's strict layout unless core is a direct dependency. |

Markup details, `hasTrack` and every attribute Splide adds: `references/structure.md`.

---

## 3. CSS

Import exactly one stylesheet, once, before your own carousel CSS (an app entry file or a root layout is a good place).

```tsx
// Default theme
import '@splidejs/react-splide/css';

// or another theme
import '@splidejs/react-splide/css/skyblue';
import '@splidejs/react-splide/css/sea-green';

// or only the core styles
import '@splidejs/react-splide/css/core';
```

| Import | File | Bytes | Contents |
|---|---|---|---|
| `@splidejs/react-splide/css` | `dist/css/splide.min.css` | 5,002 | Core plus default theme |
| `@splidejs/react-splide/css/default` | `dist/css/themes/splide-default.min.css` | 5,002 | Same content as `/css` |
| `@splidejs/react-splide/css/skyblue` | `dist/css/themes/splide-skyblue.min.css` | 5,137 | Core plus Sky Blue theme |
| `@splidejs/react-splide/css/sea-green` | `dist/css/themes/splide-sea-green.min.css` | 5,354 | Core plus Sea Green theme |
| `@splidejs/react-splide/css/core` | `dist/css/splide-core.min.css` | 1,964 | Layout and state rules only, no arrow, dot or focus styling |

Each theme file is complete. Never load a theme together with `/css/core`, or two themes. What each file styles and how to override it: `references/themes.md`.

Old snippets (React Splide 0.6 and earlier) import `@splidejs/splide/dist/css/...` from the core package. That path resolves only when `@splidejs/splide` is reachable from your code: with npm and yarn hoisting it usually is, with pnpm it isn't unless you add core as a direct dependency. Prefer the `@splidejs/react-splide/css` subpaths.

The core CSS keeps every carousel at `.splide { visibility: hidden }` until Splide adds `is-initialized`, so the markup is invisible until React has mounted the component (and during server rendering until hydration).

---

## 4. First carousel

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const items = [ '01', '02', '03', '04', '05' ];

export default function FeaturedCarousel() {
  return (
    <Splide
      tag="section"
      options={ {
        label      : 'Featured items',
        perPage    : 3,
        gap        : '1rem',
        breakpoints: {
          640: { perPage: 1 },
        },
      } }
    >
      { items.map( item => (
        <SplideSlide key={ item }>{ item }</SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Tested: a `<section>` root with no `role` attribute, `aria-label="Featured items"`, `aria-roledescription="carousel"`, 3 slides per page at 1280px, 2 pagination dots, and slides labelled `1 of 5` and so on.

- `<Splide>` renders the root, the track and the list. Each `<SplideSlide>` is one `<li class="splide__slide">`. Put any content inside a slide.
- Options go in the `options` prop (all of them are in `references/options.md`). An inline object literal is fine: on re-render the component deep-compares the new options with the previous ones and does nothing when they are equal.
- Label every carousel. Prefer `options.label` (or `options.labelledby` with the ID of a visible heading) over the `aria-label` prop: in development with StrictMode, Splide's `destroy()` strips an `aria-label`, `aria-labelledby` or `role` passed as a prop and React doesn't put it back (tested). Options survive the remount. Production is unaffected either way.
- `tag="section"` gives the root an implicit `region` landmark. With the default `<div>`, Splide adds `role="region"` itself. For a decorative carousel use `options={ { role: 'group', label: '...' } }`. Details: `references/structure.md`.

---

## 5. When the instance exists

`Splide.componentDidMount()` (`src/js/components/Splide/Splide.tsx`):

```js
this.splide = new SplideCore( rootElement, options );
this.bind( this.splide );                 // event props, see below
this.splide.mount( extensions, transition );
```

So there is no selector, no `DOMContentLoaded` and no script order to manage. Each `<Splide>` creates exactly one core instance on the element it rendered.

| Moment | What happens |
|---|---|
| Render | Plain markup. The instance doesn't exist yet. |
| `componentDidMount` | `new Splide()`, event props bound, `mount()`. `mounted` and `ready` fire here. |
| Parent's `useEffect` | Runs after the child mounted, so `ref.current?.splide` is set |
| Re-render | Options deep-compared and applied with `splide.options = options`. If the list's child elements changed, `splide.refresh()`. See `references/reactivity.md`. |
| `componentWillUnmount` | `splide.destroy()` (completely), then `ref.current.splide` is `undefined` |
| StrictMode (development only) | Mount, destroy, mount again on the same DOM node. `onMounted` fires twice (tested). |

Event props are bound before `mount()`, so `onMounted`, `onReady`, `onArrowsMounted`, `onPaginationMounted` and `onNavigationMounted` all fire. A listener added later with `splide.on()` in a `useEffect` misses `mounted` and `ready` (they fire only once per mount) and the first `arrows:mounted` / `pagination:mounted` (those fire again after an options update, and `pagination:mounted` also after `refresh()`). Event props are captured at mount and never re-read, so a handler that reads state sees the first render's values: `references/events.md` and `references/reactivity.md`.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function RefCarousel() {
  const ref = useRef<Splide>( null );
  const [ index, setIndex ] = useState( 0 );
  const [ mountedCount, setMountedCount ] = useState( 0 );

  useEffect( () => {
    // Children mount before the parent's effects run, so the instance exists here.
    const splide = ref.current?.splide;

    if ( ! splide ) {
      return;
    }

    // Too late for 'mounted' or 'ready': they fired inside componentDidMount.
    // A namespace keeps off() from removing the handlers bound by event props.
    splide.on( 'moved.status', ( newIndex: number ) => setIndex( newIndex ) );

    return () => {
      splide.off( 'moved.status' );
    };
  }, [] );

  return (
    <>
      <Splide
        ref={ ref }
        options={ { label: 'Ref example', speed: 0 } }
        onMounted={ () => setMountedCount( count => count + 1 ) }
      >
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
        <SplideSlide>03</SplideSlide>
      </Splide>

      <button type="button" className="next" onClick={ () => ref.current?.go( '>' ) }>
        Next
      </button>
      <p className="status">Slide { index + 1 }, mounted { mountedCount }</p>
    </>
  );
}
```

- `useRef<Splide>( null )` types the ref as the component class. `ref.current.splide` is the core instance. The component also has `go( control )` and `sync( splide )` shortcuts.
- Read `ref.current?.splide` inside each effect run. Don't keep the instance from an earlier run: under StrictMode the first instance is destroyed and replaced.
- `splide.off( 'moved' )` removes every `moved` handler registered without a namespace, including the one the `onMoved` prop registered (tested: the prop stopped firing, a namespaced listener kept firing). Namespaces match exactly, so register your own listeners with one (`'moved.status'`) and remove only that namespace.
- `speed: 0` only makes the test deterministic. Leave it out in real code.

---

## 6. Errors thrown on mount

All core assertion messages start with `[splide]`. They are thrown from `componentDidMount`, so an error boundary above `<Splide>` catches them. Without one, React unmounts the whole root. Each row below was reproduced with `<Splide>`.

| Message | Cause in React |
|---|---|
| `[splide] A track/list element is missing.` | `hasTrack={ false }` without a `<SplideTrack>` (or a hand-written `.splide__track` with a `.splide__list` as its direct child) |
| `[splide] Invalid JSON` | The `data-splide` attribute holds invalid JSON (section 9) |
| `[splide] height or heightRatio is missing.` | `direction: 'ttb'` without `height` or `heightRatio` |
| `TypeError: Cannot read properties of undefined (reading 'style')` | Custom `.splide__arrow--prev` / `--next` buttons without a `.splide__arrows` wrapper. See `references/structure.md`. |
| `[splide] Already mounted!` | Only if you call `ref.current.splide.mount()` yourself. The component mounts once. |
| `[splide] null is invalid.` | Not reachable: the component passes its own root element, never a selector |

A child of `<Splide>` that isn't a `<SplideSlide>` doesn't throw: with the default `hasTrack`, it lands inside the `<ul>` (tested: an `<h2>` became the list's first child). It doesn't count as a slide, but it breaks the layout. Put headings and controls outside `<Splide>`, or use `hasTrack={ false }`.

---

## 7. Several carousels on one page

Each `<Splide>` owns one instance, so render as many as you need. Map over data with a stable `key`, and wrap the repeated markup in your own component:

```tsx
import { useId } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Shared defaults: the React counterpart of Splide.defaults.
const carouselDefaults: Options = {
  perPage: 3,
  gap    : '1rem',
  i18n   : {
    prev : 'Vorherige Folie',
    next : 'Nächste Folie',
    first: 'Zur ersten Folie',
    last : 'Zur letzten Folie',
  },
};

type ProductCarouselProps = {
  title: string;
  items: string[];
  options?: Options;
};

function ProductCarousel( { title, items, options }: ProductCarouselProps ) {
  const headingId = useId();

  return (
    <div className="product-carousel">
      <h2 id={ headingId }>{ title }</h2>
      <Splide options={ { ...carouselDefaults, ...options, labelledby: headingId } }>
        { items.map( item => (
          <SplideSlide key={ item }>{ item }</SplideSlide>
        ) ) }
      </Splide>
    </div>
  );
}

const shelves = [
  { title: 'New arrivals', items: [ 'A1', 'A2', 'A3', 'A4', 'A5' ] },
  { title: 'Best sellers', items: [ 'B1', 'B2', 'B3', 'B4', 'B5' ], options: { perPage: 2 } },
  { title: 'On sale', items: [ 'C1', 'C2', 'C3', 'C4', 'C5' ], options: { type: 'loop' } },
];

export default function Shelves() {
  return (
    <>
      { shelves.map( shelf => (
        <ProductCarousel key={ shelf.title } { ...shelf } />
      ) ) }
    </>
  );
}
```

Tested: three instances with IDs `splide01`, `splide02`, `splide03`, each with its own `perPage`, the German arrow labels and an `aria-labelledby` pointing at its heading. The loop carousel's prev arrow reads "Zur letzten Folie", because in `loop` mode the first slide's prev arrow uses i18n `last`.

Notes:

- Per-carousel options: spread the shared object and add or override keys (`{ ...carouselDefaults, ...options }`). The spread is shallow. If both levels set a nested object (`i18n`, `breakpoints`, `classes`), the inner one replaces the outer one completely, so spread those too (`i18n: { ...carouselDefaults.i18n, ...options.i18n }`). Splide itself deep-merges whatever you pass over its built-in defaults.
- `useId()` gives each instance a unique heading ID for `labelledby`.
- Each root without an `id` gets `splide01`, `splide02`, and so on, in mount order. A root with an `id` prop keeps it and doesn't use up a number (tested). Track, list and slide IDs derive from it (`splide01-track`, `splide01-slide01`). Under StrictMode the remount reuses the ID already on the element.
- The ID counter lives inside React Splide's copy of core. If the same page also runs the vanilla `@splidejs/splide` build (or a second bundled copy), each copy counts from `splide01`, so two roots can get the same ID (from source: `utils/string/uniqueId`). Give the roots explicit `id` props in that case.
- Nested carousels (a `<Splide>` inside a `<SplideSlide>`): React mounts children first, so the inner carousel mounts before the outer one. Splide checks only the first match of each control selector, so the outer carousel ignores its own `.splide__arrows` or `.splide__pagination` placeholders if they come after its track (tested). Put the outer controls before its `<SplideTrack>`. See `references/structure.md`.
- Thumbnails: render two `<Splide>` components, then in a `useEffect` read `const thumbs = thumbsRef.current?.splide` and call `mainRef.current?.sync( thumbs )` when it is set. See `references/tutorials/thumbnail-carousel.md`.

---

## 8. Shared defaults

`Splide.defaults` doesn't work with React Splide. The component uses the core class inlined in its own bundle, and that class isn't exported, so setting defaults on the class from `@splidejs/splide` changes a different class:

```tsx
import { Splide as SplideCore } from '@splidejs/splide';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Has no effect on <Splide>: React Splide bundles its own copy of the core class.
SplideCore.defaults = { perPage: 3 };

export default function DefaultsIgnored() {
  return (
    <Splide options={ { label: 'Defaults test' } }>
      <SplideSlide>01</SplideSlide>
      <SplideSlide>02</SplideSlide>
      <SplideSlide>03</SplideSlide>
      <SplideSlide>04</SplideSlide>
    </Splide>
  );
}
```

Tested: the carousel still shows one slide per page. The runtime import also adds a second copy of core (about 30 KB minified) to the bundle. `import type` from `@splidejs/splide` is free, but `import type { Options } from '@splidejs/react-splide'` is the safer source.

Share defaults the React way instead, as in section 7:

- a module-level `Options` object that every carousel spreads, or
- a wrapper component (`ProductCarousel` above) that owns the defaults and takes overrides as a prop.

---

## 9. Options in the `data-splide` attribute

The core constructor reads `data-splide` from the root, and `<Splide>` passes unknown attributes to the root, so this works:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DataAttributeCarousel() {
  const [ gap, setGap ] = useState( '1rem' );

  return (
    <>
      <Splide
        data-splide='{"perPage":2}'
        options={ { label: 'Data attribute', perPage: 3, gap } }
      >
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
        <SplideSlide>03</SplideSlide>
        <SplideSlide>04</SplideSlide>
        <SplideSlide>05</SplideSlide>
      </Splide>

      <button type="button" className="wider" onClick={ () => setGap( '2rem' ) }>
        Wider gap
      </button>
    </>
  );
}
```

Tested behaviour:

- At construction, `data-splide` wins over the `options` prop, the same precedence as vanilla (`label`/`labelledby` from the root's attributes, then the built-in defaults, then `options`, then `data-splide`). The carousel above starts with 2 slides per page, not 3.
- After the first change to the `options` prop (here the gap), the component calls `splide.options = options` with the whole prop, so every key in the prop replaces what `data-splide` set. The carousel switched to 3 per page.
- The attribute is read once. Changing it after mount does nothing.
- Invalid JSON, or an empty `data-splide=""`, throws `[splide] Invalid JSON`. JSON can't hold functions, so `easingFunc` and similar options must go in the prop.
- TypeScript doesn't check `data-*` attributes, so a typo in the JSON is only found at runtime.

Use the `options` prop. It is typed, it is reactive, and it has a single source of truth. `data-splide` is useful only when the options arrive as a JSON string, for example from a CMS: `data-splide={ jsonFromCms }`, or better `options={ JSON.parse( jsonFromCms ) }`.

---

## 10. Vite and Next.js

- Vite: nothing special. Import the components and one CSS subpath, as above. All examples in this skill were built with Vite and `@vitejs/plugin-react`.
- TypeScript (Vite, Next.js, any project with `moduleResolution: "bundler"`, `node16` or `nodenext`): the imports are untyped (`any`) until you add one tsconfig line. See `references/typescript.md`.
- Next.js App Router: mark the component that renders `<Splide>` with `'use client'`, and import the CSS once. Server-rendered carousel markup stays hidden until hydration mounts it. See `references/nextjs.md`.

---

## 11. Vanilla snippets converted to React

| Vanilla (CDN or bundler) | React Splide |
|---|---|
| `<link rel="stylesheet" href=".../dist/css/splide.min.css">` or `import '@splidejs/splide/css'` | `import '@splidejs/react-splide/css'` |
| `.../dist/css/themes/splide-skyblue.min.css` | `import '@splidejs/react-splide/css/skyblue'` |
| `.../dist/css/splide-core.min.css` | `import '@splidejs/react-splide/css/core'` |
| `<section class="splide" aria-label="..."> <div class="splide__track"> <ul class="splide__list"> <li class="splide__slide">` | `<Splide tag="section" options={ { label: '...' } }> <SplideSlide>` |
| `new Splide( '.splide', options ).mount()` | `<Splide options={ options }>` (mounts in `componentDidMount`) |
| `document.addEventListener( 'DOMContentLoaded', ... )` | Not needed |
| `splide.on( 'moved', fn )` before `mount()` | `onMoved={ ( splide, newIndex, prevIndex, destIndex ) => ... }` (instance first). Names: `references/events.md`. |
| `var splide = new Splide( ... ).mount(); splide.go( '>' )` | `const ref = useRef<Splide>( null )`, then `ref.current?.go( '>' )` or `ref.current?.splide?.go( '>' )` |
| `new Splide( el ).mount( { Grid } )` | `<Splide extensions={ { Grid } }>` |
| `new Splide( el ).mount( {}, MyTransition )` | `<Splide transition={ MyTransition }>` |
| `main.sync( thumbs ); main.mount(); thumbs.mount();` | `mainRef.current?.sync( thumbsRef.current.splide )` in a `useEffect` |
| `Splide.defaults = { ... }` | A shared options object or a wrapper component (section 8) |
| `splide.options = { perPage: 2 }` | Change the `options` prop (state) |
| `splide.add( slide )` / `splide.remove( i )` | Render a different list of `<SplideSlide>` children (the component calls `refresh()`) |
| `splide.destroy()` | Unmount the component, or `options.destroy` in a breakpoint |
| `import { EventInterface } from '@splidejs/splide'` in an extension | Works, but adds a second copy of core. Inside an extension you can use `Splide.on( 'move.myExt', fn )` and `Splide.off( 'move.myExt' )` on the instance it receives. See `references/custom-extensions.md`. |
