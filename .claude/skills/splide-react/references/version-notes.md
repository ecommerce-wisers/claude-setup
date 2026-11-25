# Version notes

Scope: the React Splide release history that matters when you read older code or answers (0.4 to 0.7.12), how to migrate 0.5/0.6 code to 0.7, the core v3 to v4 changes that affect React code, and the known issues of the latest release. Sources: the React Splide repository history (`git log`, commits from 0.6.0 on), the npm registry metadata and published `dist` files of each version, the React Splide docs (current and the archived v3 page), and the core comparison in the vanilla skill's version reference.

## Contents

- [Versions at a glance](#versions-at-a-glance)
- [What 0.7 changed](#what-07-changed)
- [Migrating 0.5 and 0.6 code to 0.7](#migrating-05-and-06-code-to-07)
- [Core v3 to v4 changes that affect React code](#core-v3-to-v4-changes-that-affect-react-code)
- [Changes inside core v4 (4.0 to 4.1)](#changes-inside-core-v4-40-to-41)
- [Known issues in 0.7.12](#known-issues-in-0712)

---

## Versions at a glance

| React Splide | Published | Core | What matters |
|---|---|---|---|
| 0.1.0 to 0.4.4 | 2020-05-31 to 2020-08-29 | v2 (`^2.3.5` to `^2.4.13`), a runtime dependency | First releases. 0.2.0 to 0.4.4 declare `peerDependencies` `react` and `react-dom` `16.13.1` exactly, so newer React versions get peer warnings or install errors. `<SplideSlide>` passes only `className` and `children` to the `<li>` (no `data-*` attributes). |
| 0.5.0 to 0.5.3 | 2021-10-12 to 2021-10-15 | v3 (`^3.1.5`), inlined into the dist | Rewritten in TypeScript. `Splide` (class) and `SplideSlide` "since 0.5.0". No `peerDependencies` from here on. |
| 0.6.0 to 0.6.20 | 2021-10-16 to 2022-01-26 | v3 (`^3.1.6` to `^3.6.12`) | Same props as 0.5. 0.6.8 removed an internal destroy-and-remount from `sync()`. |
| 0.7.0 | 2022-04-05 | v4.0.0 (`^4.0.0`) | The v4 rewrite: `<SplideTrack>`, `hasTrack`, `tag`, HTML attributes on the root, CSS subpaths, renamed and removed props. Next section. |
| 0.7.1 | 2022-04-05 | | `exports` gains the `require` condition (CommonJS, Jest). 0.7.0 exported only the ES module. |
| 0.7.2, 0.7.3 | 2022-04-05, 2022-04-06 | 4.0.1 from 0.7.3 | The `exports` `default` condition points at a misspelled file (`react-splid.esm.js`). Tools that resolve through `import` or `require` are unaffected. |
| 0.7.4 | 2022-04-10 | | `default` condition fixed |
| 0.7.5 | 2022-04-11 | | `children` added to `SplideProps`. With `@types/react` 18 (no implicit `children`), 0.7.0 to 0.7.4 give a type error on `<Splide>` children. |
| 0.7.6 to 0.7.12 | 2022-05-26 to 2022-09-30 | `^4.0.3` to `^4.1.3` | Core updates only. 0.7.12 is the latest release and inlines core 4.1.3, which behaves the same as 4.1.4. The repository hasn't changed since 2022. |

The 0.5/0.6 props, from the 0.6.20 type definitions (0.4.4 has the same set as plain JavaScript):

| 0.5/0.6 prop | Type | What it did |
|---|---|---|
| `id`, `className` | `string` | Set on the root. No other HTML attribute reached the root (no `aria-label`). |
| `options` | `Options` | Core options, reactive as in 0.7 |
| `Extensions` | `Record<string, ComponentConstructor>` | Passed to `mount()` |
| `Transition` | `ComponentConstructor` | Passed to `mount()` |
| `hasSliderWrapper` | `boolean` | Wrapped the track in `<div class="splide__slider">` |
| `hasAutoplayProgress` | `boolean` | Rendered `<div class="splide__progress"><div class="splide__progress__bar">` after the track |
| `hasAutoplayControls` | `boolean` | Rendered `<div class="splide__autoplay">` with `<button class="splide__play">` and `<button class="splide__pause">` |
| `playButtonLabel`, `pauseButtonLabel` | `string` (default `'Play'`, `'Pause'`) | Text of those two buttons |
| `renderControls` | `() => ReactNode` | Extra markup rendered after the track (custom arrows and the like) |
| event props | | The same 28 `on*` props as 0.7 (`onMounted`, `onMoved`, ..., `onLazyLoadLoaded`), called with the instance first |

0.5/0.6 `<SplideSlide>` types omit `style`, `tabIndex`, `role`, `aria-label`, `aria-current`, `aria-controls` and `aria-hidden`; other `<li>` attributes pass through. 0.6 docs import CSS from the core package: `import '@splidejs/splide/dist/css/themes/splide-default.min.css'`.

---

## What 0.7 changed

From the 0.7 commits ("Update components and examples for v4", "Rename `is` to `tag`, `Extensions` to `extensions` and `Transition` to `transition`", "Bypass CSS files", "Remove placeholders") and the published 0.7.0 package:

- **Core v4.** Everything in [Core v3 to v4 changes](#core-v3-to-v4-changes-that-affect-react-code) applies.
- **`<SplideTrack>`** (new) renders `div.splide__track > ul.splide__list`. **`hasTrack`** (default `true`): with `false`, `<Splide>` renders its children as they are and you place `<SplideTrack>` and any controls yourself. This replaces `hasSliderWrapper`, `hasAutoplayProgress`, `hasAutoplayControls`, `playButtonLabel`, `pauseButtonLabel` and `renderControls`, which are gone.
- **`tag`** (new): `'div'` (default), `'section'`, `'header'`, `'footer'` or `'nav'` for the root. (During development it was called `as`; no release shipped `as`.)
- **HTML attributes on the root.** `SplideProps` extends `SplideHTMLAttributes`, React's `HTMLAttributes` minus `DOMAttributes`. `aria-label`, `data-*`, `style` and so on reach the root. React's DOM event handler props are not in the type, and `onClick` is Splide's slide click.
- **`Extensions` and `Transition` renamed** to `extensions` and `transition`. TypeScript rejects the old names. In JavaScript they are ignored: the extension never mounts, the prop lands on the root as an attribute, and React logs "React does not recognize the `Extensions` prop on a DOM element" in development (tested).
- **`<SplideSlide>` accepts every `<li>` attribute**, including `style`, `role` and `aria-*`.
- **CSS subpaths**: `@splidejs/react-splide/css`, `/css/core`, `/css/<theme>`, copied into the React package. The docs say the old `@splidejs/splide/dist/css/...` paths still work; they do only when `@splidejs/splide` resolves from your code (not under pnpm unless it is a direct dependency).
- **`Options` type re-exported**: `import type { Options } from '@splidejs/react-splide'`.

Unchanged from 0.6: the class component with `ref.current.splide`, `go()` and `sync()`, the 28 event props and their arguments, reactive `options`, `refresh()` when the slide elements change, `destroy()` on unmount, and the inlined core (so `Splide.defaults` on `@splidejs/splide` never reached `<Splide>` in 0.5 or 0.6 either).

---

## Migrating 0.5 and 0.6 code to 0.7

The docs' steps:

1. Read the core v4 breaking changes (below).
2. If they affect your carousel, change your code following the core migration guide, except for "Slider Element".
3. Replace the removed props by restructuring the component (example below).
4. Rename `Extensions` to `extensions` and `Transition` to `transition`.
5. Update the CSS imports (short paths are available).
6. Optionally, translate the new i18n strings.
7. Optionally, add `aria-label` or `aria-labelledby` to each carousel. In 0.7 use `options.label` / `options.labelledby` (StrictMode strips the props in development; `references/getting-started.md`).

A typical 0.6 carousel, for recognition only (it doesn't run with 0.7):

```tsx
// React Splide 0.6
import '@splidejs/splide/dist/css/themes/splide-default.min.css';

<Splide
  options={ { type: 'loop', autoplay: true } }
  Extensions={ { MarkReady } }
  hasSliderWrapper
  hasAutoplayProgress
  hasAutoplayControls
  renderControls={ () => (
    <div className="splide__arrows">
      <button className="splide__arrow splide__arrow--prev">Prev</button>
      <button className="splide__arrow splide__arrow--next">Next</button>
    </div>
  ) }
>
  <SplideSlide>01</SplideSlide>
  ...
</Splide>
```

The same carousel in 0.7 (tested in production and development with StrictMode):

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// ComponentConstructor without importing @splidejs/splide.
type ComponentConstructor = NonNullable<SplideProps[ 'extensions' ]>[ string ];

const MarkReady: ComponentConstructor = Splide => ( {
  mount() {
    Splide.root.dataset.extension = 'mounted';
  },
} );

export default function MigratedCarousel() {
  return (
    <Splide
      hasTrack={ false }
      extensions={ { MarkReady } }
      options={ { label: 'Migrated carousel', type: 'loop', autoplay: true } }
    >
      { /* was hasSliderWrapper */ }
      <div style={ { position: 'relative' } }>
        <SplideTrack>
          <SplideSlide>01</SplideSlide>
          <SplideSlide>02</SplideSlide>
          <SplideSlide>03</SplideSlide>
        </SplideTrack>
      </div>

      { /* was renderControls */ }
      <div className="splide__arrows">
        <button className="splide__arrow splide__arrow--prev" type="button">Prev</button>
        <button className="splide__arrow splide__arrow--next" type="button">Next</button>
      </div>

      { /* was hasAutoplayProgress */ }
      <div className="splide__progress">
        <div className="splide__progress__bar" />
      </div>

      { /* was hasAutoplayControls, playButtonLabel and pauseButtonLabel */ }
      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Play</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

Tested: the extension mounted, Splide used the two custom arrows (prev labelled `Go to last slide`, because the carousel loops), the toggle read `Pause autoplay`, the progress bar moved, and the loop clones were created.

Notes on the conversion:

- `hasSliderWrapper` becomes a plain `position: relative` wrapper, not `splide__slider`: v4 ignores that class (only the Sea Green theme still styles it, see `references/themes.md`).
- `hasAutoplayControls` rendered two separate buttons. v4 supports one toggle button. Its visible text is whatever you put in the two spans; its `aria-label` comes from `i18n.play` / `i18n.pause`. Splide doesn't style or label separate `splide__play` / `splide__pause` buttons any more. To keep two buttons, call `ref.current?.splide?.Components.Autoplay.play()` and `.pause()` from your own `onClick` handlers.
- `renderControls` markup goes straight into the children with `hasTrack={ false }`. Keep the `.splide__arrows` wrapper: custom arrows without it throw on mount.
- Add `type="button"` to every button (the React Splide docs examples omit it, in 0.6 and 0.7).
- `ComponentConstructor` isn't exported by React Splide. `NonNullable<SplideProps[ 'extensions' ]>[ string ]` gives the same type without importing `@splidejs/splide`.

---

## Core v3 to v4 changes that affect React code

React Splide 0.7 inlines core v4. These core changes (details in the vanilla skill's version reference, summarized here) change how React code must be written.

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

In React, the root's `tabIndex={ 0 }` prop works in production but is removed during the StrictMode remount in development (`references/structure.md`).

### New options that matter in React

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

`label`, `labelledby` and `role` are the reliable way to label the root in React (the props of the same name don't survive StrictMode in development). `reducedMotion`: when the user prefers reduced motion, `speed` and `rewindSpeed` become `0` and autoplay starts paused, so the carousel must still be operable with arrows or pagination.

### i18n keys added in v4

| Key | Default | Used for |
|---|---|---|
| `carousel` | `'carousel'` | Root `aria-roledescription` |
| `slide` | `'slide'` | Slide `aria-roledescription` |
| `slideLabel` | `'%s of %s'` | Slide `aria-label` (`{slide number} of {slide count}`) |
| `select` | `'Select a slide to show'` | Pagination `aria-label`. **Not in the docs' list.** |

Unchanged keys: `prev` `'Previous slide'`, `next` `'Next slide'`, `first` `'Go to first slide'`, `last` `'Go to last slide'`, `slideX` `'Go to slide %s'`, `pageX` `'Go to page %s'`, `play` `'Start autoplay'`, `pause` `'Pause autoplay'`. See `references/i18n.md`.

### Breakpoints behave like media queries

v3 updated the current options with the matching breakpoint, so resizing and reloading could differ. v4 merges the options of **every** matching breakpoint into the base options, and a key set only inside a breakpoint resets to its default outside it. Example from the docs, opened on a mobile (640px or less):

```tsx
<Splide
  options={ {
    arrows     : true,
    perPage    : 3,
    breakpoints: {
      1200: { arrows: false },
      800 : { perPage: 2 },
      640 : {},
    },
  } }
>
```

gives 2 slides and no arrows, because `1200` and `800` also match. React note: core reads `breakpoints` when the carousel mounts (`Media.ts` keeps the objects it got at setup), so changing `breakpoints` inside the `options` prop later doesn't register new media queries. Tested: adding a matching `5000: { perPage: 3 }` breakpoint through the prop after mount left the carousel at 1 per page. Change `key` to remount if the breakpoints themselves must change.

### Removed slider element and autoplay buttons

- `arrows: 'slider'` and `pagination: 'slider'` are gone (use `true`), and so is the `splide__slider` element (in React 0.6, `hasSliderWrapper`).
- The separate play and pause buttons (`splide__autoplay`, `splide__play`, `splide__pause`; React 0.6 `hasAutoplayControls`) are replaced by one `button.splide__toggle` with `span.splide__toggle__play` and `span.splide__toggle__pause`.

### Markup, ARIA and CSS selectors

- v4 adds `role="region"` (unless the root is a `<section>` or has a role), `aria-roledescription`, slide roles and labels, a live region on the track, and pagination tab roles. Every value: `references/structure.md`.
- The track no longer has to be a direct child of the root, which is what makes `hasTrack={ false }` with wrappers possible.
- Core CSS now targets track modifiers. Custom CSS written for v3 such as `.splide--fade > .splide__track > .splide__list > .splide__slide` should use `.splide__track--fade > .splide__list > .splide__slide`. The full selector table is in `references/themes.md` and the vanilla skill.

| Change | v3 | v4 |
|---|---|---|
| Removed | `splide__slider` | none |
| Removed | `splide__autoplay`, `splide__play`, `splide__pause` | none |
| Added | none | `splide__toggle`, `splide__toggle__play`, `splide__toggle__pause` |
| Added | none | `splide__sr` (visually hidden live-region text, appended to the track briefly) |
| Added | none | `is-focus-in` on the root (4.0.0) |
| Added | none | `is-overflow` on the root (4.1.0) |
| Added | none | Track modifiers `splide__track--{slide\|loop\|fade}`, `splide__track--{ltr\|rtl\|ttb}`, `splide__track--draggable`, `splide__track--nav`. The root still gets `splide--*`. |
| Added | none | `splide__arrows--{ltr\|rtl\|ttb}`, `splide__pagination--{ltr\|rtl\|ttb}` |
| Unchanged | `splide`, `splide__track`, `splide__list`, `splide__slide`, `splide__slide--clone`, `splide__slide__container`, `splide__arrows`, `splide__arrow`, `splide__arrow--prev`, `splide__arrow--next`, `splide__pagination`, `splide__pagination__page`, `splide__progress`, `splide__progress__bar`, `splide__spinner`, `is-initialized`, `is-active`, `is-visible`, `is-prev`, `is-next`, `is-loading`, `is-rendered` | same |

`STATUS_CLASSES` (removed from slides on destroy): v3 `is-active, is-visible, is-prev, is-next, is-loading`. v4.0 adds `is-focus-in`, v4.1 adds `is-overflow`.

### Events

| Event | v3 | v4 |
|---|---|---|
| `repositioned` | Emitted after the list was repositioned (on resize) | **Removed** in 4.0.0. The nearest public event is `resized`. |
| `shifted` | Public (`'shifted'`) | Public through 4.0.17. **Internal since 4.1.0** (value `'sh'`, exported as `EVENT_SHIFTED` in ESM only) |
| `slide:keydown` | Public (`'slide:keydown'`) | Public through 4.0.17. **Internal since 4.1.0** (value `'sk'`, `EVENT_SLIDE_KEYDOWN`) |
| `overflow` | none | **New in 4.1.0**. Argument `isOverflow: boolean`. Fires when overflow status changes (`is-overflow` toggles). |
| `ei` (`EVENT_END_INDEX_CHANGED`) | none | New internal event in 4.1.0 |

Unchanged: `mounted`, `ready`, `move`, `moved`, `click`, `active`, `inactive`, `visible`, `hidden`, `refresh`, `updated`, `resize`, `resized`, `drag`, `dragging`, `dragged`, `scroll`, `scrolled`, `destroy`, `arrows:mounted`, `arrows:updated`, `pagination:mounted`, `pagination:updated`, `navigation:mounted`, `autoplay:play`, `autoplay:playing`, `autoplay:pause`, `lazyload:loaded`. Arguments: `references/events.md`.

React Splide has no event prop for `overflow`, `shifted`, `slide:keydown` or `repositioned`, in 0.6 or 0.7. Listen to `overflow` with `splide.on( 'overflow', fn )`, inside an extension if you need the first value (`references/events.md`). The string names `'shifted'` and `'slide:keydown'` never fire in 4.1 or later.

### Components and instance API

Reached through `ref.current?.splide` in React:

| v3 | v4 |
|---|---|
| `Components.Move.isBusy()` | Moved to `Components.Controller.isBusy()`. Returns `true` only while moving/scrolling **and** `waitForTransition` is on. |
| `Components.Controller.scroll( destination, useIndex?, snap?, duration?, callback? )` | `Components.Controller.scroll( destination, duration?, snap?, callback? )`. `destination` is a **position** in px. The `useIndex` flag is gone. To scroll to an index: `Controller.scroll( Components.Move.toPosition( index, true ), duration )`. |
| `Components.Scroll.scroll( position, duration?, callback? )` | `Components.Scroll.scroll( position, duration?, snap?, callback? )`. The source has a 5th internal `noConstrain?` parameter. |

| Area | v3 | v4 |
|---|---|---|
| `Components.Elements` | `root, slider, track, list, slides, arrows, prev, next, bar, autoplay, play, pause` | `root, track, list, slides, arrows?, pagination?, prev?, next?, bar?, toggle?` |
| `Components.Layout` | `listSize, slideSize, sliderSize(), totalSize, getPadding` | adds `sliderSize( withoutGap? )`, `isOverflow()` (4.1), `resize( force? )` |
| `Components.Arrows` | `arrows` | adds `update()` |
| `Components.Move` | had `isBusy()` | `isBusy()` removed. Adds `reposition()`. |
| `Components.Live` | none | New: `disable( disabled: boolean )` |
| `splide.event` | `EventBusObject` | `EventInterfaceObject` |
| ESM exports | `Splide`, `SplideRenderer`, `EventBus`, `EventInterface`, `RequestInterval`, `State`, `Throttle`, `CLASSES`, `STATUS_CLASSES`, `CLASS_*` (including `CLASS_SLIDER`, `CLASS_AUTOPLAY`, `CLASS_PLAY`, `CLASS_PAUSE`), `EVENT_*` (including `EVENT_REPOSITIONED`) | `EventBus` removed. Added `EventBinder`, `DEFAULTS`, `SLIDE`/`LOOP`/`FADE`, `LTR`/`RTL`/`TTB`, `CLASS_TOGGLE`, `CLASS_TOGGLE_PLAY`, `CLASS_TOGGLE_PAUSE`, `CLASS_SR`, `CLASS_FOCUS_IN`, `CLASS_OVERFLOW`, `EVENT_OVERFLOW`, `EVENT_END_INDEX_CHANGED`. Removed `CLASS_SLIDER`, `CLASS_AUTOPLAY`, `CLASS_PLAY`, `CLASS_PAUSE`, `EVENT_REPOSITIONED`. |

`EventBus` is gone. Inside an extension, use `EventInterface( Splide )` imported from `@splidejs/splide` (this adds a second copy of core to the bundle), or `Splide.on( 'move.myExt', fn )` / `Splide.off( 'move.myExt' )` on the instance the extension receives. `Components.Elements.toggle` replaces `autoplay`, `play` and `pause`.

---

## Changes inside core v4 (4.0 to 4.1)

Useful when a 0.7.0 to 0.7.10 project (core 4.0.x) is upgraded to 0.7.11 or 0.7.12 (core 4.1.x):

| Version | Change |
|---|---|
| 4.0.3 | Track `aria-atomic` and `aria-busy` added to the live region (docs' accessibility page marks these `^4.0.3`) |
| 4.1.0 | `omitEnd` option |
| 4.1.0 | `is-overflow` root class, `overflow` event, `Layout.isOverflow()`; core CSS hides pagination while not overflowing |
| 4.1.0 | `shifted` and `slide:keydown` became internal (`'sh'`, `'sk'`), plus the new internal `'ei'` event |
| 4.1.0 | Fade slides no longer absolutely positioned in core CSS |
| 4.1.0 | Class constants built from `CLASS_PREFIX` (`splide__`) and `STATUS_CLASS_PREFIX` (`is-`). No class names changed. |
| 4.1.3 | Bundled by React Splide 0.7.12. 4.1.4 changed only `package.json`. |

---

## Known issues in 0.7.12

No release after 0.7.12 fixes these. Each one is tested and has a workaround:

| Issue | Workaround | Details |
|---|---|---|
| Types don't resolve under `moduleResolution: "bundler"`, `node16` or `nodenext` (no `types` condition in `exports`), so every import is `any` | One tsconfig line | `references/typescript.md` |
| No `'use client'` directive in the dist | Mark the component that renders `<Splide>` as a client component | `references/nextjs.md` |
| Event props are captured at mount and never re-read (stale state in handlers) | Read the latest values from a ref, or use functional `setState` | `references/reactivity.md` |
| `Splide.defaults` on `@splidejs/splide` doesn't reach `<Splide>` (inlined core) | Share an options object or a wrapper component | `references/getting-started.md` |
| `aria-label`, `aria-labelledby`, `role`, `tabIndex` and `style` props on the root are stripped by `destroy()` during the StrictMode remount (development only) | `options.label`, `options.labelledby`, `options.role`; style with classes | `references/structure.md` |
| A changed `className` on `<Splide>` or `<SplideSlide>` wipes Splide's classes; the root loses `is-initialized` and turns invisible | Keep those class names constant; put changing classes on a wrapper or the slide content | `references/themes.md`, `references/structure.md` |
| `extensions` and `transition` are spread onto the root as attributes (`extensions="[object Object]"`) | Harmless; ignore it | `references/structure.md` |
| With `type: 'loop'`, once the slide list has changed, every later re-render of the component calls `splide.refresh()` (the stored child list includes the old clones, which each refresh replaces). Tested: 1 refresh per unrelated state update. Calling `setState` inside `onRefresh` then loops until React aborts with "Maximum update depth exceeded" (error #185). | Give `<Splide>` a `key` derived from the slides (`key={ items.map( item => item.id ).join() }`) so a change remounts it. Tested: no refreshes on later re-renders. Never set state in `onRefresh`. | `references/reactivity.md` |
| A nested `<Splide>` mounts before its parent, so the parent's control placeholders after its track are ignored | Put the parent's controls before its `<SplideTrack>` | `references/structure.md` |
