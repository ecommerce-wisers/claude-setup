# URL Hash Navigation extension (React)

Linkable slides with `@splidejs/splide-extension-url-hash` in React Splide: markup, how the hash and history change, a replace-history alternative, and React-specific behavior.

Source: https://splidejs.com/extensions/url-hash/
Package: @splidejs/splide-extension-url-hash@0.3.0 (peer: none declared; built against @splidejs/splide ^4.0.0, its devDependency; tested with React Splide 0.7.12 and React 19)

Contents: [Install](#install-and-mount) · [Markup](#markup) · [How it works](#how-it-works) · [Methods and events](#methods-and-events) · [Examples](#examples) · [React notes](#react-notes) · [Caveats](#caveats)

The extension ties the carousel to the URL hash. It parses the hash to choose the start index, and it updates the hash every time the carousel moves. It also follows later hash changes, such as links and the Back/Forward buttons.

## Install and mount

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-url-hash
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { URLHash } from '@splidejs/splide-extension-url-hash';
import '@splidejs/react-splide/css';

<Splide options={ options } extensions={ { URLHash } }>...</Splide>
```

- The export is `URLHash` (capital `URL`). The package ships no CSS and has no options of its own.
- Pass it only to the carousel that uses hashes: it rewrites the URL fragment on every carousel it is mounted on (see [Caveats](#caveats)).
- TypeScript on Linux (CI, Docker): the package's type entry points at a folder with the wrong letter case, so the import is typed `any` there (no error with `skipLibCheck: true`). Details in `references/extensions/packages.md`.

## Markup

Give each slide its hash value with the `data-splide-hash` attribute on `<SplideSlide>`, which passes it to the `<li>`. Write the value without `#`.

```tsx
<Splide options={ { label: 'Chapters' } } extensions={ { URLHash } }>
  <SplideSlide data-splide-hash="slide01">...</SplideSlide>
  <SplideSlide data-splide-hash="slide02">...</SplideSlide>
  <SplideSlide data-splide-hash="slide03">...</SplideSlide>
</Splide>
```

| Attribute | On | Value |
|---|---|---|
| `data-splide-hash` | `.splide__slide` (real slides; clones are ignored) | The fragment for that slide, without `#`. It is matched exactly and case-sensitively against `location.hash` with its first `#` removed. |

The core `start` option is the fallback when the URL hash matches no slide.

## How it works

1. **Before mount** (the extension's `setup()`): it reads `location.hash`. If it equals a slide's `data-splide-hash`, the start index becomes that slide's index and overrides `start`. Otherwise the index is `options.start || 0`.
2. **On every `active` event**, when a slide becomes active:
   - If the slide has `data-splide-hash`, the extension sets `location.hash = value`. This **adds a browser history entry**, and Back/Forward step through the slides.
   - If the slide has no hash, it calls `history.replaceState( null, null, ' ' )`, which removes the fragment from the URL (path and query stay) without adding a history entry. It also replaces `history.state` with `null`.
3. **On `window` `hashchange`**: if the new hash matches a slide, it calls `splide.go( index )`. Hashes that match nothing are ignored. So `<a href="#slide03">` links anywhere on the page move the carousel.
4. **On mount**: the first `active` event writes the initial slide's hash, which adds one history entry (none if the URL already has that hash). Exception: if an extension mounted before URL Hash calls `Slides.update()` during its own mount, that first `active` fires before URL Hash listens, so no hash is written at mount. Video and Grid do this. Extensions mount in the key order of the `extensions` object.

Tested in React: opening the page with `#slide-05` started on that slide, the next arrow set `#slide-06` and added one history entry, a `<a href="#slide-09">` link moved the carousel, and Back returned to `slide-06`.

## Methods and events

The extension has no public methods. `splide.Components.URLHash` has only the lifecycle functions `setup()` and `mount()`. The `hashchange` listener is removed when the carousel is destroyed (on unmount).

The extension emits no events of its own. It listens to the core `active` event and the window `hashchange` event. For your own logic, use the core event props: `onActive` receives `( splide, slide )`, and `slide.slide.dataset.splideHash` is the slide's hash (first example).

## Examples

### Linkable slides (official demo settings)

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { URLHash } from '@splidejs/splide-extension-url-hash';
import '@splidejs/react-splide/css';

const slides = Array.from( { length: 9 }, ( _, i ) => `slide-${ String( i + 1 ).padStart( 2, '0' ) }` );

export default function LinkableSlides() {
  const [ active, setActive ] = useState( '' );

  return (
    <>
      <nav>
        <a href="#slide-01">First</a> <a href="#slide-05">Fifth</a> <a href="#slide-09">Last</a>
      </nav>

      <Splide
        id="hash-carousel"
        options={ {
          label      : 'Linkable slides',
          perPage    : 3,
          perMove    : 1,
          height     : '9rem',
          focus      : 'center',
          trimSpace  : false,
          breakpoints: {
            600: {
              perPage: 2,
              height : '6rem',
            },
          },
        } }
        extensions={ { URLHash } }
        onActive={ ( splide, slide ) => setActive( slide.slide.dataset.splideHash ?? '' ) }
      >
        { slides.map( hash => (
          <SplideSlide key={ hash } data-splide-hash={ hash } style={ { background: '#eee', display: 'grid', placeItems: 'center' } }>
            { hash }
          </SplideSlide>
        ) ) }
      </Splide>

      <p id="active">Active: { active }</p>
    </>
  );
}
```

The docs code sample ends with `splide.mount( { URLHash );`, which is a syntax error; in React the extension goes in the `extensions` prop. With `perPage > 1`, keep `perMove: 1`, `focus: 'center'` and `trimSpace: false` as in the demo. Otherwise the end index is clamped and the last slides can never become active, so their hashes never appear and `go()` to them lands on an earlier slide. Plain `<a href="#...">` links work because the extension listens to `hashchange`; no click handler is needed.

### Replace history instead of pushing (custom extension)

The extension always pushes a history entry. To keep Back for leaving the page, use a small custom extension instead of URL Hash:

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { SplideProps } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';

type ExtensionConstructor = NonNullable<SplideProps[ 'extensions' ]>[ string ];

// Like URL Hash, but replaces the history entry instead of pushing one per slide.
const HashReplace: ExtensionConstructor = ( splide, components ) => {
  function indexOf( hash: string ): number {
    return components.Elements.slides.findIndex( slide => slide.getAttribute( 'data-splide-hash' ) === hash );
  }

  function onHashChange(): void {
    const index = indexOf( location.hash.slice( 1 ) );
    if ( index > -1 ) {
      splide.go( index );
    }
  }

  return {
    setup() {
      const index = indexOf( location.hash.slice( 1 ) );
      if ( index > -1 ) {
        components.Controller.setIndex( index );
      }
    },
    mount() {
      splide.on( 'active.hashReplace', slide => {
        const hash = slide.slide.getAttribute( 'data-splide-hash' );
        if ( hash ) {
          history.replaceState( history.state, '', `#${ hash }` ); // keeps a router's history.state
        }
      } );
      window.addEventListener( 'hashchange', onHashChange );
    },
    destroy() {
      splide.off( 'active.hashReplace' );
      window.removeEventListener( 'hashchange', onHashChange );
    },
  };
};

