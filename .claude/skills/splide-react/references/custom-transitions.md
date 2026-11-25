# Custom transitions

Source: https://splidejs.com/guides/transition/ and https://splidejs.com/integration/react-splide/

Writing a Transition component in TypeScript and passing it to React Splide with the `transition` prop: the interface, how `Move` drives it, the contract it must keep, the built-in Slide and Fade, how options and `useScroll` affect it, and four tested examples.

Checked against the Splide core 4.1.4 source (`src/js/transitions`, `components/Move`, `components/Scroll`, `components/Controller`, `core/Splide`) and React Splide 0.7.12, which bundles core 4.1.3 (the two differ only in `package.json`). Every example was type-checked with `tsc --strict`, built with Vite and run in Chrome with React 19, as a production build and as a development build under `<StrictMode>`. Read `references/custom-extensions.md` first: a Transition is built the same way as an extension, and that file covers events, options, lifecycle, cleanup and the bundle cost of core imports in full.

## Contents

- [Packages and types](#packages-and-types)
- [The Transition interface](#the-transition-interface)
- [How Move drives the transition (source)](#how-move-drives-the-transition-source)
- [The contract your transition must keep](#the-contract-your-transition-must-keep)
- [Built-in Slide transition](#built-in-slide-transition)
- [Built-in Fade transition](#built-in-fade-transition)
- [Interaction with options](#interaction-with-options)
- [useScroll and the Scroll component](#usescroll-and-the-scroll-component)
- [Registration: the transition prop](#registration-the-transition-prop)
- [CleanSplide: keeping the props off the DOM](#cleansplide-keeping-the-props-off-the-dom)
- [Example 1: the docs' CSS transition](#example-1-the-docs-css-transition)
- [Example 2: CSS slide transition that respects options](#example-2-css-slide-transition-that-respects-options)
- [Example 3: zoom crossfade for type fade](#example-3-zoom-crossfade-for-type-fade)
- [Example 4: Scroll-driven transition with distance-based duration](#example-4-scroll-driven-transition-with-distance-based-duration)
- [Pitfalls checklist](#pitfalls-checklist)
- [Docs vs source](#docs-vs-source)

## Packages and types

```bash
npm install @splidejs/react-splide
npm install @splidejs/splide@4.1.4   # for the core types; required under pnpm, optional under npm
```

```ts
import type { Components, Options, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
```

- `import type` adds nothing to the bundle. A **value** import from `@splidejs/splide` (`EventInterface`, `SLIDE`, `EVENT_MOVED`) adds a second copy of core, about 12 kB gzipped, because React Splide inlines its own (measured in `references/custom-extensions.md`). The examples below use `splide.event.bind()` and string constants instead.
- Without `@splidejs/splide` as a dependency, derive the types: `type Components = Parameters<NonNullable<SplideProps[ 'extensions' ]>[ string ]>[ 1 ]` and `type TransitionComponent = Components[ 'Transition' ]` (details in `references/custom-extensions.md`).
- **Annotate the return type as `TransitionComponent`.** The `transition` prop is typed `ComponentConstructor`, which returns `BaseComponent` (all optional members). A function whose inferred return is `{ start, cancel }` fails with "has no properties in common with type 'BaseComponent'" (TypeScript's weak type check). With `: TransitionComponent` it compiles, and `start`/`cancel` are checked (tested).
- React Splide's own types need one tsconfig line under `moduleResolution: "bundler"`. See `references/typescript.md`.

## The Transition interface

A Transition is a component constructor, `( Splide, Components, options ) => TransitionComponent`:

```ts
// From @splidejs/splide dist/types/index.d.ts:
// interface BaseComponent       { setup?(): void; mount?(): void; destroy?( completely?: boolean ): void; }
// interface TransitionComponent extends BaseComponent {
//   start( index: number, done: () => void ): void;
//   cancel(): void;
// }
```

| Member | Required | Called by | Purpose |
|---|---|---|---|
| `start( index, done )` | yes | `Move.move()` | Animate to `index`, then call `done()`. |
| `cancel()` | yes | `Move.cancel()` | Stop the running animation now. |
| `setup()` | no | `Splide#mount()` | Runs right after construction, before any component's `mount()`. |
| `mount()` | no | `Splide#mount()` | Runs **last**, after every core component and extension has mounted, and before the `mounted` event. |
| `destroy( completely )` | no | `Splide#destroy()` | Runs **first** (destroy goes in reverse order). Unmounting the React component passes `true`. |

`start` parameters:

| Param | Type | Description |
|---|---|---|
| `index` | `number` | The destination slide index, always a **real** index (`0` to `length - 1`), never a clone index. In loop mode, `Move` has already shifted the list so that moving to `index` goes the right way (see below). |
| `done` | `() => void` | Must be called once the movement has finished. |

The Transition is constructed after all core components and extensions, so `Components.Move`, `Components.Elements`, `Components.Slides`, `Components.Controller` and `Components.Scroll` can be destructured in its constructor. It lives at `Components.Transition`, and `Move` picks it up in `Move.mount()`.

The docs' skeleton, typed:

```ts
function MyTransition( splide: SplideCore, Components: Components, options: Options ): TransitionComponent {
  // Optional
  function mount() {
  }

  // Required
  function start( index: number, done: () => void ) {
  }

  // Required
  function cancel() {
  }

  // Optional
  function destroy() {
  }

  return {
    mount,
    start,
    cancel,
    destroy,
  };
}
```

The third argument, `options`, is the live options object. Read `speed`, `easing` and `rewindSpeed` from it at `start()` time.

## How Move drives the transition (source)

Every animated move goes through `Controller.go( control, allowSameIndex?, callback? )`, which is used by `splide.go()` (and so `ref.current.go()` and autoplay), arrows, pagination, keyboard, wheel, sync and non-free drag releases. It calls `Move.move( dest, index, prev, callback )`:

```ts
// components/Move/Move.ts (types stripped)
function move( dest, index, prev, callback ) {
  if ( dest !== index && canShift( dest > prev ) ) {
    cancel();                                               // Move.cancel(): freeze + Transition.cancel()
    translate( shift( getPosition(), dest > prev ), true ); // loop mode: jump one full loop
  }

  set( MOVING );                                            // splide.state = MOVING
  emit( EVENT_MOVE, index, prev, dest );

  Transition.start( index, () => {                          // this arrow function is `done`
    set( IDLE );
    emit( EVENT_MOVED, index, prev, dest );
    callback && callback();
  } );
}

function cancel() {
  translate( getPosition(), true );   // pin the list where it currently is (mid-animation position)
  Transition.cancel();
}
```

- `dest` can be a clone index in loop mode (`-1`, `length`). `index` is the real index. When they differ, `Move` first cancels any running transition and instantly shifts the list by a full loop, so animating to `toPosition( index )` looks continuous. Verified: `go( '<' )` from 0 in a 6-slide loop gave `move(5,0,-1)`, and `go( '>' )` from 5 gave `move(0,5,6)`.
- `done` sets the state to `IDLE`, emits `moved( index, prev, dest )` (the `onMoved` prop), and runs the optional `callback` given to `Controller.go()`.
- **`Move.cancel()` callers:** `Drag` on pointerdown (whenever `Controller.isBusy()` is false, which with the default `waitForTransition` means always), and `Move.move()` itself before a loop shift. A cancelled move never gets its `done` in the built-in CSS path, so no `moved` fires for it. The state stays `MOVING` until the next move finishes.
- **Moves that bypass the Transition:**
  - `Move.jump( index )`, which `Move.reposition()` calls on `mounted`, `resized`, `updated` and `refresh` when not busy. `updated` includes changes of the `options` prop, and `refresh` includes changes of the rendered slides.
  - `Controller.scroll()` and `Scroll.scroll()`: `drag: 'free'` releases scroll directly and emit `scroll`/`scrolled`, not `move`/`moved`.
  - Dragging itself, which calls `Move.translate()` on every pointer move.
- `start()` may be called again before the previous `done` has run. With the default `waitForTransition: false`, `Controller.go()` doesn't wait. The new call **replaces** the old one, and the old `done` must not run.
- `Arrows` call `Controller.go( control, true )`, pagination clicks call `Controller.go( '>' + page, true )`, and non-free drag releases call `Controller.go( toDest( position ), true )`. `allowSameIndex = true` means `start()` can be asked to go to the index it is already at.

Helpers you will use from `Components.Move`:

| Method | Description |
|---|---|
| `toPosition( index, trimming? )` | Converts an index to a list position in px (negative for ltr and ttb, positive for rtl). Pass `true` to apply `trimSpace` clamping, as the built-in does. |
| `getPosition()` | Current visual position, measured with `getBoundingClientRect`, so it is correct mid-animation. |
| `translate( position, preventLoop? )` | Sets `transform: translateX(px)` (`translateY` for ttb) on the list. **Does nothing when `type` is `'fade'`.** Without `preventLoop`, in loop mode a position beyond the ends is shifted back by a loop and `sh` is emitted. The built-in passes `true`. |
| `jump( index )` | `translate( toPosition( index, true ) )`, with no animation. |
| `shift( position, backwards )`, `toIndex( position )`, `getLimit( max )`, `exceededLimit( max?, position? )` | Lower-level helpers. |

## The contract your transition must keep

1. **Call `done()` exactly once for each `start()`** that is not cancelled or superseded, including when nothing has to move. When the target position equals the current one (`focus: 'center'` with `trimSpace` near the edges, a drag released exactly in place, `allowSameIndex` moves, a hidden slider measuring 0), a CSS transition never starts and `transitionend` never fires. The built-in Slide checks `Math.abs( destination - position ) >= 1 && speed >= 1` and otherwise calls `Move.jump( index ); done();` synchronously.
2. **If `done` is never called** (verified with the docs' example, 6 slides, `perPage: 3, focus: 'center'`, going 0 to 1, where both positions are 0):
   - The state stays `MOVING` and `moved` (and `onMoved`) never fires.
   - `is-active` and `is-visible` stay on the old slide, because slide classes update on `moved`.
   - `aria-hidden` stops updating.
   - With `waitForTransition: true`, the slider ignores every `go()` and drag from then on.
   - `Move.reposition()` stops running on resize.
   - With `wheel: true, releaseWheel: true`, the wheel never hands scrolling back to the page.

   The failure depends on layout: in a test with 3 slides the two positions differed by a fraction of a pixel (0.016px and 0.008px), which still ran the CSS transition and fired `transitionend`. Don't rely on that.
3. **`cancel()` must stop the animation immediately** and leave the list where `Move.cancel()` pinned it. In the built-in CSS path it does not call `done`.
4. **Handle re-entry.** If `start()` runs while an animation is in flight, store the new `done` and drop the old one. Clear any timers from the previous call.
5. **Read options inside `start()`.** Breakpoints, `reducedMotion`, changes of the `options` prop and the temporary override `Drag` applies during a release all change `options.speed` in place (see [Interaction with options](#interaction-with-options)).
6. **Clean up in `destroy()`:** timers, `requestAnimationFrame` loops, and any `addEventListener` you made yourself. `Elements.destroy()` strips the list's `style` attribute, and each slide's original `style` attribute is restored on destroy. Listeners registered through `splide.event.bind()` or `splide.on()` are removed for you.

## Built-in Slide transition

The default for `type: 'slide'` and `type: 'loop'` (`transitions/Slide/Slide.ts`, types stripped):

```ts
function Slide( Splide, Components, options ) {
  const { Move, Controller, Scroll } = Components;
  const { list } = Components.Elements;
  const transition = value => { list.style.transition = value; };
  let endCallback;

  function mount() {
    EventInterface( Splide ).bind( list, 'transitionend', e => {
      if ( e.target === list && endCallback ) {
        cancel();
        endCallback();
      }
    } );
  }

  function start( index, done ) {
    const destination = Move.toPosition( index, true );
    const position    = Move.getPosition();
    const speed       = getSpeed( index );

    if ( Math.abs( destination - position ) >= 1 && speed >= 1 ) {
      if ( options.useScroll ) {
        Scroll.scroll( destination, speed, false, done );     // JS (rAF) animation, uses easingFunc
      } else {
        transition( `transform ${ speed }ms ${ options.easing }` );
        Move.translate( destination, true );
        endCallback = done;
      }
    } else {
      Move.jump( index );                                    // nothing to animate, or speed 0
      done();
    }
  }

  function cancel() {
    transition( '' );
    Scroll.cancel();
  }

  function getSpeed( index ) {
    const { rewindSpeed } = options;

    if ( Splide.is( 'slide' ) && rewindSpeed ) {
      const prev = Controller.getIndex( true );
      const end  = Controller.getEnd();

      if ( ( prev === 0 && index >= end ) || ( prev >= end && index === 0 ) ) {
        return rewindSpeed;
      }
    }

    return options.speed;
  }

  return { mount, start, cancel };
}
```

Summary:

- It uses a CSS `transform` transition on the list, with duration `speed` (or `rewindSpeed`) and timing function `easing`.
- It finishes on the list's own `transitionend`, checking `e.target === list` so that transitions on child elements are ignored.
- It jumps instantly when the distance is under 1px or the speed is under 1ms.
- It delegates to `Scroll.scroll()` when `useScroll` is true.
- `cancel()` clears the CSS transition and cancels any Scroll animation.
- It never clears `endCallback`. A stray `transitionend` on the list after a finished move would call the old `done` again, which is why Example 2 nulls it.

The built-in `Slide` and `Fade` constructors are not exported by core or by React Splide, so a custom transition can't wrap them. Example 2 re-implements Slide as a starting point.

## Built-in Fade transition

The default for `type: 'fade'` (`transitions/Fade/Fade.ts`):

```ts
function Fade( Splide, Components, options ) {
  const { Slides } = Components;

  function mount() {
    EventInterface( Splide ).on( [ 'mounted', 'refresh' ], init );
  }

  // Stacks every slide at the head of the list.
  function init() {
    Slides.forEach( Slide => {
      Slide.style( 'transform', `translateX(-${ 100 * Slide.index }%)` );
    } );
  }

  function start( index, done ) {
    Slides.style( 'transition', `opacity ${ options.speed }ms ${ options.easing }` );
    setTimeout( done );           // nextTick: done on the next macrotask
  }

  return { mount, start, cancel: () => {} };
}
```

The visible fade is pure CSS. `Elements` adds `splide__track--fade` to the track, and the stylesheet (`@splidejs/react-splide/css` and `/css/core`) has:

```css
.splide__track--fade > .splide__list > .splide__slide { margin: 0 !important; opacity: 0; z-index: 0; }
.splide__track--fade > .splide__list > .splide__slide.is-active { opacity: 1; z-index: 1; }
```

Because `done` runs right away, `moved` fires about 0ms after `move`. That swaps the `is-active` class, and the opacity transition (set in `start`) then plays for `speed` ms **after** `moved`. Fade ignores `rewindSpeed` and `useScroll`. Its `cancel` does nothing. With `type: 'fade'`, `Move.translate()` does nothing and no clones are created.

## Interaction with options

The built-in transitions read options at `start()` time. A custom transition only gets these behaviours if it implements them.

| Option | Default | How it affects transitions |
|---|---|---|
| `type` | `'slide'` | Chooses the default transition (`'fade'` gives Fade, anything else gives Slide) when no `transition` prop is passed. A custom transition is used for every type. With `'fade'`, `Move.translate()` is a no-op, so a translate-based transition shows nothing. |
| `speed` | `400` | Duration in ms. `0` means jump: Slide checks `speed >= 1`. |
| `rewindSpeed` | not set, so `speed` is used | Slide only, `type: 'slide'` only. Used when moving from index `0` to an index `>= Controller.getEnd()`, or from `>= end` to `0`. That includes a plain `go( end )` from 0, not only rewinds (verified: `go( 5 )` from 0 in a 6-slide slider took the `rewindSpeed`). |
| `easing` | `'cubic-bezier(0.25, 1, 0.5, 1)'` | CSS timing function for the CSS path (Slide) and the opacity fade (Fade). Ignored by the Scroll path. |
| `easingFunc` | `t => 1 - Math.pow( 1 - t, 4 )` | JS easing used by `Scroll.scroll()`: `useScroll` transitions, `drag: 'free'`, and any transition that calls `Scroll.scroll()`. In React, define it at module scope: React Splide compares functions in `options` by identity, so an inline arrow pushes the options again on every render. |
| `useScroll` | not set | Slide only: animate with `Scroll.scroll()` (requestAnimationFrame) instead of a CSS transition. |
| `reducedMotion` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Merged into `options` while `(prefers-reduced-motion: reduce)` matches, so `speed` becomes 0 and Slide jumps. **`Drag` lifts it during a release:** it calls `Media.reduce( false )`, then `Controller.go()` (which calls `start()` synchronously), then `Media.reduce( true )`. Drag releases therefore animate at normal speed, as "essential" motion. Read `options.speed` synchronously in `start()` and keep the value. Reading it later (in a timeout) gives 0 again. |
| `waitForTransition` | not set (falsy) | When `true`, `Controller.isBusy()` is true while the state is `MOVING` or `SCROLLING`: `go()` is ignored, pointerdown is prevented, and `reposition()` is skipped. When falsy (the default), `start()` can be re-entered and drags call `cancel()` mid-move. |
| `rewind` | `false` | Only changes which index `go()` resolves. The transition just gets `index`. |
| `drag: 'free'` | | Releases use `Controller.scroll()` and bypass the Transition entirely. |
| `focus`, `trimSpace`, `perPage`, `padding`, `gap`, `direction` | | All handled by `Move.toPosition( index, true )` and `Move.translate()`. Use them rather than computing positions yourself. `focus: 'center'` with `trimSpace: true` makes several indices map to the same position, so the zero-distance guard is required. |

Changing the `options` prop (`speed`, `easing`, `useScroll`) reaches the transition without a remount: React Splide runs `splide.options = options`, which mutates the same object your transition reads in `start()`. The tests below changed `speed`, `perPage`, `focus` and `useScroll` at runtime through that same `splide.options` setter.

## useScroll and the Scroll component

`useScroll: true` is the built-in alternative to a CSS transition. The Slide transition then calls `Components.Scroll.scroll( destination, speed, false, done )`. For a custom easing curve you usually don't need a custom transition:

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

// easeInOutCubic. Defined once: an inline arrow would be a new function on every render,
// and React Splide would push the options again (emitting `updated`) each time.
const easeInOutCubic = ( t: number ) => ( t < 0.5 ? 4 * t * t * t : 1 - Math.pow( -2 * t + 2, 3 ) / 2 );

export default function Example() {
  const [ clicks, setClicks ] = useState( 0 );

  const options: Options = {
    useScroll : true,
    speed     : 600,
    easingFunc: easeInOutCubic,
    label     : 'useScroll',
  };

  return (
    <div>
      <Splide options={ options }>
        { Array.from( { length: 6 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
      </Splide>
      <button type="button" className="rerender" onClick={ () => setClicks( n => n + 1 ) }>
        Re-render ({ clicks })
      </button>
    </div>
  );
}
```

Verified in production and development: two re-renders emitted no `updated` (the easing function is the same object each time), and a move emitted `move`, `scroll`, `moved`, `scrolled`.

`Scroll.scroll( destination, duration?, snap?, onScrolled? )` (`components/Scroll/Scroll.ts`):

- Each frame, it moves the list from the current position toward `destination` along `easingFunc` (default easeOutQuart), using `RequestInterval( duration, onEnd, update, 1 )`. It translates **without** `preventLoop`, so in loop mode positions past the ends are shifted and `sh` is emitted.
- `duration` falsy (including `0`) does **not** mean instant. It means automatic: `max( distance / 1.5, 800 )` ms. If the distance is under 1px, the duration is 0 and it ends on the next frame. To jump, use `Move.jump()`.
- `snap = true` snaps the destination to the nearest slide (in `slide` type, only while not past the limits).
- In `slide` type, overshooting the limits applies friction and bounces back to the limit over 600ms.
- It sets the state to `SCROLLING` and emits `scroll`. On completion: state `IDLE`, then `onScrolled()`, then `scrolled`. Used as a transition, `onScrolled` is `done`, so the observed order is `move`, `scroll`, `moved`, `scrolled`. The state also goes `MOVING`, `SCROLLING`, then `IDLE`.
- `Scroll.cancel()`: if a scroll is running, it stops it **and calls `onEnd`**, which means the callback (your `done`) runs and `moved` and `scrolled` fire. This differs from the CSS path, where a cancelled move never emits `moved`. Scroll also runs `cancel` on `updated` and `refresh`, so an `options` prop change or a slide change mid-scroll finishes the move.
- On every `move` event, Scroll stops an in-flight scroll **without** calling its callback, so a superseded `done` is dropped.
- `Scroll.destroy` stops the interval.
- `Controller.scroll( destination, duration?, snap?, callback? )` wraps `Scroll.scroll` and afterwards sets the index to the slide closest to where it stopped. This is what `drag: 'free'` uses. It emits no `move`/`moved`.

A custom transition can call `Scroll.scroll()` itself to get the JS animation with its own duration rules ([Example 4](#example-4-scroll-driven-transition-with-distance-based-duration)).

## Registration: the transition prop

Pass the constructor to the `transition` prop. Core receives it as the second argument of `mount()`:

```tsx
<Splide transition={ MyTransition } options={ { label: 'Gallery' } }>...</Splide>
```

- **Read once, at mount.** React Splide's `componentDidMount` calls `splide.mount( extensions, transition )`, and nothing reads the prop again. Define the transition at module scope. To switch transitions, remount the carousel with a new `key`.
- **With extensions:** `<Splide extensions={ { SlideCounter } } transition={ MyTransition }>`. The Transition is still constructed and mounted last.
- **`{ Transition: MyTransition }` in `extensions` is ignored.** `mount()` assigns `{ Transition: this._T }` after the extensions (verified). Only the `transition` prop works.
- **Remounts keep it.** `this._T = Transition || this._T || ( fade ? Fade : Slide )`. A breakpoint remount after `destroy: true` calls `mount()` without arguments and reuses your transition (verified). A React remount (StrictMode, a new `key`) builds a new core instance from the props again.
- **Per carousel.** Each `<Splide>` needs its own `transition` prop. `Splide.defaults` can't set one, and React Splide ignores `Splide.defaults` set on an imported core class anyway.

**The development warning.** `render()` spreads every prop except `options` and the event props onto the root element, so React receives a function in an unknown DOM attribute. In development it logs:

```
Invalid value for prop `transition` on <div> tag. Either remove it from the element, or pass a string or number value to keep it in the DOM.
```

Production builds drop the attribute silently (tested: no `transition` attribute, the custom transition ran). The carousel works either way. `extensions` is rendered too, as `extensions="[object Object]"`, without a warning. The subclass below removes both.

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Components, Options, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
import '@splidejs/react-splide/css';

// A minimal transition: jump, then report done.
function Jump( splide: SplideCore, Components: Components, options: Options ): TransitionComponent {
  return {
    start( index, done ) {
      Components.Move.jump( index );
      done();
    },
    cancel() {},
  };
}

export default function Example() {
  return (
    <Splide transition={ Jump } options={ { label: 'Plain transition prop' } }>
      <SplideSlide>01</SplideSlide>
      <SplideSlide>02</SplideSlide>
      <SplideSlide>03</SplideSlide>
    </Splide>
  );
}
```

Verified: in production this logged nothing and a move emitted `move` then `moved` from `Jump`. In development the only console error was the warning above.

## CleanSplide: keeping the props off the DOM

React Splide's `render()` calls its public `omit( props, keys )` method to strip `options` and the event props. A subclass can strip two more. Save it as `CleanSplide.tsx`; the examples below import it from `./CleanSplide`:

```tsx
import { Splide } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';

// <Splide> that keeps `extensions` and `transition` off the root element.
// React Splide 0.7.12 spreads both onto the DOM: React renders extensions="[object Object]"
// and, in development, logs "Invalid value for prop `transition` on <div> tag".
export class CleanSplide extends Splide {
  omit<K extends keyof SplideProps>( props: SplideProps, keys: readonly K[] ): Omit<SplideProps, K> {
    delete props.extensions;
    delete props.transition;
    return super.omit( props, keys );
  }
}
```

Use it exactly like `<Splide>`: same props, and a ref typed `useRef<CleanSplide>( null )` gives the same `ref.current.splide`. `componentDidMount` still reads `this.props.extensions` and `this.props.transition`, because `omit` only edits the copy that `render()` spreads onto the root. Tested in production and development: no `extensions` or `transition` attribute, no warning, and extensions, transitions and event props all worked. The examples below use it. In the Next.js App Router, a file that renders it must be a client component, as for `<Splide>` (`references/nextjs.md`).

It relies on React Splide's internal `render()` calling `omit()`, which is true for 0.7.12, the latest release (the repository has been unchanged since 2022). If a future version stops calling it, the subclass simply stops stripping the two props.

## Example 1: the docs' CSS transition

This is the docs' example converted. It hardcodes 800ms, ignores `speed`, `easing`, `rewindSpeed`, `reducedMotion` and `useScroll`, and has no zero-distance guard, so it gets stuck in `MOVING` when the position doesn't change (see the contract above). Use it to learn the mechanism. For real use, take Example 2. The docs bind `transitionend` with `EventInterface( Splide ).bind()`. This version uses `splide.event.bind()`, which does the same (removed on destroy) without importing core.

```tsx
import { SplideSlide } from '@splidejs/react-splide';
import type { Components, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
import { CleanSplide } from './CleanSplide';
import '@splidejs/react-splide/css';

// The docs' example: hardcoded 800ms, no zero-distance guard (see the contract).
function MyTransition( splide: SplideCore, Components: Components ): TransitionComponent {
  const { Move } = Components;
  const { list } = Components.Elements;
  let endCallback: ( () => void ) | undefined;

  function onTransitionEnd( e: TransitionEvent ) {
    if ( e.target === list && endCallback ) {
      // Removes the transition property
      cancel();

      // Calls the `done` callback
      endCallback();
    }
  }

  function mount() {
    // Removed on destroy, like everything registered through splide.event
    splide.event.bind( list, 'transitionend.myTransition', onTransitionEnd );
  }

  function start( index: number, done: () => void ) {
    // Converts the index to the position
    const destination = Move.toPosition( index, true );

    // Applies the CSS transition
    list.style.transition = 'transform 800ms cubic-bezier(.44,.65,.07,1.01)';

    // Moves the carousel to the destination.
    Move.translate( destination );

    // Keeps the callback to invoke later.
    endCallback = done;
  }

  function cancel() {
    list.style.transition = '';
  }

  return { mount, start, cancel };
}

export default function Example() {
  return (
    <CleanSplide transition={ MyTransition } options={ { label: 'Custom transition example' } }>
      { Array.from( { length: 6 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </CleanSplide>
  );
}
```

Verified in production and development: `go( 2 )` emitted `moved` about 800ms after `move` (plus a frame or two), and the list ended at `translateX()` equal to `Move.toPosition( 2, true )`. With `perPage: 3, focus: 'center'` set through `splide.options`, going 0 to 1 emitted `move` and never `moved`, and the state stayed `MOVING`.

## Example 2: CSS slide transition that respects options

This is a production template: the built-in Slide behaviour plus a few hardening steps. Change the `transition` string or add effects here. It:

- uses `speed`, `easing`, `rewindSpeed` and `useScroll`;
- jumps on zero distance or zero speed, which also covers `reducedMotion`;
- passes `preventLoop` like the built-in;
- checks `propertyName` so that other transitioned properties on the list don't end the move early;
- clears the stored `done` so it can't run twice;
- has a fallback timer in case `transitionend` never fires;
- cleans up on destroy.

```tsx
import { SplideSlide } from '@splidejs/react-splide';
import type { Components, Options, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
import { CleanSplide } from './CleanSplide';
import '@splidejs/react-splide/css';

// The built-in Slide behaviour plus hardening. Change the `transition` string for a different motion.
function CssSlide( splide: SplideCore, Components: Components, options: Options ): TransitionComponent {
  const { Move, Controller, Scroll } = Components;
  const { list } = Components.Elements;
  let endCallback: ( () => void ) | null = null;
  let timer = 0;

  function mount() {
    splide.event.bind( list, 'transitionend.cssSlide', ( e: TransitionEvent ) => {
      if ( e.target === list && e.propertyName === 'transform' ) {
        finish();
      }
    } );
  }

  function start( index: number, done: () => void ) {
    const destination = Move.toPosition( index, true );
    const position    = Move.getPosition();
    const speed       = getSpeed( index );          // read now: Drag lifts reducedMotion only during this call

    if ( Math.abs( destination - position ) < 1 || speed < 1 ) {
      Move.jump( index );                           // nothing to animate: transitionend would never fire
      done();
      return;
    }

    if ( options.useScroll ) {
      Scroll.scroll( destination, speed, false, done );
      return;
    }

    endCallback = done;                             // replaces (drops) any in-flight done
    list.style.transition = `transform ${ speed }ms ${ options.easing }`;
    Move.translate( destination, true );

    clearTimeout( timer );
    timer = window.setTimeout( finish, speed + 100 );   // safety net if transitionend is missed
  }

  function finish() {
    const done = endCallback;
    cancel();

    if ( done ) {
      done();
    }
  }

  function cancel() {
    clearTimeout( timer );
    list.style.transition = '';
    endCallback = null;
    Scroll.cancel();                                // like the built-in; runs done if a useScroll move is in flight
  }

  function getSpeed( index: number ): number {
    const { rewindSpeed } = options;

    if ( splide.is( 'slide' ) && rewindSpeed ) {
      const prev = Controller.getIndex( true );
      const end  = Controller.getEnd();

      if ( ( prev === 0 && index >= end ) || ( prev >= end && index === 0 ) ) {
        return rewindSpeed;
      }
    }

    return options.speed!;
  }

  function destroy() {
    clearTimeout( timer );
    endCallback = null;                             // the list's style attribute is reset by Elements on destroy
  }

  return { mount, start, cancel, destroy };
}

export default function Example() {
  return (
    <CleanSplide
      transition={ CssSlide }
      options={ { speed: 500, rewind: true, rewindSpeed: 1200, label: 'CSS slide' } }
    >
      { Array.from( { length: 6 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </CleanSplide>
  );
}
```

Verified in production and development with `speed: 500, rewind: true, rewindSpeed: 1200` and 6 slides: moves took about 500ms (`go( 1 )`, and `go( 5 )` from 1), and the rewind from 5 to 0 took about 1200ms. With `speed: 0` set at runtime, `moved` came immediately. With `perPage: 3, focus: 'center'`, going 0 to 1 (no distance) emitted `moved` immediately. `useScroll: true` emitted `move`, `scroll`, `moved`, `scrolled`. `Move.cancel()` mid-move cleared the transition with no `moved`, and the next `go()` worked.

Notes:

- The fallback timer is not in the built-in. Without it, `transitionend` is the only completion signal, and if it is missed the move never completes. If it fires first (heavy main-thread load), removing the transition snaps the list to the destination it already has.
- For a different motion, change only the `transition` string, for example `transform ${ speed }ms steps(6)` or a custom `cubic-bezier()`, or toggle a class on `splide.root` in `start` and `finish` to drive extra CSS (scale or dim the slides while moving).
- Keep `Move.translate( destination, true )`. Don't write `list.style.transform` yourself: `translate` resolves `translateX` or `translateY` for the direction and skips fade.
- `splide.is( 'slide' )` uses the string, because the `SLIDE` constant would import core.

## Example 3: zoom crossfade for type fade

A custom transition for `type: 'fade'`. `'fade'` keeps the fade CSS (`margin: 0`, stacking z-index), makes `Move.translate` a no-op, and creates no clones. The built-in Fade's stacking `init()` doesn't run once you replace it, so this does the stacking itself. Opacity and scale are set inline on every slide, which overrides the stylesheet's `.is-active` opacity, so the incoming slide fades in during the move instead of after `moved`. `done` runs when the animation has finished, which is not what the built-in Fade does.

```tsx
import { SplideSlide } from '@splidejs/react-splide';
import type { Components, Options, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
import { CleanSplide } from './CleanSplide';
import '@splidejs/react-splide/css';

// Crossfade with a zoom, for type 'fade'. `zoomScale` is a custom option.
function ZoomFade( splide: SplideCore, Components: Components, options: Options ): TransitionComponent {
  const { Slides } = Components;
  let endCallback: ( () => void ) | null = null;
  let timer = 0;

  function mount() {
    // Slides are re-registered on refresh, so restack then too (the built-in Fade does the same).
    splide.on( 'mounted.zoomFade refresh.zoomFade', () => {
      Slides.style( 'transition', '' );
      show( splide.index );
    } );
  }

  function show( index: number ) {
    const scale = options.zoomScale || 1.08;

    Slides.forEach( Slide => {
      const active = Slide.index === index;
      Slide.style( 'opacity', active ? 1 : 0 );
      // translateX(-100% * i) stacks every slide at the head of the list, like the built-in Fade.
      Slide.style( 'transform', `translateX(-${ 100 * Slide.index }%) scale(${ active ? 1 : scale })` );
    } );
  }

  function start( index: number, done: () => void ) {
    const { speed = 0, easing } = options;
    clearTimeout( timer );

    if ( speed < 1 ) {                                  // speed 0 / reducedMotion: switch instantly
      Slides.style( 'transition', '' );
      show( index );
      endCallback = null;
      done();
      return;
    }

    Slides.style( 'transition', `opacity ${ speed }ms ${ easing }, transform ${ speed }ms ${ easing }` );
    show( index );
    endCallback = done;
    timer = window.setTimeout( finish, speed );         // done after the effect, unlike the built-in Fade
  }

  function finish() {
    const done = endCallback;
    endCallback = null;

    if ( done ) {
      done();
    }
  }

  function cancel() {
    clearTimeout( timer );
    Slides.style( 'transition', '' );                   // snap to the target state
    endCallback = null;
  }

  return { mount, start, cancel, destroy: cancel };     // slide style attributes are restored on destroy
}

export default function Example() {
  return (
    <CleanSplide
      transition={ ZoomFade }
      options={ { type: 'fade', rewind: true, speed: 600, zoomScale: 1.1, label: 'Zoom fade' } }
    >
      { Array.from( { length: 6 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </CleanSplide>
  );
}
```

Verified in production and development: about 250ms into a 600ms move, slide 1's computed opacity was between 0 and 1 while slide 0 was still `is-active`. `moved` fired about 600ms after `move`. Afterwards the inline opacities were `0,1,0,0,0,0`, slide 1 had `translateX(-100%) scale(1)`, `is-active` was on slide 1, and there were no clones. With `speed: 0`, `move` and `moved` came together.

Notes:

- `type: 'fade'` is required. With `'slide'` or `'loop'` the slides sit side by side, without fade CSS or stacking.
- `Slides.style( prop, value )` styles every slide (pass `true` as the third argument to style `.splide__slide__container` instead). `Slide.style()` styles one slide.
- The `z-index` of `.is-active` still switches on `moved`. Links in the incoming slide become clickable once the move completes.
- `zoomScale` is a custom option, so it is typed `any` through the `Options` index signature.

## Example 4: Scroll-driven transition with distance-based duration

This delegates the animation to the Scroll component, so it runs on requestAnimationFrame with `easingFunc`, and scales the duration with distance: long jumps (pagination to a far page) take longer than one-slide steps.

```tsx
import { SplideSlide } from '@splidejs/react-splide';
import type { Components, Options, Splide as SplideCore, TransitionComponent } from '@splidejs/splide';
import { CleanSplide } from './CleanSplide';
import '@splidejs/react-splide/css';

// JS animation through the Scroll component, with a duration that grows with distance.
function DistanceScroll( splide: SplideCore, Components: Components, options: Options ): TransitionComponent {
  const { Move, Scroll } = Components;

  function start( index: number, done: () => void ) {
    const speed       = options.speed || 0;
    const destination = Move.toPosition( index, true );
    const distance    = Math.abs( destination - Move.getPosition() );

    if ( distance < 1 || speed < 1 ) {
      Move.jump( index );
      done();
      return;
    }

    // 1.5px per ms, clamped to [speed, 3 * speed]. Never pass 0: Scroll treats 0 as "auto" (>= 800ms).
    const duration = Math.min( Math.max( distance / 1.5, speed ), speed * 3 );
    Scroll.scroll( destination, duration, false, done );
  }

  function cancel() {
    Scroll.cancel();   // note: runs the pending done (moved fires), unlike the CSS path
  }

  return { start, cancel };   // Scroll stops its own interval on destroy
}

// Module scope: functions inside options are compared by identity on every render.
const easeOutCubic = ( t: number ) => 1 - Math.pow( 1 - t, 3 );

export default function Example() {
  return (
    <CleanSplide
      transition={ DistanceScroll }
      options={ { speed: 400, easingFunc: easeOutCubic, label: 'Distance scroll' } }
    >
      { Array.from( { length: 6 }, ( _, i ) => <SplideSlide key={ i }>{ i + 1 }</SplideSlide> ) }
    </CleanSplide>
  );
}
```

Verified in production and development with `speed: 400` on a 1264px-wide slider: one slide took about 843ms (1264 / 1.5) plus frame overhead, and four slides about 1200ms (capped at `3 * speed`). Each move emitted `move`, `scroll`, `moved`, `scrolled`.

## Pitfalls checklist

- Call `done()` on every path, synchronously when there is nothing to animate. Guard with `Math.abs( Move.toPosition( index, true ) - Move.getPosition() ) < 1 || speed < 1`.
- Read `options.speed`, `options.easing` and `options.rewindSpeed` inside `start()`, not in the constructor and not later in a timeout.
- Use `Move.toPosition( index, true )` and `Move.translate( position, true )`. Don't compute pixel offsets or write `transform` on the list yourself.
- Filter `transitionend` with `e.target === list`, and preferably `e.propertyName === 'transform'`.
- Make `cancel()` stop the animation instantly, and make `start()` replace an in-flight `done`.
- `type: 'fade'` turns `Move.translate()` into a no-op and removes the built-in stacking once you replace Fade.
- Annotate the constructor's return type as `TransitionComponent`, or the `transition` prop rejects it.
- Use the `transition` prop. `{ Transition }` in `extensions` is ignored. The prop is read once, at mount.
- The plain `transition` prop logs a React warning in development. `CleanSplide` removes it.
- Define the transition and any `easingFunc` at module scope.
- `Scroll.scroll( dest, 0 )` is an automatic duration (800ms minimum), not a jump.
- Value imports from `@splidejs/splide` (`EventInterface`, `SLIDE`) add a second copy of core. Use `splide.event.bind()` and strings.

## Docs vs source

- **Lifecycle.** The docs say: `start()` and `cancel()` required, `mount()` and `destroy()` optional. The source also calls an optional `setup()`, and passes `destroy( completely )` a boolean. The Transition is constructed and mounted last and destroyed first.
- **When `cancel()` runs.** The docs say: when a transition is interrupted, for example a drag during a transition with `waitForTransition: false`. The source calls it through `Move.cancel()`, both from `Drag` on every pointerdown while the controller isn't busy (even with no transition running) and from `Move.move()` before a loop shift. In the CSS path a cancelled move never emits `moved`. In the Scroll path (`useScroll`), `Scroll.cancel()` does run `done`.
- **When `done` runs.** The docs say `done` "must be called after the transition ends". The built-in Fade calls it on the next macrotask (`setTimeout`), so `moved` fires before the fade has visibly finished. The built-in Slide calls it synchronously when there is nothing to animate.
- **Docs example.** It calls `Move.translate( destination )` without `preventLoop`, while the built-in passes `true`. The two are equivalent for real indices. It also hardcodes `800ms` and has no zero-distance guard. Verified: with 6 slides, `perPage: 3, focus: 'center'`, going 0 to 1 never calls `done`, so the slider stays `MOVING` and `moved` never fires.
- **`waitForTransition` loop points.** The options docs say that "Even if `false`, the carousel forcibly waits for the transition on the loop points". The 4.1.x source has no such check. The only busy test is `Controller.isBusy()`, which needs `waitForTransition: true`. At loop points, `Move.move()` cancels the running transition and shifts the list instead.
- **`rewindSpeed`.** Documented as "the transition speed on rewind". In the source it applies (slide type only) to any move between index 0 and an index `>= getEnd()`, in either direction, including a direct `go( end )`. Fade ignores it.
- **`useScroll`.** It is not on the options page. The type comment says "Determines whether to use the Transition component or not". In the source, only the built-in Slide transition reads it. Custom transitions ignore it unless they implement it, as Example 2 does.
- **`easingFunc`.** The docs say it is "for the drag free mode". In the source, `Scroll` uses it for every `Scroll.scroll()`, including `useScroll` transitions and custom transitions that call it.
- **React Splide's `transition` prop.** The React Splide page says only "Registers the custom transition component." In the source it is read once in `componentDidMount`, and it is also spread onto the root element, which makes React log an "Invalid value for prop `transition`" warning in development.
