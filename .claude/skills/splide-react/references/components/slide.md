# Slide component (React)

The Slide sub-component (one object per slide, clones included) as reached from React Splide: properties, `style()`, `isWithin()`, `update()`, and the slide event props.

Contents: [Access from React](#access-from-react) · [Properties](#properties) · [Methods](#methods) · [Example](#example) · [update()](#update) · [Attributes set on mount](#attributes-set-on-mount-non-clones) · [Events](#events-emitted--listened) · [Options read](#options-read) · [React notes](#react-notes)

Source: https://splidejs.com/components/slide/
Source file: `src/js/components/Slides/Slide.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: the Slide sub-component. Each slide element, clones included, gets one Slide object, which manages that slide's status classes, ARIA attributes and events.

## Access from React

There is no `splide.Components.Slide`. Get Slide objects from the Slides component, or from event props:

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Slides.getAt( 0 );   // also .get(), .getIn( page ), .filter( matcher ), .forEach( fn )
```

- The `onActive`, `onInactive`, `onVisible`, `onHidden` and `onClick` props pass the Slide object as the second argument: `onActive={ ( splide, Slide ) => ... }`. `onLazyLoadLoaded` passes it third: `( splide, img, Slide )`.
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true`, and the Slide objects are destroyed. A remount creates new ones. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- Slide objects are destroyed and recreated on every `refresh`. In React that is every change in the `<SplideSlide>` children (React Splide calls `splide.refresh()`), plus `splide.refresh()` calls and clone count changes. Do not keep Slide objects in state or refs across renders that change the slides.
- To type a helper that takes the instance without importing from `@splidejs/splide`, use `type SplideInstance = NonNullable<Splide[ 'splide' ]>` (as in the example). If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

## Properties

| Name | Type | Description |
|---|---|---|
| `index` | `number` | The slide index. For clones it is negative (head clones: `-count .. -1`) or `>= length` (tail clones: `length .. length + count - 1`). |
| `slideIndex` | `number` | For a clone, the index of the original slide it copies. `-1` for normal slides. |
| `slide` | `HTMLElement` | The slide element. For a real slide, the `<li>` that `<SplideSlide>` rendered. For a clone, a DOM copy that React did not render. |
| `container` | `HTMLElement \| undefined` | The `.splide__slide__container` direct child, if any, otherwise `undefined`. |
| `isClone` | `boolean` | `true` if this Slide is a clone (`slideIndex > -1`). |

Properties are fixed when the object is created. Slide objects are destroyed and recreated on every `refresh`, so do not keep references across refreshes.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `style( prop, value, useContainer? )` | `void` | Sets an inline style on the slide, or on its container when `useContainer` is `true` and a container exists. |
| `isWithin( from, distance )` | `boolean` | `true` if this slide is within `distance` slides of index `from`. |
| `update()` | `void` | (undocumented, from source) Re-computes `is-active`, `is-visible`, `is-prev`, `is-next` and the ARIA/tabindex state, emitting `active`/`inactive`/`visible`/`hidden` on changes. |

Lifecycle members `mount()` and `destroy()` (undocumented, from source) are also on the object. The Slides component calls them. `destroy()` removes status classes and Splide attributes, and restores the original `style` and `aria-label`.

### style()

`style( prop: string, value: string | number, useContainer?: boolean ): void`

| Param | Description |
|---|---|
| `prop` | A CSS property name, assigned as `element.style[ prop ] = value`. Use camelCase (`'backgroundColor'`). |
| `value` | A CSS value. Numbers are stringified without a unit. `''` removes the inline property. `null` leaves it unchanged. |
| `useContainer` | Optional. Apply to `container` instead of `slide`. Falls back to `slide` when there is no container. |

Layout re-applies `width`, `height` and the gap margin on resize and option updates. Styles you set on those properties get overwritten. A `style` prop on `<SplideSlide>` that sets the same property also overwrites yours when its value changes, since React updates only the properties that changed.

### isWithin()

`isWithin( from: number, distance: number ): boolean`

| Param | Description |
|---|---|
| `from` | The index of a base slide. |
| `distance` | The distance to test. The method returns `true` if the slide is within this number. |

Returns `true` if `abs( from - index ) <= distance`. For non-clone slides with `rewind` or the `loop` type, the distance wraps around: `min( diff, length - diff )`. LazyLoad uses this to decide which images to load.

## Example

Mark slides near the active one, log activations, and move to a clicked slide:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// The core instance type, taken from the React component, so no import from '@splidejs/splide' is needed.
type SplideInstance = NonNullable<Splide[ 'splide' ]>;

// Mark slides within 1 of the active slide. Splide's own classes stay intact because this
// uses classList, not a React className.
function markNear( splide: SplideInstance, index: number ) {
  splide.Components.Slides.forEach( Slide => {
    Slide.slide.classList.toggle( 'is-near', Slide.isWithin( index, 1 ) );
  } );
}

export default function SlideExample() {
  const [ log, setLog ] = useState( '' );

  return (
    <>
      <Splide
        options={ { perPage: 1, label: 'Slide demo' } }
        onMounted={ ( splide ) => markNear( splide, splide.index ) }
        onMoved={ ( splide, index ) => markNear( splide, index ) }
        onActive={ ( splide, Slide ) => setLog( `active ${ Slide.index }` ) }
        onClick={ ( splide, Slide ) => {
          Slide.style( 'opacity', 0.5 );
          splide.go( Slide.isClone ? Slide.slideIndex : Slide.index );
        } }
      >
        { [ 1, 2, 3, 4, 5 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <p id="log">{ log }</p>
    </>
  );
}
```

Tested: at mount slides 1 and 2 have `is-near` and the log reads `active 0` (`onActive` fires during mount, because event props are bound first); clicking slide 3 moves there, marks slides 2 to 4, logs `active 2` and sets its inline opacity to `0.5`.

## update()

(undocumented, from source) `update(): void`

It runs automatically on `moved`, `scrolled` and the internal `sh` (loop shift) event, and also on `move` when `updateOnMove` is `true`. `Slides.update()` calls it for every slide. It does nothing after the Slide is destroyed. What it updates:

- `is-active`: when `splide.index === index`, or (with `cloneStatus`, default `true`) when `splide.index === slideIndex` for clones. On a change, `aria-current="true"` is set when `isNavigation` is on, and `active( Slide )` or `inactive( Slide )` is emitted.
- `is-visible`: when the slide's rect lies fully inside the track's rect along the axis. For `fade`, visible equals active. On a change, `visible( Slide )` or `hidden( Slide )` is emitted.
- `aria-hidden="true"` on slides that are not visible, except the active slide when it is not a clone (code: `hidden = ! visible && ( ! isActive() || isClone )`). Visible slides never get it. It is not changed while moving or scrolling, to avoid screen readers reading mid-transition.
- `tabindex="-1"` on focusable descendants (`focusableNodes` option) of hidden slides, removed otherwise. With `slideFocus`, the slide itself gets `tabindex="0"` or `-1`.
- `is-prev` / `is-next` when `index === splide.index - 1` / `+ 1`.
- If a slide that has focus becomes not visible, focus moves to the active slide.

The change checks compare with the element's current classes (`hasClass( slide, 'is-active' )`). If React rewrites a slide's `class` attribute (a changed `className` prop), the next update puts the classes back and emits `active` or `visible` again for that slide (from source).

## Attributes set on mount (non-clones)

| Attribute | Value |
|---|---|
| `id` | `{rootId}-slide01`, `-slide02`, ... (1-based, zero-padded to 2 digits). Clones get `{rootId}-clone01`, ... from the Clones component. An `id` you pass on `<SplideSlide>` is overwritten. |
| `role` | `'tabpanel'` when `pagination` is on, otherwise `'group'`. |
| `aria-roledescription` | `i18n.slide` (`'slide'`). |
| `aria-label` | The existing `aria-label` of the element, else `i18n.slideLabel` (`'%s of %s'`), e.g. `'1 of 5'`. An `aria-label` prop on `<SplideSlide>` is kept. |

When the slider is a navigation (`isNavigation`, on `navigation:mounted`):

- `aria-label` becomes `i18n.slideX` (`'Go to slide %s'`).
- `aria-controls` lists the ids of the matching slides in the synced sliders.
- `role="button"` is set when `slideFocus` is on, and `aria-roledescription` is removed.

## Events emitted / listened

Emitted (handlers receive the Slide object):

| Event | React prop | Arguments (prop) | When |
|---|---|---|---|
| `active` | `onActive` | `( splide, Slide )` | The slide becomes active. |
| `inactive` | `onInactive` | `( splide, Slide )` | The slide stops being active. |
| `visible` | `onVisible` | `( splide, Slide )` | The slide becomes fully visible in the track. |
| `hidden` | `onHidden` | `( splide, Slide )` | The slide stops being fully visible. |
| `click` | `onClick` | `( splide, Slide, e )` | DOM `click` on the slide element (also for clones). |
| `sk` (internal slide-keydown event) | none | core: `( Slide, e )` | DOM `keydown` on the slide. Sync uses it for navigation sliders. |

- `onClick` on `<Splide>` is this slide event, not a DOM click on the root: React Splide's prop types remove React's DOM event handlers from `<Splide>`.
- A React `onClick` on `<SplideSlide>` (or on an element inside it) fires for real slides only. Clones are DOM copies that React never rendered, so React handlers do not fire on them (tested). Use the `onClick` prop of `<Splide>`, which fires for clones too, and map `Slide.slideIndex` back to your data (see `references/components/other-components.md`, Clones).
- `active`, `inactive`, `visible` and `hidden` fire for clones too, and several fire on every move. Updating state from them re-renders the parent each time; that is fine for small trees, but keep heavy work out.

Listened:

| Event | Action |
|---|---|
| `moved`, `scrolled`, `sh` | `update()` |
| `move` | `update()`, only when `updateOnMove` is `true`. |
| `navigation:mounted` | Applies the navigation attributes above. |

## Options read

| Option | Use |
|---|---|
| `pagination` | Slide `role` (`tabpanel` vs `group`). |
| `i18n.slide`, `i18n.slideLabel`, `i18n.slideX` | ARIA labels. |
| `isNavigation` | `aria-current` on the active slide, navigation attributes. |
| `slideFocus` | `tabindex` on slides; `role="button"` for navigation. Defaults to the value of `isNavigation` (set by Sync). |
| `updateOnMove` | Update status at `move` instead of waiting for `moved`. |
| `cloneStatus` | Whether clones of the active slide also get `is-active` (default `true`). |
| `focusableNodes` | Selector for descendants whose `tabindex` is toggled (default `'a, button, textarea, input, select, iframe'`). |
| `rewind`, `type` | Wrap-around distance in `isWithin()`. `fade` visibility rule. |
| `direction` | Edges used for the visibility test (through Direction, axis only). |

`isNavigation`, `updateOnMove`, `i18n`, `pagination` and `slideFocus` are read once, when the Slide object is created (at mount and on each `refresh`). `cloneStatus`, `focusableNodes`, `rewind` and `type` are read at call time. So an `options` prop change to `pagination` or `i18n` reaches existing slides only after the next `refresh` (a change in the slide children, or `ref.current?.splide?.refresh()`).

## React notes

- Style status with Splide's classes in CSS (`.splide__slide.is-active`, `.is-visible`, `.is-prev`, `.is-next`) instead of a React `className` computed from the index: a changing `className` on `<SplideSlide>` rewrites the attribute and drops Splide's classes until the next update (tested).
- `<SplideSlide>` passes every attribute to the `<li>`, so `data-*` attributes (`data-splide-interval`, `data-splide-hash`, ...) work as JSX props.