const extensions = { HashReplace };
const slides = [ 'intro', 'features', 'pricing', 'faq' ];

export default function Sections() {
  return (
    <Splide id="sections" options={ { label: 'Sections' } } extensions={ extensions }>
      { slides.map( hash => (
        <SplideSlide key={ hash } data-splide-hash={ hash }>{ hash }</SplideSlide>
      ) ) }
    </Splide>
  );
}
```

`replaceState` does not fire `hashchange`, so the carousel doesn't react to its own updates. Tested in production and development: opening `#pricing` started there, moving to the next slide set `#faq` without growing `history.length` and kept an existing `history.state` object, and setting `location.hash` still moved the carousel. The `destroy()` matters in React: unmounting (and StrictMode's test unmount) must remove the window listener.

## React notes

- **StrictMode.** In development the carousel mounts, is destroyed and mounts again. The second mount reads the hash the first one wrote and writes the same value again, which adds no history entry. Tested: `history.length` after load was the same in production and development.
- **SSR.** The server never sees the URL fragment, so server HTML can't depend on it. The extension picks the start slide from `location.hash` at mount, on the client. The active classes are added client-side anyway, so there is no hydration mismatch.
- **Routers.** Not tested with a router. A hash-based router (`HashRouter`, `createHashRouter`) owns the fragment too, so the two fight over it: don't combine them. With a history-based router, URL Hash's `location.hash = value` creates entries with `null` state, and a slide without a hash calls `replaceState( null, ... )`, which drops the router's `history.state`. If that causes trouble, use the `HashReplace` extension above, which keeps `history.state`.
- **Component state.** Read the active slide with `onActive` (handlers are captured at mount, so use the state setter or a ref; see `references/reactivity.md`). To move the carousel from React, set `location.hash` or call `ref.current?.go( index )`.
- **One carousel per page.** The rule from the caveats applies per page, not per component: two mounted carousels with URL Hash both write the fragment.

## Caveats

- **History.** Every slide change adds a history entry, and mounting usually adds one too, so Back steps through slides before it leaves the page. Use the custom extension above if you don't want that.
- **Slides without a hash clear the fragment.** URL Hash on a carousel whose active slide has no `data-splide-hash` removes the existing fragment at mount and on every move (`page#section2` becomes `page`). Pass it only to the carousel that uses hashes.
- **One carousel per page.** Each instance writes `location.hash` on its own moves, and an instance whose active slide has no hash clears the fragment. Values must be unique across the page.
- **Page jumps.** If a hash value equals the `id` of an element on the page, the browser scrolls to that element whenever the hash is set. Don't reuse element ids (Splide's slide ids look like `<root id>-slide01`, so a carousel with `id="hash-carousel"` has slides `hash-carousel-slide01`, which don't clash with `slide-01`).
- **Encoding.** `location.hash` returns percent-encoded text, so values with spaces or non-ASCII characters (`café`) never match after a reload. Use `[A-Za-z0-9_-]`.
- **`go()` right after mount.** The mount-time hash write queues a `hashchange`. In plain JavaScript, with the default `speed: 400`, that event arrives before the new slide's `active` (which fires on `moved`) and sends the carousel back: `splide.go( 2 )` right after `mount()` ended at index 0. In React, `go( 2 )` in `onMounted` and in a `useEffect` both ended on index 2, in production and development (tested). A `useLayoutEffect` runs synchronously right after mount like the plain JavaScript case (not tested). Prefer the URL hash or `options.start` for the initial slide.
- **Grid.** It doesn't work with the Grid extension. Grid's generated pages have no `data-splide-hash`, and URL Hash resolves the hash against the slides before Grid groups them.
- Works with `type: 'loop'`; `Elements.slides` holds only the real slides, so indexes match.
