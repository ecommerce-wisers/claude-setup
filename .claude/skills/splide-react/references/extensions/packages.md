# Official extensions: npm packages

Package names, versions, exports, CSS and type imports for the five official extensions with React Splide, and how the `extensions` prop mounts them.

Contents: [Packages](#packages) · [Install and import](#install-and-import) · [Package layout](#package-layout) · [The extensions prop](#the-extensions-prop) · [Options and events](#options-and-events) · [TypeScript](#typescript) · [Combined example](#combined-example-auto-scroll--intersection)

## Packages

The versions below were checked against the npm registry `latest` tags on 2026-11-25. No extension has had a release since 2022.

| Extension | Package @ latest | Released | Named export | Exported types | CSS |
|---|---|---|---|---|---|
| Auto Scroll | `@splidejs/splide-extension-auto-scroll@0.5.3` | 2022-09-06 | `AutoScroll` | `AutoScrollComponent` | none |
| Intersection | `@splidejs/splide-extension-intersection@0.2.0` | 2022-04-05 | `Intersection` | none | none |
| Video | `@splidejs/splide-extension-video@0.8.0` | 2022-09-05 | `Video` | `VideoComponent` | `@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css` (required) |
| Grid | `@splidejs/splide-extension-grid@0.4.1` | 2022-04-06 | `Grid` | none | none |
| URL Hash | `@splidejs/splide-extension-url-hash@0.3.0` | 2022-04-05 | `URLHash` (capital `URL`) | none | none |

- Every package has only a **named** export. There is no default export: `import AutoScroll from '...'` passes `tsc` but the Vite build fails with `"default" is not exported by ".../splide-extension-auto-scroll.esm.js"` (tested).
- None of them declares `peerDependencies`. Each lists `@splidejs/splide` only as a devDependency: Auto Scroll `^4.0.17`, Intersection `^4.0.0`, Video `^4.0.8`, Grid `^4.0.1`, URL Hash `^4.0.0`. All five are v4 extensions. The lines built for Splide v3 are older: Auto Scroll ≤ 0.3.7, Intersection 0.1.x, Video ≤ 0.6.8, Grid 0.3.x, URL Hash 0.2.x. Don't install those.
- They work with the copy of core 4.1.3 that React Splide 0.7.12 bundles. An extension is a plain function that receives the Splide instance, so it doesn't matter which core build created it (all five tested with `<Splide>` in headless Chrome, React 19, production and development with StrictMode).
- The only runtime dependency is `@vimeo/player` (Video). It is bundled into Video's dist files, so the copy npm installs next to it is used only for types.

## Install and import

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-auto-scroll @splidejs/splide-extension-intersection @splidejs/splide-extension-video @splidejs/splide-extension-grid @splidejs/splide-extension-url-hash
```

Install only the ones you use. All imports in one place:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import type { AutoScrollComponent } from '@splidejs/splide-extension-auto-scroll';
import { Intersection } from '@splidejs/splide-extension-intersection';
import { Video } from '@splidejs/splide-extension-video';
import type { VideoComponent } from '@splidejs/splide-extension-video';
import { Grid } from '@splidejs/splide-extension-grid';
import { URLHash } from '@splidejs/splide-extension-url-hash';

import '@splidejs/react-splide/css';                                               // core theme
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css'; // only with Video
```

The other four ship no CSS. Video's stylesheet positions the player and draws the play button; without it the overlay has no layout. Import it once, in the same place as the core CSS (in Next.js, in a layout or the client component).

For plain HTML pages there are CDN builds that register on `window.splide.Extensions`. With a bundler you don't need them, and you don't need `window.splide` at all.

## Package layout

Each `package.json` has `main` (`dist/js/<name>.cjs.js`), `module` (`dist/js/<name>.esm.js`), `types` (`dist/types/index.d.ts`) and `jsdelivr`, and **no `exports` field**:

- Bundlers (Vite, webpack, Next.js) use the `module` build. Node uses `main`.
- Deep imports work, which is how the Video CSS path above resolves.
- The ESM builds have no `import` statements: each bundles the few core helpers it needs instead of importing `@splidejs/splide`. So installing an extension doesn't add a second copy of core.
- **SSR.** Importing any of the five on the server is safe: none of them touches `window` or `document` at import time (checked by requiring each package in Node). They only run inside `<Splide>`'s `componentDidMount`, which never runs on the server. `renderToString` output contains your `data-splide-*` attributes, unchanged.
- **Next.js App Router.** React Splide has no `'use client'` directive, so the file that renders `<Splide>` must be a client component (see `references/nextjs.md`). Import the extensions in that same client file. They are functions, so a server component can't pass them down as props.

## The extensions prop

```tsx
const extensions = { AutoScroll, Intersection }; // module scope: read once, at mount

<Splide options={ options } extensions={ extensions }>...</Splide>
```

- `<Splide>` calls `splide.mount( extensions, transition )` in `componentDidMount`. The prop is **read once**. Adding, removing or swapping an extension later does nothing. To change the set, remount the carousel with a new `key`.
- An inline object (`extensions={ { AutoScroll } }`) works the same, because a new object on each render is ignored after mount. A module-scope constant just makes that obvious.
- The keys become `splide.Components.<Key>`. Keep the standard keys `AutoScroll`, `Intersection`, `Video`, `Grid` and `URLHash`: Intersection finds AutoScroll and Video by those names, and a key that matches a core component (`Scroll`, `Slides`, `Layout`, ...) replaces that component.
- Extensions mount after all core components, in the key order of the object. The order matters for Video with Grid (`references/extensions/grid.md`) and for URL Hash's first hash write (`references/extensions/url-hash.md`).
- Give each carousel only the extensions it needs. Two of them act with no options at all: AutoScroll starts scrolling unless `autoScroll: false` (or `type: 'fade'`), and URLHash clears the URL fragment when the active slide has no `data-splide-hash`. Intersection does nothing without `inView`/`outView`, Video only touches slides with video attributes, and Grid only acts with a `grid` option.
- `render()` doesn't strip the prop, so the root element gets an `extensions="[object Object]"` attribute (also in server-rendered HTML). It is harmless.

## Options and events

- Options go under their keys in the `options` prop: `autoScroll`, `intersection`, `video`, `grid`. URL Hash has no options.
- When the `options` prop changes (deep comparison), React Splide sets `splide.options = options`, which emits `updated`. **Auto Scroll** and **Grid** react to `updated` (new speed, grid on or off). **Intersection** reads its options once at mount, and **Video** reads them when each player is created, so later changes to those two have no effect unless you remount.
- **Never put a DOM element in `options`.** React Splide deep-copies the options at mount (and compares them deeply on every update), and copying an element that React renders recurses into React's internal fiber objects: the app crashes with "Maximum call stack size exceeded" (tested with Intersection's `root`). See `references/extensions/intersection.md` for the wrapper that passes an element safely.
- **Events.** React Splide has no event props for extension events (`intersection`, `intersection:in`, `intersection:out`, `video:play`, `video:pause`, `video:ended`, `video:error`, `video:click`). Auto Scroll, Grid and URL Hash emit no events of their own. Register with `splide.on()` in `onMounted`, which runs inside `mount()` and receives the instance first:

  ```tsx
  <Splide
    extensions={ extensions }
    onMounted={ splide => {
      splide.on( 'intersection', entry => setVisible( entry.isIntersecting ) );
    } }
  >
  ```

  Handlers registered on the instance are removed when it is destroyed (unmount, or StrictMode's test unmount in development), and the remount calls `onMounted` again for the new instance. Like every event prop, `onMounted` is captured at mount, so use stable state setters or a ref for changing values (see `references/reactivity.md`).
- Registering in a `useEffect` through `ref.current?.splide` also caught the first `intersection` event in tests, but effects run after paint, while `onMounted` is guaranteed to run before any extension event. Events that fire synchronously during mount (Grid's `refresh`, the first `active`) are seen only by event props, which React Splide binds before `mount()`.
- Core events caused by extensions (`active`, `refresh`, `resized`, `autoplay:play`, ...) do have props: use them directly.

## TypeScript

- Each package's `types` field points at its declarations. Importing an extension augments core's `Options` (typed `autoScroll`, `intersection`, `video`, `grid`) and `Components` (`Components.AutoScroll?`, `Components.Video?`, ...). Because React Splide's `Options` is core's type, `import type { Options } from '@splidejs/react-splide'` picks the augmentations up. Tested: `speed: 'fast'`, `video: { mute: 'yes' }`, `grid: { rows: '2' }` and the typo `inView: { autoScoll: true }` are all compile errors.
- The packages export no options types. Derive them from `Options` (tested):

  ```ts
  type AutoScrollOptions = Exclude<NonNullable<Options['autoScroll']>, boolean>;
  type VideoOptions = NonNullable<Options['video']>;
  type GridOptions = Exclude<NonNullable<Options['grid']>, false | null>;
  type IntersectionOptions = NonNullable<Options['intersection']>;
  ```

- Component types: `AutoScrollComponent` and `VideoComponent` are exported (`import type`). Reach the instances as `ref.current?.splide?.Components.AutoScroll` (typed `AutoScrollComponent | undefined`).
- For your own extensions and wrappers, type the constructor without importing core: `type ExtensionConstructor = NonNullable<SplideProps['extensions']>[ string ];` (`SplideProps` comes from `@splidejs/react-splide`).
- React Splide's own types need one tsconfig line under `moduleResolution: "bundler"` (the Vite and Next.js default). See `references/typescript.md`.
- Keep `skipLibCheck: true`. With `false`, Video's declarations fail as well: "Cannot find type definition file for 'youtube'", "Cannot find namespace 'YT'", and "Module '@vimeo/player' has no exported member 'Options'" (the `^2.17.1` range installs a newer `@vimeo/player` whose types changed). Without `@types/youtube`, `video.playerOptions.youtube` is untyped (any key is accepted).
- **URL Hash on case-sensitive file systems** (Linux CI, Docker): its `dist/types/extensions/index.d.ts` re-exports `./URLHash/URLHash`, but the folder is named `UrlHash`. On Linux the `URLHash` import is silently typed `any` (with `skipLibCheck: true`), or TypeScript reports "Cannot find module './URLHash/URLHash'" (with `false`). macOS and Windows resolve it. The runtime import is unaffected. Checked by running the TypeScript compiler API with a case-sensitive file lookup.

## Combined example: Auto Scroll + Intersection

A logo ticker that scrolls only while at least 25% of it is on screen. It pauses on hover and focus, has the core toggle button, respects reduced motion, and shows its visibility in React state.

```tsx
import { useState } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';
import { Intersection } from '@splidejs/splide-extension-intersection';
import '@splidejs/react-splide/css';

// False on the server, where the options are never used.
const reduce = typeof window !== 'undefined'
  && window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches;

const options: Options = {
  label      : 'Our partners',
  height     : '6rem',
  type       : 'loop',
  drag       : 'free',
  focus      : 'center',
  perPage    : 4,
  gap        : '1rem',
  arrows     : false,
  pagination : false,
  breakpoints: {
    640: { perPage: 2 },
  },
  autoScroll: {
    speed          : 1,
    autoStart      : false, // Intersection starts it
    useToggleButton: true,
  },
  intersection: {
    threshold: 0.25,
    inView   : { autoScroll: ! reduce },
    outView  : { autoScroll: false },
  },
};

// Module scope: the object is read once, at mount.
const extensions = { AutoScroll, Intersection };

const partners = [ 'Partner 01', 'Partner 02', 'Partner 03', 'Partner 04', 'Partner 05', 'Partner 06' ];

export default function PartnerTicker() {
  const [ visible, setVisible ] = useState( false );

  return (
    <>
      <div style={ { height: '110vh' } }>Scroll down</div>

      <Splide
        tag="section"
        id="ticker"
        hasTrack={ false }
        options={ options }
        extensions={ extensions }
        onMounted={ splide => {
          // No event prop exists for extension events. setVisible is stable, so no stale closure.
          splide.on( 'intersection', entry => setVisible( entry.isIntersecting ) );
        } }
      >
        <SplideTrack>
          { partners.map( name => (
            <SplideSlide key={ name } style={ { display: 'grid', placeItems: 'center', background: '#f1f3f5' } }>
              { name }
            </SplideSlide>
          ) ) }
        </SplideTrack>
        <button className="splide__toggle" type="button">
          <span className="splide__toggle__play">Play</span>
          <span className="splide__toggle__pause">Pause</span>
        </button>
      </Splide>

      <p id="status">{ visible ? 'Ticker on screen' : 'Ticker off screen' }</p>
      <div style={ { height: '110vh' } } />
    </>
  );
}
```

- `hasTrack={ false }` with `<SplideTrack>` lets the toggle button sit inside the root, where Splide looks for it.
- The slide height comes from the `height` option. Splide sets each slide's inline `width` and `height` in its layout pass, so a `height` in a `<SplideSlide style>` is overwritten. Other inline styles stay.
- Tested in headless Chrome (production and development with StrictMode): off screen at load, no scrolling; scrolled into view, scrolling starts, the toggle reads "Pause auto scroll" and the status updates; clicking the toggle pauses.

Details for each extension: `references/extensions/auto-scroll.md`, `references/extensions/intersection.md`, `references/extensions/video.md`, `references/extensions/grid.md`, `references/extensions/url-hash.md`.
