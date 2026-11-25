# Accessibility

Source: https://splidejs.com/guides/accessibility/

Splide v4.1.4 from jsDelivr, no build step. The docs text was checked against the v4.1.3 source (`src/js/components/*`) and the 4.1.4 dist build. The two match for everything on this page. Facts marked **(source)** come from the code and do not appear on the docs page.

## Contents

- [Overview](#overview)
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
- [What destroy() does to attributes](#what-destroy-does-to-attributes)
- [Checklist](#checklist)
- [Not handled automatically](#not-handled-automatically)
- [Docs vs source](#docs-vs-source)
- [Open concerns from the docs](#open-concerns-from-the-docs)

## Overview

Splide v4 follows the W3C (WAI-ARIA APG) Carousel Design Pattern. It also adds a live region so screen readers read the new content when slides change. The docs point out that the W3C pattern contradicts itself and does not cover every carousel type: its examples are "fade" carousels, and applying them unchanged to a "sliding" carousel breaks it. Splide's behavior below is the author's adaptation.

Related options and their defaults (from `constants/defaults.ts` and the Options docs):

| Option | Type | Default | Role in accessibility |
|---|---|---|---|
| `role` | `string` | `'region'` | `role` on the root. Ignored for `<section>` |
| `label` | `string` | root's HTML `aria-label` | `aria-label` on the root. Responsive option |
| `labelledby` | `string` | root's HTML `aria-labelledby` | `aria-labelledby` on the root. Responsive option |
| `keyboard` | `boolean \| 'global' \| 'focused'` | `false` (not set) | Custom arrow-key shortcuts |
| `paginationKeyboard` | `boolean` | `true` | Tabs-pattern keys on pagination |
| `paginationDirection` | `'ltr' \| 'rtl' \| 'ttb'` | same as `direction` | Pagination key direction and `aria-orientation` |
| `slideFocus` | `boolean` | `false`, or `true` when `isNavigation` | `tabindex="0"` on visible slides |
| `focusableNodes` | `string` (selector) | `'a, button, textarea, input, select, iframe'` | Elements that get `tabindex="-1"` inside hidden slides |
| `live` | `boolean` | `true` | Live region on the track. Never active with `isNavigation` |
| `pauseOnHover` | `boolean` | `true` | Pauses autoplay on hover |
| `pauseOnFocus` | `boolean` | `true` | Pauses autoplay while focus is inside |
| `reducedMotion` | `Options` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Applied under `(prefers-reduced-motion: reduce)` |
| `i18n` | `Record<string, string>` | see [i18n.md](i18n.md) | All ARIA label texts |
| `waitForTransition` | `boolean` | `false` | When `true`, focus can move in pagination without the slide following during a transition |

## Basic vs Tabbed pattern

The W3C pattern has three sub-patterns. Every Splide carousel uses one of them, chosen by whether pagination exists:

| Pattern | Applied to |
|---|---|
| Basic | Carousels without pagination (`pagination: false`) |
| Tabbed | Carousels with pagination. The pagination follows the Tabs Pattern: slides are `tabpanel`, dots are `tab` |
| Grouped | Not used |

The Tabbed pattern needs complex keyboard handling and `tabindex` management, described below. The author would prefer a `role="carousel"`, since a carousel is not a Tabs UI.

**(source)** The slide role (`tabpanel` or `group`) is fixed when slides are registered, which happens on `mount()` and `refresh()`. If a breakpoint or `splide.options = {...}` toggles `pagination` later, slide roles stay stale until `splide.refresh()`.

## Landmark, role and accessible name

W3C: "A carousel container element that encompasses all components of the carousel, including both carousel controls and slides, has either role region or role group."

- By default Splide makes the root a `region` landmark. Carousels usually hold content tied to the main page (banners, galleries, recommended articles, product lists), and a landmark lets assistive technology users jump to it.
- For a decorative carousel, or one unrelated to the main content, use `group`:

```html
<div class="splide" role="group" aria-label="...">
</div>
```

```js
new Splide( '.splide', { role: 'group' } );
```

Rules for the `role` option (Options docs plus `Elements.ts`):

- It must be a landmark role or `group`.
- `role: ''` or `role: undefined` stops Splide from setting any role.
- **(source)** Splide sets `role` only when the root has no `role` attribute in HTML, the root is not a `<section>`, and the option is truthy. A `role` written in HTML is never overwritten.
- `<section>`: Splide ignores the option, because a named `<section>` already has the implicit `region` role. A `<section>` gets that role only with an accessible name, so always give it `aria-label` or `aria-labelledby`.
- On elements that have or may have a landmark role, such as `<header>` (`banner`) or `<nav>`, adding `role="region"` can be wrong.

Accessible name:

| Option | Attribute set on root | Notes |
|---|---|---|
| `label` | `aria-label` | Overwrites the HTML attribute. Defaults to the root's HTML `aria-label` |
| `labelledby` | `aria-labelledby` | Overwrites the HTML attribute. Defaults to the root's HTML `aria-labelledby` |

- When a visible heading works as the carousel's name, reference it with `aria-labelledby` (or `labelledby`) instead of `aria-label`.
- W3C says the label should not include the word "carousel", because `aria-roledescription` is already "carousel". Some screen readers announce label plus description, and some ignore the description. The author's view is that including "carousel" is acceptable: hearing it twice is better than not hearing it.
- `label` and `labelledby` are responsive. **(source)** `Elements` reapplies both on every `updated` event, so breakpoints can change them.
- The label may be wrong after destruction, especially if it contains "carousel" or "slider". Change it in the breakpoint that destroys the carousel:

```js
new Splide( '.splide', {
  label: 'My Carousel', // Can be omitted if the label is defined in HTML
  breakpoints: {
    640: {
      destroy: true,
      label  : 'My Gallery', // Used after destruction
    }
  }
} ).mount();
```

- **(source)** Options merge in this order: HTML `aria-label`/`aria-labelledby`, then `DEFAULTS`, then `Splide.defaults`, then constructor options, then the `data-splide` JSON. A `label` in `Splide.defaults` therefore overrides every root's HTML `aria-label`, so do not put `label` in `Splide.defaults`.

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

```js
new Splide( '.splide', {
  direction          : 'ttb',
  paginationDirection: 'ltr',
} );
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

### Navigation carousels (`isNavigation: true`), from source

A focused navigation slide responds to **Space** and **Enter** by moving the synced carousels to that slide (`Sync.ts`, `TRIGGER_KEYS = [' ', 'Enter']`). Clicking it does the same.

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

## Live region

Content in a slide that just became visible is hard to find with a screen reader. Sighted users see it, but users who rely on audio or Braille have to look for the updated slide. A live region solves this. For some screen readers, adding `aria-live="polite"` alone does **not** work, because typical carousels do not meet the `aria-relevant` conditions those readers use to detect a slide change. Splide works around this.

How it works **(source, `Live.ts`)**:

1. It is enabled when `live: true` (default) and `isNavigation` is not `true`. A navigation carousel never activates a live region, because it controls another carousel.
2. On mount it sets `aria-atomic="true"` on the track. It sets `aria-live="off"` if autoplay is running, otherwise `aria-live="polite"`.
3. On every `moved` and `scrolled` event, it sets `aria-busy="true"` on the track and appends `<span class="splide__sr">…</span>` to the track. After 90 ms it removes the span and sets `aria-busy="false"`. JAWS needs `aria-atomic` for `aria-busy` to work, and Windows Narrator goes silent if the span is removed immediately.
4. It switches to `aria-live="off"` on `autoplay:play` and back to `polite` on `autoplay:pause`. Without this, screen readers would keep speaking while the page is open, even with the carousel out of view. With `pauseOnFocus: true`, moving focus into the carousel pauses autoplay and turns the live region back on.
5. `.splide__sr` is visually hidden by `splide.min.css` and `splide-core.min.css`. Without either stylesheet, add the rule yourself:

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

- Disable it with `live: false`. **(source)** You can also toggle it at runtime with `splide.Components.Live.disable(true | false)`. This does nothing when the live region is not enabled.
- The author tested it with Windows Narrator, JAWS, NVDA and VoiceOver. It cannot be guaranteed for every screen reader, browser and device.

## Autoplay play/pause (WCAG 2.2.1, 2.2.2)

> The user is allowed to turn off the time limit before encountering it. (Timing Adjustable)
>
> For any auto-updating information that starts automatically and is presented in parallel with other content, there is a mechanism for the user to pause, stop, or hide it. (Pause, Stop, Hide)

The W3C pattern requires an autoplaying carousel to:

1. Have a button that pauses and resumes autoplay.
2. Pause autoplay when focus moves into the carousel.
3. Pause autoplay while the mouse hovers over the carousel.

Users with disabilities, such as blindness, dexterity impairments or cognitive limitations, need enough time to read the content. If the carousel rotates on its own, give users a pause/resume button. Splide supports a play/pause toggle built on the APG Toggle Button pattern, which you add in HTML (markup below). Requirements 2 and 3 need no work: `pauseOnFocus` and `pauseOnHover` are `true` by default.

The toggle's attributes **(source, `Autoplay.ts`)**, set only when `autoplay` is truthy:

| Attribute / class | Value |
|---|---|
| `aria-controls` | ID of the track |
| `aria-label` | `i18n.play` ("Start autoplay") while autoplay is stopped, `i18n.pause` ("Pause autoplay") while it runs |
| `is-active` class | Present while running. The core CSS uses it to swap `.splide__toggle__play` and `.splide__toggle__pause` |

"Stopped" means paused by the toggle or by `autoplay: 'pause'`. A temporary hover or focus pause does not change the label. Splide does not set `aria-pressed`.

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

```js
new Splide( '.splide', {
  autoplay     : true,
  reducedMotion: {
    speed      : 0,
    rewindSpeed: 0,
    autoplay   : 'pause',
  }, // this is the default; shown for clarity
} ).mount();
```

## Focus visible (WCAG 2.4.7)

> Any keyboard operable user interface has a mode of operation where the keyboard focus indicator is visible. (Focus Visible)

Focus styles should appear for keyboard users only, which is what `:focus-visible` does. When v4 came out, IE did not support it and Safari had only added it in its latest release (2022-03-15). Splide therefore patches the gap with a class, and the v4 themes (`splide.min.css` and the `themes/` files) include focus styles for arrows, pagination, slides (thumbnails) and the toggle.

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

The default theme's rules, for reference (from `dist/css/splide.min.css`, 4.1.4):

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

```js
{
  focusableNodes: 'a, button, ..., area, [contenteditable]',
}
```

A complete value (the `...` above stands for the defaults):

```js
new Splide( '.splide', {
  focusableNodes: 'a, button, textarea, input, select, iframe, area, audio[controls], video[controls], summary, [contenteditable]',
} ).mount();
```

`focusableNodes` cannot handle `tabindex="0"`, because that value has to be restored when the slide becomes visible again. **(source)** Splide removes the `tabindex` attribute from matched elements when their slide becomes visible, so a `[tabindex]` selector would leave those elements unfocusable. Toggle such elements yourself with the `visible` and `hidden` events. Slides that start out of view never fire `hidden` (tested), so also set the initial state on `ready`:

```js
var splide = new Splide( '.splide' );

function setFocusable( slide, focusable ) {
  slide.querySelectorAll( '[data-focusable]' ).forEach( function ( el ) {
    el.setAttribute( 'tabindex', focusable ? '0' : '-1' );
  } );
}

splide.on( 'visible', function ( Slide ) { setFocusable( Slide.slide, true ); } );
splide.on( 'hidden', function ( Slide ) { setFocusable( Slide.slide, false ); } );

// Slides that start out of view never fire `hidden`, so set the initial state on `ready`.
splide.on( 'ready', function () {
  splide.Components.Slides.forEach( function ( Slide ) {
    setFocusable( Slide.slide, Slide.slide.classList.contains( 'is-visible' ) );
  } );
} );

splide.mount();
```

Visibility rules **(source, `Slide.ts`)**:

- "Visible" means the slide lies completely inside the track's rectangle, rounded to whole pixels. For `type: 'fade'`, the active slide is the only visible one.
- A real slide gets `aria-hidden="true"` when it is not visible **and** not active. The active slide is never hidden, even if only partly in view. A clone gets it whenever it is not visible.
- `aria-hidden` is not changed while the carousel is moving or scrolling, which keeps Windows Narrator from reading early. Slide status updates on `moved`, `scrolled` and the internal shift event, and on repositioning (`mounted`, `resized`, `updated`, `refresh`). With `updateOnMove: true` it also updates on `move`. Classes, `tabindex` and the `active`/`visible` events change then, but `aria-hidden` waits for `moved`.

## Role and ARIA attribute reference (WCAG 4.1.2)

> For all user interface components, the name and role can be programmatically determined; states, properties, and values that can be set by the user can be programmatically set; and notification of changes to these items is available to user agents, including assistive technologies. (Name, Role, Value)

Splide sets the attributes. You add an accessible name and translate the texts (see [i18n.md](i18n.md)).

### Basic carousel (no pagination)

**Root (`.splide`)**

| Attribute | Value |
|---|---|
| `role` | `region` (the `role` option) if the element is not `<section>`. **(source)** Not set if the HTML already has a `role` or the option is empty |
| `aria-roledescription` | `carousel` (`i18n.carousel`) |
| `aria-label` | **(source)** The `label` option, if non-empty |
| `aria-labelledby` | **(source)** The `labelledby` option, if non-empty |
| `id` | **(source)** Kept if present, otherwise `splide01`, `splide02`, and so on |

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
| `aria-label` | `%s of %s`, meaning "{slide number} of {slide length}" (`i18n.slideLabel`). **(source)** A slide's own HTML `aria-label` is kept and restored on destroy |
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
| `aria-label` | "Go to slide %s" (`i18n.slideX`). **(source)** Overwrites any HTML `aria-label` on the slide |
| `aria-current` | **(source)** `true` on the active slide, otherwise absent |
| `tabindex` | **(source)** `0` when visible, `-1` when hidden (`slideFocus` defaults to `true` here) |

**Arrows (`.splide__arrow--prev` / `.splide__arrow--next`)**

| Attribute | Value |
|---|---|
| `aria-controls` | The ID of the track element |
| `aria-label` | Previous arrow: "Previous slide" (`i18n.prev`) or "Go to last slide" (`i18n.last`). Next arrow: "Next slide" (`i18n.next`) or "Go to first slide" (`i18n.first`). **(source)** The "first"/"last" texts are used when the arrow would wrap around (`loop` or `rewind`). Splide overwrites the `aria-label` on arrows written in HTML too |
| `disabled` | Set when there is no slide to go to |

**(source)** Generated arrow markup: `<div class="splide__arrows splide__arrows--{direction}">` holds `<button class="splide__arrow splide__arrow--prev" type="button">` with `<svg ... focusable="false"><path d="..."/></svg>`, inserted before the track.

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

**(source)** Splide appends the generated `<ul>` as the last child of the track's parent (normally the root), so it comes after a toggle button written before it. A `.splide__pagination` element already in the HTML is reused instead, which lets you control its position in the DOM and in the Tab order.

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
| Root | Existing `id`, or `splide` + a two-digit counter: `splide01`, `splide02`, and so on |
| Track | Existing `id`, or `{rootId}-track` |
| List | Existing `id`, or `{rootId}-list` |
| Slide n | `{rootId}-slide01`, `-slide02`, and so on. **Overwrites any existing slide `id`** and is not restored on destroy |
| Clone n | `{rootId}-clone01`, and so on |

Give the root a stable `id` if anything references slide IDs, such as `aria-controls` targets or deep links.

## Accessible markup example

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">

<section id="featured" class="splide" aria-labelledby="featured-heading">
  <h2 id="featured-heading">Featured products</h2>

  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" aria-label="Linen shirt">
        <a href="/products/linen-shirt"><img src="shirt.jpg" alt="Linen shirt in sand color"></a>
      </li>
      <li class="splide__slide" aria-label="Wool scarf">
        <a href="/products/wool-scarf"><img src="scarf.jpg" alt="Grey wool scarf"></a>
      </li>
      <li class="splide__slide" aria-label="Canvas tote">
        <a href="/products/canvas-tote"><img src="tote.jpg" alt="Natural canvas tote bag"></a>
      </li>
    </ul>
  </div>

  <!-- Autoplay controls: required when autoplay is on (WCAG 2.2.2) -->
  <div class="splide__progress"><div class="splide__progress__bar"></div></div>
  <button class="splide__toggle" type="button">
    <span class="splide__toggle__play">Start</span>
    <span class="splide__toggle__pause">Pause</span>
  </button>
</section>

<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  new Splide( '#featured', {
    perPage : 2,
    autoplay: true,
    breakpoints: { 640: { perPage: 1 } },
  } ).mount();
</script>
```

This works without extra JS because:

- `<section>` with `aria-labelledby` is a named `region`, so Splide adds no `role`.
- Each slide has a content-based `aria-label`, which Splide keeps instead of "1 of 3".
- The visible toggle texts "Start" and "Pause" appear inside Splide's labels "Start autoplay" and "Pause autoplay", which satisfies WCAG 2.5.3 Label in Name. If you rename the visible texts, change `i18n.play` and `i18n.pause` to match.
- **(source)** `perPage` is lower than the slide count. Autoplay only starts when there are more slides than `perPage` (`Slides.isEnough()`). Otherwise nothing plays, yet the toggle still shows the "Pause autoplay" state, and after one click it is stuck on "Start autoplay" (tested with `perPage: 3` and three slides).

## What destroy() does to attributes

From source (`Elements.ts`, `Slide.ts`, `Arrows.ts`, `Pagination.ts`, `Live.ts`):

| Element | `destroy(false)` or breakpoint `destroy: true` | `destroy()` / `destroy(true)` / breakpoint `destroy: 'completely'` |
|---|---|---|
| Root | Removes `style` and `aria-roledescription`. **Keeps** `role`, `aria-label` and `aria-labelledby` | Removes `role`, `tabindex`, `disabled`, `aria-controls`, `aria-current`, `aria-label`, `aria-labelledby`, `aria-hidden`, `aria-orientation`, `aria-roledescription` and `style`, **including values written in HTML** |
| Track, list | Remove those same attributes plus `style`. The track also loses `aria-live`, `aria-atomic` and `aria-busy` | Same |
| Slides | Lose status classes and all of the attributes above. Original `style` and HTML `aria-label` restored | Same |
| Arrows | Generated arrows are removed. HTML arrows lose all of the attributes above, including your own `aria-label` | Same |
| Pagination | Generated `<ul>` is removed. With an HTML placeholder, only its children are removed | Same |
| Toggle button | Keeps `aria-controls`, `aria-label` and `is-active`: Autoplay's `destroy()` only cancels the timer (tested) | Same |

`splide.destroy()` with no argument means `completely = true`.

**(source) Destroy leaves `tabindex="-1"` on focusable elements inside slides.** `Slide.destroy()` cleans the slide element but not its descendants. Links and buttons in slides that were hidden at destroy time stay out of the Tab order, which is a real problem when a breakpoint destroys the carousel to show a grid. Clear them on `destroy`. Register the handler in an extension so it is registered again on every breakpoint remount (see [events.md](events.md)):

```js
function RestoreFocusables( splide ) {
  return {
    mount: function () {
      splide.on( 'destroy', function () {
        // Runs after all components are destroyed (clones already removed).
        splide.root.querySelectorAll( '.splide__slide' ).forEach( function ( slide ) {
          slide.querySelectorAll( splide.options.focusableNodes ).forEach( function ( el ) {
            el.removeAttribute( 'tabindex' ); // also removes any tabindex you set yourself
          } );
        } );
      } );
    },
  };
}

new Splide( '.splide', {
  breakpoints: { 640: { destroy: true } },
} ).mount( { RestoreFocusables: RestoreFocusables } );
```

## Checklist

From the docs:

1. Give every carousel an accessible name: `aria-labelledby` pointing at a visible heading, or `aria-label` / the `label` option.
2. Translate all texts with `i18n` for your users ([i18n.md](i18n.md)). Prefer content-based wording ("Next article" over "Next slide").
3. Keep the default `region` landmark for content carousels. Use `role: 'group'` (or `role="group"` in HTML) for decorative or unrelated ones. Do not add `region` to elements with their own landmark roles (`<header>`, `<nav>`, and so on).
4. A `<section>` root needs `aria-label` or `aria-labelledby`.
5. With autoplay, provide the play/pause toggle and keep `pauseOnHover` and `pauseOnFocus` at `true`.
6. Do not rely on autoplay alone to move slides. Keep arrows, pagination or drag available, because reduced motion pauses autoplay.
7. Without a theme, add focus styles with `:focus-visible`, plus `.splide.is-focus-in ...:focus` as a fallback.
8. If slides contain `<area>`, `<audio controls>`, `<video controls>`, `<summary>` or `[contenteditable]`, extend `focusableNodes`.
9. Handle elements with `tabindex="0"` inside slides yourself, via the `visible` and `hidden` events plus an initial pass on `ready`.
10. With `isNavigation: true`, set `pagination: false`. Otherwise roles and ARIA attributes get mixed up.
11. If breakpoints destroy the carousel, change `label` in that breakpoint so it does not say "carousel" or "slider".
12. Keep `paginationKeyboard: true`, which the W3C pattern requires. Turn on `keyboard` only if you want the extra arrow-key shortcuts.

## Not handled automatically

From source unless noted:

- **No default accessible name.** Splide sets `aria-label` only when you provide one. An unnamed `region` is not exposed as a landmark.
- **The toggle button is not generated.** Write `<button class="splide__toggle" type="button">` yourself. Splide sets only `aria-controls`, `aria-label` and `is-active`, not `aria-pressed`. Make sure the visible text is part of the accessible name (WCAG 2.5.3).
- **The progress bar** gets no ARIA. Do not add `role="progressbar"` unless you also maintain `aria-valuenow`.
- **Image `alt` text, headings and slide content semantics** are up to you. For lazy-loaded images, set `alt` in the HTML.
- **Custom arrow labels:** Splide overwrites `aria-label` on arrows from `i18n`, so set texts through `i18n`, not HTML.
- **Navigation slide labels:** Splide overwrites them with `i18n.slideX` ("Go to slide %s").
- **Focusable elements outside `focusableNodes`** and elements with `tabindex="0"`, as described above.
- **Focus does not move into the slides** after an arrow or dot is activated.
- **Arrow-key shortcuts** outside the pagination are off unless you set `keyboard`.
- **Duplicated IDs in loop clones:** clones are deep copies (`cloneNode(true)`). The clone itself gets a new ID, but descendant IDs, and any `aria-labelledby`/`aria-describedby` that point at them, are duplicated. Avoid IDs inside slides when `type: 'loop'`.
- **Clones copy the original's labels.** They inherit `role`, `aria-roledescription` and `aria-label`, and rely on `aria-hidden="true"` whenever they are not fully visible.
- **Reduced motion** covers only `speed`, `rewindSpeed` and `autoplay`. Your own CSS animations and transitions inside slides are not affected, and drag stays animated.
- **The live region** is off while autoplay runs, never runs on navigation carousels, and is not guaranteed for every screen reader.
- **Root attributes on full destroy:** `destroy()` removes `role`, `aria-label` and `aria-labelledby` from the root, even if they came from your HTML.
- **`tabindex="-1"` left after destroy:** focusable descendants of slides that were hidden at destroy time keep it. See [What destroy() does to attributes](#what-destroy-does-to-attributes) for a fix.
- **Stale slide role** after toggling `pagination` at runtime. Call `refresh()`.

## Docs vs source

| Topic | Docs say | Source (4.1.3 / 4.1.4) does |
|---|---|---|
| Navigation slide role without `slideFocus` | `group` | `setAttribute(slide, 'role', '')`, which **removes** `role` |
| Navigation slide `aria-roledescription` | "Not assigned because the slide is interactive" | Removed only when `slideFocus` is true. Otherwise `slide` remains |
| Inactive pagination dot `aria-selected` | `false` | Attribute **removed** (absent) |
| Pagination `aria-label` default | "Select slide to show" (accessibility page) | `'Select a slide to show'` (`constants/i18n.ts`; the i18n page agrees) |
| Focus CSS example | `.splide.is-focus-in .splide_arrow:focus` | The class is `splide__arrow` (two underscores) |
| `keyboard` key table (Options page) | Down arrow: "Go to the previous page in TTB" | Down arrow goes to the **next** page (`go('>')`) |
| List `role="presentation"`, slide `aria-current`, list `aria-orientation` (nav + ttb), pagination `aria-orientation`, generated IDs | Not documented | Set as described above |

## Open concerns from the docs

The author asked accessibility experts for feedback on two points:

- The APG Tabs pattern says Tab from the tab list moves to "the tabpanel unless the first element containing meaningful content inside the tabpanel is focusable", and its examples make each tabpanel focusable with `tabindex="0"`. MDN says "Focusable elements should have interactive semantics", and `tabpanel` is not interactive. Firefox DevTools flags the W3C example as failing WCAG. Splide therefore does not make slides focusable by default.
- Should slides in a Tabbed carousel be focusable? If they should, Basic carousel slides should be too for consistency, because both patterns share `aria-roledescription="carousel"` and users expect the same behavior.
