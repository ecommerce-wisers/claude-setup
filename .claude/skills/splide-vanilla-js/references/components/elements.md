# Elements component

Source: https://splidejs.com/components/elements/
Source file: `src/js/components/Elements/Elements.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: finds the elements the slider is made of in the HTML, and sets up ids, roles, ARIA attributes and modifier classes on root, track and list.

Access: `splide.Components.Elements` (filled in by `splide.mount()`). Construction and `setup()` run inside `mount()`. A missing track or list throws `[splide] A track/list element is missing.` there.

## Properties

All elements are searched inside the root, ignoring elements that belong to a nested slider (an element counts only if its closest `.splide` ancestor is this root). This check means the root element itself must have the class `splide`. Without it, nothing is found and mount throws the track/list error.

| Name | Type | Description |
|---|---|---|
| `root` | `HTMLElement` | The root element (`splide.root`). |
| `track` | `HTMLElement` | `.splide__track`. |
| `list` | `HTMLElement` | `.splide__list`, a direct child of the track. |
| `slides` | `HTMLElement[]` | `.splide__slide` direct children of the list, excluding clones (`.splide__slide--clone`). |
| `arrows` | `HTMLElement \| undefined` | `.splide__arrows` wrapper from the HTML, if any. |
| `pagination` | `HTMLUListElement \| undefined` | (undocumented, from source) `.splide__pagination` from the HTML, if any. Pagination uses it as a placeholder to fill with dots. A generated pagination list is not stored here. |
| `prev` | `HTMLButtonElement \| undefined` | `.splide__arrow--prev` from the HTML, if any. Generated arrows are not stored here; use `Components.Arrows.arrows` (the docs say `Arrow#arrows`). |
| `next` | `HTMLButtonElement \| undefined` | `.splide__arrow--next` from the HTML, if any. Docs show the signature as `slider: HTMLButtonElement`; source says `next?: HTMLButtonElement`. Generated arrows are not stored here. |
| `bar` | `HTMLElement \| undefined` | `.splide__progress__bar` (autoplay progress bar), if any. Docs show the signature as `slider: HTMLElement`; source says `bar?: HTMLElement`. |
| `toggle` | `HTMLElement \| undefined` | `.splide__toggle` (autoplay play/pause button), if any. |

The docs mark `arrows`, `prev`, `next`, `bar` and `toggle` as possibly `undefined`. In source, all optional members are `?` (`arrows?`, `pagination?`, `prev?`, `next?`, `bar?`, `toggle?`).

Notes:

- These are snapshots taken in `setup()` and taken again on every `refresh` (for example after `splide.add()` / `splide.remove()`). The `slides` array keeps its identity: it is emptied and refilled. The other properties are reassigned, so read `splide.Components.Elements.track` fresh instead of caching it across refreshes.
- `slides` holds the real slides in DOM order at collection time. For Slide objects, including clones, use `Components.Slides`.

```js
const splide = new Splide( '.splide' ).mount();
const { root, track, list, slides } = splide.Components.Elements;

console.log( track.id );        // e.g. 'splide01-track'
console.log( slides.length );   // same as splide.length
```

## Methods

The docs list no methods. The source object has only lifecycle members, which Splide calls. Do not call them yourself.

| Signature | Returns | Description |
|---|---|---|
| `setup()` | `void` | (lifecycle, undocumented, from source) Collects the elements, then sets ids, roles, classes and labels. It runs while the component is created, and again on every `refresh`. |
| `mount()` | `void` | (lifecycle, undocumented, from source) Binds the `refresh`/`updated` handlers and the keyboard-focus tracking. |
| `destroy( completely? )` | `void` | (lifecycle, undocumented, from source) Removes classes and attributes (see below). |

### What setup/mount do to the DOM (from source)

On `setup()`:

- `root.id`: kept if present, otherwise a generated `splide01`, `splide02`, ... (counter per page).
- `track.id`: kept, or `{rootId}-track`. `list.id`: kept, or `{rootId}-list`.
- `role` on root: `options.role` (default `'region'`). Not set if the root already has a `role` attribute, or is a `<section>`, or `role` is falsy.
- `aria-roledescription` on root: `i18n.carousel` (`'carousel'`).
- `role="presentation"` on the list.
- Modifier classes on root: `splide--{type}`, `splide--{direction}`, `splide--draggable` (when `drag` is truthy), `splide--nav` (when `isNavigation`), plus `is-active`.
- Modifier classes on track: `splide__track--{type}`, `splide__track--{direction}`, `splide__track--draggable`, `splide__track--nav`.
- `aria-label` on root from `options.label`. `aria-labelledby` on root from `options.labelledby`. An empty or missing value removes the attribute.

On `mount()`:

- A capture-phase listener on `document` (`touchstart mousedown keydown`) tracks whether the user is using the keyboard.
- On `focusin` in the root, the class `is-focus-in` is added if the last input was a key press, and removed otherwise. Use it for keyboard-only focus styles.

On `destroy( completely )`:

- Removes the modifier classes from root and track.
- Removes `style` and the Splide ARIA attributes (`role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription`) from track and list.
- From root, removes only `style` and `aria-roledescription`, unless `completely` is `true`, in which case the full attribute list is removed from root too.
- Ids are not removed.

## Events emitted / listened

Emitted: none.

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `refresh` | `destroy()` then `setup()`: elements are collected again and attributes re-applied. |
| Splide | `updated` | Re-computes modifier classes and `aria-label`/`aria-labelledby` from the new options. |
| DOM document | `touchstart`, `mousedown`, `keydown` (capture) | Tracks keyboard vs pointer use. |
| DOM root | `focusin` | Toggles `is-focus-in`. |

## Options read

| Option | Use |
|---|---|
| `role` | Root role (default `'region'`). |
| `label`, `labelledby` | Root `aria-label` / `aria-labelledby`. |
| `i18n.carousel` | Root `aria-roledescription`. |
| `type`, `direction`, `drag`, `isNavigation` | Modifier classes on root and track. |
