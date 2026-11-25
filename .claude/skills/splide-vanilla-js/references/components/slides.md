# Slides component

Source: https://splidejs.com/components/slides/
Source file: `src/js/components/Slides/Slides.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner. For the Slide object itself, see `slide.md`.

Purpose: manages every Slide sub-component, clones included. It adds, removes, finds and iterates slides.

Access: `splide.Components.Slides` (filled in by `splide.mount()`). `splide.add()` and `splide.remove()` are aliases of `Slides.add()` and `Slides.remove()`. `splide.length` is `Slides.getLength( true )`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `register( slide, index, slideIndex )` | `void` | Creates and mounts a Slide object for an element and adds it to the sorted collection. Used internally by Slides and Clones. |
| `get( excludeClones? )` | `SlideComponent[]` | All Slide objects sorted by `index`, optionally without clones. |
| `getIn( page )` | `SlideComponent[]` | Slides that belong to a page. |
| `getAt( index )` | `SlideComponent \| undefined` | The Slide object with `Slide.index === index` (clone indices work). |
| `add( slides, index? )` | `void` | Inserts one or more slides, then refreshes the carousel. |
| `remove( matcher )` | `void` | Removes matching slides from the DOM, then refreshes the carousel. |
| `forEach( iteratee, excludeClones? )` | `void` | Iterates over Slide objects. |
| `filter( matcher )` | `SlideComponent[]` | Slide objects that match. |
| `style( prop, value, useContainer? )` | `void` | Sets an inline style on every slide (or its container). |
| `getLength( excludeClones? )` | `number` | Number of slides. |
| `isEnough()` | `boolean` | `true` if there are more Slide objects (clones included) than `perPage`. |
| `update()` | `void` | (undocumented, from source) Calls `update()` on every Slide object (status classes, ARIA, events). |

Lifecycle members `mount()` and `destroy()` are also on the object. Splide calls them.

`SlideMatcher` is one of:

- a number: matches `Slide.index`;
- an array of numbers;
- a selector string: `Slide.slide.matches( selector )`;
- a predicate function `( Slide, index, Slides ) => boolean`, called like `Array.prototype.filter` over the Slide objects (clones included).

`SlidesIteratee` is `( Slide, index, Slides ) => void`.

### register()

`register( slide: HTMLElement, index: number, slideIndex: number ): void`

| Param | Description |
|---|---|
| `slide` | A slide element to register. |
| `index` | A slide index. |
| `slideIndex` | For clones, the index of the original slide. Must be `-1` for normal slides. |

This only registers the object; it does not insert the element or recompute layout. Use `add()` to add slides.

### get()

`get( excludeClones?: boolean ): SlideComponent[]`

| Param | Description |
|---|---|
| `excludeClones` | Optional. Determines whether to exclude clones or not. |

Without `excludeClones`, it returns the internal array itself. Copy it (`.slice()`) before mutating.

### getIn()

`getIn( page: number ): SlideComponent[]`

| Param | Description |
|---|---|
| `page` | A page index. |

Returns the Slides whose `index` is in `Controller.toIndex( page ) .. + perPage - 1` (only 1 slide when `focus`/`isNavigation` is set).

### getAt()

`getAt( index: number ): SlideComponent | undefined`

| Param | Description |
|---|---|
| `index` | A slide index. |

Returns a Slide object if available, otherwise `undefined`.

```js
const splide = new Splide( '.splide' ).mount();
const active = splide.Components.Slides.getAt( splide.index );
active.slide.classList.add( 'highlight' );
```

### add()

Docs say `add( slide: string | Element | Array<string | Element>, index?: number ): void` (the params table names the first argument `items`).

| Param | Description |
|---|---|
| `slides` | A slide element, an HTML string, or an array of them. |
| `index` | Optional. Insert before the real slide at this position (in `Elements.slides`). If omitted or out of range, the slide is appended to the list. |

Docs example (the bare calls are `splide.Components.Slides.add( ... )` or `splide.add( ... )`):

```js
// Adds a slide by HTML:
splide.add( '<li class="splide__slide"></li>' );

