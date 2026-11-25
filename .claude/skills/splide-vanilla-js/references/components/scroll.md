# Scroll component

Source: https://splidejs.com/components/scroll/
Source file: `src/js/components/Scroll/Scroll.ts` (constants in `src/js/components/Scroll/constants.ts`), Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: scrolls the carousel to any pixel position with requestAnimationFrame instead of the Transition component. Free drag mode (`drag: 'free'`), `Controller.scroll()` and the `useScroll` option use it.

Access: `splide.Components.Scroll` (filled in by `splide.mount()`).

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `scroll( destination, duration?, snap?, callback? )` | `void` | Animates the list to a position. Unlike `Controller.scroll()`, it does not update the current/previous index. |
| `cancel()` | `void` | Stops a running scroll and emits `scrolled` (and runs its callback). Does nothing if no scroll is running. |

Lifecycle members `mount()` and `destroy()` are also on the object. `destroy` stops the timer without emitting. Splide calls them.

### scroll()

Docs say `scroll( position: number, duration?: number, snap?: boolean, callback?: AnyFunction ): void` (the params table calls the first argument `destination`). The source interface is the same. The implementation takes a fifth, internal argument: `scroll( destination, duration?, snap?, onScrolled?, noConstrain?: boolean )`.

| Param | Description |
|---|---|
| `destination` | Target position in px (see Move: negative values move an `ltr` carousel forward). Docs say "A position or an index to scroll to. The `useIndex` must be `true` to specify the index." Source: there is no `useIndex` in v4; the value is always a position. Convert an index with `splide.Components.Move.toPosition( index, true )`. |
| `duration` | Optional. Duration in ms. If omitted or `0`, it is `max( abs( destination - current ) / 1.5, 800 )`. If the distance is under 1px, the duration is `0`. |
| `snap` | Optional. Replaces the destination with the position of the closest slide (`Controller.toDest()`), keeping whole slider-length offsets. Skipped for a `slide` type carousel that is currently past its limits. |
| `callback` | Optional. Called when the scroll ends, after the state is set back to `IDLE` and before `scrolled` is emitted. |
| `noConstrain` | (undocumented, from source; internal) If `true`, no bounce at the edges. Used by the bounce-back scroll itself. |

Behaviour from source:

- Stops any previous scroll first, without emitting `scrolled` for it.
- Sets the state to `SCROLLING` (`Splide.STATES.SCROLLING`, 5), emits `scroll`, and starts a one-shot timer.
- On each frame, the list moves toward `from + (to - from) * easing( rate )`. The easing is `options.easingFunc( t )` if given, otherwise easeOutQuart `1 - (1 - t)^4`.
- Bounce (`slide` type only): past the first or last limit, the movement is damped by a friction that multiplies by 0.6 each frame. When the per-frame step drops under 10px, a 600 ms scroll back to the nearest limit runs instead (same callback).
- Does not change `splide.index`, so arrows and pagination are not updated (pagination updates on `scroll`/`scrolled` but uses the unchanged index). Use `Controller.scroll()` when the index should follow.
- A `move` (for example from `go()`) stops any running scroll silently: no `scrolled` event, no callback.

```js
const splide = new Splide( '.splide' ).mount();
const { Scroll, Move } = splide.Components;

// Animate to slide 2 over 1s without touching the index:
Scroll.scroll( Move.toPosition( 2, true ), 1000, false, function () {
  console.log( 'scroll finished at', Move.getPosition() );
} );
```

### cancel()

`cancel(): void`

If a scroll is running: stops the timer, sets the state to `IDLE`, calls the pending callback and emits `scrolled`. Otherwise it does nothing. Drag calls it on pointer-down. The component calls it on `updated` and `refresh`.

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `scroll` | `()` | Each `scroll()` call (including the internal bounce-back scroll). |
| `scrolled` | `()` | When a scroll ends or is cancelled through `cancel()`. |

State changes: `SCROLLING` on start, `IDLE` on end.

Listened:

| Event | Action |
|---|---|
| `move` | Stops a running scroll silently (no `scrolled`, no callback). |
| `updated`, `refresh` | `cancel()` (emits `scrolled` if a scroll was running). |

## Options read

| Option | Use |
|---|---|
| `easingFunc` | Custom easing `( t ) => number` for scrolls. Default easeOutQuart. |
| `type` | Bounce and snap restrictions apply only to `slide`. |

`useScroll` (read by the Slide transition, not this component) makes normal `go()` transitions use `Scroll.scroll( position, speed, false, done )` instead of a CSS transition.
