# Options

Source: https://splidejs.com/guides/options/ (with the i18n table from https://splidejs.com/guides/i18n/ and the reduced-motion notes from https://splidejs.com/guides/accessibility/). Every option, type and default below was checked against the Splide v4.1.3 source (`src/js/types/options.ts`, `constants/defaults.ts`, `constants/classes.ts`, `constants/i18n.ts`, `components/*`, `core/Splide/Splide.ts`) and the 4.1.4 `dist/types/index.d.ts`. The two releases differ only in their version banner.

## Contents

- [Conventions and setup](#conventions-and-setup)
- [Option tables](#option-tables)
  - [Type and direction](#type-and-direction)
  - [Layout and size](#layout-and-size)
  - [Movement and transition](#movement-and-transition)
  - [Loop clones](#loop-clones)
  - [Drag and flick](#drag-and-flick)
  - [Autoplay](#autoplay)
  - [Controls: arrows and pagination](#controls-arrows-and-pagination)
  - [Lazy load](#lazy-load)
  - [Keyboard and wheel](#keyboard-and-wheel)
  - [Navigation and focus](#navigation-and-focus)
  - [Accessibility](#accessibility)
  - [Responsive](#responsive)
  - [Customization](#customization)
  - [Extension options](#extension-options)
- [Option details and examples](#option-details-and-examples)
  - [type](#type) · [direction](#direction) · [start](#start)
  - [perPage / perMove](#perpage--permove) · [focus + trimSpace + omitEnd](#focus--trimspace--omitend) · [gap](#gap) · [padding](#padding) · [width](#width)
  - [height / fixedWidth / fixedHeight / heightRatio](#height--fixedwidth--fixedheight--heightratio) · [autoWidth / autoHeight](#autowidth--autoheight) · [cover](#cover)
  - [rewind / rewindByDrag](#rewind--rewindbydrag) · [speed / rewindSpeed](#speed--rewindspeed) · [easing / easingFunc](#easing--easingfunc) · [waitForTransition](#waitfortransition) · [updateOnMove](#updateonmove) · [useScroll](#usescroll-undocumented-from-source)
  - [cloneStatus / clones](#clonestatus--clones)
  - [drag: 'free' + snap](#drag-free--snap) · [noDrag](#nodrag) · [dragMinThreshold](#dragminthreshold) · [flickPower](#flickpower) · [flickMaxPages](#flickmaxpages)
  - [autoplay / interval / pause options](#autoplay--interval--pause-options)
  - [arrows / arrowPath](#arrows--arrowpath) · [pagination / paginationKeyboard / paginationDirection](#pagination--paginationkeyboard--paginationdirection)
  - [lazyLoad / preloadPages](#lazyload--preloadpages)
  - [keyboard](#keyboard) · [wheel / releaseWheel / wheelMinThreshold / wheelSleep](#wheel--releasewheel--wheelminthreshold--wheelsleep)
  - [isNavigation / slideFocus](#isnavigation--slidefocus) · [focusableNodes](#focusablenodes)
  - [label / labelledby / role](#label--labelledby--role) · [live](#live)
  - [breakpoints + mediaQuery](#breakpoints--mediaquery) · [destroy (inside breakpoints)](#destroy-inside-breakpoints) · [reducedMotion](#reducedmotion)
  - [classes](#classes) · [i18n](#i18n)
- [Readonly options](#readonly-options)
- [Setting options: three ways and precedence](#setting-options-three-ways-and-precedence)
- [Docs vs source: summary](#docs-vs-source-summary)

---

## Conventions and setup

- **(R)** after an option name means the docs mark it as **responsive**: you can change it with `breakpoints` or `splide.options = { … }` after mount. Options without (R) are **readonly** in the docs. See [Readonly options](#readonly-options) for what the source really does.
- **Default `—`** means the option is `undefined` in the source (no entry in `DEFAULTS`). The component then uses an inline fallback, which is listed in the description.
- Where the docs give `false` and the source leaves the value `undefined`, this file writes `false`. The two behave the same.
- Number values for sizes become `px`. Strings are passed through as CSS (`'10rem'`, `'80vw'`, `'calc(…)'`).
- The examples assume this page setup. They use the global `Splide` from the classic script: no npm, no imports.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js" defer></script>

<section class="splide" aria-label="Example carousel">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">01</li>
      <li class="splide__slide">02</li>
      <li class="splide__slide">03</li>
      <!-- … -->
    </ul>
  </div>
</section>

<script>
  document.addEventListener( 'DOMContentLoaded', function () {
    new Splide( '.splide', { /* options */ } ).mount();
  } );
</script>
```

---

## Option tables

### Type and direction

| Option | Type | Default | Description |
|---|---|---|---|
| `type` | `'slide' \| 'loop' \| 'fade'` (string) | `'slide'` | The carousel type. `'slide'` uses the slide transition. `'loop'` is an endless carousel built with clones. `'fade'` uses the fade transition and does not support `perPage`. To loop a fade carousel, enable `rewind`. See [type](#type). |
| `direction` (R) | `'ltr' \| 'rtl' \| 'ttb'` | `'ltr'` | `'ltr'` = left to right, `'rtl'` = right to left, `'ttb'` = top to bottom (vertical; needs `height` or `heightRatio`). See [direction](#direction). |
| `start` | number | `0` | The start index. Source: `options.start \|\| 0`, clamped to the valid range. |

### Layout and size

| Option | Type | Default | Description |
|---|---|---|---|
| `width` (R) | number \| string | — | The carousel's max width (set as `max-width` on the root). Accepts CSS such as `10em` or `80vw`. |
| `height` (R) | number \| string | — | Accepts CSS except `%`. Horizontal: the height of every slide (or of its `.splide__slide__container`). `'ttb'`: the track height. Docs table says "carousel height" and the option section says "slide height". Source does both, by direction. See [height / fixedWidth / fixedHeight / heightRatio](#height--fixedwidth--fixedheight--heightratio). |
| `fixedWidth` (R) | number \| string | — | Fixes the width of slides (CSS). The layout ignores `perPage` when this is set. |
| `fixedHeight` (R) | number \| string | — | Fixes the height of slides (CSS except `%`). Overrides `height`/`heightRatio` for the slide height. Docs also say it overrides `perPage`. |
| `heightRatio` (R) | number | — | Slide height = carousel (list) width × ratio. For example, width `1000` and ratio `0.3` give `300`. Ignored when `height` or `fixedHeight` is set. |
| `autoWidth` | boolean | `false` | When `true`, each slide's width comes from its own width (your CSS). Don't set `perPage`/`perMove`, or set them to `1`. With `lazyLoad`, give each slide an explicit width. |
| `autoHeight` | boolean | `false` | When `true`, each slide's height comes from its own height. Don't set `perPage`/`perMove`, or set them to `1`. With `lazyLoad`, give each slide an explicit height. Source: only used when `direction: 'ttb'`. |
| `perPage` (R) | number (integer) | `1` | The number of slides to display in a page. Must be an integer. |
| `perMove` (R) | number (integer) | — | The number of slides to move at once. Must be an integer. Source fallback: `perPage`, or `1` when `focus`/`isNavigation` is set. |
| `gap` (R) | number \| string | — | The gap between slides (CSS, e.g. `1em`). It's applied as the slide's `margin-right` (`margin-left` in RTL, `margin-bottom` in TTB). |
| `padding` (R) | number \| string \| `{ left?, right? }` \| `{ top?, bottom? }` (values number \| string) | — | Padding left/right for a horizontal carousel, or top/bottom for a vertical one. One value sets both sides. An object sets each side. Applied to the track. See [padding](#padding). |
| `focus` (R) | number \| `'center'` | — | Which slide is active when a page shows several slides. See [focus + trimSpace + omitEnd](#focus--trimspace--omitend). |
| `trimSpace` | boolean \| `'move'` | `true` | Trims the empty space before the first slide and after the last. `true` trims, and the carousel may stay in place when asked to move. `false` keeps the space. `'move'` trims and always moves when asked. Source: only for `type: 'slide'`. |
| `omitEnd` | boolean | `false` | Since **v4.1.0**. Omits pagination dots that would only change the active slide without moving the carousel, and disables the next arrow once the last page is reached. Needs `type: 'slide'` (or unset) and `focus`. |
| `cover` | boolean | `false` | Converts the slide image's `src` into a CSS `background-image` on the parent. Needs `height`, `fixedHeight` or `heightRatio`. Doesn't support `srcset`. See [cover](#cover). |

### Movement and transition

| Option | Type | Default | Description |
|---|---|---|---|
| `rewind` (R) | boolean | `false` | Rewinds the carousel at either end. Doesn't work in `loop` mode. |
| `speed` (R) | number (ms) | `400` | The transition speed in milliseconds. With `0`, the carousel jumps to the target slide immediately (source: any value below `1` jumps). |
| `rewindSpeed` (R) | number (ms) | — | The transition speed on rewind, in milliseconds. Uses `speed` by default. Source: only for `type: 'slide'`. |
| `easing` (R) | string | `'cubic-bezier(0.25, 1, 0.5, 1)'` | The timing function for the CSS transition, e.g. `linear`, `ease` or `cubic-bezier()`. |
| `easingFunc` (R) | `( t: number ) => number` | `t => 1 - Math.pow( 1 - t, 4 )` | The easing function for drag-free mode (easeOutQuart). The default is inline in the Scroll component. Also used by `useScroll` and snapping. |
| `waitForTransition` | boolean | `false` | Disables all actions while the carousel is moving. Docs: even when `false`, the carousel waits for the transition at the loop points. |
| `updateOnMove` | boolean | `false` | Updates the `is-active` status class just before the carousel moves, instead of after. |
| `useScroll` | boolean | — | **Undocumented, from source.** When truthy, the slide transition is run by the Scroll component (a JS `requestAnimationFrame` animation using `easingFunc` for `speed` ms) instead of a CSS `transform` transition. See [useScroll](#usescroll-undocumented-from-source). |

### Loop clones

| Option | Type | Default | Description |
|---|---|---|---|
| `clones` (R) | number | — (computed) | Sets the number of clones on each side of a loop carousel. Don't set it unless you know how it affects the carousel. |
| `cloneStatus` | boolean | `true` | Adds `is-active` to clones of the active slide too. Source types list it as responsive. |

### Drag and flick

| Option | Type | Default | Description |
|---|---|---|---|
| `drag` (R) | boolean \| `'free'` | `true` | `true` enables drag, `false` disables it, `'free'` enables drag-free mode (no snapping to a slide after the drag). |
| `snap` (R) | boolean | `false` | Snaps to the closest slide in drag-free mode. |
| `rewindByDrag` (R) | boolean | — | Lets the user rewind by dragging. `rewind` must be `true`. |
| `noDrag` | string (selector) | — | A selector for nodes that can't start a drag. |
| `dragMinThreshold` (R) | number \| `{ mouse: number, touch: number }` | `10` | The distance needed to start moving the carousel by touch. Pass an object to also set the mouse threshold. Source: touch `\|\| 10` (so `0` isn't accepted) and mouse `\|\| 0`. |
| `flickPower` (R) | number | `600` | The power of a "flick". The larger it is, the farther the carousel runs. Around 500 is recommended. Inline default `\|\| 600`. |
| `flickMaxPages` (R) | number | `1` | Limits how many pages a flick can move. Inline default `\|\| 1`. Source: ignored in `drag: 'free'`. |

### Autoplay

| Option | Type | Default | Description |
|---|---|---|---|
| `autoplay` | boolean \| `'pause'` | `false` | Enables autoplay. With `'pause'`, it doesn't start when the carousel becomes active. You then need a play button or `splide.Components.Autoplay.play()`. With the Intersection extension, you can autoplay only while the carousel is in the viewport. |
| `interval` | number (ms) | `5000` | The autoplay interval in milliseconds. `data-splide-interval` on a slide overrides it for that slide. |
| `pauseOnHover` | boolean | `true` | Pauses autoplay on mouseover. Keep it `true` for accessibility. |
| `pauseOnFocus` | boolean | `true` | Pauses autoplay while the carousel contains the focused element. Keep it `true` for accessibility. |
| `resetProgress` | boolean | `true` | Resets the autoplay progress when autoplay is asked to start again. `false` resumes from where it paused. |

### Controls: arrows and pagination

| Option | Type | Default | Description |
|---|---|---|---|
| `arrows` (R) | boolean | `true` | Creates arrows, or uses the ones found in the HTML. Must stay `true` when you provide arrows in HTML. |
| `arrowPath` | string | — | Replaces the arrow SVG path, e.g. `'m7.61 0.807-2.12...'`. The SVG is 40×40. Source default path: `m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z`. |
| `pagination` (R) | boolean | `true` | Creates pagination (indicator dots). Must be `false` when you use `isNavigation`, or the roles and ARIA attributes break. |
| `paginationKeyboard` (R) | boolean | `true` | Enables keyboard shortcuts for pagination while it has focus. The W3C Carousel Design Pattern requires this. |
| `paginationDirection` (R) | `'ltr' \| 'rtl' \| 'ttb'` | — (the `direction` value) | Sets the pagination direction. It affects the appearance, the shortcuts and the ARIA attributes. |

### Lazy load

| Option | Type | Default | Description |
|---|---|---|---|
| `lazyLoad` | boolean \| `'nearby'` \| `'sequential'` | `false` | `false` disables lazy loading. `'nearby'` loads only images around the active slide (page). `'sequential'` loads images one after another. Images need `data-splide-lazy` (and optionally `data-splide-lazy-srcset`). Source: `true` behaves like `'nearby'`. |
| `preloadPages` | number | `1` | How many pages (not slides) around the active slide to load in advance. Only for `'nearby'`. Inline default `\|\| 1`, so `0` acts like `1`. |

### Keyboard and wheel

| Option | Type | Default | Description |
|---|---|---|---|
| `keyboard` (R) | boolean \| `'global'` \| `'focused'` | `false` | Keyboard shortcuts. `true` = only while the carousel contains focus. `'focused'` = same as `true`. `'global'` = listens on `window` (discouraged). `false` = disabled. |
| `wheel` | boolean | `false` | Navigates with the mouse wheel. Also set `waitForTransition: true` and/or `wheelSleep`, or the carousel will reach the end at once. |
| `wheelMinThreshold` | number | — | The threshold that cuts off the small deltas of inertia scrolling. Inline default `\|\| 0`. |
| `wheelSleep` | number (ms) | — | How long to wait, in milliseconds, before accepting the next wheel input. The timer starts when the transition begins. Inline default `\|\| 0`. |
| `releaseWheel` | boolean | `false` | Releases the wheel event at the first or last slide, so the page can keep scrolling. |

### Navigation and focus

| Option | Type | Default | Description |
|---|---|---|---|
| `isNavigation` | boolean | `false` | Makes slides clickable so they control another carousel. Use `splide.sync()` to link the carousels. Turn off `pagination` with this. |
| `slideFocus` | boolean | — (computed) | Adds `tabindex="0"` to visible slides. Defaults to `false` without `isNavigation` and `true` with it (source: `slideFocus === undefined ? isNavigation : slideFocus`). Clickable elements must be focusable for keyboard users. |
| `focusableNodes` | string (selector) | `'a, button, textarea, input, select, iframe'` | A selector for focusable elements inside slides. When a slide gets `aria-hidden="true"`, Splide gives these elements `tabindex="-1"`. |

### Accessibility

| Option | Type | Default | Description |
|---|---|---|---|
| `role` | string | `'region'` | The `role` attribute for the root. It must be a landmark role or `group`. Pass `''` or `undefined` to leave it unset. Ignored when the root is a `<section>`. Source: also ignored when the root already has a `role` attribute. |
| `label` (R) | string | the root's `aria-label` attribute, else `''` | Sets `aria-label` on the root, overriding the HTML attribute. |
| `labelledby` (R) | string | the root's `aria-labelledby` attribute, else `''` | Sets `aria-labelledby` on the root, overriding the HTML attribute. |
| `live` | boolean | `true` | Enables an ARIA live region. Not enabled when `isNavigation` is on. |

### Responsive

| Option | Type | Default | Description |
|---|---|---|---|
| `breakpoints` | `Record<number \| string, ResponsiveOptions>` | — | Options for specific breakpoints. `max-width` (desktop-first) by default; set `mediaQuery: 'min'` for `min-width` (mobile-first). |
| `mediaQuery` | `'min' \| 'max'` | `'max'` | With `'min'`, breakpoints use `min-width`; otherwise `max-width`. Source: undefined, and any value other than `'min'` means max. |
| `reducedMotion` | Options | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Options applied when `(prefers-reduced-motion: reduce)` matches. |
| `destroy` (R) | boolean \| `'completely'` | `false` | `true` destroys the carousel but keeps watching the breakpoints, so it can remount. `'completely'` destroys it for good. Meant for `breakpoints`. |

### Customization

| Option | Type | Default | Description |
|---|---|---|---|
| `classes` | `Record<string, string>` | `CLASSES` (see [classes](#classes)) | Class names for the elements Splide creates. Each value **replaces** the default for that key, so include the default class too. |
| `i18n` | `Record<string, string>` | `I18N` (see [i18n](#i18n)) | Text strings. Partial objects are merged into the defaults. |

### Extension options

`ResponsiveOptions` has an index signature (`[ key: string ]: any`), so the options object also takes options for extensions (e.g. `autoScroll`, `intersection`, `video`, `grid`). Splide core ignores keys it doesn't know. They are merged, passed to `breakpoints` and stored in `data-splide` like any other option. See `references/extensions/`.

---

## Option details and examples

### `type`

```js
// Endless carousel (clones are added on both sides)
new Splide( '.splide', { type: 'loop', perPage: 3 } ).mount();

// Fade. One slide at a time; rewind makes it loop
new Splide( '.splide', { type: 'fade', rewind: true } ).mount();
```

- `'loop'` clones slides (`.splide__slide--clone`). See [cloneStatus / clones](#clonestatus--clones). `rewind` is ignored in loop.
- `'fade'` doesn't support `perPage`. The core CSS stacks slides (`.splide__track--fade > .splide__list > .splide__slide { opacity:0 }`, `.is-active { opacity:1 }`) and the transition is `opacity {speed}ms {easing}`. `rewindSpeed` isn't used for fade. Drag on a fade carousel moves one slide (`'+'`/`'-'`), or `'>'`/`'<'` with `rewind` + `rewindByDrag`.
- `trimSpace`, `trimSpace: 'move'`, `omitEnd` and the drag "friction" at the edges only apply to `'slide'`.
- **Never change `type` in breakpoints or after mount.** Source: `mount()` picks the transition (`Fade` or `Slide`) from `type` *before* breakpoints are applied, and the Controller and Scroll cache `isLoop`/`isSlide` when they're built.

### `direction`

```js
// Right to left. The root gets `splide--rtl`, and the core CSS sets `direction: rtl` on it
new Splide( '.splide', { direction: 'rtl', perPage: 2 } ).mount();

// Vertical. Needs `height` or `heightRatio` (fixedHeight alone is not enough)
new Splide( '.splide', {
  direction: 'ttb',
  height   : '30rem',
  perPage  : 3,
  gap      : '1rem',
} ).mount();
```

- Source (`Layout`): in `'ttb'` the track height is `height` (or list width × `heightRatio`) minus the top/bottom padding. If neither is set, Splide throws `[splide] height or heightRatio is missing.` `fixedHeight` only sizes the slides.
- With `'ttb'`, `gap` becomes `margin-bottom`, `padding` takes `top`/`bottom`, and `fixedWidth`/`autoWidth` logic switches to `fixedHeight`/`autoHeight` (clone count, slide size). Keyboard and pagination keys switch to Up/Down. The theme rotates the arrows (`.splide__arrows--ttb`).
- RTL: keyboard Left/Right are reversed, the drag direction is reversed, and the theme mirrors the arrows (`.splide__arrows--rtl`). Padding object keys stay physical (`left` is still the left side).
- Classes added: `splide--{direction}` on the root, `splide__track--{direction}` on the track, `splide__arrows--{direction}` on the arrows wrapper.
- Source (`Media`): when a breakpoint change alters `direction`, Splide calls `refresh()` automatically. `splide.options = { direction }` only emits `updated`, so call `splide.refresh()` after it.

### `start`

```js
new Splide( '.splide', { start: 3, perPage: 3 } ).mount();
```

Read once when the Controller is created. It can't be changed after mount; use `splide.go()` instead.

### `perPage` / `perMove`

```js
// 3 slides visible, move one by one
new Splide( '.splide', { perPage: 3, perMove: 1 } ).mount();
```

- Both must be integers.
- Source (`Controller`): one step (arrow, `'>'`, `'<'`, autoplay) moves `perMove || ( focus/isNavigation ? 1 : perPage )` slides.
- Pagination dots: `Math.ceil( length / perPage )`, or `endIndex + 1` when `focus`/`isNavigation` is set.
- Slide width (without `fixedWidth`/`autoWidth`): `calc((100% + gap) / perPage - gap)`.
- When there are no more slides than `perPage` (`Slides.isEnough()` is false) and there's no `focus`, the carousel can't move and autoplay won't play.
- `fixedWidth` makes the *layout* ignore `perPage`, but `perPage` still decides the step size, the page count and `isEnough()`. Keep it at `1` with `fixedWidth`/`autoWidth`, or set `perMove`.
- `'fade'` doesn't support `perPage`.

### `focus` + `trimSpace` + `omitEnd`

`focus` decides which slide is active when a page shows several slides. A number is the slot (0-based) where the active slide sits. `'center'` centers it. Source (`Move.offset`): `'center'` → `(listSize - slideSize) / 2`; number `n` → `n × (slideWidth + gap)`.

Once `focus` is set (or `isNavigation` is on):

- each step moves 1 slide unless `perMove` is set;
- the last reachable index is the last slide;
- pagination has one dot per reachable index.

By default Splide trims the empty space at the edges, so the carousel may not move even though the active slide changes:

```js
new Splide( '.splide', {
  perPage: 3,
  focus  : 'center',
} ).mount();
```

To show those spaces, turn trimming off:

```js
new Splide( '.splide', {
  perPage  : 3,
  focus    : 'center',
  trimSpace: false,
} ).mount();
```

To trim the spaces and still have the carousel move every time it's asked to, use `'move'`:

```js
new Splide( '.splide', {
  trimSpace: 'move',
  focus    : 'center',
  perPage  : 3,
} ).mount();
```

| `trimSpace` | Behavior |
|---|---|
| `true` | Trims spaces. The carousel may stay in the same place even when asked to move. |
| `false` | Disables trimming of the edge spaces. |
| `'move'` | Trims spaces and always moves the carousel when asked (it skips indices that wouldn't change the position). |

Docs say trimming applies "if the `focus` option is available". Source (`Move.trim`): it clamps the position for *any* `type: 'slide'` carousel when `trimSpace` is truthy. It also matters with `fixedWidth`/`autoWidth` without `focus`. It has no effect for `loop`/`fade`.

`omitEnd` (since v4.1.0) removes the redundant pagination dots, i.e. the ones that change only the active slide, and disables the next arrow once the last page is reached. Requirements: `type` is `'slide'` (or unset) and `focus` is set. Source: the end index is found by comparing *trimmed* positions, so `trimSpace` must not be `false`. With 9 slides, this shows 7 dots:

```js
new Splide( '.splide', {
  perPage: 3,
  focus  : 0,
  omitEnd: true,
} ).mount();
```

It's also useful with `fixedWidth` and `autoWidth`:

```js
new Splide( '.splide', {
  height   : '10rem',
  autoWidth: true,
  focus    : 0,
  omitEnd  : true,
} ).mount();
```

### `gap`

```js
new Splide( '.splide', { perPage: 3, gap: '1rem' } ).mount();   // or gap: 16 (px)
```

### `padding`

```js
// By number (px)
padding: 10

// By the CSS format
padding: '1rem'

// Specifies each value for a horizontal carousel
padding: { left: 10, right: 20 }
padding: { left: '1rem', right: '2rem' }

// Specifies each value for a vertical carousel
padding: { top: 10, bottom: 20 }
```

A loop carousel with `20%` padding on each side, so the neighbouring slides peek in:

```js
new Splide( '.splide', {
  type   : 'loop',
  padding: '20%',
} ).mount();
```

Source (`Layout`):

- The padding goes on `.splide__track` (`padding-left`/`padding-right`, or `padding-top`/`padding-bottom` in TTB).
- A missing side in an object is `0`.
- In `'ttb'` the track height is `height − top − bottom`.
- Object keys are physical in RTL (`left` stays left).
- In breakpoints, an object `padding` is deep-merged with an object base `padding` (e.g. `{ left: 0 }` keeps the base `right`). A string or number base is replaced.

### `width`

```js
new Splide( '.splide', { width: '80%' } ).mount();
```

Sets `max-width` on the root element.

### `height` / `fixedWidth` / `fixedHeight` / `heightRatio`

```js
// Fixed-size slides (perPage is ignored by the layout)
new Splide( '.splide', {
  fixedWidth : '10rem',
  fixedHeight: '6rem',
  gap        : '1rem',
} ).mount();

// Slide height = 30% of the carousel width (recomputed on resize)
new Splide( '.splide', {
  heightRatio: 0.3,
} ).mount();
```

How the source (`Layout`) resolves sizes:

| | Horizontal (`ltr`/`rtl`) | Vertical (`ttb`) |
|---|---|---|
| Slide width | `autoWidth` → none (your CSS); else `fixedWidth`; else `calc((100% + gap)/perPage - gap)` | `fixedWidth` or none |
| Slide height | `fixedHeight`; else `height`; else list width × `heightRatio`; else none (CSS / flex stretch) | `fixedHeight`; else `autoHeight` → none; else `calc((100% + gap)/perPage - gap)` |
| Track height | none | `height` or list width × `heightRatio` (required), minus top/bottom padding |

- Slide heights go on the `.splide__slide__container` child when one exists, and on the slide otherwise. Widths always go on the slide.
- Don't use `%` for `height`/`fixedHeight`.
- Docs: "`fixedHeight` … The carousel will ignore `perPage`, `height` and `heightRatio`." Source: true for the *slide* height. In `'ttb'` the track still needs `height` or `heightRatio`.
- `heightRatio` uses the width of `.splide__list` (the track's content box, i.e. without the track padding).

### `autoWidth` / `autoHeight`

```html
<li class="splide__slide" style="width: 180px">…</li>
<li class="splide__slide" style="width: 250px">…</li>
<li class="splide__slide" style="width: 200px">…</li>
```
```js
new Splide( '.splide', {
  autoWidth: true,
  gap      : '1rem',
  // perPage / perMove: leave unset (or 1)
} ).mount();

// Vertical carousel whose slides keep their own height
new Splide( '.splide', {
  direction : 'ttb',
  height    : '24rem',
  autoHeight: true,
} ).mount();
```

- Don't set `perPage`/`perMove`, or set them to `1`.
- With `lazyLoad`, every slide needs an explicit width (`autoWidth`) or height (`autoHeight`). Otherwise slides measure 0 before their images load.
- Source: `autoHeight` is read only in vertical mode (`direction: 'ttb'`). A horizontal carousel has no adaptive-height mode. Its slides get `height`/`fixedHeight`/`heightRatio`, or the flex row stretches them to the tallest slide.
- For `loop` with `autoWidth`, the clone count is the number of real slides on each side.
- Pair them with `omitEnd` (and `focus: 0`) to avoid trailing dots that don't move the carousel.

### `cover`

```html
<li class="splide__slide"><img src="photo.jpg" alt=""></li>
<!-- or -->
<li class="splide__slide"><div class="splide__slide__container"><img src="photo.jpg" alt=""></div></li>
```
```js
new Splide( '.splide', { cover: true, heightRatio: 0.5 } ).mount();
```

Source (`Cover`):

- Takes the first `<img>` that is a **direct child** of `.splide__slide__container`, or of the slide.
- Sets `background: center/cover no-repeat url("src")` on the container (or the slide), and hides the `<img>` (`display: none`).
- Needs `height`, `fixedHeight` or `heightRatio`, because the hidden image no longer gives the slide a height.
- Doesn't support `srcset`.
- Works with `lazyLoad`: it re-applies on `lazyload:loaded`.
- Readonly: set once at mount.

### `rewind` / `rewindByDrag`

```js
new Splide( '.splide', { rewind: true } ).mount();

new Splide( '.splide', {
  rewind      : true,
  rewindByDrag: true,  // dragging past the last slide goes back to the first
} ).mount();
```

- `rewind` doesn't work in loop mode.
- Source: `rewindByDrag` only acts when `rewind` is also on. For `'slide'` it works when the drag goes past an edge. For `'fade'` it makes a drag use `'>'`/`'<'` instead of `'+'`/`'-'`.
- When the next arrow will rewind, its `aria-label` becomes `i18n.first`. When the previous arrow will jump to the end, its label becomes `i18n.last`.

### `speed` / `rewindSpeed`

```js
new Splide( '.splide', {
  speed      : 600,
  rewind     : true,
  rewindSpeed: 1000,
} ).mount();
```

- `speed: 0` (source: any value below `1`) jumps with no animation.
- Source (`transitions/Slide`): `rewindSpeed` is used only for `type: 'slide'`, when moving from index `0` to the end index (or beyond), or from the end index back to `0`. Without it, `speed` is used.
- `'fade'` always uses `speed`.
- Under `prefers-reduced-motion: reduce`, `reducedMotion` sets both to `0` (see [reducedMotion](#reducedmotion)).

### `easing` / `easingFunc`

```js
new Splide( '.splide', {
  easing    : 'ease-in-out',                         // CSS timing function for normal moves and fade
  easingFunc: function ( t ) { return 1 - Math.pow( 1 - t, 3 ); },  // JS easing: drag-free, snap, useScroll
} ).mount();
```

- `easing` is used in the CSS `transition: transform {speed}ms {easing}` (slide/loop) and `opacity {speed}ms {easing}` (fade). It's ignored when `useScroll` is on.
- `easingFunc` takes `t` from 0 to 1 and returns the eased progress. The default is `t => 1 - Math.pow( 1 - t, 4 )`. The Scroll component uses it: drag-free scrolling, snapping, `Controller.scroll()` and `useScroll` transitions.
- A function can't be passed through `data-splide` (JSON).

### `waitForTransition`

```js
new Splide( '.splide', { waitForTransition: true } ).mount();
```

- When `true`, `go()` (arrows, pagination, keyboard, wheel, autoplay) does nothing while the carousel is `MOVING` or `SCROLLING`, and a new drag can't start mid-transition.
- Default `false`. Source: not in `DEFAULTS`, and `!!undefined` is `false`.
- Docs say "Even if `false`, the carousel forcibly waits for the transition on the loop points." In the v4 source, `Controller.isBusy()` checks only this option. At loop edges, `Move` shifts the list position instead of waiting.
- Recommended with `wheel`.

### `updateOnMove`

```js
new Splide( '.splide', {
  updateOnMove: true,
  type        : 'loop',
  perPage     : 3,
  perMove     : 1,
  focus       : 'center',
} ).mount();
```

- Switches `is-active` when the move *starts* (`move` event), not after it ends (`moved`). This gives CSS transitions on `.is-active` time to run along with the movement.
- Source: read when each Slide object is created, so a change takes effect only after `refresh()`.

### `useScroll` (undocumented, from source)

```js
new Splide( '.splide', {
  useScroll : true,
  speed     : 800,                             // used as the scroll duration
  easingFunc: function ( t ) { return t; },    // linear
} ).mount();
```

- Not on the options page. It's declared in `types/options.ts` ("Determines whether to use the Transition component or not").
- In `transitions/Slide`, a truthy `useScroll` replaces the CSS transition with `Components.Scroll.scroll( destination, speed, false, done )`. That's a `requestAnimationFrame` animation that sets `transform` each frame using `easingFunc`. It is **not** native browser scrolling.
- The `scroll` and `scrolled` events then also fire on every normal move, and the state is `SCROLLING` during it.
- `easing` is ignored.
- Has no effect on `'fade'`.
- Read on each transition.

### `cloneStatus` / `clones`

```js
new Splide( '.splide', { type: 'loop', perPage: 3, cloneStatus: false } ).mount();
new Splide( '.splide', { type: 'loop', clones: 0 } ).mount();   // no clones at all (breaks the loop look)
```

- Clones exist only for `type: 'loop'`. They get the `splide__slide--clone` class (`classes.clone`) and the id `{rootId}-clone01`, `-clone02`, and so on.
- `cloneStatus: true` (default) also gives `is-active` to the clones of the active slide. That matters when `perPage`/`focus` shows a clone next to the real slide. Source: read on every slide update.
- Source (`Clones.computeCloneCount`), when `clones` is undefined:
  - with `fixedWidth` (`fixedHeight` in TTB): `Math.ceil( trackWidth / slideWidth )`;
  - with `autoWidth` (`autoHeight` in TTB): the number of real slides;
  - otherwise `perPage × 2`.
- After `updated`/`resize`, Splide refreshes when the required clone count grows or drops to `0`.

### `drag: 'free'` + `snap`

```js
// Drag freely, no snapping
new Splide( '.splide', {
  drag   : 'free',
  perPage: 3,
} ).mount();

// Drag freely, then snap to the closest slide
new Splide( '.splide', {
  type   : 'loop',
  drag   : 'free',
  snap   : true,
  perPage: 3,
} ).mount();
```

| `drag` | Behavior |
|---|---|
| `true` | Enables drag |
| `false` | Disables drag (clicks still work) |
| `'free'` | Drag-free mode: momentum scrolling with `easingFunc`, no snapping unless `snap: true` |

- In free mode the flick distance is `|velocity| × flickPower` with no page limit (`flickMaxPages` is ignored).
- `snap` only works with `drag: 'free'`.
- When `drag` is truthy, the root gets `splide--draggable` and the track `splide__track--draggable` (`user-select: none` in core CSS).
- `prefers-reduced-motion` settings are lifted while a drag is being released, so drag-triggered moves still animate.

### `noDrag`

```js
new Splide( '.splide', {
  noDrag: 'input, textarea, .no-drag',
} ).mount();
```

- A drag can't start on elements matching the selector.
- Source: pagination buttons (`.splide__pagination__page`) and arrows (`.splide__arrow`) never start a drag, whatever this option says.
- Checked with `matches()` against the pointer-down target itself, so add descendants explicitly (e.g. `'.no-drag, .no-drag *'`).
- Read on every pointer down.

### `dragMinThreshold`

These two settings give the same result:

```js
{
  dragMinThreshold: 10,
}

{
  dragMinThreshold: {
    mouse: 0,
    touch: 10,
  },
}
```

- The value lets Splide tell whether the user wants to scroll the page vertically or drag the carousel horizontally.
- The touch threshold doesn't accept `0`. Source: `touch || 10`, `mouse || 0`.
- A drag starts only when the move along the carousel axis is larger than the move across it *and* larger than the threshold.

### `flickPower`

```js
new Splide( '.splide', { flickPower: 500 } ).mount();
```

- The flick destination is `position + sign(v) × min( |v| × flickPower, limit )`, where `v` is the pointer velocity in px/ms over the last 200 ms.
- Larger values run farther. Around 500 is recommended. The source fallback is `|| 600`.

### `flickMaxPages`

```js
new Splide( '.splide', { perPage: 2, flickMaxPages: 3 } ).mount();
```

- Caps the flick distance at `listWidth × flickMaxPages` (fallback `|| 1`).
- Ignored when `drag: 'free'` (the limit is then `Infinity`).

### `autoplay` / `interval` / pause options

```js
new Splide( '.splide', {
  type        : 'loop',
  autoplay    : true,
  interval    : 4000,
  pauseOnHover: true,
  pauseOnFocus: true,
  resetProgress: false,
} ).mount();
```

Override the interval for single slides:

```html
<li class="splide__slide" data-splide-interval="1000"></li>
<li class="splide__slide" data-splide-interval="10000"></li>
```

- `autoplay: 'pause'` mounts with autoplay stopped. Start it with a `.splide__toggle` button or `splide.Components.Autoplay.play()`.
- Autoplay plays only when there are more slides than `perPage` (`Slides.isEnough()`).
- Source: `pauseOnHover` uses `mouseenter`/`mouseleave` on the root, and `pauseOnFocus` uses `focusin`/`focusout` on the root.
- Under reduced motion, autoplay becomes `'pause'` (see [reducedMotion](#reducedmotion)).
- Full autoplay UI (toggle, progress bar): `references/autoplay.md`.

### `arrows` / `arrowPath`

```js
new Splide( '.splide', {
  arrowPath: 'm15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z',
} ).mount();
```

- `arrowPath` is the `d` of a single `<path>` inside `<svg viewBox="0 0 40 40" width="40" height="40" focusable="false">`. Draw it on a 40×40 grid, **pointing right**. Both arrows use the same path. The theme mirrors the previous arrow (`.splide__arrow--prev svg { transform: scaleX(-1) }`), swaps the mirroring in RTL and rotates both by ±90° in TTB.
- Source: `arrowPath` only affects arrows that Splide generates, i.e. when the HTML has no `.splide__arrow--prev` + `.splide__arrow--next`. Arrows written in your HTML keep their own SVG.
- `arrows: false` hides an HTML arrows wrapper with `display: none`. With HTML arrows, `arrows` must stay `true`.
- Generated buttons get `classes.arrow` + `classes.prev`/`classes.next`. The wrapper (`classes.arrows`) is inserted just before the track, or the buttons go into an existing `.splide__arrows` placeholder.
- See `references/arrows.md`.

### `pagination` / `paginationKeyboard` / `paginationDirection`

```js
new Splide( '.splide', {
  arrows             : false,
  paginationDirection: 'ttb',   // vertical dots: Up/Down keys, aria-orientation="vertical"
} ).mount();
```

- Pagination is a `<ul class="splide__pagination splide__pagination--{dir}" role="tablist" aria-label="{i18n.select}">`. It's appended as the last child of the track's parent, or uses an existing `ul.splide__pagination` placeholder.
- Each `<li role="presentation">` holds a `<button class="splide__pagination__page" role="tab" aria-controls="…" aria-label="…">`. Only the current button is tabbable (roving `tabindex`).
- `pagination` also sets the slide role: slides get `role="tabpanel"` when it's `true` and `role="group"` when it's `false`. Source: applied when slides are created, so it changes after `refresh()`.
- Must be `false` with `isNavigation`.
- `paginationKeyboard` (source): while a dot has focus, these keys focus the target dot and go to that page:
  - Right/Left move to the next/previous dot, swapped for RTL, Down/Up for TTB, with wrap-around;
  - `Home` goes to the first dot and `End` to the last.

  The handler stops propagation, so it takes precedence over `keyboard`.
- `paginationDirection` defaults to `direction`. It affects the `splide__pagination--{dir}` class, the key mapping and `aria-orientation` (`vertical` for `'ttb'`).
- See `references/pagination.md`.

### `lazyLoad` / `preloadPages`

| `lazyLoad` | Behavior |
|---|---|
| `false` | Disables lazy loading |
| `'nearby'` | Loads only the images around the active slide (page). Source: `true` behaves the same. |
| `'sequential'` | Loads the images one after another, starting at mount, whatever the current slide |

The `<img>` in the slide needs `data-splide-lazy` with the path or URL of the source file:

```html
<li class="splide__slide">
  <img data-splide-lazy="path-to-the-image" alt="cat">
</li>

<!-- or -->
<li class="splide__slide">
  <img
    src="data:image/png;base64,..."
    data-splide-lazy="path-to-the-image"
    alt="cat"
  >
</li>
```

Or use `data-splide-lazy-srcset` for the `srcset`:

```html
<li class="splide__slide">
  <img
    data-splide-lazy-srcset="600.jpg 600w, 1000.jpg 1000w"
    sizes="100vw"
    data-splide-lazy="1000.jpg"
    alt="cat"
  >
</li>
```

```js
new Splide( '.splide', { lazyLoad: 'nearby', preloadPages: 1, perPage: 2 } ).mount();
```

- A placeholder `src` must be different from the `data-splide-lazy` value.
- `preloadPages` only applies to `'nearby'`. It isn't strictly a page index. It sets the range of indices to load: `perPage × ( preloadPages + 1 ) − 1`. With `perPage: 2` at index `0` that's `3`, so the range is `[ -3, 3 ]` (the negative part covers clones). With `rewind` or `loop`, the distance wraps around the ends.
- Source (`LazyLoad`):
  - While an image loads, its slide gets `is-loading` and a `<span class="splide__spinner">` (`classes.spinner`) is added to the image's parent. An image with no `src` is hidden until it loads.
  - On load the spinner is removed, the image is shown, and `lazyload:loaded` ( `img, Slide` ) and `resize` are emitted.
  - On error, `is-loading` is removed but the spinner stays.
  - Clones are included.
  - Images are found with `[data-splide-lazy], [data-splide-lazy-srcset]` inside each slide.

### `keyboard`

| `keyboard` | Behavior |
|---|---|
| `true` | Shortcuts only while the carousel contains focus (`keydown` on the root) |
| `false` | Disables shortcuts (default) |
| `'global'` | Shortcuts anywhere, listening to the `keydown` event on `window` (discouraged) |
| `'focused'` | Same as `true` |

| Key | Action |
|---|---|
| ← | Previous page in LTR, next page in RTL |
| → | Next page in LTR, previous page in RTL |
| ↑ | Previous page in TTB |
| ↓ | Next page in TTB |

```js
new Splide( '.splide', { keyboard: true } ).mount();
```

- Docs table: "↓ Go to the previous page in TTB". Source (`Direction` maps `ArrowRight` to `ArrowDown` in TTB): ↓ goes to the **next** page.
- In TTB, ← and → do nothing.
- Older Splide added `tabindex="0"` to the root for `'focused'`. v4 doesn't, because `tabindex="0"` on non-interactive content should be avoided, and a carousel must have a `region` or `group` role, which isn't interactive.
- When pagination has focus and both `keyboard` and `paginationKeyboard` are on, the pagination shortcuts take precedence.
- Source: shortcuts are ignored for one tick right after a move starts. The JSDoc in `types/options.ts` still describes v3 behavior (`true`/`'global'` listen on the document, `'focused'` adds `tabindex="0"`). The code does what the table above says.
- Re-read on `updated`, so it works in breakpoints.

### `wheel` / `releaseWheel` / `wheelMinThreshold` / `wheelSleep`

```js
new Splide( '.splide', {
  direction        : 'ttb',
  height           : '20rem',
  wheel            : true,
  releaseWheel     : true,
  waitForTransition: true,   // or wheelSleep: 500. The docs demo also sets this
  wheelMinThreshold: 20,
} ).mount();
```

- `wheel` listens to `wheel` on the track. Turn on `waitForTransition` and/or set `wheelSleep`, or one wheel gesture fires many `go()` calls and the carousel jumps to the end.
- Source (`Wheel`):
  - Only `deltaY` is used, so a vertical wheel drives horizontal carousels too and horizontal trackpad swipes are ignored.
  - A step happens when `|deltaY| > wheelMinThreshold` (fallback `|| 0`) and more than `wheelSleep` ms (fallback `|| 0`) have passed since the last accepted wheel.
  - `deltaY < 0` goes to `'<'`, otherwise `'>'`.
- `releaseWheel: true` stops calling `preventDefault()` when the carousel is idle and has no slide to go to in that direction (first/last slide), so the page keeps scrolling.
- `wheel` is bound at mount (readonly). `wheelMinThreshold`, `wheelSleep` and `releaseWheel` are read on every wheel event.

### `isNavigation` / `slideFocus`

A thumbnail carousel that controls a main carousel:

```js
var main = new Splide( '#main', {
  type      : 'fade',
  rewind    : true,
  pagination: false,
  arrows    : false,
} );

var thumbs = new Splide( '#thumbs', {
  fixedWidth  : 100,
  fixedHeight : 60,
  gap         : 10,
  rewind      : true,
  pagination  : false,   // required with isNavigation
  isNavigation: true,    // slideFocus defaults to true here
} );

main.sync( thumbs );     // before both mount() calls
main.mount();
thumbs.mount();
```

- Source (`Sync`, `Slide`, `Controller`, `Live`), when `isNavigation` is on:
  - clicking a slide, or pressing `Enter`/`Space` on it, calls `go( slide.index )` on this carousel, and `sync()` passes the move on to the linked carousels;
  - slides get `aria-label` = `i18n.slideX` ("Go to slide %s"), `aria-controls` = the ids of the matching slides in the synced carousels, and `aria-current="true"` on the active slide;
  - with `slideFocus`, slides get `role="button"` and lose `aria-roledescription`;
  - the list gets `aria-orientation="vertical"` in TTB;
  - root and track get `splide--nav` / `splide__track--nav`;
  - it behaves as if `focus` were set: step 1 (unless `perMove`), end index = last slide;
  - the live region is disabled.
- `slideFocus: true`: visible slides get `tabindex="0"` and hidden slides `tabindex="-1"`. It defaults to `isNavigation`.
- If a focused slide becomes invisible, focus moves to the active slide.
- Both options are readonly. `slideFocus` is read when slides are created (so on refresh), and `isNavigation` when `Sync` is created.
- See `references/tutorials/` (thumbnail carousel).

### `focusableNodes`

```js
new Splide( '.splide', {
  focusableNodes: 'a, button, textarea, input, select, iframe, video[controls], summary, [tabindex]',
} ).mount();
```

- Source (`Slide.updateVisibility`): a slide is hidden when it's not fully inside the track and is not the active slide. An off-screen clone is hidden even when it's active.
- Hidden slides get `aria-hidden="true"`. This is not updated while moving or scrolling.
- Every descendant matching `focusableNodes` gets `tabindex="-1"`. When the slide is shown again, that `tabindex` is removed, so the elements return to natural tab order.
- The default selector misses `[tabindex]`, `video[controls]`, `summary`, `[contenteditable]` and so on. Extend it for those.
- Read on every visibility update.

### `label` / `labelledby` / `role`

```html
<section class="splide" aria-labelledby="carousel-heading">
  <h2 id="carousel-heading">Featured products</h2>
  …
</section>
```
```js
// A label that also fits after destruction (e.g. avoid "carousel" once it becomes a plain list)
new Splide( '.splide', {
  label: 'My Carousel', // Can be omitted if the label is defined in HTML
  breakpoints: {
    640: {
      destroy: true,
      label  : 'My Gallery', // Used after destruction
    },
  },
} ).mount();

// Decorative carousel: group instead of a landmark
new Splide( '.splide', { role: 'group', label: 'Photos' } ).mount();
```

- `label` sets `aria-label` on the root and `labelledby` sets `aria-labelledby`. Both override the HTML attributes.
- Source: their defaults are read *from* the root's `aria-label` / `aria-labelledby`, so HTML labels survive. Both are re-applied on `updated`, and an empty value removes the attribute. Setting `label` in `Splide.defaults` overrides every carousel's HTML `aria-label`.
- If a visible heading works as the carousel's name, point `aria-labelledby` at it instead of using `aria-label`.
- W3C suggests leaving the word "carousel" out of the label, because `aria-roledescription` is already `carousel` (`i18n.carousel`). Some screen readers announce label + description and some ignore the description, so the docs author considers including "carousel" acceptable.
- `role`:
  - Use a landmark role or `group`. The default is `region`.
  - Pass `''`/`undefined` so Splide doesn't add one.
  - Ignored for a `<section>` root, which already has an implicit role when it has an accessible name, so give it `aria-label`/`aria-labelledby`.
  - Avoid `region` on elements that have or may have a landmark role, e.g. `<header>` (`banner`) and `<nav>`.
  - Source: also skipped when the root already has a `role` attribute. Set once.

### `live`

```js
new Splide( '.splide', { live: false } ).mount();
```

- Source (`Live`): when enabled (and `isNavigation` is off), the track gets `aria-live="polite"` and `aria-atomic="true"`, so screen readers read the slide content after each change.
- While autoplay is playing, `aria-live` switches to `off`, and back to `polite` when it pauses.
- After each `moved`/`scrolled`, Splide briefly (90 ms) appends `<span class="splide__sr">…</span>` and toggles `aria-busy` to force an announcement.
- Readonly.

### `breakpoints` + `mediaQuery`

Desktop-first (the default `mediaQuery: 'max'`) uses `max-width`, so a breakpoint applies at that width **and below**:

```js
new Splide( '.splide', {
  perPage: 4,
  gap    : '2rem',
  breakpoints: {
    1024: { perPage: 3 },
    640 : { perPage: 2, gap: '1rem' },
    480 : { perPage: 1 },
  },
} ).mount();
```

Mobile-first (`mediaQuery: 'min'`) uses `min-width`, so a breakpoint applies at that width **and above**:

```js
new Splide( '.splide', {
  mediaQuery: 'min',
  perPage   : 1,
  breakpoints: {
    640 : { perPage: 2 },
    1024: { perPage: 4, gap: '2rem' },
  },
} ).mount();
```

How it works (source, `components/Media/Media.ts`):

1. **Registration (once, at mount).** Breakpoint keys are sorted numerically: descending for `'max'`, ascending for `'min'`. Each key becomes `matchMedia( '(max-width:{key}px)' )` or `'(min-width:{key}px)'`. Then `reducedMotion` is registered last with `(prefers-reduced-motion: reduce)`. Media is the first component, so this happens before any other component reads options.
2. **On every `change` of any of those queries** (and once at mount):
   1. Start from an empty object.
   2. Merge the options of **every matching query** in that sorted order, then `reducedMotion` if it matches.
   3. Remove all previous breakpoint overrides, so the options go back to the base options (constructor + `data-splide` + any `splide.options = …` changes).
   4. Merge the result on top and emit `updated`.
3. So breakpoints **stack** like CSS media queries:
   - With `'max'`, larger keys merge first and smaller keys override them. At 500px, `1024` and `640` both match, and `640` wins on shared keys.
   - With `'min'`, smaller keys merge first and larger keys override them. At 1200px, `640` and `1024` both match, and `1024` wins.
   - Values from a wider (max) / narrower (min) breakpoint still apply unless a more specific one overrides them.
   - When nothing matches, every option goes back to its base value.
4. Both `max-width` and `min-width` are inclusive: `640` matches at exactly 640px.
5. **Keys must be plain pixel numbers.** The key is put into the query as `{key}px`, so `'40em'` becomes `(max-width:40empx)`, which never matches.
6. Object values (`padding`, `classes`, `i18n`, `dragMinThreshold`, `reducedMotion`) are **deep-merged** into the base object, not replaced.
7. After merging:
   - if the merged `destroy` is truthy, Splide destroys the carousel (`'completely'` → `destroy( true )`);
   - else if the carousel was destroyed, it remounts;
   - else if `direction` changed, it calls `refresh()`.
8. `reducedMotion` always wins over breakpoints, because it's merged last.
9. `breakpoints` objects from `Splide.defaults`, the constructor and `data-splide` are deep-merged, so keys from all three combine.

Which options to put in breakpoints: the responsive ones (marked **(R)**). Readonly options in a breakpoint take effect only if that breakpoint already matches when the carousel mounts or remounts after `destroy: true`. They don't react when the viewport crosses the breakpoint later. Never put `type` in breakpoints (see [type](#type)). See [Readonly options](#readonly-options).

### `destroy` (inside breakpoints)

| `destroy` | Behavior |
|---|---|
| `true` | Destroys the carousel but keeps watching breakpoints, so it can remount |
| `'completely'` | Destroys the carousel completely (breakpoints are no longer watched, sync links are dropped) |

```js
// A carousel for desktop only (destroyed at 640px and below):
new Splide( '.splide', {
  breakpoints: {
    640: {
      destroy: true,
    },
  },
} ).mount();

// A carousel for mobile only (destroyed at 640px and above):
new Splide( '.splide', {
  mediaQuery: 'min',
  breakpoints: {
    640: {
      destroy: true,
    },
  },
} ).mount();
```

Source details:

- When the breakpoint matches at mount, Splide still mounts fully and then destroys right after `ready`. A destroy requested in the `CREATED` state is postponed.
- When the breakpoint stops matching, Splide calls `mount()` again with the same options and extensions.
- After a non-complete destroy:
  - generated arrows, pagination and clones are removed;
  - the slides' inline styles and ARIA attributes are restored;
  - the root loses `is-active`, `splide--{type}`, `splide--{direction}`, its inline `style` and `aria-roledescription`, but keeps `aria-label`, `role` and `is-initialized`.
- The core CSS `.splide.is-initialized:not(.is-active) .splide__list { display: block }` then stacks the slides. Style `.splide:not(.is-active) .splide__list` to lay out the destroyed state, e.g. as a grid.
- All handlers added with `splide.on()` are removed on every destroy and are **not** restored on remount. Re-attach them in an extension's `mount()`, or listen for `mounted` from an extension.
- `splide.options = { destroy: true }` does **not** destroy: only a media-query change checks `destroy`. Call `splide.destroy( false )` (keep watching breakpoints) or `splide.destroy()` (= completely) instead.

### `reducedMotion`

The default value is:

```js
{
  speed      : 0,
  rewindSpeed: 0,
  autoplay   : 'pause',
}
```

- When `(prefers-reduced-motion: reduce)` matches, Splide merges these options on top of everything else: transitions are instant and autoplay doesn't start after page load. The exception is drag/swipe, which counts as "essential" motion. Source: `Drag` lifts the reduced-motion options while a drag is released.
- Don't rely on autoplay alone to change slides unless the carousel is purely decorative. Under reduced motion the user would otherwise have no way to rotate it. You can emulate the media feature in Chrome DevTools: Rendering → "Emulate CSS media feature prefers-reduced-motion".
- Source: your `reducedMotion` object is **deep-merged** with the default. `{ speed: 200 }` keeps `rewindSpeed: 0` and `autoplay: 'pause'`. To opt out, set every key to your normal values:

```js
new Splide( '.splide', {
  autoplay: true,
  speed   : 400,
  reducedMotion: { speed: 400, rewindSpeed: 400, autoplay: true },  // opt out (not recommended)
} ).mount();
```

- Readonly: registered once at mount, like `breakpoints`.

### `classes`

Each key is a class string that Splide puts on an element **it creates**. The `classes` object is deep-merged with the defaults, so keys you leave out keep their defaults. **A value you provide replaces that key's default class entirely.** Splide doesn't add the default class next to yours, so include it yourself. The themes, and Splide's own lookups such as `.splide__arrows` on refresh or `.splide__arrow` / `.splide__pagination__page` in drag detection, rely on the defaults. You can give several classes separated by spaces.

```js
new Splide( '.splide', {
  classes: {
    // Add classes for arrows.
    arrows: 'splide__arrows your-class-arrows',
    arrow : 'splide__arrow your-class-arrow',
    prev  : 'splide__arrow--prev your-class-prev',
    next  : 'splide__arrow--next your-class-next',

    // Add classes for pagination.
    pagination: 'splide__pagination your-class-pagination', // container
    page      : 'splide__pagination__page your-class-page', // each button
  },
} ).mount();
```

Don't remove the default classes, because Splide uses them for styling.

| Key | Default (`constants/classes.ts`) | Applied to (source) |
|---|---|---|
| `slide` | `splide__slide` | Slides inserted with `splide.add()` (slides in your HTML must already have `splide__slide`) |
| `clone` | `splide__slide--clone` | Each loop clone |
| `arrows` | `splide__arrows` | The arrows wrapper `<div>` when Splide creates it |
| `arrow` | `splide__arrow` | Both generated arrow buttons |
| `prev` | `splide__arrow--prev` | The generated previous button |
| `next` | `splide__arrow--next` | The generated next button |
| `pagination` | `splide__pagination` | The generated pagination `<ul>` |
| `page` | `splide__pagination__page` | Each pagination `<button>` |
| `spinner` | `splide__spinner` | The lazy-load spinner `<span>` (also used to find an existing spinner) |

These classes are fixed: the `classes` option doesn't change them. Splide uses them to find your HTML or adds them itself:

| Kind | Class names |
|---|---|
| Structure (lookup) | `splide` (root), `splide__track`, `splide__list`, `splide__slide` (`:not(.splide__slide--clone)`), `splide__slide__container`, `splide__arrows`, `splide__arrow--prev`, `splide__arrow--next`, `splide__pagination`, `splide__progress`, `splide__progress__bar`, `splide__toggle`, `splide__toggle__play`, `splide__toggle__pause`, `splide__sr` |
| Modifiers (added) | `splide--{type}`, `splide--{direction}`, `splide--draggable`, `splide--nav` and the same set on the track as `splide__track--…`; `splide__arrows--{direction}`; `splide__pagination--{direction}` |
| Status (added) | `is-initialized`, `is-active`, `is-prev`, `is-next`, `is-visible`, `is-loading`, `is-focus-in`, `is-overflow` (v4.1+) |

Readonly in the docs. Source: generated arrows keep the class object captured at mount. Pagination re-reads it when it rebuilds.

### `i18n`

Pass an object. Splide merges it into the default collection, so give only the keys you change:

```js
new Splide( '.splide', {
  i18n: {
    prev: 'Previous article',
    next: 'Next article',
  },
} ).mount();

// For every carousel on the page (before any `new Splide`)
Splide.defaults = {
  i18n: {
    prev: 'Diapositive précédente',
    next: 'Diapositive suivante',
  },
};
```

| Key | Default (`constants/i18n.ts`) | Used for (source) |
|---|---|---|
| `prev` | `Previous slide` | `aria-label` of the previous arrow |
| `next` | `Next slide` | `aria-label` of the next arrow |
| `first` | `Go to first slide` | `aria-label` of the **next** arrow when clicking it rewinds to the first slide |
| `last` | `Go to last slide` | `aria-label` of the **previous** arrow when clicking it jumps to the last slide |
| `slideX` | `Go to slide %s` | `aria-label` of pagination buttons (when `perPage` is 1, or with `focus`/`isNavigation`) and of each navigation slide (`isNavigation`) |
| `pageX` | `Go to page %s` | `aria-label` of pagination buttons when `perPage > 1` and there's no `focus` |
| `play` | `Start autoplay` | `aria-label` of the `.splide__toggle` button while autoplay is stopped |
| `pause` | `Pause autoplay` | `aria-label` of the `.splide__toggle` button while autoplay is playing |
| `carousel` | `carousel` | `aria-roledescription` of the root |
| `slide` | `slide` | `aria-roledescription` of each slide |
| `select` | `Select a slide to show` | `aria-label` of the pagination `<ul>` |
| `slideLabel` | `%s of %s` | `aria-label` of each slide: {slide number} of {slide count} |

- Each `%s` is replaced in order: the 1-based number, then the length for `slideLabel`.
- Source: a slide that already has an `aria-label` in the HTML keeps it instead of `slideLabel`.
- Docs (i18n page) say `first` is the label "of a previous arrow" and `last` "of a next arrow". Source (`Arrows.update`): `first` goes on the **next** arrow and `last` on the **previous** arrow, as in the table above.
- Translate the defaults for accessibility, and prefer text based on the content ("Next article" rather than "Next slide").
- See `references/i18n.md`.

---

## Readonly options

The docs split options into **responsive** (marked (R) above) and **readonly**. Readonly options must not be changed after the carousel is created. They may seem to react to changes, but Splide doesn't guarantee it.

How runtime changes reach the components (source):

- `splide.options = { … }` calls `Media.set( opts, base = true, notify = true )`. It deep-merges `opts` into the current options *and* into the base options, then emits `updated`. The setter only works after `mount()`, because before that `this._C` is undefined and it throws.
- A breakpoint change merges the matching options and emits `updated` (see [breakpoints + mediaQuery](#breakpoints--mediaquery)).
- Components that listen to `updated` re-read options: Elements (classes, label), Layout (sizes), Controller (perPage/perMove/end), Move (reposition), Arrows (rebuild), Pagination (rebuild), Drag (`drag`), Keyboard (`keyboard`), Clones (count check), Cover, Sync (`aria-orientation`).
- Options that a component cached when it was created stay as they were until the next `mount()` (after a destroy). Options read in slide objects change on `refresh()`.

Readonly options, and what the source does when you change one after mount:

| Option | Where the source reads it | Changing it after mount |
|---|---|---|
| `type` | `mount()` picks the Fade/Slide transition *before* breakpoints apply; Controller and Scroll cache `isLoop`/`isSlide`; Clones are built at mount | No effect, or a broken state. Never change it, not even in breakpoints. |
| `role` | Elements setup; skipped once the root has a `role` attribute | No effect |
| `start` | Controller construction | No effect (use `go()`) |
| `autoplay` | Autoplay construction | No effect (use `Components.Autoplay.play()` / `pause()`) |
| `pauseOnHover`, `pauseOnFocus` | Listeners bound at mount | No effect |
| `interval` | The timer is created with it; re-read on every `move` (with `data-splide-interval` winning) | Applies from the next move (not guaranteed) |
| `resetProgress` | Read on each `play()` | Re-read (not guaranteed) |
| `lazyLoad` | LazyLoad construction/mount | No effect |
| `preloadPages` | Read on each check | Re-read (not guaranteed) |
| `wheel` | Bound at mount | No effect |
| `wheelMinThreshold`, `wheelSleep`, `releaseWheel` | Read on every wheel event | Re-read (not guaranteed) |
| `cover` | Mount | No effect |
| `live` | Live construction | No effect |
| `isNavigation` | Sync and Live construction; each Slide when created; `hasFocus()` and root classes re-read | Partly applied; inconsistent. Don't. |
| `slideFocus` | Sync setup; each Slide when created | Only after `refresh()` |
| `updateOnMove` | Each Slide when created | Only after `refresh()` |
| `omitEnd` | Controller construction | No effect |
| `mediaQuery`, `breakpoints`, `reducedMotion` | Media construction/setup | No effect until a remount |
| `classes` | Arrows cache it; pagination re-reads on rebuild; `slide`/`clone`/`spinner` read on add/clone/lazy register | Partly applied |
| `i18n` | Elements and Arrows cache it; Slides at creation; pagination and the autoplay toggle re-read | Partly applied |
| `autoWidth`, `autoHeight` | Layout on every resize/`updated`; Clones count | Re-read (not guaranteed) |
| `cloneStatus` | Every slide update | Re-read. Source types even list it as responsive. |
| `noDrag` | Every pointer down | Re-read (not guaranteed) |
| `waitForTransition` | Every `go()` | Re-read (not guaranteed) |
| `arrowPath` | When Splide (re)generates arrows, which happens on every `updated` | Applies to generated arrows (not guaranteed) |
| `focusableNodes` | Every visibility update | Re-read (not guaranteed) |
| `trimSpace` | Every position calculation | Re-read (not guaranteed) |
| `useScroll` (undocumented) | Every transition | Re-read |

Responsive options with caveats:

- `destroy` works only through breakpoints. The setter doesn't destroy.
- `direction`: a breakpoint change triggers `refresh()`. With the setter, call `splide.refresh()` yourself.
- `pagination`: the dots update on `updated`, but the slide `role` (`tabpanel`/`group`) only updates on `refresh()`.

**Breakpoints and readonly options at mount:** Media merges the matching breakpoints (and `reducedMotion`) before any other component is created. So a readonly option inside a breakpoint *does* apply if that breakpoint matches when the carousel mounts, or remounts after `destroy: true`. It's not re-applied when the viewport crosses the breakpoint later. `type` is the exception: the transition is chosen before breakpoints are merged.

---

## Setting options: three ways and precedence

**1. JavaScript.** The constructor takes options as its second argument:

```js
new Splide( '.splide', {
  type   : 'loop',
  perPage: 3,
} ).mount();
```

**2. Data attribute.** Pass JSON in `data-splide`:

```html
<div class="splide" data-splide='{"type":"loop","perPage":3}'>
</div>
```

- The attribute value is in single quotes so the JSON can use double quotes. Otherwise escape them as `&quot;`.
- Options in the data attribute **override** the constructor options.
- Source: the attribute is parsed in the constructor with `JSON.parse`. Invalid JSON throws `Error: [splide] Invalid JSON`.
- JSON can't hold functions (`easingFunc`) or `undefined`. Use `""` to clear `role`.

**3. Static property (advanced).** Defaults shared by every carousel created afterwards:

```js
Splide.defaults = {
  type   : 'loop',
  perPage: 2,
};
```

- Splide uses these as defaults and overrides them with the constructor and data-attribute options.
- Source: `Splide.defaults` starts as an empty object `{}`. The built-in defaults are the internal `DEFAULTS`: not reachable from the classic script, and exported as `DEFAULTS` only by `splide.esm.js`.
- Set `Splide.defaults` (or `Splide.defaults.x = …`) **before** calling `new Splide()`. It's read in the constructor.

**Precedence.** Source, `core/Splide/Splide.ts` constructor (simplified):

```js
options = merge(
  { label: root.getAttribute( 'aria-label' ) || '', labelledby: root.getAttribute( 'aria-labelledby' ) || '' },
  DEFAULTS,            // built-in defaults (constants/defaults.ts)
  Splide.defaults,     // your global defaults
  options || {}        // constructor options
);
merge( options, JSON.parse( root.getAttribute( 'data-splide' ) ) );   // data attribute wins
```

From lowest to highest:

1. HTML `aria-label` / `aria-labelledby` (as `label` / `labelledby`)
2. built-in `DEFAULTS`
3. `Splide.defaults`
4. constructor options
5. `data-splide`

Then, at runtime:

6. matching `breakpoints`, in sorted order
7. `reducedMotion` when the user prefers reduced motion

`splide.options = { … }` (after mount) changes the base layer, i.e. 1–5. While a breakpoint matches, its values still win over the setter on shared keys.

- `merge()` is **deep** for plain objects: `breakpoints`, `classes`, `i18n`, `reducedMotion`, object `padding` and `dragMinThreshold` merge key by key across all layers.
- Arrays are copied. Strings, numbers, booleans and functions are replaced.
- `splide.options` (getter) returns the live options object with the effective values.

---

## Docs vs source: summary

**Undocumented, from source:**

- `useScroll` (boolean). Runs normal moves through the Scroll component (`requestAnimationFrame` + `easingFunc`, duration = `speed`) instead of a CSS transition. See [useScroll](#usescroll-undocumented-from-source).
- The index signature `[ key: string ]: any` in `ResponsiveOptions`, which lets extension options ride along.

**Discrepancies and clarifications** (the source is authoritative; full notes are inline above):

| Topic | Docs | Source |
|---|---|---|
| `cloneStatus` | Readonly | Listed in `ResponsiveOptions` and read on every slide update |
| `keyboard`, `direction` | Responsive | Typed in the non-responsive `Options` interface, but they behave as responsive (`keyboard` re-binds on `updated`; a `direction` breakpoint change triggers `refresh()`) |
| `keyboard` JSDoc | `true` = while focused; `'global'` = window | JSDoc in `types/options.ts` is outdated (`true`/`'global'` → document; `'focused'` adds `tabindex="0"`). The code matches the docs. |
| Keyboard table, ↓ in TTB | "previous page" | Next page |
| `height` | Table: "carousel height"; section: "slide height" | Horizontal: slide (or container) height. TTB: track height (required, or `heightRatio`) |
| `fixedHeight` | "ignores `perPage`, `height` and `heightRatio`" | For the slide height only. A TTB track still needs `height`/`heightRatio`, otherwise it throws `height or heightRatio is missing.` |
| `fixedWidth` | "ignores `perPage`" | Layout only. `perPage` still sets the step size, the page count and `isEnough()` |
| `autoHeight` | General option | Only read in `direction: 'ttb'` |
| `trimSpace` | "if the `focus` option is available" | Trims any `type: 'slide'` carousel. No effect on loop/fade. |
| `omitEnd` | Needs `type: 'slide'` + `focus` | Also needs `trimSpace` not `false` |
| `lazyLoad` | `false`, `'nearby'`, `'sequential'` | `true` behaves like `'nearby'` |
| `waitForTransition` | Default `false`; "forcibly waits on loop points" | Not in `DEFAULTS` (undefined = false). No forced wait found: `isBusy()` checks only this option |
| `rewindSpeed` | Speed on rewind | Only for `type: 'slide'`. Fade ignores it. |
| `flickMaxPages` | Limits flick pages | Ignored in `drag: 'free'` |
| `role` | Ignored for `<section>` | Also ignored when the root already has a `role` attribute |
| `label` / `labelledby` | No default | Default = the root's `aria-label` / `aria-labelledby` |
| `classes` | "provide them with original classes" | Confirmed: a value replaces the default class for that key; nothing is added alongside it |
| `i18n.first` / `i18n.last` (i18n page) | `first` → previous arrow, `last` → next arrow | `first` → next arrow (when it rewinds), `last` → previous arrow (when it jumps to the end) |
| `destroy` | Responsive | Only breakpoint changes act on it. `splide.options = { destroy: true }` does nothing. |
| Inline defaults | `dragMinThreshold` 10, `flickPower` 600, `flickMaxPages` 1, `preloadPages` 1 | Not in `DEFAULTS`; applied inline as `\|\| 10` (touch; mouse `\|\| 0`), `\|\| 600`, `\|\| 1`, `\|\| 1`. `wheelMinThreshold` and `wheelSleep` fall back to `\|\| 0`. `easingFunc` falls back to easeOutQuart. `slideFocus` falls back to `isNavigation`. |
