# Custom extensions

Source: https://splidejs.com/guides/extension/ and https://splidejs.com/integration/react-splide/

Writing your own Splide components in TypeScript and passing them to React Splide with the `extensions` prop: lifecycle, options, events, the core helpers and what they cost in the bundle, getting extension state into React, and replacing core components.

Checked against the Splide core 4.1.4 source (`src/js`) and React Splide 0.7.12 (`src/js` and its dist, which bundles its own copy of core 4.1.3; the two core versions differ only in `package.json`). Every example on this page was type-checked with `tsc --strict`, built with Vite and run in Chrome with React 19, both as a production build and as a development build under `<StrictMode>`.

## Contents

- [Packages, imports and types](#packages-imports-and-types)
- [The component shape](#the-component-shape)
- [Passing extensions to Splide](#passing-extensions-to-splide)
- [Lifecycle (source-verified)](#lifecycle-source-verified)
- [Accessing the instance and other components](#accessing-the-instance-and-other-components)
- [Options for your extension](#options-for-your-extension)
- [Events](#events)
- [Core helpers with React Splide's bundled core](#core-helpers-with-react-splides-bundled-core)
- [Bundle cost of runtime imports from core](#bundle-cost-of-runtime-imports-from-core)
- [Getting extension state into React](#getting-extension-state-into-react)
- [Cleanup](#cleanup)
- [Registering extensions](#registering-extensions)
- [Overriding a built-in component](#overriding-a-built-in-component)
- [Example 1: SlideNumber (the docs example)](#example-1-slidenumber-the-docs-example)
- [Example 2: Slide counter with options, rendered by React](#example-2-slide-counter-with-options-rendered-by-react)
- [Example 3: Keyboard shortcuts](#example-3-keyboard-shortcuts)
- [Example 4: Listeners that survive breakpoint destroy](#example-4-listeners-that-survive-breakpoint-destroy)
- [Pitfalls checklist](#pitfalls-checklist)
- [Docs vs source](#docs-vs-source)

## Packages, imports and types

```bash
npm install @splidejs/react-splide
```

React Splide's dist inlines a full copy of Splide core, so the carousel itself needs nothing else. Extensions need two more things:

- **Core types** (`ComponentConstructor`, `Components`, `BaseComponent`, `KeyboardComponent`, the core `Splide` class). They live in `@splidejs/splide`. npm and Yarn classic hoist it (it is a dependency of React Splide), but pnpm's strict layout only exposes direct dependencies, so add it:

  ```bash
  npm install @splidejs/splide@4.1.4
  ```

  Use `import type`. It is erased at build time and adds 0 bytes (measured, see [Bundle cost](#bundle-cost-of-runtime-imports-from-core)). Import `Options` from React Splide, as everywhere else in this skill: `import type { Options } from '@splidejs/react-splide'`.
- **Runtime helpers** (`EventInterface`, `RequestInterval` and the rest). Importing any value from `@splidejs/splide` works with React Splide's instance, but it bundles a second copy of core. Read [Core helpers](#core-helpers-with-react-splides-bundled-core) and [Bundle cost](#bundle-cost-of-runtime-imports-from-core) before you do it.

React Splide's own types need one tsconfig line under `moduleResolution: "bundler"` (the Vite and Next.js default). See `references/typescript.md`.

Without `@splidejs/splide` as a dependency, derive the core types from React Splide's (tested with `tsc --strict`):

```ts
import type { SplideProps } from '@splidejs/react-splide';

type ComponentConstructor = NonNullable<SplideProps[ 'extensions' ]>[ string ];
type SplideCore          = Parameters<ComponentConstructor>[ 0 ];   // the core instance type
type Components          = Parameters<ComponentConstructor>[ 1 ];
type TransitionComponent = Components[ 'Transition' ];
type MoveComponent       = Components[ 'Move' ];                    // any core component the same way
```

Naming: the React component is called `Splide`, and so is the core class. Name the extension's first parameter `splide` and import the core type as `Splide as SplideCore`.

## The component shape

An extension is a plain function. Splide calls it with `( Splide, Components, options )` and uses the object it returns as the component.

```ts
// From @splidejs/splide dist/types/index.d.ts:
// type ComponentConstructor = ( Splide: Splide, Components: Components, options: Options ) => BaseComponent;
// interface BaseComponent { setup?(): void; mount?(): void; destroy?( completely?: boolean ): void; }
```

| Argument | Type | Description |
|---|---|---|
| `Splide` | `Splide` (core) | The core Splide **instance** (not the class, and not the React component). The same object as `ref.current.splide`. |
| `Components` | `Components` | The shared object of all component objects, keyed by name (`Components.Move`, `Components.Elements`, your own extensions, `Components.Transition`). |
| `options` | `Options` | The instance's live options object. It is the same object for the instance's whole life and is mutated in place by breakpoints and by option updates, including the ones React Splide makes when the `options` prop changes (see [Options](#options-for-your-extension)). |

What the returned object can contain:

| Member | Called by Splide | When |
|---|---|---|
| `setup()` | yes, optional | Right after this component is constructed, before any component's `mount()`. Not mentioned in the guide, but in `BaseComponent` and used by the core `Media` and `Elements`. |
| `mount()` | yes, optional | After every component (core, extensions, Transition) has been constructed and set up. Runs in registration order. |
| `destroy( completely )` | yes, optional | On `splide.destroy()`, including destroys triggered by breakpoints and by unmounting the React component. `completely` is `true` for a final destroy and `false` when breakpoints may remount the slider. |
| anything else | no | Public API, reachable as `splide.Components.<Name>.<member>`. |

The docs' skeleton in TypeScript, with the log lines this page's test checked:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

// Module scope: created once, so the `extensions` object below is stable.
const MyExtension: ComponentConstructor = ( splide, Components, options ) => {
  // Constructor body: core components exist, slides are not registered yet.
  console.log( 'construct', Components.Slides.get().length );

  function setup() {
    console.log( 'setup' );
  }

  function mount() {
    console.log( 'mount', splide.index, Components.Slides.get().length, options.perPage );
  }

  function destroy( completely?: boolean ) {
    console.log( 'destroy', completely );
  }

  return { setup, mount, destroy };
};

const extensions = { MyExtension };

export default function Example() {
  return (
    <Splide extensions={ extensions } options={ { perPage: 2, label: 'My carousel' } }>
      <SplideSlide>01</SplideSlide>
      <SplideSlide>02</SplideSlide>
      <SplideSlide>03</SplideSlide>
    </Splide>
  );
}
```

Logged in a production build: `construct 0`, `setup`, `mount 0 3 2`. In development under StrictMode the log continues with `destroy true`, `construct 0`, `setup`, `mount 0 3 2`: React mounts the component, unmounts it and mounts it again, and React Splide builds a new core instance each time (see [Passing extensions](#passing-extensions-to-splide)).

## Passing extensions to Splide

```tsx
const extensions = { SlideNumber, SlideCounter };   // module scope: one stable object

<Splide extensions={ extensions } options={ { label: 'Products' } }>...</Splide>
```

What React Splide does with the prop (`src/js/components/Splide/Splide.tsx`):

- **It is read once, at mount.** `componentDidMount` runs `new Splide( root, options )`, binds the event props, then calls `splide.mount( extensions, transition )`. `componentDidUpdate` only handles `options` and changed slides. Passing a different `extensions` object later does nothing (tested: an extension added by a state update never mounted). To swap extensions, remount the carousel with a new `key` (tested: the new extension mounted).
- **Keep the constructors stable.** An inline `extensions={ { SlideNumber } }` object works, because the prop is never compared. But a constructor defined inside the component body captures that first render's props and state forever. Define extensions at module scope, or create per-instance factories once with `useState( () => ... )` and hand them changing values through a ref ([Getting extension state into React](#getting-extension-state-into-react)).
- **The prop ends up on the DOM.** `render()` removes only `options` and the event props before spreading the rest onto the root, so the root gets `extensions="[object Object]"` (tested). It is harmless. The `transition` prop is worse: React logs an error in development. `CleanSplide` in `references/custom-transitions.md` fixes both with a small subclass.
- **StrictMode (development only)** mounts, destroys completely and mounts again. Your constructor runs twice, on two different core instances, with a `destroy( true )` in between. An extension that adds DOM nodes and has no `destroy()` shows two of them in development and one in production (tested with the docs' SlideNumber).
- **Unmounting** calls `splide.destroy()`, so every extension gets `destroy( true )`.
- **SSR.** Extensions run only in the browser (from `componentDidMount`), so `window` and `document` are safe inside them. The module that defines them is still imported on the server, so keep module-level code free of browser globals. A file that renders `<Splide>` must be a client component in the Next.js App Router (`references/nextjs.md`).

## Lifecycle (source-verified)

`Splide#mount( Extensions?, Transition? )` in `core/Splide/Splide.ts` does the following. React Splide calls it from `componentDidMount`, after binding the event props, so `onMounted` and `onReady` fire.

1. Asserts the state is `CREATED` or `DESTROYED` (otherwise it throws `Already mounted!`), then sets the state to `CREATED`.
2. Picks the transition: `this._T = Transition || this._T || ( this.is( 'fade' ) ? Fade : Slide )`.
3. Picks the extensions: `this._E = Extensions || this._E`. Calling `mount()` with no argument reuses the previous set. Breakpoint remounts do that.
4. Builds `Constructors = assign( {}, CoreComponents, this._E, { Transition: this._T } )`.
5. For each constructor, **in key order**: `component = Constructor( splide, Components, options )`, `Components[ key ] = component`, then `component.setup && component.setup()`. Each component is constructed and set up before the next one is constructed.
6. For each entry of `Components`, in key order: `component.mount && component.mount()`.
7. Emits `mounted`, adds `is-initialized` to the root, sets the state to `IDLE`, then emits `ready`.

The key order is the 19 core components, then your extensions in the order you listed them, then `Transition`:

`Media, Direction, Elements, Slides, Layout, Clones, Move, Controller, Arrows, Autoplay, Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination, Sync, Wheel, Live, <your extensions...>, Transition`

Logged in Chrome for two extensions: the first constructed and set up, the second constructed and set up, the first mounted, the second mounted, `Transition.mount()`, the `mounted` event, the `ready` event. On `destroy()`: the second extension's `destroy( true )`, the first's, then the `destroy` event.

What is available at each stage of **your** extension:

| Stage | Core components | Slides registered (`Slides.get()`) | Arrows/pagination created | `Components.Transition` | Other extensions |
|---|---|---|---|---|---|
| Constructor body | All constructed and set up. `Elements` has collected `root`, `track`, `list` and `slides` | No, `[]` | No | `undefined` on the first mount. On a breakpoint remount it is the previous, destroyed Transition object | Only the ones listed before yours |
| `setup()` | Same as the constructor | No | No | Same as the constructor | Only the ones listed before yours |
| `mount()` | All mounted | Yes, clones included | Yes | Constructed but not mounted yet | Those listed before yours are mounted |
| `on( 'mounted' )` / `on( 'ready' )` | Everything mounted. `ready` comes after the state is `IDLE` | Yes | Yes | Mounted | All mounted |

So you can destructure `Components.Elements` (`track`, `list`, `root`, `slides`) in the constructor, as the docs' SlideNumber does. Read `Components.Transition`, `Slides.get()` and the arrows inside `mount()` or later.

`Splide#destroy( completely = true )`:

1. If the state is still `CREATED` (destroy was called during `mount()`, for example by a matching `destroy` breakpoint at load), destruction is postponed until `ready`.
2. Otherwise, it calls `component.destroy( completely )` on every component **in reverse order**: Transition first, then the extensions last-to-first, then the core components.
3. It emits `destroy`. Every `EventInterface( splide )` tears itself down on this event.
4. `splide.event.destroy()` removes every handler registered with `splide.on()`, including the ones React Splide registered for your event props.
5. If `completely` is true, it empties `splide.splides` (the sync targets).
6. It sets the state to `DESTROYED`.

Your `destroy()` therefore runs while all event handlers are still attached and other components still exist.

`completely`:

- Unmounting the React component, `splide.destroy()` or `splide.destroy( true )`, and the breakpoint option `destroy: 'completely'` pass `true`. Nothing remounts automatically.
- The breakpoint option `destroy: true` calls `splide.destroy( false )`. When that breakpoint stops matching, `Media` calls `splide.mount()` again on the same instance, with the stored extensions and transition. React is not involved: no re-render, and `ref.current.splide` stays the same object.

**Remounting constructs everything again.** Each `mount()` calls your constructor again, so you get a new closure and state kept inside the constructor is lost. The controller index also restarts from the `start` option. A React remount (StrictMode, a new `key`, navigating away and back) goes further: it creates a new core instance. To carry state across breakpoint remounts, keep it outside the constructor in a factory closure (see [Example 4](#example-4-listeners-that-survive-breakpoint-destroy)).

Remount caveats:

- `splide.root` keeps the `is-initialized` class after a breakpoint destroy. While mounted, the root also has `is-active`, which `Elements` removes on destroy, and the CSS rule `.splide.is-initialized:not(.is-active) .splide__list { display: block }` is what un-flexes a destroyed slider. To test for destroyed, use `splide.state.is( 7 )` (`Splide.STATES.DESTROYED`).
- `Splide.STATES` = `{ CREATED: 1, MOUNTED: 2, IDLE: 3, MOVING: 4, SCROLLING: 5, DRAGGING: 6, DESTROYED: 7 }`. `MOUNTED` is never set by `mount()`, which goes from `CREATED` straight to `IDLE`. Reading `Splide.STATES` needs a runtime import, so write the numbers out (with a comment) to keep core out of the bundle.

## Accessing the instance and other components

Inside an extension, the instance is the first argument. From React, it is `ref.current?.splide` (`undefined` before mount and after unmount), or the first argument of any event prop. The `<Splide>` component itself also has `go( control )` and `sync( splide )`.

Instance members:

| Member | Description |
|---|---|
| `root` | Root element. |
| `index` | Current index (getter, `Controller.getIndex()`). It is already the new index during `move`. |
| `length` | Slide count without clones (getter). |
| `options` | Getter for the live options object. The setter `splide.options = {...}` merges into the current and base options and emits `updated`. React Splide uses the setter when the `options` prop changes. |
| `state` | `{ set( n ), is( n \| n[] ) }`. Read it with `is()`. `set()` is internal. |
| `Components` | Same object as the second argument. |
| `splides` | Sync targets, `[ { splide, isParent? } ]`. |
| `event` | The instance's own `EventInterface` object (`on`, `off`, `emit`, `bind`, `unbind`, `dispatch`, `destroy`). Public in the type declarations but not described in the guides. Handy in React because it needs no import (see [Core helpers](#core-helpers-with-react-splides-bundled-core)). |
| `is( type )` | `true` if `options.type === type`. |
| `go( control )` | `n`, `'+n'`, `'-n'`, `'>'`, `'<'`, `'>n'` (page n). |
| `on`, `off`, `emit` | See [Events](#events). |
| `add( slides, index? )`, `remove( matcher )`, `refresh()`, `sync( splide )`, `destroy( completely? )`, `mount()` | As in the APIs reference. In React, render slides as children instead of calling `add()`/`remove()`. |

Core components and their public members (from `components/*/*.ts`):

| Component | Public members |
|---|---|
| `Media` | `set( options, base?, notify? )`. `reduce( enable )` is internal. |
| `Direction` | `resolve( prop, axisOnly?, direction? )` maps `'width'`/`'left'`/`'right'`/`'X'`/`'Y'`/`'ArrowLeft'`/`'ArrowRight'` to the current direction, and `orient( value )` returns `-value` for ltr/ttb and `value` for rtl. |
| `Elements` | `root`, `track`, `list`, `slides` (original slide elements, no clones), `arrows?`, `pagination?`, `prev?`, `next?`, `bar?`, `toggle?`. The optional ones are only elements present in your markup (for example from `hasTrack={ false }` custom structure). Generated arrows live in `Arrows.arrows`. |
| `Slides` | `get( excludeClones? )`, `getIn( page )`, `getAt( index )`, `forEach( fn( Slide, i, all ), excludeClones? )`, `filter( index \| index[] \| selector \| predicate )`, `style( prop, value, useContainer? )`, `getLength( excludeClones? )`, `isEnough()`, `add( slides, index? )`, `remove( matcher )`, `update()`, `register( el, index, slideIndex )` |
| Slide (an item from `Slides.get()`) | `index`, `slideIndex` (the original's index for a clone, `-1` otherwise), `slide` (element), `container`, `isClone`, `update()`, `style( prop, value, useContainer? )`, `isWithin( from, distance )` |
| `Layout` | `listSize()`, `slideSize( index, withoutGap? )`, `sliderSize( withoutGap? )`, `totalSize( index?, withoutGap? )`, `getPadding( right )`, `isOverflow()`, `resize( force? )` |
| `Clones` | none |
| `Move` | `move( dest, index, prev, callback? )`, `jump( index )`, `translate( position, preventLoop? )`, `shift( position, backwards )`, `cancel()`, `toIndex( position )`, `toPosition( index, trimming? )`, `getPosition()`, `getLimit( max )`, `exceededLimit( max?, position? )`. `reposition()` is internal. |
| `Controller` | `go( control, allowSameIndex?, callback? )`, `scroll( destination, duration?, snap?, callback? )`, `getNext( destination? )`, `getPrev( destination? )`, `getAdjacent( prev, destination? )`, `getEnd()`, `setIndex( index )`, `getIndex( prev? )`, `toIndex( page )`, `toPage( index )`, `toDest( position )`, `hasFocus()`, `isBusy()` |
| `Arrows` | `arrows` (`{ prev?, next? }`), `update()` |
| `Autoplay` | `play()`, `pause()`, `isPaused()` |
| `Cover` | none |
| `Scroll` | `scroll( position, duration?, snap?, callback? )`, `cancel()` |
| `Drag` | `disable( disabled )`, `isDragging()` |
| `Keyboard` | `disable( disabled )` |
| `LazyLoad` | `check()` |
| `Pagination` | `items` (`[ { li, button, page } ]`), `getAt( index )`, `update()` |
| `Sync` | `remount()` |
| `Wheel` | none |
| `Live` | `disable( disabled )` |
| `Transition` | `start( index, done )`, `cancel()`. See `references/custom-transitions.md`. |

**Exposing your own API.** Return extra members and they become reachable from outside, for example `ref.current?.splide?.Components.SlideCounter`. Other extensions can call them too, but only from `mount()` or later, or when they are listed after yours. In TypeScript the `Components` type has an index signature of `BaseComponent | undefined`, so describe your API with an interface that extends `BaseComponent`, give the constructor that return type, and cast when you read it:

```ts
interface SlideCounterComponent extends BaseComponent {
  get(): string;
}

const counter = splide.Components.SlideCounter as SlideCounterComponent;
```

## Options for your extension

**How options reach it.** Options you don't recognise are kept, so add your own key to the `options` prop. The core `Options` type has an index signature (`[ key: string ]: any`), so custom keys type-check but are typed `any`:

```tsx
<Splide
  extensions={ extensions }
  options={ {
    slideCounter: { pages: true },                                // your extension's option
    breakpoints : {
      640: { slideCounter: { format: '{current}/{total}' } },     // deep-merged over the base value
    },
  } }
>
```

The core constructor merges, in this order: `{ label, labelledby }` from the root attributes, `DEFAULTS`, `Splide.defaults`, the constructor options (the `options` prop), then `data-splide`. `Splide.defaults` on `import { Splide } from '@splidejs/splide'` has no effect on `<Splide>`, because React Splide uses its bundled copy of core (tested). Share an options object instead.

The object your extension receives is built as `Object.create( mergedOptions )`:

- The **prototype** holds the base options.
- **Own properties** hold the overrides from the currently matching breakpoints, plus `reducedMotion` when the media query matches. When the matching breakpoints change, `Media` deletes all own properties and merges the matching breakpoint options in again (deep-merged over the base value), then emits `updated`.
- `splide.options = { ... }` merges into both the own properties and the prototype, then emits `updated` with the options.
- Splide's merge copies every non-null, non-array object recursively into a new plain object (own enumerable keys only), copies arrays with `slice()`, and replaces any other value, functions included.

How the `options` prop feeds this (`componentDidUpdate`): when the new prop is not deep-equal to the previous one, React Splide runs `splide.options = options`. Equal content does nothing, so an inline object literal is fine. Functions inside options are compared by identity, so an inline arrow causes an update (and an `updated` event) on every render.

Rules that follow from this:

- **Don't put DOM elements, `Date`s or class instances in options.** The merge turns them into plain-object copies (tested: `myEl: document.body` arrived as a different, plain object). That includes `ref.current`. Pass a selector or id, or a function that returns the element, and keep that function stable.
- **Read options when you need them, not once in the constructor.** `options.slideCounter` read inside `render()` reflects the breakpoint that applies now and the latest prop. A value cached in the constructor goes stale. Re-render on `updated`.
- **Removing a key from the prop doesn't remove it.** The setter only merges. Tested: going from `{ myExt: { a: 1 } }` to `{}` emitted `updated` with `options.myExt` still `{ a: 1 }`. Turn a feature off with an explicit value such as `myExt: false`, as [Example 2](#example-2-slide-counter-with-options-rendered-by-react) does.
- Breakpoint options merge over the base, so `{ slideCounter: { format } }` in a breakpoint keeps the base `pages` value.
- **Merge your defaults yourself.** The core ESM build doesn't export `merge`. The package exports one at `@splidejs/splide/src/js/utils`, but its types point at the TypeScript sources, so `tsc --strict` type-checks core's own source files and fails on them (tested). Use a spread or the helper below.

Defaults via a shallow merge, which is enough for flat option objects:

```ts
const SLIDE_COUNTER_DEFAULTS = { format: '{current} / {total}', pages: false };

function config( options: Options ) {
  // `true`, `false`, `undefined` and other primitives contribute no keys to a spread.
  return { ...SLIDE_COUNTER_DEFAULTS, ...options.slideCounter };
}
```

Defaults via a deep merge, for nested option objects:

```ts
type Plain = Record<string, unknown>;

function mergeOptions<T extends Plain>( defaults: T, value: unknown ): T {
  const out: Plain = { ...defaults };

  if ( value && typeof value === 'object' ) {
    Object.keys( value ).forEach( key => {
      const v = ( value as Plain )[ key ];
      const d = defaults[ key ];
      out[ key ] = v && typeof v === 'object' && ! Array.isArray( v ) && d && typeof d === 'object'
        ? mergeOptions( d as Plain, v )
        : v;
    } );
  }

  return out as T;
}
```

The "boolean or object" pattern used by official extensions (`myExt: true` or `myExt: { ... }`): treat `false` and `undefined` as off, and merge an object over the defaults. Check `options.myExt` again on `updated`, because breakpoints and the `options` prop may toggle it (as [Example 2](#example-2-slide-counter-with-options-rendered-by-react) does).

## Events

### `Splide#on` / `off` / `emit`

```ts
const MyExtension: ComponentConstructor = splide => {
  function mount() {
    splide.on( 'resize.myExtension', onResize );   // namespaced: survives splide.off( 'resize' )
  }

  function onResize() {
    // do something
  }

  return { mount };
};
```

Source behaviour (`constructors/EventBinder` and `EventInterface`):

- `on( 'move', fn )`, `on( 'move resize', fn )` (several events, separated by spaces), `on( [ 'move', 'resize' ], fn )`, and namespaces `on( 'move.myExt', fn )`.
- Handlers get the emitted arguments spread out: `( index, prev, dest )` for `move`. `this` inside a non-arrow handler is the handler function itself.
- Handlers run in registration order. Core components register in their `mount()`, before yours, so on `updated` or `refresh` the core (`Controller`, `Layout` and the others) has already re-initialised when your handler runs. React Splide registers the event props before `mount()`, so those run before every component's handlers.
- `off( events )` takes no callback. It removes handlers registered **through `splide.on`** whose event name **and namespace** match exactly. `off( 'move' )` removes only un-namespaced `move` handlers (React Splide's event-prop handlers are un-namespaced, so it removes those too), `off( 'move.myExt' )` removes only that namespace, and neither touches handlers registered through an `EventInterface`. Verified: after `splide.off( 'moved' )`, a `'moved.ns'` handler and an `EventInterface` handler both still fired. **Namespace every handler your extension registers with `splide.on`.**
- `emit( name, ...args )` dispatches a bubbling `CustomEvent` on the shared bus (a `DocumentFragment`), with the args array as `detail`. Everything on the same instance hears it: `splide.on`, every `EventInterface( splide )`, and `splide.event.on`.
- Everything registered with `splide.on` is removed by `destroy()`, **including breakpoint destroys**, and is not restored on remount. That includes React Splide's event props (see [Example 4](#example-4-listeners-that-survive-breakpoint-destroy)).

### Event props versus listeners in an extension

| You want | Use |
|---|---|
| A core event in a component, the carousel has no `destroy` breakpoints | The event prop (`onMoved`, `onActive`...). `references/events.md` lists all 28. |
| A core event that must keep firing after a breakpoint destroy and remount | A listener registered in an extension's `mount()`, calling the latest React handler through a ref ([Example 4](#example-4-listeners-that-survive-breakpoint-destroy)). |
| An event with no prop: `overflow`, extension events (`autoScroll:*`, `video:*`), your own custom events | `splide.on()` in an extension, or in a `useEffect` through `ref.current?.splide` with a namespace and cleanup. |
| A value emitted during mount (the first `overflow`, a custom event your extension emits in `mount()`) | An extension, or a public API you read after mount. Tested: a listener added in `onMounted` and one added in `useEffect` both missed an event emitted in an extension's `mount()`, because `mounted` fires after every component's `mount()`. |

Event props are captured at mount and never re-read, so they see the first render's state unless they go through a ref or use functional `setState` (`references/reactivity.md`).

### `EventInterface( splide )`

The docs' example. In React it needs a runtime import from core, which costs about 12 kB gzipped ([Bundle cost](#bundle-cost-of-runtime-imports-from-core)).

```ts
import { EventInterface } from '@splidejs/splide';

const MyExtension: ComponentConstructor = splide => {
  const { on, off, bind, unbind } = EventInterface( splide );

  function mount() {
    // Listens to an internal event
    on( 'resize', () => {} );

    // Removes the listener
    off( 'resize' );

    // Listens to a native event
    bind( window, 'resize', () => {} );

    // Removes the native listener (the docs' comment here repeats "Listens to a native event")
    unbind( window, 'resize' );
  }

  return { mount };
};
```

`EventInterface( splide? )` returns a fresh object each call, with its own listener list. Passing the instance shares the instance's bus and registers this object's `destroy` on the instance's `destroy` event. Without an argument it creates a private bus that nothing else hears.

| Member | Signature | Behaviour |
|---|---|---|
| `on` | `on( events: string \| string[], callback )` | Listens on the shared bus. It stores a wrapper, not your callback, so `off( name, callback )` cannot target it. Use a namespace instead. |
| `off` | `off( events: string \| string[] )` | Removes listeners **registered through this same object** that have the same event name and namespace. Listeners from other `EventInterface` objects or from `splide.on` are untouched, which is the "protected from unexpected removal" the docs mention. |
| `emit` | `emit( event: string, ...args )` | Same as `splide.emit`. |
| `bind` | `bind( targets: EventTarget \| EventTarget[], events: string \| string[], callback, options?: AddEventListenerOptions )` | `addEventListener` on each target, recorded for cleanup. Accepts namespaces (`'click.myExt'`) and several events separated by spaces. Null or undefined targets are skipped, so `bind( Elements.prev, ... )` is safe when there is no custom arrow. Pass a real array, not a `NodeList`: a `NodeList` is treated as a single target and throws. A target without `addEventListener` (an old-IE `MediaQueryList`) falls back to `addListener`. |
| `unbind` | `unbind( targets, events, callback? )` | Removes matching recorded listeners (target, event name and namespace; also the callback when you pass one). `bind( window, 'resize.x', fn )` needs `unbind( window, 'resize.x' )`. |
| `dispatch` | `dispatch( target, type, detail? )` | Dispatches a bubbling `CustomEvent` on any target and returns it. |
| `destroy` | `destroy()` | Removes everything this object registered, both bus and DOM listeners. It runs automatically on the instance's `destroy` event. You can call it yourself to reset, and the object stays usable afterwards. The core `Arrows` does `event.destroy(); mount();` on `updated`. |
| `bus` | `DocumentFragment` | Internal. |

`EventBinder()` is the DOM-only part: `{ bind, unbind, dispatch, destroy }` with the same semantics. It is **not** tied to any instance, so call `destroy()` yourself in your `destroy()`. The core `Media` uses it for `matchMedia` listeners that must outlive `destroy( false )`.

### The free alternative: `splide.event`

`splide.event` is the instance's own `EventInterface`, created by React Splide's bundled core, so it needs no import:

- `splide.on( 'move.myExt', fn )` for bus events.
- `splide.event.bind( target, 'click.myExt', fn )` for DOM events. Removed automatically by every destroy (tested).
- These share the instance's main listener list with user code and React Splide's event props, so namespace them. `splide.event.off( 'click' )` or `splide.off( 'move' )` elsewhere would otherwise remove them.

### Event names and arguments

| Event | Arguments | Notes |
|---|---|---|
| `mounted` | none | After all components have mounted. Prop: `onMounted`. |
| `ready` | none | After `mounted`, with the state `IDLE`. |
| `move` | `index, prev, dest` | Before the transition starts. `dest` can be a clone index in loop mode. |
| `moved` | `index, prev, dest` | When the Transition calls `done`. |
| `click` | `Slide, event` | A slide was clicked. The `onClick` prop on `<Splide>` is this event, not a DOM click. |
| `active` / `inactive` | `Slide` | |
| `visible` / `hidden` | `Slide` | |
| `refresh` | none | `splide.refresh()`, `add()` and `remove()`. React Splide calls `refresh()` when the rendered slides change. |
| `updated` | `options` | Options changed: breakpoints, the `splide.options` setter, or a changed `options` prop. |
| `resize` / `resized` | none | `resize`: window `resize`/`load` (throttled to one per animation frame), a lazy-loaded image finishing, or images in slides added with `add()` loading. `resized`: after `Layout` re-measures (only when the root size changed, or forced on init, `updated` and `refresh`). |
| `drag` / `dragging` / `dragged` | none | |
| `scroll` / `scrolled` | none | Scroll component: `drag: 'free'`, `useScroll`, `Controller.scroll()`. |
| `overflow` | `overflow` (boolean) | No event prop. First emitted during mount. |
| `destroy` | none | During `destroy()`, after the components' `destroy()`. |
| `arrows:mounted` | `prev, next` | |
| `arrows:updated` | `prev, next, prevIndex, nextIndex` | |
| `pagination:mounted` | `data, item` | `data = { list, items }` |
| `pagination:updated` | `data, prev, curr` | |
| `navigation:mounted` | `splides` | |
| `autoplay:play` / `autoplay:pause` | none | |
| `autoplay:playing` | `rate` (0 to 1) | |
| `lazyload:loaded` | `img, Slide` | Prop: `onLazyLoadLoaded`. |
| `sk`, `sh`, `ei` | `sk`: `Slide, event` | Internal: slide keydown, shifted (loop jump), end index changed. |

Event props receive the core instance first, then these arguments: `onMoved={ ( splide, index, prev, dest ) => {} }`.

**Custom events.** Emit your own events, namespaced by convention: `splide.emit( 'slidecounter:updated', current, total )`. Listen with `splide.on( 'slidecounter:updated', ( current, total ) => {} )`. There is no event prop for them. Avoid names that clash with core events.

## Core helpers with React Splide's bundled core

`EventInterface`, `EventBinder`, `RequestInterval`, `State` and `Throttle` are exported by `@splidejs/splide`, not by React Splide. They do work with the instance React Splide creates. Tested in production and under StrictMode with an extension that used all five:

- `EventInterface( splide )`: `on( 'move' )` fired, `bind( document.body, 'click' )` fired, and both were removed on unmount. Under StrictMode only the live instance's listeners fired. It works because `EventInterface` only touches `splide.event` (the bus and the `destroy` event), not the class.
- `EventBinder`, `RequestInterval`, `State`, `Throttle`: standalone, so they work anywhere. A `RequestInterval` cancelled in `destroy()` stopped ticking after unmount, and `EventBinder.destroy()` removed its listener.
- What doesn't carry over: `splide instanceof Splide` (the imported class) is `false`, and `Splide.defaults` on the imported class is ignored. `Splide.STATES` has the same values.

The cost is a second copy of core in the bundle, about 12 kB gzipped ([next section](#bundle-cost-of-runtime-imports-from-core)). Each has a no-import replacement:

| Helper | Without the import |
|---|---|
| `EventInterface( splide )` | `splide.on( 'event.ns', fn )` and `splide.event.bind( el, 'click.ns', fn )`, both removed on destroy |
| `EventBinder()` | `addEventListener`, removed in your `destroy()` |
| `RequestInterval` | `setTimeout`/`setInterval` or a `requestAnimationFrame` loop, cleared in `destroy()` |
| `Throttle` | a `requestAnimationFrame` flag |
| `State` | a variable |
| `EVENT_*`, `CLASS_*`, `SLIDE`/`LOOP`/`FADE` | the strings: `'move'`, `'is-active'`, `'slide'` |
| `Splide.STATES` | the numbers, with a comment: `splide.state.is( 7 ) // DESTROYED` |

The helpers, for when you do import them:

### `RequestInterval( interval, onInterval, onUpdate?, limit? )`

A `setInterval` driven by `requestAnimationFrame` (used by `Autoplay`, `Scroll` and `Live`).

| Arg | Description |
|---|---|
| `interval` | Duration in ms. `0` makes it fire on every frame. |
| `onInterval()` | Called each time the interval elapses. |
| `onUpdate( rate )` | Optional, called every frame with progress from 0 to 1. |
| `limit` | Optional, the number of intervals after which it pauses itself. |

| Method | Behaviour |
|---|---|
| `start( resume? )` | Without `resume`, it calls `cancel()` first and starts from 0. With `resume = true`, it continues from the paused progress. It is paused (not running) until you call `start()`. |
| `pause()` | Sets paused. The progress (rate) is kept for `start( true )`. |
| `rewind()` | Resets the start time and rate to 0 and calls `onUpdate( 0 )`. It keeps running if it was running. |
| `cancel()` | Cancels the frame, resets the rate to 0, and sets paused. |
| `set( interval )` | Changes the duration. It takes effect on the next frame. |
| `isPaused()` | `true` before the first `start()`, after `pause()` or `cancel()`, and after the limit is reached. |

Quirk: the `limit` counter is never reset. With `limit: 2`, it fired twice and paused, and a later `start()` fired only once more before pausing (verified).

### `Throttle( func, duration? )`

`Throttle( func, duration = 0 )` creates a `RequestInterval( duration, func, null, 1 )` and returns a function that starts it when it isn't running. Calls made while it is pending are ignored, and `func` runs once after `duration` ms, on the next frame when `duration` is 0. It is **trailing-only**, and **`func` receives no arguments**: the throttled function's arguments are dropped (verified). `Layout` uses `Throttle( () => emit( 'resize' ) )` on window `resize`.

### `State( initialState )`

Returns `{ set( value ), is( value | value[] ) }`. This is what `splide.state` is.

### Constants

- `EVENT_*`: `EVENT_MOUNTED 'mounted'`, `EVENT_READY 'ready'`, `EVENT_MOVE 'move'`, `EVENT_MOVED 'moved'`, `EVENT_CLICK 'click'`, `EVENT_ACTIVE 'active'`, `EVENT_INACTIVE 'inactive'`, `EVENT_VISIBLE 'visible'`, `EVENT_HIDDEN 'hidden'`, `EVENT_REFRESH 'refresh'`, `EVENT_UPDATED 'updated'`, `EVENT_RESIZE 'resize'`, `EVENT_RESIZED 'resized'`, `EVENT_DRAG 'drag'`, `EVENT_DRAGGING 'dragging'`, `EVENT_DRAGGED 'dragged'`, `EVENT_SCROLL 'scroll'`, `EVENT_SCROLLED 'scrolled'`, `EVENT_OVERFLOW 'overflow'`, `EVENT_DESTROY 'destroy'`, `EVENT_ARROWS_MOUNTED 'arrows:mounted'`, `EVENT_ARROWS_UPDATED 'arrows:updated'`, `EVENT_PAGINATION_MOUNTED 'pagination:mounted'`, `EVENT_PAGINATION_UPDATED 'pagination:updated'`, `EVENT_NAVIGATION_MOUNTED 'navigation:mounted'`, `EVENT_AUTOPLAY_PLAY 'autoplay:play'`, `EVENT_AUTOPLAY_PLAYING 'autoplay:playing'`, `EVENT_AUTOPLAY_PAUSE 'autoplay:pause'`, `EVENT_LAZYLOAD_LOADED 'lazyload:loaded'`, `EVENT_SLIDE_KEYDOWN 'sk'`, `EVENT_SHIFTED 'sh'`, `EVENT_END_INDEX_CHANGED 'ei'`.
- `CLASS_*` (for example `CLASS_ACTIVE 'is-active'`, `CLASS_SLIDE 'splide__slide'`, `CLASS_TRACK 'splide__track'`, `CLASS_LIST 'splide__list'`, `CLASS_INITIALIZED 'is-initialized'`), plus `CLASSES` (the default `classes` option) and `STATUS_CLASSES`.
- `DEFAULTS` (the built-in default options), `SLIDE`, `LOOP`, `FADE`, `LTR`, `RTL`, `TTB`.
- The states are not exported as named constants. Use `Splide.STATES` or the numbers.

Importing even one constant costs the full second copy of core. Use the strings.

The same extension written both ways. With the helpers:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import { EventInterface, RequestInterval } from '@splidejs/splide';   // runtime import: adds core (see below)
import '@splidejs/react-splide/css';

// Goes back to the first slide after `idleRewind` ms without a move or a pointer/key press.
const IdleRewind: ComponentConstructor = ( splide, Components, options ) => {
  const { on, bind } = EventInterface( splide );   // everything it registers is removed on destroy
  const timer = RequestInterval( options.idleRewind || 0, () => {
    timer.pause();
    splide.go( 0 );
  } );

  function restart() {
    if ( options.idleRewind && splide.index !== 0 ) {
      timer.set( options.idleRewind );
      timer.start();                               // start() without `true` restarts from 0
    } else {
      timer.cancel();
    }
  }

  return {
    mount() {
      on( 'moved updated', restart );
      bind( splide.root, 'pointerdown keydown', restart );
    },
    destroy() {
      timer.cancel();                              // timers are not cleaned up for you
    },
  };
};

const extensions = { IdleRewind };

export default function Example() {
  return (
    <Splide extensions={ extensions } options={ { idleRewind: 1000, label: 'Idle rewind' } }>
      { [ 1, 2, 3, 4 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
    </Splide>
  );
}
```

Without runtime imports (same behaviour, same test):

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';   // types only: no bundle cost
import '@splidejs/react-splide/css';

// The same extension without runtime imports from '@splidejs/splide'.
const IdleRewind: ComponentConstructor = ( splide, Components, options ) => {
  let timer = 0;

  function restart() {
    clearTimeout( timer );

    if ( options.idleRewind && splide.index !== 0 ) {
      timer = window.setTimeout( () => splide.go( 0 ), options.idleRewind );
    }
  }

  return {
    mount() {
      // splide.on() and splide.event.bind() are removed on destroy. Namespace them,
      // because they share the instance's listener list with user code.
      splide.on( 'moved.idleRewind updated.idleRewind', restart );
      splide.event.bind( splide.root, 'pointerdown.idleRewind keydown.idleRewind', restart );
    },
    destroy() {
      clearTimeout( timer );
    },
  };
};

const extensions = { IdleRewind };

export default function Example() {
  return (
    <Splide extensions={ extensions } options={ { idleRewind: 1000, label: 'Idle rewind' } }>
      { [ 1, 2, 3, 4 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
    </Splide>
  );
}
```

Verified for both, in production and development: after two moves (index 2), a `pointerdown` 500ms later restarted the 1000ms timer (still at index 2 after another 700ms), and the carousel went back to index 0 once 1000ms passed without activity.

## Bundle cost of runtime imports from core

React Splide's dist inlines core, so any runtime import from `@splidejs/splide` adds a second copy. Measured with Vite 8.3.8 production builds of the same app (React 19, React Splide, one tiny extension), total JS:

| What the app imports from `@splidejs/splide` | JS (minified) | gzip | Added (minified / gzip) |
|---|---|---|---|
| Nothing (extension uses `splide.on`) | 248.24 kB | 79.73 kB | none |
| `import type { ComponentConstructor }` only | 248.24 kB | 79.73 kB | none |
| `EVENT_MOVE` (one string constant) | 278.83 kB | 91.72 kB | +30.59 kB / +11.99 kB |
| `EventInterface` | 278.85 kB | 91.73 kB | +30.61 kB / +12.00 kB |
| `RequestInterval` | 278.88 kB | 91.74 kB | +30.64 kB / +12.01 kB |
| All five helpers | 278.96 kB | 91.78 kB | +30.73 kB / +12.05 kB |
| The `Splide` class (for `Splide.STATES`) | 278.86 kB | 91.73 kB | +30.62 kB / +12.00 kB |

The bundler keeps the whole core file for any value import, even a string constant: about 30.6 kB minified, 12 kB gzipped, roughly the size of `splide.min.js`. Type-only imports are free. Official extension packages such as `@splidejs/splide-extension-auto-scroll` are a separate matter: check their own size.

## Getting extension state into React

An extension runs outside React's render cycle. Three ways to get its state into components, all tested:

| Pattern | Use it when | Watch out for |
|---|---|---|
| **Callback through a ref** ([Example 2](#example-2-slide-counter-with-options-rendered-by-react)) | The default. React renders the UI, the extension only computes. | Create the extension once (`useState( () => ... )` or module scope), and point the ref at the latest callback on every render. |
| **Custom event plus a public API** (below) | Several components listen, or the extension is shared code that shouldn't know about React. | Listeners added after mount miss what `mount()` emitted, so read the current value from the public API first. Listeners registered from React with `splide.on` are removed by breakpoint destroys. |
| **Calling the extension's API** (`ref.current?.splide?.Components.X`) | React drives the extension (buttons, form fields). | `splide` is `undefined` before mount and after unmount. Don't call movement methods while a breakpoint has the slider destroyed (see Example 4). |

Calling a React state setter from an extension is safe. During `mount()` it runs inside React Splide's `componentDidMount` and schedules a re-render. On that re-render React Splide compares `options` (deep) and the slide elements (shallow), so an unchanged carousel isn't updated or refreshed.

Custom event plus a public API:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { BaseComponent, Splide as SplideCore } from '@splidejs/splide';
import '@splidejs/react-splide/css';

interface SlideCounterComponent extends BaseComponent {
  get(): string;
}

// Emits 'slidecounter:updated' and exposes get() as a public API.
function SlideCounter( splide: SplideCore ): SlideCounterComponent {
  let value = '';

  function update() {
    value = `${ splide.index + 1 } / ${ splide.length }`;
    splide.emit( 'slidecounter:updated', value );
  }

  return {
    mount() {
      update();   // emitted during mount, before any listener added after mount exists
      splide.on( 'move.slideCounter', update );
    },
    get: () => value,
  };
}

const extensions = { SlideCounter };

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ text, setText ] = useState( '' );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( ! splide ) {
      return;
    }

    // The first emit already happened, so read the current value from the public API.
    setText( ( splide.Components.SlideCounter as SlideCounterComponent ).get() );
    splide.on( 'slidecounter:updated.react', ( value: string ) => setText( value ) );

    return () => {
      splide.off( 'slidecounter:updated.react' );
    };
  }, [] );

  return (
    <div>
      <Splide ref={ ref } extensions={ extensions } options={ { label: 'Counter via events' } }>
        <SplideSlide>01</SplideSlide>
        <SplideSlide>02</SplideSlide>
        <SplideSlide>03</SplideSlide>
      </Splide>
      <p className="slide-counter">{ text }</p>
    </div>
  );
}
```

Verified in production and development: the paragraph showed `1 / 3` right after mount, then `2 / 3` after the next arrow. Under StrictMode the effect runs again with the second core instance (the first instance's listeners went with its destroy).

## Cleanup

Removed automatically on any destroy (`completely` true or false), including unmount:

- Handlers from `splide.on()` and `splide.event.bind()`.
- Everything registered through any `EventInterface( splide )`, bus and DOM listeners alike.
- Splide's own changes: the core restores slide `style` attributes, removes status classes and ARIA attributes, strips `style` from the track and list, and removes the arrows and pagination it generated.

You must undo yourself, in `destroy()`:

- DOM nodes you created. The docs' SlideNumber doesn't do this: under StrictMode the page showed two counters (tested). In production the same happens on any breakpoint remount.
- Classes and inline styles you put on elements Splide doesn't reset (the root, or outside elements).
- Timers: `setTimeout`, `setInterval`, `requestAnimationFrame`, and `RequestInterval` (call `cancel()`).
- Observers (`IntersectionObserver`, `ResizeObserver`, `MutationObserver`), and `EventBinder()` objects.
- Every `addEventListener` you made.
- React state you set from the extension, if it shouldn't outlive the carousel: report `null` from `destroy()`, as Example 2 does.

Make `destroy()` idempotent and safe after a partial mount: check for `null` and reset references.

## Registering extensions

Pass an object of constructors to the `extensions` prop. The key becomes the component name, `Components.<key>`.

```tsx
import { MyExtension01 } from './my-extension-01';
import { MyExtension02 } from './my-extension-02';

const extensions = { MyExtension01, MyExtension02 };

export function Carousel() {
  return <Splide extensions={ extensions }>...</Splide>;
}
```

- **Several extensions:** list them all in one object. They are constructed, set up and mounted in the order listed, after all core components, and destroyed in reverse.
- **With a custom transition:** the `transition` prop (`references/custom-transitions.md`). A `Transition` key inside `extensions` is ignored, because core assigns `{ Transition: this._T }` last (verified).
- **Names:** must be unique within the object. A core name **replaces** that core component (next section). Your own names should not collide with core names unless you mean that.
- **Read once:** the prop is read at mount. Breakpoint remounts reuse the same extensions. A new set needs a React remount (`key`).
- **Several carousels:** pass the same constructors to each `<Splide>`. Each instance calls them separately. Per-instance state belongs in a factory created per component (`useState( () => createX() )`), not at module scope.
- **Official extensions** go in the same object: `extensions={ { AutoScroll, MyExtension } }`.

## Overriding a built-in component

The guide says "do not use names of core components". The source allows it, and it is the way to replace a core component: `assign( {}, CoreComponents, Extensions, { Transition } )` overwrites the core constructor with yours, and **your constructor keeps the core component's slot** in the construct, setup, mount and destroy order. With `extensions={ { Keyboard } }`, `Keyboard` was still the 14th of 20 keys (tested). Components that come later and capture it at construction time (`const { Move } = Components`) get yours.

Constraints:

- It is a full replacement. The core constructors are not exported by core or by React Splide, so you can't wrap or extend the original.
- Implement the API other code relies on:

| Replaced component | Used by |
|---|---|
| `Media` | `Splide` (`options` setter calls `set`, which React Splide uses for prop changes), `Sync` (`set`), `Drag` (`reduce`) |
| `Direction` | `Clones`, `Drag`, `Keyboard`, `Layout`, `Move`, `Pagination`, `Slide` (`resolve`, `orient`) |
| `Elements` | nearly every component (`root`, `track`, `list`, `slides`, ...), and React Splide (`Elements.list.children` to detect slide changes) |
| `Slides` | `Splide` (`add`, `remove`, `getLength`), `Autoplay`, `Clones`, `Controller`, `Cover`, `Layout`, `LazyLoad`, `Move`, `Pagination`, `Fade` |
| `Layout` | `Clones`, `Drag`, `Move`, `Scroll` |
| `Move` | `Controller`, `Scroll`, `Drag`, `Slide` transition |
| `Controller` | `Splide` (`go`, `index`), `Arrows`, `Move`, `Pagination`, `Scroll`, `Slides`, `Wheel`, `Drag`, `Slide` transition |
| `Scroll` | `Controller`, `Move`, `Drag`, `Slide` transition |
| `Autoplay` | `Live` (`isPaused`) |
| `Sync` | `Splide#sync()` (`remount`), which `ref.current.sync()` calls |
| `Arrows`, `Clones`, `Cover`, `Drag`, `Keyboard`, `LazyLoad`, `Pagination`, `Wheel`, `Live` | No other core component calls them. Keep their public methods (for example `Keyboard.disable`) if your code does. |

- Replacing the leaf components in the last row is low-risk. Replacing anything above needs the full interface.
- To turn a core feature off, use its option (`keyboard: false`, `wheel: false`, `arrows: false`, `drag: false`). Replacing it with `() => ( {} )` also works but is rarely needed.

Example: replace `Keyboard` with one that also handles Home and End, keeping `disable()` and the `keyboard` option:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Components, KeyboardComponent, Options, Splide as SplideCore } from '@splidejs/splide';
import '@splidejs/react-splide/css';

// Same name as the core component, so it takes the core Keyboard's slot.
function Keyboard( splide: SplideCore, Components: Components, options: Options ): KeyboardComponent {
  const { resolve } = Components.Direction;
  let target: Window | HTMLElement | null = null;
  let disabled = false;

  function mount() {
    init();
    splide.on( 'updated.keyboard', () => {
      destroy();
      init();
    } );
  }

  function init() {
    const { keyboard } = options;

    if ( keyboard ) {
      // Same targets as the core: 'global' listens on window, any other truthy value on the root.
      target = keyboard === 'global' ? window : splide.root;
      target.addEventListener( 'keydown', onKeydown as EventListener );
    }
  }

  function destroy() {
    if ( target ) {
      target.removeEventListener( 'keydown', onKeydown as EventListener );
      target = null;
    }
  }

  function disable( value: boolean ) {
    disabled = value;
  }

  function onKeydown( e: KeyboardEvent ) {
    if ( disabled ) {
      return;
    }

    if ( e.key === resolve( 'ArrowLeft' ) ) {          // ArrowRight in rtl, ArrowUp in ttb
      splide.go( '<' );
    } else if ( e.key === resolve( 'ArrowRight' ) ) {  // ArrowLeft in rtl, ArrowDown in ttb
      splide.go( '>' );
    } else if ( e.key === 'Home' ) {
      splide.go( 0 );
    } else if ( e.key === 'End' ) {
      splide.go( Components.Controller.getEnd() );
    } else {
      return;
    }

    e.preventDefault();
  }

  return { mount, destroy, disable };
}

const extensions = { Keyboard };

export default function Example() {
  return (
    <Splide extensions={ extensions } options={ { keyboard: true, label: 'Keyboard override' } }>
      { Array.from( { length: 9 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </Splide>
  );
}
```

Verified in production and development: `Keyboard` kept slot 14 of 20. With 9 slides, End went to index 8, Home to 0 and ArrowRight to 1. `disable( true )` blocked keys, and `keyboard: 'global'` set at runtime moved on key presses on `document.body`.

## Example 1: SlideNumber (the docs example)

The docs build it in two steps: first create the element, then add `update()` with a `move` listener. Below is the final version, with the `destroy()` the docs leave out. In React it is required, because StrictMode remounts in development.

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

const SlideNumber: ComponentConstructor = ( splide, Components ) => {
  // Safe in the constructor: Elements collected track/list in its setup().
  const { track } = Components.Elements;
  let elm: HTMLDivElement | undefined;

  function mount() {
    elm = document.createElement( 'div' );
    elm.style.textAlign = 'center';
    elm.style.marginTop = '0.5em';

    // Insert the element after the track
    track.parentElement!.insertBefore( elm, track.nextSibling );

    update();
    splide.on( 'move.slideNumber', update );
  }

  function update() {
    elm!.textContent = `${ splide.index + 1 }/${ splide.length }`;
  }

  // Required in React: StrictMode (development) mounts, destroys and mounts again,
  // so without this the page shows two numbers.
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

`Elements` holds the main elements (`track`, `list`, `slides`). `splide.index` is the current index, already updated when `move` fires. `splide.length` is the slide count without clones. Verified: `1/3`, then `2/3` after the next arrow, and a single number in development (two without `destroy()`).

Inserting your own node into the root that React renders works: React only touches the nodes it created, and Splide adds its arrows and pagination there the same way. For UI that depends on React state, render it with React instead (Example 2).

## Example 2: Slide counter with options, rendered by React

This version adds options with defaults, slide or page counting, breakpoint-aware updates, the ability to turn it off at runtime, and cleanup. The extension only computes the text. React renders it, getting it through a ref (the callback pattern from [Getting extension state into React](#getting-extension-state-into-react)).

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

interface SlideCounterOptions {
  format?: string;    // '{current}' and '{total}' are replaced
  pages?: boolean;    // true: count pages (perPage), false: count slides
}

type Report = ( text: string | null ) => void;

const SLIDE_COUNTER_DEFAULTS: Required<SlideCounterOptions> = {
  format: '{current} / {total}',
  pages : false,
};

// A factory: the extension calls whatever function the ref holds at that moment,
// so it never sees a stale React callback.
function createSlideCounter( report: { current: Report } ): ComponentConstructor {
  return ( splide, Components, options ) => {
    const { Controller } = Components;

    // Read on every render: breakpoints and the options prop mutate `options` in place.
    function render() {
      if ( options.slideCounter === false ) {
        report.current( null );
        return;
      }

      const conf: Required<SlideCounterOptions> = { ...SLIDE_COUNTER_DEFAULTS, ...options.slideCounter };
      const index = splide.index;
      let current = index + 1;
      let total   = splide.length;

      if ( conf.pages ) {
        // Same page maths as the Pagination component.
        current = Controller.toPage( index ) + 1;
        total   = Controller.hasFocus()
          ? Controller.getEnd() + 1
          : Math.ceil( splide.length / options.perPage! );
      }

      report.current( conf.format.replace( '{current}', `${ current }` ).replace( '{total}', `${ total }` ) );
    }

    return {
      mount() {
        render();
        // Core components re-initialise on these events before this handler runs.
        splide.on( 'move.slideCounter refresh.slideCounter updated.slideCounter', render );
      },
      destroy() {
        report.current( null );   // also runs on breakpoint destroys
      },
    };
  };
}

export default function Example() {
  const [ counter, setCounter ] = useState<string | null>( null );
  const [ enabled, setEnabled ] = useState( true );

  // Latest-value ref: any callback works here, including one that changes every render.
  const report = useRef<Report>( setCounter );
  report.current = setCounter;

  // Created once per component instance. Extensions are only read at mount.
  const [ extensions ] = useState( () => ( { SlideCounter: createSlideCounter( report ) } ) );

  const options: Options = {
    perPage     : 3,
    label       : 'Products',
    slideCounter: enabled ? { pages: true } : false,
    breakpoints : {
      700: { perPage: 2, slideCounter: { format: '{current} of {total}' } },
    },
  };

  return (
    <div>
      <Splide extensions={ extensions } options={ options }>
        { Array.from( { length: 10 }, ( _, i ) => (
          <SplideSlide key={ i }>Product { i + 1 }</SplideSlide>
        ) ) }
      </Splide>
      { counter !== null && <p className="slide-counter">{ counter }</p> }
      <button type="button" onClick={ () => setEnabled( value => ! value ) }>Toggle counter</button>
    </div>
  );
}
```

Verified in production and development, with 10 slides: `1 / 4`, then `2 / 4` after the next arrow. At 650px wide (the 700px breakpoint, `perPage: 2`) it showed `2 of 5`, and `2 / 4` again at 1280px. The toggle removed the paragraph (`slideCounter: false` reached the extension through the `options` prop and `updated`) and brought it back.

Notes:

- `slideCounter: enabled ? { pages: true } : false` uses `false` on purpose: leaving the key out would keep the old value.
- `report.current( null )` in `destroy()` hides the counter while a `destroy: true` breakpoint has the slider destroyed. The remount reports again.
- Splide's `live` option (on by default) already announces slide changes to screen readers, so the counter has no `aria-live` of its own.

## Example 3: Keyboard shortcuts

These shortcuts add to the core arrow keys (the `keyboard` option): Home and End go to the first slide and the last reachable index, PageUp and PageDown go to the previous and next page, `1` to `9` jump to that page, and `p` toggles autoplay. The extension uses its own option, `shortcuts: true` (listens on the root, so focus must be inside the slider) or `'global'` (listens on `window`). It ignores key presses while the user types in form fields or with modifier keys held, and rebinds on `updated` so breakpoints and the `options` prop can change the mode.

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

const KeyboardShortcuts: ComponentConstructor = ( splide, Components, options ) => {
  const { Controller, Autoplay } = Components;
  let target: Window | HTMLElement | null = null;

  function mount() {
    listen();
    splide.on( 'updated.shortcuts', () => {
      unlisten();
      listen();
    } );
  }

  function listen() {
    const mode = options.shortcuts;

    if ( mode ) {
      target = mode === 'global' ? window : splide.root;
      target.addEventListener( 'keydown', onKeydown as EventListener );
    }
  }

  function unlisten() {
    if ( target ) {
      target.removeEventListener( 'keydown', onKeydown as EventListener );
      target = null;
    }
  }

  function isTyping( el: EventTarget | null ) {
    return el instanceof Element
      && !! el.closest( 'input, textarea, select, [contenteditable]:not([contenteditable="false"])' );
  }

  function onKeydown( e: KeyboardEvent ) {
    if ( e.defaultPrevented || e.altKey || e.ctrlKey || e.metaKey || isTyping( e.target ) ) {
      return;
    }

    const { key } = e;
    let handled = true;

    if ( key === 'Home' ) {
      splide.go( 0 );
    } else if ( key === 'End' ) {
      splide.go( Controller.getEnd() );
    } else if ( key === 'PageUp' ) {
      splide.go( '<' );
    } else if ( key === 'PageDown' ) {
      splide.go( '>' );
    } else if ( /^[1-9]$/.test( key ) ) {
      splide.go( `>${ +key - 1 }` );                  // '>n' goes to page n (0-based)
    } else if ( ( key === 'p' || key === 'P' ) && options.autoplay ) {
      Autoplay.isPaused() ? Autoplay.play() : Autoplay.pause();
    } else {
      handled = false;
    }

    if ( handled ) {
      e.preventDefault();                              // stop Home/End/PageUp/PageDown scrolling the page
    }
  }

  return { mount, destroy: unlisten };
};

const extensions = { KeyboardShortcuts };

export default function Example() {
  return (
    <Splide
      extensions={ extensions }
      options={ { perPage: 2, autoplay: true, shortcuts: true, label: 'Shortcuts' } }
    >
      { Array.from( { length: 9 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </Splide>
  );
}
```

Verified in production and development with 9 slides and `perPage: 2` (indices are 0-based): PageDown went to 2, `3` to 4, End to 7, PageUp to 6, Home to 0, `p` paused autoplay, and Ctrl+End did nothing.

Notes:

- With `shortcuts: true`, key presses reach the root only from focused elements inside it (slides with `slideFocus`, arrows, links). Pagination buttons handle ArrowLeft, ArrowRight, Home and End themselves and call `stopPropagation()`, so those keys never reach this handler from a pagination button.
- `Autoplay.play()` only makes sense when the `autoplay` option is on, which is why the handler checks it.
- For RTL or vertical sliders, map arrow keys with `Components.Direction.resolve( 'ArrowLeft' )` or `resolve( 'ArrowRight' )`, as the core `Keyboard` does.
- React's own `onKeyDown` isn't typed on `<Splide>` (its props omit DOM handlers), and an extension also covers `'global'` mode, which is why this is an extension.

## Example 4: Listeners that survive breakpoint destroy

**The problem.** With `breakpoints: { 700: { destroy: true } }`, narrowing the window calls `splide.destroy( false )`, which removes every handler registered with `splide.on()`. Widening it again calls `splide.mount()`. That remount re-runs every extension but does **not** restore handlers registered outside them, and React Splide's event props are registered outside them, once, in `componentDidMount`. Tested: `onMoved` fired for the moves before the first breakpoint destroy and never again, while a `moved` listener registered in an extension kept firing. React doesn't re-render on a breakpoint remount, so nothing re-binds them. If the page loads inside the destroying breakpoint, the destroy is postponed until `ready`, so the event props are dropped right away. And the index restarts from the `start` option after the remount.

**The fix.** Register everything inside an extension. Its `mount()` runs on every (re)mount and its `destroy()` on every destroy. A factory function keeps data across remounts, and a ref gives the handlers the latest React state.

**Don't call `go()` on a destroyed slider.** Verified with core: `splide.go( 2 )` while the `destroy: true` breakpoint matched set the index to 2, put `transform: translateX(0px)` on the list, and set the state from `DESTROYED` back to `IDLE` (the transition's `done` ran). On widening, `Media` saw no destroyed state and never remounted: no `is-active`, no arrows. `ref.current.go()` calls the same method, so external React controls need the guard below.

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { ComponentConstructor } from '@splidejs/splide';
import '@splidejs/react-splide/css';

type Handlers = Record<string, ( ...args: any[] ) => void>;   // keys: 'moved', 'move resize', 'ready.app'...

const DESTROYED = 7;   // Splide.STATES.DESTROYED, written out to avoid a runtime import

// 1) Bus handlers registered on every (re)mount. They call the ref's current handlers.
function persistentListeners( handlers: { current: Handlers } ): ComponentConstructor {
  return splide => ( {
    mount() {
      Object.keys( handlers.current ).forEach( events => {
        splide.on( events, ( ...args: any[] ) => handlers.current[ events ]( ...args ) );
      } );
    },
  } );
}

// 2) Restores the index after a remount (otherwise it restarts at `start`).
function rememberIndex(): ComponentConstructor {
  let saved: number | null = null;   // lives outside the constructor

  return ( splide, Components ) => ( {
    setup() {
      // setup() runs before any mount(), so Controller, Pagination and Move all mount with this index.
      if ( saved !== null ) {
        Components.Controller.setIndex( saved );
      }
    },
    destroy() {
      saved = splide.index;   // components still exist during destroy()
    },
  } );
}

export default function Example() {
  const ref = useRef<Splide>( null );
  const [ moves, setMoves ] = useState<number[]>( [] );

  // Latest handlers, read at call time: no stale state.
  const handlers = useRef<Handlers>( {} );
  handlers.current = {
    moved: ( index: number ) => setMoves( list => [ ...list, index ] ),
  };

  const [ extensions ] = useState( () => ( {
    PersistentListeners: persistentListeners( handlers ),
    RememberIndex      : rememberIndex(),
  } ) );

  // 3) External controls: plain React handlers, guarded while a breakpoint has the slider destroyed.
  function go( control: string ) {
    const splide = ref.current?.splide;

    if ( splide && ! splide.state.is( DESTROYED ) ) {
      splide.go( control );
    }
  }

  return (
    <div>
      <button type="button" className="my-prev" onClick={ () => go( '<' ) }>Prev</button>
      <button type="button" className="my-next" onClick={ () => go( '>' ) }>Next</button>
      <Splide
        ref={ ref }
        extensions={ extensions }
        options={ { label: 'Gallery', breakpoints: { 700: { destroy: true } } } }
        onMoved={ ( _splide, index ) => console.log( 'onMoved prop', index ) }   // lost after the first breakpoint destroy
      >
        { Array.from( { length: 9 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
      </Splide>
      <p className="moves">Moved to: { moves.join( ', ' ) }</p>
    </div>
  );
}
```

Verified in production and development: after three clicks on Next (index 3), narrowing to 650px destroyed the slider, and Next did nothing (no transform on the list). Widening remounted it at index 3 with the right slide and pagination page active. Next then moved to 4. The paragraph read `Moved to: 1, 2, 3, 4`, while the `onMoved` prop had logged only 1, 2 and 3.

Notes:

- `useState( () => ( { ... } ) )` creates the extensions once per component instance. `rememberIndex()` inside it gives each carousel its own saved index. StrictMode's remount in development builds a new core instance but reuses these factories, which is harmless here (the saved index is 0).
- `persistentListeners` reads the handler keys at mount. Keep the set of keys fixed, and change only what the handlers do.
- The `DESTROYED` constant is `Splide.STATES.DESTROYED` written out, so the file has no runtime import from core.

Use `destroy( completely )` when an extension must behave differently on a final destroy, for example to keep UI visible while breakpoints have the slider destroyed:

```ts
function destroy( completely?: boolean ) {
  if ( completely ) {
    elm?.remove();              // unmount or final destroy: remove it
  } else if ( elm ) {
    elm.hidden = true;          // breakpoint destroy: hide it; the next mount() decides again
  }
}
```

(Each remount constructs a new closure, so the new `mount()` must find or reuse the hidden element, for example by a class lookup, instead of creating another one.)

## Pitfalls checklist

- Extensions (and the transition) are read once, at mount. Changing the prop does nothing. Remount with a `key` to switch.
- Define extensions at module scope or once per component (`useState( () => ... )`). Pass changing React values through a ref.
- Give every extension that touches the DOM a `destroy()`. StrictMode remounts in development and shows duplicates otherwise.
- Read `options.*` when you need it. Re-render on `updated`, which also fires when the `options` prop changes.
- Turn options off with `false`. Removing the key from the `options` prop keeps the old value.
- Don't pass DOM elements (including `ref.current`) or other non-plain objects in options: the merge copies them into plain objects. Keep functions in options stable.
- Read `Slides.get()`, `Arrows.arrows` and `Components.Transition` in `mount()` or later, not in the constructor.
- Namespace every `splide.on` and `splide.event.bind` (`'move.myExt'`). `splide.off( 'move' )` removes un-namespaced handlers, React Splide's event props included.
- `off()` can't remove one particular callback. Use a namespace or a separate `EventInterface` object.
- Event props and any `splide.on` made outside an extension are lost on the first breakpoint destroy. Register them in an extension.
- Listeners added in `onMounted` or `useEffect` miss events emitted during mount. Read the current value instead.
- Don't call `go()` while a breakpoint has the slider destroyed: it resets the state to `IDLE` and the slider never remounts. Guard with `splide.state.is( 7 )`.
- Remove every DOM node, timer, observer and `addEventListener` in `destroy()`. It runs on breakpoint destroys too.
- Remounts construct new closures. Keep state that must persist in a factory closure.
- Only use core component names on purpose (to replace one), and implement the API other components use.
- A `Transition` key in `extensions` is ignored. Use the `transition` prop.
- Any runtime import from `@splidejs/splide` adds a second copy of core (about 12 kB gzipped). `import type` is free. `splide.on` and `splide.event.bind` need no import.
- `Throttle` drops arguments, and `RequestInterval` starts paused until `start()`.

## Docs vs source

- **Lifecycle.** The docs say: optional `mount` and `destroy`. The source also calls an optional `setup()` right after construction (before any `mount()`), and passes `destroy( completely )` a boolean.
- **Core names.** The docs say: "Do not use names of core components". The source allows it, and a same-named extension replaces the core component in the same slot. The `Transition` key is the exception: it is always overwritten by the transition.
- **Registration example typo.** The docs' registration example uses `{ MyExtention01, MyExtension02 }`. `MyExtention01` (misspelled) would throw a `ReferenceError` as a shorthand property. Use `MyExtension01`.
- **`off`.** The docs say `EventInterface#off( 'resize' )` "Removes the listener", and the `Splide#off` JSDoc says it removes "all handlers for the specified event". In the source, `off` only removes handlers registered through the same object (`splide.on` or that particular `EventInterface`) that have the same event name and namespace. Namespaced handlers and other objects' handlers survive (verified).
- **`unbind` comment.** The docs' comment above `unbind( window, 'resize' )` repeats "Listens to a native event". It removes the listener.
- **`options` argument.** The docs call it "The latest options". In the source it is the live, prototype-backed options object, mutated in place by breakpoints and the `options` setter (and so by the `options` prop).
- **SlideNumber cleanup.** The docs' SlideNumber has no `destroy()` and leaves its element behind. Under React's StrictMode there were two counters (tested).
- **React Splide's `extensions` prop.** The React Splide page says only "Registers extensions as an object literal." In the source the prop is read once in `componentDidMount`, later changes are ignored, and it is rendered onto the root element as `extensions="[object Object]"`.
