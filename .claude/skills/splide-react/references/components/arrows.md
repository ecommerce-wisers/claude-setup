# Arrows component (React)

`splide.Components.Arrows` as reached from React Splide: the arrow buttons, `update()`, arrow events and the custom arrow markup in JSX.

Contents: [Access from React](#access-from-react) · [Properties](#properties) · [Methods](#methods) · [Generated markup](#generated-markup) · [Custom arrows in JSX](#custom-arrows-in-jsx) · [Events](#events-emitted--listened) · [Options read](#options-read) · [React notes](#react-notes)

Source: https://splidejs.com/components/arrows/
Source file: `src/js/components/Arrows/Arrows.ts` (SVG constants in `src/js/components/Arrows/path.ts`) in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: manages the previous/next arrow buttons. It uses arrows found in the slider markup, or generates them when they are missing.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Arrows;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.Arrows.update() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Arrows` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

## Properties

| Name | Type | Description |
|---|---|---|
| `arrows` | `{ prev?: HTMLButtonElement, next?: HTMLButtonElement }` | The previous and next arrow buttons. They are either the buttons from your markup or the ones this component generated. The object is empty when there are no arrows (for example `arrows: false` and no arrows in the markup). |

## Methods

| Signature | Returns | Description |
|---|---|---|
| `update()` | `void` | (undocumented, from source; marked `@internal`) Recomputes `disabled` and `aria-label` on both arrows, then emits `arrows:updated`. |

Lifecycle members `mount()` and `destroy()` (from `BaseComponent`) are also on the object. Splide calls them. Do not call them yourself.

### arrows (property)

- The object is created once and keeps its identity. `mount()` copies `prev` and `next` into it.
- `destroy()` does not clear it. After an `updated` remount that turns arrows off, it can still reference buttons that were removed from the DOM.
- `Elements.prev` / `Elements.next` only hold arrows that were in the markup. Use `Arrows.arrows` to get generated arrows.
- Generated arrows are rebuilt on every `updated` event. In React that is every `options` prop change that is not deep-equal to the previous one (tested: after a `perPage` change the prev button was a new element). Style them in `onArrowsMounted`, which fires again after each rebuild, instead of reading them once.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function ArrowsEventsExample() {
  const ref = useRef<Splide>( null );
  const [ perPage, setPerPage ] = useState( 1 );
  const [ labels, setLabels ] = useState( '' );

  useEffect( () => {
    // Runs after <Splide> mounted, so Components is filled in.
    const arrows = ref.current?.splide?.Components.Arrows.arrows;
    setLabels( `${ arrows?.prev?.getAttribute( 'aria-label' ) } | ${ arrows?.next?.getAttribute( 'aria-label' ) }` );
  }, [] );

  return (
    <>
      <Splide
        ref={ ref }
        options={ { perPage, rewind: true, label: 'Arrows demo' } }
        onArrowsMounted={ ( splide, prev, next ) => {
          // Runs at mount and again after every options change, because generated arrows are rebuilt.
          prev.classList.add( 'my-prev' );
          next.classList.add( 'my-next' );
        } }
      >
        { [ 1, 2, 3, 4 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>

      <p id="labels">{ labels }</p>
      <button type="button" id="per-page" onClick={ () => setPerPage( 2 ) }>Two per page</button>
      <button type="button" id="update" onClick={ () => ref.current?.splide?.Components.Arrows.update() }>
        Update arrows
      </button>
    </>
  );
}
```

Tested: `labels` reads `Go to last slide | Next slide` (with `rewind`, going back from slide 0 wraps), and `my-prev` is on the new button after the `perPage` change.

### update()

Marked internal, but safe to call. The component already calls it on `mounted`, `moved`, `refresh`, `scrolled` and when the end index changes. What it does:

- `prev.disabled = Controller.getPrev() < 0` and `next.disabled = Controller.getNext() < 0`, so a `slide` type carousel without `rewind` disables the arrow at the first/last page.
- `aria-label` of `prev`: `i18n.last` when going back would wrap to a higher index (rewind/loop), otherwise `i18n.prev`.
- `aria-label` of `next`: `i18n.first` when going forward would wrap to a lower index, otherwise `i18n.next`.
- Emits `arrows:updated( prev, next, prevIndex, nextIndex )` (React prop `onArrowsUpdated`).
- Does nothing when either arrow is missing.

```tsx
ref.current?.splide?.Components.Arrows.update();
```

## Generated markup

When arrows are enabled and the markup does not contain both `.splide__arrow--prev` and `.splide__arrow--next`, the component builds:

```html
<div class="splide__arrows splide__arrows--ltr">
  <button class="splide__arrow splide__arrow--prev" type="button" aria-label="Previous slide" aria-controls="splide01-track">
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 40 40" width="40" height="40" focusable="false">
      <path d="m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z"></path>
    </svg>
  </button>
  <button class="splide__arrow splide__arrow--next" type="button" aria-label="Next slide" aria-controls="splide01-track">...</button>
</div>
```

- Class names come from `options.classes.arrows`, `.arrow`, `.prev` and `.next`. The path comes from `options.arrowPath`, or the default above.
- Both buttons get the same right-pointing path. The theme CSS (`@splidejs/react-splide/css`) flips the prev SVG with `transform: scaleX(-1)` (and rotates it for `ttb`).
- With no `.splide__arrows` element in the markup, the wrapper is created and inserted right before `.splide__track`. With an empty `.splide__arrows` element in the markup, the two buttons are appended into it.
- The wrapper always gets the modifier class `splide__arrows--{direction}` (`ltr`, `rtl` or `ttb`).
- These elements are created by Splide, outside React. React never renders or removes them.

## Custom arrows in JSX

Render your own buttons with `hasTrack={ false }` and `<SplideTrack>`:

```tsx
import { useRef } from 'react';
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function CustomArrowsExample() {
  const ref = useRef<Splide>( null );

  return (
    <>
      <Splide ref={ ref } hasTrack={ false } options={ { label: 'Gallery' } }>
        <div className="splide__arrows">
          <button type="button" className="splide__arrow splide__arrow--prev">Prev</button>
          <button type="button" className="splide__arrow splide__arrow--next">Next</button>
        </div>
        <SplideTrack>
          <SplideSlide>1</SplideSlide>
          <SplideSlide>2</SplideSlide>
        </SplideTrack>
      </Splide>

      { /* Outside the root, so Splide does not find it. Wire it yourself. */ }
      <button type="button" id="outside-next" onClick={ () => ref.current?.go( '>' ) }>Next (outside)</button>
    </>
  );
}
```

Tested: Splide adds `aria-label="Previous slide"`, `aria-controls="<root id>-track"`, `disabled` on prev at slide 0, the `splide__arrows--ltr` class, and the click handlers. No arrows are generated.

- Arrows are looked up anywhere inside the root, skipping elements that belong to a nested slider.
- Custom arrows must sit inside a `.splide__arrows` wrapper. `init()` calls `style()` on the wrapper. If you supply both buttons without the wrapper, the wrapper is `undefined` and `mount()` throws a TypeError (tested: `Cannot read properties of undefined (reading 'style')`). In React the throw happens in `componentDidMount`, so without an error boundary the whole tree unmounts.
- Buttons outside the root are not found. Wire those yourself: `onClick={ () => ref.current?.go( '>' ) }` (the React component's `go( control )`), or `ref.current?.splide?.go( '>' )`.
- The Arrows component keeps the button elements it found when it was created, even across `updated` remounts (from source). A button that React renders later, or a replacement element, is not wired. Keep them rendered instead of `{ show && <button ...> }`: hide them with CSS, use the `arrows` option, or remount the carousel with a `key`.
- On destroy (unmount), your buttons lose the attributes Splide added (`role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription`). Generated arrows are removed. Under StrictMode in development the carousel mounts, is destroyed and mounts again, and the second mount adds the attributes back.

## Events emitted / listened

Emitted:

| Event | React prop | Handler arguments (prop) | When |
|---|---|---|---|
| `arrows:mounted` | `onArrowsMounted` | `( splide, prev, next )` | After arrows are set up, only when the `arrows` option is truthy. Fires again after each `updated` remount (every options prop change). |
| `arrows:updated` | `onArrowsUpdated` | `( splide, prev, next, prevIndex, nextIndex )` | Every time `update()` runs. `prevIndex`/`nextIndex` come from `Controller.getPrev()`/`getNext()`; `-1` means there is no slide in that direction. |

Event props are bound before `splide.mount()`, so `onArrowsMounted` fires for the first mount. They are read once: a handler sees the state of the first render (see `references/reactivity.md`).

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `updated` | Full remount (`destroy()` then `mount()`): arrows are regenerated and listeners re-bound. |
| Splide | `mounted`, `moved`, `refresh`, `scrolled`, internal `ei` (end index changed) | `update()`. Bound only when arrows are enabled. |
| DOM | `click` on `prev` | `Controller.go( '<', true )` |
| DOM | `click` on `next` | `Controller.go( '>', true )` |

When `arrows` is `false` and arrows exist in the markup, the wrapper is hidden with `display: none` and no listeners are bound.

## Options read

| Option | Use |
|---|---|
| `arrows` | Enables the arrows. `false` hides markup arrows (`display: none`) and generates none. Can change through the `options` prop (the `updated` remount reads it again). |
| `arrowPath` | SVG `d` attribute for generated arrows (40x40 viewBox). |
| `classes.arrows`, `classes.arrow`, `classes.prev`, `classes.next` | Class names for the generated wrapper and buttons. |
| `i18n.prev`, `i18n.next`, `i18n.first`, `i18n.last` | `aria-label` values. |
| `direction` | Wrapper modifier class `splide__arrows--{direction}`. |
| `type`, `rewind`, `perPage`, `perMove`, `focus` | Indirectly, through `Controller.getPrev()` / `getNext()`, which decide `disabled` and the wrap labels. |

## React notes

- For disabled or hidden arrows driven by state, pass `arrows` in the `options` prop rather than toggling the buttons in JSX. React Splide sets `splide.options`, which remounts this component.
- Keep the `className` of your `.splide__arrows` wrapper constant. Splide adds `splide__arrows--{direction}` to it, and a `className` change makes React rewrite the whole `class` attribute, which drops that class. Generated arrows are not React elements, so this applies only to your own markup.
