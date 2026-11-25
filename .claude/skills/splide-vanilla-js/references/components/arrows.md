# Arrows component

Source: https://splidejs.com/components/arrows/
Source file: `src/js/components/Arrows/Arrows.ts` (SVG constants in `src/js/components/Arrows/path.ts`), Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner, so everything here applies to 4.1.4.

Purpose: manages the previous/next arrow buttons. It uses arrows found in the slider HTML, or generates them when they are missing.

Access: `splide.Components.Arrows` (filled in by `splide.mount()`; `splide.Components` is an empty object before mount).

## Properties

| Name | Type | Description |
|---|---|---|
| `arrows` | `{ prev?: HTMLButtonElement, next?: HTMLButtonElement }` | The previous and next arrow buttons. They are either the buttons from your HTML or the ones this component generated. The object is empty when there are no arrows (for example `arrows: false` and no arrows in the HTML). |

## Methods

| Signature | Returns | Description |
|---|---|---|
| `update()` | `void` | (undocumented, from source; marked `@internal`) Recomputes `disabled` and `aria-label` on both arrows, then emits `arrows:updated`. |

Lifecycle members `mount()` and `destroy()` (from `BaseComponent`) are also on the object. Splide calls them. Do not call them yourself.

### arrows (property)

- The object is created once and keeps its identity. `mount()` copies `prev` and `next` into it.
- `destroy()` does not clear it. After an `updated` remount that turns arrows off, it can still reference buttons that were removed from the DOM.
- `Elements.prev` / `Elements.next` only hold arrows that were in the HTML. Use `Arrows.arrows` to get generated arrows.

```js
const splide = new Splide( '.splide' );

// Generated arrows are rebuilt on every 'updated' event, so hook the event instead of reading once:
splide.on( 'arrows:mounted', function ( prev, next ) {
  prev.classList.add( 'my-prev' );
  next.classList.add( 'my-next' );
} );

splide.mount();

const arrows = splide.Components.Arrows.arrows;
console.log( arrows.prev, arrows.next );
```

### update()

Marked internal, but safe to call. The component already calls it on `mounted`, `moved`, `refresh`, `scrolled` and when the end index changes. What it does:

- `prev.disabled = Controller.getPrev() < 0` and `next.disabled = Controller.getNext() < 0`, so a `slide` type carousel without `rewind` disables the arrow at the first/last page.
- `aria-label` of `prev`: `i18n.last` when going back would wrap to a higher index (rewind/loop), otherwise `i18n.prev`.
- `aria-label` of `next`: `i18n.first` when going forward would wrap to a lower index, otherwise `i18n.next`.
- Emits `arrows:updated( prev, next, prevIndex, nextIndex )`.
- Does nothing when either arrow is missing.

```js
splide.Components.Arrows.update();
```

### Generated markup

When arrows are enabled and the HTML does not contain both `.splide__arrow--prev` and `.splide__arrow--next`, the component builds:

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
- Both buttons get the same right-pointing path. The theme CSS (`splide.min.css`) flips the prev SVG with `transform: scaleX(-1)` (and rotates it for `ttb`).
- With no `.splide__arrows` element in the HTML, the wrapper is created and inserted right before `.splide__track`. With an empty `.splide__arrows` element in the HTML, the two buttons are appended into it.
- The wrapper always gets the modifier class `splide__arrows--{direction}` (`ltr`, `rtl` or `ttb`).

### Custom arrows in HTML

```html
<section class="splide" aria-label="Gallery">
  <div class="splide__arrows">
    <button class="splide__arrow splide__arrow--prev" type="button">Prev</button>
    <button class="splide__arrow splide__arrow--next" type="button">Next</button>
  </div>
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">1</li>
      <li class="splide__slide">2</li>
    </ul>
  </div>
</section>
```

- Arrows are looked up anywhere inside the root, skipping elements that belong to a nested slider.
- Custom arrows must sit inside a `.splide__arrows` wrapper. `init()` calls `style()` on the wrapper. If you supply both buttons without the wrapper, the wrapper is `undefined` and `mount()` throws a TypeError.
- Buttons outside the root are not found. Wire those yourself: `button.addEventListener( 'click', function () { splide.go( '>' ); } )`.
- On destroy, user-provided arrows are kept but lose the attributes Splide added (`role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription`). Generated arrows are removed.

## Events emitted / listened

Emitted (listen with `splide.on( name, handler )`):

| Event | Handler arguments | When |
|---|---|---|
| `arrows:mounted` | `( prev, next )` | After arrows are set up, only when the `arrows` option is truthy. Fires again after each `updated` remount. |
| `arrows:updated` | `( prev, next, prevIndex, nextIndex )` | Every time `update()` runs. `prevIndex`/`nextIndex` come from `Controller.getPrev()`/`getNext()`; `-1` means there is no slide in that direction. |

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `updated` | Full remount (`destroy()` then `mount()`): arrows are regenerated and listeners re-bound. |
| Splide | `mounted`, `moved`, `refresh`, `scrolled`, internal `ei` (end index changed) | `update()`. Bound only when arrows are enabled. |
| DOM | `click` on `prev` | `Controller.go( '<', true )` |
| DOM | `click` on `next` | `Controller.go( '>', true )` |

When `arrows` is `false` and arrows exist in the HTML, the wrapper is hidden with `display: none` and no listeners are bound.

## Options read

| Option | Use |
|---|---|
| `arrows` | Enables the arrows. `false` hides HTML arrows (`display: none`) and generates none. |
| `arrowPath` | SVG `d` attribute for generated arrows (40x40 viewBox). |
| `classes.arrows`, `classes.arrow`, `classes.prev`, `classes.next` | Class names for the generated wrapper and buttons. |
| `i18n.prev`, `i18n.next`, `i18n.first`, `i18n.last` | `aria-label` values. |
| `direction` | Wrapper modifier class `splide__arrows--{direction}`. |
| `type`, `rewind`, `perPage`, `perMove`, `focus` | Indirectly, through `Controller.getPrev()` / `getNext()`, which decide `disabled` and the wrap labels. |
