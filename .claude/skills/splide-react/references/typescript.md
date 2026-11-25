# TypeScript with React Splide

Making the types resolve, keeping strict mode, and typing refs, handlers and extensions. Tested with TypeScript 5.9, `@types/react` 19.3, React Splide 0.7.12, Vite 8 and Next.js 16.4.

Contents: [The types don't resolve](#the-types-dont-resolve) · [Fix 1: one tsconfig line](#fix-1-one-tsconfig-line) · [Fix 2: patch the package](#fix-2-patch-the-package) · [Fixes that don't work](#fixes-that-dont-work) · [skipLibCheck](#skiplibcheck) · [Typing](#typing)

## The types don't resolve

With `"moduleResolution": "bundler"` (the default in Vite and Next.js projects), `"node16"` or `"nodenext"`, every import from React Splide fails:

```
error TS7016: Could not find a declaration file for module '@splidejs/react-splide'.
  There are types at '.../node_modules/@splidejs/react-splide/dist/types/index.d.ts', but this result
  could not be resolved when respecting package.json "exports".
```

React Splide's `package.json` has an `exports` map without a `types` condition, so TypeScript can't reach `dist/types/index.d.ts`. Under `strict`, the implicit `any` is an error, and `next build` stops at its type-check step.

`@splidejs/splide` and the extension packages are fine: core has a `types` condition, and the extensions have no `exports` map at all.

## Fix 1: one tsconfig line

```json
{
  "compilerOptions": {
    "resolvePackageJsonExports": false
  }
}
```

Tested: `tsc --noEmit` passes, and `next build` type-checks, compiles and prerenders. The runtime isn't affected, because bundlers don't read this option.

The trade-off: it applies to every package. TypeScript then finds types through the `types` field and the files on disk instead of `exports`. Most packages ship both (React, React DOM and Next.js's own subpath imports such as `next/navigation` and `next/image` type-check fine with it). If another package's types break, use fix 2 instead.

## Fix 2: patch the package

Add a `types` condition to React Splide's `exports`, then keep the change with your package manager's patch tool (`pnpm patch @splidejs/react-splide`, `yarn patch`, or `patch-package` with npm):

```diff
   "exports": {
     ".": {
+      "types": "./dist/types/index.d.ts",
       "require": "./dist/js/react-splide.cjs.js",
       "import": "./dist/js/react-splide.esm.js",
       "default": "./dist/js/react-splide.esm.js"
     },
```

`types` must come first in the object. Tested with `next build`. The patch has to be redone if the version changes, though 0.7.12 (September 2022) is still the latest release.

## Fixes that don't work

- **A tsconfig `paths` entry pointing at the `.d.ts`** (`"@splidejs/react-splide": [ "./node_modules/@splidejs/react-splide/dist/types/index.d.ts" ]`). `tsc` passes, but Next.js also applies `paths` when bundling, so the import resolves to the `.d.ts` and the build fails at prerender with `Element type is invalid: expected a string ... but got: undefined`. Vite breaks the same way when tsconfig paths are enabled (`resolve.tsconfigPaths` or `vite-tsconfig-paths`).
- **A shim that re-exports the subpath** (`declare module '@splidejs/react-splide' { export * from '@splidejs/react-splide/dist/types/index'; }`). The subpath isn't in `exports` either, so every named import fails with `has no exported member`.
- **`declare module '@splidejs/react-splide';`** silences the error by typing everything as `any`, so you lose every type.

## skipLibCheck

Keep `"skipLibCheck": true` (the Vite and Next.js templates already set it). With `false` and `@types/react` 19, React Splide's own declaration files fail:

```
node_modules/@splidejs/react-splide/dist/types/components/SplideSlide/SplideSlide.d.ts: error TS2503: Cannot find namespace 'JSX'.
```

React 19's types no longer declare the global `JSX` namespace, and the 0.7.12 declarations still use it. At runtime nothing is wrong.

## Typing

Import types from React Splide itself. It re-exports `Options` and its own prop and handler types:

```tsx
import { useRef } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import type { Options, SplideEventHandlers, SplideProps } from '@splidejs/react-splide';

// Core types without importing @splidejs/splide.
type SplideInstance = NonNullable<Splide[ 'splide' ]>;
type Extension = NonNullable<SplideProps[ 'extensions' ]>[ string ];
type Slide = Parameters<SplideEventHandlers[ 'onActive' ]>[ 1 ];

const options: Options = { perPage: 2, label: 'Typed' };

const LogIndex: Extension = ( splide, Components ) => ( {
  mount() {
    splide.on( 'moved', () => { document.body.dataset.index = String( Components.Controller.getIndex() ); } );
  },
} );

const onMoved: SplideEventHandlers[ 'onMoved' ] = ( splide, newIndex ) => {
  console.log( newIndex, splide.length );
};

export function Typed() {
  const ref = useRef<Splide>( null );   // Splide is the component class, so it works as the ref type
  const onActive = ( splide: SplideInstance, slide: Slide ) => console.log( slide.index );

  return (
    <>
      <Splide ref={ ref } options={ options } extensions={ { LogIndex } } onMoved={ onMoved } onActive={ onActive }>
        { [ 1, 2, 3, 4 ].map( n => <SplideSlide key={ n }>{ n }</SplideSlide> ) }
      </Splide>
      <button type="button" onClick={ () => ref.current?.splide?.go( '>' ) }>Next</button>
    </>
  );
}
```

- `Splide[ 'splide' ]` is the core instance type, so you don't need `@splidejs/splide` just for types.
- When you do want core types (`ComponentConstructor`, `Components`, `SlideComponent`, `PaginationData`, `EventMap`, `ResponsiveOptions` and others), use `import type` from `@splidejs/splide`. It's installed as React Splide's dependency, but under pnpm's strict layout you can only import it after adding it to your own `package.json` (pin `4.1.4`, so it dedupes with React Splide's `^4.1.3`).
- Keep core imports type-only. React Splide bundles its own copy of core, so a runtime import from `@splidejs/splide` adds a second copy to your bundle, and that copy's `Splide.defaults` doesn't reach `<Splide>`.
- `SplideEventHandlers` covers only the 28 event props. For other events, `splide.on( 'overflow', ( isOverflow: boolean ) => ... )` is typed through core's `EventMap`.
- The `<Splide>` prop type removes React's DOM event handlers, so `onKeyDown`, `onMouseEnter` and the like are type errors there. Put them on a wrapper element. `onClick` on `<Splide>` is Splide's slide `click` event, typed as `( splide, Slide, MouseEvent )`.
- A custom transition must be annotated `: TransitionComponent` (from `import type { TransitionComponent } from '@splidejs/splide'`). Returning `{ start, cancel }` without the annotation fails against the extension type (tested). See `references/custom-transitions.md`.
- The extension packages don't export their option types. Derive them from `Options`, for example `Exclude<NonNullable<Options[ 'autoScroll' ]>, boolean>`. With `skipLibCheck: false`, the Video extension's own types fail too (the `YT` namespace and `@vimeo/player` options), and on case-sensitive file systems (Linux CI) the URL Hash types resolve to `any` because of a folder-name casing mismatch. Details: `references/extensions/packages.md`.
