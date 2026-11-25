# Options

Every Splide option for React Splide 0.7.12, passed through the `options` prop: type, default, what it does, and whether a change after mount reaches the carousel.

Source: https://splidejs.com/guides/options/ (with the i18n table from https://splidejs.com/guides/i18n/ and the reduced-motion notes from https://splidejs.com/guides/accessibility/) and https://splidejs.com/integration/react-splide/. Every option, type and default below was checked against the Splide core source (`src/js/types/options.ts`, `constants/defaults.ts`, `constants/classes.ts`, `constants/i18n.ts`, `components/*`, `core/Splide/Splide.ts`) and React Splide's `src/js/components/Splide/Splide.tsx`. React Splide bundles its own copy of core 4.1.3, which behaves the same as 4.1.4. Claims marked *tested* were run in headless Chrome with React 19, in a production build and in development with StrictMode.

## Contents

- [Conventions and setup](#conventions-and-setup)
- [How the options prop reaches Splide](#how-the-options-prop-reaches-splide)
  - [At mount](#at-mount)
  - [On re-render](#on-re-render)
  - [Live update results](#live-update-results)
  - [Changing readonly options: remount with `key`](#changing-readonly-options-remount-with-key)
  - [Pitfalls of updating options](#pitfalls-of-updating-options)
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
- [Setting options: sources and precedence](#setting-options-sources-and-precedence)
- [Docs vs source: summary](#docs-vs-source-summary)

---

## Conventions and setup

- **(R)** after an option name means the Splide docs mark it as **responsive**: you can change it with `breakpoints` or after mount. Options without (R) are **readonly** in the docs. The **Live** column says what the source really does when the `options` prop changes after mount:

| Live | Meaning |
|---|---|
| Yes | A change in the `options` prop takes effect at once (React Splide sets `splide.options`, which emits `updated`, and the component re-reads it). |
| Next use | Re-read each time Splide uses it (next move, pointer down, wheel event, transition). It works, but the docs don't guarantee it. |
| After refresh | Read when the slide objects are built. Applies after `splide.refresh()` (React Splide calls it when the slide children change) or after a remount. |
| Partly | Some parts update and others keep the mount value. See the row. |
| Remount | Read once at mount. Change the `key` on `<Splide>` to apply a new value (see [Changing readonly options](#changing-readonly-options-remount-with-key)). |

- *(tested)* in the Live column means the behaviour was checked in the browser (see [Live update results](#live-update-results)).
- **Default `unset`** means the option is `undefined` in the source (no entry in `DEFAULTS`). The component then uses an inline fallback, which is listed in the description.
- Where the docs give `false` and the source leaves the value `undefined`, this file writes `false`. The two behave the same, as long as you leave the key out: an explicit `undefined` in your object replaces the default (see [Pitfalls](#pitfalls-of-updating-options)).
- Number values for sizes become `px`. Strings are passed through as CSS (`'10rem'`, `'80vw'`, `'calc(…)'`).
- Type the options with `import type { Options } from '@splidejs/react-splide'`. React Splide re-exports the core type, so you don't need `@splidejs/splide` as a dependency. Under `moduleResolution: "bundler"` (the Vite and Next.js default) the package's types need one tsconfig line; see `references/typescript.md`.

The examples assume this setup. Option snippets on this page are `const name: Options = { … }` objects that you pass as `<Splide options={ name }>`. Declaring them at module level, outside the component, keeps the same object on every render.

```tsx
// Carousel.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Module level: the same object on every render
const options: Options = {
  label  : 'Example carousel',
  perPage: 3,
  gap    : '1rem',
};

const slides = [ '01', '02', '03', '04', '05', '06' ];

export default function Carousel() {
  return (
    <Splide options={ options }>
      { slides.map( text => (
        <SplideSlide key={ text }>{ text }</SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Give the carousel its name with `options.label` (or `options.labelledby`) rather than an `aria-label` prop. In development, StrictMode mounts, destroys and mounts the carousel again, and Splide's `destroy()` strips an `aria-label` that React put on the root. The option survives (tested).

---

## How the options prop reaches Splide

### At mount

`componentDidMount` runs `new Splide( root, options )`, binds the event props, then calls `mount( extensions, transition )`. Every option applies here, including the readonly ones. React Splide keeps a deep copy of the options for later comparisons.

In development with StrictMode, the carousel mounts, is destroyed and mounts again. The options apply both times, so this is harmless.

### On re-render

`componentDidUpdate` runs after every render of `<Splide>`:

1. It deep-compares the new `options` prop with the saved copy. If they differ, it sets `splide.options = options`, which is the core setter: `Media.set( options, true, true )`. That deep-merges **the whole prop object** into the current options and into the base options, then emits `updated` (so `onUpdated( splide, options )` fires). Components that listen to `updated` re-read what they need.
2. It compares the list's child elements with the previous ones (shallow). If they changed, it calls `splide.refresh()`.

What that means in practice (tested):

- Inline objects are fine. `options={ { perPage, gap: '1rem' } }` creates a new object on every render, but equal content doesn't trigger anything (0 `updated` events after a re-render).
- Functions inside options are compared by identity. A new `easingFunc` arrow on every render means an `updated` on every render. Define functions at module level, or with `useCallback`/`useMemo`.
- Only keys present in the prop are written. A key you remove keeps its last value (see [Pitfalls](#pitfalls-of-updating-options)).

Responsive options can come straight from state:

```tsx
// LiveOptions.tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const slides = [ 1, 2, 3, 4, 5, 6 ];

export default function LiveOptions() {
  const [ perPage, setPerPage ] = useState( 2 );
  const [ arrows, setArrows ] = useState( true );

  return (
    <>
      <label>
        Slides per page{ ' ' }
        <select value={ perPage } onChange={ e => setPerPage( Number( e.target.value ) ) }>
          { [ 1, 2, 3 ].map( n => <option key={ n } value={ n }>{ n }</option> ) }
        </select>
      </label>
      <button type="button" aria-pressed={ arrows } onClick={ () => setArrows( on => ! on ) }>
        Arrows
      </button>

      {/* perPage and arrows are responsive: each change reaches Splide through `updated` */}
      <Splide options={ { label: 'Products', perPage, arrows, gap: '1rem' } }>
        { slides.map( n => (
          <SplideSlide key={ n }>Product { n }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

### Live update results

The `options` prop was changed after mount in a production build and in development with StrictMode; the results were the same:

| Change in the `options` prop | Result |
|---|---|
| `perPage: 2` → `3` | Applied: slide width and dot count updated |
| `gap: '1rem'` → key removed | Gap stays `16px` |
| `gap` → `0` | Applied |
| `arrows: false`, then `true` | Arrows removed, then created again |
| `pagination: false`, then `true` | Pagination removed, then created again |
| `drag: false` | `splide--draggable` removed, dragging off |
| `keyboard: true` | ArrowRight now moves the carousel |
| `label` | Root `aria-label` updated |
| `focus`, `padding`, `width`, `rewind` | Applied (dots, track padding, `max-width`, the previous arrow enabled at index 0) |
| `arrowPath` | Generated arrows redrawn with the new path |
| `classes` (`arrow`, `page`) | Pagination buttons get the new class; generated arrows keep the mount classes |
| `i18n` (`prev`, `select`, `slideLabel`) | Pagination `aria-label` updated at once; slide labels only after `refresh()`; arrow labels and the root `aria-roledescription` never |
| `direction: 'rtl'` | Root class and movement switch, but with a `gap` the slides keep the old `margin-right` next to the new `margin-left` until `refresh()` |
| `start` | No effect |
| `autoplay: false` → `true` | No effect, autoplay stays paused |
| `type: 'slide'` → `'loop'` | Broken: clones are added, but `go( '<' )` at index 0 doesn't wrap. With a remount it wraps to the last slide |
| `type: 'slide'` → `'fade'` | Broken: the root class changes, but slides aren't stacked and the active slide stays at `opacity: 0` |
| `destroy: true` | No effect |
| new values inside `breakpoints` | Ignored: breakpoints are registered at mount |
| any change while a breakpoint matches | Keys in both the prop and the breakpoint take the prop's value until the next media query change |
| any change while reduced motion is on | A `speed` in the prop replaces the reduced-motion `0` until the next media query change |

### Changing readonly options: remount with `key`

For a readonly option (marked Remount in the tables), give `<Splide>` a `key` that changes with it. React then unmounts the old carousel (`splide.destroy()`) and mounts a new one, which reads every option and binds the event props again. The new carousel starts at `start` (default `0`), so pass the old index if the position should survive:

```tsx
// TypeSwitch.tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type CarouselType = 'slide' | 'loop' | 'fade';

const slides = [ 1, 2, 3, 4, 5 ];

export default function TypeSwitch() {
  const ref = useRef<Splide>( null );
  const [ type, setType ] = useState<CarouselType>( 'slide' );
  const [ start, setStart ] = useState( 0 );

  function change( next: CarouselType ) {
    setStart( ref.current?.splide?.index ?? 0 ); // keep the current slide across the remount
    setType( next );
  }

  return (
    <>
      { ( [ 'slide', 'loop', 'fade' ] as const ).map( t => (
        <button key={ t } type="button" aria-pressed={ t === type } onClick={ () => change( t ) }>
          { t }
        </button>
      ) ) }

      {/* `type` is readonly: a new key mounts a new carousel */}
      <Splide key={ type } ref={ ref } options={ { label: 'Gallery', type, start, rewind: true } }>
        { slides.map( n => (
          <SplideSlide key={ n }>Slide { n }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested: after two moves, switching to `'fade'` and then `'loop'` gives a working fade carousel and then a loop carousel with clones, both still at index 2.

A remount replaces the DOM state Splide added (classes, ARIA attributes, clones). Anything you attached to the old instance with `splide.on()` or `sync()` has to be attached again; event props are bound again automatically.

### Pitfalls of updating options

**A removed key keeps its value.** The setter merges; it never deletes. Going from `{ perPage: 3, gap: '1rem' }` to `{ perPage: 3 }` leaves the gap at `1rem` (tested). To reset an option, pass its default value explicitly (`gap: 0`, `padding: 0`, `arrows: true`). For an option that is unset by default, such as `focus`, pass `undefined` explicitly (tested: `focus: undefined` brings back one dot per page). Don't do that for options that have a default; see the next point.

**An explicit `undefined` replaces the default.** Splide's merge copies `undefined` like any other value. `{ perPage: props.perPage }` with no prop gives `perPage: undefined`: no pagination dots and a disabled next arrow, because the carousel thinks it has too few slides. `{ i18n: props.labels }` with no labels throws `Cannot read properties of undefined (reading 'carousel')` at mount (both tested). Leave the key out instead (`...( labels && { i18n: labels } )`) or use a fallback (`perPage: props.perPage ?? 1`).

**An update overrides matching breakpoints.** Because the whole prop is written onto the current options, every key in both the prop and an active breakpoint takes the prop's value until the next media query change (tested: `perPage: 4` with `1400: { perPage: 2 }` at 1280px shows 2; changing only `gap` makes it 4; crossing 1400px and back makes it 2 again). The same goes for `reducedMotion`: a `speed` in the prop brings the animation back for users who asked for reduced motion. Two ways out:

- Don't change the options after mount: keep them in a module-level constant or a `useMemo`, and remount with `key` for the rare change.
- Keep responsive keys out of the base: use `mediaQuery: 'min'` with a `0` breakpoint for the smallest screens, so `perPage` only lives inside breakpoints. Leave `speed` out of the prop if you change other options at runtime.

```tsx
// ResponsiveCards.tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// perPage lives only inside breakpoints, so later updates of other options can't override it
const responsive: Options = {
  mediaQuery : 'min',
  breakpoints: {
    0   : { perPage: 1 },
    640 : { perPage: 2 },
    1024: { perPage: 3 },
  },
};

const cards = [ 'A', 'B', 'C', 'D', 'E', 'F' ];

export default function ResponsiveCards() {
  const [ compact, setCompact ] = useState( false );

  return (
    <>
      <button type="button" aria-pressed={ compact } onClick={ () => setCompact( on => ! on ) }>
        Compact
      </button>

      <Splide options={ { ...responsive, label: 'Cards', gap: compact ? '0.5rem' : '2rem' } }>
        { cards.map( card => (
          <SplideSlide key={ card }>Card { card }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested: 3 per page at 1280px, still 3 after toggling the gap, 2 at 800px.

**New breakpoint values are ignored.** Splide registers `breakpoints`, `mediaQuery` and `reducedMotion` once at mount. Changing the values later does nothing (tested). Remount with `key`.

**Event props stop after a breakpoint `destroy: true`.** When a breakpoint destroys the carousel and a later media change remounts it, the core removes every `splide.on()` handler, including the ones React Splide bound for the event props, and doesn't add them back. `onMoved` and the other props are silent from then on (tested). Extensions are run again on every mount, so listen from an extension instead; see [destroy (inside breakpoints)](#destroy-inside-breakpoints).

**Event props see the first render.** Handlers are bound once at mount and never re-read, so a handler that reads state sees the values from the first render. Use a ref or a functional state update; see `references/events.md` and `references/reactivity.md`.

---

## Option tables

### Type and direction

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `type` | `'slide' \| 'loop' \| 'fade'` (string) | `'slide'` | Remount *(tested)* | The carousel type. `'slide'` uses the slide transition. `'loop'` is an endless carousel built with clones. `'fade'` uses the fade transition and does not support `perPage`. To loop a fade carousel, enable `rewind`. A live change breaks the carousel. See [type](#type). |
| `direction` (R) | `'ltr' \| 'rtl' \| 'ttb'` | `'ltr'` | Partly *(tested)*: remount | `'ltr'` = left to right, `'rtl'` = right to left, `'ttb'` = top to bottom (vertical; needs `height` or `heightRatio`). See [direction](#direction). |
| `start` | number | `0` | Remount *(tested)* | The start index. Source: `options.start \|\| 0`, clamped to the valid range. |

### Layout and size

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `width` (R) | number \| string | unset | Yes *(tested)* | The carousel's max width (set as `max-width` on the root). Accepts CSS such as `10em` or `80vw`. |
| `height` (R) | number \| string | unset | Yes | Accepts CSS except `%`. Horizontal: the height of every slide (or of its `.splide__slide__container`). `'ttb'`: the track height. Docs table says "carousel height" and the option section says "slide height". Source does both, by direction. See [height / fixedWidth / fixedHeight / heightRatio](#height--fixedwidth--fixedheight--heightratio). |
| `fixedWidth` (R) | number \| string | unset | Yes | Fixes the width of slides (CSS). The layout ignores `perPage` when this is set. |
| `fixedHeight` (R) | number \| string | unset | Yes | Fixes the height of slides (CSS except `%`). Overrides `height`/`heightRatio` for the slide height. Docs also say it overrides `perPage`. |
| `heightRatio` (R) | number | unset | Yes | Slide height = carousel (list) width × ratio. For example, width `1000` and ratio `0.3` give `300`. Ignored when `height` or `fixedHeight` is set. |
| `autoWidth` | boolean | `false` | Next use | When `true`, each slide's width comes from its own width (your CSS). Don't set `perPage`/`perMove`, or set them to `1`. With `lazyLoad`, give each slide an explicit width. |
| `autoHeight` | boolean | `false` | Next use | When `true`, each slide's height comes from its own height. Don't set `perPage`/`perMove`, or set them to `1`. With `lazyLoad`, give each slide an explicit height. Source: only used when `direction: 'ttb'`. |
| `perPage` (R) | number (integer) | `1` | Yes *(tested)* | The number of slides to display in a page. Must be an integer. |
| `perMove` (R) | number (integer) | unset | Yes | The number of slides to move at once. Must be an integer. Source fallback: `perPage`, or `1` when `focus`/`isNavigation` is set. |
| `gap` (R) | number \| string | unset | Yes *(tested)* | The gap between slides (CSS, e.g. `1em`). It's applied as the slide's `margin-right` (`margin-left` in RTL, `margin-bottom` in TTB). Removing the key doesn't reset it; pass `0`. |
| `padding` (R) | number \| string \| `{ left?, right? }` \| `{ top?, bottom? }` (values number \| string) | unset | Yes *(tested)* | Padding left/right for a horizontal carousel, or top/bottom for a vertical one. One value sets both sides. An object sets each side. Applied to the track. See [padding](#padding). |
| `focus` (R) | number \| `'center'` | unset | Yes *(tested)* | Which slide is active when a page shows several slides. See [focus + trimSpace + omitEnd](#focus--trimspace--omitend). |
| `trimSpace` | boolean \| `'move'` | `true` | Next use | Trims the empty space before the first slide and after the last. `true` trims, and the carousel may stay in place when asked to move. `false` keeps the space. `'move'` trims and always moves when asked. Source: only for `type: 'slide'`. |
| `omitEnd` | boolean | `false` | Remount | Since **v4.1.0**. Omits pagination dots that would only change the active slide without moving the carousel, and disables the next arrow once the last page is reached. Needs `type: 'slide'` (or unset) and `focus`. |
| `cover` | boolean | `false` | Remount | Converts the slide image's `src` into a CSS `background-image` on the parent. Needs `height`, `fixedHeight` or `heightRatio`. Doesn't support `srcset`. See [cover](#cover). |

### Movement and transition

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `rewind` (R) | boolean | `false` | Yes *(tested)* | Rewinds the carousel at either end. Doesn't work in `loop` mode. |
| `speed` (R) | number (ms) | `400` | Yes | The transition speed in milliseconds. With `0`, the carousel jumps to the target slide immediately (source: any value below `1` jumps). |
| `rewindSpeed` (R) | number (ms) | unset | Yes | The transition speed on rewind, in milliseconds. Uses `speed` by default. Source: only for `type: 'slide'`. |
| `easing` (R) | string | `'cubic-bezier(0.25, 1, 0.5, 1)'` | Yes | The timing function for the CSS transition, e.g. `linear`, `ease` or `cubic-bezier()`. |
| `easingFunc` (R) | `( t: number ) => number` | `t => 1 - Math.pow( 1 - t, 4 )` | Yes | The easing function for drag-free mode (easeOutQuart). The default is inline in the Scroll component. Also used by `useScroll` and snapping. Compared by identity: keep the function stable. |
| `waitForTransition` | boolean | `false` | Next use | Disables all actions while the carousel is moving. Docs: even when `false`, the carousel waits for the transition at the loop points. |
| `updateOnMove` | boolean | `false` | After refresh | Updates the `is-active` status class just before the carousel moves, instead of after. |
| `useScroll` | boolean | unset | Next use | **Undocumented, from source.** When truthy, the slide transition is run by the Scroll component (a JS `requestAnimationFrame` animation using `easingFunc` for `speed` ms) instead of a CSS `transform` transition. See [useScroll](#usescroll-undocumented-from-source). |

### Loop clones

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `clones` (R) | number | unset (computed) | Yes (more clones refresh at once; fewer wait for the next refresh) | Sets the number of clones on each side of a loop carousel. Don't set it unless you know how it affects the carousel. |
| `cloneStatus` | boolean | `true` | Next use | Adds `is-active` to clones of the active slide too. Source types list it as responsive. |

### Drag and flick

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `drag` (R) | boolean \| `'free'` | `true` | Yes *(tested)* | `true` enables drag, `false` disables it, `'free'` enables drag-free mode (no snapping to a slide after the drag). |
| `snap` (R) | boolean | `false` | Yes | Snaps to the closest slide in drag-free mode. |
| `rewindByDrag` (R) | boolean | unset | Yes | Lets the user rewind by dragging. `rewind` must be `true`. |
| `noDrag` | string (selector) | unset | Next use | A selector for nodes that can't start a drag. |
| `dragMinThreshold` (R) | number \| `{ mouse: number, touch: number }` | `10` | Yes | The distance needed to start moving the carousel by touch. Pass an object to also set the mouse threshold. Source: touch `\|\| 10` (so `0` isn't accepted) and mouse `\|\| 0`. |
| `flickPower` (R) | number | `600` | Yes | The power of a "flick". The larger it is, the farther the carousel runs. Around 500 is recommended. Inline default `\|\| 600`. |
| `flickMaxPages` (R) | number | `1` | Yes | Limits how many pages a flick can move. Inline default `\|\| 1`. Source: ignored in `drag: 'free'`. |

### Autoplay

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `autoplay` | boolean \| `'pause'` | `false` | Remount *(tested)* | Enables autoplay. With `'pause'`, it doesn't start when the carousel becomes active. You then need a play button or `splide.Components.Autoplay.play()`. With the Intersection extension, you can autoplay only while the carousel is in the viewport. To start and stop at runtime, call `play()`/`pause()` on the instance instead of changing the option. |
| `interval` | number (ms) | `5000` | Next use (from the next move) | The autoplay interval in milliseconds. `data-splide-interval` on a slide overrides it for that slide. |
| `pauseOnHover` | boolean | `true` | Remount | Pauses autoplay on mouseover. Keep it `true` for accessibility. |
| `pauseOnFocus` | boolean | `true` | Remount | Pauses autoplay while the carousel contains the focused element. Keep it `true` for accessibility. |
| `resetProgress` | boolean | `true` | Next use (next `play()`) | Resets the autoplay progress when autoplay is asked to start again. `false` resumes from where it paused. |

### Controls: arrows and pagination

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `arrows` (R) | boolean | `true` | Yes *(tested)* | Creates arrows, or uses the ones found in the markup. Must stay `true` when you provide arrows yourself (`hasTrack={ false }`). |
| `arrowPath` | string | unset | Yes *(tested)*, generated arrows only | Replaces the arrow SVG path, e.g. `'m7.61 0.807-2.12...'`. The SVG is 40×40. Source default path: `m15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z`. |
| `pagination` (R) | boolean | `true` | Yes *(tested)*; slide roles after refresh | Creates pagination (indicator dots). Must be `false` when you use `isNavigation`, or the roles and ARIA attributes break. |
| `paginationKeyboard` (R) | boolean | `true` | Yes | Enables keyboard shortcuts for pagination while it has focus. The W3C Carousel Design Pattern requires this. |
| `paginationDirection` (R) | `'ltr' \| 'rtl' \| 'ttb'` | unset (the `direction` value) | Yes | Sets the pagination direction. It affects the appearance, the shortcuts and the ARIA attributes. |

### Lazy load

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `lazyLoad` | boolean \| `'nearby'` \| `'sequential'` | `false` | Remount | `false` disables lazy loading. `'nearby'` loads only images around the active slide (page). `'sequential'` loads images one after another. Images need `data-splide-lazy` (and optionally `data-splide-lazy-srcset`). Source: `true` behaves like `'nearby'`. |
| `preloadPages` | number | `1` | Next use | How many pages (not slides) around the active slide to load in advance. Only for `'nearby'`. Inline default `\|\| 1`, so `0` acts like `1`. |

### Keyboard and wheel

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `keyboard` (R) | boolean \| `'global'` \| `'focused'` | `false` | Yes *(tested)* | Keyboard shortcuts. `true` = only while the carousel contains focus. `'focused'` = same as `true`. `'global'` = listens on `window` (discouraged). `false` = disabled. |
| `wheel` | boolean | `false` | Remount | Navigates with the mouse wheel. Also set `waitForTransition: true` and/or `wheelSleep`, or the carousel will reach the end at once. |
| `wheelMinThreshold` | number | unset | Next use | The threshold that cuts off the small deltas of inertia scrolling. Inline default `\|\| 0`. |
| `wheelSleep` | number (ms) | unset | Next use | How long to wait, in milliseconds, before accepting the next wheel input. The timer starts when the transition begins. Inline default `\|\| 0`. |
| `releaseWheel` | boolean | `false` | Next use | Releases the wheel event at the first or last slide, so the page can keep scrolling. |

### Navigation and focus

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `isNavigation` | boolean | `false` | Remount | Makes slides clickable so they control another carousel. Link the carousels with `sync()` (see [isNavigation / slideFocus](#isnavigation--slidefocus)). Turn off `pagination` with this. |
| `slideFocus` | boolean | unset (computed) | After refresh (partly): remount | Adds `tabindex="0"` to visible slides. Defaults to `false` without `isNavigation` and `true` with it (source: `slideFocus === undefined ? isNavigation : slideFocus`). Clickable elements must be focusable for keyboard users. |
| `focusableNodes` | string (selector) | `'a, button, textarea, input, select, iframe'` | Next use | A selector for focusable elements inside slides. When a slide gets `aria-hidden="true"`, Splide gives these elements `tabindex="-1"`. |

### Accessibility

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `role` | string | `'region'` | Remount | The `role` attribute for the root. It must be a landmark role or `group`. Pass `''` to leave it unset. Ignored when the root is a `<section>` (`tag="section"`). Source: also ignored when the root already has a `role` attribute (for example a `role` prop). |
| `label` (R) | string | the root's `aria-label` attribute, else `''` | Yes *(tested)* | Sets `aria-label` on the root, overriding the attribute. Prefer it to an `aria-label` prop, which StrictMode loses in development. |
| `labelledby` (R) | string | the root's `aria-labelledby` attribute, else `''` | Yes | Sets `aria-labelledby` on the root, overriding the attribute. |
| `live` | boolean | `true` | Remount | Enables an ARIA live region. Not enabled when `isNavigation` is on. |

### Responsive

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `breakpoints` | `Record<number \| string, ResponsiveOptions>` | unset | Remount *(tested)* | Options for specific breakpoints. `max-width` (desktop-first) by default; set `mediaQuery: 'min'` for `min-width` (mobile-first). |
| `mediaQuery` | `'min' \| 'max'` | `'max'` | Remount | With `'min'`, breakpoints use `min-width`; otherwise `max-width`. Source: undefined, and any value other than `'min'` means max. |
| `reducedMotion` | Options | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Remount | Options applied when `(prefers-reduced-motion: reduce)` matches. |
| `destroy` (R) | boolean \| `'completely'` | `false` | Breakpoints only *(tested)* | `true` destroys the carousel but keeps watching the breakpoints, so it can remount. `'completely'` destroys it for good. Meant for `breakpoints`; in the base options it does nothing after mount. |

### Customization

| Option | Type | Default | Live | Description |
|---|---|---|---|---|
| `classes` | `Record<string, string>` | `CLASSES` (see [classes](#classes)) | Partly *(tested)* | Class names for the elements Splide creates. Each value **replaces** the default for that key, so include the default class too. |
| `i18n` | `Record<string, string>` | `I18N` (see [i18n](#i18n)) | Partly *(tested)* | Text strings. Partial objects are merged into the defaults. |

### Extension options

`ResponsiveOptions` has an index signature (`[ key: string ]: any`), so the options object also takes options for extensions (e.g. `autoScroll`, `intersection`, `video`, `grid`). Splide core ignores keys it doesn't know. They are merged and passed to `breakpoints` like any other option. Pass the extensions themselves through the `extensions` prop. See `references/extensions/`.

---

## Option details and examples

### `type`

```tsx
// Endless carousel (clones are added on both sides)
const loop: Options = { type: 'loop', perPage: 3 };

// Fade. One slide at a time; rewind makes it loop
const fade: Options = { type: 'fade', rewind: true };
```

- `'loop'` clones slides (`.splide__slide--clone`). See [cloneStatus / clones](#clonestatus--clones). `rewind` is ignored in loop. The clones are copies of the slide DOM made by Splide, so React doesn't manage them: state changes inside a slide don't reach its clones until the next `refresh()`.
- `'fade'` doesn't support `perPage`. The core CSS stacks slides (`.splide__track--fade > .splide__list > .splide__slide { opacity:0 }`, `.is-active { opacity:1 }`) and the transition is `opacity {speed}ms {easing}`. `rewindSpeed` isn't used for fade. Drag on a fade carousel moves one slide (`'+'`/`'-'`), or `'>'`/`'<'` with `rewind` + `rewindByDrag`.
- `trimSpace`, `trimSpace: 'move'`, `omitEnd` and the drag "friction" at the edges only apply to `'slide'`.
- **Never change `type` in breakpoints or through the `options` prop.** Source: `mount()` picks the transition (`Fade` or `Slide`) from `type` *before* breakpoints are applied, and the Controller and Scroll cache `isLoop`/`isSlide` when they're built. Tested in React: a live change to `'loop'` or `'fade'` leaves a broken carousel. Remount with `key` (see [TypeSwitch](#changing-readonly-options-remount-with-key)).

### `direction`

```tsx
// Right to left. The root gets `splide--rtl`, and the core CSS sets `direction: rtl` on it
const rtl: Options = { direction: 'rtl', perPage: 2 };

// Vertical. Needs `height` or `heightRatio` (fixedHeight alone is not enough)
const vertical: Options = {
  direction: 'ttb',
  height   : '30rem',
  perPage  : 3,
  gap      : '1rem',
};
```

- Source (`Layout`): in `'ttb'` the track height is `height` (or list width × `heightRatio`) minus the top/bottom padding. If neither is set, Splide throws `[splide] height or heightRatio is missing.` In React that error comes from `componentDidMount`, so it takes down the tree unless an error boundary catches it.
- With `'ttb'`, `gap` becomes `margin-bottom`, `padding` takes `top`/`bottom`, and `fixedWidth`/`autoWidth` logic switches to `fixedHeight`/`autoHeight` (clone count, slide size). Keyboard and pagination keys switch to Up/Down. The theme rotates the arrows (`.splide__arrows--ttb`).
- RTL: keyboard Left/Right are reversed, the drag direction is reversed, and the theme mirrors the arrows (`.splide__arrows--rtl`). Padding object keys stay physical (`left` is still the left side). Splide doesn't read `dir`; set `direction` yourself (see `references/i18n.md`).
- Classes added: `splide--{direction}` on the root, `splide__track--{direction}` on the track, `splide__arrows--{direction}` on the arrows wrapper.
- Source (`Media`): when a breakpoint change alters `direction`, Splide calls `refresh()` automatically. A change through the `options` prop only emits `updated`. Tested: the root class and the movement switch, but with a `gap` the slides keep the old `margin-right` next to the new `margin-left` until `refresh()`. Remount with `key={ direction }`, or call `ref.current?.splide?.refresh()` after the change.

### `start`

```tsx
const startAtFour: Options = { start: 3, perPage: 3 };
```

Read once when the Controller is created. Changing it later does nothing (tested); use `ref.current?.splide?.go( index )` instead, or pass it to a remounted carousel as in [TypeSwitch](#changing-readonly-options-remount-with-key).

### `perPage` / `perMove`

```tsx
// 3 slides visible, move one by one
const threeByOne: Options = { perPage: 3, perMove: 1 };
```

- Both must be integers.
- Source (`Controller`): one step (arrow, `'>'`, `'<'`, autoplay) moves `perMove || ( focus/isNavigation ? 1 : perPage )` slides.
- Pagination dots: `Math.ceil( length / perPage )`, or `endIndex + 1` when `focus`/`isNavigation` is set.
- Slide width (without `fixedWidth`/`autoWidth`): `calc((100% + gap) / perPage - gap)`.
- When there are no more slides than `perPage` (`Slides.isEnough()` is false) and there's no `focus`, the carousel can't move and autoplay won't play.
- `fixedWidth` makes the *layout* ignore `perPage`, but `perPage` still decides the step size, the page count and `isEnough()`. Keep it at `1` with `fixedWidth`/`autoWidth`, or set `perMove`.
- `'fade'` doesn't support `perPage`.
- Live: yes (tested). Beware of `perPage: undefined` from an optional prop: it disables the carousel (see [Pitfalls](#pitfalls-of-updating-options)).

### `focus` + `trimSpace` + `omitEnd`

`focus` decides which slide is active when a page shows several slides. A number is the slot (0-based) where the active slide sits. `'center'` centers it. Source (`Move.offset`): `'center'` → `(listSize - slideSize) / 2`; number `n` → `n × (slideWidth + gap)`.

Once `focus` is set (or `isNavigation` is on):

- each step moves 1 slide unless `perMove` is set;
- the last reachable index is the last slide;
- pagination has one dot per reachable index.

By default Splide trims the empty space at the edges, so the carousel may not move even though the active slide changes:

```tsx
const centered: Options = {
  perPage: 3,
  focus  : 'center',
};
```

To show those spaces, turn trimming off:

```tsx
const untrimmed: Options = {
  perPage  : 3,
  focus    : 'center',
  trimSpace: false,
};
```

To trim the spaces and still have the carousel move every time it's asked to, use `'move'`:

```tsx
const alwaysMove: Options = {
  trimSpace: 'move',
  focus    : 'center',
  perPage  : 3,
};
```

| `trimSpace` | Behavior |
|---|---|
| `true` | Trims spaces. The carousel may stay in the same place even when asked to move. |
| `false` | Disables trimming of the edge spaces. |
| `'move'` | Trims spaces and always moves the carousel when asked (it skips indices that wouldn't change the position). |

Docs say trimming applies "if the `focus` option is available". Source (`Move.trim`): it clamps the position for *any* `type: 'slide'` carousel when `trimSpace` is truthy. It also matters with `fixedWidth`/`autoWidth` without `focus`. It has no effect for `loop`/`fade`.

`omitEnd` (since v4.1.0) removes the redundant pagination dots, i.e. the ones that change only the active slide, and disables the next arrow once the last page is reached. Requirements: `type` is `'slide'` (or unset) and `focus` is set. Source: the end index is found by comparing *trimmed* positions, so `trimSpace` must not be `false`. With 9 slides, this shows 7 dots:

```tsx
const omitted: Options = {
  perPage: 3,
  focus  : 0,
  omitEnd: true,
};
```

It's also useful with `fixedWidth` and `autoWidth`:

```tsx
const autoWidthOmitted: Options = {
  height   : '10rem',
  autoWidth: true,
  focus    : 0,
  omitEnd  : true,
};
```

`focus` is live (tested). `omitEnd` is read when the Controller is created: remount to change it. Removing `focus` from the prop keeps the old value; pass `focus: undefined` to go back to no focus (tested).

### `gap`

```tsx
const spaced: Options = { perPage: 3, gap: '1rem' };   // or gap: 16 (px)
```

Live (tested). Removing `gap` from the prop keeps the old gap; pass `gap: 0`.

### `padding`

```tsx
// By number (px)
const byNumber: Options = { padding: 10 };

// By the CSS format
const byCss: Options = { padding: '1rem' };

// Specifies each value for a horizontal carousel
const horizontal: Options = { padding: { left: 10, right: 20 } };
const horizontalCss: Options = { padding: { left: '1rem', right: '2rem' } };

// Specifies each value for a vertical carousel
const vertical: Options = { direction: 'ttb', height: '20rem', padding: { top: 10, bottom: 20 } };
```

A loop carousel with `20%` padding on each side, so the neighbouring slides peek in:

```tsx
const peek: Options = {
  type   : 'loop',
  padding: '20%',
};
```

Source (`Layout`):

- The padding goes on `.splide__track` (`padding-left`/`padding-right`, or `padding-top`/`padding-bottom` in TTB).
- A missing side in an object is `0`.
- In `'ttb'` the track height is `height − top − bottom`.
- Object keys are physical in RTL (`left` stays left).
- In breakpoints, an object `padding` is deep-merged with an object base `padding` (e.g. `{ left: 0 }` keeps the base `right`). A string or number base is replaced. The same deep merge applies to an update through the `options` prop.

### `width`

```tsx
const narrow: Options = { width: '80%' };
```

Sets `max-width` on the root element. Live (tested).

### `height` / `fixedWidth` / `fixedHeight` / `heightRatio`

```tsx
// Fixed-size slides (perPage is ignored by the layout)
const fixedSize: Options = {
  fixedWidth : '10rem',
  fixedHeight: '6rem',
  gap        : '1rem',
};

// Slide height = 30% of the carousel width (recomputed on resize)
const ratio: Options = {
  heightRatio: 0.3,
};
```

How the source (`Layout`) resolves sizes:

| | Horizontal (`ltr`/`rtl`) | Vertical (`ttb`) |
|---|---|---|
| Slide width | `autoWidth` → none (your CSS); else `fixedWidth`; else `calc((100% + gap)/perPage - gap)` | `fixedWidth` or none |
| Slide height | `fixedHeight`; else `height`; else list width × `heightRatio`; else none (CSS / flex stretch) | `fixedHeight`; else `autoHeight` → none; else `calc((100% + gap)/perPage - gap)` |
| Track height | none | `height` or list width × `heightRatio` (required), minus top/bottom padding |

- Slide heights go on the `.splide__slide__container` child when one exists, and on the slide otherwise. Widths always go on the slide. Splide writes these as inline styles on the `<li>` that `<SplideSlide>` renders; a `style` prop on `<SplideSlide>` sets the starting values, and React re-applies a style property only when its value changes between renders.
- Don't use `%` for `height`/`fixedHeight`.
- Docs: "`fixedHeight` … The carousel will ignore `perPage`, `height` and `heightRatio`." Source: true for the *slide* height. In `'ttb'` the track still needs `height` or `heightRatio`.
- `heightRatio` uses the width of `.splide__list` (the track's content box, i.e. without the track padding).

### `autoWidth` / `autoHeight`

```tsx
// AutoWidthTags.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const tags = [
  { text: 'Design', width: 180 },
  { text: 'Engineering', width: 250 },
  { text: 'Marketing', width: 200 },
  { text: 'Operations', width: 220 },
];

const notes = [
  { text: 'Short note', height: 80 },
  { text: 'A longer note', height: 160 },
  { text: 'Medium note', height: 120 },
];

export default function AutoWidthTags() {
  return (
    <>
      {/* perPage / perMove: leave unset (or 1) */}
      <Splide options={ { label: 'Departments', autoWidth: true, gap: '1rem' } }>
        { tags.map( tag => (
          <SplideSlide key={ tag.text } style={ { width: tag.width } }>{ tag.text }</SplideSlide>
        ) ) }
      </Splide>

      {/* Vertical carousel whose slides keep their own height */}
      <Splide options={ { label: 'Notes', direction: 'ttb', height: '24rem', autoHeight: true } }>
        { notes.map( note => (
          <SplideSlide key={ note.text } style={ { height: note.height } }>{ note.text }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

- Don't set `perPage`/`perMove`, or set them to `1`.
- With `lazyLoad`, every slide needs an explicit width (`autoWidth`) or height (`autoHeight`). Otherwise slides measure 0 before their images load.
- Source: `autoHeight` is read only in vertical mode (`direction: 'ttb'`). A horizontal carousel has no adaptive-height mode. Its slides get `height`/`fixedHeight`/`heightRatio`, or the flex row stretches them to the tallest slide.
- For `loop` with `autoWidth`, the clone count is the number of real slides on each side.
- Pair them with `omitEnd` (and `focus: 0`) to avoid trailing dots that don't move the carousel.
- If a slide's width changes after mount (new content from state), Splide doesn't measure again by itself. Call `ref.current?.splide?.refresh()` (source: `refresh` re-runs the layout and repositions).

### `cover`

```tsx
// CoverPhotos.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const photos = [
  { src: '/images/photo-1.png', alt: 'Red rooftops' },
  { src: '/images/photo-2.png', alt: 'Blue harbour' },
  { src: '/images/photo-3.png', alt: 'Green hills' },
];

export default function CoverPhotos() {
  return (
    <Splide options={ { label: 'Photos', cover: true, heightRatio: 0.5 } }>
      { photos.map( photo => (
        <SplideSlide key={ photo.src }>
          {/* the <img> must be a direct child of the slide or of .splide__slide__container */}
          <img src={ photo.src } alt={ photo.alt } />
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Source (`Cover`):

- Takes the first `<img>` that is a **direct child** of `.splide__slide__container`, or of the slide.
- Sets `background: center/cover no-repeat url("src")` on the container (or the slide), and hides the `<img>` (`display: none`). Tested: the slide gets the background and its height is half its width.
- Needs `height`, `fixedHeight` or `heightRatio`, because the hidden image no longer gives the slide a height.
- Doesn't support `srcset`.
- Works with `lazyLoad`: it re-applies on `lazyload:loaded`.
- Readonly: set once at mount. It re-applies on `updated` and `refresh`, so a new `src` from React shows up after the next options update or children change.
- For new code, CSS `object-fit: cover` on the `<img>` does the same without hiding the image.

### `rewind` / `rewindByDrag`

```tsx
const rewinding: Options = { rewind: true };

const rewindByDrag: Options = {
  rewind      : true,
  rewindByDrag: true,  // dragging past the last slide goes back to the first
};
```

- `rewind` doesn't work in loop mode.
- Source: `rewindByDrag` only acts when `rewind` is also on. For `'slide'` it works when the drag goes past an edge. For `'fade'` it makes a drag use `'>'`/`'<'` instead of `'+'`/`'-'`.
- When the next arrow will rewind, its `aria-label` becomes `i18n.first`. When the previous arrow will jump to the end, its label becomes `i18n.last`.
- Live (tested): the arrows update their disabled state on the `updated` event.

### `speed` / `rewindSpeed`

```tsx
const speeds: Options = {
  speed      : 600,
  rewind     : true,
  rewindSpeed: 1000,
};
```

- `speed: 0` (source: any value below `1`) jumps with no animation.
- Source (`transitions/Slide`): `rewindSpeed` is used only for `type: 'slide'`, when moving from index `0` to the end index (or beyond), or from the end index back to `0`. Without it, `speed` is used.
- `'fade'` always uses `speed`.
- Under `prefers-reduced-motion: reduce`, `reducedMotion` sets both to `0` (see [reducedMotion](#reducedmotion)). If `speed` is in your prop and the prop changes later, the update writes your `speed` back over the reduced-motion `0` (tested). Leave `speed` out of options that change at runtime.

### `easing` / `easingFunc`

```tsx
// Module level, so the function keeps its identity across renders
const easings: Options = {
  easing    : 'ease-in-out',                    // CSS timing function for normal moves and fade
  easingFunc: t => 1 - Math.pow( 1 - t, 3 ),    // JS easing: drag-free, snap, useScroll
};
```

- `easing` is used in the CSS `transition: transform {speed}ms {easing}` (slide/loop) and `opacity {speed}ms {easing}` (fade). It's ignored when `useScroll` is on.
- `easingFunc` takes `t` from 0 to 1 and returns the eased progress. The default is `t => 1 - Math.pow( 1 - t, 4 )`. The Scroll component uses it: drag-free scrolling, snapping, `Controller.scroll()` and `useScroll` transitions.
- React Splide compares functions by identity. An inline `easingFunc: t => …` inside the component's JSX is a new function on every render, so every render calls the options setter (tested: one `updated` per new function), which also overrides matching breakpoints. Keep it at module level or in `useMemo`/`useCallback`.

### `waitForTransition`

```tsx
const waiting: Options = { waitForTransition: true };
```

- When `true`, `go()` (arrows, pagination, keyboard, wheel, autoplay) does nothing while the carousel is `MOVING` or `SCROLLING`, and a new drag can't start mid-transition.
- Default `false`. Source: not in `DEFAULTS`, and `!!undefined` is `false`.
- Docs say "Even if `false`, the carousel forcibly waits for the transition on the loop points." In the v4 source, `Controller.isBusy()` checks only this option. At loop edges, `Move` shifts the list position instead of waiting.
- Recommended with `wheel`.

### `updateOnMove`

```tsx
const earlyActive: Options = {
  updateOnMove: true,
  type        : 'loop',
  perPage     : 3,
  perMove     : 1,
  focus       : 'center',
};
```

- Switches `is-active` when the move *starts* (`move` event), not after it ends (`moved`). This gives CSS transitions on `.is-active` time to run along with the movement.
- Source: read when each Slide object is created, so a change takes effect only after `refresh()`.

### `useScroll` (undocumented, from source)

```tsx
const scrolled: Options = {
  useScroll : true,
  speed     : 800,      // used as the scroll duration
  easingFunc: t => t,   // linear
};
```

- Not on the options page. It's declared in `types/options.ts` ("Determines whether to use the Transition component or not").
- In `transitions/Slide`, a truthy `useScroll` replaces the CSS transition with `Components.Scroll.scroll( destination, speed, false, done )`. That's a `requestAnimationFrame` animation that sets `transform` each frame using `easingFunc`. It is **not** native browser scrolling.
- The `scroll` and `scrolled` events (`onScroll`, `onScrolled`) then also fire on every normal move, and the state is `SCROLLING` during it.
- `easing` is ignored.
- Has no effect on `'fade'`.
- Read on each transition.

### `cloneStatus` / `clones`

```tsx
const noCloneStatus: Options = { type: 'loop', perPage: 3, cloneStatus: false };
const noClones: Options = { type: 'loop', clones: 0 };   // no clones at all (breaks the loop look)
```

- Clones exist only for `type: 'loop'`. They get the `splide__slide--clone` class (`classes.clone`) and the id `{rootId}-clone01`, `-clone02`, and so on.
- `cloneStatus: true` (default) also gives `is-active` to the clones of the active slide. That matters when `perPage`/`focus` shows a clone next to the real slide. Source: read on every slide update.
- Source (`Clones.computeCloneCount`), when `clones` is undefined:
  - with `fixedWidth` (`fixedHeight` in TTB): `Math.ceil( trackWidth / slideWidth )`;
  - with `autoWidth` (`autoHeight` in TTB): the number of real slides;
  - otherwise `perPage × 2`.
- After `updated`/`resize`, Splide refreshes when the required clone count grows or drops to `0`.
- Clones are `cloneNode( true )` copies outside React's control. React handlers don't work on them: React listens at its root container and finds the handler through the element's React internals, which `cloneNode` doesn't copy. Tested: clicking a button inside a clone doesn't run its `onClick`; the same button in the real slide does. Handle clicks with a native listener on the list (read a `data-*` attribute from `event.target`), or keep interactive content out of loop slides.

### `drag: 'free'` + `snap`

```tsx
// Drag freely, no snapping
const free: Options = {
  drag   : 'free',
  perPage: 3,
};

// Drag freely, then snap to the closest slide
const freeSnap: Options = {
  type   : 'loop',
  drag   : 'free',
  snap   : true,
  perPage: 3,
};
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
- Live (tested): `drag: false` removes `splide--draggable` and stops dragging.

### `noDrag`

```tsx
const noDragInputs: Options = {
  noDrag: 'input, textarea, .no-drag',
};
```

- A drag can't start on elements matching the selector.
- Source: pagination buttons (`.splide__pagination__page`) and arrows (`.splide__arrow`) never start a drag, whatever this option says.
- Checked with `matches()` against the pointer-down target itself, so add descendants explicitly (e.g. `'.no-drag, .no-drag *'`).
- Read on every pointer down.

### `dragMinThreshold`

These two settings give the same result:

```tsx
const thresholdNumber: Options = {
  dragMinThreshold: 10,
};

const thresholdObject: Options = {
  dragMinThreshold: {
    mouse: 0,
    touch: 10,
  },
};
```

- The value lets Splide tell whether the user wants to scroll the page vertically or drag the carousel horizontally.
- The touch threshold doesn't accept `0`. Source: `touch || 10`, `mouse || 0`.
- A drag starts only when the move along the carousel axis is larger than the move across it *and* larger than the threshold.

### `flickPower`

```tsx
const softFlick: Options = { flickPower: 500 };
```

- The flick destination is `position + sign(v) × min( |v| × flickPower, limit )`, where `v` is the pointer velocity in px/ms over the last 200 ms.
- Larger values run farther. Around 500 is recommended. The source fallback is `|| 600`.

### `flickMaxPages`

```tsx
const longFlick: Options = { perPage: 2, flickMaxPages: 3 };
```

- Caps the flick distance at `listWidth × flickMaxPages` (fallback `|| 1`).
- Ignored when `drag: 'free'` (the limit is then `Infinity`).

### `autoplay` / `interval` / pause options

```tsx
// AutoplayIntervals.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const promos = [
  { id: 'sale', text: 'Winter sale', interval: 1000 },
  { id: 'new', text: 'New arrivals', interval: 10000 },
  { id: 'ship', text: 'Free shipping' },
];

export default function AutoplayIntervals() {
  return (
    <Splide
      options={ {
        label        : 'Promotions',
        type         : 'loop',
        autoplay     : true,
        interval     : 4000,
        pauseOnHover : true,
        pauseOnFocus : true,
        resetProgress: false,
      } }
    >
      { promos.map( promo => (
        // data-splide-interval overrides `interval` for this slide
        <SplideSlide key={ promo.id } data-splide-interval={ promo.interval }>
          { promo.text }
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

- `autoplay: 'pause'` mounts with autoplay stopped. Start it with a `.splide__toggle` button or `splide.Components.Autoplay.play()`.
- Autoplay plays only when there are more slides than `perPage` (`Slides.isEnough()`).
- Source: `pauseOnHover` uses `mouseenter`/`mouseleave` on the root, and `pauseOnFocus` uses `focusin`/`focusout` on the root.
- Under reduced motion, autoplay becomes `'pause'` (see [reducedMotion](#reducedmotion)).
- An `undefined` interval on a `<SplideSlide>` renders no attribute, so that slide uses `interval`.
- `autoplay` is read once (tested: switching it on through the prop does nothing). For a play/pause control driven by React state, call `ref.current?.splide?.Components.Autoplay.play()` / `pause()`, or remount with `key`.
- Full autoplay UI (toggle, progress bar): `references/autoplay.md`.

### `arrows` / `arrowPath`

```tsx
const squareArrows: Options = {
  arrowPath: 'm15.5 0.932-4.3 4.38 14.5 14.6-14.5 14.5 4.3 4.4 14.6-14.6 4.4-4.3-4.4-4.4-14.6-14.6z',
};
```

- `arrowPath` is the `d` of a single `<path>` inside `<svg viewBox="0 0 40 40" width="40" height="40" focusable="false">`. Draw it on a 40×40 grid, **pointing right**. Both arrows use the same path. The theme mirrors the previous arrow (`.splide__arrow--prev svg { transform: scaleX(-1) }`), swaps the mirroring in RTL and rotates both by ±90° in TTB.
- Source: `arrowPath` only affects arrows that Splide generates, i.e. when the markup has no `.splide__arrow--prev` + `.splide__arrow--next`. Arrows you render yourself (with `hasTrack={ false }` and `<SplideTrack>`) keep their own SVG.
- `arrows: false` hides your own arrows wrapper with `display: none`. With your own arrows, `arrows` must stay `true`.
- Generated buttons get `classes.arrow` + `classes.prev`/`classes.next`. The wrapper (`classes.arrows`) is inserted just before the track, or the buttons go into an existing `.splide__arrows` placeholder.
- Live (tested): generated arrows are removed and rebuilt on every `updated`, with the current `arrowPath`. Don't keep references to the generated buttons across updates; read them from `splide.Components.Arrows.arrows` or use `onArrowsMounted`.
- See `references/arrows.md`.

### `pagination` / `paginationKeyboard` / `paginationDirection`

```tsx
const verticalDots: Options = {
  arrows             : false,
  paginationDirection: 'ttb',   // vertical dots: Up/Down keys, aria-orientation="vertical"
};
```

- Pagination is a `<ul class="splide__pagination splide__pagination--{dir}" role="tablist" aria-label="{i18n.select}">`. It's appended as the last child of the track's parent, or uses an existing `ul.splide__pagination` placeholder. Splide creates these nodes itself; React doesn't know about them.
- Each `<li role="presentation">` holds a `<button class="splide__pagination__page" role="tab" aria-controls="…" aria-label="…">`. Only the current button is tabbable (roving `tabindex`).
- `pagination` also sets the slide role: slides get `role="tabpanel"` when it's `true` and `role="group"` when it's `false`. Source: applied when slides are created, so it changes after `refresh()`.
- Must be `false` with `isNavigation`.
- `paginationKeyboard` (source): while a dot has focus, these keys focus the target dot and go to that page:
  - Right/Left move to the next/previous dot, swapped for RTL, Down/Up for TTB, with wrap-around;
  - `Home` goes to the first dot and `End` to the last.

  The handler stops propagation, so it takes precedence over `keyboard`.
- `paginationDirection` defaults to `direction`. It affects the `splide__pagination--{dir}` class, the key mapping and `aria-orientation` (`vertical` for `'ttb'`).
- Live (tested): the pagination is rebuilt on every `updated`, so `onPaginationMounted` fires again each time.
- See `references/pagination.md`.

### `lazyLoad` / `preloadPages`

| `lazyLoad` | Behavior |
|---|---|
| `false` | Disables lazy loading |
| `'nearby'` | Loads only the images around the active slide (page). Source: `true` behaves the same. |
| `'sequential'` | Loads the images one after another, starting at mount, whatever the current slide |

Each `<img>` needs `data-splide-lazy` with the path or URL of the source file, and optionally `data-splide-lazy-srcset`:

```tsx
// LazyGallery.tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const photos = [ 1, 2, 3, 4, 5, 6 ].map( n => ( {
  src  : `/images/photo-${ n }.png`,
  small: `/images/photo-${ n }-400.png`,
  alt  : `Photo ${ n }`,
} ) );

export default function LazyGallery() {
  return (
    <Splide options={ { label: 'Gallery', lazyLoad: 'nearby', preloadPages: 1, perPage: 2 } }>
      { photos.map( photo => (
        <SplideSlide key={ photo.src }>
          <img
            data-splide-lazy={ photo.src }
            data-splide-lazy-srcset={ `${ photo.small } 400w, ${ photo.src } 800w` }
            sizes="50vw"
            alt={ photo.alt }
          />
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

- Leave `src` off, or give a placeholder `src` that differs from the `data-splide-lazy` value. Splide sets `src`/`srcset` on the DOM node and removes the `data-splide-lazy*` attributes. React doesn't undo that on re-render, because it only touches attributes whose prop value changed (tested: after a re-render the loaded `src` stays). Don't pass the real URL as `src` too, or nothing is lazy.
- `preloadPages` only applies to `'nearby'`. It isn't strictly a page index. It sets the range of indices to load: `perPage × ( preloadPages + 1 ) − 1`. With `perPage: 2` at index `0` that's `3`, so the range is `[ -3, 3 ]` (the negative part covers clones). With `rewind` or `loop`, the distance wraps around the ends. Tested: at mount the first four images load and the fifth and sixth keep `data-splide-lazy`.
- Source (`LazyLoad`):
  - While an image loads, its slide gets `is-loading` and a `<span class="splide__spinner">` (`classes.spinner`) is added to the image's parent. An image with no `src` is hidden until it loads.
  - On load the spinner is removed, the image is shown, and `lazyload:loaded` ( `img, Slide` ) and `resize` are emitted. In React that's `onLazyLoadLoaded( splide, img, Slide )` (capital L in Load).
  - On error, `is-loading` is removed but the spinner stays.
  - Clones are included.
  - Images are found with `[data-splide-lazy], [data-splide-lazy-srcset]` inside each slide.
- Browser-native `loading="lazy"` on the `<img>` is an alternative that needs no option.

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

```tsx
const keys: Options = { keyboard: true };
```

- Docs table: "↓ Go to the previous page in TTB". Source (`Direction` maps `ArrowRight` to `ArrowDown` in TTB): ↓ goes to the **next** page.
- In TTB, ← and → do nothing.
- Older Splide added `tabindex="0"` to the root for `'focused'`. v4 doesn't, because `tabindex="0"` on non-interactive content should be avoided, and a carousel must have a `region` or `group` role, which isn't interactive.
- When pagination has focus and both `keyboard` and `paginationKeyboard` are on, the pagination shortcuts take precedence.
- Source: shortcuts are ignored for one tick right after a move starts. The JSDoc in `types/options.ts` still describes v3 behavior (`true`/`'global'` listen on the document, `'focused'` adds `tabindex="0"`). The code does what the table above says.
- Re-read on `updated`, so it works in breakpoints and through the prop (tested).
- `<Splide>` has no `onKeyDown` prop (its props omit React's DOM handlers); Splide listens with `addEventListener` on the root.

### `wheel` / `releaseWheel` / `wheelMinThreshold` / `wheelSleep`

```tsx
const wheelNav: Options = {
  direction        : 'ttb',
  height           : '20rem',
  wheel            : true,
  releaseWheel     : true,
  waitForTransition: true,   // or wheelSleep: 500. The docs demo also sets this
  wheelMinThreshold: 20,
};
```

- `wheel` listens to `wheel` on the track. Turn on `waitForTransition` and/or set `wheelSleep`, or one wheel gesture fires many `go()` calls and the carousel jumps to the end.
- Source (`Wheel`):
  - Only `deltaY` is used, so a vertical wheel drives horizontal carousels too and horizontal trackpad swipes are ignored.
  - A step happens when `|deltaY| > wheelMinThreshold` (fallback `|| 0`) and more than `wheelSleep` ms (fallback `|| 0`) have passed since the last accepted wheel.
  - `deltaY < 0` goes to `'<'`, otherwise `'>'`.
- `releaseWheel: true` stops calling `preventDefault()` when the carousel is idle and has no slide to go to in that direction (first/last slide), so the page keeps scrolling.
- `wheel` is bound at mount (readonly). `wheelMinThreshold`, `wheelSleep` and `releaseWheel` are read on every wheel event.

### `isNavigation` / `slideFocus`

A thumbnail carousel that controls a main carousel. Link them after both have mounted:

```tsx
// ThumbnailSync.tsx
import { useEffect, useRef } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const mainOptions: Options = {
  label     : 'Product photos',
  type      : 'fade',
  rewind    : true,
  pagination: false,
  arrows    : false,
};

const thumbOptions: Options = {
  label       : 'Thumbnails',
  fixedWidth  : 100,
  fixedHeight : 60,
  gap         : 10,
  rewind      : true,
  pagination  : false,   // required with isNavigation
  isNavigation: true,    // slideFocus defaults to true here
};

const photos = [ 1, 2, 3, 4, 5 ].map( n => ( { src: `/images/photo-${ n }.png`, alt: `Photo ${ n }` } ) );

export default function ThumbnailSync() {
  const main   = useRef<Splide>( null );
  const thumbs = useRef<Splide>( null );

  useEffect( () => {
    const thumbsSplide = thumbs.current?.splide;

    if ( main.current && thumbsSplide ) {
      main.current.sync( thumbsSplide );
    }
  }, [] );

  return (
    <>
      <Splide ref={ main } options={ mainOptions }>
        { photos.map( photo => (
          <SplideSlide key={ photo.src }><img src={ photo.src } alt={ photo.alt } /></SplideSlide>
        ) ) }
      </Splide>

      <Splide ref={ thumbs } options={ thumbOptions }>
        { photos.map( photo => (
          <SplideSlide key={ photo.src }><img src={ photo.src } alt="" /></SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

- `sync()` on a mounted carousel remounts the Sync component of both, which adds the navigation ARIA. Tested in production and under StrictMode: thumbnails get `role="button"` and `Go to slide N`, and clicking the third one moves the main carousel to index 2.
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
- Source, and tested on the core: `refresh()` rebuilds the slides without the navigation step, so navigation slides go back to `role="group"`, `aria-roledescription="slide"` and the `slideLabel` text. React Splide calls `refresh()` whenever the slide children change, so if the thumbnail list changes, remount both carousels (`key`) and sync again.
- See `references/tutorials/` (thumbnail carousel).

### `focusableNodes`

```tsx
const focusables: Options = {
  focusableNodes: 'a, button, textarea, input, select, iframe, video[controls], summary, [tabindex]',
};
```

- Source (`Slide.updateVisibility`): a slide is hidden when it's not fully inside the track and is not the active slide. An off-screen clone is hidden even when it's active.
- Hidden slides get `aria-hidden="true"`. This is not updated while moving or scrolling.
- Every descendant matching `focusableNodes` gets `tabindex="-1"`. When the slide is shown again, that `tabindex` is removed, so the elements return to natural tab order.
- The default selector misses `[tabindex]`, `video[controls]`, `summary`, `[contenteditable]` and so on. Extend it for those.
- Read on every visibility update.

### `label` / `labelledby` / `role`

Point `labelledby` at a visible heading; `useId()` gives a stable id:

```tsx
// LabelledCarousel.tsx
import { useId } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const products = [ 'Linen shirt', 'Wool scarf', 'Canvas bag' ];

export default function LabelledCarousel() {
  const headingId = useId();

  return (
    <>
      <h2 id={ headingId }>Featured products</h2>
      <Splide tag="section" options={ { labelledby: headingId } }>
        { products.map( product => (
          <SplideSlide key={ product }>{ product }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

```tsx
// A label that also fits after destruction (e.g. avoid "carousel" once it becomes a plain list)
const relabelled: Options = {
  label: 'My Carousel',
  breakpoints: {
    640: {
      destroy: true,
      label  : 'My Gallery', // Used after destruction
    },
  },
};

// Decorative carousel: group instead of a landmark
const grouped: Options = { role: 'group', label: 'Photos' };
```

- `label` sets `aria-label` on the root and `labelledby` sets `aria-labelledby`. Both override the attributes.
- Source: their defaults are read *from* the root's `aria-label` / `aria-labelledby`, so attributes survive. Both are re-applied on `updated` (live, tested), and an empty value removes the attribute.
- React: an `aria-label` or `aria-labelledby` prop on `<Splide>` works in production, but under StrictMode in development it's gone after the second mount, because `destroy()` strips it and React doesn't put it back. `options.label` / `options.labelledby` survive (tested). Use the options.
- If a visible heading works as the carousel's name, point `labelledby` at it instead of using `label`.
- W3C suggests leaving the word "carousel" out of the label, because `aria-roledescription` is already `carousel` (`i18n.carousel`). Some screen readers announce label + description and some ignore the description, so the docs author considers including "carousel" acceptable.
- `role`:
  - Use a landmark role or `group`. The default is `region`.
  - Pass `''` so Splide doesn't add one (don't pass `undefined` from a missing prop by accident; it has the same effect).
  - Ignored for `tag="section"`, which already has an implicit role when it has an accessible name, so give it `label`/`labelledby`. The Splide docs recommend `tag="section"` for React.
  - Avoid `region` on elements that have or may have a landmark role, e.g. `tag="header"` (`banner`) and `tag="nav"`.
  - Source: also skipped when the root already has a `role` attribute (a `role` prop on `<Splide>`). Set once.

### `live`

```tsx
const quiet: Options = { live: false };
```

- Source (`Live`): when enabled (and `isNavigation` is off), the track gets `aria-live="polite"` and `aria-atomic="true"`, so screen readers read the slide content after each change.
- While autoplay is playing, `aria-live` switches to `off`, and back to `polite` when it pauses.
- After each `moved`/`scrolled`, Splide briefly (90 ms) appends `<span class="splide__sr">…</span>` and toggles `aria-busy` to force an announcement.
- Readonly.

### `breakpoints` + `mediaQuery`

Desktop-first (the default `mediaQuery: 'max'`) uses `max-width`, so a breakpoint applies at that width **and below**:

```tsx
const desktopFirst: Options = {
  perPage: 4,
  gap    : '2rem',
  breakpoints: {
    1024: { perPage: 3 },
    640 : { perPage: 2, gap: '1rem' },
    480 : { perPage: 1 },
  },
};
```

Mobile-first (`mediaQuery: 'min'`) uses `min-width`, so a breakpoint applies at that width **and above**:

```tsx
const mobileFirst: Options = {
  mediaQuery: 'min',
  perPage   : 1,
  breakpoints: {
    640 : { perPage: 2 },
    1024: { perPage: 4, gap: '2rem' },
  },
};
```

How it works (source, `components/Media/Media.ts`):

1. **Registration (once, at mount).** Breakpoint keys are sorted numerically: descending for `'max'`, ascending for `'min'`. Each key becomes `matchMedia( '(max-width:{key}px)' )` or `'(min-width:{key}px)'`. Then `reducedMotion` is registered last with `(prefers-reduced-motion: reduce)`. Media is the first component, so this happens before any other component reads options.
2. **On every `change` of any of those queries** (and once at mount):
   1. Start from an empty object.
   2. Merge the options of **every matching query** in that sorted order, then `reducedMotion` if it matches.
   3. Remove all previous breakpoint overrides, so the options go back to the base options (the `options` prop at mount, plus every later prop update).
   4. Merge the result on top and emit `updated` (`onUpdated` fires).
3. So breakpoints **stack** like CSS media queries:
   - With `'max'`, larger keys merge first and smaller keys override them. At 500px, `1024` and `640` both match, and `640` wins on shared keys.
   - With `'min'`, smaller keys merge first and larger keys override them. At 1200px, `640` and `1024` both match, and `1024` wins.
   - Values from a wider (max) / narrower (min) breakpoint still apply unless a more specific one overrides them.
   - When nothing matches, every option goes back to its base value.
4. Both `max-width` and `min-width` are inclusive: `640` matches at exactly 640px.
5. **Keys must be plain pixel numbers.** The key is put into the query as `{key}px`, so `'40em'` becomes `(max-width:40empx)`, which never matches. A `0` key with `'min'` always matches (tested), which is useful for the smallest screens.
6. Object values (`padding`, `classes`, `i18n`, `dragMinThreshold`, `reducedMotion`) are **deep-merged** into the base object, not replaced.
7. After merging:
   - if the merged `destroy` is truthy, Splide destroys the carousel (`'completely'` → `destroy( true )`);
   - else if the carousel was destroyed, it remounts;
   - else if `direction` changed, it calls `refresh()`.
8. `reducedMotion` always wins over breakpoints, because it's merged last.

React specifics (tested):

- Breakpoints are evaluated in the browser with `matchMedia`, only after mount. During server rendering and before mount there are no breakpoint styles; the core CSS keeps the carousel hidden until it mounts (`.splide { visibility: hidden }` until `is-initialized`). For layout before mount, add your own CSS media queries.
- The `breakpoints` object is registered at mount. Changing its values through the prop does nothing; remount with `key`.
- Any later change of the `options` prop writes its keys over matching breakpoints until the next media change (see [Pitfalls](#pitfalls-of-updating-options)). Keep the options stable, or keep responsive keys only inside breakpoints (as in [ResponsiveCards](#pitfalls-of-updating-options)).

Which options to put in breakpoints: the responsive ones (marked **(R)**). Readonly options in a breakpoint take effect only if that breakpoint already matches when the carousel mounts or remounts after `destroy: true`. They don't react when the viewport crosses the breakpoint later. Never put `type` in breakpoints (see [type](#type)). See [Readonly options](#readonly-options).

### `destroy` (inside breakpoints)

| `destroy` | Behavior |
|---|---|
| `true` | Destroys the carousel but keeps watching breakpoints, so it can remount |
| `'completely'` | Destroys the carousel completely (breakpoints are no longer watched, sync links are dropped) |

```tsx
// A carousel for desktop only (destroyed at 640px and below):
const desktopOnly: Options = {
  breakpoints: {
    640: {
      destroy: true,
    },
  },
};

// A carousel for mobile only (destroyed at 640px and above):
const mobileOnly: Options = {
  mediaQuery: 'min',
  breakpoints: {
    640: {
      destroy: true,
    },
  },
};
```

Source details:

- When the breakpoint matches at mount, Splide still mounts fully and then destroys right after `ready`. A destroy requested in the `CREATED` state is postponed.
- When the breakpoint stops matching, Splide calls `mount()` again with the same options and extensions.
- After a non-complete destroy:
  - generated arrows, pagination and clones are removed;
  - the slides' inline styles and ARIA attributes are restored;
  - the root loses `is-active`, `splide--{type}`, `splide--{direction}`, its inline `style` and `aria-roledescription`, but keeps `aria-label`, `role` and `is-initialized`.
- The core CSS `.splide.is-initialized:not(.is-active) .splide__list { display: block }` then stacks the slides. Style `.splide:not(.is-active) .splide__list` to lay out the destroyed state, e.g. as a grid.
- All handlers added with `splide.on()` are removed on every destroy and are **not** restored on remount. In React that includes the event props: after the remount, `onMoved`, `onActive` and the rest never fire again (tested). Extensions are run again on every mount, so listen from an extension:

```tsx
// DesktopOnly.tsx
import { useMemo, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type Extension = NonNullable<SplideProps[ 'extensions' ]>[ string ];

const options: Options = {
  label      : 'Highlights',
  breakpoints: {
    640: { destroy: true }, // a plain list at 640px and below
  },
};

// Splide runs extensions again on every mount, so this listener comes back
// after the breakpoint remounts the carousel. An onMoved prop would not.
function indexTracker( onIndex: ( index: number ) => void ): Extension {
  return splide => ( {
    mount() {
      onIndex( splide.index );
      splide.on( 'moved', onIndex );
    },
  } );
}

const slides = [ 1, 2, 3, 4 ];

export default function DesktopOnly() {
  const [ index, setIndex ] = useState( 0 );

  // Read once at mount. setIndex never changes, so the closure can't go stale
  const extensions = useMemo( () => ( { IndexTracker: indexTracker( setIndex ) } ), [] );

  return (
    <>
      <p>Showing slide { index + 1 }</p>
      <Splide options={ options } extensions={ extensions }>
        { slides.map( n => (
          <SplideSlide key={ n }>Slide { n }</SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

  Tested in production and under StrictMode: the text follows the pagination before the destroy and again after the carousel remounts at a wider viewport.
- `destroy: true` in the base options (or a change through the prop) does **not** destroy: only a media-query change checks `destroy` (tested). To remove the carousel from React, render it conditionally, which unmounts `<Splide>` and calls `splide.destroy()`. To turn it into a plain list without unmounting, call `ref.current?.splide?.destroy()` (`destroy( false )` keeps watching breakpoints, so a later media change can mount it again).
- React Splide doesn't know the carousel was destroyed. A re-render with changed options still calls the setter on the destroyed instance, which only updates the stored options.

### `reducedMotion`

The default value is:

```tsx
const reducedMotionDefault: Options = {
  reducedMotion: {
    speed      : 0,
    rewindSpeed: 0,
    autoplay   : 'pause',
  },
};
```

- When `(prefers-reduced-motion: reduce)` matches, Splide merges these options on top of everything else: transitions are instant and autoplay doesn't start after page load. The exception is drag/swipe, which counts as "essential" motion. Source: `Drag` lifts the reduced-motion options while a drag is released.
- Don't rely on autoplay alone to change slides unless the carousel is purely decorative. Under reduced motion the user would otherwise have no way to rotate it. You can emulate the media feature in Chrome DevTools: Rendering → "Emulate CSS media feature prefers-reduced-motion".
- Source: your `reducedMotion` object is **deep-merged** with the default. `{ speed: 200 }` keeps `rewindSpeed: 0` and `autoplay: 'pause'`. To opt out, set every key to your normal values:

```tsx
const optOut: Options = {
  autoplay: true,
  speed   : 400,
  reducedMotion: { speed: 400, rewindSpeed: 400, autoplay: true },  // opt out (not recommended)
};
```

- Readonly: registered once at mount, like `breakpoints`.
- React: a later change of the `options` prop that contains `speed` writes it over the reduced-motion `0` until the next media change (tested). Keep `speed` out of options you change at runtime.

### `classes`

Each key is a class string that Splide puts on an element **it creates**. The `classes` object is deep-merged with the defaults, so keys you leave out keep their defaults. **A value you provide replaces that key's default class entirely.** Splide doesn't add the default class next to yours, so include it yourself. The themes, and Splide's own lookups such as `.splide__arrows` on refresh or `.splide__arrow` / `.splide__pagination__page` in drag detection, rely on the defaults. You can give several classes separated by spaces. (For the root, use the `className` prop on `<Splide>`; React Splide always adds `splide`. For slides, use `className` on `<SplideSlide>`; it always adds `splide__slide`.)

```tsx
const customClasses: Options = {
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
};
```

Don't remove the default classes, because Splide uses them for styling.

| Key | Default (`constants/classes.ts`) | Applied to (source) |
|---|---|---|
| `slide` | `splide__slide` | Slides inserted with `splide.add()` (`<SplideSlide>` always renders `splide__slide`) |
| `clone` | `splide__slide--clone` | Each loop clone |
| `arrows` | `splide__arrows` | The arrows wrapper `<div>` when Splide creates it |
| `arrow` | `splide__arrow` | Both generated arrow buttons |
| `prev` | `splide__arrow--prev` | The generated previous button |
| `next` | `splide__arrow--next` | The generated next button |
| `pagination` | `splide__pagination` | The generated pagination `<ul>` |
| `page` | `splide__pagination__page` | Each pagination `<button>` |
| `spinner` | `splide__spinner` | The lazy-load spinner `<span>` (also used to find an existing spinner) |

These classes are fixed: the `classes` option doesn't change them. Splide uses them to find your markup or adds them itself:

| Kind | Class names |
|---|---|
| Structure (lookup) | `splide` (root), `splide__track`, `splide__list`, `splide__slide` (`:not(.splide__slide--clone)`), `splide__slide__container`, `splide__arrows`, `splide__arrow--prev`, `splide__arrow--next`, `splide__pagination`, `splide__progress`, `splide__progress__bar`, `splide__toggle`, `splide__toggle__play`, `splide__toggle__pause`, `splide__sr` |
| Modifiers (added) | `splide--{type}`, `splide--{direction}`, `splide--draggable`, `splide--nav` and the same set on the track as `splide__track--…`; `splide__arrows--{direction}`; `splide__pagination--{direction}` |
| Status (added) | `is-initialized`, `is-active`, `is-prev`, `is-next`, `is-visible`, `is-loading`, `is-focus-in`, `is-overflow` (v4.1+) |

Readonly in the docs. Source: generated arrows keep the class object captured at mount. Pagination re-reads it when it rebuilds. Tested through the prop: new pagination classes apply at once, the arrows keep the mount classes.

React note: Splide adds its modifier and status classes to the root with `classList`. When React re-renders `<Splide>` with a different `className`, it rewrites the whole `class` attribute, and every class Splide added is gone, `is-initialized` included. The core CSS then hides the carousel (`.splide { visibility: hidden }`). Tested: `className` from `'is-light'` to `'is-dark'` leaves `class="splide is-dark"` and a hidden carousel. Keep `className` constant, and put classes that change on a wrapper element (or use `data-*` attributes).

### `i18n`

Pass an object. Splide merges it into the default collection, so give only the keys you change:

```tsx
const articleTexts: Options = {
  i18n: {
    prev: 'Previous article',
    next: 'Next article',
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
- Source: a slide that already has an `aria-label` (an `aria-label` prop on `<SplideSlide>`) keeps it instead of `slideLabel`.
- Docs (i18n page) say `first` is the label "of a previous arrow" and `last` "of a next arrow". Source (`Arrows.update`): `first` goes on the **next** arrow and `last` on the **previous** arrow, as in the table above.
- Translate the defaults for accessibility, and prefer text based on the content ("Next article" rather than "Next slide").
- `Splide.defaults` can't set texts for every `<Splide>` (React Splide bundles its own core); share an options object instead (see [Setting options](#setting-options-sources-and-precedence)).
- Live: partly (tested). Switch languages with `key`. See `references/i18n.md`.

---

## Readonly options

The docs split options into **responsive** (marked (R) above) and **readonly**. Readonly options must not be changed after the carousel is created. They may seem to react to changes, but Splide doesn't guarantee it. The React docs say the same about the `options` prop: it's reactive, "but do not change readonly options, or the carousel will be broken."

How runtime changes reach the components (source):

- A change of the `options` prop calls `splide.options = options`, which calls `Media.set( opts, base = true, notify = true )`. It deep-merges the prop into the current options *and* into the base options, then emits `updated`.
- A breakpoint change merges the matching options and emits `updated` (see [breakpoints + mediaQuery](#breakpoints--mediaquery)).
- Components that listen to `updated` re-read options: Elements (root classes, label), Layout (sizes), Controller (perPage/perMove/end), Move (reposition), Arrows (rebuild), Pagination (rebuild), Drag (`drag`), Keyboard (`keyboard`), Clones (count check), Cover, Sync (`aria-orientation`).
- Options that a component cached when it was created stay as they were until the next mount (a `key` change, or a remount after a breakpoint destroy). Options read in slide objects change on `refresh()`.

Readonly options, and what the source does when you change one through the `options` prop:

| Option | Where the source reads it | Changing it after mount |
|---|---|---|
| `type` | `mount()` picks the Fade/Slide transition *before* breakpoints apply; Controller and Scroll cache `isLoop`/`isSlide`; Clones are built at mount | Broken state (tested). Never change it, not even in breakpoints. Remount with `key`. |
| `role` | Elements setup; skipped once the root has a `role` attribute | No effect |
| `start` | Controller construction | No effect (tested). Use `go()` |
| `autoplay` | Autoplay construction | No effect (tested). Use `Components.Autoplay.play()` / `pause()` |
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
| `mediaQuery`, `breakpoints`, `reducedMotion` | Media construction/setup | No effect until a remount (tested for `breakpoints`) |
| `classes` | Arrows cache it; pagination re-reads on rebuild; `slide`/`clone`/`spinner` read on add/clone/lazy register | Partly applied (tested) |
| `i18n` | Elements and Arrows cache it; Slides at creation; pagination and the autoplay toggle re-read | Partly applied (tested) |
| `autoWidth`, `autoHeight` | Layout on every resize/`updated`; Clones count | Re-read (not guaranteed) |
| `cloneStatus` | Every slide update | Re-read. Source types even list it as responsive. |
| `noDrag` | Every pointer down | Re-read (not guaranteed) |
| `waitForTransition` | Every `go()` | Re-read (not guaranteed) |
| `arrowPath` | When Splide (re)generates arrows, which happens on every `updated` | Applies to generated arrows (tested) |
| `focusableNodes` | Every visibility update | Re-read (not guaranteed) |
| `trimSpace` | Every position calculation | Re-read (not guaranteed) |
| `useScroll` (undocumented) | Every transition | Re-read |

Responsive options with caveats:

- `destroy` works only through breakpoints. The setter doesn't destroy (tested).
- `direction`: a breakpoint change triggers `refresh()`. Through the prop, remount or call `splide.refresh()` yourself (tested: old margins stay otherwise).
- `pagination`: the dots update on `updated`, but the slide `role` (`tabpanel`/`group`) only updates on `refresh()`.

**Breakpoints and readonly options at mount:** Media merges the matching breakpoints (and `reducedMotion`) before any other component is created. So a readonly option inside a breakpoint *does* apply if that breakpoint matches when the carousel mounts, or remounts after `destroy: true`. It's not re-applied when the viewport crosses the breakpoint later. `type` is the exception: the transition is chosen before breakpoints are merged.

---

## Setting options: sources and precedence

**1. The `options` prop.** The normal way. It's passed to the core constructor at mount and to the setter on later changes.

**2. Shared defaults: an options object, not `Splide.defaults`.** React Splide bundles its own copy of the core, so `Splide.defaults` on `import { Splide } from '@splidejs/splide'` never reaches `<Splide>` (tested: ignored), and a runtime import from `@splidejs/splide` adds a second copy of the core to the bundle. Export a shared object from a module and spread it:

```tsx
// In a shared module, e.g. carousel-defaults.ts
const baseOptions: Options = {
  gap : '1rem',
  i18n: {
    prev: 'Diapositive précédente',
    next: 'Diapositive suivante',
  },
};

// Spread is shallow: merge nested objects yourself, or the inner keys are lost
const productOptions: Options = {
  ...baseOptions,
  perPage: 3,
  i18n   : { ...baseOptions.i18n, next: 'Produit suivant' },
};
```

Unlike `Splide.defaults`, a spread replaces nested objects (`i18n`, `classes`, `breakpoints`, `reducedMotion`) instead of merging them. Splide still merges what you pass with its own built-in defaults.

**3. `data-splide` (not useful in React).** `<Splide>` spreads unknown props onto the root, so a `data-splide` JSON attribute reaches the core constructor, which merges it **over** the options prop. Tested: `options={ { perPage: 2 } }` with `data-splide='{"perPage":3}'` gives 3 at mount, but the first options update writes `perPage: 2` again. Use the prop.

**Precedence.** Source, `core/Splide/Splide.ts` constructor (simplified):

```js
options = merge(
  { label: root.getAttribute( 'aria-label' ) || '', labelledby: root.getAttribute( 'aria-labelledby' ) || '' },
  DEFAULTS,            // built-in defaults (constants/defaults.ts)
  Splide.defaults,     // global defaults (unreachable from React Splide)
  options || {}        // the options prop
);
merge( options, JSON.parse( root.getAttribute( 'data-splide' ) ) );   // data attribute wins
```

From lowest to highest:

1. an `aria-label` / `aria-labelledby` prop on `<Splide>` (as `label` / `labelledby`)
2. built-in `DEFAULTS`
3. `Splide.defaults` (empty for React Splide)
4. the `options` prop
5. a `data-splide` prop

Then, at runtime:

6. matching `breakpoints`, in sorted order
7. `reducedMotion` when the user prefers reduced motion

A later change of the `options` prop changes the base layer (1–5) and also writes the same values onto the current options, so on shared keys it overrides a matching breakpoint or `reducedMotion` until the next media query change (tested).

- `merge()` is **deep** for plain objects: `breakpoints`, `classes`, `i18n`, `reducedMotion`, object `padding` and `dragMinThreshold` merge key by key across all layers.
- Arrays are copied. Strings, numbers, booleans, functions and `undefined` are replaced, so an explicit `undefined` overrides the default (tested).
- `ref.current?.splide?.options` (getter) returns the live options object with the effective values, including the active breakpoint. Treat it as read-only in React; change options through the prop.

---

## Docs vs source: summary

**Undocumented, from source:**

- `useScroll` (boolean). Runs normal moves through the Scroll component (`requestAnimationFrame` + `easingFunc`, duration = `speed`) instead of a CSS transition. See [useScroll](#usescroll-undocumented-from-source).
- The index signature `[ key: string ]: any` in `ResponsiveOptions`, which lets extension options ride along.

**Discrepancies and clarifications** (the source is authoritative; full notes are inline above):

| Topic | Docs | Source |
|---|---|---|
| `options` prop (React docs) | "Reactive", but don't change readonly options | Confirmed, with extra limits: removed keys keep their values, an update overrides matching breakpoints and `reducedMotion`, new breakpoint values are ignored, functions are compared by identity |
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
| `aria-label` prop (React docs examples) | Recommended on `<Splide>` | Lost under StrictMode in development after the remount; `options.label` survives |
| `classes` | "provide them with original classes" | Confirmed: a value replaces the default class for that key; nothing is added alongside it |
| `i18n.first` / `i18n.last` (i18n page) | `first` → previous arrow, `last` → next arrow | `first` → next arrow (when it rewinds), `last` → previous arrow (when it jumps to the end) |
| `destroy` | Responsive | Only breakpoint changes act on it. `destroy: true` in the prop does nothing. |
| Inline defaults | `dragMinThreshold` 10, `flickPower` 600, `flickMaxPages` 1, `preloadPages` 1 | Not in `DEFAULTS`; applied inline as `\|\| 10` (touch; mouse `\|\| 0`), `\|\| 600`, `\|\| 1`, `\|\| 1`. `wheelMinThreshold` and `wheelSleep` fall back to `\|\| 0`. `easingFunc` falls back to easeOutQuart. `slideFocus` falls back to `isNavigation`. |
