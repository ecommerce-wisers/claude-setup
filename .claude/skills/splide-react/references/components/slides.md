# Slides component (React)

`splide.Components.Slides` as reached from React Splide: finding and iterating Slide objects, and why slides come from JSX state instead of `add()` / `remove()`.

Contents: [Access from React](#access-from-react) · [Methods](#methods) · [Adding and removing slides in React](#adding-and-removing-slides-in-react) · [register()](#register) · [get() / getIn() / getAt()](#get) · [add()](#add) · [remove()](#remove) · [forEach() / filter()](#foreach) · [style() / getLength() / isEnough() / update()](#style) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/slides/
Source file: `src/js/components/Slides/Slides.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4. For the Slide object itself, see `references/components/slide.md`.

Purpose: manages every Slide sub-component, clones included. It adds, removes, finds and iterates slides.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Slides;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.Slides.getAt( splide.index ) }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Slides` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

`splide.add()` and `splide.remove()` are aliases of `Slides.add()` and `Slides.remove()`. `splide.length` is `Slides.getLength( true )`.

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `register( slide, index, slideIndex )` | `void` | Creates and mounts a Slide object for an element and adds it to the sorted collection. Used internally by Slides and Clones. |
| `get( excludeClones? )` | `SlideComponent[]` | All Slide objects sorted by `index`, optionally without clones. |
| `getIn( page )` | `SlideComponent[]` | Slides that belong to a page. |
| `getAt( index )` | `SlideComponent \| undefined` | The Slide object with `Slide.index === index` (clone indices work). |
| `add( slides, index? )` | `void` | Inserts one or more slides, then refreshes the carousel. Do not use it with React-rendered slides (see below). |
| `remove( matcher )` | `void` | Removes matching slides from the DOM, then refreshes the carousel. Do not use it with React-rendered slides (see below). |
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

## Adding and removing slides in React

Render the slides from state. When the `<SplideSlide>` children change, React Splide compares the list's child elements after the render and calls `splide.refresh()`, which rebuilds Elements, Slide objects, clones, layout and pagination.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function DynamicSlidesExample() {
  const ref = useRef<Splide>( null );
  const [ items, setItems ] = useState( [ 1, 2, 3 ] );
  const [ info, setInfo ] = useState( '' );

  // Runs after <Splide> has called refresh() for the new children, so the Slide objects are current.
  useEffect( () => {
    const Slides = ref.current?.splide?.Components.Slides;

    if ( Slides ) {
      const visible = Slides.filter( '.is-visible' ).map( Slide => Slide.index );
      setInfo( `${ Slides.getLength( true ) } slides, visible ${ visible.join( ',' ) }` );
    }
  }, [ items ] );

  return (
    <>
      <Splide ref={ ref } options={ { perPage: 2, label: 'Dynamic slides' } }>
        { items.map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <button type="button" id="add" onClick={ () => setItems( list => [ ...list, list.length + 1 ] ) }>Add</button>
      <button type="button" id="prepend" onClick={ () => setItems( list => [ 0, ...list ] ) }>Insert at start</button>
      <button type="button" id="remove" onClick={ () => setItems( list => list.slice( 1 ) ) }>Remove first</button>
      <p id="info">{ info }</p>
    </>
  );
}
```

Tested in production and under StrictMode: `3 slides, visible 0,1` at mount, `4 slides` after Add, the inserted slide becomes `splide01-slide01` (ids are reassigned on refresh), and after two removals `3 slides` with 2 dots.

- Use stable `key`s, so React keeps the same `<li>` for the same item.
- Read Slides in a `useEffect` that depends on the slide data, as above. `<Splide>`'s `componentDidUpdate` (which calls `refresh()`) runs before the parent's effects.
- An `onRefresh` prop is too early: event props are bound before mount, so they run before the components' own `refresh` listeners. Tested: inside `onRefresh` after adding a fourth slide, `Slides.getLength( true )` was still `3`; a listener added after mount with `splide.on( 'refresh', ... )` saw `4`.
- Do not call `splide.add()` / `splide.remove()` (or `Slides.add()` / `Slides.remove()`) on a carousel whose slides React renders. They insert and remove `<li>` elements behind React's back. Tested: after `splide.remove( 0 )`, the next state change that removed the same slide made React throw `NotFoundError: Failed to execute 'removeChild' on 'Node'`, and the whole tree unmounted. Slides that `add()` inserted are also unknown to React and stay in the list across renders.

## register()

`register( slide: HTMLElement, index: number, slideIndex: number ): void`

| Param | Description |
|---|---|
| `slide` | A slide element to register. |
| `index` | A slide index. |
| `slideIndex` | For clones, the index of the original slide. Must be `-1` for normal slides. |

This only registers the object; it does not insert the element or recompute layout. Rarely needed in React.

## get()

`get( excludeClones?: boolean ): SlideComponent[]`

| Param | Description |
|---|---|
| `excludeClones` | Optional. Determines whether to exclude clones or not. |

Without `excludeClones`, it returns the internal array itself. Copy it (`.slice()`) before mutating, and do not put it in React state (it is mutated in place on every refresh).

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

```tsx
onMoved={ ( splide, index ) => splide.Components.Slides.getAt( index )?.slide.classList.add( 'highlight' ) }
```

## add()

Docs say `add( slide: string | Element | Array<string | Element>, index?: number ): void` (the params table names the first argument `items`).

| Param | Description |
|---|---|
| `slides` | A slide element, an HTML string, or an array of them. |
| `index` | Optional. Insert before the real slide at this position (in `Elements.slides`). If omitted or out of range, the slide is appended to the list. |

Behaviour from source:

- An HTML string is parsed with `DOMParser`, and only its first element is used. Strings that do not produce an element are ignored.
- The `classes.slide` class (`splide__slide`) is added automatically.
- After all items are inserted, `refresh` is emitted synchronously. Elements re-collects the slides, Slide objects and clones are rebuilt, and layout and pagination are redone.
- For each added element, `resize` is emitted once all its `<img>` elements fire `load` or `error`, or immediately if it has none.

In React, render slides from state instead (above). `add()` is only safe when no slide in the list is rendered by React.

## remove()

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

Behaviour from source:

- Matching runs over all Slide objects, clones included. Number matchers only hit clones when you pass clone indices. A selector or predicate can match clones too, but clones are regenerated on the following refresh anyway.
- The matched elements are removed from the DOM, then `refresh` is emitted synchronously.

In React, remove the item from state instead. Removing a React-rendered `<li>` with `remove()` breaks React's next update of the list (tested, see above).

## forEach()

`forEach( iteratee: SlidesIteratee, excludeClones?: boolean ): void`

| Param | Description |
|---|---|
| `iteratee` | An iteratee function that takes a Slide object, an index and an array with Slides. |
| `excludeClones` | Optional. Determines whether to exclude clones or not. |

```tsx
splide.Components.Slides.forEach( Slide => {
  Slide.slide.classList.toggle( 'is-near', Slide.isWithin( index, 1 ) );
} );
```

(From the tested example in `references/components/slide.md`.)

### filter()

`filter( matcher: SlideMatcher ): SlideComponent[]`

| Param | Description |
|---|---|
| `matcher` | An index, an array of indices, a selector string, or a predicate function (same arguments as for `remove()`). |

Returns an array of SlideComponent objects (clones included unless the matcher excludes them). The example above uses `Slides.filter( '.is-visible' )`. A predicate such as `Slide => ! Slide.isClone` keeps only real slides.

## style()

`style( prop: string, value: string | number, useContainer?: boolean ): void`

| Param | Description |
|---|---|
| `prop` | A CSS property name (camelCase, assigned to `element.style`). |
| `value` | A CSS value to add. `''` clears it. `null` is ignored. |
| `useContainer` | Optional. Apply to each slide's `.splide__slide__container` instead, falling back to the slide when there is none. |

Applies to clones too. Layout overwrites `width`, `height` and the gap margin on resize and updates. A `style` prop on `<SplideSlide>` overwrites the same property when its value changes.

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

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `refresh` | `onRefresh` | `( splide )` | At the end of `add()` and `remove()`. React Splide's own `splide.refresh()` emits it too. |
| `resize` | `onResize` | `( splide )` | After the images in an added slide load (or immediately when it has none). |

Each Slide object also emits `active`, `inactive`, `visible`, `hidden` and `click` (`onActive`, `onInactive`, `onVisible`, `onHidden`, `onClick`; see `references/components/slide.md`).

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
