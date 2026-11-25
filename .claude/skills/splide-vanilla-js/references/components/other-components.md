# Other components (from source; not on the docs site)

Everything in this file is from source; none of these components has a page on the docs site.

Source files: `src/js/components/{Media,Clones,Cover,Keyboard,LazyLoad,Live,Sync,Wheel}/*.ts` and `src/js/transitions/{Slide,Fade}/*.ts`, Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Access: `splide.Components.<Name>` after `splide.mount()`.

Every built-in component is created inside `mount()`, in this order: Media, Direction, Elements, Slides, Layout, Clones, Move, Controller, Arrows, Autoplay, Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination, Sync, Wheel, Live. Extensions come next, then `Transition`. Each component's `setup()` runs right after it is created. Once all exist, every `mount()` runs in the same order, then Splide emits `mounted` and `ready`.

Many of these components read their options only once, when the slider mounts. The "Read when" column says when each option is read. Breakpoints that match at mount time do apply, because Media merges them first.

Contents: Media, Clones, Cover, Keyboard, LazyLoad, Live, Sync, Wheel, Transition (Slide / Fade).

---

## Media

Purpose: watches the `breakpoints` and `prefers-reduced-motion` media queries and merges the matching options into the current options. In v3 this was the Options component.

Access: `splide.Components.Media`

### Methods

| Signature | Returns | Description |
|---|---|---|
| `set( options, base?, notify? )` | `void` | Merges `options` into the current options. With `base`, also merges them into the base (prototype) options, which survive breakpoint changes. Emits `updated( options )` if `notify` is `true` or the slider is past the `CREATED` state. The interface does not mark it internal, but the implementation JSDoc does. |
| `reduce( reduced )` | `void` | (`@internal`) If `(prefers-reduced-motion: reduce)` matches: `true` merges `reducedMotion` options in, and `false` deletes those keys from the current options. Does nothing when the query does not match. The interface names the param `reduced`; the implementation names it `enable`. |

Also `setup()` and `destroy( completely )` (lifecycle).

`splide.options = { ... }` is the public way to call it: it calls `Media.set( options, true, true )`. Prefer it over calling `set()` directly.

```js
const splide = new Splide( '.splide', { perPage: 3 } ).mount();
splide.options = { perPage: 2, gap: '1rem' }; // merged into base options, emits 'updated'
```

### Behaviour

- `setup()` registers one `MediaQueryList` per breakpoint. With `mediaQuery: 'max'` (the default), the query is `(max-width:Npx)` and breakpoints are sorted large to small. With `'min'`, it is `(min-width:Npx)`, sorted small to large. `reducedMotion` is registered last with `(prefers-reduced-motion: reduce)`. `setup()` then runs `update()` once.
- `update()` runs on every query `change`:
  1. Deletes all current (own) option values, which falls back to the base options.
  2. Merges every matching breakpoint's options in order, so later entries win, then merges `reducedMotion` if it matches.
  3. Calls `set( merged )`, which emits `updated` unless this is the initial mount.
  4. If the merged options contain `destroy`, it calls `splide.destroy( destroy === 'completely' )`. Otherwise, if the slider was destroyed by an earlier breakpoint, it remounts it (`splide.mount()`). Otherwise, if `direction` changed, it calls `splide.refresh()`.
- `destroy: true` in a breakpoint destroys the slider while the query matches and remounts it when it stops matching. `destroy: 'completely'` destroys it and unbinds the media listeners, so it never comes back.

### Events emitted / listened

- Emits `updated( options )`, and indirectly `destroy` / `mounted` / `refresh` through the calls above.
- Listens to DOM `change` on each `MediaQueryList`.

### Options read

| Option | Read when | Use |
|---|---|---|
| `breakpoints` | Creation | `{ [px]: options }`. |
| `mediaQuery` | `setup()` | `'max'` (default) or `'min'`. |
| `reducedMotion` | Creation | Options applied under reduced motion (default `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }`). |
| `destroy` | Each update | `true` or `'completely'`, usually set inside a breakpoint. |
| `direction` | Each update | A change triggers `refresh()`. |

---

## Clones

Purpose: generates the clone slides that `type: 'loop'` needs.

Access: `splide.Components.Clones`

### Methods

