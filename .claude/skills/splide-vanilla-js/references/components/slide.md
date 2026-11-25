# Slide component

Source: https://splidejs.com/components/slide/
Source file: `src/js/components/Slides/Slide.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: the Slide sub-component. Each slide element, clones included, gets one Slide object, which manages that slide's status classes, ARIA attributes and events.

Access: there is no `splide.Components.Slide`. Get Slide objects from the Slides component: `splide.Components.Slides.getAt( i )`, `.get()`, `.getIn( page )`, `.filter( matcher )`, `.forEach( fn )`. Slide objects are also passed to the `active`, `inactive`, `visible`, `hidden`, `click` and `lazyload:loaded` event handlers.

## Properties

| Name | Type | Description |
|---|---|---|
| `index` | `number` | The slide index. For clones it is negative (head clones: `-count .. -1`) or `>= length` (tail clones: `length .. length + count - 1`). |
| `slideIndex` | `number` | For a clone, the index of the original slide it copies. `-1` for normal slides. |
| `slide` | `HTMLElement` | The slide element. |
| `container` | `HTMLElement \| undefined` | The `.splide__slide__container` direct child, if any, otherwise `undefined`. |
| `isClone` | `boolean` | `true` if this Slide is a clone (`slideIndex > -1`). |

Properties are fixed when the object is created. Slide objects are destroyed and recreated on every `refresh` (`splide.add()`, `splide.remove()`, `splide.refresh()`, clone count changes), so do not keep references across refreshes.

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

```js
const Slide = splide.Components.Slides.getAt( 0 );
Slide.style( 'opacity', 0.5 );
Slide.style( 'background', 'center/cover url("a.jpg")', true );
```

Layout re-applies `width`, `height` and the gap margin on resize and option updates. Styles you set on those properties get overwritten.

### isWithin()

`isWithin( from: number, distance: number ): boolean`

| Param | Description |
|---|---|
| `from` | The index of a base slide. |
| `distance` | The distance to test. The method returns `true` if the slide is within this number. |

Returns `true` if `abs( from - index ) <= distance`. For non-clone slides with `rewind` or the `loop` type, the distance wraps around: `min( diff, length - diff )`. LazyLoad uses this to decide which images to load.

```js
// Mark slides within 1 of the active slide:
splide.on( 'moved', function ( index ) {
  splide.Components.Slides.forEach( function ( Slide ) {
    Slide.slide.classList.toggle( 'is-near', Slide.isWithin( index, 1 ) );
  } );
} );
```

### update()

(undocumented, from source) `update(): void`

It runs automatically on `moved`, `scrolled` and the internal `sh` (loop shift) event, and also on `move` when `updateOnMove` is `true`. `Slides.update()` calls it for every slide. It does nothing after the Slide is destroyed. What it updates:

- `is-active`: when `splide.index === index`, or (with `cloneStatus`, default `true`) when `splide.index === slideIndex` for clones. On a change, `aria-current="true"` is set when `isNavigation` is on, and `active( Slide )` or `inactive( Slide )` is emitted.
- `is-visible`: when the slide's rect lies fully inside the track's rect along the axis. For `fade`, visible equals active. On a change, `visible( Slide )` or `hidden( Slide )` is emitted.
- `aria-hidden="true"` on slides that are not visible, except the active slide when it is not a clone (code: `hidden = ! visible && ( ! isActive() || isClone )`). Visible slides never get it. It is not changed while moving or scrolling, to avoid screen readers reading mid-transition.
- `tabindex="-1"` on focusable descendants (`focusableNodes` option) of hidden slides, removed otherwise. With `slideFocus`, the slide itself gets `tabindex="0"` or `-1`.
- `is-prev` / `is-next` when `index === splide.index - 1` / `+ 1`.
- If a slide that has focus becomes not visible, focus moves to the active slide.

## Attributes set on mount (non-clones)

| Attribute | Value |
|---|---|
| `id` | `{rootId}-slide01`, `-slide02`, ... (1-based, zero-padded to 2 digits). Clones get `{rootId}-clone01`, ... from the Clones component. |
| `role` | `'tabpanel'` when `pagination` is on, otherwise `'group'`. |
| `aria-roledescription` | `i18n.slide` (`'slide'`). |
| `aria-label` | The existing `aria-label` of the element, else `i18n.slideLabel` (`'%s of %s'`), e.g. `'1 of 5'`. |

When the slider is a navigation (`isNavigation`, on `navigation:mounted`):

- `aria-label` becomes `i18n.slideX` (`'Go to slide %s'`).
- `aria-controls` lists the ids of the matching slides in the synced sliders.
- `role="button"` is set when `slideFocus` is on, and `aria-roledescription` is removed.

## Events emitted / listened

Emitted (handlers receive the Slide object):

| Event | Arguments | When |
|---|---|---|
| `active` | `( Slide )` | The slide becomes active. |
| `inactive` | `( Slide )` | The slide stops being active. |
| `visible` | `( Slide )` | The slide becomes fully visible in the track. |
| `hidden` | `( Slide )` | The slide stops being fully visible. |
| `click` | `( Slide, e )` | DOM `click` on the slide element (also for clones). |
| `sk` (internal slide-keydown event) | `( Slide, e )` | DOM `keydown` on the slide. Sync uses it for navigation sliders. |

```js
splide.on( 'active', function ( Slide ) {
  console.log( 'active:', Slide.index, Slide.slide );
} );

splide.on( 'click', function ( Slide, e ) {
  console.log( 'clicked slide', Slide.isClone ? Slide.slideIndex : Slide.index );
} );
```

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

`isNavigation`, `updateOnMove`, `i18n`, `pagination` and `slideFocus` are read once, when the Slide object is created (at mount and on each `refresh`). `cloneStatus`, `focusableNodes`, `rewind` and `type` are read at call time.
