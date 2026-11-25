# Events

Source: https://splidejs.com/guides/events/

Splide v4.1.4 from jsDelivr, no build step. The event list is the docs list plus every constant in `constants/events.ts`. Arguments come from the emit calls in `components/*` and `core/Splide/Splide.ts`, cross-checked with the `EventMap` in the 4.1.4 `dist/types/index.d.ts`. Facts marked **(source)** are not on the docs page.

## Contents

- [Listening to events](#listening-to-events)
- [Multiple events, arrays and namespaces](#multiple-events-arrays-and-namespaces)
- [Removing handlers with `off()`](#removing-handlers-with-off)
- [Emitting custom events](#emitting-custom-events)
- [Handlers that must be registered before `mount()`](#handlers-that-must-be-registered-before-mount)
- [Handler removal on destroy, and the breakpoint `destroy` caveat](#handler-removal-on-destroy-and-the-breakpoint-destroy-caveat)
- [Event list](#event-list)
- [Event details](#event-details)
- [Internal events (from source)](#internal-events-from-source)
- [Argument types](#argument-types)
- [Event constants (ESM build)](#event-constants-esm-build)
- [Event sequences](#event-sequences)
- [Docs vs source](#docs-vs-source)

## Listening to events

Use `Splide#on()` to listen to events:

```js
var splide = new Splide( '.splide' ).mount();

splide.on( 'move', function () {
  // do something
} );
```

Signature: `on( events: string | string[], callback: Function ): this`. It returns the instance, so it can be chained: `new Splide( '.splide' ).on( 'ready', fn ).mount()`.

From source (`constructors/EventInterface`, `constructors/EventBinder`):

- Events are `CustomEvent`s dispatched on a private `DocumentFragment` (`splide.event.bus`), not on the root element. `splide.root.addEventListener( 'move', ... )` does **not** work. Use `splide.on()`.
- The callback receives the event's arguments spread out, as in `function ( newIndex, prevIndex, destIndex )`. It does not receive an `Event` object.
- Inside the callback, `this` is the callback function itself, not the Splide instance. Use a closure variable such as `splide`.
- Handlers run **synchronously**, in registration order. Internal components register their own handlers during `mount()`. A handler you register after `mount()` usually runs after the components have reacted (for example, slide classes already updated on `moved`), and one registered before `mount()` runs before them. `refresh()`, `add()` and `remove()` re-register the slide components, which changes that order, so do not depend on it. Use the event arguments, or `splide.index`, instead of reading DOM classes.
- Registering the same function twice creates two handlers. There is no de-duplication.

## Multiple events, arrays and namespaces

```js
var splide = new Splide( '.splide' );

// Listens to a single event:
splide.on( 'move', function () {} );

// Listens to multiple events:
splide.on( 'move resize', function () {} );

// Appends a namespace:
splide.on( 'move.myNamespace resize.myNamespace', function () {} );

// (source) An array works too; its items are joined with spaces:
splide.on( [ 'moved', 'scrolled' ], function () {} );
```

- Separate several names with spaces. One callback is registered for each name.
- `name.namespace` adds a namespace. It only matters for `off()`: a namespaced handler still fires on the plain event (`move.myNamespace` fires on every `move`).
- **(source)** Only the first dot counts. `'move.a.b'` has the namespace `a`.

## Removing handlers with `off()`

`off( events: string | string[] ): this` removes the registered handlers for the given event(s). To remove only particular handlers, identify them with a namespace:

```js
var splide = new Splide( '.splide' );

// Removes all handlers assigned to "move":
splide.off( 'move' );

// Only removes handlers that belong to the specified namespace:
splide.off( 'move.myNamespace' );
```

**(source)** Matching is exact on both the event name and the namespace:

| Call | Removes |
|---|---|
| `off( 'move' )` | `move` handlers registered **without** a namespace. `move.myNamespace` handlers stay |
| `off( 'move.myNamespace' )` | Only `move` handlers in `myNamespace` |
| `off( 'move.ns resize.ns' )` | `move` and `resize` handlers in `ns` |
| `off( '.ns' )` | Nothing. Name each event |

You cannot remove one specific function: `off()` takes no callback, and the internal binder stores a wrapper rather than your function. Give the handler a unique namespace instead:

```js
splide.on( 'moved.counter', updateCounter );
// later
splide.off( 'moved.counter' );
```

## Emitting custom events

`emit( event: string, ...args: any[] ): this` triggers every handler for that name and passes `args` to them. Any string works as a name, so extensions and page code can use the bus:

```js
splide.on( 'gallery:open', function ( slideIndex, source ) {
  console.log( 'open', slideIndex, source );
} );

splide.emit( 'gallery:open', 2, 'button' );
```

**(source)** Emitting a built-in name (for example `splide.emit( 'refresh' )`) also triggers the internal components that listen to it. `splide.refresh()` is literally `emit( 'refresh' )`.

## Handlers that must be registered before `mount()`

These events fire inside `mount()`. A handler added after `mount()` misses them.

| Event | Why |
|---|---|
| `mounted` | Emitted once, at the end of `mount()` (docs) |
| `arrows:mounted` | Emitted while the Arrows component mounts (docs) |
| `pagination:mounted` | Emitted while the Pagination component mounts (docs) |
| `navigation:mounted` | Emitted while the Sync component mounts, when `isNavigation: true` (docs) |
| `ready` | **(source)** Emitted at the end of `mount()`, after `mounted` |
| `overflow` | **(source)** Always emitted once during mount with the initial state, and again on every change. The Overflow guide also says to register it before `mount()` |
| `resized` | **(source)** Emitted once while Layout mounts |
| `active`, `visible` | **(source)** Emitted for the initial active and visible slides during `mounted` |
| `arrows:updated`, `pagination:updated` | **(source)** First emitted during mount, before the matching `*:mounted` |
| `autoplay:play` | **(source)** Emitted during mount if autoplay starts immediately |

```js
var splide = new Splide( '.splide' ).mount();

splide.on( 'mounted', function () {
  // This won't be executed.
} );

var splide2 = new Splide( '.splide' );

splide2.on( 'mounted', function () {
  // This will be executed.
} );

splide2.mount();
```

(The docs example reuses one variable name. It is split here for clarity.)

## Handler removal on destroy, and the breakpoint `destroy` caveat

Splide removes **all** handlers on destroy. If you use the `destroy` option in breakpoints, the carousel is destroyed and later remounted, and the handlers you added with `on()` do not come back. You have to detect the remount and register them again by writing your own extension.

From source (`Splide.destroy()`, `Media.update()`):

1. Breakpoint `destroy: true` calls `splide.destroy( false )`: components are destroyed, `destroy` is emitted (your handler still runs), then every handler on the bus is removed. Media query listeners are kept.
2. When the breakpoint stops matching, Media calls `splide.mount()` again. `mount()` reuses the extensions passed the first time, so an extension's `mount()` runs again and can register its handlers again.
3. Breakpoint `destroy: 'completely'`, or `splide.destroy()` / `destroy( true )`, also removes the media query listeners, so nothing remounts.
4. DOM listeners you added yourself with `addEventListener` are **not** removed. Remove them in the extension's `destroy()`, or use `EventInterface().bind()` (ESM), which is removed automatically.

**Global-build version** (no imports; `splide.on()` inside an extension):

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function MyHandlers( splide, Components, options ) {
    var counter = document.querySelector( '#counter' );

    function onResize() { /* page-level listener */ }

    return {
      mount: function () {
        // Runs on the first mount and on every breakpoint remount.
        splide.on( 'moved', function ( newIndex ) {
          counter.textContent = ( newIndex + 1 ) + ' / ' + splide.length;
        } );
        window.addEventListener( 'resize', onResize );
      },
      destroy: function ( completely ) {
        // splide.on() handlers are removed by Splide; your own DOM listeners are not.
        window.removeEventListener( 'resize', onResize );
      },
    };
  }

  new Splide( '.splide', {
    breakpoints: {
      640: { destroy: true }, // destroyed below 640px, remounted above it
    },
  } ).mount( { MyHandlers: MyHandlers } );
</script>
```

**ESM version with `EventInterface`** (the docs' extension style; needs `type="module"`):

```html
<script type="module">
  import { Splide, EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function MyHandlers( splide, Components, options ) {
    const { on, bind } = EventInterface( splide ); // everything here is removed on destroy

    return {
      mount() {
        on( 'moved', ( newIndex ) => console.log( 'moved to', newIndex ) );
        bind( window, 'resize', () => console.log( 'resize' ) );
      },
    };
  }

  new Splide( '.splide', { breakpoints: { 640: { destroy: true } } } ).mount( { MyHandlers } );
</script>
```

## Event list

| Event | Arguments | Fired |
|---|---|---|
| `mounted` | none | Right after all components are mounted |
| `ready` | none | Right after `mounted`, once the state is `IDLE` |
| `click` | `Slide: SlideComponent`, `e: MouseEvent` | When the user clicks any slide |
| `move` | `newIndex: number`, `prevIndex: number`, `destIndex: number` | Right before the carousel moves |
| `moved` | `newIndex: number`, `prevIndex: number`, `destIndex: number` | Right after the carousel moves |
| `active` | `Slide: SlideComponent` | When a slide becomes active |
| `inactive` | `Slide: SlideComponent` | When the active slide becomes inactive |
| `visible` | `Slide: SlideComponent` | When a slide becomes visible in the viewport |
| `hidden` | `Slide: SlideComponent` | When a slide becomes hidden from the viewport |
| `refresh` | none | When Splide refreshes the carousel |
| `updated` | `options: Options` | Whenever options are updated |
| `resize` | none | Whenever the window is resized |
| `resized` | none | Whenever Splide finishes resizing the carousel |
| `drag` | none | When the user starts dragging the carousel |
| `dragging` | none | While the user is dragging the carousel |
| `dragged` | none | Right after the user finishes dragging the carousel |
| `scroll` | none | Right before Splide scrolls the carousel (Scroll component) |
| `scrolled` | none | Right after Splide finishes scrolling the carousel (Scroll component) |
| `overflow` | `isOverflow: boolean` **(source)** | When the total width of the slides becomes wider or narrower than the list element |
| `destroy` | none | When Splide destroys the carousel |
| `arrows:mounted` | `prev: HTMLButtonElement`, `next: HTMLButtonElement` | Right after the arrows are mounted |
| `arrows:updated` | `prev: HTMLButtonElement`, `next: HTMLButtonElement`, `prevIndex: number`, `nextIndex: number` | Whenever the arrows' status is updated |
| `pagination:mounted` | `data: PaginationData`, `item: PaginationItem` | Right after the pagination is mounted |
| `pagination:updated` | `data: PaginationData`, `prev: PaginationItem`, `curr: PaginationItem` | Whenever the pagination's status is updated |
| `navigation:mounted` | `splides` (docs: `Splide[]`; source: `SyncTarget[]`) | Right after a navigation carousel (`isNavigation: true`) is mounted |
| `autoplay:play` | none | When autoplay starts |
| `autoplay:pause` | none | Right after autoplay is paused |
| `autoplay:playing` | `rate: number` (0 to 1) | While the autoplay timer is ticking |
| `lazyload:loaded` | `img: HTMLImageElement`, `Slide: SlideComponent` | Right after the LazyLoad component loads an image |
| `sk` | `Slide: SlideComponent`, `e: KeyboardEvent` | **Internal, from source.** A key is pressed on a slide |
| `sh` | none | **Internal, from source.** A loop carousel's position wrapped across the clones during drag or scroll |
| `ei` | none | **Internal, from source.** The end index changed after a resize |

## Event details

### `mounted`

Fired right after all components are mounted. Register the handler before calling `mount()` (example above).

**(source)** It fires before the root gets `is-initialized` and before the state becomes `IDLE`. Handlers registered before `mount()` also run before the Move component's first reposition, so slides do not have `is-active`/`is-visible` yet. Use `ready` when you need the finished DOM.

### `ready`

Fired right after `mounted`, by which point the state has changed to `IDLE`.

**(source)** Also emitted inside `mount()`, so register it before `mount()`. After a breakpoint remount, both `mounted` and `ready` fire again (only for handlers registered again by an extension).

### `click`

Fired when the user clicks any slide.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The clicked Slide component |
| `e` | `MouseEvent` | The `MouseEvent` object |

**(source)**

- Clones fire it too. In loop mode, use `Slide.isClone ? Slide.slideIndex : Slide.index` to get the real index.
- It does not fire for the click that ends a drag: Drag cancels that click in the capture phase on the track.
- Splide does not call `preventDefault()`, so links inside slides still navigate.

```js
splide.on( 'click', function ( Slide, e ) {
  var realIndex = Slide.isClone ? Slide.slideIndex : Slide.index;
  openLightbox( realIndex );
} );
```

### `move`

Fired right before the carousel moves.

| Argument | Type | Description |
|---|---|---|
| `newIndex` | `number` | The new slide index |
| `prevIndex` | `number` | The previous slide index |
| `destIndex` | `number` | The destination slide index. In loop mode it can be negative or greater than the number of slides, because of the clones |

**(source)** Emitted by `Move.move()` after the state becomes `MOVING`. It is not emitted when `go()` is ignored: same index, no slide to go to, or busy with `waitForTransition`. Free-drag (`drag: 'free'`) scrolling emits `scroll`/`scrolled` instead.

```js
var counter = document.querySelector( '#counter' );

splide.on( 'move', function ( newIndex ) {
  counter.textContent = ( newIndex + 1 ) + ' / ' + splide.length;
} );
```

### `moved`

Fired right after the carousel moves. Same arguments as `move`.

| Argument | Type | Description |
|---|---|---|
| `newIndex` | `number` | The new slide index |
| `prevIndex` | `number` | The previous slide index |
| `destIndex` | `number` | The destination slide index. In loop mode it can be negative or greater than the number of slides, because of the clones |

**(source)** Fired when the CSS transition ends (`transitionend` on the list). If the speed is `0`, for example under `prefers-reduced-motion`, or the carousel is already in position, the transition is skipped and `moved` fires **synchronously** right after `move`. With `useScroll: true`, `moved` fires just before `scrolled`.

### `active`

Fired when a slide becomes active.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes active |

**(source)** Fired when the slide gains `is-active`: on `moved`, or on `move` with `updateOnMove: true`. Also fired on repositioning (`mounted`, `resized`, `updated`, `refresh`). In loop mode with `cloneStatus: true` (the default), clones of the active slide become active too, so the event can fire more than once per move. Filter with `if ( Slide.isClone ) return;`.

### `inactive`

Fired when the active slide becomes inactive.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes inactive |

**(source)** Clones fire it too, as for `active`.

### `visible`

Fired when a slide becomes visible in the viewport.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes visible |

**(source)** "Visible" means completely inside the track (whole-pixel rounding). For `type: 'fade'` it means active. The event fires when the slide gains the `is-visible` class: on `moved`, `scrolled`, the internal `sh`, and repositioning. Clones fire it too. The initial visible slides fire during `mount()`.

### `hidden`

Fired when a slide becomes hidden from the viewport.

| Argument | Type | Description |
|---|---|---|
| `Slide` | `SlideComponent` | The Slide component that becomes hidden |

**(source)** Fired when a slide loses `is-visible`. A slide that was never visible does not fire `hidden` on mount.

```js
// Pause videos that scroll out of view
splide.on( 'hidden', function ( Slide ) {
  var video = Slide.slide.querySelector( 'video' );
  if ( video ) video.pause();
} );
```

### `refresh`

Fired when Splide refreshes the carousel.

**(source)** Emitted by `splide.refresh()`, by `add()` and `remove()`, by Clones when a loop carousel needs more clones (or none), and by Media when a breakpoint changes `direction`. Components rebuild in response (see [apis.md](apis.md#refresh)).

### `updated`

Fired whenever options are updated, for example when a breakpoint starts or stops matching.

| Argument | Type | Description |
|---|---|---|
| `options` | `Options` | An object with the latest options |

**(source)** Emitted by `Media.set()`: on breakpoint and `prefers-reduced-motion` media-query changes, and on every `splide.options = {...}` assignment. It is not emitted while the instance is being created or mounted (state `CREATED`). On a breakpoint that destroys the carousel, `updated` fires just before `destroy`.

```js
splide.on( 'updated', function ( options ) {
  console.log( 'perPage is now', options.perPage );
} );
```

### `resize`

Fired whenever the window is resized. Splide reduces the frequency with `requestAnimationFrame`.

**(source)** Also emitted on the window `load` event, after each image that LazyLoad loads, and after all images in a slide added with `add()` finish loading. Layout reacts by re-measuring and emits `resized` only if the root's size changed.

### `resized`

Fired whenever Splide finishes resizing the carousel.

**(source)** Emitted by `Layout.resize()` when the root's width or height changed, and always on Layout init: mount, `updated`, `refresh`. Move repositions and Controller re-checks the end index in response.

### `drag`

Fired when the user starts dragging the carousel.

**(source)** Fired on the first pointer move after a pointer down on a draggable part of the track. At that point Splide may not yet have decided the gesture is a drag, for example a vertical page scroll on touch. The state becomes `DRAGGING`.

### `dragging`

Fired while the user is dragging the carousel.

**(source)** Fired on each pointer move once the drag threshold (`dragMinThreshold`) is passed.

### `dragged`

Fired right after the user finishes dragging the carousel.

**(source)** Fired on pointer up if `drag` fired. The state returns to `IDLE`. If the drag threshold was passed, the release then triggers a `go()` (`move`/`moved`) or, with `drag: 'free'`, a scroll (`scroll`/`scrolled`).

### `scroll`

Fired right before Splide scrolls the carousel with the Scroll component. It is not fired for the normal CSS transition; use `move` instead.

**(source)** The Scroll component is used by `drag: 'free'`, by `useScroll: true`, by the bounce-back after over-dragging, and by `Components.Controller.scroll()`. The state becomes `SCROLLING`.

### `scrolled`

Fired right after the Scroll component finishes scrolling the carousel. It is not fired for the normal CSS transition; use `moved` instead.

### `overflow`

Fired when the total width of the slides becomes wider or narrower than the list element as the viewport is resized (v4.1.0+).

| Argument | Type | Description |
|---|---|---|
| `isOverflow` | `boolean` | **(source; the Overflow guide shows it, the Events page does not)** `true` if the slides are wider than the list (enough slides), `false` if not |

**(source)** The root's `is-overflow` class is toggled at the same moment. The event fires once during mount with the initial value, so register it before `mount()`.

```js
var splide = new Splide( '.splide' );

splide.on( 'overflow', function ( isOverflow ) {
  splide.options = {
    arrows    : isOverflow,
    pagination: isOverflow,
    drag      : isOverflow,
  };
} );

splide.mount();
```

For a loop carousel (from the Overflow guide), also reset the position and toggle the clones:

```js
var splide = new Splide( '.splide', { type: 'loop' } );

splide.on( 'overflow', function ( isOverflow ) {
  // Reset the carousel position
  splide.go( 0 );

  splide.options = {
    arrows    : isOverflow,
    pagination: isOverflow,
    drag      : isOverflow,
    clones    : isOverflow ? undefined : 0, // Toggle clones
  };
} );

splide.mount();
```

(The guide writes `new Splide( { type: 'loop' } )` without a target, which throws. A selector is added here.)

### `destroy`

Fired when Splide destroys the carousel.

**(source)** Fired after every component's `destroy()` has run (generated arrows, pagination and clones are already removed) and **before** the handlers are removed, so your `destroy` handler still runs. It fires for breakpoint destroys too.

### `arrows:mounted`

Fired right after the arrows are mounted. Register the listener before calling `mount()`.

| Argument | Type | Description |
|---|---|---|
| `prev` | `HTMLButtonElement` | The previous arrow |
| `next` | `HTMLButtonElement` | The next arrow |

**(source)** Fires only when `arrows` is `true`. It fires again after every `updated`, because Arrows remount then.

### `arrows:updated`

Fired whenever the arrows' status is updated.

| Argument | Type | Description |
|---|---|---|
| `prev` | `HTMLButtonElement` | The previous arrow |
| `next` | `HTMLButtonElement` | The next arrow |
| `prevIndex` | `number` | Docs: "An old index". **(source)** The slide index the previous arrow goes to (`Controller.getPrev()`), `-1` if none (arrow disabled) |
| `nextIndex` | `number` | Docs: "A new index". **(source)** The slide index the next arrow goes to (`Controller.getNext()`), `-1` if none (arrow disabled) |

**(source)** Emitted on `mounted`, `moved`, `refresh`, `scrolled` and `ei`, and while the arrows mount.

```js
// Hide arrows instead of disabling them
splide.on( 'arrows:updated', function ( prev, next, prevIndex, nextIndex ) {
  prev.hidden = prevIndex < 0;
  next.hidden = nextIndex < 0;
} );
```

### `pagination:mounted`

Fired right after the pagination is mounted. Register the listener before calling `mount()`.

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | An object with pagination data |
| `item` | `PaginationItem` | Data of the active item |

**(source)** Fires only when `pagination` is `true`. The pagination is rebuilt, and this event fires again, on every `updated`, `refresh` and `ei`, so changes made here survive rebuilds.

```js
var splide = new Splide( '.splide' );

splide.on( 'pagination:mounted', function ( data ) {
  data.list.classList.add( 'my-pagination' );
  data.items.forEach( function ( item ) {
    item.button.textContent = String( item.page + 1 ); // visible page numbers
  } );
} );

splide.mount();
```

### `pagination:updated`

Fired whenever the pagination's status is updated.

| Argument | Type | Description |
|---|---|---|
| `data` | `PaginationData` | An object with pagination data |
| `prev` | `PaginationItem` | Data of the previous item |
| `curr` | `PaginationItem` | Data of the current item |

**(source)** Emitted on `move`, `scroll` and `scrolled` (so it fires before the transition ends), and once while mounting. On the first call `prev` and `curr` are the same item. Either can be `undefined` in edge cases, and the source guards for that.

### `navigation:mounted`

Fired right after the navigation carousel (`isNavigation: true`) is mounted. Register the listener before calling `mount()`.

| Argument | Type | Description |
|---|---|---|
| `splides` | Docs and types: `Splide[]` | Docs: "An array with Splide instances to navigate". **(source)** `Splide.splides`, which is an array of `SyncTarget` objects (`{ splide: Splide, isParent?: boolean }`), not bare instances |

```js
thumbnails.on( 'navigation:mounted', function ( splides ) {
  splides.forEach( function ( target ) {
    console.log( target.splide.root.id, target.isParent ? '(parent)' : '' );
  } );
} );
```

### `autoplay:play`

Fired when autoplay starts.

**(source)** Fired by `Autoplay.play()` only when autoplay was paused and there are enough slides: at mount (unless `autoplay: 'pause'` or reduced motion), when hover or focus ends, on the toggle button, and on `Components.Autoplay.play()`. The live region switches to `aria-live="off"`.

### `autoplay:pause`

Fired right after autoplay is paused.

**(source)** Fired only if autoplay was running: on hover (`pauseOnHover`), on focus (`pauseOnFocus`), on the toggle button, and on `Components.Autoplay.pause()`. The live region switches back to `aria-live="polite"`.

### `autoplay:playing`

Fired while the autoplay timer is ticking.

| Argument | Type | Description |
|---|---|---|
| `rate` | `number` | The current progress, from 0 to 1 |

**(source)** Fired on every animation frame while autoplay runs. Splide also sets `.splide__progress__bar`'s `width` to `rate * 100%` if the bar exists.

```js
var bar = document.querySelector( '.my-progress' );

splide.on( 'autoplay:playing', function ( rate ) {
  bar.style.width = String( 100 * rate ) + '%';
} );
```

### `lazyload:loaded`

Fired right after the LazyLoad component loads an image.

| Argument | Type | Description |
|---|---|---|
| `img` | `HTMLImageElement` | The loaded image element |
| `Slide` | `SlideComponent` | The Slide component that contains the image |

**(source)** Not fired on image `error` (only the `is-loading` class is removed). Each load is followed by a `resize` emit.

## Internal events (from source)

These are defined after an `/** @internal */` marker in `constants/events.ts`. The docs do not list them and they may change. The string values are short codes, not descriptive names.

| ESM constant | String | Arguments | Emitted by | Used by |
|---|---|---|---|---|
| `EVENT_SLIDE_KEYDOWN` | `'sk'` | `Slide: SlideComponent`, `e: KeyboardEvent` | `keydown` on each slide element, clones included | Sync: Space/Enter on navigation slides |
| `EVENT_SHIFTED` | `'sh'` | none | `Move.translate()` without `preventLoop`, when a loop carousel's position wraps past the clones. In practice this happens while dragging, during Scroll-component scrolling (free drag, `useScroll`), and on `jump()`. A normal `go()` transition does not emit it | Slides (status update) |
| `EVENT_END_INDEX_CHANGED` | `'ei'` | none | Controller, on `resized`, when the last reachable index changed (e.g. `autoWidth`/`fixedWidth`) | Controller, Arrows, Pagination |

With the global build, use the strings: `splide.on( 'sk', function ( Slide, e ) { ... } )`.

## Argument types

`SlideComponent`:

| Property | Type | Description |
|---|---|---|
| `index` | `number` | Slide index. For clones, the clone's position: negative or `>= length` |
| `slideIndex` | `number` | For clones, the index of the original slide. `-1` for real slides |
| `slide` | `HTMLElement` | The slide element |
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

## Event constants (ESM build)

The global build has no constants, so use the strings. The ESM build (`https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js`) exports:

| Constant | Value |
|---|---|
| `EVENT_MOUNTED` | `'mounted'` |
| `EVENT_READY` | `'ready'` |
| `EVENT_MOVE` | `'move'` |
| `EVENT_MOVED` | `'moved'` |
| `EVENT_CLICK` | `'click'` |
| `EVENT_ACTIVE` | `'active'` |
| `EVENT_INACTIVE` | `'inactive'` |
| `EVENT_VISIBLE` | `'visible'` |
| `EVENT_HIDDEN` | `'hidden'` |
| `EVENT_REFRESH` | `'refresh'` |
| `EVENT_UPDATED` | `'updated'` |
| `EVENT_RESIZE` | `'resize'` |
| `EVENT_RESIZED` | `'resized'` |
| `EVENT_DRAG` | `'drag'` |
| `EVENT_DRAGGING` | `'dragging'` |
| `EVENT_DRAGGED` | `'dragged'` |
| `EVENT_SCROLL` | `'scroll'` |
| `EVENT_SCROLLED` | `'scrolled'` |
| `EVENT_OVERFLOW` | `'overflow'` |
| `EVENT_DESTROY` | `'destroy'` |
| `EVENT_ARROWS_MOUNTED` | `'arrows:mounted'` |
| `EVENT_ARROWS_UPDATED` | `'arrows:updated'` |
| `EVENT_PAGINATION_MOUNTED` | `'pagination:mounted'` |
| `EVENT_PAGINATION_UPDATED` | `'pagination:updated'` |
| `EVENT_NAVIGATION_MOUNTED` | `'navigation:mounted'` |
| `EVENT_AUTOPLAY_PLAY` | `'autoplay:play'` |
| `EVENT_AUTOPLAY_PLAYING` | `'autoplay:playing'` |
| `EVENT_AUTOPLAY_PAUSE` | `'autoplay:pause'` |
| `EVENT_LAZYLOAD_LOADED` | `'lazyload:loaded'` |
| `EVENT_SLIDE_KEYDOWN` | `'sk'` (internal) |
| `EVENT_SHIFTED` | `'sh'` (internal) |
| `EVENT_END_INDEX_CHANGED` | `'ei'` (internal) |

## Event sequences

From source, in emit order.

**`mount()`**

1. Components are constructed and set up. Media applies breakpoints and reduced motion; no `updated` is emitted.
2. Components mount in order: Media, Direction, Elements, Slides, Layout (`resized`, `overflow`), Clones, Move, Controller, Arrows (`arrows:updated`, then `arrows:mounted`), Autoplay (`autoplay:play` if it starts), Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination (`pagination:updated`, then `pagination:mounted`), Sync (`navigation:mounted` if `isNavigation`), Wheel, Live, then extensions, then Transition.
3. `mounted`. Handlers run in order, and Move's reposition among them emits `active` and `visible` for the initial slides, and Arrows emit `arrows:updated`.
4. The root gets `is-initialized`, and the state becomes `IDLE`.
5. `ready`.

**`go()` with a CSS transition**

1. State `MOVING`, then `move`. Pagination emits `pagination:updated`. With `updateOnMove`, slide status events fire here. In loop mode, the list may first be repositioned silently across the clones, with no event.
2. Transition end: state `IDLE`, then `moved`. Slides emit `inactive`/`active` and `hidden`/`visible`. Arrows emit `arrows:updated`. Live toggles `aria-busy`.

**Drag (non-free)**

`drag`, then `dragging` repeatedly, then `dragged`, then the `go()` sequence (`move` and `moved`).

**Breakpoint with `destroy: true`**

`updated`, then `destroy` (handlers removed). On leaving the breakpoint, a full `mount()` sequence runs again.

**`destroy()`**

Components are destroyed in reverse order, then `destroy` is emitted, then all handlers are removed, and the state becomes `DESTROYED`.

## Docs vs source

| Event | Docs say | Source says |
|---|---|---|
| `overflow` | No arguments listed (Events page) | `emit( EVENT_OVERFLOW, overflow )`: one `boolean`. The Overflow guide and the `EventMap` type agree with the source |
| `arrows:updated` | `prevIndex`: "An old index", `nextIndex`: "A new index" | `Controller.getPrev()` / `getNext()`: the target index of each arrow, `-1` when unavailable |
| `navigation:mounted` | `splides: Splide[]` (the `EventMap` type agrees) | `Splide.splides`, an array of `SyncTarget` objects `{ splide, isParent? }` |
| `off( 'move' )` | Removes all `move` handlers | Removes only un-namespaced `move` handlers |
| `resize` | "Whenever the window is resized" | Also on window `load`, after each LazyLoad image, and after images in `add()`-ed slides load |
| Internal events | Not documented | `sk`, `sh`, `ei` (see above) |
