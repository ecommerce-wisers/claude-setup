---
name: splide-vanilla-js
description: Build, configure or debug Splide v4 sliders/carousels in plain JavaScript loaded from the CDN with no npm, bundler or build step. Covers markup, options, breakpoints, events, API, components, official extensions, custom extensions/transitions and a11y. Use for static HTML, PHP, WordPress or Shopify themes.
---

# Splide v4 — Vanilla JS, CDN only

Splide **4.1.4** in plain JavaScript, loaded with `<script>` / `<link>` tags from jsDelivr, or from the copies bundled in `files/` (see [Bundled files](#bundled-files)). Never add `npm install`, bundler config, or bare `import … from '@splidejs/…'` specifiers (unless the page already has an import map; see `references/extensions/cdn.md`). Docs: https://splidejs.com/documents/

This file is the quick reference. The whole docs site (guides, every option, every event, all components, tutorials, extensions), checked against the Splide source, is in `references/`. Read the matching file before writing anything beyond the basics (see [Reference index](#reference-index)).

## Workflow

1. Load Splide from the pinned CDN. When the project forbids third-party CDNs or the page must work offline, copy the bundled `files/splide.min.js` and `files/splide.min.css` into the project and load them from there (see [Bundled files](#bundled-files)).
2. Write valid markup: root → track → list → slides. Never put extra elements between the track and the list.
3. Pick the CSS: a full theme (`splide.min.css`), or `splide-core.min.css` when the project styles its own arrows and pagination.
4. Set options. Use `breakpoints` for responsive behaviour (`mediaQuery: 'min'` = mobile-first).
5. Register `mounted` / `arrows:mounted` / `pagination:mounted` listeners **before** `mount()`.
6. Extensions: load each extension's CDN script after core, then pass it to `mount()`.
7. Run the [accessibility checklist](#accessibility-checklist).

## Load

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js" defer></script>
```

| File under `https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/` | Use |
|---|---|
| `css/splide.min.css` | Default theme (same as `css/themes/splide-default.min.css`) |
| `css/themes/splide-sea-green.min.css`, `css/themes/splide-skyblue.min.css` | Alternative themes |
| `css/splide-core.min.css` | Layout, pagination layout (hidden unless `is-overflow`), toggle-icon switching, spinner, `.splide__sr`. No arrow, dot or progress-bar visuals |
| `js/splide.min.js` (`js/splide.js` unminified) | Classic script → `window.Splide` (+ `Splide.defaults`, `Splide.STATES`) |
| `js/splide.esm.js` | Native ES module, self-contained. Also exports `EventInterface`, `EventBinder`, `RequestInterval`, `State`, `Throttle`, `DEFAULTS`, `CLASSES`, `EVENT_*`, `CLASS_*`, `SLIDE`/`LOOP`/`FADE`, `LTR`/`RTL`/`TTB` |

The classic script exposes **only** `window.Splide`. When you need `EventInterface` or the constants, use a module script. That still needs no bundler:

```html
<script type="module">
  import { Splide, EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  new Splide( '#hero' ).mount();
</script>
```

Module scripts are deferred. An inline module importing the full `https://` URL works even from a page opened on disk, but a module loaded from a local file (`<script type="module" src="./app.js">`) is blocked on `file://`, so serve over http(s) when you split code into files. Use one build per page; the two builds define separate `Splide` classes. Always pin the exact version; never use `@latest`. SRI hashes are in `references/getting-started.md`.

If the page has an AMD loader (RequireJS, e.g. Magento 2), the UMD build registers with `define()` and **doesn't** create `window.Splide`. Use the ESM build, or `require( [ 'splide' ], Splide => … )` with a path mapping.

### Bundled files

This skill ships the two production files from the 4.1.4 package. Paths are relative to this skill's directory.

| Bundled file | Same as CDN file | Bytes |
|---|---|---|
| `files/splide.min.js` | `dist/js/splide.min.js` (classic script → `window.Splide`) | 29,803 |
| `files/splide.min.css` | `dist/css/splide.min.css` (default theme) | 5,002 |

Use them when the project forbids third-party CDNs, the page must work offline, or there's no network to download from. **Copy them into the project's asset folder** and link the copies; never point a page at the skill directory. Don't read them into context: they're minified, and everything they do is documented in `references/`.

```html
<link rel="stylesheet" href="assets/splide.min.css">
<script src="assets/splide.min.js" defer></script>
```

- Shopify: copy both into the theme's `assets/`, then `{{ 'splide.min.css' | asset_url | stylesheet_tag }}` and `<script src="{{ 'splide.min.js' | asset_url }}" defer></script>`.
- WordPress: copy both into the theme, then `wp_enqueue_style( 'splide', get_theme_file_uri( 'assets/splide.min.css' ), [], '4.1.4' )` and `wp_enqueue_script( 'splide', get_theme_file_uri( 'assets/splide.min.js' ), [], '4.1.4', true )`. Make your init script depend on the `'splide'` handle.
- Not bundled: `splide-core.min.css`, the other themes, `splide.js`, `splide.esm.js` and every extension. Self-host those by downloading them from the CDN URLs above and in `references/extensions/cdn.md`.

## Markup

```html
<section class="splide" aria-label="Featured products">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">Slide 01</li>
      <li class="splide__slide">Slide 02</li>
    </ul>
  </div>
</section>
```

- Decorative carousel: `<div class="splide" role="group" aria-label="...">`.
- An optional `div.splide__slide__container` (direct child of the slide) receives `height`/`heightRatio`/`fixedHeight` instead of the slide.
- Arrows placeholder `<div class="splide__arrows"></div>` and pagination placeholder `<ul class="splide__pagination"></ul>` are siblings of the track, never inside it.
- Custom arrows: `div.splide__arrows > button.splide__arrow.splide__arrow--prev` / `--next` (must be `<button type="button">`; Splide doesn't add `type` to buttons you supply, and it overwrites their `aria-label` with the i18n strings). Splide finds HTML arrows and pagination by the **default** class names only; the `classes` option doesn't change the lookup.
- Autoplay UI: `div.splide__progress > div.splide__progress__bar`; `button.splide__toggle[type=button] > span.splide__toggle__play + span.splide__toggle__pause`.
- Per-slide data attributes: `data-splide-interval="3000"` (autoplay), `data-splide-lazy` / `data-splide-lazy-srcset` (on `<img>`).
- Options as JSON on the root: `data-splide='{"perPage":3}'`. Precedence: root `aria-label`/`aria-labelledby` < built-in defaults < `Splide.defaults` < constructor options < `data-splide` (objects merge deeply). Invalid or empty JSON throws `[splide] Invalid JSON`. `Splide.defaults` only affects instances created after it's set.
- State classes on the root: `is-initialized`, `is-overflow`, `is-focus-in`, and `splide--{type}`, `splide--{ltr|rtl|ttb}` (there's no `is-rtl` or `is-ttb`). On slides: `is-active`, `is-visible`, `is-prev`, `is-next`. Full list and theme selectors: `references/themes.md`.

Full markup, every data attribute and every ARIA attribute Splide adds: `references/structure.md`.

## Mount

```js
document.addEventListener( 'DOMContentLoaded', () => {
  document.querySelectorAll( '.splide' ).forEach( el => {
    new Splide( el, { perPage: 3, gap: '1rem' } ).mount();
  } );
} );
```

- Create one instance per carousel. A selector string only matches the first element.
- Nothing renders without `mount()`.
- With `defer` on the core script, keep the init inside `DOMContentLoaded` (deferred scripts run before it fires), or in a deferred file loaded after core.

## Options (most used)

Every option with type, default, notes and examples: `references/options.md`.

| Option | Default | Notes |
|---|---|---|
| `type` | `'slide'` | `'loop'`, `'fade'` (fade ignores `perPage`) |
| `perPage` / `perMove` | `1` / — | integers |
| `gap`, `padding` | — | CSS units; padding `{ left, right }` or `{ top, bottom }` |
| `focus` | — | number or `'center'` |
| `trimSpace` | `true` | `false` / `'move'` |
| `omitEnd` | `false` | v4.1+; needs `type: 'slide'` + `focus` |
| `start` | `0` | |
| `rewind`, `rewindByDrag` | `false` | not for loop |
| `speed` / `rewindSpeed` | `400` / `speed` | ms |
| `width`, `height`, `heightRatio` | — | height: no `%` |
| `fixedWidth`, `fixedHeight` | — | override `perPage` for layout only; `perPage` still sets the step and the page count |
| `autoWidth`, `autoHeight` | `false` | keep perPage/perMove = 1 |
| `direction` | `'ltr'` | `'rtl'`, `'ttb'` (ttb needs `height` or `heightRatio`) |
| `arrows`, `pagination` | `true` | pagination must be `false` with `isNavigation` |
| `paginationKeyboard` / `paginationDirection` | `true` / `direction` | |
| `arrowPath` | — | SVG path on 40×40 |
| `drag` | `true` | `'free'` + `snap: true` |
| `noDrag` | — | selector |
| `dragMinThreshold` | `10` | or `{ mouse, touch }` |
| `flickPower` / `flickMaxPages` | `600` / `1` | |
| `waitForTransition` | `false` | |
| `easing` | `cubic-bezier(0.25, 1, 0.5, 1)` | |
| `autoplay` | `false` | `'pause'` = wait for `Autoplay.play()` |
| `interval` | `5000` | |
| `pauseOnHover` / `pauseOnFocus` / `resetProgress` | `true` | |
| `lazyLoad` | `false` | `'nearby'` / `'sequential'`; `data-splide-lazy="src"` on img |
| `preloadPages` | `1` | nearby only |
| `keyboard` | `false` | `true` / `'focused'` / `'global'` |
| `wheel`, `releaseWheel` | `false` | + `wheelMinThreshold`, `wheelSleep` |
| `isNavigation` | `false` | thumbnails, with `sync()` |
| `slideFocus` | — | `true` if isNavigation |
| `focusableNodes` | `'a, button, textarea, input, select, iframe'` | |
| `cloneStatus` / `clones` | `true` / auto | clones only for loop |
| `cover` | `false` | legacy bg-image mode; needs a height option |
| `updateOnMove` | `false` | |
| `useScroll` | `false` | undocumented; moves with a JS (rAF) animation instead of the CSS transition; fires `scroll`/`scrolled`, ignores `easing` |
| `live` | `true` | ARIA live region |
| `mediaQuery` | `'max'` | `'min'` = mobile-first |
| `breakpoints` | — | `{ 768: { perPage: 1 } }`; merges like CSS media queries |
| `reducedMotion` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | |
| `destroy` | `false` | use inside breakpoints; `'completely'` |
| `classes` | — | keys `slide clone arrows arrow prev next pagination page spinner`; a value **replaces** that default, so include `splide__…` yourself |
| `i18n` | — | `references/i18n.md` |
| `label` / `labelledby` / `role` | — / — / `'region'` | |

## API

Full signatures, the `go()` control grammar and the state machine: `references/apis.md`.

```js
const splide = new Splide( '.splide', opts );
splide.on( 'mounted', fn );           // before mount
splide.mount( Extensions?, Transition? );

splide.go( 2 );      // slide index (number, not '2' — a numeric string does nothing)
splide.go( '+1' );   // '-1', '+2'… relative slides ('+' alone = '+1')
splide.go( '>' );    // next page; '<' prev page; '>2' = page index 2 (zero-based); '<N' ignores N
splide.index; splide.length; splide.root;
splide.options = { arrows: false };   // merges, emits 'updated'
splide.is( 'loop' );
splide.add( '<li class="splide__slide">New</li>', index? );
splide.remove( 0 | [0,1] | '.selector' | ( Slide, i ) => bool );
splide.refresh();
splide.destroy( true );               // true = completely
splide.state.is( Splide.STATES.IDLE );// CREATED MOUNTED IDLE MOVING SCROLLING DRAGGING DESTROYED
Splide.defaults = { ... };            // global defaults
main.sync( thumbs );                  // before mount
splide.on( 'move.myNs', fn ); splide.off( 'move.myNs' ); splide.emit( 'custom', a );
```

`off()` takes event names only, not a callback, and matches the namespace exactly. `off( 'move' )` removes only the handlers registered **without** a namespace, and `off( 'move.myNs' )` removes only that namespace. `destroy()` defaults to `completely = true`. `sync()` also works after mount (when both are mounted and idle), but the documented order is before mount.

## Events

Every event, including internal ones from the source, with argument types: `references/events.md`.

| Event | Args |
|---|---|
| `mounted`, `ready`, `refresh`, `resize`, `resized`, `destroy` | — |
| `move`, `moved` | `newIndex, prevIndex, destIndex` |
| `active`, `inactive`, `visible`, `hidden` | `Slide` |
| `click` | `Slide, MouseEvent` |
| `updated` | `options` |
| `drag`, `dragging`, `dragged`, `scroll`, `scrolled` | — |
| `overflow` | `isOverflow` |
| `arrows:mounted` | `prev, next` |
| `arrows:updated` | `prev, next, prevIndex, nextIndex` |
| `pagination:mounted` | `data { list, items[{ li, button, page }] }, item` |
| `pagination:updated` | `data, prev, curr` |
| `navigation:mounted` | `splides`: array of `{ splide, isParent? }`, not bare instances |
| `autoplay:play`, `autoplay:pause` | — |
| `autoplay:playing` | `rate` 0–1 |
| `lazyload:loaded` | `img, Slide` |

All handlers are removed on destroy. If `destroy` is used in breakpoints, re-bind via an extension, because its `mount()` runs again on remount.

## Components (`splide.Components.X`)

One file per component, with members that exist in the source but not in the docs: `references/components/`.

- **Controller**: `go(control, allowSameIndex?, callback?)`, `scroll(destPx, duration?, snap?, cb?)`, `getNext()`, `getPrev()` (→ `-1` if none), `getEnd()`, `getIndex(prev?)`, `setIndex(i)`, `toIndex(page)`, `toPage(index)`, `isBusy()` (only `true` while moving with `waitForTransition` set)
- **Slides**: `get(excludeClones?)`, `getAt(i)`, `getIn(page)`, `add()`, `remove()`, `forEach(fn, excludeClones?)`, `filter()`, `style(prop, val, useContainer?)`, `getLength(excludeClones?)`, `isEnough()`
- **Slide object**: `index`, `slideIndex` (-1 unless clone), `slide`, `container`, `isClone`, `style()`, `isWithin(from, distance)`
- **Elements**: `root`, `track`, `list`, `slides`, `arrows`, `prev`, `next`, `bar`, `toggle`
- **Move**: `move()`, `jump(i)`, `translate(pos)`, `cancel()`, `toIndex(pos)`, `toPosition(i, trim?)`, `getPosition()`, `getLimit(max)`, `exceededLimit()`
- **Layout**: `listSize()`, `slideSize(i?, noGap?)`, `sliderSize()`, `totalSize(i?, noGap?)`, `getPadding(right)`, `isOverflow()` (4.1+)
- **Autoplay**: `play()`, `pause()`, `isPaused()` · **Drag**: `disable(bool)`, `isDragging()` · **Arrows**: `arrows.{prev,next}` · **Pagination**: `items`, `getAt(i)` · **Direction**: `resolve(prop)`, `orient(v)` · **Scroll**: `scroll()`, `cancel()`
- Without doc pages (Clones, Cover, Keyboard, LazyLoad, Live, Media, Sync, Wheel): `references/components/other-components.md`.

## Recipes

Complete, copy-paste pages: `references/tutorials/` (image carousel, thumbnail carousel, gallery, progress bar).

**Thumbnails (synced):**
```js
const main = new Splide( '#main', { type: 'fade', rewind: true, pagination: false, arrows: false } );
const thumbs = new Splide( '#thumbs', {
  fixedWidth: 100, fixedHeight: 60, gap: 10, rewind: true,
  pagination: false, isNavigation: true,
  breakpoints: { 600: { fixedWidth: 60, fixedHeight: 44 } },
} );
main.sync( thumbs );
main.mount();
thumbs.mount();
```
```css
#thumbs .splide__slide { opacity: .6; }
#thumbs .splide__slide.is-active { opacity: 1; }
```

**Hide controls when there aren't enough slides** (details: `references/overflow.md`):
```css
.splide:not(.is-overflow) .splide__arrows,
.splide:not(.is-overflow) .splide__pagination { display: none; }
.splide:not(.is-overflow) .splide__list { justify-content: center; }
```
```js
splide.on( 'overflow', isOverflow => {
  splide.go( 0 );
  splide.options = { arrows: isOverflow, pagination: isOverflow, drag: isOverflow,
    clones: isOverflow ? undefined : 0 }; // clones only for loop
} );
```

**Numbered pagination:**
```js
splide.on( 'pagination:mounted', data => {
  data.items.forEach( item => { item.button.textContent = String( item.page + 1 ); } );
} );
```

**Progress bar:**
```js
splide.on( 'mounted move', () => {
  const end = splide.Components.Controller.getEnd() + 1;
  bar.style.width = `${ Math.min( ( splide.index + 1 ) / end, 1 ) * 100 }%`;
} );
```

**Custom buttons outside the root:** `prevBtn.addEventListener( 'click', () => splide.go( '<' ) )`, and set `arrows: false`.

**Images:** `.splide__slide img { width:100%; height:100%; object-fit:cover; }` with `heightRatio: 0.5` or `fixedHeight`.

**Mobile-only carousel:** `mediaQuery: 'min', breakpoints: { 1024: { destroy: true } }`.

**Shopify theme editor:** mount per section and re-init on `shopify:section:load`; call `destroy()` on `shopify:section:unload` (keep instances in a Map keyed by section id). Or wrap the carousel in a custom element and mount in `connectedCallback` / destroy in `disconnectedCallback`.

## Custom extension

Classic script, using only `window.Splide`:

```js
function SlideNumber( Splide, Components, options ) {
  let el;

  function mount() {
    el = document.createElement( 'div' );
    el.className = 'splide__counter';
    Components.Elements.track.after( el );
    update( Splide.index );
    Splide.on( 'move.slideNumber', update );   // removed automatically on destroy
  }

  function update( index ) {
    el.textContent = `${ index + 1 } / ${ Splide.length }`;
  }

  function destroy() {
    Splide.off( 'move.slideNumber' );
    el && el.remove();
  }

  return { mount, destroy };
}

new Splide( '#s' ).mount( { SlideNumber } );
```

With the ESM build you can use `EventInterface( Splide )` instead; its `on`/`bind` handlers are cleaned up on destroy. With the classic script, `splide.event.bind( target, 'click', fn )` does the same job; it's public in the typings and also cleaned up on destroy.

- Lifecycle: optional `setup()` (right after construction, before any `mount`), then `mount()`, then `destroy( completely )`. Components are destroyed in reverse order.
- An extension with a core component's name (e.g. `Keyboard`) **replaces** that built-in. A `Transition` key inside the extensions object is ignored; only `mount()`'s second argument sets the transition.
- A `destroy: true` breakpoint cycle permanently drops `splide.on` handlers registered outside extensions, and the index resets to `start` on remount. Re-bind and restore the index in an extension (`setup()`).
- Lifecycle details, the helper APIs (EventInterface, EventBinder, RequestInterval, State, Throttle) and tested examples in both ESM and classic form: `references/custom-extensions.md`.

Custom transition: same shape, but it must return `start( index, done )` and `cancel()`. Pass it as `mount( {}, MyTransition )`. Use `Components.Move.toPosition( index, true )` + `Move.translate( dest, true )` and call `done()` on `transitionend`. **Always call `done()`**, including when the distance is 0 (no `transitionend` fires), or the carousel stays `MOVING` and `moved` never fires. `cancel()` also runs on every pointerdown and before loop shifts. Built-in Slide/Fade can't be wrapped (neither build exports them). See `references/custom-transitions.md`.

## Official extensions

Load core first, then each extension's classic script. Each one registers itself on `window.splide.Extensions` (lowercase `splide`). Loading several merges them; nothing is overwritten. Mount selectively:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js"></script>
<script>
  const { AutoScroll } = window.splide.Extensions;
  new Splide( '#logos', {
    type: 'loop', drag: 'free', focus: 'center', perPage: 4,
    autoScroll: { speed: 1 },
  } ).mount( { AutoScroll } );
</script>
```

| Extension | Script: `https://cdn.jsdelivr.net/npm/` + | Global | Options / markup | Reference |
|---|---|---|---|---|
| Auto Scroll | `@splidejs/splide-extension-auto-scroll@0.5.3/dist/js/splide-extension-auto-scroll.min.js` | `AutoScroll` | `autoScroll: { speed:1, autoStart:true, rewind, pauseOnHover:true, pauseOnFocus:true, useToggleButton }`; `Components.AutoScroll.play/pause/isPaused` | `references/extensions/auto-scroll.md` |
| Intersection | `@splidejs/splide-extension-intersection@0.2.0/dist/js/splide-extension-intersection.min.js` | `Intersection` | `intersection: { inView:{autoplay,keyboard,autoScroll,video}, outView:{…}, root, rootMargin, threshold, once }`; events `intersection`, `intersection:in`, `intersection:out` | `references/extensions/intersection.md` |
| Video | `@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.min.js` **+ CSS** `@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css` | `Video` | `video: { autoplay, mute, loop, volume:0.2, hideControls, disableOverlayUI, host, playerOptions:{youtube,vimeo,htmlVideo} }`; slide attrs `data-splide-youtube` / `data-splide-vimeo` / `data-splide-html-video`; events `video:play/pause/ended/error/click` | `references/extensions/video.md` |
| Grid | `@splidejs/splide-extension-grid@0.4.1/dist/js/splide-extension-grid.min.js` | `Grid` | `grid: { rows, cols, dimensions:[[r,c]], gap:{row,col} }`; `grid: false` in breakpoints disables it | `references/extensions/grid.md` |
| URL Hash | `@splidejs/splide-extension-url-hash@0.3.0/dist/js/splide-extension-url-hash.min.js` | `URLHash` | slide attr `data-splide-hash="name"` | `references/extensions/url-hash.md` |

- Those are the latest versions; none has been released since 2022. Each also ships a self-contained `.esm.js` (zero imports, no import map needed). All URLs, a combined page and tested examples: `references/extensions/cdn.md`.
- Prefer `mount( { AutoScroll } )` over `mount( window.splide.Extensions )`. The latter turns on every loaded extension for that carousel: AutoScroll starts unless `autoScroll: false`, and URLHash clears an existing `#fragment`.
- **Never write a top-level `var splide = …`.** It overwrites the `window.splide` namespace, so `window.splide.Extensions` becomes `undefined`. Use `const`/`let`, or another name.
- Video: the YouTube IFrame API is fetched at mount from `//www.youtube.com/player_api` (so it fails on `file://`), and the Vimeo player is bundled. Docs-only `.splide--has-video` is never added. `volume` isn't applied to Vimeo.
- Intersection: the documented "autoplay only in view" pattern is `autoplay: 'pause', intersection: { inView: { autoplay: true }, outView: { autoplay: false } }`. But it calls `Autoplay.play()` whenever `autoplay` is truthy (including `'pause'`), which overrides the user's pause toggle and reduced motion. For WCAG-strict sites, use the plain `IntersectionObserver` recipe in `references/autoplay.md`. Its `keyboard` toggling does nothing unless `keyboard` is set.
- AMD pages (RequireJS/Magento): neither `window.Splide` nor `window.splide` is set. The workaround is in `references/extensions/cdn.md`.

## Accessibility checklist

Full ARIA map and what Splide does not handle: `references/accessibility.md`.

- Every carousel has `aria-label` or `aria-labelledby` (or the `label` option); use `role="group"` if it's decorative.
- Autoplay → include the `splide__toggle` play/pause button (WCAG 2.2.2). Don't rely on autoplay alone; it's paused under reduced motion.
- Visible focus styles: the themes include them; with core CSS use `.splide.is-focus-in .splide__arrow:focus`.
- Elements with `tabindex="0"`, `video[controls]` and `summary` inside slides aren't managed automatically. Extend `focusableNodes`, or toggle them on `visible`/`hidden`.
- Translate the i18n strings on non-English sites, and prefer content-specific labels. The `first` label goes on the *next* arrow and `last` on the *prev* arrow when they wrap (the docs have these swapped).
- `destroy` in a breakpoint leaves `tabindex="-1"` on links and buttons that were in hidden slides. Use the fix extension in `references/accessibility.md`.

## Pitfalls

- Listeners for `mounted`/`*:mounted` registered after `mount()` never fire.
- `sync()` must run before both `mount()` calls. Mount the **main** carousel before the `isNavigation` one, or the thumbnails get no `role="button"`, `aria-label` or `aria-controls`. The navigation carousel needs `pagination: false`.
- 4.1.4 bug: `refresh()`, `add()`, `remove()` or a `direction` breakpoint removes the thumbnails' a11y attributes. After `mount()`, register `thumbs.on( 'refresh', () => thumbs.emit( 'navigation:mounted', thumbs.splides ) )`.
- Don't name a custom progress bar `splide__progress__bar`; Splide takes it over as the autoplay timer bar.
- `autoWidth` + `lazyLoad` needs explicit slide widths.
- `rewind` does nothing in `loop`. Autoplay in a `slide` carousel without `rewind` stops at the last page.
- Autoplay toggle: `.splide__toggle__pause` shows while playing and `__play` while stopped (the docs say the opposite).
- The 4.1.x CSS already hides pagination when the slides don't overflow; arrows stay visible unless you hide them.
- `ttb` direction requires `height` or `heightRatio`. `fixedHeight` alone throws `height or heightRatio is missing.`
- v4 defaults: `keyboard: false`, `waitForTransition: false`. `arrows: 'slider'` was removed. v3 → v4 checklist: `references/version4.md`.
- Don't call `add()`/`remove()` while the carousel is moving.
- Custom arrow buttons must sit inside a `div.splide__arrows` wrapper. Bare `.splide__arrow` buttons make `mount()` throw.
- The root must keep the class `splide`, or the track/list lookup fails.
- `autoplay`, `wheel`, `cover`, `lazyLoad`, `live` (and Autoplay's `pauseOnHover`/`pauseOnFocus`) are read once at mount. Changing them through `splide.options` later does nothing, so remount instead.
- `noDrag` matches the event target itself, not its ancestors, so list inner elements too (`'.no-drag, .no-drag *'`).
- `destroy()` keeps `is-initialized` (so slides stack) and strips your own `role`/`aria-label`/`aria-labelledby` from the root.
- `shifted` and `slide:keydown` became internal in 4.1 (`'sh'`, `'sk'`), so listeners using the old names never fire.
- `EventInterface`, `EVENT_*` and `CLASS_*` are `undefined` with the classic script. Use string event names, or the ESM build.
- An inline init script that runs before a `defer`red core script throws `Splide is not defined`. Wrap it in `DOMContentLoaded`.

## Reference index

Read the file that matches the task. Each one is a complete, source-checked version of its docs page, converted to CDN/no-build form.

| File | Read when |
|---|---|
| `references/getting-started.md` | CDN tags, dist files, first page, multiple carousels, `data-splide`, `Splide.defaults` |
| `references/themes.md` | Choosing or overriding a theme; the theme's selectors, sizes and colors |
| `references/structure.md` | Full markup, custom element placement, data attributes, generated ARIA |
| `references/options.md` | Any option detail: types, defaults, breakpoints/mediaQuery merging, readonly options, `classes` keys |
| `references/arrows.md` | Custom or relocated arrows, arrow SVG, arrow events |
| `references/pagination.md` | Custom or numbered pagination, pagination ARIA, page count |
| `references/autoplay.md` | Autoplay, progress bar, play/pause toggle, per-slide intervals |
| `references/auto-width.md` | Variable-width or variable-height slides |
| `references/overflow.md` | Too few slides: hiding controls, centering, `is-overflow` |
| `references/accessibility.md` | ARIA roles, keyboard, focus, labels, WCAG |
| `references/i18n.md` | Translating labels, RTL, passing strings from a theme |
| `references/apis.md` | Instance properties and methods, `go()` grammar, states |
| `references/events.md` | Every event and its arguments, namespaces, timing |
| `references/custom-extensions.md` | Writing an extension; EventInterface / RequestInterval APIs |
| `references/custom-transitions.md` | Writing a transition; how Slide/Fade work; `useScroll` |
| `references/version4.md` | Migrating v3 code or reading old examples |
| `references/components/*.md` | `splide.Components.*` API (Arrows, Autoplay, Controller, Direction, Drag, Elements, Layout, Move, Pagination, Scroll, Slide, Slides) |
| `references/components/other-components.md` | Clones, Cover, Keyboard, LazyLoad, Live, Media, Sync, Wheel (from source) |
| `references/tutorials/*.md` | Complete pages: image carousel, thumbnail carousel, gallery, carousel progress |
| `references/extensions/*.md` | Auto Scroll, Intersection, Video, Grid, URL Hash; `cdn.md` lists every CDN URL |

Bundled assets to copy into a project, not to read: `files/splide.min.js` and `files/splide.min.css` (see [Bundled files](#bundled-files)).
