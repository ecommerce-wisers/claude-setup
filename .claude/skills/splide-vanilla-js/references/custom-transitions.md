# Custom Transitions

Source: https://splidejs.com/guides/transition/

Checked against the Splide 4.1.3 source (`src/js/transitions`, `components/Move`, `components/Scroll`, `components/Controller`, `core/Splide`) and the 4.1.4 dist build, which match for everything here. Every example was run in Chrome against the 4.1.4 jsDelivr files. Read `custom-extensions.md` first: a Transition is built the same way as an extension, and that file covers `EventInterface`, options, lifecycle and cleanup in full.

## Contents

- [Loading: ESM module or classic script](#loading-esm-module-or-classic-script)
- [The Transition interface](#the-transition-interface)
- [How Move drives the transition (source)](#how-move-drives-the-transition-source)
- [The contract your transition must keep](#the-contract-your-transition-must-keep)
- [Built-in Slide transition](#built-in-slide-transition)
- [Built-in Fade transition](#built-in-fade-transition)
- [Interaction with options](#interaction-with-options)
- [useScroll and the Scroll component](#usescroll-and-the-scroll-component)
- [Registration](#registration)
- [Example 1: the docs' CSS transition](#example-1-the-docs-css-transition)
- [Example 2: CSS slide transition that respects options](#example-2-css-slide-transition-that-respects-options)
- [Example 3: zoom crossfade for type fade](#example-3-zoom-crossfade-for-type-fade)
- [Example 4: Scroll-driven transition with distance-based duration](#example-4-scroll-driven-transition-with-distance-based-duration)
- [Pitfalls checklist](#pitfalls-checklist)
- [Docs vs source](#docs-vs-source)

## Loading: ESM module or classic script

| | (a) Native ESM | (b) Classic script (UMD) |
|---|---|---|
| Tags | `<script type="module">` with `import { Splide, EventInterface, SLIDE } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js'` | `<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>`, then a normal `<script>` |
| Helpers | `EventInterface`, `RequestInterval`, `Throttle`, `EVENT_*`, `SLIDE`/`LOOP`/`FADE` | `window.Splide` and `Splide.STATES` only. Use strings (`'slide'`, `'moved'`) and `addEventListener`/`removeEventListener`, removed in `destroy()`. |
| Serving | http(s). Local module files fail from `file://`. | Works from `file://` too. |

CSS for both: `<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">`.

Pick ESM when you want `EventInterface` (automatic listener cleanup) or the constants. Pick classic when the page already uses `splide.min.js` (WordPress, Shopify and PHP themes) or must work from `file://`. Module scripts are deferred, so they run after parsing. Wrap classic scripts in `<head>` in `DOMContentLoaded`. Don't load both builds: they are separate copies.

A transition needs `Components` only, and `Components` is available in both builds, so the transition code is identical apart from event binding.

## The Transition interface

A Transition is a component constructor, `( Splide, Components, options ) => TransitionComponent`:

```js
// From dist/types/index.d.ts:
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
| `destroy( completely )` | no | `Splide#destroy()` | Runs **first** (destroy goes in reverse order). |

`start` parameters:

| Param | Type | Description |
|---|---|---|
| `index` | `number` | The destination slide index, always a **real** index (`0` to `length - 1`), never a clone index. In loop mode, `Move` has already shifted the list so that moving to `index` goes the right way (see below). |
| `done` | `() => void` | Must be called once the movement has finished. |

The Transition is constructed after all core components and extensions, so `Components.Move`, `Components.Elements`, `Components.Slides`, `Components.Controller` and `Components.Scroll` can be destructured in its constructor. It lives at `Components.Transition`, and `Move` picks it up in `Move.mount()`.

The docs' skeleton (both builds):

```js
function MyTransition( Splide, Components ) {
  // Optional
  function mount() {
  }

  // Required
  function start( index, done ) {
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

The constructor also receives `options` as its third argument (the live options object). Use it to read `speed`, `easing` and `rewindSpeed` at `start()` time.

## How Move drives the transition (source)

Every animated move goes through `Controller.go( control, allowSameIndex?, callback? )`, which is used by `splide.go()`, arrows, pagination, keyboard, wheel, sync and non-free drag releases. It calls `Move.move( dest, index, prev, callback )`:

```js
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
- `done` sets the state to `IDLE`, emits `moved( index, prev, dest )`, and runs the optional `callback` given to `Controller.go()`.
- **`Move.cancel()` callers:** `Drag` on pointerdown (whenever `Controller.isBusy()` is false, which with the default `waitForTransition` means always), and `Move.move()` itself before a loop shift. A cancelled move never gets its `done` in the built-in CSS path, so no `moved` fires for it. The state stays `MOVING` until the next move finishes.
- **Moves that bypass the Transition:**
  - `Move.jump( index )`, which `Move.reposition()` calls on `mounted`, `resized`, `updated` and `refresh` when not busy.
  - `Controller.scroll()` and `Scroll.scroll()`: `drag: 'free'` releases scroll directly and emit `scroll`/`scrolled`, not `move`/`moved`.
  - Dragging itself, which calls `Move.translate()` on every pointer move.
- `start()` may be called again before the previous `done` has run. With the default `waitForTransition: false`, `Controller.go()` doesn't wait. The new call **replaces** the old one, and the old `done` must not run.
- `Arrows` call `Controller.go( control, true )` and non-free drag releases call `Controller.go( toDest( position ), true )`. `allowSameIndex = true` means `start()` can be asked to go to the index it is already at.

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
2. **If `done` is never called** (verified with the docs' example, `perPage: 3, focus: 'center'`, going 0 to 1, where both positions are 0):
   - The state stays `MOVING` and `moved` never fires.
   - `is-active` and `is-visible` stay on the old slide, because slide classes update on `moved`.
   - `aria-hidden` stops updating.
   - With `waitForTransition: true`, the slider ignores every `go()` and drag from then on.
   - `Move.reposition()` stops running on resize.
   - With `wheel: true, releaseWheel: true`, the wheel never hands scrolling back to the page.
3. **`cancel()` must stop the animation immediately** and leave the list where `Move.cancel()` pinned it. In the built-in CSS path it does not call `done`.
4. **Handle re-entry.** If `start()` runs while an animation is in flight, store the new `done` and drop the old one. Clear any timers from the previous call.
5. **Read options inside `start()`.** Breakpoints, `reducedMotion` and the temporary override `Drag` applies during a release all change `options.speed` in place (see [Interaction with options](#interaction-with-options)).
6. **Clean up in `destroy()`:** timers, `requestAnimationFrame` loops, and manual `addEventListener`s (classic). `Elements.destroy()` strips the list's `style` attribute, and each slide's original `style` attribute is restored on destroy.

## Built-in Slide transition

The default for `type: 'slide'` and `type: 'loop'` (`transitions/Slide/Slide.ts`, types stripped):

```js
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

## Built-in Fade transition

The default for `type: 'fade'` (`transitions/Fade/Fade.ts`):

```js
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

The visible fade is pure CSS. `Elements` adds `splide__track--fade` to the track, and the stylesheet has:

```css
.splide__track--fade > .splide__list > .splide__slide { margin: 0 !important; opacity: 0; z-index: 0; }
.splide__track--fade > .splide__list > .splide__slide.is-active { opacity: 1; z-index: 1; }
```

Because `done` runs right away, `moved` fires about 0ms after `move`. That swaps the `is-active` class, and the opacity transition (set in `start`) then plays for `speed` ms **after** `moved`. Fade ignores `rewindSpeed` and `useScroll`. Its `cancel` does nothing. With `type: 'fade'`, `Move.translate()` does nothing and no clones are created.

## Interaction with options

The built-in transitions read options at `start()` time. A custom transition only gets these behaviours if it implements them.

| Option | Default | How it affects transitions |
|---|---|---|
| `type` | `'slide'` | Chooses the default transition (`'fade'` gives Fade, anything else gives Slide) when no transition is passed to `mount()`. A custom transition is used for every type. With `'fade'`, `Move.translate()` is a no-op, so a translate-based transition shows nothing. |
| `speed` | `400` | Duration in ms. `0` means jump: Slide checks `speed >= 1`. |
| `rewindSpeed` | not set, so `speed` is used | Slide only, `type: 'slide'` only. Used when moving from index `0` to an index `>= Controller.getEnd()`, or from `>= end` to `0`. That includes a plain `go( end )` from 0, not only rewinds (verified: `go( 5 )` from 0 in a 6-slide slider took the `rewindSpeed`). |
| `easing` | `'cubic-bezier(0.25, 1, 0.5, 1)'` | CSS timing function for the CSS path (Slide) and the opacity fade (Fade). Ignored by the Scroll path. |
| `easingFunc` | `t => 1 - Math.pow( 1 - t, 4 )` | JS easing used by `Scroll.scroll()`: `useScroll` transitions, `drag: 'free'`, and any transition that calls `Scroll.scroll()`. |
| `useScroll` | not set | Slide only: animate with `Scroll.scroll()` (requestAnimationFrame) instead of a CSS transition. |
| `reducedMotion` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Merged into `options` while `(prefers-reduced-motion: reduce)` matches, so `speed` becomes 0 and Slide jumps. **`Drag` lifts it during a release:** it calls `Media.reduce( false )`, then `Controller.go()` (which calls `start()` synchronously), then `Media.reduce( true )`. Drag releases therefore animate at normal speed, as "essential" motion. Read `options.speed` synchronously in `start()` and keep the value. Reading it later (in a timeout) gives 0 again. |
| `waitForTransition` | not set (falsy) | When `true`, `Controller.isBusy()` is true while the state is `MOVING` or `SCROLLING`: `go()` is ignored, pointerdown is prevented, and `reposition()` is skipped. When falsy (the default), `start()` can be re-entered and drags call `cancel()` mid-move. |
| `rewind` | `false` | Only changes which index `go()` resolves. The transition just gets `index`. |
| `drag: 'free'` | | Releases use `Controller.scroll()` and bypass the Transition entirely. |
| `focus`, `trimSpace`, `perPage`, `padding`, `gap`, `direction` | | All handled by `Move.toPosition( index, true )` and `Move.translate()`. Use them rather than computing positions yourself. `focus: 'center'` with `trimSpace: true` makes several indices map to the same position, so the zero-distance guard is required. |

## useScroll and the Scroll component

`useScroll: true` is the built-in alternative to a CSS transition. The Slide transition then calls `Components.Scroll.scroll( destination, speed, false, done )`. For a custom easing curve you usually don't need a custom transition:

```js
new Splide( '.splide', {
  useScroll : true,
  speed     : 600,
  easingFunc: t => ( t < 0.5 ? 4 * t * t * t : 1 - Math.pow( -2 * t + 2, 3 ) / 2 ), // easeInOutCubic
} ).mount();
```

`Scroll.scroll( destination, duration?, snap?, onScrolled? )` (`components/Scroll/Scroll.ts`):

- Each frame, it moves the list from the current position toward `destination` along `easingFunc` (default easeOutQuart), using `RequestInterval( duration, onEnd, update, 1 )`. It translates **without** `preventLoop`, so in loop mode positions past the ends are shifted and `sh` is emitted.
- `duration` falsy (including `0`) does **not** mean instant. It means automatic: `max( distance / 1.5, 800 )` ms. If the distance is under 1px, the duration is 0 and it ends on the next frame. To jump, use `Move.jump()`.
- `snap = true` snaps the destination to the nearest slide (in `slide` type, only while not past the limits).
- In `slide` type, overshooting the limits applies friction and bounces back to the limit over 600ms.
- It sets the state to `SCROLLING` and emits `scroll`. On completion: state `IDLE`, then `onScrolled()`, then `scrolled`. Used as a transition, `onScrolled` is `done`, so the observed order is `move`, `scroll`, `moved`, `scrolled`. The state also goes `MOVING`, `SCROLLING`, then `IDLE`.
- `Scroll.cancel()`: if a scroll is running, it stops it **and calls `onEnd`**, which means the callback (your `done`) runs and `moved` and `scrolled` fire. This differs from the CSS path, where a cancelled move never emits `moved`. Scroll also runs `cancel` on `updated` and `refresh`.
- On every `move` event, Scroll stops an in-flight scroll **without** calling its callback, so a superseded `done` is dropped.
- `Scroll.destroy` stops the interval.
- `Controller.scroll( destination, duration?, snap?, callback? )` wraps `Scroll.scroll` and afterwards sets the index to the slide closest to where it stopped. This is what `drag: 'free'` uses. It emits no `move`/`moved`.

A custom transition can call `Scroll.scroll()` itself to get the JS animation with its own duration rules ([Example 4](#example-4-scroll-driven-transition-with-distance-based-duration)).

## Registration

Pass the constructor as the **second** argument to `mount()`:

```js
// ESM
import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
import { MyTransition } from './my-transition.js';   // your module (http(s) only), or define it inline

new Splide( '#splide' ).mount( {}, MyTransition );
```

```html
<!-- Classic -->
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function MyTransition( Splide, Components, options ) { /* ... */ return { start, cancel }; }
  new Splide( '#splide' ).mount( {}, MyTransition );
</script>
```

- **With extensions:** `mount( { SlideCounter, KeyboardShortcuts }, MyTransition )`. The Transition is still constructed and mounted last.
- **`{ Transition: MyTransition }` in the first argument is ignored.** `mount()` assigns `{ Transition: this._T }` after the extensions (verified). Only the second argument works.
- **Remounts keep it.** `this._T = Transition || this._T || ( fade ? Fade : Slide )`. A later `mount()` without arguments (manual, or a breakpoint remount after `destroy: true`) reuses your transition (verified).
- **Per instance.** Each `new Splide()` needs its own `mount( ..., MyTransition )`. You can't set a transition globally through `Splide.defaults`.
- The built-in `Slide` and `Fade` constructors are not exported by either build, so a custom transition can't wrap them. Example 2 re-implements Slide as a starting point.

## Example 1: the docs' CSS transition

This is the docs' example converted. It hardcodes 800ms, ignores `speed`, `easing`, `rewindSpeed`, `reducedMotion` and `useScroll`, and has no zero-distance guard, so it gets stuck in `MOVING` when the position doesn't change (see the contract above). Use it to learn the mechanism. For real use, take Example 2.

ESM:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">

<section class="splide" aria-label="Custom transition example">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide">01</li>
      <li class="splide__slide">02</li>
      <li class="splide__slide">03</li>
    </ul>
  </div>
</section>

<script type="module">
  import { Splide, EventInterface } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function MyTransition( Splide, Components ) {
    const { bind } = EventInterface( Splide );
    const { Move } = Components;
    const { list } = Components.Elements;

    let endCallback;

    function mount() {
      bind( list, 'transitionend', e => {
        if ( e.target === list && endCallback ) {
          // Removes the transition property
          cancel();

          // Calls the `done` callback
          endCallback();
        }
      } );
    }

    function start( index, done ) {
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

    return {
      mount,
      start,
      cancel,
    };
  }

  new Splide( '.splide' ).mount( {}, MyTransition );
</script>
```

Classic: bind `transitionend` yourself and remove it in `destroy()`:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function MyTransition( Splide, Components ) {
    const { Move } = Components;
    const { list } = Components.Elements;
    let endCallback;

    function onTransitionEnd( e ) {
      if ( e.target === list && endCallback ) {
        cancel();
        endCallback();
      }
    }

    function mount() {
      list.addEventListener( 'transitionend', onTransitionEnd );
    }

    function start( index, done ) {
      const destination = Move.toPosition( index, true );
      list.style.transition = 'transform 800ms cubic-bezier(.44,.65,.07,1.01)';
      Move.translate( destination );
      endCallback = done;
    }

    function cancel() {
      list.style.transition = '';
    }

    function destroy() {
      list.removeEventListener( 'transitionend', onTransitionEnd );
    }

    return { mount, start, cancel, destroy };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide' ).mount( {}, MyTransition );
  } );
</script>
```

Verified: `go( 2 )` emitted `moved` about 826ms after `move`, and the list ended at `translateX(-1936px)`, which equals `Move.toPosition( 2, true )`.

## Example 2: CSS slide transition that respects options

This is a production template: the built-in Slide behaviour plus a few hardening steps. Change the `transition` string or add effects here. It:

- uses `speed`, `easing`, `rewindSpeed` and `useScroll`;
- jumps on zero distance or zero speed, which also covers `reducedMotion`;
- passes `preventLoop` like the built-in;
- checks `propertyName` so that other transitioned properties on the list don't end the move early;
- clears the stored `done` so it can't run twice;
- has a fallback timer in case `transitionend` never fires;
- cleans up on destroy.

Verified with `speed: 300, rewind: true, rewindSpeed: 900`: moves took about 310ms, and the rewind from 5 to 0 took about 910ms. `speed: 0` emitted `moved` immediately. `focus: 'center'` going 0 to 1 (no distance) emitted `moved` immediately. Loop moves across clones were correct. `useScroll: true` emitted `move`, `scroll`, `moved`, `scrolled`. `Move.cancel()` mid-move cleared the transition with no `moved`, and the next `go()` worked.

ESM:

```html
<script type="module">
  import { Splide, EventInterface, SLIDE } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function CssSlide( splide, Components, options ) {
    const { bind } = EventInterface( splide );
    const { Move, Controller, Scroll } = Components;
    const { list } = Components.Elements;
    let endCallback = null;
    let timer       = 0;

    function mount() {
      bind( list, 'transitionend', e => {
        if ( e.target === list && e.propertyName === 'transform' ) {
          finish();
        }
      } );
    }

    function start( index, done ) {
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
      timer = setTimeout( finish, speed + 100 );      // safety net if transitionend is missed
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

    function getSpeed( index ) {
      const { rewindSpeed } = options;

      if ( splide.is( SLIDE ) && rewindSpeed ) {
        const prev = Controller.getIndex( true );
        const end  = Controller.getEnd();

        if ( ( prev === 0 && index >= end ) || ( prev >= end && index === 0 ) ) {
          return rewindSpeed;
        }
      }

      return options.speed;
    }

    function destroy() {
      clearTimeout( timer );
      endCallback = null;                             // the list's style attribute is reset by Elements on destroy
    }

    return { mount, start, cancel, destroy };
  }

  new Splide( '.splide', { speed: 500, rewind: true, rewindSpeed: 1200 } ).mount( {}, CssSlide );
</script>
```

Classic: `'slide'` instead of `SLIDE`, and a manual `transitionend` listener removed in `destroy()`:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function CssSlide( splide, Components, options ) {
    const { Move, Controller, Scroll } = Components;
    const { list } = Components.Elements;
    let endCallback = null;
    let timer       = 0;

    function onTransitionEnd( e ) {
      if ( e.target === list && e.propertyName === 'transform' ) {
        finish();
      }
    }

    function mount() {
      list.addEventListener( 'transitionend', onTransitionEnd );
    }

    function start( index, done ) {
      const destination = Move.toPosition( index, true );
      const position    = Move.getPosition();
      const speed       = getSpeed( index );

      if ( Math.abs( destination - position ) < 1 || speed < 1 ) {
        Move.jump( index );
        done();
        return;
      }

      if ( options.useScroll ) {
        Scroll.scroll( destination, speed, false, done );
        return;
      }

      endCallback = done;
      list.style.transition = `transform ${ speed }ms ${ options.easing }`;
      Move.translate( destination, true );

      clearTimeout( timer );
      timer = setTimeout( finish, speed + 100 );
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
      Scroll.cancel();
    }

    function getSpeed( index ) {
      const { rewindSpeed } = options;

      if ( splide.is( 'slide' ) && rewindSpeed ) {
        const prev = Controller.getIndex( true );
        const end  = Controller.getEnd();

        if ( ( prev === 0 && index >= end ) || ( prev >= end && index === 0 ) ) {
          return rewindSpeed;
        }
      }

      return options.speed;
    }

    function destroy() {
      clearTimeout( timer );
      endCallback = null;
      list.removeEventListener( 'transitionend', onTransitionEnd );
    }

    return { mount, start, cancel, destroy };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', { speed: 500, rewind: true, rewindSpeed: 1200 } ).mount( {}, CssSlide );
  } );
</script>
```

Notes:

- The fallback timer is not in the built-in. Without it, `transitionend` is the only completion signal, and if it is missed the move never completes. If it fires first (heavy main-thread load), removing the transition snaps the list to the destination it already has.
- For a different motion, change only the `transition` string, for example `transform ${ speed }ms steps(6)` or a custom `cubic-bezier()`, or toggle a class on `splide.root` in `start` and `finish` to drive extra CSS (scale or dim the slides while moving).
- Keep `Move.translate( destination, true )`. Don't write `list.style.transform` yourself: `translate` resolves `translateX` or `translateY` for the direction and skips fade.

## Example 3: zoom crossfade for type fade

A custom transition for `type: 'fade'`. `'fade'` keeps the fade CSS (`margin: 0`, stacking z-index), makes `Move.translate` a no-op, and creates no clones. The built-in Fade's stacking `init()` doesn't run once you replace it, so this does the stacking itself. Opacity and scale are set inline on every slide, which overrides the stylesheet's `.is-active` opacity, so the incoming slide fades in during the move instead of after `moved`. `done` runs when the animation has finished, which is not what the built-in Fade does. Verified: about 150ms into a 400ms move, slide 1's computed opacity was 0.85 while slide 0 was still `is-active`. `moved` fired at about 401ms. Afterwards the inline opacities were `0,1,0,0,0,0`, slide 1 had `translateX(-100%) scale(1)`, and `is-active` was on slide 1.

ESM:

```html
<script type="module">
  import { Splide, EventInterface, EVENT_MOUNTED, EVENT_REFRESH } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';

  function ZoomFade( splide, Components, options ) {
    const { on } = EventInterface( splide );
    const { Slides } = Components;
    let endCallback = null;
    let timer       = 0;

    function mount() {
      // Slides are re-registered on refresh, so restack then too (the built-in Fade does the same).
      on( [ EVENT_MOUNTED, EVENT_REFRESH ], () => {
        Slides.style( 'transition', '' );
        show( splide.index );
      } );
    }

    function show( index ) {
      const scale = options.zoomScale || 1.08;            // custom option

      Slides.forEach( Slide => {
        const active = Slide.index === index;
        Slide.style( 'opacity', active ? 1 : 0 );
        // translateX(-100% * i) stacks every slide at the head of the list, like the built-in Fade.
        Slide.style( 'transform', `translateX(-${ 100 * Slide.index }%) scale(${ active ? 1 : scale })` );
      } );
    }

    function start( index, done ) {
      const { speed, easing } = options;
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
      timer = setTimeout( finish, speed );                // done after the effect, unlike the built-in Fade
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

  new Splide( '.splide', { type: 'fade', rewind: true, speed: 600, zoomScale: 1.1 } ).mount( {}, ZoomFade );
</script>
```

Classic: identical except for `splide.on` with a namespace instead of `EventInterface`:

```html
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function ZoomFade( splide, Components, options ) {
    const { Slides } = Components;
    let endCallback = null;
    let timer       = 0;

    function mount() {
      splide.on( 'mounted.zoomFade refresh.zoomFade', () => {
        Slides.style( 'transition', '' );
        show( splide.index );
      } );
    }

    function show( index ) {
      const scale = options.zoomScale || 1.08;

      Slides.forEach( Slide => {
        const active = Slide.index === index;
        Slide.style( 'opacity', active ? 1 : 0 );
        Slide.style( 'transform', `translateX(-${ 100 * Slide.index }%) scale(${ active ? 1 : scale })` );
      } );
    }

    function start( index, done ) {
      const { speed, easing } = options;
      clearTimeout( timer );

      if ( speed < 1 ) {
        Slides.style( 'transition', '' );
        show( index );
        endCallback = null;
        done();
        return;
      }

      Slides.style( 'transition', `opacity ${ speed }ms ${ easing }, transform ${ speed }ms ${ easing }` );
      show( index );
      endCallback = done;
      timer = setTimeout( finish, speed );
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
      Slides.style( 'transition', '' );
      endCallback = null;
    }

    return { mount, start, cancel, destroy: cancel };
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', { type: 'fade', rewind: true, speed: 600, zoomScale: 1.1 } ).mount( {}, ZoomFade );
  } );
</script>
```

Notes:

- `type: 'fade'` is required. With `'slide'` or `'loop'` the slides sit side by side, without fade CSS or stacking.
- `Slides.style( prop, value )` styles every slide (pass `true` as the third argument to style `.splide__slide__container` instead). `Slide.style()` styles one slide.
- The `z-index` of `.is-active` still switches on `moved`. Links in the incoming slide become clickable once the move completes.

## Example 4: Scroll-driven transition with distance-based duration

This delegates the animation to the Scroll component, so it runs on requestAnimationFrame with `easingFunc`, and scales the duration with distance: long jumps (pagination to a far page) take longer than one-slide steps. It needs no helpers, so the code is the same in both builds. Verified with `speed: 400` on a 953px-wide slider: one slide took about 649ms and four slides about 1214ms (capped at `3 * speed`). Each move emitted `move`, `scroll`, `moved`, `scrolled`.

```html
<!-- Classic; for ESM use: import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js'; inside <script type="module"> -->
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script>
  function DistanceScroll( splide, Components, options ) {
    const { Move, Scroll } = Components;

    function start( index, done ) {
      const destination = Move.toPosition( index, true );
      const distance    = Math.abs( destination - Move.getPosition() );

      if ( distance < 1 || options.speed < 1 ) {
        Move.jump( index );
        done();
        return;
      }

      // 1.5px per ms, clamped to [speed, 3 * speed]. Never pass 0: Scroll treats 0 as "auto" (>= 800ms).
      const duration = Math.min( Math.max( distance / 1.5, options.speed ), options.speed * 3 );
      Scroll.scroll( destination, duration, false, done );
    }

    function cancel() {
      Scroll.cancel();   // note: runs the pending done (moved fires), unlike the CSS path
    }

    return { start, cancel };   // Scroll stops its own interval on destroy
  }

  document.addEventListener( 'DOMContentLoaded', () => {
    new Splide( '.splide', {
      speed     : 400,
      easingFunc: t => 1 - Math.pow( 1 - t, 3 ),   // optional: easeOutCubic instead of the default easeOutQuart
    } ).mount( {}, DistanceScroll );
  } );
</script>
```

## Pitfalls checklist

- Call `done()` on every path, synchronously when there is nothing to animate. Guard with `Math.abs( Move.toPosition( index, true ) - Move.getPosition() ) < 1 || speed < 1`.
- Read `options.speed`, `options.easing` and `options.rewindSpeed` inside `start()`, not in the constructor and not later in a timeout.
- Use `Move.toPosition( index, true )` and `Move.translate( position, true )`. Don't compute pixel offsets or write `transform` on the list yourself.
- Filter `transitionend` with `e.target === list`, and preferably `e.propertyName === 'transform'`.
- Make `cancel()` stop the animation instantly, and make `start()` replace an in-flight `done`.
- `type: 'fade'` turns `Move.translate()` into a no-op and removes the built-in stacking once you replace Fade.
- Pass the transition as `mount( extensions, Transition )`. `{ Transition }` in the extensions object is ignored.
- `Scroll.scroll( dest, 0 )` is an automatic duration (800ms minimum), not a jump.
- Classic scripts: remove your own `transitionend` listener in `destroy()`.

## Docs vs source

- **Lifecycle.** The docs say: `start()` and `cancel()` required, `mount()` and `destroy()` optional. The source also calls an optional `setup()`, and passes `destroy( completely )` a boolean. The Transition is constructed and mounted last and destroyed first.
- **When `cancel()` runs.** The docs say: when a transition is interrupted, for example a drag during a transition with `waitForTransition: false`. The source calls it through `Move.cancel()`, both from `Drag` on every pointerdown while the controller isn't busy (even with no transition running) and from `Move.move()` before a loop shift. In the CSS path a cancelled move never emits `moved`. In the Scroll path (`useScroll`), `Scroll.cancel()` does run `done`.
- **When `done` runs.** The docs say `done` "must be called after the transition ends". The built-in Fade calls it on the next macrotask (`setTimeout`), so `moved` fires before the fade has visibly finished. The built-in Slide calls it synchronously when there is nothing to animate.
- **Docs example.** It calls `Move.translate( destination )` without `preventLoop`, while the built-in passes `true`. The two are equivalent for real indices. It also hardcodes `800ms` and has no zero-distance guard. Verified: with `perPage: 3, focus: 'center'` going 0 to 1 it never calls `done`, so the slider stays `MOVING` and `moved` never fires.
- **`waitForTransition` loop points.** The options docs say that "Even if `false`, the carousel forcibly waits for the transition on the loop points". The 4.1.x source has no such check. The only busy test is `Controller.isBusy()`, which needs `waitForTransition: true`. At loop points, `Move.move()` cancels the running transition and shifts the list instead.
- **`rewindSpeed`.** Documented as "the transition speed on rewind". In the source it applies (slide type only) to any move between index 0 and an index `>= getEnd()`, in either direction, including a direct `go( end )`. Fade ignores it.
- **`useScroll`.** It is not on the options page. The type comment says "Determines whether to use the Transition component or not". In the source, only the built-in Slide transition reads it. Custom transitions ignore it unless they implement it, as Example 2 does.
- **`easingFunc`.** The docs say it is "for the drag free mode". In the source, `Scroll` uses it for every `Scroll.scroll()`, including `useScroll` transitions and custom transitions that call it.
