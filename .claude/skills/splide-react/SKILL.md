---
name: splide-react
description: "Build, configure or debug Splide v4 carousels in React or Next.js with @splidejs/react-splide: the Splide, SplideTrack and SplideSlide components, options and event props, refs and the core API, thumbnail sync, official and custom extensions, custom transitions, TypeScript setup, server rendering and accessibility. Use for any React, Next.js or Vite project that uses Splide."
---

# Splide v4 in React (`@splidejs/react-splide`)

React Splide **0.7.12** (the latest release, September 2022) wraps Splide core. Its build bundles its own copy of core 4.1.3, which behaves exactly like 4.1.4 (only `package.json` changed between them). Options, events and the `Components` API are core's; React Splide adds the components, the `options` prop, event props and a ref. Tested with React 19.3 (the examples need 16.8+ for hooks) and Next.js 16.4. Docs: https://splidejs.com/integration/react-splide/

This file is the quick reference. The full docs, converted to React and tested in headless Chrome in production and under StrictMode, are in `references/`. Read the matching file before writing anything beyond the basics (see [Reference index](#reference-index)).

## Workflow

1. Install `@splidejs/react-splide` (and any extension packages), and import one CSS file once.
2. TypeScript: add `"resolvePackageJsonExports": false` to `tsconfig.json`. Without it the types don't resolve and `next build` fails (see [TypeScript](#typescript)).
3. Next.js App Router: render `<Splide>` from a file marked `'use client'`.
4. Render `<Splide>` > `<SplideSlide>`. When anything besides slides goes inside the root (arrows, toggle, progress bar), use `hasTrack={ false }` with `<SplideTrack>`.
5. Pass options through `options`. Name the carousel with `options.label` or `options.labelledby`, not `aria-label` props (StrictMode drops those in development).
6. Use an event prop when one exists (exact names in [Event props](#event-props)). Remember they capture the first render's values and run before Splide's own handlers. For other events, use `splide.on()` in an extension's `setup()` or after mount.
7. Reach the core instance with `ref.current?.splide`, in effects and handlers only.
8. Run the [accessibility checklist](#accessibility-checklist).

## Install

```bash
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

| CSS import | File |
|---|---|
| `@splidejs/react-splide/css` | Default theme (`splide.min.css`) |
| `@splidejs/react-splide/css/sea-green`, `/css/skyblue`, `/css/default` | Themes (`dist/css/themes/splide-<name>.min.css`) |
| `@splidejs/react-splide/css/core` | Layout only, no arrow, dot or progress visuals |

- The CSS files are identical to core 4.1.4's. Only `.` and the `/css` subpaths are exported, so deep imports such as `@splidejs/react-splide/dist/...` fail in Vite.
- No `peerDependencies`, so React 18 and 19 install without warnings. No `'use client'` directive in the build.
- `@splidejs/splide` is installed as a dependency, but React Splide uses its own bundled copy. Import core only with `import type`: a runtime import adds a second core (+30.6 kB minified, +12 kB gzip, measured with Vite), and that copy's `Splide.defaults` doesn't reach `<Splide>`.
- Install details, the package layout and a vanilla-to-React conversion table: `references/getting-started.md`. Themes: `references/themes.md`.

## TypeScript

React Splide's `exports` map has no `types` condition, so under `moduleResolution: "bundler"` (the Vite and Next.js default), `node16` or `nodenext`, every import is an implicit `any` (`TS7016`), and `next build` stops at type-checking. Fix:

```json
{ "compilerOptions": { "resolvePackageJsonExports": false } }
```

Don't use a tsconfig `paths` entry pointing at the `.d.ts`: Next.js (and Vite with tsconfig paths enabled) applies `paths` at runtime too, and the page crashes with "Element type is invalid ... got: undefined". Keep `skipLibCheck: true`. Typing refs, handlers and extensions without importing core: `references/typescript.md`.

## Markup

```tsx
'use client'; // Next.js App Router only
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label      : 'Featured products',
  perPage    : 3,
  gap        : '1rem',
  rewind     : true,
  breakpoints: { 1024: { perPage: 2 }, 640: { perPage: 1 } },
};

export default function ProductCarousel( { products }: { products: { id: string; image: string; title: string }[] } ) {
  return (
    <Splide tag="section" options={ options }>
      { products.map( p => (
        <SplideSlide key={ p.id }>
          <img src={ p.image } alt={ p.title } />
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

That renders `<section class="splide">` > `<div class="splide__track">` > `<ul class="splide__list">` > `<li class="splide__slide">`. Splide then adds the arrows, pagination, classes and ARIA.

| `<Splide>` prop | Type | Default | Notes |
|---|---|---|---|
| `options` | `Options` | none | Deep-compared on every render; a change goes through `splide.options = ...` |
| `hasTrack` | `boolean` | `true` | `false`: render `<SplideTrack>` yourself, so other elements can sit beside it |
| `tag` | `'div' \| 'section' \| 'header' \| 'footer' \| 'nav'` | `'div'` | `section` for content carousels (Splide adds `role="region"` only to non-section roots) |
| `extensions` | `Record<string, ComponentConstructor>` | none | Read once at mount. Also rendered as an `extensions="[object Object]"` attribute |
| `transition` | `ComponentConstructor` | none | Read once at mount. Logs "Invalid value for prop `transition`" in development |
| event props | see [Event props](#event-props) | none | Bound once at mount |
| `className`, `id`, `style`, `data-*`, other HTML attributes | | | Go to the root. React's DOM event handlers aren't typed; `onClick` is Splide's slide click |

Everything you put inside a default `<Splide>` renders inside `<ul class="splide__list">`. Arrows, a placeholder, a toggle or a progress bar need `hasTrack={ false }`:

```tsx
<Splide hasTrack={ false } tag="section" options={ { label: 'Hero', type: 'loop', autoplay: true } }>
  <div className="custom-wrapper">
    <SplideTrack>
      { slides.map( s => <SplideSlide key={ s.id }>{ s.content }</SplideSlide> ) }
    </SplideTrack>

    <div className="splide__arrows">
      <button className="splide__arrow splide__arrow--prev" type="button">Prev</button>
      <button className="splide__arrow splide__arrow--next" type="button">Next</button>
    </div>

    <div className="splide__progress"><div className="splide__progress__bar" /></div>

    <button className="splide__toggle" type="button">
      <span className="splide__toggle__play">Play</span>
      <span className="splide__toggle__pause">Pause</span>
    </button>
  </div>
</Splide>
```

- Custom arrows must be `<button type="button">` inside a `div.splide__arrows` wrapper. Bare `.splide__arrow` buttons make `componentDidMount` throw, which unmounts the whole tree without an error boundary. Splide finds elements by the default class names only and overwrites the arrows' `aria-label`.
- Attributes on `<SplideSlide>` go to the `<li>`: `data-splide-interval`, `data-splide-hash`, `data-splide-youtube`, and `data-splide-lazy` on an `<img>` inside it.
- Keep `className` on `<Splide>` and `<SplideSlide>` constant. A change rewrites the class attribute, drops `is-initialized` and the carousel turns invisible.
- A nested `<Splide>` mounts before its parent, so put the parent's own controls before its `<SplideTrack>`.
- Full DOM, every attribute Splide writes and what survives a re-render: `references/structure.md`.

## Mount and lifecycle

| React | React Splide |
|---|---|
| `componentDidMount` | `new Splide( root, options )`, bind the event props, `mount( extensions, transition )` |
| `componentDidUpdate` | `splide.options = options` if they changed (deep compare); `refresh()` if the list's children changed |
| `componentWillUnmount` | `splide.destroy()` |

- The instance only exists in the browser, after mount: `ref.current?.splide` is `undefined` in the first render and after unmount.
- StrictMode (development) mounts, destroys and mounts again. Mount-time props fire twice, and props that `destroy()` strips (`aria-label`, `aria-labelledby`, `role`, `tabIndex`, `style`) stay stripped. Use `options.label`, `labelledby` and `role`.
- How props, children and state changes reach the carousel, and the key-remount pattern: `references/reactivity.md`.

## Options (most used)

Every option with type, default, notes and examples: `references/options.md`. "Live" says what a change through the `options` prop does after mount: **Yes** (applied), **Next use** (applied the next time Splide reads it), **Remount** (change the `key` on `<Splide>`).

| Option | Default | Live | Notes |
|---|---|---|---|
| `type` | `'slide'` | Remount | `'loop'`, `'fade'` (fade ignores `perPage`). A live change breaks the carousel |
| `perPage` / `perMove` | `1` / unset | Yes | integers |
| `gap`, `padding` | unset | Yes | CSS units; padding `{ left, right }` or `{ top, bottom }` |
| `focus` | unset | Yes | number or `'center'` |
| `trimSpace` | `true` | Next use | `false` / `'move'` |
| `omitEnd` | `false` | Remount | v4.1+. Docs say it needs `type: 'slide'` + `focus`; without `focus` the end arrow is still disabled but the dots aren't trimmed |
| `start` | `0` | Remount | |
| `rewind`, `rewindByDrag` | `false` | Yes | not for loop |
| `speed` / `rewindSpeed` | `400` / `speed` | Yes | ms |
| `width`, `height`, `heightRatio` | unset | Yes | height: no `%`. `heightRatio` multiplies the list width, not the slide width |
| `fixedWidth`, `fixedHeight` | unset | Yes | layout only; `perPage` still sets the step and page count |
| `autoWidth`, `autoHeight` | `false` | Next use | keep perPage/perMove = 1 |
| `direction` | `'ltr'` | Remount | `'rtl'`, `'ttb'` (ttb needs `height` or `heightRatio`) |
| `arrows`, `pagination` | `true` | Yes | `pagination: false` with `isNavigation` |
| `drag` | `true` | Yes | `'free'` + `snap: true` |
| `keyboard` | `false` | Yes | `true` / `'focused'` / `'global'` |
| `wheel` | `false` | Remount | + `releaseWheel`, `wheelMinThreshold`, `wheelSleep` |
| `autoplay` | `false` | Remount | `'pause'` = wait for `Autoplay.play()` |
| `interval` | `5000` | Next use | per slide: `data-splide-interval` |
| `pauseOnHover` / `pauseOnFocus` | `true` | Remount | |
| `lazyLoad` | `false` | Remount | `'nearby'` / `'sequential'`; `data-splide-lazy` on `<img>` |
| `isNavigation` | `false` | Remount | thumbnails, with `sync()` |
| `clones` | auto | Yes | loop only |
| `cover` | `false` | Remount | needs a height option |
| `live` | `true` | Remount | ARIA live region |
| `mediaQuery`, `breakpoints`, `reducedMotion` | `'max'` / unset / `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Remount | breakpoints merge like CSS media queries |
| `destroy` | `false` | breakpoints only | `'completely'` strips the label and role too |
| `classes` | defaults | Partly | a value **replaces** the default class, so include `splide__...` yourself |
| `i18n` | defaults | Partly | pagination updates at once, arrows never: remount with `key={ locale }`. See `references/i18n.md` |
| `label` / `labelledby` | unset | Yes | use these instead of aria props |
| `role` | `'region'` | Remount | `'group'` for decorative carousels |

- Inline `options={ { ... } }` objects are fine: equal content does nothing. Functions inside options compare by identity.
- An update merges the whole prop: a removed key keeps its old value, an explicit `undefined` replaces the default (`perPage: undefined` freezes the carousel, `i18n: undefined` throws), and every key also in a matching breakpoint overrides it until the next media query change.
- Never put a DOM element or `ref.current` in the options: the deep compare walks into React's fiber and overflows the stack.

## Ref and instance API

```tsx
const ref = useRef<Splide>( null );            // Splide is the component class

<Splide ref={ ref } options={ options }>...</Splide>

useEffect( () => {
  const splide = ref.current?.splide;          // core instance, after mount
  if ( ! splide ) return;
  console.log( splide.index, splide.length );
}, [] );

ref.current?.go( '>' );                        // component shortcuts: go( control ), sync( splide )
ref.current?.splide?.go( '+1' );               // '-1', '>' next page, '<' prev page, '>2' page index 2, 2 = slide index
ref.current?.splide?.Components.Autoplay.pause();
ref.current?.splide?.refresh();
ref.current?.splideRef.current;                // the root element
```

- The full core API (`go()` grammar, `options`, `is()`, `add()`/`remove()`, `refresh()`, `destroy()`, states) is the same as vanilla: `references/apis.md`.
- Don't call `splide.add()` or `splide.remove()` on React-rendered slides: React later throws `NotFoundError` and unmounts the tree. Render the slides from state.
- There's no `index` prop. To drive the carousel from state, call `go()` in an effect and mirror the index with `onMoved` (tested pattern in `references/reactivity.md`).

## Event props

Every handler gets the core instance first, then the event's arguments: `onMoved={ ( splide, newIndex, prevIndex, destIndex ) => ... }`. Full details, timing and every core event: `references/events.md`.

| Core event | Prop | Args after `splide` |
|---|---|---|
| `mounted`, `ready` | `onMounted`, `onReady` | none |
| `move`, `moved` | `onMove`, `onMoved` | `newIndex, prevIndex, destIndex` |
| `active`, `inactive`, `visible`, `hidden` | `onActive`, `onInactive`, `onVisible`, `onHidden` | `Slide` |
| `click` | `onClick` | `Slide, MouseEvent` |
| `updated` | `onUpdated` | `options` |
| `refresh`, `resize`, `resized`, `destroy` | `onRefresh`, `onResize`, `onResized`, `onDestroy` | none |
| `drag`, `dragging`, `dragged`, `scroll`, `scrolled` | `onDrag`, `onDragging`, `onDragged`, `onScroll`, `onScrolled` | none |
| `arrows:mounted`, `arrows:updated` | `onArrowsMounted`, `onArrowsUpdated` | `prev, next` (+ `prevIndex, nextIndex`) |
| `pagination:mounted`, `pagination:updated` | `onPaginationMounted`, `onPaginationUpdated` | `data` (+ `prev, curr`) |
| `navigation:mounted` | `onNavigationMounted` | `splides` (`{ splide, isParent? }[]`) |
| `autoplay:play`, `autoplay:pause`, `autoplay:playing` | `onAutoplayPlay`, `onAutoplayPause`, `onAutoplayPlaying` | `rate` 0 to 1 (playing only) |
| `lazyload:loaded` | `onLazyLoadLoaded` | `img, Slide` |

- That's all 28. Note the spelling `onLazyLoadLoaded`. There is **no `onOverflow`** and no prop for any extension event; an unknown name logs "Unknown event handler property" and is ignored.
- **Captured at mount.** Handlers keep the first render's props and state, and changing or adding a handler later does nothing. Read state through a ref written during render (`latest.current = value`), or set state with the setter.
- **They run before Splide's own handlers** for the same event: in `onUpdated` and `onRefresh`, `Controller.getEnd()` and `splide.length` still hold the old values. Register with `splide.on()` after mount when you need the updated state.
- **A breakpoint `destroy: true` drops them for good** (core clears every handler and React Splide never binds them again). Listen from an extension's `setup()` instead.
- **Other events:** register in an extension's `setup()` (sees every mount-time event, survives breakpoint remounts) or with `splide.on()` in an effect (misses the mount-time ones). An extension's `mount()` misses `overflow`, `resized`, `arrows:mounted`, `pagination:mounted`, `navigation:mounted` and `autoplay:play`.
- `splide.off( 'moved' )` without a namespace also removes the `onMoved` prop. Namespace your own listeners: `splide.on( 'moved.mine', fn )`.
- Don't `setState` in `onAutoplayPlaying` (it fires every frame); write to the DOM through a ref.

## Components (`splide.Components.X`)

Reached as `ref.current?.splide?.Components.X`, from the instance in an event prop, or from an extension. One file per component: `references/components/`.

- **Controller**: `go(control, allowSameIndex?, callback?)`, `scroll(destPx, duration?, snap?, cb?)`, `getNext()`, `getPrev()` (→ `-1` if none), `getEnd()`, `getIndex(prev?)`, `setIndex(i)`, `toIndex(page)`, `toPage(index)`, `isBusy()`
- **Slides**: `get(excludeClones?)`, `getAt(i)`, `getIn(page)`, `forEach(fn, excludeClones?)`, `filter()`, `style(prop, val, useContainer?)`, `getLength(excludeClones?)`, `isEnough()` (`add()`/`remove()` exist but break React)
- **Slide object**: `index`, `slideIndex` (-1 unless clone), `slide`, `container`, `isClone`, `style()`, `isWithin(from, distance)`
- **Elements**: `root`, `track`, `list`, `slides`, `arrows`, `prev`, `next`, `bar`, `toggle`
- **Move**: `move()`, `jump(i)`, `translate(pos)`, `cancel()`, `toIndex(pos)`, `toPosition(i, trim?)`, `getPosition()`, `getLimit(max)`, `exceededLimit()`
- **Layout**: `listSize()`, `slideSize(i, noGap?)`, `sliderSize()`, `totalSize(i?, noGap?)`, `getPadding(right)`, `isOverflow()`
- **Autoplay**: `play()`, `pause()`, `isPaused()` · **Drag**: `disable(bool)`, `isDragging()` · **Arrows**: `arrows.{prev,next}` · **Pagination**: `items`, `getAt(i)` · **Direction**: `resolve(prop)`, `orient(v)` · **Scroll**: `scroll()`, `cancel()`
- Without doc pages (Clones, Cover, Keyboard, LazyLoad, Live, Media, Sync, Wheel): `references/components/other-components.md`.

## Recipes

Complete, tested components: `references/tutorials/` (image carousel, thumbnail carousel, gallery, progress bar).

**Thumbnails (synced).** Both carousels are mounted by the time the parent's effect runs, and `sync()` after mount remounts the sync:

```tsx
const mainRef   = useRef<Splide>( null );
const thumbsRef = useRef<Splide>( null );

useEffect( () => {
  const thumbs = thumbsRef.current?.splide;
  if ( ! mainRef.current || ! thumbs ) return;

  mainRef.current.sync( thumbs );

  // refresh() (run by React Splide whenever the slides change) drops the thumbnails' ARIA; restore it
  thumbs.on( 'refresh.nav', () => { thumbs.emit( 'navigation:mounted', thumbs.splides ); } );
  return () => { thumbs.off( 'refresh.nav' ); };
}, [] );

const mainOptions: Options   = { label: 'Product images', type: 'fade', rewind: true, pagination: false, arrows: false };
const thumbsOptions: Options = {
  label: 'Thumbnails. Selecting one changes the product image.',
  fixedWidth: 100, fixedHeight: 60, gap: 10, rewind: true, pagination: false, isNavigation: true,
  breakpoints: { 600: { fixedWidth: 60, fixedHeight: 44 } },
};

<Splide ref={ mainRef } options={ mainOptions }>{ slides }</Splide>
<Splide ref={ thumbsRef } options={ thumbsOptions } className="thumbs">{ thumbs }</Splide>
```
```css
.thumbs .splide__slide { opacity: .6; }
.thumbs .splide__slide.is-active { opacity: 1; }
```

**Numbered pagination:**
```tsx
<Splide onPaginationMounted={ ( splide, data ) => {
  data.items.forEach( item => { item.button.textContent = String( item.page + 1 ); } );
} } />
```

**Hide controls when there aren't enough slides** (CSS; there's no `onOverflow`, see `references/overflow.md` for state-driven versions):
```css
.splide:not( .is-overflow ) .splide__arrows,
.splide:not( .is-overflow ) .splide__pagination { display: none; }
.splide:not( .is-overflow ) .splide__list { justify-content: center; }
```

**Buttons outside the root:** set `arrows: false`, then `onClick={ () => ref.current?.go( '<' ) }`. Disable them from `onMoved` state (`references/arrows.md`).

**Driving the carousel from state, reading the index, remounting with `key`:** `references/reactivity.md`.

**Loop carousel with changing slides:** give it `key={ items.map( item => item.id ).join() }`. Otherwise, once the list changes, every later re-render calls `refresh()` and restarts autoplay.

**Mobile-only carousel:** `mediaQuery: 'min', breakpoints: { 1024: { destroy: true } }`. Event props stop firing after the first destroy and remount, so use an extension for listeners.

**Images in Next.js:** `next/image` with `fill` in a `position: relative` box, with `heightRatio` or `fixedHeight` on the carousel (`references/nextjs.md`).

## Custom extension

An extension is a function `( splide, Components, options ) => ( { setup?, mount?, destroy? } )`, passed through the `extensions` prop. Define it at module level, since the prop is read once at mount:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

const SlideNumber: ComponentConstructor = ( splide, Components ) => {
  const { track } = Components.Elements;
  let elm: HTMLDivElement | undefined;

  function mount() {
    elm = document.createElement( 'div' );
    track.parentElement!.insertBefore( elm, track.nextSibling );
    update();
    splide.on( 'move.slideNumber', update );
  }

  function update() {
    elm!.textContent = `${ splide.index + 1 }/${ splide.length }`;
  }

  // Required in React: StrictMode mounts, destroys and mounts again in development.
  function destroy() {
    elm?.remove();
    elm = undefined;
  }

  return { mount, destroy };
};

const extensions = { SlideNumber };

export default function Example() {
  return (
    <Splide extensions={ extensions } options={ { label: 'Slide number example' } }>
      <SplideSlide>01</SplideSlide>
      <SplideSlide>02</SplideSlide>
      <SplideSlide>03</SplideSlide>
    </Splide>
  );
}
```

- Lifecycle: `setup()` for every extension runs before any component mounts, then `mount()` (after the built-in components), then `destroy( completely )`. Register mount-time events in `setup()`.
- To get extension state into React, call a callback held in a ref, or emit an event and listen to it. Examples, the helper APIs (`EventInterface`, `RequestInterval`, `Throttle`) and replacing a core component: `references/custom-extensions.md`.
- `ComponentConstructor` needs `@splidejs/splide` as a direct dependency under pnpm; without it, use `NonNullable<SplideProps[ 'extensions' ]>[ string ]`.
- Custom transition: return `start( index, done )` and `cancel()`, type the function's return as `TransitionComponent`, and pass it with the `transition` prop. Always call `done()`, including for a zero distance. The prop also logs a development warning; `references/custom-transitions.md` has a `CleanSplide` subclass that keeps both props off the DOM.

## Official extensions

Named exports only (a default import passes `tsc` but the Vite build fails). Pass them with the `extensions` prop under their standard keys. None has been released since 2022, and all work with React Splide's bundled core.

```tsx
import { AutoScroll } from '@splidejs/splide-extension-auto-scroll';

<Splide
  extensions={ { AutoScroll } }
  options={ { label: 'Partner logos', type: 'loop', drag: 'free', focus: 'center', perPage: 4, autoScroll: { speed: 1 } } }
>...</Splide>
```

| Extension | Package @ latest | Export | Options | React notes | Reference |
|---|---|---|---|---|---|
| Auto Scroll | `@splidejs/splide-extension-auto-scroll@0.5.3` | `AutoScroll` | `autoScroll: { speed, autoStart, rewind, pauseOnHover, pauseOnFocus, useToggleButton }` | `play()` throws when the scroller isn't mounted (fade, `autoScroll: false`) | `references/extensions/auto-scroll.md` |
| Intersection | `@splidejs/splide-extension-intersection@0.2.0` | `Intersection` | `intersection: { inView, outView, root, rootMargin, threshold, once }` | Never pass `root` as an element in the options prop (stack overflow) | `references/extensions/intersection.md` |
| Video | `@splidejs/splide-extension-video@0.8.0` + `dist/css/splide-extension-video.min.css` | `Video` | `video: { autoplay, mute, loop, volume, hideControls, playerOptions }`; `<SplideSlide data-splide-youtube="...">` | `destroy()` throws when an HTML video never played, which unmounts the app: use the `SafeVideo` wrapper | `references/extensions/video.md` |
| Grid | `@splidejs/splide-extension-grid@0.4.1` | `Grid` | `grid: { rows, cols, dimensions, gap: { row, col } }` | Moves the slide elements: remount with a key when the slides change | `references/extensions/grid.md` |
| URL Hash | `@splidejs/splide-extension-url-hash@0.3.0` | `URLHash` | `<SplideSlide data-splide-hash="name">` | Types resolve to `any` on case-sensitive file systems | `references/extensions/url-hash.md` |

Extension events (`video:*`, `intersection*`) have no props: register them with `splide.on()` in `onMounted`. Packages, CSS, types and a combined example: `references/extensions/packages.md`.

## Next.js

- Render `<Splide>` from a `'use client'` component. In a Server Component the build fails with "Class extends value undefined is not a constructor or null".
- `next/dynamic` with `ssr: false` isn't needed (server rendering works with no hydration warnings) and Next.js 16 rejects it in Server Components.
- The core CSS keeps every carousel at `visibility: hidden` until it mounts, so server HTML is invisible until hydration. A small CSS block shows the first page without layout shift (tested with `type: 'slide'`).
- The App Router runs StrictMode in development: use `options.label` instead of `aria-label`.
- Details and code: `references/nextjs.md`.

## Accessibility checklist

Full ARIA map and what React changes: `references/accessibility.md`.

- Name every carousel with `options={ { label } }` or `options={ { labelledby } }`. A `tag="section"` root needs one. Use `options={ { role: 'group' } }` for decorative carousels.
- Autoplay: render the `splide__toggle` play/pause button (needs `hasTrack={ false }`) for WCAG 2.2.2, and keep `perPage` below the slide count, or autoplay never starts and the toggle still says "Pause".
- Keep arrows, pagination or drag available; reduced motion pauses autoplay.
- Visible focus styles: the themes include them; with core CSS, use `:focus-visible` with `.splide.is-focus-in .splide__arrow:focus` as a fallback.
- Elements with `tabindex="0"`, `video[controls]` and `summary` inside slides aren't managed: extend `focusableNodes`, or toggle them in `onVisible`/`onHidden` plus an initial pass in `onReady`.
- Translate the `i18n` strings on non-English sites, and switch language with `key={ locale }` (arrow labels don't update live).
- With `isNavigation`, keep `pagination: false`, and restore the thumbnail ARIA after `refresh()` (see [Recipes](#recipes)).
- Keep `className` stable, and check the accessibility tree in development too, since that's where prop-based names disappear.

## Pitfalls

- TypeScript can't find the types without `"resolvePackageJsonExports": false`; a `paths` mapping breaks Next.js at runtime.
- Event props are captured at mount, run before Splide's own handlers, and disappear after a breakpoint `destroy` remount (see [Event props](#event-props)).
- There's no `onOverflow` prop, and the lazy-load prop is `onLazyLoadLoaded`.
- `aria-label`, `aria-labelledby`, `role`, `tabIndex` and `style` props on `<Splide>` vanish in development under StrictMode.
- Changing `className` on `<Splide>` hides the carousel; on `<SplideSlide>` it drops `is-active`.
- With the default `hasTrack`, every child lands inside the `<ul>`.
- Options updates merge: removed keys persist, explicit `undefined` replaces defaults, updates override active breakpoints, and changes inside `breakpoints` are ignored. `type`, `autoplay`, `start`, `lazyLoad`, `isNavigation` and the other Remount options need a `key` remount.
- Loop carousels refresh on every re-render once their slide list has changed: key them by the slide ids, and never `setState` in `onRefresh`.
- Changing what's inside a slide doesn't refresh the layout; call `refresh()` in an effect. `refresh()` also restores each `<li>`'s original `style`, and Splide overwrites inline slide `width`/`height`.
- Don't use `splide.add()`/`remove()`, don't change slides during a move, and key Grid carousels by their slides.
- A DOM element or `ref.current` inside `options` crashes with "Maximum call stack size exceeded".
- `Splide.defaults` set on `@splidejs/splide` has no effect, and runtime imports from it add a second core.
- React `onClick` handlers inside loop clones never run (clones are `cloneNode` copies). Use `<Splide onClick={ ( splide, Slide ) => ... }>` with `Slide.slideIndex`.
- `splide.off( event )` without a namespace removes the matching event prop.
- Calling `go()` while a `destroy` breakpoint has the carousel destroyed leaves it stuck; check `splide.state.is( 7 )` first.
- Video's `destroy()` crash, and Auto Scroll's `play()` throw: see [Official extensions](#official-extensions).
- Core pitfalls that apply unchanged (sync order, `ttb` height, `rewind` in loop, autoplay toggle labels, `noDrag` matching): `references/getting-started.md` and the topic files.

## Reference index

Read the file that matches the task. Each one is complete and tested, with the code in the file being the code that ran.

| File | Read when |
|---|---|
| `references/getting-started.md` | Install, package contents, CSS, first carousel, several carousels, shared defaults, `data-splide`, mount errors |
| `references/typescript.md` | Types don't resolve, `skipLibCheck`, typing refs, handlers, extensions |
| `references/nextjs.md` | App Router or Pages Router, server rendering, first paint, `next/image`, development labels |
| `references/reactivity.md` | How prop, children and state changes reach the carousel; stale handlers; `key` remounts; controlled index; StrictMode |
| `references/themes.md` | Choosing or overriding a theme, CSS Modules, core CSS, the `classes` option |
| `references/structure.md` | Rendered DOM, `hasTrack`, where JSX elements may go, data attributes, generated ARIA |
| `references/options.md` | Any option: type, default, live-update behaviour, readonly options, precedence |
| `references/arrows.md` | Custom, relocated or outside arrows |
| `references/pagination.md` | Custom, numbered or outside pagination, page count |
| `references/autoplay.md` | Autoplay, toggle, progress bar, mirroring autoplay in React state, in-view autoplay |
| `references/auto-width.md` | Variable-width or variable-height slides, size changes after mount |
| `references/overflow.md` | Too few slides: hiding controls, reading overflow without `onOverflow` |
| `references/accessibility.md` | ARIA, keyboard, focus, labels, WCAG, what React changes |
| `references/i18n.md` | Translating labels, i18n libraries, RTL, language switching |
| `references/apis.md` | Component ref API and the full core instance API, `go()` grammar, states |
| `references/events.md` | Every event, the 28 props, where to register a handler, timing, stale handlers |
| `references/custom-extensions.md` | Writing an extension in TypeScript, bridging its state to React, bundle cost |
| `references/custom-transitions.md` | Writing a transition, the `transition` prop, `CleanSplide` |
| `references/version-notes.md` | Older React Splide versions (0.5/0.6), migrating, known issues in 0.7.12 |
| `references/components/*.md` | `splide.Components.*` (Arrows, Autoplay, Controller, Direction, Drag, Elements, Layout, Move, Pagination, Scroll, Slide, Slides) from React |
| `references/components/other-components.md` | Clones, Cover, Keyboard, LazyLoad, Live, Media, Sync, Wheel |
| `references/tutorials/*.md` | Complete components: image carousel, thumbnail carousel, gallery, carousel progress |
| `references/extensions/*.md` | Auto Scroll, Intersection, Video, Grid, URL Hash; `packages.md` lists the npm packages |
