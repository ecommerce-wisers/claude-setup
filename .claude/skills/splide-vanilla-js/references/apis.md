# APIs

Source: https://splidejs.com/guides/apis/

Splide v4.1.4 from jsDelivr, no build step. Signatures were checked against `core/Splide/Splide.ts` (v4.1.3 source), the 4.1.4 `dist/js/splide.esm.js` and `dist/types/index.d.ts`. TypeScript types are kept as notes where they carry information. Facts marked **(source)** are not on the docs page.

## Contents

- [Loading](#loading)
- [Constructor](#constructor)
- [Class (static) properties](#class-static-properties)
  - [`Splide.defaults`](#splidedefaults)
  - [`Splide.STATES`](#splidestates)
- [Instance properties](#instance-properties)
- [Instance methods](#instance-methods)
  - [`mount()`](#mount)
  - [`sync()`](#sync)
  - [`go()`](#go)
  - [`on()`](#on)
  - [`off()`](#off)
  - [`emit()`](#emit)
  - [`add()`](#add)
  - [`remove()`](#remove)
  - [`is()`](#is)
  - [`refresh()`](#refresh)
  - [`destroy()`](#destroy)
- [State machine](#state-machine)
- [What works before mount()](#what-works-before-mount)
- [Components reference](#components-reference)
- [Docs vs source](#docs-vs-source)

## Loading

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  var splide = new Splide( '.splide' ).mount();
</script>
```

The script build exposes only `window.Splide`, with the static `Splide.defaults` and `Splide.STATES`. For `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, the `EVENT_*`/`CLASS_*` constants, `DEFAULTS`, `CLASSES`, `STATUS_CLASSES`, `SLIDE`/`LOOP`/`FADE` or `LTR`/`RTL`/`TTB`, import the self-contained ESM build in a module script:

```html
<script type="module">
  import { Splide, EventInterface, LOOP } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  new Splide( '.splide', { type: LOOP } ).mount();
</script>
```

Use one build per page. The ESM `Splide` is a separate copy from `window.Splide`, and its `defaults` are separate too.

## Constructor

```js
new Splide( target, options )
```

| Param | Type | Description |
|---|---|---|
| `target` | `string \| HTMLElement` | A selector (the first match is used, via `document.querySelector`) or the root element itself |
| `options` | `Options` | Optional. An object with options |

From source:

- Throws `Error: [splide] null is invalid.` if the selector matches nothing.
- Options are deep-merged in this order, with later sources winning:
  1. `{ label: root aria-label, labelledby: root aria-labelledby }` from the HTML
  2. `DEFAULTS`
  3. `Splide.defaults`
  4. `options`
  5. the JSON in the root's `data-splide` attribute
- Invalid `data-splide` JSON throws `Error: [splide] Invalid JSON`.
- Nothing is rendered until `mount()`.

```html
<div class="splide" data-splide='{"perPage":3,"gap":"1rem"}'> ... </div>
```

## Class (static) properties

### `Splide.defaults`

`defaults: Options`

Changes the default options for all Splide instances.

```js
Splide.defaults = {
  type   : 'loop',
  perPage: 3,
};
```

**(source)** Defaults are read in the constructor, so set them before `new Splide()`. Assigning a new object replaces anything set before. To add or change one key, assign it directly, for example `Splide.defaults.i18n = {...}` or `Splide.defaults.perPage = 3`. Nested objects such as `i18n`, `classes` and `reducedMotion` are deep-merged with the built-in defaults.

### `Splide.STATES`

`STATES: Record<string, number>`

The collection of carousel states. You will not need `MOUNTED`, because the state changes straight to `IDLE`.

| Key | Value **(source)** | Meaning (docs) |
|---|---|---|
| `CREATED` | `1` | Splide has just been created |
| `MOUNTED` | `2` | Splide has mounted its components. **(source)** Never set by any code in v4 |
| `IDLE` | `3` | Splide is ready |
| `MOVING` | `4` | Splide is moving |
| `SCROLLING` | `5` | **(source, not in docs)** The Scroll component is scrolling: free drag, `useScroll`, `Controller.scroll()` |
| `DRAGGING` | `6` | **(source, not in docs)** The user is dragging |
| `DESTROYED` | `7` | Splide has been destroyed |

```js
var splide = new Splide( '.splide' );

if ( splide.state.is( Splide.STATES.IDLE ) ) {
  // do something
}
```

## Instance properties

| Property | Type | Access | Description |
|---|---|---|---|
| `root` | `HTMLElement` | readonly | The root element Splide is applied to |
| `event` | `EventInterfaceObject` | readonly | The instance's event interface. You don't need to access it: use `on()`, `off()` and `emit()` |
| `Components` | `Components` | readonly | All component objects, keyed by name. Empty until `mount()` |
| `state` | `StateObject` | readonly | The state object: `state.is( state: number \| number[] ): boolean` and `state.set( state: number ): void`. See [`STATES`](#splidestates) |
| `splides` | `SyncTarget[]` | readonly | Data for the instances this one is synced with (^3.3.1). Each item: `splide: Splide`, and `isParent: boolean \| undefined`, which is `true` if that instance is the parent that called `sync()` |
| `options` | `Options` | get / set | Get: an object with the latest options. Set: merges the given options into the current ones and emits `updated` |
| `length` | `number` | get | The number of slides. **(source)** Clones are excluded (`Slides.getLength( true )`) |
| `index` | `number` | get | The active slide index. **(source)** Always a real index from `0` to `length - 1`, even in loop mode |

`Components` example:

```js
var splide = new Splide( '.splide' );
splide.mount();

var Autoplay = splide.Components.Autoplay;
Autoplay.pause();
```

### `options`

Only responsive options can be changed by assigning an object:

```js
var splide = new Splide( '.splide' ).mount();

splide.options = {
  perPage: 4,
};
```

Do not change other (readonly) options, and do not set a property directly. Either can break the carousel:

```js
// Don't change the type because the option is not responsive
splide.options = { type: 'fade' };

// Don't directly set a value
splide.options.perPage = 4;
```

**(source)** Responsive options (`ResponsiveOptions` in the types): `label`, `labelledby`, `speed`, `rewind`, `rewindSpeed`, `rewindByDrag`, `width`, `height`, `fixedWidth`, `fixedHeight`, `heightRatio`, `perPage`, `perMove`, `clones`, `cloneStatus`, `focus`, `gap`, `padding`, `arrows`, `pagination`, `paginationKeyboard`, `paginationDirection`, `easing`, `easingFunc`, `drag`, `snap`, `dragMinThreshold`, `flickPower`, `flickMaxPages`, `destroy`.

From source:

- The setter calls `Components.Media.set( options, true, true )`. It merges into the current options and into the base options, so the change survives later breakpoint switches, and it always emits `updated`.
- The setter throws a `TypeError` before `mount()`, because `Components.Media` does not exist yet.
- `splide.options` is an object whose prototype holds the base options and whose own properties hold the active breakpoint overrides. Reading a property (`splide.options.perPage`) works, but `Object.keys()` or `JSON.stringify()` shows only the overrides.

## Instance methods

All methods except `is()` return `this`, so calls can be chained: `new Splide( '.splide' ).on( 'ready', fn ).mount()`.

### `mount()`

`mount( Extensions?: Record<string, ComponentConstructor>, Transition?: ComponentConstructor ): this`

Initializes the instance. You can pass extensions or a custom transition component.

| Param | Description |
|---|---|
| `Extensions` | Optional. An object with extensions, `{ Name: ComponentConstructor }` |
| `Transition` | Optional. A Transition component |

Return: `this`

From source:

- Throws `Error: [splide] Already mounted!` unless the state is `CREATED` or `DESTROYED`. A destroyed instance can be mounted again.
- `Extensions` and `Transition` are remembered. A later `mount()` without arguments, including the automatic remount after a breakpoint `destroy: true`, reuses them.
- A `ComponentConstructor` is a function `( Splide, Components, options ) => { setup?(), mount?(), destroy?( completely ) }`. Each one is available as `splide.Components[ Name ]`.
- Order: every component is constructed and its `setup()` called (built-ins first, then extensions, then `Transition`). Then every `mount()` runs. Then `mounted` is emitted, the root gets `is-initialized`, the state becomes `IDLE`, and `ready` is emitted.
- The default transition is the Fade component when `type` is `'fade'`, otherwise the Slide component.

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  // A minimal extension, written against the global build
  function Logger( splide, Components, options ) {
    return {
      mount: function () {
        splide.on( 'moved', function ( index ) {
          console.log( 'Now at', index, 'of', splide.length );
        } );
      },
    };
  }

  new Splide( '.splide' ).mount( { Logger: Logger } );
</script>
```

### `sync()`

`sync( splide: Splide ): this`

Syncs the carousel with the given one. This method must be called before `mount()`.

```js
var primary   = new Splide( '#primary' );
var secondary = new Splide( '#secondary' );

primary.sync( secondary );
primary.mount();
secondary.mount();
```

| Param | Description |
|---|---|
| `splide` | A Splide instance to sync with |

Return: `this`

From source:

- It pushes `{ splide }` to `this.splides` and `{ splide: this, isParent: true }` to `splide.splides`.
- If it is called while this instance is already `IDLE` (mounted), it calls `Components.Sync.remount()` on both instances. The other instance must also be mounted, or this throws.
- Syncing works both ways. On `move`, each instance calls `target.go( target.is( 'loop' ) ? destIndex : newIndex )` on the other.
- **Mount the carousel without `isNavigation` before the `isNavigation` one.** A navigation carousel reads the other instance's `Components.Slides` while it mounts, to build `aria-controls`, and throws a `TypeError` if that instance is not mounted yet. Mounting `primary` before `secondary` in the example above follows this rule.
- A carousel can sync with more than one other carousel by calling `sync()` once per target.

### `go()`

`go( control: number | string ): this`

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

```js
var splide = new Splide( '.splide' );
splide.mount();

// Goes to the slide 1:
splide.go( 1 );

// Increments the index:
splide.go( '+2' );

// Goes to the next page:
splide.go( '>' );

// Goes to the page 2:
splide.go( '>2' );
```

| Param | Description |
|---|---|
| `control` | A control pattern |

Return: `this`

More from source:

- The pattern regex is `/([+\-<>])(\d+)?/` and is not anchored: the first `+`, `-`, `<` or `>` anywhere in the string is used.
- Out-of-range targets for `'+N'`, `'-N'`, `'>'` and `'<'`: if the current index is not yet at the first or last reachable index, and `perMove` is not set, Splide stops at that edge first. From the edge, `type: 'loop'` wraps, `rewind: true` jumps to the other end, and a plain `slide` carousel does nothing.
- `go()` does nothing when:
  - `waitForTransition: true` and the carousel is `MOVING` or `SCROLLING`
  - the target resolves to `-1` (no slide to go to)
  - the target equals the current index (the public `go()` passes `allowSameIndex = false`)
  - for `'+'`, `'-'`, `'>'` and `'<'`: there are not enough slides (Slide components, clones included, `<= perPage`) and no `focus`/`isNavigation`. A number target is clamped to the end index, which is then `0`, so it does nothing either
- Calling `go()` before `mount()` throws a `TypeError`.
- `go()` takes no callback. For one that runs when the transition ends, call the component directly: `splide.Components.Controller.go( control, allowSameIndex?, callback? )`. Or listen for `moved`.

### `on()`

`on( events: string | string[], callback: AnyFunction ): this`

Registers an event handler.

```js
var splide = new Splide();

// Listens to a single event:
splide.on( 'move', function () {} );

// Listens to multiple events:
splide.on( 'move resize', function () {} );

// Appends a namespace:
splide.on( 'move.myNamespace resize.myNamespace', function () {} );
```

| Param | Description |
|---|---|
| `events` | An event name, names separated by spaces, or **(source)** an array of names. Use a dot (`.`) to append a namespace |
| `callback` | A callback function. It receives the event's arguments (see [events.md](events.md)) |

Return: `this`

**(source)** The callback's `this` is the callback function itself, not the Splide instance, so use a closure variable. Registering the same function twice creates two handlers. Full details are in [events.md](events.md).

### `off()`

`off( events: string | string[] ): this`

Removes all registered handlers for the given event or events. To remove only particular handlers, identify them with a namespace.

```js
var splide = new Splide( '.splide' );

// Removes all handlers assigned to "move":
splide.off( 'move' );

// Only removes handlers that belong to the specified namespace:
splide.off( 'move.myNamespace' );
```

| Param | Description |
|---|---|
| `events` | An event name or names separated by spaces. Use a dot (`.`) to append a namespace |

Return: `this`

**(source)** `off( 'move' )` removes only the `move` handlers registered **without** a namespace. Handlers registered as `'move.myNamespace'` survive it and need `off( 'move.myNamespace' )`. There is no way to remove a single handler by function reference, and `'.myNamespace'` alone matches nothing.

### `emit()`

`emit( event: string, ...args: any[] ): this`

Emits an event and triggers the registered handlers.

| Param | Description |
|---|---|
| `event` | An event name to emit |
| `args` | Optional. Any number of arguments to pass to handlers |

Return: `this`

```js
splide.on( 'myplugin:opened', function ( id, data ) { /* ... */ } );
splide.emit( 'myplugin:opened', 'slide-3', { foo: 1 } );
```

**(source)** Handlers run synchronously, in registration order. Emitting a built-in event name, such as `refresh`, triggers the internal components that listen to it, too.

### `add()`

`add( slides: string | HTMLElement | Array<string | HTMLElement>, index?: number ): this`

Inserts a slide or slides at the given position.

```js
var splide = new Splide( '.splide' );
splide.mount();

// Adds a slide by HTML:
splide.add( '<li class="splide__slide"></li>' );

// or adds an element:
var slide = document.createElement( 'li' );
slide.classList.add( 'splide__slide' );
splide.add( slide );
```

| Param | Description |
|---|---|
| `slides` | A slide element, an HTML string for a slide, or an array of either |
| `index` | Optional. The index to insert the slide at. **(source)** Omitted or out of range means append |

Return: `this`

Do not call this method while the carousel is moving: it refreshes and repositions the carousel.

From source:

- An HTML string is parsed with `DOMParser`, and only its **first** element is used. Pass an array to add several slides.
- The `splide__slide` class (`classes.slide`) is added automatically.
- It emits `refresh`. It also emits `resize` once all `<img>` elements in the added slide have loaded or errored, or immediately if there are none.

### `remove()`

`remove( matcher: SlideMatcher ): this`

Removes the slides that match the matcher. `SlideMatcher = number | number[] | string | SlidesPredicate`: an index, an array of indices, a selector, or a predicate function that takes these arguments:

| Argument | Description |
|---|---|
| `Slide` | The `Slide` component object being processed |
| `index` | The index of the current item |
| `Slides` | The array of `Slide` objects being iterated over |

```js
var splide = new Splide( '.splide' );
splide.mount();

// Removes the slide at 0:
splide.remove( 0 );

// Removes slides at 0 and 1:
splide.remove( [ 0, 1 ] );

// Removes slides by a selector:
splide.remove( '.is-visible' );

// Removes slides by a predicate function:
splide.remove( function ( Slide ) { return Slide.index % 2 === 0; } );
```

| Param | Description |
|---|---|
| `matcher` | An index, an array of indices, a selector string, or a predicate function |

Return: `this`

Do not call this method while the carousel is moving: it refreshes and repositions the carousel.

**(source)** The matcher runs over every Slide component, clones included. Numbers match `Slide.index`, and clone indices are negative or `>= length`, so numbers only hit real slides. Selectors and predicates can also match clones, which is harmless because clones are rebuilt on the `refresh` that follows. The predicate's `index` argument is the array position, not `Slide.index`.

### `is()`

`is( type: string ): boolean`

Checks the carousel type.

```js
var splide = new Splide( '.splide' );
splide.mount();

if ( splide.is( 'loop' ) ) {
  // do something
}
```

| Param | Description |
|---|---|
| `type` | A type to test: `'slide'`, `'loop'` or `'fade'` |

Return: `true` if the type matches the current one, otherwise `false`. **(source)** It compares `options.type === type`, and works before `mount()`.

### `refresh()`

`refresh(): this`

Refreshes the carousel. Most components reinitialize themselves.

Return: `this`

**(source)** It only emits `refresh`. In response, Elements re-collects the slides from the DOM (direct `.splide__slide` children of the list, clones excluded), Slides re-registers them (new IDs, labels and roles), Clones regenerates the clones, Pagination rebuilds, Arrows update, Layout re-measures and Move repositions. So `refresh()` also picks up slides you inserted into or removed from the list with plain DOM methods.

### `destroy()`

`destroy( completely?: boolean ): this`

Destroys the carousel.

| Param | Description |
|---|---|
| `completely` | Optional. If `true`, Splide will not remount the carousel through breakpoints. **(source)** Defaults to **`true`** |

Return: `this`

From source:

- `splide.destroy()` is a complete destroy. Pass `false` to keep the breakpoint listeners, so a breakpoint without `destroy` can remount it.
- Breakpoint `destroy: true` calls `destroy( false )`. Breakpoint `destroy: 'completely'` calls `destroy( true )`.
- Called while the state is still `CREATED` (before mount, or during mount), the destroy is postponed until `ready`.
- Sequence: each component's `destroy( completely )` runs in reverse order (generated arrows, pagination and clones are removed, attributes and classes are cleaned), then `destroy` is emitted, then **all** handlers are removed (including ones from `on()`), then `completely` empties `this.splides`, and the state becomes `DESTROYED`.
- A complete destroy also removes `role`, `aria-label` and `aria-labelledby` from the root, even when they came from HTML (see [accessibility.md](accessibility.md)).
- A destroyed instance can be mounted again with `mount()`. Handlers registered with `on()` before the destroy are gone and must be registered again (see [events.md](events.md)).

## State machine

From source (`core/Splide/Splide.ts`, `Move.ts`, `Scroll.ts`, `Drag.ts`):

| Transition | Where |
|---|---|
| (constructor) to `CREATED` (1) | `new Splide()` |
| `CREATED` / `DESTROYED` to `CREATED` | Start of `mount()` |
| `CREATED` to `IDLE` (3) | End of `mount()`, between `mounted` and `ready` |
| `IDLE` to `MOVING` (4) | `Move.move()`, right before `move` is emitted |
| `MOVING` to `IDLE` | Transition end, right before `moved` |
| `IDLE` to `SCROLLING` (5) | `Scroll.scroll()`, right before `scroll` |
| `SCROLLING` to `IDLE` | Scroll end, right before `scrolled` |
| any to `DRAGGING` (6) | First pointer move after a pointer down on the track, right before `drag` |
| `DRAGGING` to `IDLE` | Pointer up, right before `dragged` (the release move then sets `MOVING` or `SCROLLING`) |
| any to `DESTROYED` (7) | End of `destroy()` |

`MOUNTED` (2) is defined but never used. `state.is()` accepts a number or an array: `splide.state.is( [ Splide.STATES.MOVING, Splide.STATES.SCROLLING ] )`.

## What works before mount()

From source:

| Works before `mount()` | Throws (`TypeError`) before `mount()` |
|---|---|
| `on()`, `off()`, `emit()`, `sync()`, `is()`, reading `options`, `root`, `state`, `splides` | `go()`, `add()`, `remove()`, `length`, `index`, setting `options` |
| `destroy()` (postponed until `ready`) | Using `Components.*`, which is an empty object until mounted |

`refresh()` only emits an event, so before mount it triggers just your own `refresh` handlers.

## Components reference

From `dist/types/index.d.ts` (4.1.4). Every component may also have `setup()`, `mount()` and `destroy( completely? )`. These are internal APIs; the docs cover only some of them on separate component pages.

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
| `<extension name>` | Whatever your extension returns |

`SlideComponent` (returned by `Slides.get()`/`getAt()` and passed to many events): `index`, `slideIndex` (the original index for a clone, `-1` otherwise), `slide` (element), `container` (`.splide__slide__container` or `undefined`), `isClone`, `update()`, `style( prop, value, useContainer? )`, `isWithin( from, distance ): boolean`.

## Docs vs source

| Topic | Docs say | Source says |
|---|---|---|
| `destroy( completely? )` default | Optional, default not stated (reads as `false`) | `completely = true`: `destroy()` is a complete destroy |
| `STATES` | `CREATED`, `MOUNTED`, `IDLE`, `MOVING`, `DESTROYED` | Also `SCROLLING` (5) and `DRAGGING` (6). Values are 1 to 7. `MOUNTED` is never set |
| `sync()` timing | Must be called before `mount()` | Also accepted after mount when this instance is `IDLE` and the other is mounted (both Sync components remount) |
| `off( 'move' )` | "Removes the registered all handlers" for the event | Removes only un-namespaced handlers for that event |
| `go( '<${i}' )` | Not listed | Not supported: same as `'<'` |
| `'>${i}'` page numbering | "Goes to the page `i`" | Zero-based page index |