// or adds an element:
const slide = document.createElement( 'li' );
slide.classList.add( 'splide__slide' );
splide.add( slide );
```

Behaviour from source:

- An HTML string is parsed with `DOMParser`, and only its first element is used. Strings that do not produce an element are ignored.
- The `classes.slide` class (`splide__slide`) is added automatically.
- After all items are inserted, `refresh` is emitted synchronously. Elements re-collects the slides, Slide objects and clones are rebuilt, and layout and pagination are redone.
- For each added element, `resize` is emitted once all its `<img>` elements fire `load` or `error`, or immediately if it has none.
- Call it after `mount()`.

```js
splide.add( [
  '<li class="splide__slide"><img src="4.jpg" alt=""></li>',
  '<li class="splide__slide"><img src="5.jpg" alt=""></li>',
], 0 ); // insert at the start
```

### remove()

`remove( matcher: SlideMatcher ): void`. The source interface names the parameter `selector`; the implementation names it `matcher`.

| Param | Description |
|---|---|
| `matcher` | An index, an array of indices, a selector string, or a predicate function. |

The predicate gets these arguments:

| Argument | Description |
|---|---|
| `Slide` | The `Slide` component object being processed. |
| `index` | The index of the current item in the array (not `Slide.index`). |
| `Slides` | The array of `Slide` objects being iterated over. |

Docs example (shown here through `splide.remove`):

```js
splide.remove( 0 );

// Removes slides at 0 and 1:
splide.remove( [ 0, 1 ] );

// Removes slides by a selector:
splide.remove( '.is-visible' );

// Removes slides by a predicate function:
splide.remove( function ( Slide ) { return Slide.index % 2 === 0; } );
```

Behaviour from source:

- Matching runs over all Slide objects, clones included. Number matchers only hit clones when you pass clone indices. A selector or predicate can match clones too, but clones are regenerated on the following refresh anyway.
- The matched elements are removed from the DOM, then `refresh` is emitted synchronously.

### forEach()

`forEach( iteratee: SlidesIteratee, excludeClones?: boolean ): void`

| Param | Description |
|---|---|
| `iteratee` | An iteratee function that takes a Slide object, an index and an array with Slides. |
| `excludeClones` | Optional. Determines whether to exclude clones or not. |

```js
splide.Components.Slides.forEach( function ( Slide ) {
  Slide.slide.setAttribute( 'data-pos', String( Slide.index ) );
}, true );
```

### filter()

`filter( matcher: SlideMatcher ): SlideComponent[]`

| Param | Description |
|---|---|
| `matcher` | An index, an array of indices, a selector string, or a predicate function (same arguments as for `remove()`). |

Returns an array of SlideComponent objects (clones included unless the matcher excludes them).

```js
const visible = splide.Components.Slides.filter( '.is-visible' );
const realOnly = splide.Components.Slides.filter( function ( Slide ) { return ! Slide.isClone; } );
```

### style()

`style( prop: string, value: string | number, useContainer?: boolean ): void`

| Param | Description |
|---|---|
| `prop` | A CSS property name (camelCase, assigned to `element.style`). |
| `value` | A CSS value to add. `''` clears it. `null` is ignored. |
| `useContainer` | Optional. Apply to each slide's `.splide__slide__container` instead, falling back to the slide when there is none. |

Applies to clones too. Layout overwrites `width`, `height` and the gap margin on resize and updates.

### getLength()

`getLength( excludeClones?: boolean ): number`

| Param | Description |
|---|---|
| `excludeClones` | Optional. Determines whether to exclude clones or not. |

With `excludeClones`, returns `Elements.slides.length` (real slides). Without it, returns the number of Slide objects, clones included.

### isEnough()

`isEnough(): boolean`

Returns `get().length > options.perPage` (clones included). Autoplay will not `play()`, and Controller cannot move, when this is `false` (unless `focus`/`isNavigation` is set).

### update()

(undocumented, from source) `update(): void`

Calls `Slide.update()` on every Slide object. Move calls it from `reposition()` (on mount, resize, update and refresh).

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `refresh` | `()` | At the end of `add()` and `remove()`. |
| `resize` | `()` | After the images in an added slide load (or immediately when it has none). |

Each Slide object also emits `active`, `inactive`, `visible`, `hidden` and `click` (see `slide.md`).

Listened:

| Event | Action |
|---|---|
| `refresh` | Destroys all Slide objects and registers the real slides again from `Elements.slides`. Clones re-register themselves through the Clones component. |

## Options read

| Option | Use |
|---|---|
| `classes.slide` | Class added to slides inserted by `add()`. |
| `perPage` | `isEnough()` and `getIn()`. |
| `focus`, `isNavigation` | `getIn()` page size of 1 (through `Controller.hasFocus()`). |
