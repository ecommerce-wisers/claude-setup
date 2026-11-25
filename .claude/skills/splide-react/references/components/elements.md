# Elements component (React)

`splide.Components.Elements` as reached from React Splide: the root, track, list, slides and optional control elements, and what Splide writes onto the markup React rendered.

Contents: [Access from React](#access-from-react) · [Properties](#properties) · [Methods](#methods) · [What setup/mount do to the DOM](#what-setupmount-do-to-the-dom-from-source) · [Splide's classes and React's className](#splides-classes-and-reacts-classname) · [Events](#events-emitted--listened) · [Options read](#options-read)

Source: https://splidejs.com/components/elements/
Source file: `src/js/components/Elements/Elements.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

Purpose: finds the elements the slider is made of in the markup, and sets up ids, roles, ARIA attributes and modifier classes on root, track and list.

## Access from React

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Elements;
ref.current?.splideRef.current;       // the root element, without the core instance
```

- In an event prop, the core instance is the first argument: `onMounted={ ( splide ) => splide.Components.Elements.track }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render. For the root element alone, `splideRef` is set as soon as React attaches refs.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first. Construction and `setup()` run inside `mount()`. A missing track or list throws `[splide] A track/list element is missing.` there, which in React is an error thrown from `componentDidMount`.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.Elements` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested; see `references/components/other-components.md`, Media).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

With the default `hasTrack`, `<Splide>` renders the root (`tag`, default `div`, with class `splide`), and `<SplideTrack>` inside it renders `div.splide__track > ul.splide__list`. `<SplideSlide>` renders `li.splide__slide`. With `hasTrack={ false }` you render `<SplideTrack>` yourself, next to arrows, pagination, a progress bar or a toggle.

## Properties

All elements are searched inside the root, ignoring elements that belong to a nested slider (an element counts only if its closest `.splide` ancestor is this root). This check means the root element itself must have the class `splide`. `<Splide>` always adds it (`classNames( 'splide', className )`).

| Name | Type | Description |
|---|---|---|
| `root` | `HTMLElement` | The root element (`splide.root`, the same node as `ref.current.splideRef.current`). |
| `track` | `HTMLElement` | `.splide__track`. |
| `list` | `HTMLElement` | `.splide__list`, a direct child of the track. |
| `slides` | `HTMLElement[]` | `.splide__slide` direct children of the list, excluding clones (`.splide__slide--clone`). |
| `arrows` | `HTMLElement \| undefined` | `.splide__arrows` wrapper from the markup, if any. |
| `pagination` | `HTMLUListElement \| undefined` | (undocumented, from source) `.splide__pagination` from the markup, if any. Pagination uses it as a placeholder to fill with dots. A generated pagination list is not stored here. |
| `prev` | `HTMLButtonElement \| undefined` | `.splide__arrow--prev` from the markup, if any. Generated arrows are not stored here; use `Components.Arrows.arrows` (the docs say `Arrow#arrows`). |
| `next` | `HTMLButtonElement \| undefined` | `.splide__arrow--next` from the markup, if any. Docs show the signature as `slider: HTMLButtonElement`; source says `next?: HTMLButtonElement`. Generated arrows are not stored here. |
| `bar` | `HTMLElement \| undefined` | `.splide__progress__bar` (autoplay progress bar), if any. Docs show the signature as `slider: HTMLElement`; source says `bar?: HTMLElement`. |
| `toggle` | `HTMLElement \| undefined` | `.splide__toggle` (autoplay play/pause button), if any. |

The docs mark `arrows`, `prev`, `next`, `bar` and `toggle` as possibly `undefined`. In source, all optional members are `?` (`arrows?`, `pagination?`, `prev?`, `next?`, `bar?`, `toggle?`).

Notes:

