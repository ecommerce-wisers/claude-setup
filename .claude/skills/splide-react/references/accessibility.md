# Accessibility

What React Splide's core does for accessibility (roles, ARIA attributes, keyboard, focus, the live region, autoplay controls, reduced motion), and what React changes about it.

Source: https://splidejs.com/guides/accessibility/ and https://splidejs.com/integration/react-splide/

React Splide 0.7.12 (`@splidejs/react-splide`) bundles its own copy of Splide core 4.1.3, which behaves the same as 4.1.4. The core facts were checked against the 4.1.3 source (`src/js/components/*`). Facts marked **(source)** come from the code and are not on the docs page. **(tested)** means the example ran in headless Chrome with React 19.3, as a production build and in development under StrictMode.

## Contents

- [Overview](#overview)
- [Setup](#setup)
- [What React changes](#what-react-changes)
- [Basic vs Tabbed pattern](#basic-vs-tabbed-pattern)
- [Landmark, role and accessible name](#landmark-role-and-accessible-name)
- [Keyboard interaction](#keyboard-interaction)
- [Roving tabindex](#roving-tabindex)
- [Focus management](#focus-management)
- [Live region](#live-region)
- [Autoplay play/pause (WCAG 2.2.1, 2.2.2)](#autoplay-playpause-wcag-221-222)
- [Reducing motion (WCAG 2.3.3)](#reducing-motion-wcag-233)
- [Focus visible (WCAG 2.4.7)](#focus-visible-wcag-247)
- [Focusable elements in hidden slides (WCAG 1.3.1)](#focusable-elements-in-hidden-slides-wcag-131)
- [Role and ARIA attribute reference (WCAG 4.1.2)](#role-and-aria-attribute-reference-wcag-412)
- [Generated IDs](#generated-ids)
- [Accessible markup example](#accessible-markup-example)
- [What destroy does to attributes](#what-destroy-does-to-attributes)
- [Checklist](#checklist)
- [Not handled automatically](#not-handled-automatically)
- [Docs vs source](#docs-vs-source)
- [Open concerns from the docs](#open-concerns-from-the-docs)

## Overview

Splide v4 follows the W3C (WAI-ARIA APG) Carousel Design Pattern. It also adds a live region so screen readers read the new content when slides change. The docs point out that the W3C pattern contradicts itself and does not cover every carousel type: its examples are "fade" carousels, and applying them unchanged to a "sliding" carousel breaks it. Splide's behavior below is the author's adaptation.

Everything in this file is done by the core inside `<Splide>`: React Splide adds no accessibility code of its own. It renders the root (`tag`, default `div`), the track (`div.splide__track`), the list (`ul.splide__list`) and the slides (`li.splide__slide`), and the core then sets the attributes on those elements after mount.

Related options and their defaults (from `constants/defaults.ts` and the Options docs). Pass them in the `options` prop:

| Option | Type | Default | Role in accessibility |
|---|---|---|---|
| `role` | `string` | `'region'` | `role` on the root. Ignored for `<section>` (`tag="section"`) |
| `label` | `string` | root's `aria-label` | `aria-label` on the root. Responsive option |
| `labelledby` | `string` | root's `aria-labelledby` | `aria-labelledby` on the root. Responsive option |
| `keyboard` | `boolean \| 'global' \| 'focused'` | `false` (not set) | Custom arrow-key shortcuts |
| `paginationKeyboard` | `boolean` | `true` | Tabs-pattern keys on pagination |
| `paginationDirection` | `'ltr' \| 'rtl' \| 'ttb'` | same as `direction` | Pagination key direction and `aria-orientation` |
| `slideFocus` | `boolean` | `false`, or `true` when `isNavigation` | `tabindex="0"` on visible slides |
| `focusableNodes` | `string` (selector) | `'a, button, textarea, input, select, iframe'` | Elements that get `tabindex="-1"` inside hidden slides |
| `live` | `boolean` | `true` | Live region on the track. Never active with `isNavigation` |
| `pauseOnHover` | `boolean` | `true` | Pauses autoplay on hover |
| `pauseOnFocus` | `boolean` | `true` | Pauses autoplay while focus is inside |
| `reducedMotion` | `Options` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Applied under `(prefers-reduced-motion: reduce)` |
| `i18n` | `Record<string, string>` | see `references/i18n.md` | All ARIA label texts |
| `waitForTransition` | `boolean` | `false` | When `true`, focus can move in pagination without the slide following during a transition |

## Setup

```bash
npm install @splidejs/react-splide
```

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css'; // default theme, includes the focus styles and .splide__sr
```

Under `moduleResolution: "bundler"` (the Vite and Next.js default) the package's types need one tsconfig line. See `references/typescript.md`.

## What React changes

| Topic | What happens | What to do |
|---|---|---|
| Label passed as a prop | **(tested)** In development, StrictMode mounts, destroys and mounts again. `destroy()` removes `aria-label` and `aria-labelledby` from the root, React does not put them back, and the second mount reads the now-empty attributes. The carousel ends up unnamed. Production is not affected | Use `options={ { label: '...' } }` or `options={ { labelledby: 'heading-id' } }`. Both survive (tested) |
| `role` passed as a prop | **(tested)** Same mechanism: `role="group"` as a prop is removed by the StrictMode destroy, and the second mount sets the default `role="region"` | Use `options={ { role: 'group' } }` (tested) |
| `aria-label` on `<SplideSlide>` | **(tested)** Kept in both modes: Splide restores a slide's own `aria-label` on destroy | Nothing |
| `className` that changes after mount | **(tested)** React rewrites the whole `class` attribute, which drops the classes Splide added: on a `.splide__toggle` this removes `is-active` (the button shows "Start" while autoplay runs); on `<Splide>` it removes `is-initialized`, and the carousel turns invisible | Keep `className` stable on `<Splide>`, `<SplideSlide>` and elements Splide decorates. Style state with a wrapper or a data attribute |
| Event props | Bound once, before `mount()`. **(tested)** They run before Splide's own handlers for the same event, and they are not bound again after a breakpoint `destroy` remount | See `references/events.md` |
| `extensions` prop | Read once, at mount, and remembered for breakpoint remounts | Define extensions outside the component or keep them in `useState` |
| Unmount | `componentWillUnmount` calls `splide.destroy()`, a complete destroy | See [What destroy does to attributes](#what-destroy-does-to-attributes) |
| Server rendering | The HTML has no ARIA attributes until the core mounts in the browser, and the core CSS keeps `.splide` at `visibility: hidden` until `is-initialized` | See `references/nextjs.md` |

## Basic vs Tabbed pattern

The W3C pattern has three sub-patterns. Every Splide carousel uses one of them, chosen by whether pagination exists:

| Pattern | Applied to |
|---|---|
| Basic | Carousels without pagination (`pagination: false`) |
| Tabbed | Carousels with pagination. The pagination follows the Tabs Pattern: slides are `tabpanel`, dots are `tab` |
| Grouped | Not used |

The Tabbed pattern needs complex keyboard handling and `tabindex` management, described below. The author would prefer a `role="carousel"`, since a carousel is not a Tabs UI.

**(source)** The slide role (`tabpanel` or `group`) is fixed when slides are registered, which happens on mount and `refresh()`. If a breakpoint or an `options` prop change toggles `pagination` later, slide roles stay stale until `splide.refresh()`. **(tested)** After `pagination` went from `true` to `false` through the prop, slides kept `role="tabpanel"` until this effect ran, then had `role="group"`:

```tsx
useEffect( () => {
  ref.current?.splide?.refresh();
}, [ showPagination ] );
```

## Landmark, role and accessible name

W3C: "A carousel container element that encompasses all components of the carousel, including both carousel controls and slides, has either role region or role group."

- By default Splide makes the root a `region` landmark. Carousels usually hold content tied to the main page (banners, galleries, recommended articles, product lists), and a landmark lets assistive technology users jump to it.
- For a decorative carousel, or one unrelated to the main content, use `group`. In React, pass it as an option, not as a prop (a `role` prop is lost in development, see above):

```tsx
<Splide options={ { role: 'group', label: 'Team photos' } }>
  <SplideSlide>1</SplideSlide>
  <SplideSlide>2</SplideSlide>
</Splide>
```

Rules for the `role` option (Options docs plus `Elements.ts`):

- It must be a landmark role or `group`.
- `role: ''` or `role: undefined` stops Splide from setting any role.
- **(source)** Splide sets `role` only when the root has no `role` attribute, the root is not a `<section>`, and the option is truthy. A `role` attribute already on the root is never overwritten.
- `tag="section"`: Splide ignores the option, because a named `<section>` already has the implicit `region` role. A `<section>` gets that role only with an accessible name, so always give it `label` or `labelledby`. The React docs recommend `tag="section"`.
- `tag="header"` (`banner`), `tag="footer"` (`contentinfo`) or `tag="nav"` (`navigation`) have their own landmark roles, and adding `role="region"` to them can be wrong. The React docs say to "provide the most appropriate landmark `role` together" with those tags; do it with the `role` option.

Accessible name:

| Option | Attribute set on root | Notes |
|---|---|---|
| `label` | `aria-label` | Overwrites the attribute. Defaults to the root's `aria-label` at construction |
| `labelledby` | `aria-labelledby` | Overwrites the attribute. Defaults to the root's `aria-labelledby` at construction |

- When a visible heading works as the carousel's name, reference it with `labelledby` instead of `label`.
- W3C says the label should not include the word "carousel", because `aria-roledescription` is already "carousel". Some screen readers announce label plus description, and some ignore the description. The author's view is that including "carousel" is acceptable: hearing it twice is better than not hearing it.
- `label` and `labelledby` are responsive. **(source)** `Elements` reapplies both on every `updated` event, so breakpoints can change them, and so can a change to the `options` prop.
- The label may be wrong after destruction, especially if it contains "carousel" or "slider". Change it in the breakpoint that destroys the carousel:

```tsx
const options: Options = {
  label: 'My Carousel',
  breakpoints: {
    640: {
      destroy: true,
      label  : 'My Gallery', // Used after destruction
    },
  },
};
```

- **(source)** Options merge in this order: the root's `aria-label`/`aria-labelledby`, then `DEFAULTS`, then `Splide.defaults`, then the `options` prop, then the root's `data-splide` JSON. In React, `Splide.defaults` set on an `import { Splide } from '@splidejs/splide'` has no effect on `<Splide>`, because React Splide bundles its own copy of core (tested). Share an options object instead, and keep `label` out of it, so one carousel's name doesn't land on another.

**(tested)** Which way of naming survives:

| Code | Production | Development (StrictMode) |
|---|---|---|
| `<Splide aria-label="Team photos">` | Named | **Unnamed** |
| `<Splide aria-labelledby="heading">` | Named | **Unnamed** |
| `<Splide role="group">` | `role="group"` | **`role="region"`** |
| `<Splide options={ { label: 'Team photos' } }>` | Named | Named |
| `<Splide options={ { labelledby: 'heading' } }>` | Named | Named |
| `<Splide options={ { role: 'group' } }>` | `role="group"` | `role="group"` |
| `<SplideSlide aria-label="Article one">` | Kept | Kept |

## Keyboard interaction

Since v4, custom shortcuts such as arrow keys moving the carousel are **disabled by default**. Only the shortcuts defined by the Tabs Pattern are enabled, on the pagination.

### Pagination key bindings (`paginationKeyboard: true`, default)

While a pagination button has focus:

| Key | Action |
|---|---|
| Left arrow | Horizontal pagination: moves focus to the previous dot (the next dot in RTL) and shows its slide |
| Right arrow | Horizontal pagination: moves focus to the next dot (the previous dot in RTL) and shows its slide |
| Up arrow | Vertical pagination: moves focus to the previous dot and shows its slide |
| Down arrow | Vertical pagination: moves focus to the next dot and shows its slide |
| Space | Shows the slide if it is not active |
| Enter | Shows the slide if it is not active |
| Home | Moves focus to the first dot and shows the first slide |
| End | Moves focus to the last dot and shows the last slide |

- Moving focus to a dot activates its slide. The APG calls this "selection follows focus". With `waitForTransition: true`, the selection may not follow focus while a transition is running.
- **(source)** Arrow keys wrap around: Right on the last dot goes to the first, and Left on the first goes to the last. Handled keys call `preventDefault()` and `stopPropagation()`.
- **(source)** Space and Enter are not handled by Splide. They are the button's native click, which calls `go('>' + page)`.
- **(source)** Vertical (`ttb`) pagination ignores Left and Right. Horizontal pagination ignores Up and Down.

### Pagination direction

The pagination direction follows `direction` by default. With `direction: 'ttb'`, pagination is treated as vertical however it looks: it gets `aria-orientation="vertical"` and responds to Up and Down. To use horizontal pagination on a vertical carousel, or the reverse, set `paginationDirection`. It affects shortcuts, ARIA attributes and the modifier class (`splide__pagination--ltr|rtl|ttb`):

```tsx
<Splide options={ { label: 'Vertical', direction: 'ttb', height: '20rem', paginationDirection: 'ltr' } }>
  { slides }
</Splide>
```

### `keyboard` option (custom shortcuts, off by default)

| Value | Behavior |
|---|---|
| `false` | Disables shortcuts (default) |
| `true` | Shortcuts work only while the carousel contains focus. **(source)** Listens for `keydown` on the root |
| `'focused'` | Same as `true` |
| `'global'` | Listens for `keydown` on `window` (discouraged) |

| Key | Action |
|---|---|
| Left arrow | Previous page in LTR, next page in RTL |
| Right arrow | Next page in LTR, previous page in RTL |
| Up arrow | Previous page in TTB |
| Down arrow | Next page in TTB (the Options docs table says "previous", which is a typo) |

- v3 added `tabindex="0"` to the root for `'focused'`. v4 does not: `tabindex="0"` should be avoided on non-interactive content, and `region` and `group` are not interactive roles.
- With both `keyboard` and `paginationKeyboard` enabled, the pagination bindings win while pagination has focus. **(source)** They stop propagation, so the root listener never sees the key.
- **(source)** Pages move with `go('<')` and `go('>')`, which respect `perPage` and `perMove`.
- React: `<Splide>`'s prop type has no `onKeyDown` (React Splide removes DOM event props from it). For your own key handling, put `onKeyDown` on a wrapper element.

### Navigation carousels (`isNavigation: true`), from source

A focused navigation slide responds to **Space** and **Enter** by moving the synced carousels to that slide (`Sync.ts`, `TRIGGER_KEYS = [' ', 'Enter']`). Clicking it does the same. **(tested)** This works with the React sync pattern (`mainRef.current.sync( thumbsRef.current.splide )` in an effect, see `references/apis.md`), in production and under StrictMode.

## Roving tabindex

Splide manages focus inside the pagination with a roving tabindex, which the tabbed interface requires. Every pagination button except the active dot gets `tabindex="-1"`, which takes it out of the page Tab sequence. The benefits:

- Keyboard users can return to the previously selected dot after leaving the carousel.
- Keyboard users leave the pagination with a single Tab, instead of tabbing through every button.

**(source)** Splide removes the `tabindex` attribute from the active dot instead of setting `tabindex="0"`. The `<button>` is focusable natively.

## Focus management

- **`slideFocus`** (`boolean`): adds `tabindex="0"` to visible slides and `tabindex="-1"` to hidden ones. It defaults to `false`, or `true` when `isNavigation` is `true`, because navigation slides are clickable and clickable elements must be focusable for keyboard users. **(source)** The `Sync` component sets that default during setup.
- **(source)** If a slide has focus when it becomes hidden, focus moves to the active slide using `focus({ preventScroll: true })`.
- **(source)** Clicking an arrow or a pagination dot does **not** move focus into the slides. Focus stays on the control.
- **`focusableNodes`**: see [Focusable elements in hidden slides](#focusable-elements-in-hidden-slides-wcag-131).
- **`is-focus-in`** class on the root: see [Focus visible](#focus-visible-wcag-247). **(source)** Splide listens for `keydown`, `mousedown` and `touchstart` on `document` in the capture phase. On every `focusin` inside the root, it adds `is-focus-in` if the last input was a key and removes it if the last input was a pointer. It is never removed on `focusout`.
- React: Splide writes `tabindex` on slides (with `slideFocus`) and on the links and buttons inside them (through `focusableNodes`) on every status update. React writes a `tabIndex` prop only when its value changes, and then it overrides Splide's value until the next status update. Leave `tabIndex` off those elements, or keep it constant.

## Live region

Content in a slide that just became visible is hard to find with a screen reader. Sighted users see it, but users who rely on audio or Braille have to look for the updated slide. A live region solves this. For some screen readers, adding `aria-live="polite"` alone does **not** work, because typical carousels do not meet the `aria-relevant` conditions those readers use to detect a slide change. Splide works around this.

How it works **(source, `Live.ts`)**:

1. It is enabled when `live: true` (default) and `isNavigation` is not `true`. A navigation carousel never activates a live region, because it controls another carousel (tested: the thumbnails track has no `aria-live`).
2. On mount it sets `aria-atomic="true"` on the track. It sets `aria-live="off"` if autoplay is running, otherwise `aria-live="polite"`.
3. On every `moved` and `scrolled` event, it sets `aria-busy="true"` on the track and appends `<span class="splide__sr">…</span>` to the track. After 90 ms it removes the span and sets `aria-busy="false"`. JAWS needs `aria-atomic` for `aria-busy` to work, and Windows Narrator goes silent if the span is removed immediately.
4. It switches to `aria-live="off"` on `autoplay:play` and back to `polite` on `autoplay:pause` (tested). Without this, screen readers would keep speaking while the page is open, even with the carousel out of view. With `pauseOnFocus: true`, moving focus into the carousel pauses autoplay and turns the live region back on.
5. `.splide__sr` is visually hidden by `@splidejs/react-splide/css` and `@splidejs/react-splide/css/core` (and the theme files). Without any of them, add the rule yourself:

```css
.splide__sr {
  clip: rect(0 0 0 0);
  border: 0;
  height: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
  width: 1px;
}
```

- Disable it with `options={ { live: false } }`. `live` is read once, at mount, so changing it through the prop later does nothing. **(source)** At runtime, toggle it with `ref.current?.splide?.Components.Live.disable( true )` (or `false`). This does nothing when the live region is not enabled.
- The span is appended inside the track, next to the `<ul>` React renders. React does not touch it, because it reconciles only the children it rendered.
- The author tested it with Windows Narrator, JAWS, NVDA and VoiceOver. It cannot be guaranteed for every screen reader, browser and device.

## Autoplay play/pause (WCAG 2.2.1, 2.2.2)

> The user is allowed to turn off the time limit before encountering it. (Timing Adjustable)
>
> For any auto-updating information that starts automatically and is presented in parallel with other content, there is a mechanism for the user to pause, stop, or hide it. (Pause, Stop, Hide)

The W3C pattern requires an autoplaying carousel to:

1. Have a button that pauses and resumes autoplay.
2. Pause autoplay when focus moves into the carousel.
3. Pause autoplay while the mouse hovers over the carousel.

Users with disabilities, such as blindness, dexterity impairments or cognitive limitations, need enough time to read the content. If the carousel rotates on its own, give users a pause/resume button. Requirements 2 and 3 need no work: `pauseOnFocus` and `pauseOnHover` are `true` by default.

### The built-in toggle

Splide supports a play/pause toggle built on the APG Toggle Button pattern. Render it yourself inside `<Splide>`, with `hasTrack={ false }` and `<SplideTrack>` so you control where it goes. The full tested example is in [Accessible markup example](#accessible-markup-example):

```tsx
<Splide options={ { label: 'Promotions', autoplay: true } } hasTrack={ false }>
  <SplideTrack>{ slides }</SplideTrack>
  <div className="splide__progress"><div className="splide__progress__bar" /></div>
  <button className="splide__toggle" type="button">
    <span className="splide__toggle__play">Start</span>
    <span className="splide__toggle__pause">Pause</span>
  </button>
</Splide>
```

The toggle's attributes **(source, `Autoplay.ts`)**, set only when `autoplay` is truthy:

| Attribute / class | Value |
|---|---|
| `aria-controls` | ID of the track |
| `aria-label` | `i18n.play` ("Start autoplay") while autoplay is stopped, `i18n.pause` ("Pause autoplay") while it runs |
| `is-active` class | Present while running. The core CSS uses it to swap `.splide__toggle__play` and `.splide__toggle__pause` |

"Stopped" means paused by the toggle or by `autoplay: 'pause'`. A temporary hover or focus pause does not change the label. Splide does not set `aria-pressed`.

React notes:

- Write `type="button"`. The official React example omits it, which makes the button submit an enclosing form.
- Splide finds the toggle with a class query when the instance mounts. It must be inside the root when `<Splide>` mounts, and it must not be swapped for a new element later. Rendering it conditionally after mount leaves it unwired.
- Keep its `className` static (tested: a changed `className` drops `is-active`).
- **(source)** Autoplay only starts when there are more slides than `perPage`. Otherwise nothing plays, yet the toggle shows the "Pause autoplay" state, and after one click it is stuck on "Start autoplay".

### A toggle outside the carousel

**(tested)** A button outside the root is not wired by Splide. Drive `Components.Autoplay` yourself and keep the stopped state in React. `pause()` stops until `play()` is called, so hover and focus do not restart it (tested):

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// A play/pause button outside the carousel root, so Splide's .splide__toggle handling doesn't apply.
export default function HeroWithExternalToggle() {
  const ref = useRef<Splide>( null );
  const [ stopped, setStopped ] = useState( false );

  const toggle = () => {
    const Autoplay = ref.current?.splide?.Components.Autoplay;
    if ( ! Autoplay ) return;

    if ( stopped ) {
      Autoplay.play();
    } else {
      Autoplay.pause(); // pause( stop = true ): stays paused until play()
    }

    setStopped( ! stopped );
  };

  return (
    <>
      <Splide id="hero" ref={ ref } options={ { label: 'Announcements', type: 'loop', autoplay: true } }>
        <SplideSlide>Sale</SplideSlide>
        <SplideSlide>New arrivals</SplideSlide>
        <SplideSlide>Free shipping</SplideSlide>
      </Splide>

      <button type="button" id="hero-toggle" aria-controls="hero-track" onClick={ toggle }>
        { stopped ? 'Start autoplay' : 'Pause autoplay' }
      </button>
    </>
  );
}
```

- `hero-track` is the track ID Splide generates from the root `id` (see [Generated IDs](#generated-ids)).
- The live region still follows autoplay: `aria-live` turned `polite` on pause and `off` on play (tested).
- Don't derive the label from `onAutoplayPause`/`onAutoplayPlay`: those also fire for the temporary hover and focus pauses, so the label would flicker.

## Reducing motion (WCAG 2.3.3)

> Motion animation triggered by interaction can be disabled, unless the animation is essential to the functionality or the information being conveyed. (Animation from Interactions)

Animations and transitions can cause headaches and nausea, for example in people with vestibular motion disorders. When Splide detects `prefers-reduced-motion: reduce`:

- `speed` and `rewindSpeed` become `0`.
- `autoplay` becomes `'pause'`, so autoplay does not start after page load.

The carousel then shows the active slide immediately, with no transition. Drag and swipe are the exception: they still animate, because that motion is arguably essential for users to understand what is happening.

To test it, open Chrome DevTools, go to the Rendering tab, and set "Emulate CSS media feature prefers-reduced-motion".

**Do not rely on autoplay alone to change slides** unless the carousel is purely decorative. With reduced motion enabled, users would have no way to rotate it.

From source (`Media.ts`, `Drag.ts`):

- `reducedMotion` is registered as a media query after all `breakpoints`, so it overrides breakpoint values for the same keys. It reacts to the OS setting changing while the page is open.
- `reducedMotion` is deep-merged with the default, so `reducedMotion: { autoplay: false }` still sets `speed: 0` and `rewindSpeed: 0`. `reducedMotion: false` disables the feature entirely, which is not recommended.
- On drag release, the reduced values are removed for that one move and then restored, which is how swipes stay animated.
- React: the media query is read by the core in the browser, so it works the same with server rendering. `reducedMotion` is read when the instance is created; change it with a `key` remount, not a prop change (see `references/reactivity.md`).

**(tested)** With reduced motion emulated, this carousel did not start autoplay (the toggle said "Start autoplay"), and a click on the next arrow moved with no transition:

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const options: Options = {
  label        : 'Highlights',
  type         : 'loop',
  autoplay     : true,
  reducedMotion: {
    speed      : 0,
    rewindSpeed: 0,
    autoplay   : 'pause',
  }, // this is the default; shown for clarity
};

export default function Highlights() {
  return (
    <Splide id="highlights" options={ options } hasTrack={ false }>
      <SplideTrack>
        <SplideSlide>1</SplideSlide>
        <SplideSlide>2</SplideSlide>
        <SplideSlide>3</SplideSlide>
      </SplideTrack>
      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Start</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

Your own CSS animations and transitions inside slides are not covered. Use a `@media (prefers-reduced-motion: reduce)` rule for them.

## Focus visible (WCAG 2.4.7)

> Any keyboard operable user interface has a mode of operation where the keyboard focus indicator is visible. (Focus Visible)

Focus styles should appear for keyboard users only, which is what `:focus-visible` does. When v4 came out, IE did not support it and Safari had only added it in its latest release (2022-03-15). Splide therefore patches the gap with a class, and the v4 themes (`@splidejs/react-splide/css` and `@splidejs/react-splide/css/<theme>`) include focus styles for arrows, pagination, slides (thumbnails) and the toggle. `@splidejs/react-splide/css/core` has none.

Without a theme, use the `is-focus-in` class, which Splide adds to the root when the carousel receives keyboard focus:

```css
/* For browsers that support :focus-visible */
.splide__arrow:focus-visible {
  outline: 3px solid skyblue;
}

/* For other browsers */
.splide.is-focus-in .splide__arrow:focus {
  outline: 3px solid skyblue;
}
```

(The docs example writes `.splide_arrow` with a single underscore in the second rule. The correct class is `.splide__arrow`, as shown here.)

`is-focus-in` is not removed as soon as the carousel loses focus, so always pair it with `:focus` as above.

The default theme's rules, for reference (the React Splide CSS files are identical to core 4.1.4's):

```css
.splide__arrow:focus-visible,
.splide__pagination__page:focus-visible,
.splide__toggle:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__arrow:focus,
.splide.is-focus-in .splide__pagination__page:focus,
.splide.is-focus-in .splide__toggle:focus { outline: 3px solid #0bf; outline-offset: 3px; }
.splide__slide:focus { outline: 0; }
@supports (outline-offset: -3px) {
  .splide__slide:focus-visible,
  .splide.is-focus-in .splide__slide:focus { outline: 3px solid #0bf; outline-offset: -3px; }
}
```

## Focusable elements in hidden slides (WCAG 1.3.1)

> Information, structure, and relationships conveyed through presentation can be programmatically determined or are available in text. (Info and Relationships)

Slides that are not completely visible in the viewport get `aria-hidden="true"`, which hides them from screen readers and other assistive technology. Focusable elements inside them, such as `<input>` or `<textarea>`, would still be reachable with Tab, but screen readers would not say what they are. To prevent that, Splide sets `tabindex="-1"` on these elements inside hidden slides:

- `<a>`
- `<button>`
- `<textarea>`
- `<input>`
- `<select>`
- `<iframe>`

This fails when a slide contains other focusable elements:

- `<area>`
- `<audio controls>`
- `<summary>`
- `<video controls>`
- an element with the `contenteditable` attribute
- an element with `tabindex="0"`

The `focusableNodes` option is a selector string, so you can cover all of these except `tabindex="0"`:

```tsx
const options: Options = {
  label         : 'Media',
  focusableNodes: 'a, button, textarea, input, select, iframe, area, audio[controls], video[controls], summary, [contenteditable]',
};
```

`focusableNodes` cannot handle `tabindex="0"`, because that value has to be restored when the slide becomes visible again. **(source)** Splide removes the `tabindex` attribute from matched elements when their slide becomes visible, so a `[tabindex]` selector would leave those elements unfocusable. Toggle such elements yourself with the `onVisible` and `onHidden` props. Slides that start out of view never fire `hidden`, so also set the initial state in `onReady` (tested: without it, the two initially hidden cards kept `tabindex="0"`):

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideEventHandlers } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// Elements with tabindex="0" inside slides: focusableNodes can't restore them, so toggle them yourself.
const setFocusable = ( slide: HTMLElement, focusable: boolean ) => {
  slide.querySelectorAll( '[data-focusable]' ).forEach( el => {
    el.setAttribute( 'tabindex', focusable ? '0' : '-1' );
  } );
};

const onVisible: SplideEventHandlers['onVisible'] = ( splide, Slide ) => setFocusable( Slide.slide, true );
const onHidden: SplideEventHandlers['onHidden'] = ( splide, Slide ) => setFocusable( Slide.slide, false );

// Slides that start out of view never fire `hidden`, so set the initial state once the carousel is ready.
const onReady: SplideEventHandlers['onReady'] = splide => {
  splide.Components.Slides.forEach( Slide => {
    setFocusable( Slide.slide, Slide.slide.classList.contains( 'is-visible' ) );
  } );
};

export default function FocusableCards() {
  return (
    <Splide
      options={ { label: 'Cards', perPage: 2 } }
      onVisible={ onVisible }
      onHidden={ onHidden }
      onReady={ onReady }
    >
      { [ 1, 2, 3, 4 ].map( n => (
        <SplideSlide key={ n }>
          <div data-focusable tabIndex={ 0 }>Card { n }</div>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

- The handlers are module-level functions, so they can't go stale (see `references/events.md`).
- React renders `tabIndex={ 0 }` once and does not write it again unless the prop value changes, so it doesn't undo the handlers. If the prop value does change on a later render, React overwrites the handler's value.

Visibility rules **(source, `Slide.ts`)**:

- "Visible" means the slide lies completely inside the track's rectangle, rounded to whole pixels. For `type: 'fade'`, the active slide is the only visible one.
- A real slide gets `aria-hidden="true"` when it is not visible **and** not active. The active slide is never hidden, even if only partly in view. A clone gets it whenever it is not visible.
- `aria-hidden` is not changed while the carousel is moving or scrolling, which keeps Windows Narrator from reading early. Slide status updates on `moved`, `scrolled` and the internal shift event, and on repositioning (`mounted`, `resized`, `updated`, `refresh`). With `updateOnMove: true` it also updates on `move`. Classes, `tabindex` and the `active`/`visible` events change then, but `aria-hidden` waits for `moved`.

## Role and ARIA attribute reference (WCAG 4.1.2)

> For all user interface components, the name and role can be programmatically determined; states, properties, and values that can be set by the user can be programmatically set; and notification of changes to these items is available to user agents, including assistive technologies. (Name, Role, Value)

Splide sets the attributes. You add an accessible name and translate the texts (see `references/i18n.md`). React renders none of these attributes: they appear after mount, in the browser.

### Basic carousel (no pagination)

**Root (`.splide`)**

| Attribute | Value |
|---|---|
| `role` | `region` (the `role` option) if the element is not `<section>`. **(source)** Not set if the root already has a `role` or the option is empty |
| `aria-roledescription` | `carousel` (`i18n.carousel`) |
| `aria-label` | **(source)** The `label` option, if non-empty |
| `aria-labelledby` | **(source)** The `labelledby` option, if non-empty |
| `id` | **(source)** Kept if present (the `id` prop), otherwise `splide01`, `splide02`, and so on |

**Track (`.splide__track`)**

| Attribute | Value |
|---|---|
| `aria-live` | `polite` if the live region is enabled. Changes to `off` when autoplay starts and back to `polite` when it pauses |
| `aria-atomic` | `true` if the live region is enabled (^4.0.3) |
| `aria-busy` | If the live region is enabled: `true` when a transition ends, `false` after a delay (^4.0.3). **(source)** The delay is 90 ms |
| `id` | **(source)** Kept if present, otherwise `{rootId}-track` |

**List (`.splide__list`)**, from source

| Attribute | Value |
|---|---|
| `role` | `presentation` |
| `aria-orientation` | `vertical` only when `isNavigation: true` and `direction: 'ttb'` |
| `id` | Kept if present, otherwise `{rootId}-list` |

**Slides (`.splide__slide`)**

| Attribute | Value |
|---|---|
| `role` | `group` |
| `aria-roledescription` | `slide` (`i18n.slide`) |
| `aria-label` | `%s of %s`, meaning "{slide number} of {slide length}" (`i18n.slideLabel`). **(source)** A slide's own `aria-label` (an `aria-label` prop on `<SplideSlide>`) is kept and restored on destroy |
| `aria-hidden` | `true` for slides outside the viewport |
| `tabindex` | **(source)** Only with `slideFocus`: `0` when visible, `-1` when hidden |
| `id` | **(source)** Always overwritten with `{rootId}-slide01`, `-slide02`, and so on |

**Slides when `isNavigation: true`**

| Attribute | Value |
|---|---|
| `role` | `button` if `slideFocus` is enabled, otherwise `group`. **(source)** Without `slideFocus` the `role` attribute is removed (see [Docs vs source](#docs-vs-source)) |
| `aria-roledescription` | Not set, because the slide is interactive. **(source)** Removed only when `slideFocus` is on; otherwise `slide` stays |
| `aria-hidden` | `true` for slides outside the viewport |
| `aria-controls` | The ID or IDs of the slides it controls in the synced carousel(s), space-separated |
| `aria-label` | "Go to slide %s" (`i18n.slideX`). **(source)** Overwrites any `aria-label` on the slide |
| `aria-current` | **(source)** `true` on the active slide, otherwise absent |
| `tabindex` | **(source)** `0` when visible, `-1` when hidden (`slideFocus` defaults to `true` here) |

**(tested)** With the React sync pattern, thumbnail slides got `role="button"`, `aria-label="Go to slide 1"`, `aria-current="true"` on the active one, `tabindex="0"`, and `aria-controls="main-slide01"` (one ID, not duplicated by the StrictMode remount).

**Arrows (`.splide__arrow--prev` / `.splide__arrow--next`)**

| Attribute | Value |
|---|---|
| `aria-controls` | The ID of the track element |
| `aria-label` | Previous arrow: "Previous slide" (`i18n.prev`) or "Go to last slide" (`i18n.last`). Next arrow: "Next slide" (`i18n.next`) or "Go to first slide" (`i18n.first`). **(source)** The "first"/"last" texts are used when the arrow would wrap around (`loop` or `rewind`). Splide overwrites the `aria-label` on arrows you render too |
| `disabled` | Set when there is no slide to go to |

**(source)** Generated arrow markup: `<div class="splide__arrows splide__arrows--{direction}">` holds `<button class="splide__arrow splide__arrow--prev" type="button">` with `<svg ... focusable="false"><path d="..."/></svg>`, inserted before the track. React does not know about these nodes, which is fine: they live outside the elements React renders. Custom arrows rendered in JSX (`<div className="splide__arrows">` with two `<button type="button">`) get the same attributes.

**Play/pause toggle button (`.splide__toggle`)**

| Attribute | Value |
|---|---|
| `aria-controls` | The ID of the track element |
| `aria-label` | "Start autoplay" while autoplay is paused, "Pause autoplay" while it plays |

### Tabbed carousel (with pagination)

Same as Basic, except for the following elements.

**Slides**

| Attribute | Value |
|---|---|
| `role` | `tabpanel` |
| `aria-roledescription` | `slide` |
| `aria-label` | `%s of %s`, meaning "{slide number} of {slide length}" |
| `aria-hidden` | `true` for slides outside the viewport |

**Pagination (`ul.splide__pagination`)**

| Attribute | Value |
|---|---|
| `role` | `tablist` |
| `aria-label` | "Select a slide to show" (`i18n.select`). The accessibility page prints "Select slide to show" |
| `aria-orientation` | **(source)** `vertical` when the pagination direction is `ttb`, otherwise absent |

**(source)** Splide appends the generated `<ul>` as the last child of the track's parent (normally the root), so it comes after a toggle button rendered before it (tested). A `<ul className="splide__pagination" />` you render yourself is reused instead, which lets you control its position in the DOM and in the Tab order. Render it empty: Splide fills it with `<li>` elements React doesn't own.

**Pagination list items (`li`)**

| Attribute | Value |
|---|---|
| `role` | `presentation` |

**Pagination buttons (`button.splide__pagination__page`, `type="button"`)**

| Attribute | Value |
|---|---|
| `role` | `tab` |
| `aria-controls` | The ID or IDs of the slides on that page |
| `aria-label` | "Go to slide %s" or "Go to page %s". **(source)** It uses `pageX` when `perPage > 1` and neither `focus` nor `isNavigation` is set, otherwise `slideX` |
| `aria-selected` | `true` for the active dot. Docs say `false` for the others; **(source)** the attribute is removed instead |
| `tabindex` | **(source)** `-1` on inactive dots, removed on the active one |

**Progress bar (`.splide__progress__bar`)**, from source: Splide only sets its inline `width`. It adds no ARIA attributes.

## Generated IDs

From source (`Elements.ts`, `Slide.ts`, `Clones.ts`):

| Element | ID |
|---|---|
| Root | The `id` prop, or `splide` + a two-digit counter: `splide01`, `splide02`, and so on |
| Track | Existing `id`, or `{rootId}-track` |
| List | Existing `id`, or `{rootId}-list` |
| Slide n | `{rootId}-slide01`, `-slide02`, and so on. **Overwrites any existing slide `id`** (an `id` prop on `<SplideSlide>`) and is not restored on destroy |
| Clone n | `{rootId}-clone01`, and so on |

- Give `<Splide>` a stable `id` prop if anything references slide or track IDs, such as `aria-controls` on your own controls or deep links. Without it, the counter depends on mount order, which differs between pages.
- **(source)** Destroy does not remove the generated root, track and list IDs, so the StrictMode remount keeps them (tested: `hero-track` and `main-slide01` were the same in development).
- Don't put an `id` on `<SplideSlide>`: Splide replaces it, and React won't put it back unless the prop value changes.

## Accessible markup example

**(tested)** Production and development: no `role` on the `<section>`, `aria-labelledby="featured-heading"` kept, the slides' own labels kept, the toggle labelled "Pause autoplay" with `aria-controls="featured-track"`, the progress bar moving, the pagination after the toggle, and `aria-live` switching from `off` to `polite` when the toggle was clicked.

```tsx
import { Splide, SplideSlide, SplideTrack } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const products = [
  { name: 'Linen shirt', href: '/products/linen-shirt', src: '/images/shirt.jpg', alt: 'Linen shirt in sand color' },
  { name: 'Wool scarf', href: '/products/wool-scarf', src: '/images/scarf.jpg', alt: 'Grey wool scarf' },
  { name: 'Canvas tote', href: '/products/canvas-tote', src: '/images/tote.jpg', alt: 'Natural canvas tote bag' },
];

const options: Options = {
  labelledby : 'featured-heading', // survives StrictMode, unlike an aria-labelledby prop
  perPage    : 2,                  // lower than the slide count, or autoplay never starts
  autoplay   : true,
  breakpoints: { 640: { perPage: 1 } },
};

export default function FeaturedProducts() {
  return (
    <Splide tag="section" id="featured" options={ options } hasTrack={ false }>
      <h2 id="featured-heading">Featured products</h2>

      <SplideTrack>
        { products.map( product => (
          <SplideSlide key={ product.href } aria-label={ product.name }>
            <a href={ product.href }><img src={ product.src } alt={ product.alt } /></a>
          </SplideSlide>
        ) ) }
      </SplideTrack>

      {/* Autoplay controls: required when autoplay is on (WCAG 2.2.2) */}
      <div className="splide__progress"><div className="splide__progress__bar" /></div>
      <button className="splide__toggle" type="button">
        <span className="splide__toggle__play">Start</span>
        <span className="splide__toggle__pause">Pause</span>
      </button>
    </Splide>
  );
}
```

This works without extra code because:

- `tag="section"` with `labelledby` is a named `region`, so Splide adds no `role`.
- Each slide has a content-based `aria-label`, which Splide keeps instead of "1 of 3".
- The visible toggle texts "Start" and "Pause" appear inside Splide's labels "Start autoplay" and "Pause autoplay", which satisfies WCAG 2.5.3 Label in Name. If you rename the visible texts, change `i18n.play` and `i18n.pause` to match.
- `perPage` is lower than the slide count, so autoplay can run.

## What destroy does to attributes

React Splide destroys the instance in three situations:

- Unmount: `componentWillUnmount` calls `splide.destroy()`, a complete destroy.
- StrictMode in development: mount, complete destroy, mount again. This is why the `aria-label`, `aria-labelledby` and `role` props are lost.
- A breakpoint with `destroy: true` (`destroy( false )`) or `destroy: 'completely'` (`destroy( true )`). The component stays mounted.

From source (`Elements.ts`, `Slide.ts`, `Arrows.ts`, `Pagination.ts`, `Live.ts`, `Autoplay.ts`):

| Element | `destroy(false)` or breakpoint `destroy: true` | `destroy()` / `destroy(true)` / breakpoint `destroy: 'completely'` / unmount |
|---|---|---|
| Root | Removes `style` and `aria-roledescription`. **Keeps** `role`, `aria-label` and `aria-labelledby` | Removes `role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription` and `style`, **including values passed as props** |
| Track, list | Remove those same attributes plus `style`. The track also loses `aria-live`, `aria-atomic` and `aria-busy` | Same |
| Slides | Lose status classes and all of the attributes above. Original `style` and `aria-label` restored | Same |
| Arrows | Generated arrows are removed. Arrows you render lose all of the attributes above, including your own `aria-label` | Same |
| Pagination | Generated `<ul>` is removed. With a placeholder you render, only its children are removed | Same |
| Toggle button | Keeps `aria-controls`, `aria-label` and `is-active`: Autoplay's `destroy()` only cancels the timer (tested in vanilla) | Same |

**(source) Destroy leaves `tabindex="-1"` on focusable elements inside slides.** `Slide.destroy()` cleans the slide element but not its descendants. Links and buttons in slides that were hidden at destroy time stay out of the Tab order, which is a real problem when a breakpoint destroys the carousel to show a list. Clear them on `destroy`, in an extension so the handler is registered again on every breakpoint remount (an event prop is not, see `references/events.md`).

**(tested)** At 1280px the hidden slides' links had `tabindex="-1"`; at 500px (destroyed) none had it; back at 1280px they had it again; at 500px again they were cleared again:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type Extension = NonNullable<SplideProps['extensions']>[ string ];

// Destroy leaves tabindex="-1" on links in slides that were hidden. Clear it on every destroy.
// An extension runs again on each breakpoint remount, so the handler is registered again.
const RestoreFocusables: Extension = splide => ( {
  mount() {
    splide.on( 'destroy', () => {
      // Runs after all components are destroyed (clones already removed).
      const { focusableNodes } = splide.options;
      if ( ! focusableNodes ) return;

      splide.root.querySelectorAll( '.splide__slide' ).forEach( slide => {
        slide.querySelectorAll( focusableNodes ).forEach( el => {
          el.removeAttribute( 'tabindex' ); // also removes any tabindex you set yourself
        } );
      } );
    } );
  },
} );

// Defined outside the component: <Splide> reads `extensions` once, at mount.
const extensions = { RestoreFocusables };

const options: Options = {
  label      : 'Latest posts',
  perPage    : 2,
  breakpoints: { 640: { destroy: true } }, // a plain list below 640px
};

const posts = [ 'One', 'Two', 'Three', 'Four' ];

export default function Posts() {
  return (
    <Splide id="posts" options={ options } extensions={ extensions }>
      { posts.map( title => (
        <SplideSlide key={ title }>
          <a href={ `/posts/${ title.toLowerCase() }` }>{ title }</a>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

`Extension` is derived from the prop types, so you don't need `@splidejs/splide` for it. React Splide puts the `extensions` prop on the root as an attribute (`extensions="[object Object]"`), which is harmless.

## Checklist

From the docs, in React terms:

1. Give every carousel an accessible name: `options={ { labelledby: 'heading-id' } }` pointing at a visible heading, or `options={ { label: '...' } }`. Not `aria-label`/`aria-labelledby` props, which StrictMode drops in development.
2. Translate all texts with `i18n` for your users (`references/i18n.md`). Prefer content-based wording ("Next article" over "Next slide").
3. Keep the default `region` landmark for content carousels. Use `options={ { role: 'group' } }` for decorative or unrelated ones. Do not add `region` to elements with their own landmark roles (`tag="header"`, `tag="nav"`, and so on).
4. A `tag="section"` root needs `label` or `labelledby`.
5. With autoplay, render the play/pause toggle (`<button className="splide__toggle" type="button">`) and keep `pauseOnHover` and `pauseOnFocus` at `true`. Keep `perPage` below the slide count.
6. Do not rely on autoplay alone to move slides. Keep arrows, pagination or drag available, because reduced motion pauses autoplay.
7. Without a theme, add focus styles with `:focus-visible`, plus `.splide.is-focus-in ...:focus` as a fallback.
8. If slides contain `<area>`, `<audio controls>`, `<video controls>`, `<summary>` or `[contenteditable]`, extend `focusableNodes`.
9. Handle elements with `tabindex="0"` inside slides yourself, with `onVisible` and `onHidden` plus an initial pass in `onReady`.
10. With `isNavigation: true`, set `pagination: false`. Otherwise roles and ARIA attributes get mixed up.
11. If breakpoints destroy the carousel, change `label` in that breakpoint so it does not say "carousel" or "slider", and clear leftover `tabindex="-1"` with an extension.
12. Keep `paginationKeyboard: true`, which the W3C pattern requires. Turn on `keyboard` only if you want the extra arrow-key shortcuts.
13. Keep `className` stable on `<Splide>`, `<SplideSlide>` and the toggle, and don't set `id` on `<SplideSlide>`.
14. Check the accessibility tree in development as well as production: StrictMode is where prop-based names disappear.

## Not handled automatically

From source unless noted:

- **No default accessible name.** Splide sets `aria-label` only when you provide one. An unnamed `region` is not exposed as a landmark.
- **The toggle button is not generated.** Render `<button className="splide__toggle" type="button">` yourself. Splide sets only `aria-controls`, `aria-label` and `is-active`, not `aria-pressed`. Make sure the visible text is part of the accessible name (WCAG 2.5.3).
- **The progress bar** gets no ARIA. Do not add `role="progressbar"` unless you also maintain `aria-valuenow`.
- **Image `alt` text, headings and slide content semantics** are up to you. For lazy-loaded images, set `alt` in the JSX.
- **Custom arrow labels:** Splide overwrites `aria-label` on arrows from `i18n`, so set texts through `i18n`, not JSX.
- **Navigation slide labels:** Splide overwrites them with `i18n.slideX` ("Go to slide %s").
- **Focusable elements outside `focusableNodes`** and elements with `tabindex="0"`, as described above.
- **Focus does not move into the slides** after an arrow or dot is activated.
- **Arrow-key shortcuts** outside the pagination are off unless you set `keyboard`.
- **Duplicated IDs in loop clones:** clones are deep copies (`cloneNode(true)`). The clone itself gets a new ID, but descendant IDs, and any `aria-labelledby`/`aria-describedby` that point at them, are duplicated. Avoid IDs inside slides when `type: 'loop'`. Clones are plain DOM copies: **(tested)** a React `onClick` on a button inside a slide fired for the real slide and not for its clone. State, refs and handlers inside slides don't reach the clones.
- **Clones copy the original's labels.** They inherit `role`, `aria-roledescription` and `aria-label`, and rely on `aria-hidden="true"` whenever they are not fully visible.
- **Reduced motion** covers only `speed`, `rewindSpeed` and `autoplay`. Your own CSS animations and transitions inside slides are not affected, and drag stays animated.
- **The live region** is off while autoplay runs, never runs on navigation carousels, and is not guaranteed for every screen reader.
- **Root attributes on full destroy:** unmount and `destroy()` remove `role`, `aria-label` and `aria-labelledby` from the root, even if they came from props. In development, StrictMode's destroy does the same before the second mount.
- **`tabindex="-1"` left after destroy:** focusable descendants of slides that were hidden at destroy time keep it. See [What destroy does to attributes](#what-destroy-does-to-attributes) for a fix.
- **Stale slide role** after toggling `pagination` at runtime. Call `refresh()`.
- **Server-rendered HTML** has no ARIA attributes and stays invisible until React mounts the carousel.

## Docs vs source

| Topic | Docs say | Source (4.1.3 / 4.1.4) and React Splide 0.7.12 do |
|---|---|---|
| Naming the carousel in React | The React page uses `<Splide aria-label="...">` and `aria-labelledby` props | Works in production. In development, StrictMode's destroy strips them and they don't come back (tested). Use `options.label` / `options.labelledby` |
| `tag="header"`, `"footer"`, `"nav"` | "Provide the most appropriate landmark `role` together" | A `role` prop is stripped in development the same way (tested). Pass it as `options.role` |
| Navigation slide role without `slideFocus` | `group` | `setAttribute(slide, 'role', '')`, which **removes** `role` |
| Navigation slide `aria-roledescription` | "Not assigned because the slide is interactive" | Removed only when `slideFocus` is true. Otherwise `slide` remains |
| Inactive pagination dot `aria-selected` | `false` | Attribute **removed** (absent) |
| Pagination `aria-label` default | "Select slide to show" (accessibility page) | `'Select a slide to show'` (`constants/i18n.ts`; the i18n page agrees) |
| Focus CSS example | `.splide.is-focus-in .splide_arrow:focus` | The class is `splide__arrow` (two underscores) |
| `keyboard` key table (Options page) | Down arrow: "Go to the previous page in TTB" | Down arrow goes to the **next** page (`go('>')`) |
| Toggle button in the official React example | `<button className="splide__toggle">` | Without `type="button"` it submits an enclosing form. Add it |
| List `role="presentation"`, slide `aria-current`, list `aria-orientation` (nav + ttb), pagination `aria-orientation`, generated IDs | Not documented | Set as described above |

## Open concerns from the docs

The author asked accessibility experts for feedback on two points:

- The APG Tabs pattern says Tab from the tab list moves to "the tabpanel unless the first element containing meaningful content inside the tabpanel is focusable", and its examples make each tabpanel focusable with `tabindex="0"`. MDN says "Focusable elements should have interactive semantics", and `tabpanel` is not interactive. Firefox DevTools flags the W3C example as failing WCAG. Splide therefore does not make slides focusable by default.
- Should slides in a Tabbed carousel be focusable? If they should, Basic carousel slides should be too for consistency, because both patterns share `aria-roledescription="carousel"` and users expect the same behavior.
