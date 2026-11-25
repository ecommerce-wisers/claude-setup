# About Version 4 (migrating from v3)

Source: https://splidejs.com/guides/version4/
Also compared: the v3.6.12 build (last v3) against the v4.0.0, v4.0.17, v4.1.0 and v4.1.4 builds and type definitions from jsDelivr, plus the v4.1.3 source. Items marked "(not in docs)" come from that comparison, not from the guide page.

## Contents

- [What v4 changed](#what-v4-changed)
- [Migration checklist](#migration-checklist)
- [Options](#options)
- [Breakpoints now behave like media queries](#breakpoints-now-behave-like-media-queries)
- [Autoplay play/pause button](#autoplay-playpause-button)
- [Slider element removed](#slider-element-removed)
- [Markup and ARIA](#markup-and-aria)
- [Classes and CSS selectors](#classes-and-css-selectors)
- [Events](#events)
- [Components and exports](#components-and-exports)
- [Changes inside v4 (4.0 to 4.1)](#changes-inside-v4-40-to-41)
- [Framework integrations](#framework-integrations)

---

## What v4 changed

The v4 release summary from the docs:

- Accessibility revised to follow the W3C [Carousel Design Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/carousel/).
- A [Live Region](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions) for screen-reader users. Screen readers announce the content of the newly visible slide every time the carousel moves. (The page shows an NVDA demo video.)
- Reduced-motion mode. When `prefers-reduced-motion: reduce` matches, non-essential transitions are disabled ([WCAG 2.3.3](https://www.w3.org/WAI/WCAG21/Understanding/animation-from-interactions.html)). Docs write the media value as "`reduced`". The source query is `(prefers-reduced-motion: reduce)`.
- A toggle (combined play/pause) button for autoplay.
- Breakpoints behave like CSS media queries, so window resize and page load give the same carousel.
- Drag Free Snap mode (`drag: 'free'` + `snap: true`).
- The `direction` option is responsive (it can change inside `breakpoints`).
- Options to control wheel sensitivity for inertia scroll (`wheelMinThreshold`, `wheelSleep`).
- `:focus-visible` patched for IE and Safari, and focus styles added to the themes.
- Bug fixes.
- Code size grew from 27 KB to 29 KB.

Breaking changes, as the docs list them:

- New default behavior for `keyboard`, `waitForTransition` and `slideFocus`.
- Separate play/pause buttons for autoplay are no longer supported. Use a toggle (combined play/pause) button.
- The `splide__slider` element is no longer supported, which removes `arrows: 'slider'` and `pagination: 'slider'`.
- Some methods in Components changed.

Also, for new features:

- Translate the strings newly added to i18n.
- Set `aria-label` or `aria-labelledby` on the root element where needed.

---

## Migration checklist

Work through this list for any v3 page or snippet.

1. **CDN URLs**: change `@splidejs/splide@3.x.x` to `@4.1.4` in every `<script>` and `<link>`. The file names didn't change: `dist/js/splide.min.js`, `dist/css/splide.min.css`, `dist/css/themes/splide-<theme>.min.css`, `dist/css/splide-core.min.css`.
2. **Label every root**: add `aria-label="..."` or `aria-labelledby="heading-id"` to `.splide` (or pass the `label` / `labelledby` options). Use `<section>` for main content, or `<div role="group">` for decorative carousels.
3. **`keyboard`**: v3 defaulted to `true` (global keydown). If you relied on arrow keys, set `keyboard: true` (works when focus is inside the carousel) or `keyboard: 'global'` (discouraged). If you used `'focused'`, v4 no longer adds `tabindex="0"` to the root, so add it in your HTML if you still want the root focusable.
4. **`waitForTransition`**: v3 defaulted to `true`. Set `waitForTransition: true` if you need input ignored during transitions.
5. **`slideFocus`**: v3 defaulted to `true`. v4 is `false` for normal carousels and `true` when `isNavigation` is on. Set it explicitly if you depended on focusable slides.
6. **`arrows: 'slider'` / `pagination: 'slider'`**: change them to `true` (or remove them), and replace `<div class="splide__slider">` with `<div style="position: relative">`, or keep the element and add `.splide__slider { position: relative; }`.
7. **Autoplay buttons**: replace `div.splide__autoplay > button.splide__play + button.splide__pause` with `button.splide__toggle[type=button] > span.splide__toggle__play + span.splide__toggle__pause`. Or keep separate buttons and wire them to `Autoplay.play()` / `Autoplay.pause()` (snippet below).
8. **i18n**: translate the new keys `carousel`, `slide`, `slideLabel` and (not in docs) `select`.
9. **Breakpoints**: check that each breakpoint gives the intended result now that every matching breakpoint is merged. A key set only inside a breakpoint resets to its default outside it.
10. **Custom CSS**: v3 modifier selectors such as `.splide--fade > .splide__track > .splide__list > .splide__slide` should target the new track modifiers (`.splide__track--fade > .splide__list > .splide__slide`), because v4's core CSS uses those (not in docs).
11. **Events** (not in docs): `repositioned` no longer exists. Since v4.1.0, `shifted` and `slide:keydown` are internal (`'sh'`, `'sk'`), so listeners using the old names never fire.
12. **Component calls**: `Components.Move.isBusy()` becomes `Components.Controller.isBusy()`. Check the parameters of any `Controller.scroll()` / `Scroll.scroll()` call (table below).
13. **Extensions** (not in docs): `EventBus` is no longer exported. Use `EventInterface( Splide )` from the ESM build, or `Splide.on` / `Splide.off` with namespaces in the classic build. `Components.Elements` no longer has `slider`, `autoplay`, `play` or `pause`. Use `toggle` instead.
14. **Reduced motion**: when the user prefers reduced motion, `speed` and `rewindSpeed` become `0` and autoplay starts paused. Make sure the carousel can still be operated (arrows or pagination), not only by autoplay.

---

## Options

### Defaults that changed

| Option | v3 default | v4 default | Notes |
|---|---|---|---|
| `keyboard` | `true` | `false` | v4 never adds `tabindex="0"` to the root (adding it to non-interactive elements is discouraged). Add it manually if needed. |
| `waitForTransition` | `true` | `false` | With two synced carousels at different speeds, ignoring input during transitions could desynchronize them. Even when `false`, Splide still waits for the transition at the loop points. |
| `slideFocus` | `true` | `false` for normal carousels, `true` when `isNavigation` is `true` | Slides in a normal carousel don't need to be focusable after the other a11y improvements. Thumbnail slides still do, because clickable elements must be focusable. In source, `DEFAULTS` has no `slideFocus`. The Sync component sets it to `isNavigation` when it's `undefined`. |
| `arrows` | `true` (type `boolean \| 'slider'`) | `true` (type `boolean`) | `'slider'` removed |
| `pagination` | `true` (type `boolean \| 'slider'`) | `true` (type `boolean`) | `'slider'` removed |

`keyboard` values in v4:

| Value | Behavior |
|---|---|
| `true` | Shortcuts work only while the carousel contains focus (listens to `keydown` on the root) |
| `false` | Shortcuts disabled (default) |
| `'global'` | Shortcuts work globally, listening to `keydown` on `window` (discouraged) |
| `'focused'` | Same as `true` |

In v3, `true`/`'global'` listened on `window`, and `'focused'` listened on the root and added `tabindex="0"` to it. The v4 source still carries that old doc comment on `keyboard` in `types/options.ts`. The code (`Keyboard.ts`) behaves as in the table above.

The docs spell the trigger option as `isNavitation` in the `slideFocus` section. The option is `isNavigation`.

### New options in v4 (not in docs as a list; from type definitions)

| Option | Type | Default | Since | Description |
|---|---|---|---|---|
| `role` | `string` | `'region'` | 4.0.0 | `role` set on the root. Not set when the root is a `<section>` or already has a `role` attribute. |
| `label` | `string` | the root's `aria-label` or `''` | 4.0.0 | Sets the root's `aria-label`. Use `labelledby` instead when there is a visible label. |
| `labelledby` | `string` | the root's `aria-labelledby` or `''` | 4.0.0 | ID of the labelling element. Sets the root's `aria-labelledby`. |
| `live` | `boolean` | `true` | 4.0.0 | Enables the live region (`aria-live` on the track). Never active with `isNavigation`. |
| `reducedMotion` | `Options` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | 4.0.0 | Options applied while `(prefers-reduced-motion: reduce)` matches |
| `paginationKeyboard` | `boolean` | `true` | 4.0.0 | Arrow keys, Home and End when pagination has focus |
| `paginationDirection` | `'ltr' \| 'rtl' \| 'ttb'` | value of `direction` | 4.0.0 | Pagination direction for key handling and `aria-orientation` |
| `rewindByDrag` | `boolean` | `undefined` (off) | 4.0.0 | Allows rewinding by drag when `rewind` is on |
| `snap` | `boolean` | `undefined` (off) | 4.0.0 | Snaps to the closest slide in drag-free mode (`drag: 'free'`) |
| `wheelMinThreshold` | `number` | `undefined` | 4.0.0 | Wheel delta threshold that cuts off small inertia-scroll deltas |
| `wheelSleep` | `number` (ms) | `undefined` | 4.0.0 | Time before the next wheel input is accepted. The timer starts when the transition begins. |
| `omitEnd` | `boolean` | `undefined` (off) | 4.1.0 | With `focus` set: disables the next arrow at the last page and omits redundant pagination dots |

The other v3 option names still exist in v4: `type`, `rewind`, `speed`, `rewindSpeed`, `width`, `height`, `fixedWidth`, `fixedHeight`, `heightRatio`, `autoWidth`, `autoHeight`, `start`, `perPage`, `perMove`, `clones`, `cloneStatus`, `focus`, `gap`, `padding`, `arrows`, `arrowPath`, `pagination`, `easing`, `easingFunc`, `drag`, `dragMinThreshold`, `flickPower`, `flickMaxPages`, `waitForTransition`, `autoplay`, `interval`, `pauseOnHover`, `pauseOnFocus`, `resetProgress`, `lazyLoad`, `preloadPages`, `keyboard`, `wheel`, `releaseWheel`, `direction`, `cover`, `slideFocus`, `isNavigation`, `trimSpace`, `updateOnMove`, `mediaQuery`, `focusableNodes`, `noDrag`, `useScroll`, `breakpoints`, `classes`, `i18n`, `destroy`. No option was renamed or removed. Only the `'slider'` values were removed.

### i18n keys added in v4

| Key | Default | Used for |
|---|---|---|
| `carousel` | `'carousel'` | Root `aria-roledescription` |
| `slide` | `'slide'` | Slide `aria-roledescription` |
| `slideLabel` | `'%s of %s'` | Slide `aria-label` (`{slide number} of {slide count}`) |
| `select` | `'Select a slide to show'` | Pagination `aria-label`. **Not in the docs' list.** |

Unchanged keys: `prev` `'Previous slide'`, `next` `'Next slide'`, `first` `'Go to first slide'`, `last` `'Go to last slide'`, `slideX` `'Go to slide %s'`, `pageX` `'Go to page %s'`, `play` `'Start autoplay'`, `pause` `'Pause autoplay'`. See `i18n.md`.

---

## Breakpoints now behave like media queries

In v3, Splide **updated the current options** with the options of the breakpoint that matched, so resizing and reloading could give different results.

```js
{
  perPage    : 3,
  gap        : '1rem',
  breakpoints: {
    1200: { perPage: 2, gap: '1rem' },
    640 : { gap: 0 },
  },
}
```

In v3, opening the page on a desktop (> 1200px) shows 3 slides. Resizing down to 640px or below shows 2 slides, because the window passed through `1200: { perPage: 2 }`. Opening the page directly on a mobile (≤ 640px) showed 3 slides instead of 2. The v3 workaround was to repeat every option in both the base options and each breakpoint:

```js
{
  perPage    : 3,
  gap        : '1rem',
  breakpoints: {
    1200: { perPage: 2, gap: '1rem' },
    640 : { perPage: 2, gap: 0 },
  },
}
```

v4 changed this because people were confused by the difference from CSS, because it guarantees the same result on resize and on refresh, and because it makes clear which options apply at a given width.

In v4, Splide merges the options of **all** breakpoints that match the current width into the base options. Opening this on a mobile (≤ 640px):

```js
{
  arrows : true,
  perPage: 3,
  breakpoints: {
    1200: { arrows: false },
    800 : { perPage: 2 },
    640 : {},
  },
}
```

gives 2 slides and no arrows, because `1200` and `800` also match.

Values reset to their default when the base options don't contain the key:

```js
{
  breakpoints: {
    1200: { fixedWidth: 200 },
  },
}
```

Above 1200px, the width goes back to the default (`undefined`).

v3 breakpoints were written the stricter way, so they keep working in v4. Check your carousel anyway.

How the source does it (`Media.ts`): breakpoints are sorted (descending for `mediaQuery: 'max'`, ascending for `'min'`). Every matching entry is merged in order and set as own properties on top of the base options (the prototype), and on every change the old own properties are cleared first. `reducedMotion` is registered as one more media query, after the breakpoints. If `direction` changes, the carousel refreshes. If the merged options contain `destroy`, the carousel is destroyed (`'completely'` = no remount).

---

## Autoplay play/pause button

Users prefer one combined play/pause button to separate buttons, so v4 dropped the separate buttons and supports a single toggle.

```diff
<div class="splide">
  ...

- <div class="splide__autoplay">
-   <button class="splide__play">Play</button>
-   <button class="splide__pause">Pause</button>
- </div>

+ <button class="splide__toggle" type="button">
+   <span class="splide__toggle__play">Play</span>
+   <span class="splide__toggle__pause">Pause</span>
+ </button>
</div>
```

To keep using separate buttons (classic script, global `Splide`):

```js
var splide   = new Splide( '.splide', { autoplay: true } ).mount();
var Autoplay = splide.Components.Autoplay;
var play     = splide.root.querySelector( '.splide__play' );
var pause    = splide.root.querySelector( '.splide__pause' );

if ( play ) {
  play.addEventListener( 'click', function () {
    Autoplay.play();
  } );
}

if ( pause ) {
  pause.addEventListener( 'click', function () {
    Autoplay.pause();
  } );
}
```

Splide no longer styles or labels these buttons. Give them `type="button"` and accessible text yourself, and hide the inactive one with the `autoplay:play` / `autoplay:pause` events if you want toggle-like behavior.

---

## Slider element removed

`<div class="splide__slider">` existed to wrap the track in a positioned element. v4 relaxed the structure rules, so it's no longer needed (see `structure.md`).

To get the same result:

1. Change `arrows` and/or `pagination` from `'slider'` to `true`, or remove them, since `true` is the default.

   ```diff
   {
   - arrows: 'slider',
   + arrows: true, // or remove this

   - pagination: 'slider',
   + pagination: true, // or remove this
   }
   ```

2. Replace the slider element with `<div style="position: relative">`, or give it a class that sets `position: relative`.

   ```html
   <div class="splide">
     <div style="position: relative">
       <div class="splide__track">
         <div class="splide__list">...</div>
       </div>
     </div>
   </div>
   ```

Or keep the slider element and add this CSS:

```css
.splide__slider {
  position: relative;
}
```

v4 behavior with a leftover `.splide__slider` (from source): Splide ignores the class. Generated arrows are inserted immediately before the track, so they land inside the wrapper. Generated pagination is appended to the track's parent element, so it also lands inside the wrapper. The Sea Green theme still has `.splide__slider > .splide__arrows` rules (see `themes.md`).

---

## Markup and ARIA

From the docs and the source:

| Area | v3 | v4 |
|---|---|---|
| Track position | Direct child of the root or of `.splide__slider` | Anywhere inside the root (not inside a nested carousel). The list must still be a direct child of the track, and slides direct children of the list. |
| Root label | Not used | Provide `aria-label` / `aria-labelledby` (or the `label` / `labelledby` options) |
| Root `role` | none | `role="region"` added unless the root is a `<section>` or already has a `role`. Use `role="group"` for non-essential carousels. |
| Root `aria-roledescription` | none | `carousel` (i18n `carousel`) |
| Root `tabindex` | `0` with `keyboard: 'focused'` | Never added |
| List `role` | — | `presentation` |
| Slide `role` | — | `tabpanel` with pagination, `group` without. With `isNavigation`: `button` if `slideFocus`, otherwise the role is removed. |
| Slide `aria-roledescription` / `aria-label` | — | `slide` / `'1 of 5'` (an existing `aria-label` on the slide is kept). With `isNavigation`: label `'Go to slide 1'`, and no roledescription when `slideFocus`. |
| Track live region | none | `aria-live="polite"` (`"off"` while autoplay plays), `aria-atomic="true"`, `aria-busy` while announcing |
| Pagination | — | `role="tablist"` on the `<ul>`, `role="presentation"` on `<li>`, `role="tab"` + `aria-selected` + roving `tabindex` on buttons |
| Toggle | separate play/pause buttons | `aria-controls` = track ID, `aria-label` = `'Pause autoplay'` / `'Start autoplay'` |

Full per-element list: `structure.md`. Background: `accessibility.md`.

---

## Classes and CSS selectors

(Not in docs. From the build comparison.)

| Change | v3 | v4 |
|---|---|---|
| Removed | `splide__slider` | — |
| Removed | `splide__autoplay`, `splide__play`, `splide__pause` | — |
| Added | — | `splide__toggle`, `splide__toggle__play`, `splide__toggle__pause` |
| Added | — | `splide__sr` (visually hidden live-region text, appended to the track briefly) |
| Added | — | `is-focus-in` on the root (4.0.0) |
| Added | — | `is-overflow` on the root (4.1.0) |
| Added | — | Track modifiers `splide__track--{slide\|loop\|fade}`, `splide__track--{ltr\|rtl\|ttb}`, `splide__track--draggable`, `splide__track--nav`. The root still gets `splide--*`. |
| Added | — | `splide__arrows--{ltr\|rtl\|ttb}`, `splide__pagination--{ltr\|rtl\|ttb}` |
| Unchanged | `splide`, `splide__track`, `splide__list`, `splide__slide`, `splide__slide--clone`, `splide__slide__container`, `splide__arrows`, `splide__arrow`, `splide__arrow--prev`, `splide__arrow--next`, `splide__pagination`, `splide__pagination__page`, `splide__progress`, `splide__progress__bar`, `splide__spinner`, `is-initialized`, `is-active`, `is-visible`, `is-prev`, `is-next`, `is-loading`, `is-rendered` | same |

`STATUS_CLASSES` (removed from slides on destroy): v3 `is-active, is-visible, is-prev, is-next, is-loading`. v4.0 adds `is-focus-in`, v4.1 adds `is-overflow`.

Core CSS selector changes v3 → v4.1.4:

| v3 core rule | v4 core rule |
|---|---|
| `.splide--draggable > .splide__slider > .splide__track, .splide--draggable > .splide__track { user-select: none }` | `.splide__track--draggable { -webkit-touch-callout: none; user-select: none }` |
| `.splide--fade > (.splide__slider >) .splide__track > .splide__list { display: block }` and absolutely positioned fade slides | 4.0: `.splide__track--fade > .splide__list { display: block }` + absolute slides. 4.1: only `.splide__track--fade > .splide__list > .splide__slide { margin: 0 !important; opacity: 0; z-index: 0 }` and `.is-active { opacity: 1; z-index: 1 }` |
| `.splide--ttb.is-active > (.splide__slider >) .splide__track > .splide__list { display: block }` | `.splide__track--ttb > .splide__list { display: block }` |
| `.splide { outline: none; ... }`, `.splide__slide { outline: none; ... }` | `outline: none` removed from core. Themes add focus rings. |
| `.splide__list { transform-style: preserve-3d }` | removed |
| `.splide__slider { position: relative }` | removed |
| — | `.splide:not(.is-overflow) .splide__pagination { display: none }` (4.1) |
| — | `.splide__sr { ... }`, `.splide__toggle...` icon switching |

v3 theme file names are the same in v4 (`splide-default`, `splide-sea-green`, `splide-skyblue`).

---

## Events

(Not in docs. From the build comparison.)

| Event | v3 | v4 |
|---|---|---|
| `repositioned` | Emitted after the list was repositioned (on resize) | **Removed** in 4.0.0. The nearest public event is `resized`. |
| `shifted` | Public (`'shifted'`) | Public through 4.0.17. **Internal since 4.1.0** (value `'sh'`, exported as `EVENT_SHIFTED` in ESM only) |
| `slide:keydown` | Public (`'slide:keydown'`) | Public through 4.0.17. **Internal since 4.1.0** (value `'sk'`, `EVENT_SLIDE_KEYDOWN`) |
| `overflow` | — | **New in 4.1.0**. Argument `isOverflow: boolean`. Fires when overflow status changes (`is-overflow` toggles). |
| `ei` (`EVENT_END_INDEX_CHANGED`) | — | New internal event in 4.1.0 |

Unchanged: `mounted`, `ready`, `move`, `moved`, `click`, `active`, `inactive`, `visible`, `hidden`, `refresh`, `updated`, `resize`, `resized`, `drag`, `dragging`, `dragged`, `scroll`, `scrolled`, `destroy`, `arrows:mounted`, `arrows:updated`, `pagination:mounted`, `pagination:updated`, `navigation:mounted`, `autoplay:play`, `autoplay:playing`, `autoplay:pause`, `lazyload:loaded`. Arguments: `events.md`.

With the classic script, use the string names. With the ESM build, the `EVENT_*` constants always hold the right values, including the internal ones.

---

## Components and exports

From the docs:

| v3 | v4 |
|---|---|
| `Components.Move.isBusy()` | Moved to `Components.Controller.isBusy()`. Returns `true` only while moving/scrolling **and** `waitForTransition` is on. |
| `Components.Controller.scroll( destination, useIndex?, snap?, duration?, callback? )` | `Components.Controller.scroll( destination, duration?, snap?, callback? )`. `destination` is a **position** in px. The `useIndex` flag is gone. To scroll to an index: `Controller.scroll( Components.Move.toPosition( index, true ), duration )`. |
| `Components.Scroll.scroll( position, duration?, callback? )` | `Components.Scroll.scroll( position, duration?, snap?, callback? )`. The source has a 5th internal `noConstrain?` parameter. |

Not in docs, from the type definitions:

| Area | v3 | v4 |
|---|---|---|
| `Components.Elements` | `root, slider, track, list, slides, arrows, prev, next, bar, autoplay, play, pause` | `root, track, list, slides, arrows?, pagination?, prev?, next?, bar?, toggle?` |
| `Components.Layout` | `listSize, slideSize, sliderSize(), totalSize, getPadding` | adds `sliderSize( withoutGap? )`, `isOverflow()` (4.1), `resize( force? )` |
| `Components.Arrows` | `arrows` | adds `update()` |
| `Components.Move` | had `isBusy()` | `isBusy()` removed. Adds `reposition()`. |
| `Components.Live` | — | New: `disable( disabled: boolean )` |
| `splide.event` | `EventBusObject` | `EventInterfaceObject` |
| ESM exports | `Splide`, `SplideRenderer`, `EventBus`, `EventInterface`, `RequestInterval`, `State`, `Throttle`, `CLASSES`, `STATUS_CLASSES`, `CLASS_*` (including `CLASS_SLIDER`, `CLASS_AUTOPLAY`, `CLASS_PLAY`, `CLASS_PAUSE`), `EVENT_*` (including `EVENT_REPOSITIONED`) | `EventBus` removed. Added `EventBinder`, `DEFAULTS`, `SLIDE`/`LOOP`/`FADE`, `LTR`/`RTL`/`TTB`, `CLASS_TOGGLE`, `CLASS_TOGGLE_PLAY`, `CLASS_TOGGLE_PAUSE`, `CLASS_SR`, `CLASS_FOCUS_IN`, `CLASS_OVERFLOW`, `EVENT_OVERFLOW`, `EVENT_END_INDEX_CHANGED`. Removed `CLASS_SLIDER`, `CLASS_AUTOPLAY`, `CLASS_PLAY`, `CLASS_PAUSE`, `EVENT_REPOSITIONED`. |
| Classic script global | `window.Splide` | `window.Splide` only (`Splide.defaults`, `Splide.STATES`) |
| `Splide` instance methods | `mount, sync, go, on, off, emit, add, remove, is, refresh, destroy`, `options`, `length`, `index` | Same API |
| `data-splide` parsing | In the Options component, at `setup()` | In the constructor. Same precedence: it overrides constructor options. |
| `Splide.defaults` | Merged into the shared `DEFAULTS` object | Merged per instance in the constructor |

---

## Changes inside v4 (4.0 to 4.1)

(Not in the guide page. Useful when reading 4.0-era examples.)

| Version | Change |
|---|---|
| 4.0.3 | Track `aria-atomic` and `aria-busy` added to the live region (docs' accessibility page marks these `^4.0.3`) |
| 4.1.0 | `omitEnd` option |
| 4.1.0 | `is-overflow` root class, `overflow` event, `Layout.isOverflow()`; core CSS hides pagination while not overflowing |
| 4.1.0 | `shifted` and `slide:keydown` became internal (`'sh'`, `'sk'`), plus the new internal `'ei'` event |
| 4.1.0 | Fade slides no longer absolutely positioned in core CSS |
| 4.1.0 | Class constants built from `CLASS_PREFIX` (`splide__`) and `STATUS_CLASS_PREFIX` (`is-`). No class names changed. |
| 4.1.4 | Latest v4. `splide.esm.js` is identical to 4.1.3 apart from the version banner. |

---

## Framework integrations

React Splide, Vue Splide and Svelte Splide have their own v3 to v4 migration instructions (`/integration/react-splide/#migrating-from-v3`, `/integration/vue-splide/#migrating-from-v3`, `/integration/svelte-splide/#migrating-from-v3`). They don't apply to this CDN-only skill.
