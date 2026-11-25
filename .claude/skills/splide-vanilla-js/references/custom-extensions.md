# Custom Extensions

Source: https://splidejs.com/guides/extension/

Checked against the Splide 4.1.3 source (`src/js`) and the 4.1.4 dist build, which match for everything here. Every example on this page was run in Chrome against the 4.1.4 jsDelivr files, in both the ESM form and the classic-script form.

## Contents

- [Loading: ESM module or classic script](#loading-esm-module-or-classic-script)
- [The component shape](#the-component-shape)
- [Lifecycle (source-verified)](#lifecycle-source-verified)
- [Accessing the instance and other components](#accessing-the-instance-and-other-components)
- [Options for your extension](#options-for-your-extension)
- [Events](#events)
- [Cleanup](#cleanup)
- [Helpers for extension authors](#helpers-for-extension-authors)
- [Registering extensions](#registering-extensions)
- [Overriding a built-in component](#overriding-a-built-in-component)
- [Example 1: SlideNumber (the docs example)](#example-1-slidenumber-the-docs-example)
- [Example 2: Slide counter with options](#example-2-slide-counter-with-options)
- [Example 3: Keyboard shortcuts](#example-3-keyboard-shortcuts)
- [Example 4: Listeners that survive breakpoint destroy](#example-4-listeners-that-survive-breakpoint-destroy)
- [Pitfalls checklist](#pitfalls-checklist)
- [Docs vs source](#docs-vs-source)

## Loading: ESM module or classic script

The docs write `import Splide from '@splidejs/splide'` and `import { EventInterface } from '@splidejs/splide'`. That needs a bundler. Without one, there are two options.

| | (a) Native ESM | (b) Classic script (UMD) |
|---|---|---|
| Tag | `<script type="module">` | `<script src=".../splide.min.js">` then a normal `<script>` |
| URL | `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js` | `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js` |
| What you get | `Splide` (default and named), `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, `SplideRenderer`, `DEFAULTS`, `CLASSES`, `STATUS_CLASSES`, every `CLASS_*` and `EVENT_*` constant, `SLIDE`/`LOOP`/`FADE`, `LTR`/`RTL`/`TTB` | `window.Splide` only, plus the statics `Splide.defaults` and `Splide.STATES` |
| Events in an extension | `EventInterface(splide)`: `on`/`off`/`emit` plus `bind`/`unbind` for DOM events, all removed on destroy | `splide.on('event.namespace', fn)` (auto-removed on destroy), and `addEventListener`/`removeEventListener` that you remove yourself in `destroy()` |
| Serving | http(s), because local module files fail from `file://` (see notes) | Works from `file://` too |

**Pick ESM** when you want the helpers (`EventInterface`, `RequestInterval`, `Throttle`, constants) or when you write the extension as its own `.js` module. **Pick classic** when the page already loads `splide.min.js` (WordPress, Shopify and PHP themes, legacy pages), when it has to run from `file://`, or when other classic scripts need `window.Splide`. Both forms build the same extension. Only the imports and the event cleanup differ.

ESM, with the CSS:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">

<script type="module">
  import {
    Splide,
    EventInterface,
    EVENT_MOVE,
  } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function MyExtension( splide, Components, options ) {
    const { on } = EventInterface( splide );
    return {
      mount() {
        on( EVENT_MOVE, index => console.log( 'moving to', index ) );
      },
    };
  }

  new Splide( '.splide' ).mount( { MyExtension } );
</script>
```

ESM with an import map. This lets the docs' bare `'@splidejs/splide'` specifier run unchanged. It is supported in all current browsers.

```html
<script type="importmap">
{
  "imports": {
    "@splidejs/splide": "https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js"
  }
}
</script>
<script type="module">
  import Splide, { EventInterface } from '@splidejs/splide';
  // ...
</script>
```

Classic:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function MyExtension( splide, Components, options ) {
    return {
      mount() {
        // Namespaced so splide.off( 'move' ) elsewhere does not remove it.
        splide.on( 'move.myExtension', index => console.log( 'moving to', index ) );
      },
    };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide' ).mount( { MyExtension } );
  } );
</script>
```

Notes:

- **Serve ESM over http(s).** Browsers block module files loaded from `file://` URLs (CORS, origin `null`). Verified in Chrome: `import ... from './my-extension.js'` failed from `file://` and worked over http. An inline module that imports only the https CDN URL did run from `file://` in Chrome, but don't rely on it. Use any static server, for example `python -m http.server`. A classic script works from `file://`.
- **Module scripts are deferred.** They run after the document is parsed, so no `DOMContentLoaded` wrapper is needed. Classic inline scripts in `<head>` need one, and classic scripts at the end of `<body>` do not.
- **Module scope.** Names declared in a module are not globals. Set `window.mySplide = splide` if classic scripts need them.
- **Load one build.** The ESM `Splide` and `window.Splide` are separate copies (`Splide !== window.Splide`, verified). Loading both doubles the download. `EventInterface` from the ESM file does work on an instance made with the UMD build, because it only uses `splide.event`, but don't load the library twice just for that. Use the classic patterns instead.
- **Keep the URL identical.** If you split extensions into your own module files, import from the exact same URL (same version, same path) everywhere. Browsers dedupe modules by URL, so a different URL gives a second copy.
- "Classic" does not mean ES5. `const`, arrow functions and template literals work in classic scripts in every current browser.
- **Constants as strings for classic scripts.** `EVENT_MOVE` = `'move'`, `EVENT_MOVED` = `'moved'`, `EVENT_UPDATED` = `'updated'`, `EVENT_REFRESH` = `'refresh'`, `EVENT_DESTROY` = `'destroy'`, `CLASS_ACTIVE` = `'is-active'`, `SLIDE` = `'slide'`, `LOOP` = `'loop'`, `FADE` = `'fade'`, `LTR`/`RTL`/`TTB` = `'ltr'`/`'rtl'`/`'ttb'`. [Event names and arguments](#event-names-and-arguments) has the full event table.

## The component shape

An extension is a plain function. Splide calls it with `( Splide, Components, options )` and uses the object it returns as the component.

```js
// Type (from dist/types/index.d.ts):
// type ComponentConstructor = ( Splide: Splide, Components: Components, options: Options ) => BaseComponent;
// interface BaseComponent { setup?(): void; mount?(): void; destroy?( completely?: boolean ): void; }
```

| Argument | Type | Description |
|---|---|---|
| `Splide` | `Splide` | The Splide **instance** (not the class). |
| `Components` | `Components` | The shared object of all component objects, keyed by name (`Components.Move`, `Components.Elements`, your own extensions, `Components.Transition`). |
| `options` | `Options` | The instance's live options object. It is the same object for the whole mount and is mutated in place by breakpoints and by `splide.options = {...}` (see [Options](#options-for-your-extension)). |

What the returned object can contain:

| Member | Called by Splide | When |
|---|---|---|
| `setup()` | yes, optional | Right after this component is constructed, before any component's `mount()`. Not mentioned in the guide, but in `BaseComponent` and used by the core `Media` and `Elements`. |
| `mount()` | yes, optional | After every component (core, extensions, Transition) has been constructed and set up. Runs in registration order. |
| `destroy( completely )` | yes, optional | On `splide.destroy()`, including destroys triggered by breakpoints. `completely` is `true` for a final destroy and `false` when breakpoints may remount the slider. |
| anything else | no | Public API, reachable as `splide.Components.<Name>.<member>`. |

The docs' skeleton (both build forms run this code unchanged, minus the docs' `export` keyword when it is inline):

```js
function MyExtension( Splide, Components, options ) {
  /**
   * Optional. Called when the component is mounted.
   */
  function mount() {
    console.log( 'Hello, Splide!' );
  }

  /**
   * Optional. Called when the Splide destroys the carousel.
   */
  function destroy() {
    console.log( 'Bye!' );
  }

  return {
    mount,
    destroy,
  };
}
```

The docs name the first parameter `Splide`. Inside the function that name shadows the `Splide` class (the import, or `window.Splide`). If you need the class there, for example `Splide.STATES`, use `window.Splide.STATES` in classic scripts or rename the parameter. The examples on this page that are not copied from the docs name it `splide`.

Accessing the instance and core components (docs example):

```js
function MyExtension( Splide, Components, options ) {
  function mount() {
    // Outputs the current index
    console.log( Splide.index );

    // Outputs all Slide sub components
    console.log( Components.Slides.get() );
  }

  return {
    mount,
  }
}
```

## Lifecycle (source-verified)

`Splide#mount( Extensions?, Transition? )` in `core/Splide/Splide.ts` does the following:

1. Asserts the state is `CREATED` or `DESTROYED` (otherwise it throws `Already mounted!`), then sets the state to `CREATED`.
2. Picks the transition: `this._T = Transition || this._T || ( this.is( 'fade' ) ? Fade : Slide )`.
3. Picks the extensions: `this._E = Extensions || this._E`. Calling `mount()` with no argument reuses the previous set, and passing an object replaces the previous set. Sets are never merged.
4. Builds `Constructors = assign( {}, CoreComponents, this._E, { Transition: this._T } )`.
5. For each constructor, **in key order**: `component = Constructor( splide, Components, options )`, `Components[ key ] = component`, then `component.setup && component.setup()`. Each component is constructed and set up before the next one is constructed.
6. For each entry of `Components`, in key order: `component.mount && component.mount()`.
7. Emits `mounted`, adds `is-initialized` to the root, sets the state to `IDLE`, then emits `ready`.

The key order is the 19 core components, then your extensions in the order you listed them, then `Transition`:

`Media, Direction, Elements, Slides, Layout, Clones, Move, Controller, Arrows, Autoplay, Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination, Sync, Wheel, Live, <your extensions...>, Transition`

Logged in Chrome for `mount( { Logger, Logger2 } )`: Logger constructed, Logger `setup`, Logger2 `setup`, Logger `mount`, Logger2 `mount`, the `mounted` event, the `ready` event. `Transition.mount()` runs between the last extension's `mount` and `mounted` (from the source). On `destroy()`, the log read: Logger2 `destroy`, Logger `destroy( true )`, then the `destroy` event.

What is available at each stage of **your** extension:

| Stage | Core components | Slides registered (`Slides.get()`) | Arrows/pagination created | `Components.Transition` | Other extensions |
|---|---|---|---|---|---|
| Constructor body | All constructed and set up. `Elements` has collected `root`, `track`, `list` and `slides` | No, `[]` | No | `undefined` on the first mount. On a remount it is the previous, destroyed Transition object | Only the ones listed before yours |
| `setup()` | Same as the constructor | No | No | Same as the constructor | Only the ones listed before yours |
| `mount()` | All mounted | Yes, clones included | Yes | Constructed but not mounted yet | Those listed before yours are mounted |
| `on( 'mounted' )` / `on( 'ready' )` | Everything mounted. `ready` comes after the state is `IDLE` | Yes | Yes | Mounted | All mounted |

So you can destructure `Components.Elements` (`track`, `list`, `root`, `slides`) in the constructor, as the docs' SlideNumber does. Read `Components.Transition`, `Slides.get()` and the arrows inside `mount()` or later.

`Splide#destroy( completely = true )`:

1. If the state is still `CREATED` (destroy was called during `mount()`, for example by a matching `destroy` breakpoint at load), destruction is postponed until `ready`.
2. Otherwise, it calls `component.destroy( completely )` on every component **in reverse order**: Transition first, then the extensions last-to-first, then the core components.
3. It emits `destroy`. Every `EventInterface( splide )` tears itself down on this event.
4. `splide.event.destroy()` removes every handler registered with `splide.on()`.
5. If `completely` is true, it empties `splide.splides` (the sync targets).
6. It sets the state to `DESTROYED`.

Your `destroy()` therefore runs while all event handlers are still attached and other components still exist.

`completely`:

- `splide.destroy()` or `splide.destroy( true )`, and the breakpoint option `destroy: 'completely'`, pass `true`. Nothing remounts automatically.
- The breakpoint option `destroy: true` calls `splide.destroy( false )`. When that breakpoint stops matching, `Media` calls `splide.mount()` again with the stored extensions and transition.

**Remounting constructs everything again.** Each `mount()`, manual or from breakpoints, calls your constructor again, so you get a new closure. State kept inside the constructor is lost. The controller index also restarts from the `start` option. To carry state across remounts, keep it outside the constructor in a factory closure (see [Example 4](#example-4-listeners-that-survive-breakpoint-destroy)).

Remount caveats:

- `splide.Components` is the same object for the instance's whole life. If you mount `{ A }`, destroy, then mount `{ B }`, the stale `A` entry stays in `Components`, and Splide calls its `mount()` again. Remount with the same set or with no argument.
- `splide.root` keeps the `is-initialized` class after destroy. While mounted, the root also has `is-active`, which `Elements` removes on destroy, and the CSS rule `.splide.is-initialized:not(.is-active) .splide__list { display: block }` is what un-flexes a destroyed slider. To test for destroyed, use `splide.state.is( Splide.STATES.DESTROYED )`.
- `Splide.STATES` = `{ CREATED: 1, MOUNTED: 2, IDLE: 3, MOVING: 4, SCROLLING: 5, DRAGGING: 6, DESTROYED: 7 }`. `MOUNTED` is never set by `mount()`, which goes from `CREATED` straight to `IDLE`.

## Accessing the instance and other components

Instance members (first argument):

| Member | Description |
|---|---|
| `root` | Root element. |
| `index` | Current index (getter, `Controller.getIndex()`). It is already the new index during `move`. |
| `length` | Slide count without clones (getter). |
| `options` | Getter for the live options object. The setter `splide.options = {...}` merges into the current and base options and emits `updated`. |
| `state` | `{ set( n ), is( n \| n[] ) }`. Read it with `is()`. `set()` is internal. |
| `Components` | Same object as the second argument. |
| `splides` | Sync targets, `[ { splide, isParent? } ]`. |
| `event` | The instance's own `EventInterface` object (`on`, `off`, `emit`, `bind`, `unbind`, `dispatch`, `destroy`). Public in the type declarations but not described in the guides. |
| `is( type )` | `true` if `options.type === type`. |
| `go( control )` | `n`, `'+n'`, `'-n'`, `'>'`, `'<'`, `'>n'` (page n). |
| `on`, `off`, `emit` | See [Events](#events). |
| `add( slides, index? )`, `remove( matcher )`, `refresh()`, `sync( splide )`, `destroy( completely? )`, `mount()` | As in the APIs reference. |

Core components and their public members (from `components/*/*.ts`):

| Component | Public members |
|---|---|
| `Media` | `set( options, base?, notify? )`. `reduce( enable )` is internal. |
| `Direction` | `resolve( prop, axisOnly?, direction? )` maps `'width'`/`'left'`/`'right'`/`'X'`/`'Y'`/`'ArrowLeft'`/`'ArrowRight'` to the current direction, and `orient( value )` returns `-value` for ltr/ttb and `value` for rtl. |
| `Elements` | `root`, `track`, `list`, `slides` (original slide elements, no clones), `arrows?`, `pagination?`, `prev?`, `next?`, `bar?`, `toggle?`. The optional ones are only elements present in your markup. Generated arrows live in `Arrows.arrows`. |
| `Slides` | `get( excludeClones? )`, `getIn( page )`, `getAt( index )`, `forEach( fn( Slide, i, all ), excludeClones? )`, `filter( index \| index[] \| selector \| predicate )`, `style( prop, value, useContainer? )`, `getLength( excludeClones? )`, `isEnough()`, `add( slides, index? )`, `remove( matcher )`, `update()`, `register( el, index, slideIndex )` |
| Slide (an item from `Slides.get()`) | `index`, `slideIndex` (the original's index for a clone, `-1` otherwise), `slide` (element), `container`, `isClone`, `update()`, `style( prop, value, useContainer? )`, `isWithin( from, distance )` |
| `Layout` | `listSize()`, `slideSize( index, withoutGap? )`, `sliderSize( withoutGap? )`, `totalSize( index?, withoutGap? )`, `getPadding( right )`, `isOverflow()`, `resize( force? )` |
| `Clones` | none |
| `Move` | `move( dest, index, prev, callback? )`, `jump( index )`, `translate( position, preventLoop? )`, `shift( position, backwards )`, `cancel()`, `toIndex( position )`, `toPosition( index, trimming? )`, `getPosition()`, `getLimit( max )`, `exceededLimit( max?, position? )`. `reposition()` is internal. |
| `Controller` | `go( control, allowSameIndex?, callback? )`, `scroll( destination, duration?, snap?, callback? )`, `getNext( destination? )`, `getPrev( destination? )`, `getEnd()`, `setIndex( index )`, `getIndex( prev? )`, `toIndex( page )`, `toPage( index )`, `toDest( position )`, `hasFocus()`, `isBusy()` |
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
| `Transition` | `start( index, done )`, `cancel()`. See `custom-transitions.md`. |

**Exposing your own API.** Return extra members and they become reachable from outside, for example `splide.Components.SlideCounter.render()`. Other extensions can call them too, but only from `mount()` or later, or when they are listed after yours.

## Options for your extension

**How options reach it.** Options you don't recognise are kept, so add your own key:

```js
new Splide( '.splide', {
  slideCounter: { pages: true },                                // your extension's option
  breakpoints : {
    640: { slideCounter: { format: '{current}/{total}' } },     // deep-merged over the base value
  },
} ).mount( { SlideCounter } );
```

Options can also come from `data-splide='{"slideCounter":{"pages":true}}'` (it must be valid JSON) or from `Splide.defaults = { slideCounter: { ... } }`, set before `new Splide()`. The constructor merges, in this order: `{ label, labelledby }` from the root attributes, `DEFAULTS`, `Splide.defaults`, the constructor options, then `data-splide`.

The object your extension receives is built as `Object.create( mergedOptions )`:

- The **prototype** holds the base options.
- **Own properties** hold the overrides from the currently matching breakpoints, plus `reducedMotion` when the media query matches. When the matching breakpoints change, `Media` deletes all own properties and merges the matching breakpoint options in again (deep-merged over the base value), then emits `updated`.
- `splide.options = { ... }` merges into both the own properties and the prototype, then emits `updated` with the options.
- Splide's merge copies plain objects recursively, copies arrays with `slice()`, and replaces any other value.

Rules that follow from this:

- **Read options when you need them, not once in the constructor.** `options.slideCounter` read inside `render()` reflects the breakpoint that applies now. A value cached in the constructor goes stale. Re-render on `updated`.
- Breakpoint options merge over the base, so `{ slideCounter: { format } }` in a breakpoint keeps the base `pages` value.
- **Merge your defaults yourself.** Splide's `merge` utility is not exported by either build.

Defaults via a shallow merge, which is enough for flat option objects:

```js
const SLIDE_COUNTER_DEFAULTS = { format: '{current} / {total}', pages: false, className: 'splide__counter' };

function config( options ) {
  // `true`, `false`, `undefined` and other primitives contribute no keys to Object.assign.
  return Object.assign( {}, SLIDE_COUNTER_DEFAULTS, options.slideCounter );
}
```

Defaults via a deep merge, for nested option objects:

```js
function mergeOptions( defaults, value ) {
  const out = Object.assign( {}, defaults );

  if ( value && typeof value === 'object' ) {
    Object.keys( value ).forEach( key => {
      const v = value[ key ];
      const d = defaults[ key ];
      out[ key ] = v && typeof v === 'object' && ! Array.isArray( v ) && d && typeof d === 'object'
        ? mergeOptions( d, v )
        : v;
    } );
  }

  return out;
}
```

The "boolean or object" pattern used by official extensions (`myExt: true` or `myExt: { ... }`): treat `false` and `undefined` as off, and merge an object over the defaults. Check `options.myExt` again on `updated` if breakpoints may toggle it (as [Example 2](#example-2-slide-counter-with-options) does).

## Events

### `Splide#on` / `off` / `emit` (both builds)

```js
function MyExtension( Splide, Components, options ) {
  function mount() {
    Splide.on( 'resize', onResize );
  }

  function onResize() {
    // do something
  }

  return {
    mount,
  };
}
```

Source behaviour (constructors/EventBinder + EventInterface):

- `on( 'move', fn )`, `on( 'move resize', fn )` (several events, separated by spaces), `on( [ 'move', 'resize' ], fn )`, and namespaces `on( 'move.myExt', fn )`.
- Handlers get the emitted arguments spread out: `( index, prev, dest )` for `move`. `this` inside a non-arrow handler is the handler function itself.
- Handlers run in registration order. Core components register in their `mount()`, before yours, so on `updated` or `refresh` the core (`Controller`, `Layout` and the others) has already re-initialised when your handler runs.
- `off( events )` takes no callback. It removes handlers registered **through `splide.on`** whose event name **and namespace** match exactly. `off( 'move' )` removes only un-namespaced `move` handlers, `off( 'move.myExt' )` removes only that namespace, and neither touches handlers registered through an `EventInterface`. Verified: after `splide.off( 'moved' )`, a `'moved.ns'` handler and an `EventInterface` handler both still fired. **Always namespace the handlers in classic-script extensions** so user code calling `splide.off( 'move' )` can't remove them.
- `emit( name, ...args )` dispatches a bubbling `CustomEvent` on the shared bus (a `DocumentFragment`), with the args array as `detail`. Everything on the same instance hears it: `splide.on`, every `EventInterface( splide )`, and `splide.event.on`.
- Everything registered with `splide.on` is removed by `destroy()`, **including breakpoint destroys**, and is not restored on remount (see [Example 4](#example-4-listeners-that-survive-breakpoint-destroy)).

### `EventInterface( splide )` (ESM only)

```js
import { EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

function MyExtension( Splide, Components, options ) {
  const { on, off, bind, unbind } = EventInterface( Splide );

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
}
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

`EventBinder()` (ESM only) is the DOM-only part: `{ bind, unbind, dispatch, destroy }` with the same semantics. It is **not** tied to any instance, so call `destroy()` yourself in your `destroy()`. The core `Media` uses it for `matchMedia` listeners that must outlive `destroy( false )`.

### Classic-script equivalents

Classic scripts have no `EventInterface`, so:

- Bus events: `splide.on( 'move.myExt', fn )`. These are removed automatically on destroy.
- DOM events: call `addEventListener` and keep the references, then call `removeEventListener` in your `destroy()`. A small helper:

```js
function createBinder() {
  const list = [];
  return {
    bind( target, type, fn, opts ) {
      if ( target ) {
        target.addEventListener( type, fn, opts );
        list.push( [ target, type, fn, opts ] );
      }
    },
    destroy() {
      list.forEach( ( [ target, type, fn, opts ] ) => target.removeEventListener( type, fn, opts ) );
      list.length = 0;
    },
  };
}
```

- Alternative: `splide.event.bind( target, 'click.myExt', fn )` uses the instance's own `EventInterface`, which is public in the types but undocumented in the guides. It is removed automatically by `destroy()`, as verified with the UMD build. The listeners share the instance's main listener list, so namespace them.

### Event names and arguments

| Event | Arguments | Notes |
|---|---|---|
| `mounted` | none | After all components have mounted. |
| `ready` | none | After `mounted`, with the state `IDLE`. |
| `move` | `index, prev, dest` | Before the transition starts. `dest` can be a clone index in loop mode. |
| `moved` | `index, prev, dest` | When the Transition calls `done`. |
| `click` | `Slide, event` | A slide was clicked. |
| `active` / `inactive` | `Slide` | |
| `visible` / `hidden` | `Slide` | |
| `refresh` | none | `splide.refresh()`, `add()` and `remove()`. |
| `updated` | `options` | Options changed: breakpoints, or the `splide.options` setter. |
| `resize` / `resized` | none | `resize`: window `resize`/`load` (throttled to one per animation frame), a lazy-loaded image finishing, or images in slides added with `add()` loading. `resized`: after `Layout` re-measures (only when the root size changed, or forced on init, `updated` and `refresh`). |
| `drag` / `dragging` / `dragged` | none | |
| `scroll` / `scrolled` | none | Scroll component: `drag: 'free'`, `useScroll`, `Controller.scroll()`. |
| `overflow` | `overflow` (boolean) | |
| `destroy` | none | During `destroy()`, after the components' `destroy()`. |
| `arrows:mounted` | `prev, next` | |
| `arrows:updated` | `prev, next, prevIndex, nextIndex` | |
| `pagination:mounted` | `data, item` | `data = { list, items }` |
| `pagination:updated` | `data, prev, curr` | |
| `navigation:mounted` | `splides` | |
| `autoplay:play` / `autoplay:pause` | none | |
| `autoplay:playing` | `rate` (0 to 1) | |
| `lazyload:loaded` | `img, Slide` | |
| `sk`, `sh`, `ei` | `sk`: `Slide, event` | Internal: slide keydown, shifted (loop jump), end index changed. |

**Custom events.** Emit your own events, namespaced by convention: `emit( 'slidecounter:updated', current, total )` (ESM) or `splide.emit( 'slidecounter:updated', current, total )` (both builds). Users listen with `splide.on( 'slidecounter:updated', ( current, total ) => {} )`. Avoid names that clash with core events.

## Cleanup

Removed automatically on any destroy (`completely` true or false):

- Handlers from `splide.on()` and `splide.event.bind()`.
- Everything registered through any `EventInterface( splide )`, bus and DOM listeners alike.
- Splide's own changes: the core restores slide `style` attributes, removes status classes and ARIA attributes, strips `style` from the track and list, and removes the arrows and pagination it generated.

You must undo yourself, in `destroy()`:

- DOM nodes you created. The docs' SlideNumber doesn't do this, and after `destroy()` plus `mount()` the page had two counters (verified).
- Classes and inline styles you put on elements Splide doesn't reset (the root, or outside elements).
- Timers: `setTimeout`, `setInterval`, `requestAnimationFrame`, and `RequestInterval` (call `cancel()`).
- Observers (`IntersectionObserver`, `ResizeObserver`, `MutationObserver`), and `EventBinder()` objects.
- In classic scripts, every `addEventListener` you made.

Make `destroy()` idempotent and safe after a partial mount: check for `null` and reset references.

## Helpers for extension authors

All of these are ESM exports. Classic scripts have only `Splide.STATES`.

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

```js
import { RequestInterval } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

const timer = RequestInterval( 3000, () => splide.go( '>' ), rate => { bar.style.width = `${ rate * 100 }%`; } );
timer.start();            // from 0
timer.pause();            // keep progress
timer.start( true );      // resume
timer.cancel();           // stop for good: call this in destroy()
```

Classic equivalent: `requestAnimationFrame` with `performance.now()`, or `setInterval`, cleared in `destroy()`.

### `Throttle( func, duration? )`

`Throttle( func, duration = 0 )` returns a function. The first call starts a `RequestInterval( duration, func, null, 1 )`. Calls made while it is pending are ignored, and `func` runs once after `duration` ms, on the next frame when `duration` is 0. It is **trailing-only**, and **`func` receives no arguments**: the throttled function's arguments are dropped (verified). `Layout` uses `Throttle( () => emit( 'resize' ) )` on window `resize`.

```js
const onScroll = Throttle( () => update(), 100 );   // reads state itself; gets no event object
bind( window, 'scroll', onScroll );
```

### `State( initialState )`

Returns `{ set( value ), is( value | value[] ) }`. This is what `splide.state` is. It is useful for small state machines in your own extension.

### Constants

- `EVENT_*`: `EVENT_MOUNTED 'mounted'`, `EVENT_READY 'ready'`, `EVENT_MOVE 'move'`, `EVENT_MOVED 'moved'`, `EVENT_CLICK 'click'`, `EVENT_ACTIVE 'active'`, `EVENT_INACTIVE 'inactive'`, `EVENT_VISIBLE 'visible'`, `EVENT_HIDDEN 'hidden'`, `EVENT_REFRESH 'refresh'`, `EVENT_UPDATED 'updated'`, `EVENT_RESIZE 'resize'`, `EVENT_RESIZED 'resized'`, `EVENT_DRAG 'drag'`, `EVENT_DRAGGING 'dragging'`, `EVENT_DRAGGED 'dragged'`, `EVENT_SCROLL 'scroll'`, `EVENT_SCROLLED 'scrolled'`, `EVENT_OVERFLOW 'overflow'`, `EVENT_DESTROY 'destroy'`, `EVENT_ARROWS_MOUNTED 'arrows:mounted'`, `EVENT_ARROWS_UPDATED 'arrows:updated'`, `EVENT_PAGINATION_MOUNTED 'pagination:mounted'`, `EVENT_PAGINATION_UPDATED 'pagination:updated'`, `EVENT_NAVIGATION_MOUNTED 'navigation:mounted'`, `EVENT_AUTOPLAY_PLAY 'autoplay:play'`, `EVENT_AUTOPLAY_PLAYING 'autoplay:playing'`, `EVENT_AUTOPLAY_PAUSE 'autoplay:pause'`, `EVENT_LAZYLOAD_LOADED 'lazyload:loaded'`, `EVENT_SLIDE_KEYDOWN 'sk'`, `EVENT_SHIFTED 'sh'`, `EVENT_END_INDEX_CHANGED 'ei'`.
- `CLASS_*` (for example `CLASS_ACTIVE 'is-active'`, `CLASS_SLIDE 'splide__slide'`, `CLASS_TRACK 'splide__track'`, `CLASS_LIST 'splide__list'`, `CLASS_INITIALIZED 'is-initialized'`), plus `CLASSES` (the default `classes` option) and `STATUS_CLASSES`.
- `DEFAULTS` (the built-in default options), `SLIDE`, `LOOP`, `FADE`, `LTR`, `RTL`, `TTB`.
- The states are not exported as named constants. Use `Splide.STATES`.

## Registering extensions

Pass an object of constructors to `mount()`. The key becomes the component name, `Components.<key>`.

```js
// ESM
import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
import { MyExtension01 } from './my-extension-01.js';   // your own module files (http(s) only)
import { MyExtension02 } from './my-extension-02.js';

new Splide( '#splide' ).mount( {
  MyExtension01,
  MyExtension02,
} );
```

```html
<!-- Classic: the extensions are plain global functions defined in earlier scripts -->
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="/js/my-extensions.js"></script>
<script>
  new Splide( '#splide' ).mount( { MyExtension01, MyExtension02 } );
</script>
```

- **Several extensions:** list them all in one object. They are constructed, set up and mounted in the order listed, after all core components, and destroyed in reverse.
- **With a custom transition:** `mount( { MyExtension01 }, MyTransition )`. A `Transition` key inside the extensions object is ignored, because `{ Transition: this._T }` is assigned last (verified). Use the second argument.
- **Names:** must be unique within the object. A core name **replaces** that core component (next section). Your own names should not collide with core names unless you mean that.
- **Remounts** (`mount()` with no arguments, or breakpoint remounts) reuse the same extensions and transition.
- **Several sliders:** pass the same constructors to each instance's `mount()`. Each instance calls them separately.

## Overriding a built-in component

The guide says "do not use names of core components". The source allows it, and it is the way to replace a core component: `assign( {}, CoreComponents, Extensions, { Transition } )` overwrites the core constructor with yours, and **your constructor keeps the core component's slot** in the construct, setup, mount and destroy order. With `mount( { Keyboard } )`, `Keyboard` was still the 14th of 20 keys (verified). Components that come later and capture it at construction time (`const { Move } = Components`) get yours.

Constraints:

- It is a full replacement. The core constructors are not exported by either build, so you can't wrap or extend the original.
- Implement the API other code relies on:

| Replaced component | Used by |
|---|---|
| `Media` | `Splide` (`options` setter calls `set`), `Sync` (`set`), `Drag` (`reduce`) |
| `Direction` | `Clones`, `Drag`, `Keyboard`, `Layout`, `Move`, `Pagination`, `Slide` (`resolve`, `orient`) |
| `Elements` | nearly every component (`root`, `track`, `list`, `slides`, ...) |
| `Slides` | `Splide` (`add`, `remove`, `getLength`), `Autoplay`, `Clones`, `Controller`, `Cover`, `Layout`, `LazyLoad`, `Move`, `Pagination`, `Fade` |
| `Layout` | `Clones`, `Drag`, `Move`, `Scroll` |
| `Move` | `Controller`, `Scroll`, `Drag`, `Slide` transition |
| `Controller` | `Splide` (`go`, `index`), `Arrows`, `Move`, `Pagination`, `Scroll`, `Slides`, `Wheel`, `Drag`, `Slide` transition |
| `Scroll` | `Controller`, `Move`, `Drag`, `Slide` transition |
| `Autoplay` | `Live` (`isPaused`) |
| `Sync` | `Splide#sync()` (`remount`) |
| `Arrows`, `Clones`, `Cover`, `Drag`, `Keyboard`, `LazyLoad`, `Pagination`, `Wheel`, `Live` | No other core component calls them. Keep their public methods (for example `Keyboard.disable`) if user code does. |

- Replacing the leaf components in the last row is low-risk. Replacing anything above needs the full interface.
- To turn a core feature off, use its option (`keyboard: false`, `wheel: false`, `arrows: false`, `drag: false`). Replacing it with `() => ( {} )` also works but is rarely needed.

Example: replace `Keyboard` with one that also handles Home and End, keeping `disable()` and the `keyboard` option. In the ESM form below, the classic form swaps `bind`/`unbind` for `addEventListener`/`removeEventListener` and the `on` call for `splide.on( 'updated.keyboard', ... )`.

```html
<script type="module">
  import { Splide, EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function Keyboard( splide, Components, options ) {
    const { on, bind, unbind } = EventInterface( splide );
    const { resolve } = Components.Direction;
    let target   = null;
    let disabled = false;

    function mount() {
      init();
      on( 'updated', () => {
        destroy();
        init();
      } );
    }

    function init() {
      const { keyboard } = options;

      if ( keyboard ) {
        // Same targets as the core: 'global' listens on window, any other truthy value on the root.
        target = keyboard === 'global' ? window : splide.root;
        bind( target, 'keydown', onKeydown );
      }
    }

    function destroy() {
      if ( target ) {
        unbind( target, 'keydown', onKeydown );
        target = null;
      }
    }

    function disable( value ) {
      disabled = value;
    }

    function onKeydown( e ) {
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

  new Splide( '.splide', { keyboard: true } ).mount( { Keyboard } );
</script>
```

## Example 1: SlideNumber (the docs example)

The docs build it in two steps: first create the element, then add `update()` with a `move` listener. Below is the final version, plus a `destroy()` that the docs leave out. Without it, a remount adds a second counter.

ESM:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">

<section class="splide" aria-label="Slide number example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">01</li>
      <li class="splide__slide">02</li>
      <li class="splide__slide">03</li>
    </ul>
  </div>
</section>

<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function SlideNumber( Splide, Components ) {
    // Safe in the constructor: Elements collected track/list in its setup().
    const { track } = Components.Elements;

    let elm;

    function mount() {
      elm = document.createElement( 'div' );
      elm.style.textAlign = 'center';
      elm.style.marginTop = '0.5em';

      // Insert the element after the track
      track.parentElement.insertBefore( elm, track.nextSibling );

      update();
      Splide.on( 'move', update );
    }

    function update() {
      elm.textContent = `${ Splide.index + 1 }/${ Splide.length }`;
    }

    // Not in the docs: without this, destroy() + mount() (or a breakpoint remount) leaves a duplicate.
    function destroy() {
      elm.remove();
    }

    return {
      mount,
      destroy,
    };
  }

  new Splide( '.splide' ).mount( { SlideNumber } );
</script>
```

Classic: the same function body. Only the loading and mounting change:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function SlideNumber( Splide, Components ) {
    const { track } = Components.Elements;
    let elm;

    function mount() {
      elm = document.createElement( 'div' );
      elm.style.textAlign = 'center';
      elm.style.marginTop = '0.5em';
      track.parentElement.insertBefore( elm, track.nextSibling );

      update();
      Splide.on( 'move.slideNumber', update );   // namespaced: survives splide.off( 'move' )
    }

    function update() {
      elm.textContent = `${ Splide.index + 1 }/${ Splide.length }`;
    }

    function destroy() {
      elm.remove();
    }

    return { mount, destroy };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    // Here `Splide` is window.Splide: the parameter only shadows it inside SlideNumber.
    new Splide( '.splide' ).mount( { SlideNumber } );
  } );
</script>
```

`Elements` holds the main elements (`track`, `list`, `slides`). `Splide.index` is the current index, already updated when `move` fires. `Splide.length` is the slide count without clones.

## Example 2: Slide counter with options

This version adds options with defaults, slide or page counting, breakpoint-aware re-rendering, the ability to turn it on and off at runtime, and cleanup. Verified: with `perPage: 3`, 10 slides and `pages: true` it shows `1 / 4`, then `2 / 4` after `go( '>' )`. At a 700px breakpoint with `{ perPage: 2, slideCounter: { format: '{current} of {total}' } }` it shows `2 of 5`. `splide.options = { slideCounter: false }` removes it, and `destroy()` removes it.

ESM:

```html
<section class="splide" aria-label="Products">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
      <!-- more slides -->
    </ul>
  </div>
</section>

<script type="module">
  import {
    Splide,
    EventInterface,
    EVENT_MOVE,
    EVENT_REFRESH,
    EVENT_UPDATED,
  } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  const SLIDE_COUNTER_DEFAULTS = {
    format   : '{current} / {total}',
    pages    : false,            // true: count pages (perPage), false: count slides
    className: 'splide__counter',
  };

  function SlideCounter( splide, Components, options ) {
    const { on } = EventInterface( splide );
    const { Controller, Elements } = Components;
    let elm = null;

    // Read on every render: breakpoints and `splide.options = {}` mutate `options` in place.
    function config() {
      return Object.assign( {}, SLIDE_COUNTER_DEFAULTS, options.slideCounter );
    }

    function mount() {
      render();
      // Core components re-initialise on these events before this handler runs.
      on( [ EVENT_MOVE, EVENT_REFRESH, EVENT_UPDATED ], render );
    }

    function render() {
      if ( options.slideCounter === false ) {
        remove();
        return;
      }

      const conf  = config();
      const index = splide.index;
      let current = index + 1;
      let total   = splide.length;

      if ( conf.pages ) {
        // Same page maths as the Pagination component.
        current = Controller.toPage( index ) + 1;
        total   = Controller.hasFocus()
          ? Controller.getEnd() + 1
          : Math.ceil( splide.length / options.perPage );
      }

      if ( ! elm ) {
        elm = document.createElement( 'div' );
        Elements.track.insertAdjacentElement( 'afterend', elm );
      }

      elm.className   = conf.className;
      elm.textContent = conf.format.replace( '{current}', current ).replace( '{total}', total );
    }

    function remove() {
      if ( elm ) {
        elm.remove();
        elm = null;
      }
    }

    return {
      mount,
      destroy: remove,
      render,                    // public: splide.Components.SlideCounter.render()
    };
  }

  new Splide( '.splide', {
    perPage     : 3,
    slideCounter: { pages: true },
    breakpoints : {
      700: { perPage: 2, slideCounter: { format: '{current} of {total}' } },
    },
  } ).mount( { SlideCounter } );
</script>
```

Classic: identical apart from no import and namespaced `splide.on` instead of `EventInterface`. The namespaced listener kept working after `splide.off( 'move' )` (verified).

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  const SLIDE_COUNTER_DEFAULTS = {
    format   : '{current} / {total}',
    pages    : false,
    className: 'splide__counter',
  };

  function SlideCounter( splide, Components, options ) {
    const { Controller, Elements } = Components;
    let elm = null;

    function config() {
      return Object.assign( {}, SLIDE_COUNTER_DEFAULTS, options.slideCounter );
    }

    function mount() {
      render();
      splide.on( 'move.slideCounter refresh.slideCounter updated.slideCounter', render );
    }

    function render() {
      if ( options.slideCounter === false ) {
        remove();
        return;
      }

      const conf  = config();
      const index = splide.index;
      let current = index + 1;
      let total   = splide.length;

      if ( conf.pages ) {
        current = Controller.toPage( index ) + 1;
        total   = Controller.hasFocus()
          ? Controller.getEnd() + 1
          : Math.ceil( splide.length / options.perPage );
      }

      if ( ! elm ) {
        elm = document.createElement( 'div' );
        Elements.track.insertAdjacentElement( 'afterend', elm );
      }

      elm.className   = conf.className;
      elm.textContent = conf.format.replace( '{current}', current ).replace( '{total}', total );
    }

    function remove() {
      if ( elm ) {
        elm.remove();
        elm = null;
      }
    }

    return { mount, destroy: remove, render };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', { perPage: 3, slideCounter: { pages: true } } ).mount( { SlideCounter } );
  } );
</script>
```

Splide's `live` option (on by default) already announces slide changes to screen readers, so the counter has no `aria-live` of its own.

## Example 3: Keyboard shortcuts

These shortcuts add to the core arrow keys (the `keyboard` option): Home and End go to the first slide and the last reachable index, PageUp and PageDown go to the previous and next page, `1` to `9` jump to that page, and `p` toggles autoplay. The extension uses its own option, `shortcuts: true` (listens on the root, so focus must be inside the slider) or `'global'` (listens on `window`). It ignores key presses while the user types in form fields or with modifier keys held, and rebinds on `updated` so breakpoints can change the mode. Verified with 9 slides and `perPage: 2`: End went to 7, PageDown to 2, `3` to 4, `p` paused autoplay, and nothing responded after `destroy()`.

ESM:

```html
<script type="module">
  import {
    Splide,
    EventInterface,
    EVENT_UPDATED,
  } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function KeyboardShortcuts( splide, Components, options ) {
    const { on, bind, unbind } = EventInterface( splide );
    const { Controller, Autoplay } = Components;
    let target = null;

    function mount() {
      listen();
      on( EVENT_UPDATED, () => {
        unlisten();
        listen();
      } );
    }

    function listen() {
      const mode = options.shortcuts;

      if ( mode ) {
        target = mode === 'global' ? window : splide.root;
        bind( target, 'keydown', onKeydown );
      }
    }

    function unlisten() {
      if ( target ) {
        unbind( target, 'keydown', onKeydown );
        target = null;
      }
    }

    function isTyping( el ) {
      return !! ( el && el.closest && el.closest( 'input, textarea, select, [contenteditable]:not([contenteditable="false"])' ) );
    }

    function onKeydown( e ) {
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
  }

  new Splide( '.splide', { perPage: 2, autoplay: true, shortcuts: true } ).mount( { KeyboardShortcuts } );
</script>
```

Classic: replace `EventInterface` with `addEventListener`/`removeEventListener` (removed in `destroy`) and a namespaced `splide.on`:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function KeyboardShortcuts( splide, Components, options ) {
    const { Controller, Autoplay } = Components;
    let target = null;

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
        target.addEventListener( 'keydown', onKeydown );
      }
    }

    function unlisten() {
      if ( target ) {
        target.removeEventListener( 'keydown', onKeydown );
        target = null;
      }
    }

    function isTyping( el ) {
      return !! ( el && el.closest && el.closest( 'input, textarea, select, [contenteditable]:not([contenteditable="false"])' ) );
    }

    function onKeydown( e ) {
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
        splide.go( `>${ +key - 1 }` );
      } else if ( ( key === 'p' || key === 'P' ) && options.autoplay ) {
        Autoplay.isPaused() ? Autoplay.play() : Autoplay.pause();
      } else {
        handled = false;
      }

      if ( handled ) {
        e.preventDefault();
      }
    }

    return { mount, destroy: unlisten };   // destroy() removes the DOM listener
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', { perPage: 2, shortcuts: true } ).mount( { KeyboardShortcuts } );
  } );
</script>
```

Notes:

- With `shortcuts: true`, key presses reach the root only from focused elements inside it (slides with `slideFocus`, arrows, links). Pagination buttons handle ArrowLeft, ArrowRight, Home and End themselves and call `stopPropagation()`, so those keys never reach this handler from a pagination button.
- `Autoplay.play()` only makes sense when the `autoplay` option is on, which is why the handler checks it.
- For RTL or vertical sliders, map arrow keys with `Components.Direction.resolve( 'ArrowLeft' )` or `resolve( 'ArrowRight' )`, as the core `Keyboard` does.

## Example 4: Listeners that survive breakpoint destroy

**The problem.** With `breakpoints: { 700: { destroy: true } }`, narrowing the window calls `splide.destroy( false )`, which removes every handler registered with `splide.on()`. Widening it again calls `splide.mount()`. That remount re-runs every extension but does **not** restore handlers registered outside them. Verified: a `splide.on( 'moved', ... )` made before `mount()` never fired again after one destroy and remount cycle, while the same handler registered inside an extension kept firing. The same goes for listeners made with `EventInterface( splide )` outside an extension. Also, if the page loads inside the destroying breakpoint, the destroy is postponed until `ready`, so handlers registered before `mount()` are dropped right away. And the index restarts from the `start` option after the remount.

**The fix.** Register everything inside an extension. Its `mount()` runs on every (re)mount and its `destroy()` on every destroy. A factory function keeps data across remounts.

ESM:

```html
<button class="my-prev" type="button">Prev</button>
<button class="my-next" type="button">Next</button>

<section class="splide" aria-label="Gallery">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">...</li>
    </ul>
  </div>
</section>

<script type="module">
  import { Splide, EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  // 1) Bus handlers re-registered on every mount.
  function persistentListeners( handlers ) {
    return function PersistentListeners( splide ) {
      function mount() {
        Object.keys( handlers ).forEach( events => {
          splide.on( events, handlers[ events ] );   // keys may be 'moved', 'move resize', 'ready.mine'...
        } );
      }

      return { mount };
    };
  }

  // 2) DOM listeners on elements outside the slider: inert while destroyed, re-bound on remount.
  function ExternalControls( splide, Components, options ) {
    const { bind } = EventInterface( splide );       // auto-unbound on every destroy

    function mount() {
      const conf = options.externalControls || {};
      bind( document.querySelector( conf.prev ), 'click', () => splide.go( '<' ) );  // null is skipped
      bind( document.querySelector( conf.next ), 'click', () => splide.go( '>' ) );
    }

    return { mount };
  }

  // 3) Restore the index after a remount (otherwise it restarts at `start`).
  function rememberIndex() {
    let saved = null;                                 // lives outside the constructor

    return function RememberIndex( splide, Components ) {
      return {
        setup() {
          // setup() runs before any mount(), so Controller, Pagination and Move all mount with this index.
          if ( saved !== null ) {
            Components.Controller.setIndex( saved );
          }
        },
        destroy() {
          saved = splide.index;                       // components still exist during destroy()
        },
      };
    };
  }

  const splide = new Splide( '.splide', {
    breakpoints     : { 700: { destroy: true } },
    externalControls: { prev: '.my-prev', next: '.my-next' },
  } );

  splide.mount( {
    PersistentListeners: persistentListeners( {
      moved      : index => console.log( 'moved to', index ),
      'ready.app': ()    => console.log( 'carousel (re)mounted' ),
    } ),
    ExternalControls,
    RememberIndex: rememberIndex(),
  } );
</script>
```

Classic: `persistentListeners` and `rememberIndex` are the same code, because they use only `splide.on` and `Components`. `ExternalControls` tracks its listeners and removes them in `destroy()`:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function persistentListeners( handlers ) {
    return function PersistentListeners( splide ) {
      return {
        mount() {
          Object.keys( handlers ).forEach( events => splide.on( events, handlers[ events ] ) );
        },
      };
    };
  }

  function ExternalControls( splide, Components, options ) {
    const bound = [];

    function listen( selector, handler ) {
      const el = selector && document.querySelector( selector );

      if ( el ) {
        el.addEventListener( 'click', handler );
        bound.push( [ el, handler ] );
      }
    }

    function mount() {
      const conf = options.externalControls || {};
      listen( conf.prev, () => splide.go( '<' ) );
      listen( conf.next, () => splide.go( '>' ) );
    }

    function destroy() {
      // Runs on breakpoint destroys too (completely === false); mount() re-binds on remount.
      bound.forEach( ( [ el, handler ] ) => el.removeEventListener( 'click', handler ) );
      bound.length = 0;
    }

    return { mount, destroy };
  }

  function rememberIndex() {
    let saved = null;

    return function RememberIndex( splide, Components ) {
      return {
        setup() {
          if ( saved !== null ) {
            Components.Controller.setIndex( saved );
          }
        },
        destroy() {
          saved = splide.index;
        },
      };
    };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', {
      breakpoints     : { 700: { destroy: true } },
      externalControls: { prev: '.my-prev', next: '.my-next' },
    } ).mount( {
      PersistentListeners: persistentListeners( { moved: index => console.log( 'moved to', index ) } ),
      ExternalControls,
      RememberIndex: rememberIndex(),
    } );
  } );
</script>
```

Verified (classic and ESM): at index 3, narrowing below 700px destroyed the slider, and clicking `.my-next` did nothing. Widening remounted it at index 3 with the right slide and pagination page active and the list translated to slide 3. After that, `.my-next` moved to 4 and the persistent `moved` handler fired.

Use `destroy( completely )` when an extension must behave differently on a final destroy, for example to remove UI that should stay visible while breakpoints have the slider destroyed:

```js
function destroy( completely ) {
  if ( completely ) {
    elm.remove();               // final destroy: remove it
  } else {
    elm.hidden = true;          // breakpoint destroy: hide it; the new instance's mount() decides again
  }
}
```

(Each remount constructs a new instance, so the new `mount()` must find or reuse the hidden element, for example by a class lookup, instead of creating another one.)

## Pitfalls checklist

- Read `options.*` when you need it. Re-render on `updated`.
- Read `Slides.get()`, `Arrows.arrows` and `Components.Transition` in `mount()` or later, not in the constructor.
- Namespace every `splide.on` in classic scripts (`'move.myExt'`). `splide.off( 'move' )` only removes un-namespaced handlers.
- `off()` can't remove one particular callback. Use a namespace or a separate `EventInterface` object.
- Remove every DOM node, timer, observer and manual `addEventListener` in `destroy()`. It runs on breakpoint destroys too.
- Remounts construct new instances. Keep state that must persist in a factory closure.
- Register handlers inside an extension if they must survive breakpoint destroys.
- Only use core component names on purpose (to replace one), and implement the API other components use.
- A `Transition` key in the extensions object is ignored. Pass a transition as the second argument to `mount()`.
- ESM needs http(s) and one consistent CDN URL. Don't load the UMD and ESM builds together.
- `Throttle` drops arguments, and `RequestInterval` starts paused until `start()`.

## Docs vs source

- **Lifecycle.** The docs say: optional `mount` and `destroy`. The source also calls an optional `setup()` right after construction (before any `mount()`), and passes `destroy( completely )` a boolean.
- **Core names.** The docs say: "Do not use names of core components". The source allows it, and a same-named extension replaces the core component in the same slot. The `Transition` key is the exception: it is always overwritten by the transition argument.
- **Registration example typo.** The docs' registration example uses `{ MyExtention01, MyExtension02 }`. `MyExtention01` (misspelled) would throw a `ReferenceError` as a shorthand property. Use `MyExtension01`.
- **`off`.** The docs say `EventInterface#off( 'resize' )` "Removes the listener", and the `Splide#off` JSDoc says it removes "all handlers for the specified event". In the source, `off` only removes handlers registered through the same object (`splide.on` or that particular `EventInterface`) that have the same event name and namespace. Namespaced handlers and other objects' handlers survive (verified).
- **`unbind` comment.** The docs' comment above `unbind( window, 'resize' )` repeats "Listens to a native event". It removes the listener.
- **`options` argument.** The docs call it "The latest options". In the source it is the live, prototype-backed options object for that mount, mutated in place by breakpoints and the `options` setter.
- **SlideNumber cleanup.** The docs' SlideNumber has no `destroy()` and leaves its element behind. After `destroy()` and `mount()` there were two counters (verified).
