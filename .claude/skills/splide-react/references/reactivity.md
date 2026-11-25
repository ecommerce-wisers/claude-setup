# Reactivity: how React Splide reacts to props and state

What happens when props, children or state change, and the patterns that follow from it. Everything here comes from the React Splide 0.7.12 source and was tested in React 19.3, in production and in development with StrictMode.

Contents: [Lifecycle](#lifecycle) · [Options](#options) · [Slides](#slides) · [Event props](#event-props) · [Read once at mount](#read-once-at-mount) · [className and style](#classname-and-style) · [Remounting with key](#remounting-with-key) · [Driving the carousel from state](#driving-the-carousel-from-state) · [StrictMode](#strictmode)

## Lifecycle

`<Splide>` is a class component. Each step maps to a core call:

| React step | What React Splide does |
|---|---|
| `componentDidMount` | `new Splide( root, options )`, binds the event props with `splide.on()`, then `splide.mount( extensions, transition )` |
| `componentDidUpdate` | If `options` isn't deep-equal to the last options: `splide.options = options`. Then, if the list's child elements changed (shallow compare): `splide.refresh()` |
| `componentWillUnmount` | `splide.destroy()` (completely) and `ref.current.splide = undefined` |

Because the handlers are bound before `mount()`, `onMounted`, `onArrowsMounted` and `onPaginationMounted` fire.

## Options

- **Inline objects are fine.** The comparison is deep, so `options={ { perPage: 3 } }` re-created on every render does nothing when the content is the same (tested). There's no need for `useMemo`. Functions inside the options (`easingFunc`) are compared by identity, so define those at module level.
- **A change merges the whole prop.** `splide.options = options` deep-merges the entire prop into the current options and emits `updated`. Changing `perPage`, `gap`, `arrows`, `pagination`, `drag`, `keyboard` or `label` updates the carousel live.
- **Removing a key doesn't reset it.** Going from `{ perPage: 2, gap: '40px', arrows: false }` to `{ perPage: 2 }` left `gap` at `40px` and the arrows off (tested). Pass the value you want (`gap: 0`, `arrows: true`).
- **An explicit `undefined` replaces the default.** `{ perPage: props.perPage }` with no prop gives a carousel that can't move, and `{ i18n: undefined }` throws at mount (tested). Leave the key out (`...( labels && { i18n: labels } )`) or use a fallback (`props.perPage ?? 1`).
- **An update overrides active breakpoints.** Every key in both the prop and a matching breakpoint takes the prop's value until the next media query change, and a `speed` in the prop undoes reduced motion the same way (tested). Keep responsive keys only inside `breakpoints` (`mediaQuery: 'min'` with a `0` breakpoint), or don't change the options after mount.
- **Changes inside `breakpoints` are ignored.** Breakpoints, `mediaQuery` and `reducedMotion` are registered once at mount (tested). Remount with `key`.
- **Never put a DOM element or `ref.current` in the options.** React Splide's deep compare and merge walk into React's internal fiber and crash with "Maximum call stack size exceeded" (tested). Pass a selector, or set it on `splide.options` after mount (see `references/extensions/intersection.md`).
- Which options can change live and which need a remount: the Live column in `references/options.md`.

## Slides

- **List changes refresh.** Adding, removing or reordering `<SplideSlide>` children changes the list's elements, and React Splide calls `splide.refresh()`. Use stable `key`s, so React keeps the same `<li>` for the same slide.
- **Content changes don't.** Changing what's inside a slide keeps the same `<li>`, so there's no refresh (tested). If the change affects size (`autoWidth`, `autoHeight`, wrapping text), refresh it yourself:

  ```tsx
  useEffect( () => {
    ref.current?.splide?.refresh();
  }, [ text ] );
  ```

- **`refresh()` restores each `<li>`'s original `style`.** A `<SplideSlide style={ { width } }>` changed from 150px to 300px went back to 150px on refresh (tested). Size an inner element or use a class. Splide's layout also overwrites inline `width` and `height` on slides, so set heights with `height`, `fixedHeight` or `heightRatio`.
- **Loop carousels refresh on every later re-render once the list has changed.** React Splide stores the list's children, clones included, and each refresh replaces the clones. So after the first slide change, every unrelated re-render calls `refresh()` again (tested: one refresh per state update), which also restarts autoplay. Give a loop carousel a `key` built from the slides, `key={ items.map( item => item.id ).join() }`, so a change remounts it instead. Never call `setState` inside `onRefresh`: with a loop it re-renders, refreshes and loops until React stops with "Maximum update depth exceeded".
- **Don't change the slides during a move** (for example from `onMove`). The refresh strips the list's transition, `moved` never fires, and the state stays `MOVING` (tested).
- **Don't call `splide.add()` or `splide.remove()` on React-rendered slides.** React later throws `NotFoundError` (removeChild) and unmounts the whole tree (tested). Change the slides through state.
- **Grid moves the slide elements.** With the Grid extension, a slide change through state throws `NotFoundError`; remount with a key built from the slides (see `references/extensions/grid.md`).

## Event props

- **Captured at mount.** React Splide reads each event prop once, in `componentDidMount`. A handler keeps seeing the props and state from the first render, and a handler added or changed later is ignored (tested: after three state updates, `onMoved` still saw `count` as `0`). Keep the latest values in a ref written during render:

  ```tsx
  const [ count, setCount ] = useState( 0 );
  const latest = useRef( count );
  latest.current = count;   // during render, not in an effect

  <Splide onMoved={ () => console.log( latest.current ) } />   // sees the current count
  ```

  Write it during render rather than in `useEffect`: `updated` and `refresh` are emitted inside React's commit, before effects run, so an effect-written ref is one value behind for them (tested). Setting state from a handler is fine without a ref, because the setter never changes: `onMoved={ ( splide, newIndex ) => setIndex( newIndex ) }`. `references/events.md` has a `useLatestHandlers` hook.
- **They run before Splide's own handlers.** Event props are registered before `mount()`, so for the same event they run before the core components update. In `onUpdated` after a `perPage` change, `Controller.getEnd()` still returned the old value; in `onRefresh`, `splide.length` was the old count; `Drag.disable( true )` called in `onUpdated` was reset (all tested). When a handler needs recomputed state, register it with `splide.on()` after mount (in `onMounted` or an effect):

  ```tsx
  useEffect( () => {
    const splide = ref.current?.splide;
    if ( ! splide ) return;
    splide.on( 'updated.end refresh.end', () => setEnd( splide.Components.Controller.getEnd() ) );
    return () => { splide.off( 'updated.end refresh.end' ); };
  }, [] );
  ```

- **A breakpoint `destroy: true` removes them for good.** When a breakpoint destroys the carousel and a later media change remounts it, core removes every `splide.on()` handler, including the event props, and React Splide never binds them again (tested: `onMoved` stayed silent). `ref.current.splide` stays defined; `splide.state.is( 7 )` is `true` while destroyed. Listen from an extension's `setup()` instead, because extensions run again on every remount (see `references/custom-extensions.md`).
- **`splide.off( 'moved' )` also removes the `onMoved` prop.** `off()` without a namespace removes every handler for the event, including React Splide's. Namespace your own listeners (`'moved.thumbs'`).
- There's no prop for `overflow` or for any extension event. See `references/events.md` for where to register those.

## Read once at mount

These are used only when the core instance is created or mounted, so a later prop change is ignored:

- the `extensions` and `transition` props (tested)
- the event props (above)
- `tag` and `hasTrack`: don't change them after mount. React would swap the root element (`tag`) or the track and list (`hasTrack`) under a live instance that still points at the old elements.
- options that are fixed at mount, such as `type` (a live change breaks the carousel), `autoplay` (changing it from `false` to `true` didn't start autoplay), `start`, `destroy`, `breakpoints` and `lazyLoad` (all tested). The full list is the Remount entries in `references/options.md`.

## className and style

- **Keep `className` on `<Splide>` constant.** A change makes React rewrite the root's `class` attribute, which drops every class Splide added, `is-initialized` included, so the core CSS hides the carousel (`visibility: hidden`) (tested). On `<SplideSlide>`, a changing `className` drops `is-active` and `is-visible`. Put changing classes on a wrapper element.
- **`style` and `tabIndex` on `<Splide>`, and `style` on `<SplideTrack>`, don't survive a destroy**, and so are lost in development (see StrictMode below).

## Remounting with key

To apply something that's read once, give `<Splide>` a `key` that changes with it. React unmounts the old carousel (which destroys it) and mounts a new one, with the event props bound again:

```tsx
<Splide key={ String( autoplay ) } options={ { autoplay, interval: 3000, label: 'Promotions' } }>
  { slides }
</Splide>
```

Tested: the keyed carousel started autoplaying after the toggle, and the unkeyed one didn't. A remount resets the index to `start`, so pass `start: currentIndex` if the position must survive. Anything attached to the old instance with `splide.on()` or `sync()` has to be attached again: put the key in the effect's dependencies.

## Driving the carousel from state

There's no `index` prop. Hold the index in state, move the carousel in an effect, and update the state from `onMoved`:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export function Controlled() {
  const ref = useRef<Splide>( null );
  const [ index, setIndex ] = useState( 0 );

  useEffect( () => {
    const splide = ref.current?.splide;
    if ( splide && splide.index !== index ) splide.go( index );
  }, [ index ] );

  return (
    <>
      <Splide ref={ ref } options={ { label: 'Controlled', perPage: 1 } } onMoved={ ( splide, newIndex ) => setIndex( newIndex ) }>
        { [ 0, 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      { [ 0, 1, 2, 3 ].map( n => (
        <button key={ n } type="button" aria-pressed={ index === n } onClick={ () => setIndex( n ) }>{ n + 1 }</button>
      ) ) }
    </>
  );
}
```

Tested: the buttons move the carousel, and the arrows and dragging update the state. The `splide.index !== index` check stops the effect from calling `go()` again for a move the carousel made itself. `ref.current?.splide` is `undefined` during the first render, so read it in effects and handlers, never in the render body. Don't call `go()` while a `destroy` breakpoint has the carousel destroyed: it resets the state to `IDLE` and the carousel never remounts (check `splide.state.is( 7 )` first).

## StrictMode

In development, StrictMode mounts each component, unmounts it and mounts it again. For `<Splide>` that means create, destroy, create. What that changes (all tested):

- **Props that `destroy()` strips stay stripped**, because React doesn't set them again: `aria-label`, `aria-labelledby`, `role` and `tabIndex` on `<Splide>`, and `style` on `<Splide>` and `<SplideTrack>`. A `role="group"` prop ends up as `role="region"`. Pass labels and role through the options instead, which survive: `options={ { label: '...' } }`, `labelledby`, `role`.
- **Mount-time props fire twice**, with one `onDestroy` in between: `onMounted`, `onReady`, `onArrowsMounted`, `onPaginationMounted`, `onAutoplayPlay`. Make them idempotent.
- **With `lazyLoad`, the first images keep their spinner** and `onLazyLoadLoaded` doesn't fire for them. Production is fine.
- Sync in an effect, refs and the other event props work as in production.

Production isn't affected. Next.js's App Router turns StrictMode on in development by default (see `references/nextjs.md`).
