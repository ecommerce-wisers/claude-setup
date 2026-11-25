# Events

Listening to Splide events in React: the 28 event props, every other core event through `splide.on()` (in an extension or after mount), namespaces and cleanup, timing, and stale handlers.

Source: https://splidejs.com/guides/events/ and https://splidejs.com/integration/react-splide/

React Splide 0.7.12 bundles Splide core 4.1.3, which behaves the same as 4.1.4. The prop list comes from `src/js/constants/events.ts` and `types/events.ts` in React Splide; event names from core's `constants/events.ts`; arguments from the emit calls in `components/*` and `core/Splide/Splide.ts`, cross-checked with core's `EventMap` type. Facts marked **(source)** are not on the docs pages. **(tested)** means the example ran in headless Chrome with React 19.3, as a production build and in development under StrictMode.

## Contents

- [Setup](#setup)
- [Event props](#event-props)
- [Event prop table](#event-prop-table)
- [Events without a prop](#events-without-a-prop)
- [Where to register a handler](#where-to-register-a-handler)
  - [In an extension](#in-an-extension)
  - [After mount, and when that is too late](#after-mount-and-when-that-is-too-late)
- [Multiple events, arrays and namespaces](#multiple-events-arrays-and-namespaces)
- [Removing handlers and cleanup](#removing-handlers-and-cleanup)
- [Emitting custom events](#emitting-custom-events)
- [Timing and order](#timing-and-order)
- [Stale handlers](#stale-handlers)
- [Breakpoint destroy drops the event props](#breakpoint-destroy-drops-the-event-props)
- [Event details](#event-details)
- [Internal events (from source)](#internal-events-from-source)
- [Argument types](#argument-types)
- [Event sequences](#event-sequences)
- [Docs vs source](#docs-vs-source)

## Setup

```bash
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideEventHandlers, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
```

The package's types need one tsconfig line under `moduleResolution: "bundler"`. See `references/typescript.md`.

## Event props

```tsx
<Splide
  options={ { label: 'Gallery' } }
  onMoved={ ( splide, newIndex, prevIndex, destIndex ) => setIndex( newIndex ) }
>
  { slides }
</Splide>
```

How they work (source, `components/Splide/Splide.tsx`):

- In `componentDidMount`, React Splide creates the core instance, then calls `splide.on( event, ( ...args ) => handler( splide, ...args ) )` for every event prop that is a function, then calls `mount()`. Because the props are registered before `mount()`, `onMounted`, `onReady` and the `*:mounted` props fire.
- Every handler gets the **core instance first**, then the core event's arguments. The core's own `on()` passes only the event's arguments.
- The props are read **once**. A handler that reads props or state sees the first render's values forever, and a handler added, removed or replaced later is ignored. See [Stale handlers](#stale-handlers).
- They are core handlers, so a breakpoint `destroy` removes them for good. See [Breakpoint destroy drops the event props](#breakpoint-destroy-drops-the-event-props).
- `onClick` is Splide's slide `click` event, typed `( splide, Slide, e: MouseEvent )`. It is not a DOM click on the root.
- The `<Splide>` prop type removes React's DOM event props (`SplideHTMLAttributes` omits `DOMAttributes`), so `onKeyDown`, `onMouseEnter` and the like are type errors there. Put DOM handlers on a wrapper element, or on the root through `ref.current?.splideRef.current` in an effect.
- Names must match exactly. `onLazyloadLoaded` (lowercase `l`) doesn't exist: React logs "Unknown event handler property" and ignores it. The prop is `onLazyLoadLoaded`.

Typing a handler outside JSX:

```tsx
const onMoved: SplideEventHandlers['onMoved'] = ( splide, newIndex, prevIndex, destIndex ) => {
  console.log( newIndex, splide.length );
};
```

## Event prop table

All 28 props. Arguments after `splide` are the core event's arguments. **(tested)** Every prop below received the core instance first, and the argument shapes matched (for example `onArrowsUpdated` got two buttons and two numbers, `onPaginationMounted` got `{ list, items }` and `{ li, button, page }`).

| Prop | Core event | Handler arguments | Fired |
|---|---|---|---|
| `onMounted` | `mounted` | `splide` | Right after all components are mounted |
| `onReady` | `ready` | `splide` | Right after `mounted`, once the state is `IDLE` |
| `onClick` | `click` | `splide, Slide: SlideComponent, e: MouseEvent` | When the user clicks any slide |
| `onMove` | `move` | `splide, newIndex: number, prevIndex: number, destIndex: number` | Right before the carousel moves |
| `onMoved` | `moved` | `splide, newIndex: number, prevIndex: number, destIndex: number` | Right after the carousel moves |
| `onActive` | `active` | `splide, Slide: SlideComponent` | When a slide becomes active |
| `onInactive` | `inactive` | `splide, Slide: SlideComponent` | When the active slide becomes inactive |
| `onVisible` | `visible` | `splide, Slide: SlideComponent` | When a slide becomes visible in the viewport |
| `onHidden` | `hidden` | `splide, Slide: SlideComponent` | When a slide becomes hidden from the viewport |
| `onRefresh` | `refresh` | `splide` | When Splide refreshes the carousel |
| `onUpdated` | `updated` | `splide, options: Options` | Whenever options are updated |
| `onResize` | `resize` | `splide` | Whenever the window is resized |
| `onResized` | `resized` | `splide` | Whenever Splide finishes resizing the carousel |
| `onDrag` | `drag` | `splide` | When the user starts dragging the carousel |
| `onDragging` | `dragging` | `splide` | While the user is dragging the carousel |
| `onDragged` | `dragged` | `splide` | Right after the user finishes dragging the carousel |
| `onScroll` | `scroll` | `splide` | Right before Splide scrolls the carousel (Scroll component) |
| `onScrolled` | `scrolled` | `splide` | Right after Splide finishes scrolling the carousel (Scroll component) |
| `onDestroy` | `destroy` | `splide` | When Splide destroys the carousel, including on unmount |
| `onArrowsMounted` | `arrows:mounted` | `splide, prev: HTMLButtonElement, next: HTMLButtonElement` | Right after the arrows are mounted |
| `onArrowsUpdated` | `arrows:updated` | `splide, prev: HTMLButtonElement, next: HTMLButtonElement, prevIndex: number, nextIndex: number` | Whenever the arrows' status is updated |
| `onPaginationMounted` | `pagination:mounted` | `splide, data: PaginationData, item: PaginationItem` | Right after the pagination is mounted |
| `onPaginationUpdated` | `pagination:updated` | `splide, data: PaginationData, prev: PaginationItem, curr: PaginationItem` | Whenever the pagination's status is updated |
| `onNavigationMounted` | `navigation:mounted` | `splide, splides` (typed `Splide[]`; **(source)** actually `SyncTarget[]`) | Right after a navigation carousel (`isNavigation: true`) is mounted, and again on `sync()` |
| `onAutoplayPlay` | `autoplay:play` | `splide` | When autoplay starts |
| `onAutoplayPlaying` | `autoplay:playing` | `splide, rate: number` (0 to 1) | While the autoplay timer is ticking |
| `onAutoplayPause` | `autoplay:pause` | `splide` | Right after autoplay is paused |
| `onLazyLoadLoaded` | `lazyload:loaded` | `splide, img: HTMLImageElement, Slide: SlideComponent` | Right after the LazyLoad component loads an image |

## Events without a prop

These have no prop. Listen with `splide.on()`, in an extension or after mount (next section):

| Event | Arguments | Notes |
|---|---|---|
| `overflow` | `isOverflow: boolean` | v4.1.0+. First emitted while Layout mounts, so after-mount listeners miss the initial value. See [`overflow`](#overflow) |
| `sk`, `sh`, `ei` | see [Internal events](#internal-events-from-source) | Internal, undocumented |
| `video:play`, `video:pause`, `video:ended`, `video:error`, `video:click` | `player: Player` | From `@splidejs/splide-extension-video` (`references/extensions/video.md`) |
| `intersection`, `intersection:in`, `intersection:out` | `entry: IntersectionObserverEntry` | From `@splidejs/splide-extension-intersection` (`references/extensions/intersection.md`) |
| Your own names | anything | See [Emitting custom events](#emitting-custom-events) |

The AutoScroll, Grid and URL Hash extensions emit no events of their own.

## Where to register a handler

| Registration | Sees the mount-time events | Runs, relative to Splide's own handlers | Survives a breakpoint `destroy` remount | Sees fresh props/state |
|---|---|---|---|---|
| Event prop | All of them | **Before** (registered before `mount()`) | **No** | No, unless you use a ref |
| Extension `setup()` | All of them: `resized`, `overflow`, `arrows:mounted`, `pagination:mounted`, `autoplay:play`, `navigation:mounted`, `mounted`, `active`, `ready` | Before (built-in components register theirs in `mount()`, after every `setup()`) | Yes | Through a ref or a state setter |
| Extension `mount()` | Only `mounted` (after Move's reposition), `active`/`visible`, `ready` | After | Yes | Through a ref or a state setter |
| `splide.on()` in `useEffect`/`useLayoutEffect` | **None** (mount is over) | After | No | Through a ref or a state setter |

**(tested)** An extension that registered the same names in `setup()` and `mount()`, plus an effect: `setup()` saw `resized, overflow, arrows:mounted, autoplay:play, pagination:mounted, mounted, active, ready`; `mount()` saw `active, mounted, ready`; the effect saw none. A vanilla page with `isNavigation: true` added `navigation:mounted` to the `setup()` list only.

**(tested)** Order for the same event: an `onMoved` prop ran while the previous slide still had `is-active`, and a `moved` handler registered in an extension's `mount()` ran after the classes had moved; an `onRefresh` prop saw `splide.length` as `3` and the `mount()` handler saw `4` after a slide was added. A handler registered in `setup()` sits before the components, like a prop. Use the event arguments (`newIndex`, `Slide`) rather than reading DOM classes or `length` early. **(source)** `refresh()` and `updated` re-register some components' handlers (Pagination, Clones, Arrows), which moves them to the end of the list, so don't build on the exact order.

### In an extension

Extensions are the React way to reach the core before `mount()`. Pass them in the `extensions` prop, which is read once at mount and reused on every breakpoint remount. Keep the object stable (module scope, or `useState`), and read changing values through a ref or a state setter.

**(tested)** Arrows, pagination and drag only when the slides overflow: at 1280px the four 200px slides fit and there were no arrows, no pagination and no `splide--draggable`; at 600px all three appeared and the root had `is-overflow`; back at 1280px they were gone:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type Extension = NonNullable<SplideProps['extensions']>[ string ];

const logos = [ 'Acme', 'Globex', 'Initech', 'Umbrella' ];

export default function Logos() {
  const [ overflow, setOverflow ] = useState( true );

  // There is no onOverflow prop. `overflow` is first emitted while Layout mounts, so register it in
  // an extension's setup(): mount() runs after the built-in components and misses the initial value.
  // useState keeps one extensions object for the component's lifetime (<Splide> reads it once, at mount).
  const [ extensions ] = useState<Record<string, Extension>>( () => ( {
    OverflowWatcher: splide => ( {
      setup() {
        splide.on( 'overflow', isOverflow => setOverflow( isOverflow ) );
      },
    } ),
  } ) );

  const options: Options = {
    label     : 'Our customers',
    fixedWidth: 200,
    gap       : 16,
    arrows    : overflow,
    pagination: overflow,
    drag      : overflow,
  };

  return (
    <Splide id="logos" options={ options } extensions={ extensions }>
      { logos.map( name => <SplideSlide key={ name }>{ name }</SplideSlide> ) }
    </Splide>
  );
}
```

- `setOverflow` never changes identity, so capturing it once is safe. For anything else, read it from a latest-value ref inside the handler.
- `splide.on( 'overflow', ... )` is typed through core's `EventMap`, so `isOverflow` is a `boolean`.
- The options come from state: `setOverflow` re-renders, the `options` prop changes, and React Splide assigns it (`updated`). That is the React version of the Overflow guide's `splide.options = { arrows: isOverflow, ... }`.
- `Extension` is derived from the prop types, so you don't need `@splidejs/splide` installed for it.
- In development, StrictMode creates two instances; the extension runs for each, so each registers its own handler.

### After mount, and when that is too late

In an effect, the instance exists and is `IDLE`. Capture it in the effect, so the cleanup talks to the same instance (StrictMode runs the effect for each of its two instances), and use a namespace so `off()` removes only your handler:

**(tested)** The label read "fits" at 1280px, "overflowing" at 600px and "fits" again at 1280px:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Listening after mount: fine for later changes, but the initial `overflow` event has already fired.
export default function OverflowAfterMount() {
  const ref = useRef<Splide>( null );
  const [ overflow, setOverflow ] = useState<boolean | null>( null );

  useEffect( () => {
    const splide = ref.current?.splide;
    if ( ! splide ) return;

    setOverflow( splide.Components.Layout.isOverflow() ); // read the initial value
    splide.on( 'overflow.app', setOverflow );             // then follow changes

    return () => {
      splide.off( 'overflow.app' );
    };
  }, [] );

  return (
    <>
      <Splide ref={ ref } options={ { label: 'Logos', fixedWidth: 200, gap: 16 } }>
        <SplideSlide>Acme</SplideSlide>
        <SplideSlide>Globex</SplideSlide>
        <SplideSlide>Initech</SplideSlide>
        <SplideSlide>Umbrella</SplideSlide>
      </Splide>
      <p id="overflow">{ overflow === null ? 'unknown' : overflow ? 'overflowing' : 'fits' }</p>
    </>
  );
}
```

After mount is too late for every event emitted during `mount()`. Read the state instead, or register in an extension's `setup()`:

| Missed event | Read the current value instead |
|---|---|
| `mounted`, `ready` | The effect itself runs after both |
| `overflow` | `splide.Components.Layout.isOverflow()`, or the root's `is-overflow` class |
| `resized` | Nothing to read: measure if you need sizes |
| `arrows:mounted` | `splide.Components.Arrows.arrows` (`{ prev, next }`) |
| `pagination:mounted` | `splide.Components.Pagination.items` |
| `navigation:mounted` | `splide.splides` (but with the usual React sync pattern it fires again during `sync()`, which runs in an effect) |
| `autoplay:play` | `! splide.Components.Autoplay.isPaused()` |
| `active`, `visible` for the first slides | `splide.index`, or `splide.Components.Slides.get()` and the `is-visible` class |
| `lazyload:loaded` for images already loaded | `img.complete` |

An after-mount listener is also removed for good by a breakpoint `destroy` (the effect doesn't run again). Use an extension when the carousel has `destroy` breakpoints.

## Multiple events, arrays and namespaces

On the core instance (`ref.current?.splide`, the `splide` argument, or inside an extension):

```tsx
// Listens to a single event:
splide.on( 'move', () => {} );

// Listens to multiple events:
splide.on( 'move resize', () => {} );

// Appends a namespace:
splide.on( 'move.myNamespace resize.myNamespace', () => {} );

// (source) An array works too; its items are joined with spaces:
splide.on( [ 'moved', 'scrolled' ], () => {} );
```

- Separate several names with spaces. One callback is registered for each name.
- `name.namespace` adds a namespace. It only matters for `off()`: a namespaced handler still fires on the plain event (`move.myNamespace` fires on every `move`).
- **(source)** Only the first dot counts. `'move.a.b'` has the namespace `a`.
- **(source)** The callback receives the event's arguments spread out, without the instance (unlike event props), and without an `Event` object. Its `this` is the callback function itself.
- **(source)** Events are `CustomEvent`s dispatched on a private `DocumentFragment` (`splide.event.bus`), not on the root element. `splideRef.current.addEventListener( 'move', ... )` does **not** work.
- Registering the same function twice creates two handlers. There is no de-duplication, which matters in effects that run more than once: always clean up.

## Removing handlers and cleanup

`off( events: string | string[] ): this` removes the registered handlers for the given event(s). **(source)** Matching is exact on both the event name and the namespace:

| Call | Removes |
|---|---|
| `off( 'move' )` | `move` handlers registered **without** a namespace. `move.myNamespace` handlers stay |
| `off( 'move.myNamespace' )` | Only `move` handlers in `myNamespace` |
| `off( 'move.ns resize.ns' )` | `move` and `resize` handlers in `ns` |
| `off( '.ns' )` | Nothing. Name each event |

(Tested in vanilla: after `off( 'move' )` the namespaced handler still fired; `off( '.ns' )` removed nothing; `off( 'move.ns' )` removed it.)

You cannot remove one specific function: `off()` takes no callback, and the internal binder stores a wrapper rather than your function. Give each handler a namespace.

Careful: `off( 'moved' )` without a namespace also removes the `onMoved` prop, because React Splide registers the props without one (tested: after `splide.off( 'moved' )` the prop stopped firing, while the carousel itself kept working). Always namespace your own handlers.

Cleanup rules in React:

- Handlers registered with `on()` in an effect: remove them in the effect's cleanup with their namespace, using the instance captured in the effect.
- Handlers on an unmounting carousel need no cleanup: unmount calls `destroy()`, which removes every handler, including the event props and the extensions' handlers.
- DOM listeners you add yourself (`window.addEventListener`) are **not** removed by Splide. Remove them in the effect cleanup, or in the extension's `destroy()`.

## Emitting custom events

`emit( event: string, ...args: any[] ): this` triggers every handler for that name and passes `args` to them. Any string works as a name, so extensions and components can use the bus:

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

(Tested: the handler received `2, 'button'`.)

**(source)** Emitting a built-in name (for example `splide.emit( 'refresh' )`) also triggers the internal components that listen to it. `splide.refresh()` is literally `emit( 'refresh' )`. There is no event prop for custom names.

## Timing and order

- **Synchronous.** Handlers run during the core call that emits the event, in registration order. Event props come first (see [Where to register a handler](#where-to-register-a-handler)), so in a prop handler the DOM classes and some instance values (`length` during `refresh`) are not updated yet. Use the arguments and `splide.index`.
- **During React's commit.** When the `options` prop changes or the slide list changes, React Splide calls `splide.options = ...` or `splide.refresh()` in `componentDidUpdate`. `updated`, `refresh` and what they trigger (`resized`, `overflow`, `arrows:mounted`, `pagination:mounted`, `active`, ...) are then emitted inside React's commit, before the parent's `useLayoutEffect` and `useEffect` run. **(tested)** An `onUpdated` handler read the new value from a ref written during render, and the previous value from a ref written in `useEffect`.
- **State updates are fine.** Calling a state setter from a handler schedules a render; React batches several calls. React doesn't warn when the handler runs during mount or commit.
- **High-frequency events.** `onAutoplayPlaying` fires on every animation frame, `onDragging` on every pointer move, `onResize` on every throttled window resize. Don't set state from them. Write to the DOM through a ref (see [`autoplay:playing`](#autoplayplaying)).
- **StrictMode (development).** The carousel mounts, is destroyed and mounts again, so **(tested)** `onMounted`, `onReady`, `onArrowsMounted`, `onPaginationMounted` and `onAutoplayPlay` fire twice, with one `onDestroy` in between. Production fires each once. Make mount handlers idempotent.
- **Unmount.** `onDestroy` fires during `componentWillUnmount`. Other components may already be gone; don't read the DOM outside the carousel there.
- **Don't change slides mid-move.** A slide-list change during a transition refreshes the carousel and loses `moved` (see `references/apis.md`).

## Stale handlers

React Splide registers each event prop once, at mount, and never reads it again. A handler that closes over props or state keeps the values from the first render.

**(tested)** After three clicks on the counter, a move reported `stale 0, latest 3`:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function StaleVsLatest() {
  const [ count, setCount ] = useState( 0 );
  const [ report, setReport ] = useState( '' );

  // Latest-value ref: written on every render, read when the event fires.
  const latest = useRef( count );
  latest.current = count;

  return (
    <>
      <button type="button" id="inc" onClick={ () => setCount( c => c + 1 ) }>Clicked { count } times</button>

      <Splide
        options={ { label: 'Demo' } }
        onMoved={ () => {
          // `count` belongs to the first render: <Splide> bound this function at mount and never re-reads it.
          setReport( `stale ${ count }, latest ${ latest.current }` );
        } }
      >
        <SplideSlide>1</SplideSlide>
        <SplideSlide>2</SplideSlide>
        <SplideSlide>3</SplideSlide>
      </Splide>

      <p id="report">{ report }</p>
    </>
  );
}
```

Fixes, in order of preference:

1. **Don't read changing values.** A handler that only calls state setters (`setIndex( newIndex )`, `setCount( c => c + 1 )`) is never stale, because setters keep their identity and functional updates get the current state.
2. **Latest-value ref** (above): `const latest = useRef( value ); latest.current = value;`, then read `latest.current` in the handler. Works for props too, such as a callback from a parent (`const onChange = useRef( props.onChange ); onChange.current = props.onChange;`).
3. **Stable wrappers for several handlers** (below).

Why the ref is written during render, not in an effect: events emitted during the commit (`updated`, `refresh` and the events they cause) run before effects, so an effect-written ref still holds the previous value then (tested). React's docs advise against writing refs during render, and React Compiler lint rules may flag it; the write is idempotent, so the main cost is that lint warning. If the commit-time events don't matter to you, `useLayoutEffect( () => { latest.current = value; } )` is the lint-clean variant.

**(tested)** A hook that keeps the newest version of each handler. After picking two products and switching the mode, a click removed instead of adding, so the handler saw the new `mode`:

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideEventHandlers } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Keeps the newest version of a set of handlers; the functions passed to <Splide> never change.
function useLatestHandlers( handlers: Partial<SplideEventHandlers> ) {
  const latest = useRef( handlers );
  latest.current = handlers;

  const [ stable ] = useState( () => {
    const out: Partial<SplideEventHandlers> = {};
    ( Object.keys( handlers ) as ( keyof SplideEventHandlers )[] ).forEach( name => {
      ( out as Record<string, unknown> )[ name ] = ( ...args: unknown[] ) =>
        ( latest.current[ name ] as ( ( ...a: unknown[] ) => void ) | undefined )?.( ...args );
    } );
    return out;
  } );

  return stable;
}

export default function Picker() {
  const [ picked, setPicked ] = useState<string[]>( [] );
  const [ mode, setMode ] = useState<'add' | 'remove'>( 'add' );

  const handlers = useLatestHandlers( {
    onClick: ( splide, Slide ) => {
      const id = Slide.slide.dataset.id ?? '';
      // `mode` is current here, because the stable wrapper calls the newest handler
      setPicked( list => mode === 'add' ? [ ...list, id ] : list.filter( item => item !== id ) );
    },
  } );

  return (
    <>
      <button type="button" id="mode" onClick={ () => setMode( m => m === 'add' ? 'remove' : 'add' ) }>Mode: { mode }</button>
      <Splide options={ { label: 'Products', perPage: 3 } } { ...handlers }>
        { [ 'a', 'b', 'c' ].map( id => <SplideSlide key={ id } data-id={ id }>Product { id }</SplideSlide> ) }
      </Splide>
      <p id="picked">{ picked.join( ',' ) }</p>
    </>
  );
}
```

- The set of handler names is fixed at the first render (that's all `<Splide>` reads anyway). Pass every handler you'll need from the start; a handler can be a no-op.
- `data-id` on `<SplideSlide>` reaches the `<li>`, so `Slide.slide.dataset.id` identifies the slide, also for clones.
- A remount with a `key` re-reads all props, but it also recreates the carousel (index, autoplay and sync reset). Use it for options that are read once, not for stale handlers.

## Breakpoint destroy drops the event props

A breakpoint with `destroy: true` calls `splide.destroy( false )`, which removes **every** handler on the bus, the event props included. When the breakpoint stops matching, the core calls `mount()` again, but React Splide binds the props only in `componentDidMount`, so they stay gone. **(tested)** After a destroy at 500px and a remount at 1280px, `onMoved` and `onMounted` never fired again.

Extensions are reused by that remount, and their `setup()`/`mount()` run again. Register handlers there:

**(tested)** The position read "article 3" after one move, "article 1" after the destroy at 500px, and "article 5" after two moves on the remounted carousel at 1280px:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type Extension = NonNullable<SplideProps['extensions']>[ string ];

const options: Options = {
  label      : 'Articles',
  perPage    : 2,
  breakpoints: { 640: { destroy: true } }, // a plain list below 640px, a carousel again above it
};

export default function Articles() {
  const [ index, setIndex ] = useState( 0 );

  // Breakpoint destroy removes every handler, including the event props, and the remount does not
  // bind the props again. An extension's setup() runs on every remount, so register there instead.
  const [ extensions ] = useState<Record<string, Extension>>( () => ( {
    IndexWatcher: splide => ( {
      setup() {
        splide.on( 'moved', newIndex => setIndex( newIndex ) );
        splide.on( 'destroy', () => setIndex( 0 ) );
      },
    } ),
  } ) );

  return (
    <>
      <Splide id="articles" options={ options } extensions={ extensions }>
        { [ 1, 2, 3, 4, 5, 6 ].map( n => <SplideSlide key={ n }>Article { n }</SplideSlide> ) }
      </Splide>
      <p id="position">Showing from article { index + 1 }</p>
    </>
  );
}
```

The `destroy` handler resets the label when the carousel turns into a list; it fires for unmount too, which is harmless. For the destroyed layout itself (focusable leftovers, labels), see `references/accessibility.md`.

## Event details

Each event as the core emits it. The prop name is in parentheses; in a prop handler the arguments come after `splide`.

### `mounted`

(`onMounted`) Fired right after all components are mounted. Register it before `mount()`: event props and an extension's `setup()` are; an effect is too late.

**(source)** It fires before the root gets `is-initialized` and before the state becomes `IDLE`. Handlers registered before `mount()` (event props) also run before the Move component's first reposition, so slides do not have `is-active`/`is-visible` yet. Use `onReady` when you need the finished DOM.

### `ready`

(`onReady`) Fired right after `mounted`, by which point the state has changed to `IDLE`.

**(source)** Also emitted inside `mount()`. After a breakpoint remount, both `mounted` and `ready` fire again, but only for handlers registered again by an extension (not for the props).

### `click`

(`onClick`) Fired when the user clicks any slide.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The clicked Slide component |
| `e` | `MouseEvent` | The `MouseEvent` object (a `PointerEvent` in Chrome) |

**(source)**

- Clones fire it too. In loop mode, use `Slide.isClone ? Slide.slideIndex : Slide.index` to get the real index.
- It does not fire for the click that ends a drag: Drag cancels that click in the capture phase on the track. **(tested)** A React `onClick` on a button inside the slide didn't get that click either, while a plain click reached it.
- Splide does not call `preventDefault()`, so links inside slides still navigate.

```tsx
<Splide onClick={ ( splide, Slide ) => setOpened( Slide.isClone ? Slide.slideIndex : Slide.index ) }>
```

(Tested: a click on a clone set a real index.)

### `move`

(`onMove`) Fired right before the carousel moves.

| Argument | Type | Description |
|---|---|---|
| `newIndex` | `number` | The new slide index |
| `prevIndex` | `number` | The previous slide index |
| `destIndex` | `number` | The destination slide index. In loop mode it can be negative or greater than the number of slides, because of the clones |

**(source)** Emitted by `Move.move()` after the state becomes `MOVING`. It is not emitted when `go()` is ignored: same index, no slide to go to, or busy with `waitForTransition`. Free-drag (`drag: 'free'`) scrolling emits `scroll`/`scrolled` instead.

```tsx
<Splide onMove={ ( splide, newIndex ) => setCounter( `${ newIndex + 1 } / ${ splide.length }` ) }>
```

(Tested: "3 / 4" after one page with `perPage: 2`.) Don't change the slide list from `onMove`: the refresh it causes loses `moved` (tested, see `references/apis.md`).

### `moved`

(`onMoved`) Fired right after the carousel moves. Same arguments as `move`.

| Argument | Type | Description |
|---|---|---|
| `newIndex` | `number` | The new slide index |
| `prevIndex` | `number` | The previous slide index |
| `destIndex` | `number` | The destination slide index. In loop mode it can be negative or greater than the number of slides, because of the clones |

**(source)** Fired when the CSS transition ends (`transitionend` on the list). If the speed is `0`, for example under `prefers-reduced-motion`, or the carousel is already in position, the transition is skipped and `moved` fires **synchronously** right after `move`. With `useScroll: true`, `moved` fires just before `scrolled`. With `type: 'fade'`, `moved` fires on the next tick (`setTimeout`), while the opacity transition is still running, whatever the `speed` (tested in vanilla: 11 ms after `move` with `speed: 1000`).

In an `onMoved` prop the slide classes have not moved yet (tested). Use `newIndex`.

### `active`

(`onActive`) Fired when a slide becomes active.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes active |

**(source)** Fired when the slide gains `is-active`: on `moved`, or on `move` with `updateOnMove: true`. Also fired on repositioning (`mounted`, `resized`, `updated`, `refresh`). In loop mode with `cloneStatus: true` (the default), clones of the active slide become active too, so the event can fire more than once per move. Filter with `if ( Slide.isClone ) return;`.

### `inactive`

(`onInactive`) Fired when the active slide becomes inactive.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes inactive |

**(source)** Clones fire it too, as for `active`.

### `visible`

(`onVisible`) Fired when a slide becomes visible in the viewport.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes visible |

**(source)** "Visible" means completely inside the track (whole-pixel rounding). For `type: 'fade'` it means active. The event fires when the slide gains the `is-visible` class: on `moved`, `scrolled`, the internal `sh`, and repositioning. Clones fire it too. The initial visible slides fire during `mount()`, so the prop sees them.

### `hidden`

(`onHidden`) Fired when a slide becomes hidden from the viewport.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes hidden |

**(source)** Fired when a slide loses `is-visible`. A slide that was never visible does not fire `hidden` on mount (tested). To set an initial hidden state, use `onReady` (example in `references/accessibility.md`).

```tsx
import type { SplideEventHandlers } from '@splidejs/react-splide';

// Pause videos that scroll out of view
const onHidden: SplideEventHandlers['onHidden'] = ( splide, Slide ) => {
  Slide.slide.querySelector( 'video' )?.pause();
};
```

### `refresh`

(`onRefresh`) Fired when Splide refreshes the carousel.

**(source)** Emitted by `splide.refresh()`, by `add()` and `remove()`, by Clones when a loop carousel needs more clones (or none), and by Media when a breakpoint changes `direction`. In React, also by React Splide itself when the rendered slide list changes. Components rebuild in response (see `references/apis.md`). A prop handler runs before that rebuild, so `splide.length` still has the old count there (tested); read it in an effect instead.

### `updated`

(`onUpdated`) Fired whenever options are updated, for example when a breakpoint starts or stops matching.

| Argument | Type | Description |
|---|---|---|
| `options` | `Options` | An object with the latest options |

**(source)** Emitted by `Media.set()`: on breakpoint and `prefers-reduced-motion` media-query changes, and on every `splide.options = {...}` assignment, which React Splide makes when the `options` prop changes (deep comparison). Breakpoints and reduced motion applied while the instance is being created or mounted (state `CREATED`) do not emit it. The `options` setter always does, even during `mount()`. On a breakpoint that destroys the carousel, `updated` fires just before `destroy`. A prop change emits it inside React's commit, before your effects run.

```tsx
const onUpdated: SplideEventHandlers['onUpdated'] = ( splide, options ) => {
  console.log( 'perPage is now', options.perPage );
};
```

### `resize`

(`onResize`) Fired whenever the window is resized. Splide reduces the frequency with `requestAnimationFrame`.

**(source)** Also emitted on the window `load` event, after each image that LazyLoad loads, and after all images in a slide added with `add()` finish loading. Layout reacts by re-measuring and emits `resized` only if the root's size changed.

### `resized`

(`onResized`) Fired whenever Splide finishes resizing the carousel.

**(source)** Emitted by `Layout.resize()` when the root's width or height changed, and always on Layout init: mount, `updated`, `refresh`. In a loop carousel, Clones emits it once more during mount, after generating the clones. Move repositions and Controller re-checks the end index in response.

### `drag`

(`onDrag`) Fired when the user starts dragging the carousel.

**(source)** Fired on the first pointer move after a pointer down on a draggable part of the track. At that point Splide may not yet have decided the gesture is a drag, for example a vertical page scroll on touch. The state becomes `DRAGGING`.

### `dragging`

(`onDragging`) Fired while the user is dragging the carousel.

**(source)** Fired on each pointer move once the drag threshold (`dragMinThreshold`) is passed. Don't set state here.

### `dragged`

(`onDragged`) Fired right after the user finishes dragging the carousel.

**(source)** Fired on pointer up if `drag` fired. The state returns to `IDLE`. If the drag threshold was passed, the release then triggers a `go()` (`move`/`moved`) or, with `drag: 'free'`, a scroll (`scroll`/`scrolled`).

### `scroll`

(`onScroll`) Fired right before Splide scrolls the carousel with the Scroll component. It is not fired for the normal CSS transition; use `onMove` instead.

**(source)** The Scroll component is used by `drag: 'free'` (including its bounce-back when a free scroll runs past the edge), by `useScroll: true`, and by `Components.Controller.scroll()`. The state becomes `SCROLLING`. Over-dragging a normal (`drag: true`) carousel past its edge snaps back with `move`/`moved`, not `scroll` (tested in vanilla).

### `scrolled`

(`onScrolled`) Fired right after the Scroll component finishes scrolling the carousel. It is not fired for the normal CSS transition; use `onMoved` instead.

### `overflow`

No prop. Fired when the total width of the slides becomes wider or narrower than the list element as the viewport is resized (v4.1.0+).

| Argument | Type | Description |
|---|---|---|
| `isOverflow` | `boolean` | **(source; the Overflow guide shows it, the Events page does not)** `true` if the slides are wider than the list (enough slides), `false` if not |

**(source)** The root's `is-overflow` class is toggled at the same moment. The event fires once during mount with the initial value, before any extension's `mount()`. Register it in an extension's `setup()` ([example](#in-an-extension)), or read `Components.Layout.isOverflow()` after mount ([example](#after-mount-and-when-that-is-too-late)). For a loop carousel, the Overflow guide also resets the position (`splide.go( 0 )`) and toggles the clones (`clones: isOverflow ? undefined : 0`); see `references/overflow.md`.

### `destroy`

(`onDestroy`) Fired when Splide destroys the carousel.

**(source)** Fired after every component's `destroy()` has run (generated arrows, pagination and clones are already removed) and **before** the handlers are removed, so your `destroy` handler still runs. It fires for breakpoint destroys, for unmount, and in development for StrictMode's destroy of the first instance.

### `arrows:mounted`

(`onArrowsMounted`) Fired right after the arrows are mounted.

| Argument | Type | Description |
|---|---|---|
| `prev` | `HTMLButtonElement` | The previous arrow |
| `next` | `HTMLButtonElement` | The next arrow |

**(source)** Fires only when `arrows` is `true`. It fires again after every `updated`, because Arrows remount then. Emitted while Arrows mounts, before an extension's `mount()`.

### `arrows:updated`

(`onArrowsUpdated`) Fired whenever the arrows' status is updated.

| Argument | Type | Description |
|---|---|---|
| `prev` | `HTMLButtonElement` | The previous arrow |
| `next` | `HTMLButtonElement` | The next arrow |
| `prevIndex` | `number` | Docs: "An old index". **(source)** The slide index the previous arrow goes to (`Controller.getPrev()`), `-1` if none (arrow disabled) |
| `nextIndex` | `number` | Docs: "A new index". **(source)** The slide index the next arrow goes to (`Controller.getNext()`), `-1` if none (arrow disabled) |

**(source)** Emitted on `mounted`, `moved`, `refresh`, `scrolled` and `ei`, and while the arrows mount.

```tsx
// Hide arrows instead of disabling them
const onArrowsUpdated: SplideEventHandlers['onArrowsUpdated'] = ( splide, prev, next, prevIndex, nextIndex ) => {
  prev.hidden = prevIndex < 0;
  next.hidden = nextIndex < 0;
};
```

(Tested: the previous arrow was hidden at the start and the next arrow at the end.) With arrows you render in JSX, React won't reset `hidden` unless you pass a `hidden` prop that changes.

### `pagination:mounted`

(`onPaginationMounted`) Fired right after the pagination is mounted.

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | An object with pagination data |
| `item` | `PaginationItem` | Data of the active item |

**(source)** Fires only when `pagination` is `true`. The pagination is rebuilt, and this event fires again, on every `updated`, `refresh` and `ei`, so changes made here survive rebuilds. React doesn't own the pagination elements, so changing them directly is fine.

```tsx
// Visible page numbers. Pagination is rebuilt (and this fires again) on every update and refresh.
const onPaginationMounted: SplideEventHandlers['onPaginationMounted'] = ( splide, data ) => {
  data.list.classList.add( 'my-pagination' );
  data.items.forEach( item => {
    item.button.textContent = String( item.page + 1 );
  } );
};
```

(Tested: the dots read "1" and "2".)

### `pagination:updated`

(`onPaginationUpdated`) Fired whenever the pagination's status is updated.

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | An object with pagination data |
| `prev` | `PaginationItem` | Data of the previous item |
| `curr` | `PaginationItem` | Data of the current item |

**(source)** Emitted on `move`, `scroll` and `scrolled` (so it fires before the transition ends), and once while mounting. On the first call `prev` and `curr` are the same item. Either can be `undefined` in edge cases, and the source guards for that.

### `navigation:mounted`

(`onNavigationMounted`) Fired right after the navigation carousel (`isNavigation: true`) is mounted, and again whenever `sync()` remounts its Sync component.

| Argument | Type | Description |
|---|---|---|
| `splides` | Docs and types: `Splide[]` | Docs: "An array with Splide instances to navigate". **(source)** `Splide.splides`, which is an array of `SyncTarget` objects (`{ splide: Splide, isParent?: boolean }`), not bare instances |

**(tested)** With the React sync pattern (`sync()` in an effect), it fired at mount with an empty array, then during `sync()` with one target, the main carousel, marked `isParent`. The handler type is wrong, so cast:

```tsx
// The type says Splide[], the value is SyncTarget[] ({ splide, isParent? }).
type SyncTarget = { splide: NonNullable<Parameters<SplideEventHandlers['onNavigationMounted']>[ 0 ]>; isParent?: boolean };

const onNavigationMounted: SplideEventHandlers['onNavigationMounted'] = ( splide, splides ) => {
  ( splides as unknown as SyncTarget[] ).forEach( target => {
    console.log( target.splide.root.id, target.isParent ? '(parent)' : '' );
  } );
};
```

### `autoplay:play`

(`onAutoplayPlay`) Fired when autoplay starts.

**(source)** Fired by `Autoplay.play()` only when autoplay was paused and there are enough slides: at mount (unless `autoplay: 'pause'` or reduced motion), when hover or focus ends, on the toggle button, and on `Components.Autoplay.play()`. The live region switches to `aria-live="off"`.

### `autoplay:pause`

(`onAutoplayPause`) Fired right after autoplay is paused.

**(source)** Fired only if autoplay was running: on hover (`pauseOnHover`), on focus (`pauseOnFocus`), on the toggle button, and on `Components.Autoplay.pause()`. The live region switches back to `aria-live="polite"`. Because hover and focus pauses fire it too, it doesn't tell you whether the user stopped autoplay (see `references/accessibility.md` for a toggle).

### `autoplay:playing`

(`onAutoplayPlaying`) Fired while the autoplay timer is ticking.

| Argument | Type | Description |
|---|---|---|
| `rate` | `number` | The current progress, from 0 to 1 |

**(source)** Fired on every animation frame while autoplay runs. Splide also sets `.splide__progress__bar`'s `width` to `rate * 100%` if the bar exists inside the root. For a bar elsewhere, write to it through a ref rather than state:

```tsx
const barRef = useRef<HTMLDivElement>( null );

<Splide
  options={ { label: 'Snippets', type: 'loop', perPage: 2, autoplay: true, interval: 2000 } }
  onAutoplayPlaying={ ( splide, rate ) => {
    if ( barRef.current ) barRef.current.style.width = `${ rate * 100 }%`;
  } }
>
  { slides }
</Splide>
<div className="my-progress"><div ref={ barRef } className="my-progress__bar" /></div>
```

(Tested: the bar's width grew.) The ref object never changes, so the handler can't go stale.

### `lazyload:loaded`

(`onLazyLoadLoaded`, capital `L` in `Load`) Fired right after the LazyLoad component loads an image.

| Argument | Type | Description |
|---|---|---|
| `img` | `HTMLImageElement` | The loaded image element |
| `Slide` | `SlideComponent` | The Slide component that contains the image |

**(source)** Not fired on image `error` (only the `is-loading` class is removed). Each load is followed by a `resize` emit. In JSX, write `data-splide-lazy` on the `<img>` (and leave `src` out, or use a placeholder).

## Internal events (from source)

These are defined after an `/** @internal */` marker in `constants/events.ts`. The docs do not list them, they have no props, and they may change. The string values are short codes, not descriptive names.

| Constant | String | Arguments | Emitted by | Used by |
|---|---|---|---|---|
| `EVENT_SLIDE_KEYDOWN` | `'sk'` | `Slide: SlideComponent`, `e: KeyboardEvent` | `keydown` on each slide element, clones included | Sync: Space/Enter on navigation slides |
| `EVENT_SHIFTED` | `'sh'` | none | `Move.translate()` without `preventLoop`, when a loop carousel's position wraps past the clones. In practice this happens while dragging, during Scroll-component scrolling (free drag, `useScroll`), and on `jump()`. A normal `go()` transition does not emit it | Slides (status update) |
| `EVENT_END_INDEX_CHANGED` | `'ei'` | none | Controller, on `resized`, when the last reachable index changed (e.g. `autoWidth`/`fixedWidth`) | Controller, Arrows, Pagination |

Use the strings: `splide.on( 'sk', ( Slide: SlideComponent, e: KeyboardEvent ) => { ... } )`. Don't import the constants from `@splidejs/splide` at runtime (it adds a second copy of core).

## Argument types

`SlideComponent`:

| Property | Type | Description |
|---|---|---|
| `index` | `number` | Slide index. For clones, the clone's position: negative or `>= length` |
| `slideIndex` | `number` | For clones, the index of the original slide. `-1` for real slides |
| `slide` | `HTMLElement` | The slide element (the `<li>` of a `<SplideSlide>`, or a clone of it) |
| `container` | `HTMLElement \| undefined` | The `.splide__slide__container` child, if any |
| `isClone` | `boolean` | Whether this is a clone |
| `update()` | `() => void` | Re-evaluates classes and attributes |
| `style( prop, value, useContainer? )` | `void` | Sets an inline style on the slide or its container |
| `isWithin( from, distance )` | `boolean` | Whether this slide is within `distance` slides of index `from` |

`PaginationData`:

| Property | Type | Description |
|---|---|---|
| `list` | `HTMLUListElement` | The UL element that contains the pagination items |
| `items` | `PaginationItem[]` | An array of PaginationItem objects |

`PaginationItem`:

| Property | Type | Description |
|---|---|---|
| `li` | `HTMLLIElement` | The list item element |
| `button` | `HTMLButtonElement` | The dot's button element |
| `page` | `number` | The page index (zero-based) |

`SyncTarget` (source): `{ splide: Splide, isParent?: boolean }`.

In TypeScript, derive them from React Splide's exported types, so `@splidejs/splide` doesn't have to be a dependency:

```tsx
import type { SplideEventHandlers } from '@splidejs/react-splide';

type SlideComponent = Parameters<SplideEventHandlers['onActive']>[ 1 ];
type PaginationData = Parameters<SplideEventHandlers['onPaginationMounted']>[ 1 ];
type PaginationItem = Parameters<SplideEventHandlers['onPaginationMounted']>[ 2 ];
```

More in `references/typescript.md`.

## Event sequences

From source, in emit order.

**`<Splide>` mount (`componentDidMount`)**

1. `new Splide( root, options )`. React Splide registers the event props.
2. `mount( extensions, transition )`: components are constructed and set up (built-ins, then extensions, then Transition; your extension's `setup()` runs here). Media applies breakpoints and reduced motion; no `updated` is emitted.
3. Components mount in order: Media, Direction, Elements, Slides, Layout (`resized`, `overflow`), Clones (`resized` again in loop mode, after generating clones), Move, Controller, Arrows (`arrows:updated`, then `arrows:mounted`), Autoplay (`autoplay:play` if it starts), Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination (`pagination:updated`, then `pagination:mounted`), Sync (`navigation:mounted` if `isNavigation`), Wheel, Live, then extensions (your `mount()`), then Transition.
4. `mounted`. Handlers run in order: the props first, then Move's reposition (which emits `active` and `visible` for the initial slides), then Arrows' `arrows:updated`.
5. The root gets `is-initialized`, and the state becomes `IDLE`.
6. `ready`.
7. Back in React: the parent's layout effects, then its effects (your `sync()` and `splide.on()` calls).

**(tested)** A slide carousel with autoplay emitted: `resized`, `overflow`, `arrows:updated`, `arrows:mounted`, `autoplay:play`, `pagination:updated`, `pagination:mounted`, `mounted`, `active`, `visible` (per visible slide), `arrows:updated`, `ready` (vanilla, same core).

**`go()` with a CSS transition**

1. State `MOVING`, then `move`. Pagination emits `pagination:updated`. With `updateOnMove`, slide status events fire here. In loop mode, the list may first be repositioned silently across the clones, with no event.
2. Transition end: state `IDLE`, then `moved`. Slides emit `inactive`/`active` and `hidden`/`visible`. Arrows emit `arrows:updated`. Live toggles `aria-busy`.

**Drag (non-free)**

`drag`, then `dragging` repeatedly, then `dragged`, then the `go()` sequence (`move` and `moved`).

**`options` prop change (`componentDidUpdate`)**

`splide.options = options` emits `updated`; Layout re-measures (`resized`, `overflow` if it changed), Arrows remount (`arrows:mounted`), Pagination rebuilds (`pagination:mounted`), Move repositions. All of it before your effects run.

**Slide list change (`componentDidUpdate`)**

`splide.refresh()` emits `refresh`; components rebuild as for `updated`.

**Breakpoint with `destroy: true`**

`updated`, then `destroy` (handlers removed, event props included). On leaving the breakpoint, a full `mount()` sequence runs again, seen only by extension handlers.

**Unmount (`componentWillUnmount`), and `destroy()`**

Components are destroyed in reverse order, then `destroy` is emitted, then all handlers are removed, and the state becomes `DESTROYED`. React Splide then clears `ref.current.splide`.

**Development (StrictMode)**

Mount sequence, then the unmount sequence, then a second mount sequence on a new instance, then your effects.

## Docs vs source

| Event / topic | Docs say | Source says |
|---|---|---|
| Event props (React page) | "You can listen to all Splide events through the Splide component" | 28 props. No prop for `overflow`, the internal events or any extension event |
| Prop naming rule (React page) | Add "on", camel-case, remove colons ("arrows:mounted" becomes "onArrowsMounted") | The rule gives `onLazyloadLoaded`, but the prop is `onLazyLoadLoaded` |
| Handler arguments (React page) | The splide instance first, then the original arguments | Correct. The props are also read only once, at mount, which the page doesn't say |
| `overflow` | No arguments listed (Events page) | `emit( EVENT_OVERFLOW, overflow )`: one `boolean`. The Overflow guide and the `EventMap` type agree with the source |
| `arrows:updated` | `prevIndex`: "An old index", `nextIndex`: "A new index" | `Controller.getPrev()` / `getNext()`: the target index of each arrow, `-1` when unavailable |
| `navigation:mounted` | `splides: Splide[]` (the `EventMap` type and `SplideEventHandlers` agree) | `Splide.splides`, an array of `SyncTarget` objects `{ splide, isParent? }` |
| `off( 'move' )` | Removes all `move` handlers | Removes only un-namespaced `move` handlers, which include the event props |
| `resize` | "Whenever the window is resized" | Also on window `load`, after each LazyLoad image, and after images in `add()`-ed slides load |
| Internal events | Not documented | `sk`, `sh`, `ei` (see above) |
