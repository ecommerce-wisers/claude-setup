# Drag component

Source: https://splidejs.com/components/drag/
Source file: `src/js/components/Drag/Drag.ts` (constants in `src/js/components/Drag/constants.ts`), Splide 4.1.3 source. The published 4.1.4 build is identical to 4.1.3 apart from the version banner.

Purpose: watches mouse and touch input on the track, moves the carousel while the user drags, and decides where to go on release (flick).

Access: `splide.Components.Drag` (filled in by `splide.mount()`).

## Properties

None.

## Methods

| Signature | Returns | Description |
|---|---|---|
| `disable( disabled )` | `void` | `true` disables dragging, `false` enables it. Reset from `options.drag` on every `mounted`/`updated` event. |
| `isDragging()` | `boolean` | (undocumented, from source) `true` while the user is actually dragging the carousel. |

Lifecycle member `mount()` is also on the object. Splide calls it.

### disable()

`disable( disabled: boolean ): void`. The docs params table names the argument `value`; the interface names it `disabled`.

| Param | Description |
|---|---|
| `disabled` | `true` to disable the component, `false` to enable it. |

Behaviour from source:

- While disabled, `mousedown`/`touchstart` on the track are ignored, and the click blocker that suppresses clicks after a drag is off.
- The listeners stay bound. `drag: false` also mounts the component and only sets the disabled flag.
- The flag is overwritten by `disable( ! options.drag )` on every `mounted` and `updated` event. A breakpoint change or `splide.options = { ... }` therefore undoes a manual `disable()`. For a persistent change, set the option (`splide.options = { drag: false }`), or call `disable()` again from an `updated` handler registered after `mount()`, so it runs after the component's own handler.

```js
const splide = new Splide( '.splide' ).mount();
const Drag = splide.Components.Drag;

Drag.disable( true );  // stop drag/swipe, e.g. while an image is zoomed
Drag.disable( false ); // allow it again

// Keep it disabled across breakpoint/options updates:
let locked = true;
splide.on( 'updated', function () {
  if ( locked ) {
    Drag.disable( true );
  }
} );
```

### isDragging()

(undocumented, from source) `isDragging(): boolean`

Returns the internal `dragging` flag:

- It becomes `true` once the pointer has moved past `dragMinThreshold` along the slider axis, and further along that axis than across it.
- It is also `true` right after pointer-down if the carousel was already moving or scrolling (the user grabbed a moving carousel).
- It goes back to `false` on pointer-up.
- It is `undefined` (falsy) before the first pointer-down.

The release move is started synchronously before the flag is cleared. Inside a `move` (or, in free mode, `scroll`) handler, `isDragging()` therefore tells whether that move came from a drag. By the time `moved` fires, or a `click` handler runs, it is already `false`.

```js
splide.on( 'move', function ( index ) {
  const byDrag = splide.Components.Drag.isDragging();
  console.log( 'moving to', index, byDrag ? '(drag/swipe)' : '(arrow, dot, API, ...)' );
} );
```

### Drag behaviour (from source)

- Start: `touchstart`/`mousedown` on the track. For mouse, only the primary button (`e.button === 0`). Move and up listeners go on the track for touch and on `window` for mouse. All pointer listeners use `{ passive: false, capture: true }`.
- Ignored targets: `.splide__pagination__page`, `.splide__arrow`, and elements matching the `noDrag` selector. The match uses `Element.matches()` on the event target itself, not its ancestors, so use a selector that also covers descendants, e.g. `noDrag: '.no-drag, .no-drag *'`.
- If `Controller.isBusy()` is `true` on pointer-down (only possible with `waitForTransition: true`), the event is cancelled with `preventDefault()` and `stopPropagation()`, and no drag starts.
- On pointer-down, any running transition (`Move.cancel()`) and scroll (`Scroll.cancel()`) are stopped where they are.
- The first pointer-move sets the state to `DRAGGING` (`Splide.STATES.DRAGGING`, 6) and emits `drag`, even before the threshold is reached.
- While dragging, the list follows the pointer and `dragging` is emitted on each move. Past the edges of a `slide` type carousel, the movement is divided by 5 (friction).
- Release: if the state was `DRAGGING`, it is set to `IDLE` and `dragged` is emitted. If a drag actually happened, the destination is computed:
  - Velocity is taken over the last 200 ms. A pause of 200 ms or more while holding gives 0 velocity. Velocity is also 0 past the edges, except for `loop`.
  - Destination is `position + sign( v ) * min( |v| * flickPower, listSize * flickMaxPages )`. `flickPower` defaults to 600 and `flickMaxPages` to 1. There is no page cap in free mode.
  - `drag: 'free'`: `Controller.scroll( destination, 0, options.snap )`.
  - `fade` type: `go( '<' / '>' )` when `rewind` and `rewindByDrag` are set, otherwise `go( '-' / '+' )`.
  - `slide` type past an edge with `rewind` and `rewindByDrag`: `go( '>' )` past the end, `go( '<' )` past the start.
  - Otherwise: `Controller.go( Controller.toDest( destination ), true )`.
  - The release move ignores `reducedMotion` (it temporarily removes the reduced-motion overrides), so the snap animation still plays for those users.
- After a real drag, `click` events inside the track are cancelled in the capture phase, so links and buttons in slides do not fire on release.
- Native `dragstart` (image/link dragging) on the track is always prevented.

## Events emitted / listened

Emitted:

| Event | Arguments | When |
|---|---|---|
| `drag` | `()` | First pointer-move after pointer-down (state becomes `DRAGGING`). |
| `dragging` | `()` | Every pointer-move while the carousel follows the pointer. |
| `dragged` | `()` | Pointer-up after a `drag` (state returns to `IDLE`). |

The release then calls `Controller.go()` or `Controller.scroll()`, which leads to `move`/`moved` or `scroll`/`scrolled`.

Listened:

| Source | Event | Action |
|---|---|---|
| Splide | `mounted`, `updated` | `disable( ! options.drag )`, and re-reads free mode (`drag === 'free'`). |
| DOM track | `touchstart`, `mousedown` | Start of a drag. |
| DOM track or window | `touchmove`, `mousemove` | Drag update. |
| DOM track or window | `touchend`, `touchcancel`, `mouseup`, `click` | Drag end. |
| DOM track | `click` (capture) | Cancelled after a drag. |
| DOM track | `dragstart` | Always prevented. |

## Options read

| Option | Default | Use |
|---|---|---|
| `drag` | `true` | `false` disables, `true` enables, `'free'` is free-drag mode (scroll with momentum). |
| `snap` | (none) | In free mode, snap to the closest slide after the flick. |
| `dragMinThreshold` | `10` | Number: touch threshold in px (mouse threshold 0). Object `{ mouse, touch }`: separate thresholds (missing touch falls back to 10, missing mouse to 0). |
| `flickPower` | `600` | Velocity multiplier for the flick distance. |
| `flickMaxPages` | `1` | Max flick distance in list widths (not used in free mode). |
| `noDrag` | (none) | Selector of targets that cannot start a drag. |
| `rewind`, `rewindByDrag` | (none) | Rewind when dragging past the edges (`slide` and `fade`). |
| `type` | `'slide'` | Friction and limit behaviour (`slide`), wrapping (`loop`), step behaviour (`fade`). |
| `direction` | `'ltr'` | Axis and sign (through Direction). |
| `waitForTransition` | (none) | Through `Controller.isBusy()`. |
| `reducedMotion` | `{ speed: 0, rewindSpeed: 0, autoplay: 'pause' }` | Temporarily removed for the release move. |
