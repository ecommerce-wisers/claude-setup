# Other components (React; from source, not on the docs site)

The core components without a page on the docs site (Media, Clones, Cover, Keyboard, LazyLoad, Live, Sync, Wheel, and the Transition), as reached from React Splide.

Contents: [Access and lifecycle](#access-and-lifecycle) · [Media](#media) · [Clones](#clones) · [Cover](#cover) · [Keyboard](#keyboard) · [LazyLoad](#lazyload) · [Live](#live) · [Sync](#sync) · [Wheel](#wheel) · [Transition (Slide / Fade)](#transition-slide--fade)

Everything in this file is from source; none of these components has a page on the docs site. React behaviour marked "tested" was run in headless Chrome with React 19, in production and under StrictMode.

Source files: `src/js/components/{Media,Clones,Cover,Keyboard,LazyLoad,Live,Sync,Wheel}/*.ts` and `src/js/transitions/{Slide,Fade}/*.ts` in Splide core. React Splide 0.7.12 bundles its own copy of core 4.1.3, which behaves the same as 4.1.4.

## Access and lifecycle

```tsx
const ref = useRef<Splide>( null );   // Splide is the React component from '@splidejs/react-splide'
// <Splide ref={ ref } ...>
ref.current?.splide?.Components.Keyboard;
```

- In an event prop, the core instance is the first argument: `onMoved={ ( splide ) => splide.Components.LazyLoad.check() }`.
- In an extension (the `extensions` prop), use the `Components` argument (see `references/custom-extensions.md`).
- `undefined` during the first render: `ref.current` is `null` and the instance does not exist yet. Read it in `useEffect`, an event handler or an event prop, never during render.
- Ready in the parent's `useEffect`: `<Splide>` creates and mounts the instance in its own `componentDidMount`, which runs first.
- Gone after unmount: `componentWillUnmount` calls `splide.destroy()` (completely) and clears `ref.current.splide`.
- With a `destroy: true` breakpoint, `ref.current.splide` still exists but `splide.state.is( 7 )` (`DESTROYED`) is `true` and `splide.Components` holds destroyed objects. A remount replaces them with new objects, so read `splide.Components.X` at call time instead of storing it. Event props and `splide.on()` listeners do not come back after that remount (tested, see [Media](#media)).
- If TypeScript reports implicit `any` for imports from `@splidejs/react-splide`, the types need one tsconfig line: see `references/typescript.md`.

Every built-in component is created inside `mount()` (in React, inside `<Splide>`'s `componentDidMount`), in this order: Media, Direction, Elements, Slides, Layout, Clones, Move, Controller, Arrows, Autoplay, Cover, Scroll, Drag, Keyboard, LazyLoad, Pagination, Sync, Wheel, Live. Extensions from the `extensions` prop come next, then `Transition` (the `transition` prop, or the default). Each component's `setup()` runs right after it is created. Once all exist, every `mount()` runs in the same order, then Splide emits `mounted` and `ready` (`onMounted`, `onReady`).

React Splide binds the event props before this, so every prop handler is registered before any component's own listener. For the same event, a prop handler therefore runs first, before the components have reacted (tested with `onRefresh` and `onUpdated`). A listener added with `splide.on()` in an effect runs after them.

Many of these components read their options only once, when the slider mounts. The "Read when" column says when each option is read. Breakpoints that match at mount time do apply, because Media merges them first. In React, "mount" means the first mount of `<Splide>`: a later `options` prop change does not re-read these options. Use a `key` on `<Splide>` to remount when one of them must change.

---

## Media

Purpose: watches the `breakpoints` and `prefers-reduced-motion` media queries and merges the matching options into the current options. In v3 this was the Options component.

Access: `ref.current?.splide?.Components.Media` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Media` inside an event prop, where the core instance is the first argument.

### Methods

| Signature | Returns | Description |
|---|---|---|
| `set( options, base?, notify? )` | `void` | Merges `options` into the current options. With `base`, also merges them into the base (prototype) options, which survive breakpoint changes. Emits `updated( options )` if `notify` is `true` or the slider is past the `CREATED` state. The interface does not mark it internal, but the implementation JSDoc does. |
| `reduce( reduced )` | `void` | (`@internal`) If `(prefers-reduced-motion: reduce)` matches: `true` merges `reducedMotion` options in, and `false` deletes those keys from the current options. Does nothing when the query does not match. The interface names the param `reduced`; the implementation names it `enable`. |

Also `setup()` and `destroy( completely )` (lifecycle).

`splide.options = { ... }` calls `Media.set( options, true, true )`, and that is what React Splide does when the `options` prop changes (deep compare against the previous prop). In React, change options through the prop; don't call `set()` or assign `splide.options` yourself, because React Splide does not see those changes and the next prop change merges over them.

### Behaviour

- `setup()` registers one `MediaQueryList` per breakpoint. With `mediaQuery: 'max'` (the default), the query is `(max-width:Npx)` and breakpoints are sorted large to small. With `'min'`, it is `(min-width:Npx)`, sorted small to large. `reducedMotion` is registered last with `(prefers-reduced-motion: reduce)`. `setup()` then runs `update()` once.
- `update()` runs on every query `change`:
  1. Deletes all current (own) option values, which falls back to the base options.
  2. Merges every matching breakpoint's options in order, so later entries win, then merges `reducedMotion` if it matches.
  3. Calls `set( merged )`, which emits `updated` unless this is the initial mount.
  4. If the merged options contain `destroy`, it calls `splide.destroy( destroy === 'completely' )`. Otherwise, if the slider was destroyed by an earlier breakpoint, it remounts it (`splide.mount()`). Otherwise, if `direction` changed, it calls `splide.refresh()`.
- `destroy: true` in a breakpoint destroys the slider while the query matches and remounts it when it stops matching. `destroy: 'completely'` destroys it and unbinds the media listeners, so it never comes back.
- A change of the `options` prop is merged into the current options on top of the active breakpoint values, and stays until the next media query change. Tested: with `breakpoints: { 1400: { perPage: 2 } }` at 1280px, changing the prop's `perPage` from 3 to 4 made `perPage` 4 at once; after widening to 1500px it stayed 4, and back at 1280px the breakpoint's 2 applied again.

What a `destroy: true` breakpoint means in React (tested):

- `<Splide>` stays mounted and `ref.current.splide` stays defined. Check `splide.state.is( 7 )` (`DESTROYED`) before using components. `splide.Components` keeps all 20 entries, as destroyed objects.
- When the breakpoint stops matching, Splide mounts again with new component objects, and the index starts over at `start`.
- Every handler added with `splide.on()` is removed by the destroy and not added back by the remount. That includes React Splide's event props: after a destroy breakpoint cycle, `onMoved` no longer fired, nor did a `splide.on( 'moved' )` listener from an effect. Listeners registered inside an extension (in its `setup()`) survive, because extensions are created again on every mount (tested). Otherwise remount `<Splide>` with a `key` when the breakpoint changes, or avoid `destroy: true`.

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const DESTROYED = 7; // Splide.STATES.DESTROYED

export default function MediaExample() {
  const ref = useRef<Splide>( null );
  const [ perPage, setPerPage ] = useState( 3 );
  const [ status, setStatus ] = useState( '' );

  const inspect = () => {
    const splide = ref.current?.splide;

    if ( ! splide ) {
      setStatus( 'not mounted' );
    } else if ( splide.state.is( DESTROYED ) ) {
      setStatus( 'destroyed by a breakpoint' ); // splide.Components still holds the destroyed components
    } else {
      setStatus( `perPage ${ splide.options.perPage }` );
    }
  };

  return (
    <>
      <Splide
        ref={ ref }
        options={ {
          perPage,
          label      : 'Media demo',
          breakpoints: {
            1400: { perPage: 2 },
            300 : { destroy: true },
          },
        } }
      >
        { [ 1, 2, 3, 4 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <button type="button" id="per-page" onClick={ () => setPerPage( 4 ) }>Base perPage 4</button>
      <button type="button" id="inspect" onClick={ inspect }>Inspect</button>
      <p id="status">{ status }</p>
    </>
  );
}
```

Tested at 1280px (`perPage 2`), after the prop change (`perPage 4`), at 1500px (`perPage 4`), at 280px (`destroyed by a breakpoint`), and back at 1280px (`perPage 2`).

### Events emitted / listened

- Emits `updated( options )` (`onUpdated`), and indirectly `destroy` / `mounted` / `refresh` (`onDestroy`, `onMounted`, `onRefresh`) through the calls above. After the first destroy, the event props no longer fire (see above).
- Listens to DOM `change` on each `MediaQueryList`.

### Options read

| Option | Read when | Use |
|---|---|---|
| `breakpoints` | Creation | `{ [px]: options }`. Changing `breakpoints` through the `options` prop later merges the new object into the options, but the media queries registered at creation stay as they were (from source). |
| `mediaQuery` | `setup()` | `'max'` (default) or `'min'`. |
| `reducedMotion` | Creation | Options applied under reduced motion (default `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }`). |
| `destroy` | Each update | `true` or `'completely'`, usually set inside a breakpoint. |
| `direction` | Each update | A change triggers `refresh()`. |

---

## Clones

Purpose: generates the clone slides that `type: 'loop'` needs.

Access: `ref.current?.splide?.Components.Clones` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Clones` inside an event prop, where the core instance is the first argument.

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

In React this matters more, because clones are DOM nodes that React never rendered:

- React event handlers inside a slide (`onClick` on a button, and so on) do not fire on clones. Tested: clicking a button inside a clone ran the `<Splide>` `onClick` prop but not the button's React `onClick`. Use the `onClick` prop and map `Slide.slideIndex` back to your data, as below.
- Component state inside a slide (a counter, an open accordion, a playing video) is not copied to its clones, and later React updates to the real slide do not reach them. Keep slides stateless, or refresh after the content changes (`ref.current?.splide?.refresh()` regenerates the clones from the current DOM).
- Portals, refs and effects inside a slide exist once, for the real slide only.

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const products = [ 'Apple', 'Banana', 'Cherry' ];

export default function ClonesExample() {
  const [ picked, setPicked ] = useState( '' );

  return (
    <>
      <Splide
        options={ { type: 'loop', perPage: 2, label: 'Products' } }
        // The core 'click' event fires for clones too. React handlers inside a slide do not,
        // because clones are DOM copies that React never rendered.
        onClick={ ( splide, Slide ) => {
          const index = Slide.isClone ? Slide.slideIndex : Slide.index;
          setPicked( products[ index ] );
        } }
      >
        { products.map( name => (
          <SplideSlide key={ name }>
            <button type="button" className="pick">{ name }</button>
          </SplideSlide>
        ) ) }
      </Splide>
      <p id="picked">{ picked }</p>
    </>
  );
}
```

Tested: 3 slides with `perPage: 2` get 8 clones (`perPage * 2` per side, the list repeated to reach 4), ids `splide01-clone01`, ...; clicking `Banana` in a clone sets `picked` to `Banana`.

### Events emitted / listened

| Event | Direction | Action |
|---|---|---|
| `refresh` | Listened | Remove all clones and generate them again. |
| `updated`, `resize` | Listened | Recompute the needed count. If more clones are needed, or none any more, emit `refresh`. |
| `refresh` | Emitted | From the check above (`onRefresh`). |

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

Access: `ref.current?.splide?.Components.Cover` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Cover` inside an event prop, where the core instance is the first argument.

### Methods

The public interface is empty. The object has `mount()` and `destroy()` (lifecycle). `destroy()` removes the backgrounds and shows the images again.

### Behaviour

- On `mounted`, `updated` and `refresh`, it goes through every Slide (clones included). It takes the first `<img>` that is a direct child of `Slide.container`, or of the slide if there is no container. If that image has a `src`, it sets `background: center/cover no-repeat url("<src>")` on the container (or slide) and `display: none` on the image.
- On `lazyload:loaded( img, Slide )`, it applies the same to that image once it has loaded.
- The slide needs a height from CSS or from the `height`, `fixedHeight` or `heightRatio` option (the docs for `cover` say so), because a background does not size the element.
- The background and `display: none` are inline styles Splide writes. If you give the `<img>` a `style` prop that sets `display`, or the slide a `style` prop that sets `background`, React overwrites Splide's value whenever that prop value changes.
- An image whose `src` changes through React after mount keeps the old background until the next `updated` or `refresh`.

The example in [LazyLoad](#lazyload) combines `cover` with lazy loading.

### Events emitted / listened

- Emits nothing.
- Listens (only when `cover` is on at mount) to `mounted`, `updated`, `refresh` (apply to all) and `lazyload:loaded` (apply to one).

### Options read

| Option | Read when | Use |
|---|---|---|
| `cover` | `mount()` only | Enables the component. Turning it on later through the `options` prop has no effect until the instance mounts again (a new `key`). |

---

## Keyboard

Purpose: arrow-key navigation for the slider.

Access: `ref.current?.splide?.Components.Keyboard` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Keyboard` inside an event prop, where the core instance is the first argument.

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
- With `'global'`, every carousel on the page that has it reacts to the arrow keys, including ones in a hidden tab or a closed modal. Disable the ones that are not in use, as below.

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

export default function KeyboardExample() {
  const ref = useRef<Splide>( null );
  const [ modalOpen, setModalOpen ] = useState( false );

  // Keyboard.disable() survives 'updated' events, so an effect on the flag is enough.
  useEffect( () => {
    ref.current?.splide?.Components.Keyboard.disable( modalOpen );
  }, [ modalOpen ] );

  return (
    <>
      <Splide
        ref={ ref }
        options={ { keyboard: 'global', wheel: true, releaseWheel: true, speed: 100, label: 'Keyboard demo' } }
      >
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <button type="button" id="modal" onClick={ () => setModalOpen( o => ! o ) }>
        { modalOpen ? 'Close modal' : 'Open modal' }
      </button>
    </>
  );
}
```

Tested: ArrowRight moves to slide 2, does nothing while the modal flag is set, and moves to slide 3 after it is cleared. The track also has `aria-live="polite"` and `aria-atomic="true"` from the Live component. Because `disable()` survives `updated` events, an effect on the flag is enough here (unlike Drag, see `references/components/drag.md`).

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
| `keyboard` | `mount()` and each `updated` | `true` / `'focused'` / `'global'` / `false`. Can change through the `options` prop. |
| `direction` | Each key press | Key mapping. |

---

## LazyLoad

Purpose: lazy-loads slide images from `data-splide-lazy` / `data-splide-lazy-srcset` attributes.

Access: `ref.current?.splide?.Components.LazyLoad` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.LazyLoad` inside an event prop, where the core instance is the first argument.

### Methods

| Signature | Returns | Description |
|---|---|---|
| `check()` | `void` | (`@internal`) Nearby mode: starts loading every pending image whose slide is within range of `splide.index`. It also runs on `moved` and `scrolled`. |

Also `mount()` and `destroy()` (lifecycle).

### Markup

`<SplideSlide>` passes attributes through, and React writes `data-*` props as attributes:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const images = [ '/images/1.jpg', '/images/2.jpg', '/images/3.jpg' ];

export default function LazyCoverExample() {
  const [ loaded, setLoaded ] = useState( 0 );

  return (
    <>
      <Splide
        options={ { cover: true, heightRatio: 0.5, lazyLoad: 'nearby', label: 'Photos' } }
        onLazyLoadLoaded={ () => setLoaded( n => n + 1 ) }
      >
        { images.map( ( src, i ) => (
          <SplideSlide key={ src }>
            <img data-splide-lazy={ src } alt={ `Photo ${ i + 1 }` } />
          </SplideSlide>
        ) ) }
      </Splide>
      <p id="loaded">{ loaded }</p>
    </>
  );
}
```

Tested in production: the two slides within range load, `onLazyLoadLoaded` fires twice, Cover turns them into backgrounds and hides the `<img>`, and slide 3 keeps its `data-splide-lazy` until it comes near. With srcset, add `data-splide-lazy-srcset="..."` (both attributes may be combined) and a `sizes` prop.

StrictMode (development only, tested): the first mount starts loading and removes the `data-splide-lazy` attributes from the images in range, then is destroyed before they finish. The second mount finds nothing to load for them, so `onLazyLoadLoaded` does not fire and their `.splide__spinner` elements stay in the DOM. Production is unaffected. Don't rely on `onLazyLoadLoaded` while developing under StrictMode.

### Behaviour

- Finds every element matching `[data-splide-lazy], [data-splide-lazy-srcset]` inside every Slide, clones included. It is meant for `<img>`: completion depends on the element firing `load` or `error`.
- An image is queued if its data values differ from its current `src`/`srcset`. For each queued image, a `<span class="splide__spinner">` is created in the image's parent (an existing one is reused), and the image is hidden (`display: none`) when it has no `src` yet.
- Loading an image: adds `is-loading` to the slide, copies the data attributes to `src`/`srcset`, then removes the data attributes. React does not put them back on later renders, because the prop values did not change.
- On `load`: removes `is-loading` and the spinner, shows the image, then emits `lazyload:loaded( img, Slide )` (`onLazyLoadLoaded`, called as `( splide, img, Slide )`) and `resize`.
- On `error`: removes `is-loading` only. The spinner stays and the image stays hidden if it had no `src`.
- `lazyLoad: true` or `'nearby'`: on mount, on each `refresh`, and on every `moved`/`scrolled`, loads the images of slides within `perPage * ( preloadPages + 1 ) - 1` slides of `splide.index` (`Slide.isWithin()`, wrapping for loop/rewind). It stops listening once everything has loaded.
- `lazyLoad: 'sequential'`: loads one image at a time in slide order. The next starts after the previous one loads or fails.
- A new slide added through state is scanned on the `refresh` that React Splide triggers. A changed `data-splide-lazy` prop on an existing image is only picked up at the next `refresh` (from source).

### Events emitted / listened

| Event | Direction | Arguments / action |
|---|---|---|
| `lazyload:loaded` | Emitted | `( img, Slide )` after an image loads successfully. React prop `onLazyLoadLoaded` (capital L in Load). |
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

Access: `ref.current?.splide?.Components.Live` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Live` inside an event prop, where the core instance is the first argument.

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
- The SR span is appended to the track element `<SplideTrack>` renders, outside the `<ul>` React manages, so it does not interfere with React's children.

```tsx
ref.current?.splide?.Components.Live.disable( true ); // silence announcements, e.g. while your own live region speaks
```

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

Access: `ref.current?.splide?.Components.Sync` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Sync` inside an event prop, where the core instance is the first argument.

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
  - It emits `navigation:mounted( splide.splides )` (`onNavigationMounted`). The Slide objects of every slider react by setting their navigation ARIA attributes. The argument is the internal array of `{ splide, isParent? }` objects, although the TypeScript `EventMap` declares it as `Splide[]`.
- `splide.sync( other )` pushes `{ splide: other }` onto `splide.splides` and `{ splide: this, isParent: true }` onto `other.splides`. In vanilla JS you call it before `mount()` on both. In React both instances are created and mounted inside their own `componentDidMount`, so sync them afterwards, in the parent's `useEffect`. Because both are already mounted, `sync()` remounts Sync on both, which emits `navigation:mounted` and applies the navigation attributes.
- `sync()` needs the other slider mounted too: on a mounted slider, it calls `other.Components.Sync.remount()`, which throws if the other instance has not mounted (from source). A `useEffect` in the common parent is safe, because both children mount before it runs.

The React component has `sync( splide )`, which calls `this.splide.sync( splide )`:

```tsx
import { useEffect, useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

const mainOptions: Options = { type: 'fade', rewind: true, pagination: false, arrows: false, label: 'Main' };
const thumbsOptions: Options = {
  fixedWidth  : 100,
  gap         : 10,
  rewind      : true,
  pagination  : false,
  isNavigation: true,
  label       : 'Thumbnails',
};

export default function SyncExample() {
  const mainRef = useRef<Splide>( null );
  const thumbsRef = useRef<Splide>( null );
  const [ slides, setSlides ] = useState( [ 1, 2, 3, 4 ] );

  useEffect( () => {
    const thumbs = thumbsRef.current?.splide;

    if ( mainRef.current && thumbs ) {
      // Both are mounted here, so sync() remounts the Sync component on both.
      mainRef.current.sync( thumbs );

      // A refresh (new slide children) rebuilds the Slide objects without the navigation
      // attributes. Apply them again; this listener runs after the rebuild.
      thumbs.on( 'refresh', () => thumbs.emit( 'navigation:mounted', thumbs.splides ) );
    }
  }, [] );

  return (
    <>
      <Splide ref={ mainRef } options={ mainOptions } id="main">
        { slides.map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <Splide ref={ thumbsRef } options={ thumbsOptions } id="thumbs">
        { slides.map( n => <SplideSlide key={ n }>Thumb { n }</SplideSlide> ) }
      </Splide>
      <button type="button" id="add" onClick={ () => setSlides( list => [ ...list, list.length + 1 ] ) }>Add a slide</button>
    </>
  );
}
```

Tested in production and under StrictMode: the third thumbnail gets `role="button"`, `aria-label="Go to slide 3"`, `aria-controls="main-slide03"` and `tabindex="0"`; clicking it, or pressing Enter on the second thumbnail, moves the main carousel.

- Core 4.1.4 drops the navigation attributes on every `refresh` of the thumbnails: the new Slide objects only get them from `navigation:mounted`, which Sync emits once, when it mounts. In React a `refresh` happens whenever the slide children change. Tested without the `refresh` listener: after adding a slide, the first thumbnail had `role="tabpanel"` and no `aria-controls` (clicks still worked). With the listener, the new fifth thumbnail gets `role="button"` and `aria-controls="main-slide05"`. It has to be added with `splide.on()` after mount, not as an `onRefresh` prop, because a prop handler runs before the Slide objects are rebuilt.

- Unmounting one carousel destroys it and removes the `move` listeners between the two, but the other keeps the destroyed instance in its `splides` array (from source). A new instance of the first one is not synced again by an effect that already ran. Mount and unmount the pair together, or give both the same `key`.
- Under StrictMode the effect runs again after the remount with the new instances, so the pair is synced once per live instance.

### Events emitted / listened

| Event | Direction | Action |
|---|---|---|
| `navigation:mounted` | Emitted | `( splides )` when `isNavigation`. React prop `onNavigationMounted`, called as `( splide, splides )`. |
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

Access: `ref.current?.splide?.Components.Wheel` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Wheel` inside an event prop, where the core instance is the first argument.

### Methods

The public interface is empty. The object has only `mount()` (lifecycle).

### Behaviour

- Only when `wheel` is truthy at mount. It binds `wheel` on the track with `{ passive: false, capture: true }`. Changing `wheel` after mount has no effect, because nothing listens to `updated`. In React, a `wheel` change in the `options` prop is ignored; remount with a `key`.
- Only `deltaY` is used; horizontal trackpad swipes (`deltaX`) are ignored. If `abs( deltaY ) > wheelMinThreshold` (default 0) and more than `wheelSleep` ms (default 0) have passed since the last accepted wheel event, it calls `splide.go( deltaY < 0 ? '<' : '>' )`.
- The page scroll is prevented, except when `releaseWheel` is set, the slider is not moving, and there is no slide in that direction (`Controller.getAdjacent( backwards ) === -1`). The page then scrolls on at the first or last slide.
- Only cancelable wheel events are handled.
- The listener is a native one on the track, so React's `onWheel` (which React attaches as a passive listener at the root) is not needed and could not cancel the page scroll anyway.

The keyboard example above also sets `wheel: true` and `releaseWheel: true`.

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

Purpose: performs the actual animation for `Move.move()`. It lives in `src/js/transitions/`, not `src/js/components/`, but it is registered as `splide.Components.Transition`. The default is the Slide transition, or Fade when `type: 'fade'`. A custom one is the second argument of `splide.mount( Extensions, Transition )`, which React Splide fills from the `transition` prop.

Access: `ref.current?.splide?.Components.Transition` (with `const ref = useRef<Splide>( null )` on `<Splide ref={ ref }>`), or `splide.Components.Transition` inside an event prop, where the core instance is the first argument.

### Methods (`TransitionComponent`)

| Signature | Returns | Description |
|---|---|---|
| `start( index, done )` | `void` | Animates to `index` and must call `done()` when finished. Move calls it, after emitting `move`. |
| `cancel()` | `void` | Stops the animation. Move calls it from `Move.cancel()`. |

Also `mount()` (lifecycle).

### A custom transition in React

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// A transition that jumps without animation. Define it outside the component:
// React Splide passes it to mount() once, so a new function on a later render is ignored.
const Instant: NonNullable<SplideProps[ 'transition' ]> = ( splide, Components ) => ( {
  mount() {},
  start( index: number, done: () => void ) {
    Components.Move.jump( index );
    done();
  },
  cancel() {},
} );

export default function TransitionExample() {
  const [ moved, setMoved ] = useState( 0 );

  return (
    <>
      <Splide transition={ Instant } options={ { label: 'Instant' } } onMoved={ () => setMoved( n => n + 1 ) }>
        { [ 1, 2, 3 ].map( n => <SplideSlide key={ n }>Slide { n }</SplideSlide> ) }
      </Splide>
      <p id="moved">{ moved }</p>
    </>
  );
}
```

Tested: a Next click makes slide 2 active in the same task, with no CSS `transition` on the list, and `onMoved` fires once.

- The prop type is `ComponentConstructor`, which returns `BaseComponent` (all members optional). Include at least one lifecycle member (`mount() {}` here), or TypeScript reports "has no properties in common with type 'BaseComponent'".
- `NonNullable<SplideProps[ 'transition' ]>` gives the type without importing from `@splidejs/splide`.
- The prop is read once, at mount. React Splide also spreads it onto the root element, so in development React logs `Invalid value for prop 'transition' on <div> tag` (tested: a console error under `--dev`, nothing in production). It is harmless. The `extensions` prop is spread the same way, as `extensions="[object Object]"`.
- Custom transitions are covered in `references/custom-transitions.md`.

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
| `type` | Creation | Chooses Slide or Fade. Changing `type` through the `options` prop later does not switch the transition. |
