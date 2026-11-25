# Themes

Source: https://splidejs.com/guides/themes/
Also: https://splidejs.com/guides/getting-started/#importing-css. The rules below are read from the v4.1.4 `dist/css` files (identical to 4.1.3) and the Sass sources in `src/css/` (v4.1.3).

## Contents

- [Theme files](#theme-files)
- [What each file styles](#what-each-file-styles)
- [Core rules (present in every file)](#core-rules-present-in-every-file)
- [Default theme rules](#default-theme-rules)
- [Sky Blue and Sea Green differences](#sky-blue-and-sea-green-differences)
- [Classes the CSS relies on](#classes-the-css-relies-on)
- [Customizing](#customizing)
- [Gotchas](#gotchas)

---

## Theme files

The docs page says only that theme CSS files are in `dist/css/themes/`: pick one and link to it. It shows previews of **Default**, **Skyblue** and **Sea Green**, each a nine-slide carousel with thumbnails. Everything else here comes from the CSS files.

```html
<!-- pick exactly ONE of these -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-default.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-skyblue.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/themes/splide-sea-green.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide-core.min.css">
```

| File | Bytes | Accent color | Notes |
|---|---|---|---|
| `dist/css/splide.min.css` | 5,002 | `#0bf` (focus only) | Byte-identical to `themes/splide-default.min.css` |
| `dist/css/themes/splide-default.min.css` | 5,002 | `#0bf` (focus only) | Grey round arrows, small grey/white dots |
| `dist/css/themes/splide-skyblue.min.css` | 5,137 | `#00bfff`, hover `#66d9ff` | Transparent arrows with blue icons, blue dots |
| `dist/css/themes/splide-sea-green.min.css` | 5,354 | `#20b2aa`, hover `#57e1d9` | Transparent green arrows, bar-shaped dots, `padding: 3em` on the root, rounded slides |
| `dist/css/splide-core.min.css` | 1,964 | — | Structural rules only |

Each theme file is self-contained: core rules plus theme rules. Never load a theme **and** `splide-core.min.css`, or two themes. There are no unminified CSS files in `dist`.

The themes are compiled from Sass variables (`src/css/template/default/foundation/colors.scss` and the `@use ... with (...)` blocks in `src/css/themes/*/index.scss`). The CDN serves only compiled CSS, so without a build step you customize with plain CSS overrides.

---

## What each file styles

| Area | Core | Default / Sky Blue / Sea Green |
|---|---|---|
| Root hidden until mounted, `position: relative` | yes | yes |
| Track `overflow: hidden`, list flex row, slide box model | yes | yes |
| Fade, TTB, RTL and draggable modifiers | yes | yes |
| Pagination **layout** (flex, centered, list reset, hidden when not overflowing) | yes | yes |
| Pagination **position and dot appearance** | no | yes |
| Arrow appearance and position | **no arrow rules at all** | yes |
| Progress bar | `width: 0` only | plus `background` and `height: 3px` |
| Play/pause toggle | show/hide of play and pause icons | plus `cursor` and focus outline |
| Lazy-load spinner | yes (`#999`) | yes (theme color in Sky Blue / Sea Green) |
| Screen-reader text `.splide__sr` | yes | yes |
| Focus rings (`:focus-visible` and `.is-focus-in`) | no | yes |
| Thumbnail (`isNavigation`) active border | no | yes |
| Slide `-webkit-tap-highlight-color`, `:focus` reset | no | yes |

Docs say `splide-core.min.css` "does not include arrows, pagination and progress bar styles". In the source it does include the pagination layout, `.splide__progress__bar { width: 0 }`, the toggle icon switching, the spinner and `.splide__sr`. What it lacks is arrow styling, dot/bar appearance and focus rings.

---

## Core rules (present in every file)

These rules make the carousel work. Don't override them unless you know why.

```css
@keyframes splide-loading {
  0%   { transform: rotate(0); }
  to   { transform: rotate(1turn); }
}

.splide { position: relative; visibility: hidden; }
.splide.is-initialized,
.splide.is-rendered { visibility: visible; }

.splide__track { overflow: hidden; position: relative; z-index: 0; }

.splide__list {
  backface-visibility: hidden;
  display: flex;
  height: 100%;
  margin: 0 !important;
  padding: 0 !important;
}
.splide.is-initialized:not(.is-active) .splide__list { display: block; }

.splide__slide {
  backface-visibility: hidden;
  box-sizing: border-box;
  flex-shrink: 0;
  list-style-type: none !important;
  margin: 0;
  position: relative;
}
.splide__slide img { vertical-align: bottom; }

.splide__container { box-sizing: border-box; position: relative; }

.splide__track--draggable {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
.splide__track--fade > .splide__list > .splide__slide { margin: 0 !important; opacity: 0; z-index: 0; }
.splide__track--fade > .splide__list > .splide__slide.is-active { opacity: 1; z-index: 1; }
.splide--rtl { direction: rtl; }
.splide__track--ttb > .splide__list { display: block; }

.splide__pagination {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  margin: 0;
  pointer-events: none;
}
.splide__pagination li {
  display: inline-block;
  line-height: 1;
  list-style-type: none;
  margin: 0;
  pointer-events: auto;
}
.splide:not(.is-overflow) .splide__pagination { display: none; }

.splide__progress__bar { width: 0; }

.splide__spinner {
  animation: splide-loading 1s linear infinite;
  border: 2px solid #999;
  border-left-color: transparent;
  border-radius: 50%;
  bottom: 0; left: 0; right: 0; top: 0;
  contain: strict;
  display: inline-block;
  height: 20px; width: 20px;
  margin: auto;
  position: absolute;
}

.splide__sr {
  clip: rect(0 0 0 0);
  border: 0;
  height: 1px; width: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
}

.splide__toggle.is-active .splide__toggle__play,
.splide__toggle__pause { display: none; }
.splide__toggle.is-active .splide__toggle__pause { display: inline; }
```

(The minified files also contain `-ms-` flexbox prefixes, omitted above.)

What these rules mean in practice:

| Rule | Effect |
|---|---|
| `.splide { visibility: hidden }` until `.is-initialized` / `.is-rendered` | Carousels are invisible until `mount()` runs. If the script fails or never mounts, the carousel stays invisible but still takes up space. |
| `.splide.is-initialized:not(.is-active) .splide__list { display: block }` | After `destroy()` (for example `destroy: true` in a breakpoint), `is-active` is removed but `is-initialized` stays, so slides stack vertically as normal blocks |
| `.splide:not(.is-overflow) .splide__pagination { display: none }` | Since v4.1, pagination is hidden while all slides fit in the track |
| `.splide__pagination { pointer-events: none }` + `li { pointer-events: auto }` | The pagination bar doesn't block drags on the slides underneath, but the dots stay clickable |
| `.splide__track--fade ... { opacity: 0 }` / `.is-active { opacity: 1 }` | Fade type: only the active slide is visible. Since v4.1 the slides keep `display: flex` layout instead of absolute positioning. |
| `.splide__track--ttb > .splide__list { display: block }` | Vertical carousels stack slides in block flow |
| `.splide__toggle` rules | `is-active` on the toggle means **playing**: the pause icon shows. Without `is-active` (paused/stopped) the play icon shows. |
| `.splide__container` | Matches **nothing** Splide generates. The slide container class is `splide__slide__container`. This is a dead rule (Sass `.splide { &__container {} }`). |

---

## Default theme rules

Everything below is in `splide.min.css` / `splide-default.min.css` on top of the core rules.

### Arrows

```css
.splide__arrow {
  align-items: center;
  background: #ccc;
  border: 0;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  height: 2em;
  justify-content: center;
  opacity: .7;
  padding: 0;
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 2em;
  z-index: 1;
}
.splide__arrow svg { fill: #000; height: 1.2em; width: 1.2em; }
.splide__arrow:hover:not(:disabled) { opacity: .9; }
.splide__arrow:disabled { opacity: .3; }
.splide__arrow:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide__arrow--prev { left: 1em; }
.splide__arrow--prev svg { transform: scaleX(-1); }
.splide__arrow--next { right: 1em; }
.splide.is-focus-in .splide__arrow:focus { outline: 3px solid #0bf; outline-offset: 3px; }

/* RTL: swap sides and mirror icons */
.splide__arrows--rtl .splide__arrow--prev { left: auto; right: 1em; }
.splide__arrows--rtl .splide__arrow--prev svg { transform: scaleX(1); }
.splide__arrows--rtl .splide__arrow--next { left: 1em; right: auto; }
.splide__arrows--rtl .splide__arrow--next svg { transform: scaleX(-1); }

/* TTB: center horizontally, put arrows at top/bottom, rotate icons */
.splide__arrows--ttb .splide__arrow { left: 50%; transform: translate(-50%); }
.splide__arrows--ttb .splide__arrow--prev { top: 1em; }
.splide__arrows--ttb .splide__arrow--prev svg { transform: rotate(-90deg); }
.splide__arrows--ttb .splide__arrow--next { bottom: 1em; top: auto; }
.splide__arrows--ttb .splide__arrow--next svg { transform: rotate(90deg); }
```

The default SVG is a 40×40 viewBox with a right-pointing chevron (`arrowPath` option). The prev arrow is that icon mirrored with `scaleX(-1)`. Arrows are positioned relative to the nearest positioned ancestor, which is the root (`.splide { position: relative }`) unless you wrap the arrows and track in your own `position: relative` element.

### Pagination

```css
.splide__pagination {
  bottom: .5em;
  left: 0;
  padding: 0 1em;
  position: absolute;
  right: 0;
  z-index: 1;
}
.splide__pagination__page {
  background: #ccc;
  border: 0;
  border-radius: 50%;
  display: inline-block;
  height: 8px;
  margin: 3px;
  opacity: .7;
  padding: 0;
  position: relative;
  transition: transform .2s linear;
  width: 8px;
}
.splide__pagination__page.is-active { background: #fff; transform: scale(1.4); z-index: 1; }
.splide__pagination__page:hover { cursor: pointer; opacity: .9; }
.splide__pagination__page:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__pagination__page:focus { outline: 3px solid #0bf; outline-offset: 3px; }

/* Vertical pagination (direction or paginationDirection 'ttb') */
.splide__pagination--ttb {
  bottom: 0;
  display: flex;
  flex-direction: column;
  left: auto;
  padding: 1em 0;
  right: .5em;
  top: 0;
}
```

The pagination is absolutely positioned **over** the bottom of the slides. To put it below the carousel, set `position: static` (see Customizing).

### Progress bar, toggle, slides, thumbnails

```css
.splide__progress__bar { background: #ccc; height: 3px; }

.splide__toggle { cursor: pointer; }
.splide__toggle:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__toggle:focus { outline: 3px solid #0bf; outline-offset: 3px; }

.splide__slide { -webkit-tap-highlight-color: rgba(0, 0, 0, 0); }
.splide__slide:focus { outline: 0; }
@supports (outline-offset: -3px) {
  .splide__slide:focus-visible { outline: 3px solid #0bf; outline-offset: -3px; }
}
@media screen and (-ms-high-contrast: none) {
  .splide__slide:focus-visible { border: 3px solid #0bf; }
}
@supports (outline-offset: -3px) {
  .splide.is-focus-in .splide__slide:focus { outline: 3px solid #0bf; outline-offset: -3px; }
}
@media screen and (-ms-high-contrast: none) {
  .splide.is-focus-in .splide__slide:focus { border: 3px solid #0bf; }
  .splide.is-focus-in .splide__track > .splide__list > .splide__slide:focus { border-color: #0bf; }
}

/* isNavigation (thumbnail) carousels */
.splide__track--nav > .splide__list > .splide__slide { border: 3px solid transparent; cursor: pointer; }
.splide__track--nav > .splide__list > .splide__slide.is-active { border: 3px solid #000; }
```

The progress bar has **no** container styles. `.splide__progress` is unstyled, and Splide sets the bar's inline `width` from 0% to 100% during each autoplay interval.

### Default theme values at a glance

| What | Selector | Value |
|---|---|---|
| Arrow size | `.splide__arrow` | `2em` × `2em`, `border-radius: 50%` |
| Arrow background / opacity | `.splide__arrow` | `#ccc`, `.7` (hover `.9`, disabled `.3`) |
| Arrow icon | `.splide__arrow svg` | `fill: #000`, `1.2em` × `1.2em` |
| Arrow position | `--prev` / `--next` | `left: 1em` / `right: 1em`, `top: 50%` + `translateY(-50%)` |
| Dot size | `.splide__pagination__page` | `8px` × `8px`, `margin: 3px`, `#ccc`, opacity `.7` |
| Active dot | `.splide__pagination__page.is-active` | `#fff`, `scale(1.4)` |
| Pagination position | `.splide__pagination` | absolute, `bottom: .5em`, `padding: 0 1em` |
| Focus ring | arrows, dots, toggle | `outline: 3px solid #0bf; outline-offset: 3px` |
| Slide focus ring | `.splide__slide` | `outline: 3px solid #0bf; outline-offset: -3px` |
| Progress bar | `.splide__progress__bar` | `height: 3px`, `background: #ccc` |
| Thumbnail border | `.splide__track--nav ... .splide__slide` | `3px solid transparent`, active `3px solid #000` |
| Spinner | `.splide__spinner` | `20px`, `2px solid #999`, left side transparent |

---

## Sky Blue and Sea Green differences

Both themes replace the arrow, pagination and thumbnail rules and recolor every focus ring and the spinner. Everything not listed matches the default theme.

| Rule | Default | Sky Blue | Sea Green |
|---|---|---|---|
| Accent (focus rings, spinner border) | `#0bf` (spinner `#999`) | `#00bfff` | `#20b2aa` |
| `.splide__arrow` | `#ccc` round, `2em`, opacity `.7` | `background: transparent; border-radius: 0`, `2.5em` × `2.5em`, no opacity | same as Sky Blue |
| `.splide__arrow svg` | `fill: #000`, `1.2em` | `fill: #00bfff`, `2.5em`, `transition: fill .2s linear` | `fill: #20b2aa`, `2.5em`, same transition |
| `.splide__arrow:hover:not(:disabled)` | `opacity: .9` | `svg { fill: #66d9ff }` | `svg { fill: #57e1d9 }` |
| `.splide__arrow:disabled` | `opacity: .3` | `opacity: .3` | `opacity: .3` |
| `.splide__pagination` | `bottom: .5em` | `bottom: .5em` | `bottom: 1em` |
| `.splide__pagination__page` | `8px` × `8px` circle, opacity `.7`, `transition: transform .2s linear` | `10px` × `10px` circle, no opacity, `transition: background-color .2s linear, transform .2s linear` | `20px` × `5px` bar, `border-radius: 2.5px`, no opacity, `transition: background-color .2s linear` |
| `.splide__pagination__page.is-active` | `#fff`, `scale(1.4)` | `#00bfff`, `scale(1.4)` | `#20b2aa`, **no scale** |
| `.splide__pagination__page:hover` | `opacity: .9` | `background: #66d9ff; opacity: .9` | `background: #57e1d9; opacity: .9` |
| `.splide__pagination--ttb` | `right: .5em` | `right: .5em` | `right: 1em`, and `.splide__pagination--ttb .splide__pagination__page { height: 20px; width: 5px }` |
| Thumbnails (`.splide__track--nav ... .splide__slide`) | border transparent, active `#000` | adds `opacity: .7`, active `opacity: 1`, border `#00bfff` | adds `border-radius: 4px; opacity: .7`, active `opacity: 1`, border `#20b2aa` |
| `.splide__slide` | — | — | `border-radius: 4px` |
| `.splide` (root) | — | — | `padding: 3em` |
| `.splide__slider > .splide__arrows .splide__arrow--prev / --next` | — | — | `left: -2.5em` / `right: -2.5em` |

Sea Green's `.splide__slider` rules are left over from v3. v4 no longer handles `.splide__slider` (see `version4.md`). To use them, wrap the track in `<div class="splide__slider">` and add `.splide__slider { position: relative; }`. Splide inserts its generated arrows right before the track, so they become a direct child of `.splide__slider`. The arrows then sit at `-2.5em`, inside the root's `3em` padding beside the slides. Without that wrapper, Sea Green arrows sit at `1em` inside the slide area like the other themes.

---

## Classes the CSS relies on

Status classes from `constants/classes.ts` (`is-` prefix). `STATUS_CLASSES` = `is-active`, `is-visible`, `is-prev`, `is-next`, `is-loading`, `is-focus-in`, `is-overflow`. Slides lose these on destroy. `is-initialized` is not in the list.

| Class | Element | Present when |
|---|---|---|
| `is-initialized` | root | Added at the end of `mount()`. **Not removed by `destroy()`.** |
| `is-rendered` | root | Added to HTML generated by `SplideRenderer` (pre-rendered markup), unless `config.hidden` |
| `is-active` | root | While mounted. Added with the modifier classes, removed on destroy. |
| `is-active` | slide | The slide at the current index. With `cloneStatus: true` (default), clones of the active slide get it too. |
| `is-active` | pagination button | The dot of the current page |
| `is-active` | `.splide__toggle` | Autoplay is playing (not stopped). Shows the pause icon. |
| `is-visible` | slide | The slide lies fully inside the track's box. For `fade`, only the active slide. |
| `is-prev` / `is-next` | slide | Slide index is `index - 1` / `index + 1` |
| `is-loading` | slide | A lazy-loaded image in it is loading (`lazyLoad`) |
| `is-focus-in` | root | Focus entered via the keyboard. Toggled on every `focusin`: on when the last input was `keydown`, off when it was `touchstart`/`mousedown`. Not removed when focus leaves, so always pair it with `:focus`. |
| `is-overflow` | root | v4.1+. The slides are larger than the list (`Layout.isOverflow()`). Always true for `fade`. Re-evaluated on resize, with an `overflow` event. |

Modifier classes (from `Elements.getClasses()`, recomputed on `updated`, so they follow breakpoints):

| Root | Track | When |
|---|---|---|
| `splide--slide`, `splide--loop`, `splide--fade` | `splide__track--slide`, `--loop`, `--fade` | `type` |
| `splide--ltr`, `splide--rtl`, `splide--ttb` | `splide__track--ltr`, `--rtl`, `--ttb` | `direction` |
| `splide--draggable` | `splide__track--draggable` | `drag` is truthy (`true` or `'free'`) |
| `splide--nav` | `splide__track--nav` | `isNavigation: true` |

Other generated classes:

| Class | Element |
|---|---|
| `splide__arrows--ltr` / `--rtl` / `--ttb` | arrows wrapper (from `direction`) |
| `splide__pagination--ltr` / `--rtl` / `--ttb` | pagination `<ul>` (from `paginationDirection`, falling back to `direction`) |
| `splide__slide--clone` | clones in `loop` mode |
| `splide__spinner` | `<span>` inserted next to a lazy image while it loads |
| `splide__sr` | visually hidden `<span>` that Splide briefly appends to the track for the live region |

There are **no** `is-rtl`, `is-ttb`, `is-dragging`, `is-moving`, `is-disabled` or `is-paused` classes. Use `splide--rtl` / `splide__track--ttb` and so on for direction, `button:disabled` for arrows at the ends, and `.splide__toggle:not(.is-active)` for paused autoplay.

---

## Customizing

### 1. Override a theme with your own CSS

Load your stylesheet **after** the theme and use selectors at least as specific as the theme's. Most theme selectors are one or two classes. The `.is-focus-in ... :focus` rules and the `.splide__track--nav > .splide__list > .splide__slide.is-active` rule use up to four. The only `!important` declarations are the list/slide margin, padding and list-style resets in core.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<link rel="stylesheet" href="/css/carousel.css">
```

```css
/* Bigger, darker arrows */
.splide__arrow {
  width: 3rem;
  height: 3rem;
  background: rgba(0, 0, 0, .6);
  opacity: 1;
}
.splide__arrow svg { fill: #fff; width: 1.25rem; height: 1.25rem; }
.splide__arrow:hover:not(:disabled) { background: rgba(0, 0, 0, .8); opacity: 1; }
.splide__arrow:disabled { opacity: .3; }

/* Move arrows outside the slides (give the root side padding) */
.splide.my-carousel { padding: 0 3.5rem; }
.my-carousel .splide__arrow--prev { left: 0; }
.my-carousel .splide__arrow--next { right: 0; }

/* Pagination below the carousel instead of over it */
.splide__pagination { position: static; padding: 1rem 0 0; }

/* Brand-colored dots */
.splide__pagination__page { background: #c4c4c4; opacity: 1; width: 10px; height: 10px; }
.splide__pagination__page.is-active { background: #e4002b; transform: scale(1.2); }

/* Focus rings in the brand color (keep both forms) */
.splide__arrow:focus-visible,
.splide__pagination__page:focus-visible,
.splide__toggle:focus-visible { outline-color: #e4002b; }
.splide.is-focus-in .splide__arrow:focus,
.splide.is-focus-in .splide__pagination__page:focus,
.splide.is-focus-in .splide__toggle:focus { outline-color: #e4002b; }

/* Thumbnail carousel active state */
.splide__track--nav > .splide__list > .splide__slide { border: 2px solid transparent; opacity: .6; }
.splide__track--nav > .splide__list > .splide__slide.is-active { border-color: #e4002b; opacity: 1; }

/* Progress bar */
.splide__progress { background: #eee; }
.splide__progress__bar { height: 4px; background: #e4002b; }
```

Scope overrides to one carousel with an ID or an extra class on the root (`#hero .splide__arrow`, `.my-carousel .splide__arrow`), since the theme selectors apply to every Splide on the page.

### 2. Start from the core CSS

Load `splide-core.min.css` and write the controls from scratch. Minimum to make the generated controls visible and usable:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide-core.min.css">
```

```css
/* Arrows: core has no arrow rules, so they would render as plain inline buttons */
.splide__arrow {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 2.5rem;
  height: 2.5rem;
  padding: 0;
  border: 0;
  border-radius: 50%;
  background: #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, .25);
  cursor: pointer;
}
.splide__arrow svg { width: 1rem; height: 1rem; fill: currentColor; }
.splide__arrow--prev { left: .5rem; }
.splide__arrow--prev svg { transform: scaleX(-1); } /* default SVG points right */
.splide__arrow--next { right: .5rem; }
.splide__arrow:disabled { opacity: .3; cursor: default; }

/* RTL / TTB, if you use them */
.splide__arrows--rtl .splide__arrow--prev { left: auto; right: .5rem; }
.splide__arrows--rtl .splide__arrow--prev svg { transform: none; }
.splide__arrows--rtl .splide__arrow--next { left: .5rem; right: auto; }
.splide__arrows--rtl .splide__arrow--next svg { transform: scaleX(-1); }

/* Pagination dots (core already centers them with flexbox) */
.splide__pagination { padding: .75rem 0 0; }
.splide__pagination__page {
  width: 8px;
  height: 8px;
  margin: 3px;
  padding: 0;
  border: 0;
  border-radius: 50%;
  background: #bbb;
  cursor: pointer;
}
.splide__pagination__page.is-active { background: #333; }

/* Progress bar: core sets only width: 0 */
.splide__progress__bar { height: 3px; background: #333; }

/* Focus: core has none */
.splide__arrow:focus-visible,
.splide__pagination__page:focus-visible,
.splide__toggle:focus-visible { outline: 3px solid #0bf; outline-offset: 3px; }
.splide.is-focus-in .splide__arrow:focus,
.splide.is-focus-in .splide__pagination__page:focus,
.splide.is-focus-in .splide__toggle:focus { outline: 3px solid #0bf; outline-offset: 3px; }
.splide__slide:focus-visible { outline: 3px solid #0bf; outline-offset: -3px; }
```

The docs' Accessibility page gives this pattern for focus styles without a theme. Its sample has a typo, `.splide_arrow` with one underscore. The class is `.splide__arrow`.

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

### 3. Add your own classes to generated elements

The `classes` option adds classes to elements Splide **creates** (generated arrows, pagination, clones, spinner). Splide's CSS uses the `splide__` classes, so keep them and append yours:

```js
new Splide( '.splide', {
  classes: {
    arrows    : 'splide__arrows your-class-arrows',
    arrow     : 'splide__arrow your-class-arrow',
    prev      : 'splide__arrow--prev your-class-prev',
    next      : 'splide__arrow--next your-class-next',
    pagination: 'splide__pagination your-class-pagination',
    page      : 'splide__pagination__page your-class-page',
  },
} ).mount();
```

`classes` keys and defaults (`CLASSES` in `constants/classes.ts`): `slide: 'splide__slide'`, `clone: 'splide__slide--clone'`, `arrows: 'splide__arrows'`, `arrow: 'splide__arrow'`, `prev: 'splide__arrow--prev'`, `next: 'splide__arrow--next'`, `pagination: 'splide__pagination'`, `page: 'splide__pagination__page'`, `spinner: 'splide__spinner'`. The object is deep-merged, so you can pass only the keys you change.

### 4. Numbered pagination with CSS only

```css
.splide__pagination {
  counter-reset: pagination-num;
}

.splide__pagination__page:before {
  counter-increment: pagination-num;
  content: counter( pagination-num );
}
```

With a theme, also enlarge the dots (for example `width: auto; height: auto; padding: .25em .5em; border-radius: 4px;`) so the numbers fit.

### 5. Hide or center controls when there aren't enough slides

```css
.splide:not(.is-overflow) .splide__arrows { display: none; }
.splide:not(.is-overflow) .splide__list { justify-content: center; }
/* pagination is already hidden by core: .splide:not(.is-overflow) .splide__pagination { display: none } */
```

To bring the pagination back while not overflowing, override with `.splide:not(.is-overflow) .splide__pagination { display: flex; }`.

### 6. Avoid the invisible-until-mounted flash

The root is `visibility: hidden` until `mount()`. Its box still takes up space, so the page doesn't jump, but it looks empty. Options:

- Reserve the final height by giving slide images a fixed height or `aspect-ratio` that matches the options you'll use.
- Show the unmounted markup. Core already lays the list out as a clipped flex row. Before mount, Splide hasn't set slide widths yet, so each slide is as wide as its content, and the layout shifts when it mounts.

  ```css
  .splide:not(.is-initialized) { visibility: visible; }
  ```

- Pre-render static HTML with `splide-renderer.min.js`. Its `is-rendered` class makes the root visible. See `getting-started.md`.

---

## Gotchas

- The pagination and arrows are absolutely positioned against the **root**. If you put a heading or other controls inside the root, arrows center on the whole root, heading included. Wrap the track (and an arrows placeholder) in your own `position: relative` element:

  ```html
  <section class="splide" aria-labelledby="h">
    <h2 id="h">Title</h2>
    <div style="position: relative">
      <div class="splide__arrows"></div>
      <div class="splide__track">...</div>
    </div>
  </section>
  ```

- `.splide__container` in core CSS is a dead selector. If you need `box-sizing` or `position: relative` on the slide container, style `.splide__slide__container` yourself.
- Splide writes inline styles: `max-width` on the root (`width`), `padding-left`/`-right` (or `-top`/`-bottom`) on the track (`padding`), `height` on the track (TTB), `width`, `margin-right`/`-left`/`-bottom` (`gap`) and `height` on slides or `.splide__slide__container`, `background` on slides or containers in `cover` mode, `transform` on the list, `width` on the progress bar, and `display: none` on images in `cover` mode, on lazy images without a `src` until they load, and on a hidden arrows wrapper or pagination placeholder. CSS without `!important` can't beat these inline styles. Use the options instead.
- The list and slides have `margin: 0 !important` / `padding: 0 !important` (list) and `list-style-type: none !important` (slide). Don't try to space slides with CSS margins. Use `gap`.
- `is-focus-in` lingers after focus leaves, so always combine it with `:focus`.
- Theme files style every `.splide` on the page. To give one carousel a different look, scope the overrides. Loading a second theme file overrides the first one globally.
- `.splide__arrow:disabled` relies on the native `disabled` attribute, so custom arrows must be `<button>` elements.