- These are snapshots taken in `setup()` and taken again on every `refresh`. In React, `refresh` runs when the `<SplideSlide>` children change (React Splide calls `splide.refresh()`). The `slides` array keeps its identity: it is emptied and refilled. The other properties are reassigned, so read `splide.Components.Elements.track` fresh instead of caching it across refreshes.
- `slides` holds the real slides in DOM order at collection time. For Slide objects, including clones, use `Components.Slides`.
- All of these are DOM nodes, not React state. Reading them never re-renders anything.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function ElementsExample() {
  const ref = useRef<Splide>( null );
  const [ dark, setDark ] = useState( false );
  const [ info, setInfo ] = useState( '' );

  useEffect( () => {
    const splide = ref.current?.splide;

    if ( splide ) {
      const { root, track, list, slides } = splide.Components.Elements;
      const sameRoot = root === ref.current?.splideRef.current;
      setInfo( `${ track.id } ${ list.id } ${ slides.length } ${ sameRoot }` );
    }
  }, [] );

  return (
    // Toggle classes on a wrapper. A className change on <Splide> itself would wipe Splide's classes.
    <div className={ dark ? 'theme-dark' : 'theme-light' }>
      <Splide ref={ ref } tag="section" className="my-carousel" options={ { label: 'Elements demo' } }>
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <p id="info">{ info }</p>
      <button type="button" id="theme" onClick={ () => setDark( d => ! d ) }>Toggle theme</button>
    </div>
  );
}
```

Tested: `info` reads `splide01-track splide01-list 3 true` (also under StrictMode, because the second mount keeps the id the first one set), the `<section>` root gets no `role` but does get `aria-roledescription="carousel"`, and toggling the theme keeps `is-initialized`.

## Methods

The docs list no methods. The source object has only lifecycle members, which Splide calls. Do not call them yourself.

| Signature | Returns | Description |
|---|---|---|
| `setup()` | `void` | (lifecycle, undocumented, from source) Collects the elements, then sets ids, roles, classes and labels. It runs while the component is created, and again on every `refresh`. |
| `mount()` | `void` | (lifecycle, undocumented, from source) Binds the `refresh`/`updated` handlers and the keyboard-focus tracking. |
| `destroy( completely? )` | `void` | (lifecycle, undocumented, from source) Removes classes and attributes (see below). |

## What setup/mount do to the DOM (from source)

On `setup()`:

- `root.id`: kept if present, otherwise a generated `splide01`, `splide02`, ... (counter per page). Pass an `id` prop on `<Splide>` for a stable id; it is spread onto the root (tested: `id="main"` gave slide ids `main-slide01`, ...).
- `track.id`: kept, or `{rootId}-track`. `list.id`: kept, or `{rootId}-list`.
- `role` on root: `options.role` (default `'region'`). Not set if the root already has a `role` attribute, or is a `<section>` (`tag="section"`), or `role` is falsy.
- `aria-roledescription` on root: `i18n.carousel` (`'carousel'`).
- `role="presentation"` on the list.
- Modifier classes on root: `splide--{type}`, `splide--{direction}`, `splide--draggable` (when `drag` is truthy), `splide--nav` (when `isNavigation`), plus `is-active`.
- Modifier classes on track: `splide__track--{type}`, `splide__track--{direction}`, `splide__track--draggable`, `splide__track--nav`.
- `aria-label` on root from `options.label`. `aria-labelledby` on root from `options.labelledby`. An empty or missing value removes the attribute. Prefer `options={ { label: '...' } }` over an `aria-label` prop: under StrictMode in development, the destroy between the two mounts strips an `aria-label` prop and React does not put it back, while `options.label` is applied again by the second mount.

On `mount()`:

- A capture-phase listener on `document` (`touchstart mousedown keydown`) tracks whether the user is using the keyboard.
- On `focusin` in the root, the class `is-focus-in` is added if the last input was a key press, and removed otherwise. Use it for keyboard-only focus styles.

On `destroy( completely )` (React Splide always destroys completely on unmount):

- Removes the modifier classes from root and track.
- Removes `style` and the Splide ARIA attributes (`role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription`) from track and list.
- From root, removes only `style` and `aria-roledescription`, unless `completely` is `true`, in which case the full attribute list is removed from root too. A `destroy: true` breakpoint destroys with `completely` `false`; `destroy: 'completely'` and unmounting use `true`.
- Ids are not removed.

## Splide's classes and React's className

Splide adds classes to elements React rendered: the root (`is-initialized`, `is-active`, `is-overflow`, `splide--{type}`, ...), the track, and each `<li>` (`is-active`, `is-visible`, `is-prev`, `is-next`). React does not know about them. When a `className` prop changes after mount, React writes the whole `class` attribute again and they are gone.

Tested: toggling `className` on `<Splide>` from `light` to `dark` left `class="splide dark"`, and the core CSS (`.splide { visibility: hidden }` until `is-initialized`) hid the whole carousel. Toggling `className` on a `<SplideSlide>` dropped its `is-active` and `is-visible`.

- Keep `className` on `<Splide>`, `<SplideTrack>` and `<SplideSlide>` constant. Put changing classes on a wrapper element (as in the example above) or use a `data-*` attribute.
- To style the active slide, use Splide's own classes in CSS (`.splide__slide.is-active`) instead of a React `className` computed from the index.
- `style` props are safer: React updates only the style properties that changed, so Splide's inline `width`, `height`, margin and `transform` survive unless you set the same property.

## Events emitted / listened

Emitted: none.

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `refresh` | `destroy()` then `setup()`: elements are collected again and attributes re-applied. |
| Splide | `updated` | Re-computes modifier classes and `aria-label`/`aria-labelledby` from the new options. |
| DOM document | `touchstart`, `mousedown`, `keydown` (capture) | Tracks keyboard vs pointer use. |
| DOM root | `focusin` | Toggles `is-focus-in`. |

An `onRefresh` prop runs before this component's `refresh` listener (event props are bound before mount), so inside `onRefresh` these properties still describe the old slides. Read them in a `useEffect` after the change instead (see `references/components/slides.md`).

## Options read

| Option | Use |
|---|---|
| `role` | Root role (default `'region'`). |
| `label`, `labelledby` | Root `aria-label` / `aria-labelledby`. Re-applied on every `updated`, so they can change through the `options` prop. |
| `i18n.carousel` | Root `aria-roledescription`. |
| `type`, `direction`, `drag`, `isNavigation` | Modifier classes on root and track. |