The public interface is empty. The object has only `mount()` and `destroy()` (lifecycle).

### Behaviour

- Clones are generated only for `type: 'loop'`. For other types the count is 0.
- Clones per side:
  - `options.clones`, if set;
  - else, with `fixedWidth` (`fixedHeight` for `ttb`): `ceil( trackWidth / slideSize( 0 ) )`;
  - else, with `autoWidth` (`autoHeight` for `ttb`): `splide.length`;
  - else: `perPage * 2`.
- Generation: the last `count` Slide objects are cloned and inserted before the first slide, and the first `count` are cloned and appended to the list. If there are fewer slides than `count`, the list is repeated.
- Each clone is a `cloneNode( true )` deep copy. It gets the class `classes.clone` (`splide__slide--clone`) and the id `{rootId}-clone01`, `-clone02`, ... It is registered through `Slides.register( clone, index, originalIndex )`, with indices `-count .. -1` (head) and `length .. length + count - 1` (tail). Then `Layout.resize( true )` runs.
- Deep copies do not keep JavaScript listeners or state added to the original slide elements. Ids on elements inside a slide are duplicated (only the clone's own id is rewritten). Bind slide interactions through Splide events (`click`, `active`) or with event delegation.
- Clones are removed and regenerated on every `refresh`.

### Events emitted / listened

| Event | Direction | Action |
|---|---|---|
| `refresh` | Listened | Remove all clones and generate them again. |
| `updated`, `resize` | Listened | Recompute the needed count. If more clones are needed, or none any more, emit `refresh`. |
| `refresh` | Emitted | From the check above. |

### Options read

| Option | Read when | Use |
|---|---|---|
| `type` | Each count | Only `'loop'` gets clones. |
| `clones` | Each count | Explicit clones per side. |
| `fixedWidth` / `fixedHeight`, `autoWidth` / `autoHeight` | Each count | Automatic count (resolved by `direction`). |
| `perPage` | Each count | Default count `perPage * 2`. |
| `classes.clone` | Generation | Clone class. |

---

## Cover

Purpose: with `cover: true`, turns the slide image into a CSS background on its parent (the slide or its `.splide__slide__container`) and hides the `<img>`.

Access: `splide.Components.Cover`

### Methods

The public interface is empty. The object has `mount()` and `destroy()` (lifecycle). `destroy()` removes the backgrounds and shows the images again.

### Behaviour

- On `mounted`, `updated` and `refresh`, it goes through every Slide (clones included). It takes the first `<img>` that is a direct child of `Slide.container`, or of the slide if there is no container. If that image has a `src`, it sets `background: center/cover no-repeat url("<src>")` on the container (or slide) and `display: none` on the image.
- On `lazyload:loaded( img, Slide )`, it applies the same to that image once it has loaded.
- The slide needs a height from CSS or from the `height`, `fixedHeight` or `heightRatio` option (the docs for `cover` say so), because a background does not size the element.

```html
<li class="splide__slide">
  <div class="splide__slide__container">
    <img src="photo.jpg" alt="...">
  </div>
</li>
```

```js
new Splide( '.splide', { cover: true, heightRatio: 0.5 } ).mount();
```

### Events emitted / listened

- Emits nothing.
- Listens (only when `cover` is on at mount) to `mounted`, `updated`, `refresh` (apply to all) and `lazyload:loaded` (apply to one).

### Options read

| Option | Read when | Use |
|---|---|---|
| `cover` | `mount()` only | Enables the component. Turning it on later has no effect until a remount. |

---

## Keyboard

Purpose: arrow-key navigation for the slider.

Access: `splide.Components.Keyboard`

### Methods

| Signature | Returns | Description |
|---|---|---|
| `disable( disabled )` | `void` | `true` ignores key presses, `false` accepts them again. The implementation names the param `value`. The flag survives `updated` events, unlike Drag's. |

Also `mount()` and `destroy()` (lifecycle). `destroy` unbinds the key listener.

### Behaviour

- `keyboard: true` or `'focused'`: the `keydown` listener is on the root, so it fires only while focus is inside the slider. `'global'`: the listener is on `window`. `false` (the default; it is not in DEFAULTS): no listener.
- ArrowLeft calls `splide.go( '<' )` and ArrowRight calls `splide.go( '>' )`. The keys are resolved by direction: swapped for `rtl`, ArrowUp / ArrowDown for `ttb`. Legacy key names (`Left`, `Right`, `Up`, `Down`) are normalised.
- The default action is not prevented, so in `ttb` mode with `'global'`, ArrowUp/ArrowDown also scroll the page.
- On every `move`, keys are ignored until the next `setTimeout( 0 )` tick, to avoid double moves.

```js
const splide = new Splide( '.splide', { keyboard: 'global' } ).mount();
splide.Components.Keyboard.disable( true );  // e.g. while a modal is open
splide.Components.Keyboard.disable( false );
```

### Events emitted / listened

| Event | Direction | Action |
|---|---|---|
| `updated` | Listened | Unbind, then rebind from the new `keyboard` value. |
| `move` | Listened | Ignore keys until the next tick. |
| DOM `keydown` | Listened | On the root or window. |

Emits nothing. Movement happens through `splide.go()`.

### Options read

| Option | Read when | Use |
|---|---|---|
| `keyboard` | `mount()` and each `updated` | `true` / `'focused'` / `'global'` / `false`. |
| `direction` | Each key press | Key mapping. |

---

## LazyLoad

Purpose: lazy-loads slide images from `data-splide-lazy` / `data-splide-lazy-srcset` attributes.

Access: `splide.Components.LazyLoad`

### Methods

| Signature | Returns | Description |
|---|---|---|
| `check()` | `void` | (`@internal`) Nearby mode: starts loading every pending image whose slide is within range of `splide.index`. It also runs on `moved` and `scrolled`. |

Also `mount()` and `destroy()` (lifecycle).

### Markup

```html
<li class="splide__slide">
  <img data-splide-lazy="images/1.jpg" alt="...">
  <!-- or with srcset (both attributes may be combined) -->
  <img data-splide-lazy="images/2.jpg" data-splide-lazy-srcset="images/2-640.jpg 640w, images/2-1280.jpg 1280w" sizes="100vw" alt="...">
</li>
```

```js
new Splide( '.splide', { lazyLoad: 'nearby', preloadPages: 1 } ).mount();
```

### Behaviour

- Finds every element matching `[data-splide-lazy], [data-splide-lazy-srcset]` inside every Slide, clones included. It is meant for `<img>`: completion depends on the element firing `load` or `error`.
- An image is queued if its data values differ from its current `src`/`srcset`. For each queued image, a `<span class="splide__spinner">` is created in the image's parent (an existing one is reused), and the image is hidden (`display: none`) when it has no `src` yet.
- Loading an image: adds `is-loading` to the slide, copies the data attributes to `src`/`srcset`, then removes the data attributes.
- On `load`: removes `is-loading` and the spinner, shows the image, then emits `lazyload:loaded( img, Slide )` and `resize`.
- On `error`: removes `is-loading` only. The spinner stays and the image stays hidden if it had no `src`.
- `lazyLoad: true` or `'nearby'`: on mount, on each `refresh`, and on every `moved`/`scrolled`, loads the images of slides within `perPage * ( preloadPages + 1 ) - 1` slides of `splide.index` (`Slide.isWithin()`, wrapping for loop/rewind). It stops listening once everything has loaded.
- `lazyLoad: 'sequential'`: loads one image at a time in slide order. The next starts after the previous one loads or fails.

### Events emitted / listened

| Event | Direction | Arguments / action |
|---|---|---|
| `lazyload:loaded` | Emitted | `( img, Slide )` after an image loads successfully. |
| `resize` | Emitted | After each successful load, so the layout is re-measured. |
| `refresh` | Listened | Re-scan the slides and restart. |
| `moved`, `scrolled` | Listened | Nearby mode: `check()`. |
| DOM `load`, `error` | Listened | On each image being loaded. |

### Options read

| Option | Read when | Use |
|---|---|---|
| `lazyLoad` | Creation and `mount()` | `false` (default), `true` / `'nearby'`, `'sequential'`. |
| `preloadPages` | Each check | Pages to preload around the active one (default 1, nearby only). |
| `perPage` | Each check | Range calculation. |
| `classes.spinner` | Each scan | Spinner class (`splide__spinner`). |

---

## Live

Purpose: makes the track an ARIA live region, so screen readers announce slide changes.

Access: `splide.Components.Live`

### Methods

| Signature | Returns | Description |
|---|---|---|
| `disable( disabled )` | `void` | `true` sets `aria-live="off"` on the track, `false` sets `aria-live="polite"`. Does nothing when the live region is not enabled. |

Also `mount()` and `destroy()` (lifecycle). `destroy` removes `aria-live`, `aria-atomic` and `aria-busy`, and the SR span.

### Behaviour

- Enabled when `live` is truthy (default `true`) and `isNavigation` is not set. Both are read at creation.
- On mount: sets `aria-atomic="true"` on the track. Sets `aria-live="polite"`, or `"off"` if autoplay is already running.
- While autoplay plays, the live region is off (`autoplay:play` sets `off`, `autoplay:pause` sets `polite`), so screen reader users are not interrupted every interval.
- On every `moved`/`scrolled`: sets `aria-busy="true"` and appends `<span class="splide__sr">…</span>` to the track. After 90 ms it removes the span and sets `aria-busy="false"`. This makes Windows Narrator and JAWS re-read the region.

### Events emitted / listened

- Emits nothing.
- Listens to `autoplay:play`, `autoplay:pause`, `moved` and `scrolled`, only when enabled.

### Options read

| Option | Read when | Use |
|---|---|---|
| `live` | Creation | Enable the live region (default `true`). |
| `isNavigation` | Creation | A navigation slider never gets a live region. |

---

## Sync

Purpose: syncs sliders registered with `splide.sync()`, and implements `isNavigation` (clickable, keyboard-activatable thumbnail slides).

Access: `splide.Components.Sync`

### Methods

| Signature | Returns | Description |
|---|---|---|
| `remount()` | `void` | Removes all Sync handlers and binds them again for the current `splide.splides`. The interface does not mark it internal; the implementation JSDoc does. `splide.sync()` calls it when the slider is already mounted. |

Also `setup()`, `mount()` and `destroy()` (lifecycle).

### Behaviour

- `setup()`: if `slideFocus` is undefined, it is set in the base options to the value of `isNavigation`, through `Media.set( { slideFocus }, true )`.
- `mount()`: for every slider in `splide.splides` that is not a parent, it adds `move` handlers on both sliders. When one moves, it calls `other.go( other.is( 'loop' ) ? dest : index )`, so the sync works in both directions.
- With `isNavigation: true`:
  - A slide `click` calls `splide.go( Slide.index )`.
  - Enter or Space on a slide calls `splide.go( Slide.index )` and prevents the default action.
  - On `mounted`/`updated`, the list gets `aria-orientation="vertical"` for `ttb`.
  - It emits `navigation:mounted( splide.splides )`. The Slide objects of every slider react by setting their navigation ARIA attributes. The argument is the internal array of `{ splide, isParent? }` objects, although the TypeScript `EventMap` declares it as `Splide[]`.
- `splide.sync( other )` pushes `{ splide: other }` onto `splide.splides` and `{ splide: this, isParent: true }` onto `other.splides`. Call it before `mount()` on both. If the slider is already mounted, `sync()` remounts Sync on both.

```js
const main   = new Splide( '#main', { type: 'fade', rewind: true, pagination: false, arrows: false } );
const thumbs = new Splide( '#thumbs', {
  fixedWidth  : 100,
  gap         : 10,
  rewind      : true,
  pagination  : false,
  isNavigation: true,
} );

main.sync( thumbs );
main.mount();
thumbs.mount();
```

### Events emitted / listened

| Event | Direction | Action |
|---|---|---|
| `navigation:mounted` | Emitted | `( splides )` when `isNavigation`. |
| `move` | Listened (both sliders) | Move the other slider. |
| `click`, `sk` (internal slide keydown) | Listened | Navigation clicks and keys. |
| `mounted`, `updated` | Listened | Navigation `aria-orientation`. |

### Options read

| Option | Read when | Use |
|---|---|---|
| `isNavigation` | Creation | Enable navigation behaviour. |
| `slideFocus` | Creation | Defaulted to `isNavigation` when undefined. |
| `direction` | `mounted`/`updated` | `aria-orientation` on the list. |
| `type` (of the other slider) | Each move | Loop sliders receive `dest` (clone-aware), others receive `index`. |

---

## Wheel

Purpose: moves the slider with the mouse wheel.

Access: `splide.Components.Wheel`

### Methods

The public interface is empty. The object has only `mount()` (lifecycle).

### Behaviour

- Only when `wheel` is truthy at mount. It binds `wheel` on the track with `{ passive: false, capture: true }`. Changing `wheel` after mount has no effect, because nothing listens to `updated`.
- Only `deltaY` is used; horizontal trackpad swipes (`deltaX`) are ignored. If `abs( deltaY ) > wheelMinThreshold` (default 0) and more than `wheelSleep` ms (default 0) have passed since the last accepted wheel event, it calls `splide.go( deltaY < 0 ? '<' : '>' )`.
- The page scroll is prevented, except when `releaseWheel` is set, the slider is not moving, and there is no slide in that direction (`Controller.getAdjacent( backwards ) === -1`). The page then scrolls on at the first or last slide.
- Only cancelable wheel events are handled.

```js
new Splide( '.splide', {
  direction        : 'ttb',
  height           : '30rem',
  wheel            : true,
  wheelMinThreshold: 20,
  wheelSleep       : 300,
  releaseWheel     : true,
} ).mount();
```

### Events emitted / listened

- Emits nothing. Movement happens through `splide.go()`.
- Listens to DOM `wheel` on the track.

### Options read

| Option | Read when | Use |
|---|---|---|
| `wheel` | `mount()` only | Enable. |
| `wheelMinThreshold` | Each wheel event | Ignore small deltas (inertia). |
| `wheelSleep` | Each wheel event | Minimum ms between accepted wheel events. |
| `releaseWheel` | Each wheel event | Let the page scroll at the ends. |

---

## Transition (Slide / Fade)

Purpose: performs the actual animation for `Move.move()`. It lives in `src/js/transitions/`, not `src/js/components/`, but it is registered as `splide.Components.Transition`. The default is the Slide transition, or Fade when `type: 'fade'`. A custom one can be passed as the second argument of `splide.mount( Extensions, Transition )`.

Access: `splide.Components.Transition`

### Methods (`TransitionComponent`)

| Signature | Returns | Description |
|---|---|---|
| `start( index, done )` | `void` | Animates to `index` and must call `done()` when finished. Move calls it, after emitting `move`. |
| `cancel()` | `void` | Stops the animation. Move calls it from `Move.cancel()`. |

Also `mount()` (lifecycle).

### Slide transition (`src/js/transitions/Slide/Slide.ts`)

- `start()`: the target is `Move.toPosition( index, true )`. The speed is `rewindSpeed` when the type is `slide`, `rewindSpeed` is set, and the move goes from index 0 to the end or from the end to 0; otherwise it is `speed`.
  - If the distance is at least 1px and the speed at least 1 ms:
    - With `useScroll`, it calls `Scroll.scroll( target, speed, false, done )`.
    - Otherwise it sets `transition: transform <speed>ms <easing>` on the list, translates the list, and calls `done` on the list's `transitionend`.
  - Otherwise it jumps (`Move.jump( index )`) and calls `done()` synchronously.
- `cancel()`: clears the list's `transition` and cancels any Scroll.

### Fade transition (`src/js/transitions/Fade/Fade.ts`)

- On `mounted` and `refresh`, each slide gets `transform: translateX(-<100 * index>%)` so all slides stack in one place.
- `start()`: sets `transition: opacity <speed>ms <easing>` on all slides, then calls `done` on the next tick. The fade itself comes from the core CSS: `.splide__track--fade > .splide__list > .splide__slide { opacity: 0 }`, and `.is-active { opacity: 1 }`.
- `cancel()`: no-op.

### Options read

| Option | Read when | Use |
|---|---|---|
| `speed` | Each `start()` | Duration in ms (`0` or under 1 means jump). |
| `rewindSpeed` | Each `start()` | Slide type rewind duration. |
| `easing` | Each `start()` | CSS timing function (default `'cubic-bezier(0.25, 1, 0.5, 1)'`). |
| `useScroll` | Each `start()` | Animate with the Scroll component instead of CSS. |
| `type` | Creation | Chooses Slide or Fade. |
