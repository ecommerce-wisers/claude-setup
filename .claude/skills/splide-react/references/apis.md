# APIs

The `<Splide>` component's ref API (`splide`, `splideRef`, `go()`, `sync()`) and the full core instance API you reach through it, with typing, timing and the React-specific rules.

Source: https://splidejs.com/guides/apis/ and https://splidejs.com/integration/react-splide/

React Splide 0.7.12 bundles Splide core 4.1.3, which behaves the same as 4.1.4. The component API was checked against `src/js/components/Splide/Splide.tsx`; the core API against `core/Splide/Splide.ts` and the 4.1.4 `dist/types/index.d.ts`. Facts marked **(source)** are not on the docs pages. **(tested)** means the example ran in headless Chrome with React 19.3, as a production build and in development under StrictMode.

## Contents

- [Setup](#setup)
- [Getting the instance](#getting-the-instance)
- [Typing](#typing)
- [Component API](#component-api)
- [Creating and mounting](#creating-and-mounting)
- [Class (static) properties](#class-static-properties)
  - [`Splide.defaults`](#splidedefaults)
  - [`Splide.STATES`](#splidestates)
- [Instance properties](#instance-properties)
- [Instance methods](#instance-methods)
  - [`mount()`](#mount)
  - [`sync()`](#sync)
  - [`go()`](#go)
  - [`on()`, `off()`, `emit()`](#on-off-emit)
  - [`add()` and `remove()`](#add-and-remove)
  - [`is()`](#is)
  - [`refresh()`](#refresh)
  - [`destroy()`](#destroy)
- [State machine](#state-machine)
- [When the instance exists](#when-the-instance-exists)
- [Components reference](#components-reference)
- [Docs vs source](#docs-vs-source)

## Setup

```bash
npm install @splidejs/react-splide
```

```tsx
import { useRef } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options, SplideEventHandlers, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The package's types need one tsconfig line under `moduleResolution: "bundler"` (the Vite and Next.js default). See `references/typescript.md`.

Don't import anything at runtime from `@splidejs/splide`. React Splide inlines its own copy of core, so a runtime import adds a second copy to the bundle, and that copy's statics (`Splide.defaults`) don't reach `<Splide>`.

## Getting the instance

There are two ways to reach the core instance:

| Where | How |
|---|---|
| Anywhere after mount | `ref.current?.splide`, with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>` |
| Inside an event prop | The first argument: `onMoved={ ( splide, newIndex ) => ... }` |
| Inside an extension | The first argument of the extension function: `splide => ( { mount() { ... } } )` |

`ref.current` is the React component (a class instance). Its `splide` property is the core instance, the same object a vanilla `new Splide()` returns.

**(tested)** Controls driven through the ref: "Next page" went from slide 1 to 3, "Page 3" to 5, "Previous page" back to 3, "One slide forward" to 4, "First slide" to 1:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label     : 'Gallery',
  perPage   : 2,
  pagination: false,
};

const slides = [ 1, 2, 3, 4, 5, 6 ];

export default function Controls() {
  const ref = useRef<Splide>( null ); // the React component; .splide is the core instance
  const [ index, setIndex ] = useState( 0 );

  const go = ( control: number | string ) => ref.current?.go( control );

  return (
    <>
      <Splide ref={ ref } options={ options } onMoved={ ( splide, newIndex ) => setIndex( newIndex ) }>
        { slides.map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <p id="status">Slide { index + 1 } of { slides.length }</p>
      <button type="button" id="prev-page" onClick={ () => go( '<' ) }>Previous page</button>
      <button type="button" id="next-page" onClick={ () => go( '>' ) }>Next page</button>
      <button type="button" id="page-3" onClick={ () => go( '>2' ) }>Page 3</button>
      <button type="button" id="plus-1" onClick={ () => go( '+1' ) }>One slide forward</button>
      <button type="button" id="first" onClick={ () => go( 0 ) }>First slide</button>
    </>
  );
}
```

The inline `onMoved` is safe here because it only calls a state setter. A handler that reads props or state goes stale (see `references/events.md`).

## Typing

```tsx
const ref = useRef<Splide>( null ); // Splide is the component class, so it works as the ref type
```

React Splide exports the component classes and these types: `SplideProps`, `SplideEventHandlers`, `SplideEventHandlerMap`, `SplideHTMLAttributes` and `Options`. Core types can be derived from them, so you don't need `@splidejs/splide` as a dependency:

```tsx
import type { Options, SplideEventHandlers, SplideProps } from '@splidejs/react-splide';
import type { Splide } from '@splidejs/react-splide';

type SplideInstance  = NonNullable<Splide['splide']>;                            // core Splide
type Extension       = NonNullable<SplideProps['extensions']>[ string ];        // ComponentConstructor
type SlideComponent  = Parameters<SplideEventHandlers['onActive']>[ 1 ];
type PaginationData  = Parameters<SplideEventHandlers['onPaginationMounted']>[ 1 ];
type PaginationItem  = Parameters<SplideEventHandlers['onPaginationMounted']>[ 2 ];
```

- `import type` from `@splidejs/splide` also works for core types (`Components`, `EventMap`, `ResponsiveOptions`), but under pnpm's strict layout only after adding core to your own `package.json`. Details in `references/typescript.md`.
- `splide.Components.MyExtension` is typed `BaseComponent | undefined` (the `Components` type has an index signature). Cast it to your extension's return type.
- The `Autoplay` component's type declares `pause()` with no argument, though the code takes `pause( stop = true )`.

## Component API

| Member | Type | Description |
|---|---|---|
| `splide` | `Splide \| undefined` (core) | The core instance. `undefined` before `componentDidMount` and after unmount |
| `splideRef` | `RefObject<HTMLDivElement>` | The root element (typed as `div` whatever the `tag`) |
| `go( control )` | `( control: number \| string ) => void` | Calls `this.splide?.go( control )`. Same grammar as [`go()`](#go). Returns nothing; a no-op when there is no instance |
| `sync( splide )` | `( splide: Splide ) => void` | Calls `this.splide?.sync( splide )`. Takes a **core** instance (`otherRef.current.splide`), not a component |

**(source)** Unlike the core methods, the component's `go()` and `sync()` return `undefined`, so they can't be chained. Call the core instance for chaining: `splide.off( 'moved.app' ).on( 'moved.app', onMoved )`.

**(tested)** Inside `useLayoutEffect` and `useEffect` of the component that renders `<Splide>`, the instance exists and is `IDLE`, and `splideRef.current` is the root:

```tsx
import { useEffect, useLayoutEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// The numbers behind Splide.STATES (fixed in v4). React Splide does not re-export the core class.
const STATES = { CREATED: 1, MOUNTED: 2, IDLE: 3, MOVING: 4, SCROLLING: 5, DRAGGING: 6, DESTROYED: 7 } as const;

export default function InstanceInfo() {
  const ref = useRef<Splide>( null );
  const [ info, setInfo ] = useState( '' );
  const [ layoutInfo, setLayoutInfo ] = useState( '' );

  // Runs after <Splide>'s componentDidMount, so the instance exists and is IDLE.
  useLayoutEffect( () => {
    setLayoutInfo( ref.current?.splide ? 'instance in layout effect' : 'no instance in layout effect' );
  }, [] );

  useEffect( () => {
    const splide = ref.current?.splide;
    const root = ref.current?.splideRef.current; // the root element

    if ( splide && root ) {
      setInfo( [
        `root ${ root.id }`,
        `length ${ splide.length }`,
        `index ${ splide.index }`,
        `idle ${ splide.state.is( STATES.IDLE ) }`,
        `loop ${ splide.is( 'loop' ) }`,
        `perPage ${ splide.options.perPage }`,
      ].join( ', ' ) );
    }
  }, [] );

  return (
    <>
      <Splide ref={ ref } id="info-carousel" options={ { label: 'Info', type: 'loop', perPage: 2 } }>
        <SplideSlide>1</SplideSlide>
        <SplideSlide>2</SplideSlide>
        <SplideSlide>3</SplideSlide>
      </Splide>
      <p id="info">{ info }</p>
      <p id="layout-info">{ layoutInfo }</p>
    </>
  );
}
```

Output (both modes): `root info-carousel, length 3, index 0, idle true, loop true, perPage 2`.

## Creating and mounting

You never call `new Splide()` or `mount()`. React Splide does, in `componentDidMount`:

1. `new Splide( rootElement, options )`, with the `options` prop.
2. It registers the event props with `splide.on()`.
3. `splide.mount( extensions, transition )`, with the `extensions` and `transition` props.

So the constructor rules apply to the `options` prop (source):

- Options are deep-merged in this order, with later sources winning:
  1. `{ label: root aria-label, labelledby: root aria-labelledby }`, from the props React rendered
  2. `DEFAULTS`
  3. `Splide.defaults` of React Splide's bundled copy (not reachable, see below)
  4. the `options` prop
  5. the JSON in the root's `data-splide` attribute. **(tested)** `<Splide options={ { perPage: 1 } } data-splide='{"perPage":2}'>` showed two slides per page
- Invalid `data-splide` JSON throws `Error: [splide] Invalid JSON` during `componentDidMount`, which unmounts the tree unless an error boundary catches it.
- Nothing is rendered by core until mount. The server-rendered and first client HTML is the bare markup.
- Later changes to `options` go through the `options` setter (see [Instance properties](#instance-properties)). `extensions`, `transition`, `tag`, `hasTrack` and the event props are read once (see `references/reactivity.md`).

## Class (static) properties

### `Splide.defaults`

`defaults: Options`. Changes the default options for all Splide instances, in vanilla.

**(tested)** In React it doesn't work: setting `Splide.defaults` on `import { Splide } from '@splidejs/splide'` changes a different copy of core, and `<Splide>` ignores it. The `Splide` that React Splide exports is the component class, which has no `defaults`. Share an options object instead:

```tsx
// carousel-defaults.ts
import type { Options } from '@splidejs/react-splide';

export const carouselDefaults: Options = {
  gap       : '1rem',
  pagination: false,
  i18n      : { prev: 'Previous item', next: 'Next item' },
};

// usage
<Splide options={ { ...carouselDefaults, label: 'Products', perPage: 3 } }>
```

A spread is shallow: an `i18n` in the second object replaces the whole `i18n` from the first. Spread nested objects too (`i18n: { ...carouselDefaults.i18n, ...mine }`).

### `Splide.STATES`

`STATES: Record<string, number>`. The collection of carousel states. React Splide doesn't re-export it, and the core class isn't reachable without a runtime import from `@splidejs/splide`. The values are fixed in v4, so copy them (as in the example above):

| Key | Value **(source)** | Meaning (docs) |
|---|---|---|
| `CREATED` | `1` | Splide has just been created |
| `MOUNTED` | `2` | Splide has mounted its components. **(source)** Never set by any code in v4 |
| `IDLE` | `3` | Splide is ready |
| `MOVING` | `4` | Splide is moving |
| `SCROLLING` | `5` | **(source, not in docs)** The Scroll component is scrolling: free drag, `useScroll`, `Controller.scroll()` |
| `DRAGGING` | `6` | **(source, not in docs)** The user is dragging |
| `DESTROYED` | `7` | Splide has been destroyed |

```tsx
const STATES = { CREATED: 1, MOUNTED: 2, IDLE: 3, MOVING: 4, SCROLLING: 5, DRAGGING: 6, DESTROYED: 7 } as const;

const isBusy = ( splide: SplideInstance ) => splide.state.is( [ STATES.MOVING, STATES.SCROLLING ] );
```

You will not need `MOUNTED`, because the state changes straight to `IDLE`.

## Instance properties

On `ref.current?.splide` (or the `splide` argument of an event prop):

| Property | Type | Access | Description |
|---|---|---|---|
| `root` | `HTMLElement` | readonly | The root element (the same element as `splideRef.current`) |
| `event` | `EventInterfaceObject` | readonly | The instance's event interface. You don't need it: use `on()`, `off()` and `emit()` |
| `Components` | `Components` | readonly | All component objects, keyed by name. Empty until mount |
| `state` | `StateObject` | readonly | `state.is( state: number \| number[] ): boolean` and `state.set( state: number ): void`. See [`STATES`](#splidestates) |
| `splides` | `SyncTarget[]` | readonly | Data for the instances this one is synced with (^3.3.1). Each item: `splide: Splide`, and `isParent: boolean \| undefined`, which is `true` if that instance is the parent that called `sync()` |
| `options` | `Options` | get / set | Get: an object with the latest options. Set: merges the given options into the current ones and emits `updated` |
| `length` | `number` | get | The number of slides. **(source)** Clones are excluded (`Slides.getLength( true )`) |
| `index` | `number` | get | The active slide index. **(source)** Always a real index from `0` to `length - 1`, even in loop mode |

`Components` example:

```tsx
<button type="button" onClick={ () => ref.current?.splide?.Components.Autoplay.pause() }>Pause</button>
```

### `options`

In React, change options through the `options` prop. React Splide compares the new prop with the previous one (deep equality) and, when they differ, assigns it: `splide.options = options`. The rules for that setter:

Only responsive options can be changed by assigning:

```tsx
const [ perPage, setPerPage ] = useState( 3 );

<Splide options={ { label: 'Products', perPage } }>   // changing perPage updates the carousel
```

Do not change other (readonly) options, and do not set a property directly. Either can break the carousel:

```tsx
// Don't change the type: the option is not responsive
<Splide options={ { type: isMobile ? 'fade' : 'slide' } }>

// Don't directly set a value
ref.current!.splide!.options.perPage = 4;
```

To change a readonly option, remount with a `key` (see `references/reactivity.md`).

**(source)** Responsive options (`ResponsiveOptions` in the types): `label`, `labelledby`, `speed`, `rewind`, `rewindSpeed`, `rewindByDrag`, `width`, `height`, `fixedWidth`, `fixedHeight`, `heightRatio`, `perPage`, `perMove`, `clones`, `cloneStatus`, `focus`, `gap`, `padding`, `arrows`, `pagination`, `paginationKeyboard`, `paginationDirection`, `easing`, `easingFunc`, `drag`, `snap`, `dragMinThreshold`, `flickPower`, `flickMaxPages`, `destroy`.

From source:

- The setter calls `Components.Media.set( options, true, true )`. It merges into the current options and into the base options, so the change survives later breakpoint switches, and it always emits `updated`.
- The setter throws a `TypeError` before mount, because `Components.Media` does not exist yet. In React the instance only exists after mount, so this can't happen through the prop.
- `splide.options` is an object whose prototype holds the base options and whose own properties hold the active breakpoint overrides. Reading a property (`splide.options.perPage`) works, but `Object.keys()` or `JSON.stringify()` shows only the overrides.
- Setting `ref.current.splide.options = {...}` yourself works the same way, but React Splide doesn't know about it. The next time the `options` prop changes, the whole prop object is assigned again and wins for every key it contains.
- Removing a key from the prop doesn't reset it, because the setter merges. Set the value you want explicitly.

## Instance methods

All core methods except `is()` return `this`, so calls can be chained: `splide.off( 'moved.app' ).on( 'moved.app', onMoved )` (tested).

### `mount()`

`mount( Extensions?: Record<string, ComponentConstructor>, Transition?: ComponentConstructor ): this`

Initializes the instance. React Splide calls it for you with the `extensions` and `transition` props:

| Param | React prop | Description |
|---|---|---|
| `Extensions` | `extensions` | Optional. An object with extensions, `{ Name: ComponentConstructor }` |
| `Transition` | `transition` | Optional. A Transition component |

From source:

- Throws `Error: [splide] Already mounted!` unless the state is `CREATED` or `DESTROYED`. A destroyed instance can be mounted again.
- `Extensions` and `Transition` are remembered. A later `mount()` without arguments, including the automatic remount after a breakpoint `destroy: true`, reuses them.
- A `ComponentConstructor` is a function `( Splide, Components, options ) => { setup?(), mount?(), destroy?( completely ) }`. Each one is available as `splide.Components[ Name ]`.
- Order: every component is constructed and its `setup()` called (built-ins first, then extensions, then `Transition`). Then every `mount()` runs. Then `mounted` is emitted, the root gets `is-initialized`, the state becomes `IDLE`, and `ready` is emitted.
- The default transition is the Fade component when `type` is `'fade'`, otherwise the Slide component.
- React: the props are read once. Define extensions outside the component (or keep them in `useState`) so they don't change identity, and use refs inside them for values that change. React Splide also writes the `extensions` and `transition` props to the root as attributes (`extensions="[object Object]"`), which is harmless.

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';

type Extension = NonNullable<SplideProps['extensions']>[ string ];

// A minimal extension
const Logger: Extension = splide => ( {
  mount() {
    splide.on( 'moved', index => {
      console.log( 'Now at', index, 'of', splide.length );
    } );
  },
} );

const extensions = { Logger };

<Splide options={ { label: 'Logged' } } extensions={ extensions }>{ slides }</Splide>
```

Extensions in depth: `references/custom-extensions.md`. Registration timing (`setup()` vs `mount()`): `references/events.md`.

### `sync()`

Core: `sync( splide: Splide ): this`. Component: `sync( splide: Splide ): void`.

Syncs the carousel with the given one. The docs say it must be called before `mount()`, which React makes impossible: the instances only exist after `componentDidMount`. Call it after mount instead, which the core supports:

**(tested)** Main and thumbnails in sync, in production and under StrictMode: thumbnail slides got `role="button"`, `aria-current` and a single `aria-controls` ID; clicking a thumbnail or pressing Enter on it moved the main carousel; the main carousel's arrow moved the thumbnails:

```tsx
import { useEffect, useRef } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const images = [ 'a', 'b', 'c', 'd', 'e' ].map( id => ( { id, src: `/images/${ id }.jpg`, alt: `Photo ${ id }` } ) );

const mainOptions: Options = {
  label     : 'Product photos',
  type      : 'fade',
  rewind    : true,
  pagination: false,
};

const thumbsOptions: Options = {
  label       : 'Product photo thumbnails',
  fixedWidth  : 100,
  fixedHeight : 60,
  gap         : 10,
  rewind      : true,
  pagination  : false,
  isNavigation: true,
};

export default function ProductGallery() {
  const mainRef = useRef<Splide>( null );
  const thumbsRef = useRef<Splide>( null );

  // Both instances exist once effects run. Calling sync() after mount remounts both Sync components.
  useEffect( () => {
    const thumbs = thumbsRef.current?.splide;

    if ( mainRef.current && thumbs ) {
      mainRef.current.sync( thumbs );
    }
  }, [] );

  return (
    <>
      <Splide id="main" ref={ mainRef } options={ mainOptions }>
        { images.map( image => (
          <SplideSlide key={ image.id }><img src={ image.src } alt={ image.alt } /></SplideSlide>
        ) ) }
      </Splide>

      <Splide id="thumbs" ref={ thumbsRef } options={ thumbsOptions }>
        { images.map( image => (
          <SplideSlide key={ image.id }><img src={ image.src } alt="" /></SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

From source:

- It pushes `{ splide }` to `this.splides` and `{ splide: this, isParent: true }` to `splide.splides`.
- If it is called while this instance is `IDLE` (mounted), it calls `Components.Sync.remount()` on both instances. The other instance must also be mounted, or this throws. In an effect, both are.
- Syncing works both ways. On `move`, each instance calls `target.go( target.is( 'loop' ) ? destIndex : newIndex )` on the other.
- Order matters only before mount (vanilla): a navigation carousel reads the other instance's `Components.Slides` while it mounts, to build `aria-controls`, and throws a `TypeError` if that instance is not mounted yet. With the React pattern both are mounted when `sync()` runs, so the order of the two `<Splide>` elements doesn't matter.
- A carousel can sync with more than one other carousel by calling `sync()` once per target.
- **(tested)** `navigation:mounted` fires twice on the thumbnails: at mount with an empty `splides` array, then during `sync()` with the main carousel as the parent target. In development each StrictMode instance gets one sync (no duplicated `aria-controls`).
- There is no `unsync()`. Unmounting either carousel destroys it; if only one is remounted (for example with a `key`), run the effect again by putting the same key in its dependencies.

Full thumbnail tutorial: `references/tutorials/thumbnail-carousel.md`.

### `go()`

Core: `go( control: number | string ): this`. Component: `go( control: number | string ): void`.

Moves the carousel using one of these control patterns:

| Pattern | Docs description | Source behavior (`Controller.parse`) |
|---|---|---|
| `i` (number) | Goes to slide `i` | Non-loop: clamped to `0` through the end index. The end index is `length - perPage`, or `length - 1` with `focus`/`isNavigation`. Loop: used as is, so out-of-range values wrap through the clones (`go( -1 )` goes to the last slide) |
| `'+${i}'` | Increments the slide index by `i` | `'+'` alone means `+1`. `'+0'` also means `+1` (`+number \|\| 1`) |
| `'-${i}'` | Decrements the slide index by `i` | `'-'` alone means `-1`. `'-0'` also means `-1` |
| `'>'` | Goes to the next page | Moves by `perMove`, or by 1 with `focus`/`isNavigation`, otherwise by `perPage` (snapped to page boundaries) |
| `'<'` | Goes to the previous page | Same rules, backwards |
| `'>${i}'` | Goes to page `i` | `i` is **zero-based**: `'>0'` is the first page. The slide index is `perPage * i`, or `i` with `focus`/`isNavigation`, clamped to the end index |
| `'<${i}'` | Not documented | **No page form.** The digits are ignored, so `'<2'` behaves like `'<'` |
| Other strings, e.g. `'3'` | Not documented | No match, nothing happens. Convert numeric strings with `Number()` first |

In most cases `'>'` and `'<'` are all you need, because they respect `perPage` and `perMove`.

```tsx
const splide = ref.current?.splide;

splide?.go( 1 );     // Goes to the slide 1
splide?.go( '+2' );  // Increments the index
splide?.go( '>' );   // Goes to the next page
splide?.go( '>2' );  // Goes to the page 2 (the third page)

ref.current?.go( '>' ); // the component's wrapper, same grammar
```

More from source:

- The pattern regex is `/([+\-<>])(\d+)?/` and is not anchored: the first `+`, `-`, `<` or `>` anywhere in the string is used.
- Out-of-range targets for `'+N'`, `'-N'`, `'>'` and `'<'`: if the current index is not yet at the first or last reachable index, and `perMove` is not set, Splide stops at that edge first. From the edge, `type: 'loop'` wraps, `rewind: true` jumps to the other end, and a plain `slide` carousel does nothing.
- `go()` does nothing when:
  - `waitForTransition: true` and the carousel is `MOVING` or `SCROLLING`
  - the target resolves to `-1` (no slide to go to)
  - the target equals the current index (the public `go()` passes `allowSameIndex = false`)
  - for `'+'`, `'-'`, `'>'` and `'<'`: there are not enough slides (Slide components, clones included, `<= perPage`) and no `focus`/`isNavigation`. A number target is clamped to the end index, which is then `0`, so it does nothing either
- Core `go()` before mount throws a `TypeError`. In React the instance doesn't exist before mount, so `ref.current?.splide?.go()` and `ref.current?.go()` simply do nothing.
- `go()` takes no callback. For one that runs when the transition ends, call the component directly: `splide.Components.Controller.go( control, allowSameIndex?, callback? )`. Or listen for `moved`.
- To drive the carousel from React state (an index in state, `go()` in an effect, `onMoved` back into state), see `references/reactivity.md`.

### `on()`, `off()`, `emit()`

- `on( events: string | string[], callback: AnyFunction ): this` registers a handler. Names are separated by spaces, an array works too, and `name.namespace` adds a namespace. The callback receives the event's arguments spread out, without the instance, and its `this` is the callback itself.
- `off( events: string | string[] ): this` removes handlers. **(source)** `off( 'move' )` removes only the `move` handlers registered **without** a namespace; `off( 'move.ns' )` removes only that namespace's; `'.ns'` alone matches nothing; there is no way to remove one function.
- `emit( event: string, ...args: any[] ): this` triggers every handler for that name, synchronously, in registration order. Emitting a built-in name, such as `refresh`, triggers the internal components that listen to it too.

```tsx
useEffect( () => {
  const splide = ref.current?.splide;
  if ( ! splide ) return;

  splide.on( 'gallery:open.app', ( slideIndex: number, source: string ) => console.log( slideIndex, source ) );
  splide.emit( 'gallery:open', 2, 'button' );

  return () => {
    splide.off( 'gallery:open.app' );
  };
}, [] );
```

Event props, `on()` in extensions vs after mount, timing and stale handlers: `references/events.md`.

### `add()` and `remove()`

`add( slides: string | HTMLElement | Array<string | HTMLElement>, index?: number ): this`
`remove( matcher: SlideMatcher ): this`

Core methods that insert or remove slide elements in the list, then emit `refresh`. `SlideMatcher = number | number[] | string | SlidesPredicate`: an index, an array of indices, a selector, or a predicate `( Slide, index, Slides ) => boolean`.

**In React, don't use them on a list React renders.** Render the slides from state; React Splide notices the changed list and calls `refresh()` itself. **(tested)** What happens otherwise:

- `splide.add( '<li>...</li>' )` inserts an element React doesn't know about. It stays in the list through later renders, and React's own new slides are appended after it.
- `splide.remove( 0 )` on a slide React rendered, followed by a state change that removes that slide, made React throw `NotFoundError: Failed to execute 'removeChild' on 'Node'` and unmount the whole tree.

**(tested)** The state-driven way. Slide labels and pagination updated on every change ("1 of 5" after two adds, "1 of 4" after a removal):

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Render slides from state instead of calling splide.add()/remove().
// <Splide> calls splide.refresh() in componentDidUpdate when the rendered slide elements change.
export default function DynamicSlides() {
  const ref = useRef<Splide>( null );
  const [ items, setItems ] = useState( [ 1, 2, 3 ] );
  const [ length, setLength ] = useState( 0 );

  // Effects run after <Splide>'s componentDidUpdate, so the refresh has already happened.
  useEffect( () => {
    setLength( ref.current?.splide?.length ?? 0 );
  }, [ items ] );

  return (
    <>
      <Splide ref={ ref } options={ { label: 'Dynamic', perPage: 2 } }>
        { items.map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <p id="length">{ length } slides</p>
      <button type="button" id="add" onClick={ () => setItems( list => [ ...list, list.length + 1 ] ) }>Add</button>
      <button type="button" id="remove" onClick={ () => setItems( list => list.slice( 1 ) ) }>Remove first</button>
    </>
  );
}
```

Don't read `splide.length` in `onRefresh` for this: event props run before Splide's components react to `refresh`, so it still has the old count (tested, see `references/events.md`).

Core rules, for when you do call them on slides React doesn't render (source):

- An HTML string is parsed with `DOMParser`, and only its **first** element is used. Pass an array to add several slides.
- The `splide__slide` class (`classes.slide`) is added automatically.
- `add()` emits `refresh`. It also emits `resize` once all `<img>` elements in the added slide have loaded or errored, or immediately if there are none. An `index` omitted or out of range means append.
- `remove()`: the matcher runs over every Slide component, clones included. Numbers match `Slide.index`, and clone indices are negative or `>= length`, so numbers only hit real slides. The predicate's `index` argument is the array position, not `Slide.index`.
- Do not call either while the carousel is moving: they refresh and reposition the carousel.

### `is()`

`is( type: string ): boolean`

Checks the carousel type: `'slide'`, `'loop'` or `'fade'`. Returns `true` if the type matches the current one. **(source)** It compares `options.type === type`.

```tsx
if ( ref.current?.splide?.is( 'loop' ) ) {
  // do something
}
```

### `refresh()`

`refresh(): this`

Refreshes the carousel. Most components reinitialize themselves.

**(source)** It only emits `refresh`. In response, Elements re-collects the slides from the DOM (direct `.splide__slide` children of the list, clones excluded), Slides re-registers them (new IDs, labels and roles), Clones regenerates the clones, Pagination rebuilds, Arrows update, Layout re-measures and Move repositions.

React Splide calls it in `componentDidUpdate` when the list's child elements changed (added, removed or reordered `<SplideSlide>`s). Call it yourself when something changes that React Splide doesn't compare:

- slide content that changes size (with `autoWidth`, `autoHeight`, or wrapping text): `useEffect( () => { ref.current?.splide?.refresh(); }, [ text ] )`
- `pagination` toggled at runtime, so slide roles switch between `tabpanel` and `group` (see `references/accessibility.md`)

**(source)** Don't refresh while the carousel is moving. Elements strips the list's inline `style`, including the `transition`, so `transitionend` never fires: `moved` is not emitted, the state stays `MOVING`, and `aria-hidden` is not updated until the next move ends (tested: `go( '>' )` followed by `refresh()` never emitted `moved`). In React this includes changing the slide list during a transition, for example from an `onMove` handler or when data arrives while autoplay is moving, because the refresh then runs in `componentDidUpdate`. **(tested)** An `onMove` that added a slide meant `onMoved` never fired and no slide had `aria-hidden` afterwards. Change slides from `onMoved`, or while the carousel is idle.

### `destroy()`

`destroy( completely?: boolean ): this`

Destroys the carousel.

| Param | Description |
|---|---|
| `completely` | Optional. If `true`, Splide will not remount the carousel through breakpoints. **(source)** Defaults to **`true`** |

From source:

- `splide.destroy()` is a complete destroy. Pass `false` to keep the breakpoint listeners, so a breakpoint without `destroy` can remount it.
- Breakpoint `destroy: true` calls `destroy( false )`. Breakpoint `destroy: 'completely'` calls `destroy( true )`.
- Called while the state is still `CREATED` (during mount), the destroy is postponed until `ready`.
- Sequence: each component's `destroy( completely )` runs in reverse order (generated arrows, pagination and clones are removed, attributes and classes are cleaned), then `destroy` is emitted, then **all** handlers are removed (including the event props), then `completely` empties `this.splides`, and the state becomes `DESTROYED`.
- A complete destroy also removes `role`, `aria-label` and `aria-labelledby` from the root, even when they came from props (see `references/accessibility.md`).

In React:

- Unmounting `<Splide>` calls `splide.destroy()` and sets `ref.current.splide` to `undefined`. To remove a carousel, stop rendering it.
- In development, StrictMode destroys the first instance right after mounting it and creates a second one. Don't keep the instance in a variable across renders; read `ref.current?.splide` when you need it.
- **(tested)** Calling `ref.current.splide.destroy()` and then `ref.current.splide.mount()` yourself works, but the event props are gone (`destroy()` removed them and React Splide binds them only in `componentDidMount`). Extensions come back, because `mount()` reuses them. Prefer a `key` remount, or a breakpoint with `destroy`.

## State machine

From source (`core/Splide/Splide.ts`, `Move.ts`, `Scroll.ts`, `Drag.ts`):

| Transition | Where |
|---|---|
| (constructor) to `CREATED` (1) | `new Splide()`, in `componentDidMount` |
| `CREATED` / `DESTROYED` to `CREATED` | Start of `mount()` |
| `CREATED` to `IDLE` (3) | End of `mount()`, between `mounted` and `ready` |
| `IDLE` to `MOVING` (4) | `Move.move()`, right before `move` is emitted |
| `MOVING` to `IDLE` | Transition end, right before `moved` |
| `IDLE` to `SCROLLING` (5) | `Scroll.scroll()`, right before `scroll` |
| `SCROLLING` to `IDLE` | Scroll end, right before `scrolled` |
| any to `DRAGGING` (6) | First pointer move after a pointer down on the track, right before `drag` |
| `DRAGGING` to `IDLE` | Pointer up, right before `dragged` (the release move then sets `MOVING` or `SCROLLING`) |
| any to `DESTROYED` (7) | End of `destroy()`, including unmount |

`MOUNTED` (2) is defined but never used. `state.is()` accepts a number or an array: `splide.state.is( [ STATES.MOVING, STATES.SCROLLING ] )`.

## When the instance exists

| Moment | `ref.current?.splide` |
|---|---|
| During render (the component body, JSX) | `undefined` on the first render. Never read it there |
| `useLayoutEffect` / `useEffect` after the first commit | The mounted instance, state `IDLE` (tested) |
| Event props and extensions | Passed in as an argument |
| After an `options` change | The same instance; React Splide updated it in `componentDidUpdate`, before your effects run |
| After unmount | `undefined` |
| Development, StrictMode | A first instance is created and destroyed, then a second one; effects see the second |

Vanilla's "before `mount()`" rules still describe the core, and they explain the timing above:

| Works before `mount()` | Throws (`TypeError`) before `mount()` |
|---|---|
| `on()`, `off()`, `emit()`, `sync()`, `is()`, reading `options`, `root`, `state`, `splides` | `go()`, `add()`, `remove()`, `length`, `index`, setting `options` |
| `destroy()` (postponed until `ready`) | Using `Components.*`, which is an empty object until mounted |

React Splide registers the event props in that window, between `new Splide()` and `mount()`, which is why `onMounted` and the `*:mounted` props fire. Your own code can't use the window, except through an extension's `setup()` (see `references/events.md`).

## Components reference

From `dist/types/index.d.ts` (4.1.4). Reach them as `ref.current?.splide?.Components.<Name>`. Every component may also have `setup()`, `mount()` and `destroy( completely? )`. These are internal APIs; the docs cover only some of them on separate component pages (`references/components/`).

| `splide.Components.` | Public members |
|---|---|
| `Media` | `set( options, base?, notify? )`, `reduce( enable )` (internal) |
| `Direction` | `resolve( prop, axisOnly?, direction? ): string`, `orient( value ): number` |
| `Elements` | `root`, `track`, `list`, `slides: HTMLElement[]`, `arrows?`, `pagination?`, `prev?`, `next?`, `bar?`, `toggle?` |
| `Slides` | `update()`, `register( slide, index, slideIndex )`, `get( excludeClones? ): SlideComponent[]`, `getIn( page )`, `getAt( index )`, `add( slides, index? )`, `remove( matcher )`, `forEach( iteratee, excludeClones? )`, `filter( matcher )`, `style( prop, value, useContainer? )`, `getLength( excludeClones? )`, `isEnough()` |
| `Layout` | `listSize()`, `slideSize( index, withoutGap? )`, `sliderSize( withoutGap? )`, `totalSize( index?, withoutGap? )`, `getPadding( right )`, `isOverflow()`, `resize( force? )` (internal) |
| `Clones` | (none) |
| `Move` | `move( dest, index, prev, callback? )`, `jump( index )`, `translate( position, preventLoop? )`, `shift( position, backwards )`, `cancel()`, `toIndex( position )`, `toPosition( index, trimming? )`, `getPosition()`, `getLimit( max )`, `exceededLimit( max?, position? )`, `reposition()` (internal) |
| `Controller` | `go( control, allowSameIndex?, callback? )`, `scroll( destination, duration?, snap?, callback? )`, `getNext( destination? )`, `getPrev( destination? )`, `getEnd()`, `setIndex( index )`, `getIndex( prev? )`, `toIndex( page )`, `toPage( index )`, `toDest( position )`, `hasFocus()`, `isBusy()`, `getAdjacent( prev, destination? )` (internal) |
| `Arrows` | `arrows: { prev?, next? }`, `update()` (internal) |
| `Autoplay` | `play()`, `pause()`, `isPaused(): boolean`. **(source)** `pause( stop = true )`: with `true`, autoplay stays paused until `play()` |
| `Cover` | (none) |
| `Scroll` | `scroll( position, duration?, snap?, callback? )`, `cancel()` |
| `Drag` | `disable( disabled: boolean )`, `isDragging(): boolean` |
| `Keyboard` | `disable( disabled: boolean )` |
| `LazyLoad` | `check()` |
| `Pagination` | `items: PaginationItem[]`, `getAt( index ): PaginationItem`, `update()` |
| `Sync` | `remount()` (internal) |
| `Wheel` | (none) |
| `Live` | `disable( disabled: boolean )` |
| `Transition` | `start( index, done )`, `cancel()` |
| `<extension name>` | Whatever your extension returns (typed `BaseComponent \| undefined`) |

`SlideComponent` (returned by `Slides.get()`/`getAt()` and passed to many events): `index`, `slideIndex` (the original index for a clone, `-1` otherwise), `slide` (element), `container` (`.splide__slide__container` or `undefined`), `isClone`, `update()`, `style( prop, value, useContainer? )`, `isWithin( from, distance ): boolean`.

`Elements.slides` and `Elements.toggle` are collected when the instance mounts and on `refresh`. Autoplay wires the toggle only at mount, so a toggle rendered later is not picked up.

## Docs vs source

| Topic | Docs say | Source says |
|---|---|---|
| `destroy( completely? )` default | Optional, default not stated (reads as `false`) | `completely = true`: `destroy()` is a complete destroy |
| `STATES` | `CREATED`, `MOUNTED`, `IDLE`, `MOVING`, `DESTROYED` | Also `SCROLLING` (5) and `DRAGGING` (6). Values are 1 to 7. `MOUNTED` is never set |
| `sync()` timing | Must be called before `mount()` | Also accepted after mount when this instance is `IDLE` and the other is mounted (both Sync components remount). React has to use this path |
| `off( 'move' )` | "Removes the registered all handlers" for the event | Removes only un-namespaced handlers for that event |
| `go( '<${i}' )` | Not listed | Not supported: same as `'<'` |
| `'>${i}'` page numbering | "Goes to the page `i`" | Zero-based page index |
| React component `go()` / `sync()` | Not documented beyond "the splide instance is available on `ref.current.splide`" | Wrappers that call the core with `?.` and return nothing |
| `Splide.defaults` in React | Not mentioned | Has no effect on `<Splide>`: React Splide bundles its own core (tested) |
