# Video extension (React)

HTML, YouTube and Vimeo videos in slides with `@splidejs/splide-extension-video` in React Splide: the unmount fix, markup, options, methods, events, classes and React-specific behavior.

Source: https://splidejs.com/extensions/video/
Package: @splidejs/splide-extension-video@0.8.0 (peer: none declared; built against @splidejs/splide ^4.0.8, its devDependency; runtime dependency `@vimeo/player` ^2.17.1, bundled into the dist files; tested with React Splide 0.7.12 and React 19)

Contents: [Install](#install-and-import) · [SafeVideo](#safevideo-required-for-html-video) · [Markup](#markup) · [Options](#options) · [Methods](#methods) · [Events](#events) · [Classes](#classes) · [Behavior](#behavior) · [Examples](#examples) · [React notes](#react-notes) · [Caveats](#caveats)

The extension assigns HTML, YouTube and/or Vimeo videos to slides and controls play and pause when the carousel moves. A video can fill the whole slide or only a part of it (the slide container).

## Install and import

```sh
npm install @splidejs/react-splide @splidejs/splide-extension-video
```

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import { Video } from '@splidejs/splide-extension-video';
import type { VideoComponent } from '@splidejs/splide-extension-video';
import '@splidejs/react-splide/css';
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css';
```

- The CSS is required. It positions the player and draws the play button. `dist/css/splide-extension-video.min.css` is the only CSS file; there is no unminified version. The package has no `exports` field, so this deep import resolves in every bundler.
- Keep the key `Video` in `extensions`. The Intersection extension looks it up as `splide.Components.Video`.
- Third-party code:
  - **Vimeo**: `@vimeo/player` v2.17.1 is bundled into the build. The Vimeo iframe itself loads from `player.vimeo.com` at play time. A URL value is first resolved through `https://vimeo.com/api/oembed.json`.
  - **YouTube**: the IFrame API is **not** bundled. When a carousel with a YouTube slide mounts, the extension adds `<script src="{location.protocol}//www.youtube.com/player_api">` to `<head>`, once per page. It skips this if `window.YT.Player` already exists or a script with that `src` is present. It wraps any existing `window.onYouTubeIframeAPIReady` and chains to it.
  - **HTML video**: a native `<video>` element; nothing is fetched until it plays.

## SafeVideo (required for HTML video)

In 0.8.0, `destroy()` throws when a carousel has an HTML video slide that **never played**: the `<video>` element is created on the first play, but the HTML player's `destroy()` reads it unconditionally (`TypeError: Cannot read properties of undefined (reading 'addEventListener')`). React Splide calls `splide.destroy()` on every unmount, so:

- **Development (StrictMode):** the test unmount right after the first mount throws, and React unmounts the whole tree. Nothing renders (tested).
- **Production:** hiding the carousel, a route change or a `key` change throws during unmount, and the whole app unmounts unless an error boundary catches it (tested).

YouTube and Vimeo slides, and HTML videos that already played, destroy cleanly. Use this wrapper instead of `Video` whenever a carousel may contain HTML video (all examples here use it):

```ts
// SafeVideo.ts
import type { SplideProps } from '@splidejs/react-splide';
import { Video } from '@splidejs/splide-extension-video';

type ExtensionConstructor = NonNullable<SplideProps[ 'extensions' ]>[ string ];

// Video 0.8.0 throws in destroy() when an HTML video slide never played, and React
// calls destroy() on every unmount (and on StrictMode's test unmount in development).
export const SafeVideo: ExtensionConstructor = ( splide, components ) => {
  const video = Video( splide, components );

  return {
    ...video,
    destroy() {
      video.disable( true ); // players that destroy() skips then ignore clicks
      try {
        video.destroy?.();
      } catch {
        // Finish the cleanup that the error interrupted.
        splide.root.querySelectorAll( '.splide__video' ).forEach( el => el.remove() );
        splide.root
          .querySelectorAll( '.splide__slide--has-video, .splide__slide__container--has-video' )
          .forEach( el => el.classList.remove( 'splide__slide--has-video', 'splide__slide__container--has-video' ) );
      }
      splide.root.classList.remove( 'is-video-disabled', 'is-playing' );
    },
  };
};
```

```tsx
const extensions = { Video: SafeVideo }; // keep the key `Video`

<Splide options={ options } extensions={ extensions }>...</Splide>
```

- The returned object keeps `play`, `pause` and `disable`, so `splide.Components.Video` and Intersection's `video` handler work unchanged.
- The cleanup matters for StrictMode: the second mount reuses the same `<li>` elements, and Video skips slides that still have `splide__slide--has-video`. With the cleanup, the second mount creates fresh players (tested: three overlays after remount, clicking plays).
- `disable( true )` first, because the error stops `destroy()` before it reaches the remaining players, whose click listeners on the slides would otherwise still create videos.
- Swallowing the error lets Splide's `destroy()` continue with the core components.

## Markup

Assign a video to a slide with one of these attributes on `<SplideSlide>`, which passes them to the `<li>`. The value can be a URL or a video ID.

| Attribute | Value |
|---|---|
| `data-splide-html-video` | Path or URL of a video file. It becomes the `src` of a `<video>`. |
| `data-splide-youtube` | A YouTube video ID (`cdz__ojQOuU`) or a URL with a `v=` query parameter (`https://www.youtube.com/watch?v=cdz__ojQOuU`). Embed URLs and short URLs are not accepted (see [Caveats](#caveats)). |
| `data-splide-vimeo` | A numeric Vimeo ID (`76979871`) or a `vimeo.com` URL (`https://vimeo.com/76979871`). |

```tsx
<Splide options={ { label: 'Videos', heightRatio: 0.5625 } } extensions={ { Video: SafeVideo } }>
  <SplideSlide data-splide-html-video="/videos/intro.mp4">
    <img src="/previews/intro.jpg" alt="Intro" />
  </SplideSlide>
  <SplideSlide data-splide-youtube="https://www.youtube.com/watch?v=cdz__ojQOuU">
    <img src="https://img.youtube.com/vi/cdz__ojQOuU/hqdefault.jpg" alt="Sample video" />
  </SplideSlide>
  <SplideSlide data-splide-vimeo="https://vimeo.com/76979871">
    <img src="/previews/vimeo.jpg" alt="Vimeo sample" />
  </SplideSlide>
</Splide>
```

The image is the preview shown under the play button until the video starts. A `data-*` attribute set to `undefined` is left out by React, which is handy for mixed lists (see the second example). To place the video in only part of the slide, add the slide container. The video then covers the container instead of the slide:

```tsx
<SplideSlide data-splide-html-video="/videos/lesson-1.mp4">
  <div className="splide__slide__container">
    <img src="/previews/lesson-1.jpg" alt="Lesson 1 preview" />
  </div>
  <p>Lesson 1: setting up.</p>
</SplideSlide>
```

Generated DOM (checked in Chrome). It is appended to the slide, or to its `.splide__slide__container` if present, after the children React rendered:

```html
<li class="splide__slide splide__slide--has-video" data-splide-youtube="3GNQL3alB-Y" ...>
  <img src="preview.jpg" alt="...">
  <div class="splide__video">
    <button class="splide__video__play" type="button" aria-label="Play Video"></button>
    <div class="splide__video__wrapper" style="display: none;">
      <div><!-- the YouTube iframe replaces this div; Vimeo's iframe / the <video> is placed inside it --></div>
    </div>
  </div>
</li>
```

The player (iframe or `<video>`) is created on the **first play** of that slide, not at mount. Before that, only the preview and the button exist. React doesn't know about these nodes; it leaves them alone as long as the slide's own children don't change.

## Options

Pass a `video` object in the `options` prop:

```tsx
options={ {
  video: {
    loop         : true,
    mute         : true,
    playerOptions: {
      youtube: { /* YT.PlayerVars */ },
    },
  },
} }
```

| Option | Type | Default | Description |
|---|---|---|---|
| `autoplay` | `boolean` | `false` | Enables autoplay: the video in the active slide starts on `mounted`, `moved` and `scrolled`. In some browsers autoplay doesn't work for HTML video without the `mute` option, because of the autoplay policy (https://developer.chrome.com/blog/autoplay/). Unmuted YouTube/Vimeo autoplay is usually blocked too. |
| `disableOverlayUI` | `boolean` | `false` | Disables the overlay controls such as the play button. The button is hidden, but clicking the slide still toggles play/pause. |
| `hideControls` | `boolean` | `false` | Asks the player to hide its UI (YouTube `controls: 0`, Vimeo `controls: false`, `<video>.controls = false`). The provider decides what can be hidden. Vimeo lets only PRO users hide the controls; free users can't. |
| `loop` | `boolean` | `false` | Loops the video. For YouTube it sets `loop: 1` and `playlist: <id>`. |
| `mute` | `boolean` | `false` | Mutes the video (YouTube `mute: 1`, Vimeo `muted: true`, `<video>.muted = true`). |
| `volume` | `number` | `0.2` | Initial volume from `0.0` to `1.0`, clamped. YouTube gets `volume * 100` when the player is ready, and `<video>` gets `.volume`. For Vimeo the 0.8.0 source checks `!player.getMuted()` before setting the volume, but `getMuted()` returns a Promise, so the volume is never applied to Vimeo players. |
| `host` | `string` | not set | Changes the YouTube `host` option (since 0.8.0). Other providers ignore it. For example, `'https://www.youtube-nocookie.com'` turns on Privacy Enhanced Mode (tested: the iframe loads from `youtube-nocookie.com`; Chrome logs a harmless `postMessage` origin warning). |
| `playerOptions` | `{ youtube?: YT.PlayerVars, vimeo?: VimeoOptions, htmlVideo?: HTMLVideoOptions }` | not set | Overrides the player-specific options. They are merged last, over the values the extension derives from the options above. Splide just forwards them to the player and doesn't check what each one does. |

`playerOptions` keys:

| Key | Type | Reference |
|---|---|---|
| `youtube` | `YT.PlayerVars` | https://developers.google.com/youtube/player_parameters#Parameters |
| `vimeo` | `VimeoOptions` (`@vimeo/player` options) | https://github.com/vimeo/player.js#embed-options |
| `htmlVideo` | `HTMLVideoOptions` | listed below |

YouTube `playerVars` the extension sets before `playerOptions.youtube` is merged: `controls` (0 or 1 from `hideControls`), `iv_load_policy: 3`, `loop`, `playlist` (only when looping), `rel: 0`, `autoplay: 0` and `mute`.

`HTMLVideoOptions`: the properties you can pass as `htmlVideo`. They are assigned as **properties** of the `<video>` element (`Object.assign`), not as attributes, so the names are camelCase. See MDN `HTMLVideoElement` and `HTMLMediaElement`.

`autoplay`, `controls`, `crossOrigin`, `currentTime`, `disablePictureInPicture`, `disableRemotePlayback`, `height`, `loop`, `muted`, `playbackRate`, `playsInline`, `poster`, `preload`, `width`, `volume`

i18n: the play button's `aria-label` is "Play Video". Change it with the `playVideo` key of the core `i18n` option: `i18n: { playVideo: 'Video abspielen' }`.

TypeScript: importing the package types the `video` option. `playerOptions.youtube` refers to the global `YT` namespace, which is untyped (any key accepted) unless `@types/youtube` is installed.

## Methods

Reach the component as `splide.Components.Video`, through a ref (`ref.current?.splide?.Components.Video`) or the instance an event prop receives. Its type is `VideoComponent`.

| Method | Signature | Description |
|---|---|---|
| `play()` | `play( index?: number ): void` | Plays the video in the slide at `index`, by default `splide.index`. It is not on the docs page, but it is in the typings (`VideoComponent`) and the source. |
| `pause()` | `pause(): void` | Pauses the playing video. Internally it calls `pause()` on every player. |
| `disable()` | `disable( disabled: boolean ): void` | Disables the play button. It does not hide the button; it adds the `is-video-disabled` class to the root element. While disabled, each player ignores `play()` and `pause()` calls from the extension, so clicks, moves and `Video.play()` do nothing. This method does not disable the HTML video, YouTube or Vimeo player UI. |

```tsx
const ref = useRef<Splide>( null );

ref.current?.splide?.Components.Video?.play();          // the active slide
ref.current?.splide?.Components.Video?.pause();         // all players
ref.current?.splide?.Components.Video?.disable( true ); // lock
```

## Events

There are no event props for these. Register them with `splide.on()` in `onMounted` (see [React notes](#react-notes)). Each handler receives the `Player` instance of the slide. Its public methods are `play()`, `pause()`, `isPaused()`, `disable( boolean )` and `destroy()`. The slide element is on the TypeScript-private field `player.slide`; the handler argument is typed `any`, so `player.slide` is readable without a cast.

| Event | Arguments | Fired |
|---|---|---|
| `video:play` | `player: Player` | When the video starts. This is not always right after the play button is clicked, because there may not be enough buffer to start the video. |
| `video:pause` | `player: Player` | When the video is paused. |
| `video:ended` | `player: Player` | When the video ends. |
| `video:error` | `player: Player` | Not on the docs page. When the player reports an error (for example an invalid ID or a failed `<video>` load). The slide also gets `is-error` and the play button shows again. |
| `video:click` | `player: Player` | Not on the docs page. When the user clicks a slide that has a video, after play/pause is toggled. All other players on the carousel pause. |

```tsx
<Splide
  extensions={ { Video: SafeVideo } }
  onMounted={ splide => {
    splide.on( 'video:play', player => setStatus( `Playing ${ player.slide.id }` ) );
    splide.on( 'video:ended', () => splide.go( '>' ) ); // advance after a video finishes
  } }
>
```

## Classes

The docs list these classes:

| Class | Element | Description |
|---|---|---|
| `.is-playing` | root | Present while a video is playing. |
| `.splide--has-video` | root | Listed on the docs page, but 0.8.0 never adds it. Don't use it in selectors; use `.splide__slide--has-video` instead. |
| `.splide__slide--has-video` | slide | Added to a slide that has a video assigned. |
| `.splide__slide__container--has-video` | container | Added to the container that has the video. |

These are also in the source:

| Class | Element | Description |
|---|---|---|
| `.is-video-disabled` | root | After `Video.disable( true )`. |
| `.is-error` | slide | After a player error. |
| `.splide__video` | generated | The overlay: `position: absolute; top: 0; left: 0; width: 100%; height: 100%`. |
| `.splide__video__play` | generated `<button>` | 40×40 round grey button (`#ccc`) with a CSS triangle. Opacity `.7`, `1` on hover. |
| `.splide__video__wrapper` | generated | Black background. The player inside is 100% × 100%. Hidden until playback starts. |

Splide adds these classes to the DOM directly. Don't try to drive them through `className` on `<Splide>` or `<SplideSlide>`.

## Behavior

- Clicking a slide that has a video (or its container) toggles play/pause, and the button is part of that area. Other videos on the same carousel pause.
- A playing video pauses on `move` and `scroll`, and on the first `dragging` after a `drag`. With `autoplay`, the active slide's video starts again after `moved` or `scrolled`.
- On a play request, the button hides and the player wrapper shows. On pause (including the automatic pause on move), the wrapper is hidden but not destroyed, so the preview image and button come back. The next play resumes from the same position.
- After `refresh`, new slides that have video attributes get players. React Splide calls `refresh()` whenever the list of slide elements changes, so slides added through state get players (tested). `Video.play( index )` (and so Intersection's `video: true`) still uses the index each slide had when its player was created (`video.autoplay` is not affected; it compares slide elements). Checked (with `splide.add( slide, 0 )`, which takes the same refresh path): after inserting a slide at index 0 of two video slides, `play( 1 )` played the slide now at index 2, and `play( 2 )` played nothing. Appending keeps the indexes; for other changes, remount with a `key`.
- With `type: 'loop'`, clones copy the data attribute and get their own players. This holds only for the clones made at mount. Clones rebuilt later (by `refresh()`, or a breakpoint that changes the clone count) copy the already-initialized slide, including `splide__slide--has-video` and its overlay, so the extension skips them: they show the play button but get no player, and clicking them does nothing. In React this happens on every slide-list change of a loop carousel (tested). Remount with a `key` instead (third example).
- The options and the data attributes are read when each player is created (mount or refresh). Changing the `video` options or a slide's `data-splide-*` value later doesn't affect existing players. Remount with a `key`.

## Examples

### HTML video lessons with status and external controls

```tsx
import { useRef, useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css';
import { SafeVideo } from './SafeVideo';

const lessons = [
  { id: 'lesson-1', title: 'Lesson 1: setting up', src: '/01.mp4' },
  { id: 'lesson-2', title: 'Lesson 2: first steps', src: '/01.mp4' },
  { id: 'lesson-3', title: 'Lesson 3: going further', src: '/01.mp4' },
];

const extensions = { Video: SafeVideo }; // keep the key `Video`

export default function Lessons() {
  const ref = useRef<Splide>( null );
  const [ status, setStatus ] = useState( 'Nothing playing' );
  const [ locked, setLocked ] = useState( false );
  const [ shown, setShown ] = useState( true );

  return (
    <>
      <p>
        <button id="play" type="button" onClick={ () => ref.current?.splide?.Components.Video?.play() }>Play active</button>
        <button id="pause" type="button" onClick={ () => ref.current?.splide?.Components.Video?.pause() }>Pause</button>
        <label>
          <input
            id="lock"
            type="checkbox"
            checked={ locked }
            onChange={ e => {
              setLocked( e.target.checked );
              ref.current?.splide?.Components.Video?.disable( e.target.checked );
            } }
          />
          Lock videos
        </label>
        <button id="shown" type="button" onClick={ () => setShown( value => ! value ) }>
          { shown ? 'Hide' : 'Show' } carousel
        </button>
      </p>

      { shown && (
        <Splide
          ref={ ref }
          id="lessons"
          options={ {
            label      : 'Lessons',
            heightRatio: 0.5625,
            video      : {
              mute         : true,
              playerOptions: {
                htmlVideo: { playsInline: true, preload: 'metadata' },
              },
            },
          } }
          extensions={ extensions }
          onMounted={ splide => {
            // Video events have no props. The setters are stable, so these closures never go stale.
            splide.on( 'video:play', player => setStatus( `Playing ${ player.slide.dataset.lesson }` ) );
            splide.on( 'video:pause', player => setStatus( `Paused ${ player.slide.dataset.lesson }` ) );
            splide.on( 'video:ended', () => splide.go( '>' ) );
            splide.on( 'video:error', player => setStatus( `Could not load ${ player.slide.dataset.lesson }` ) );
          } }
        >
          { lessons.map( lesson => (
            <SplideSlide key={ lesson.id } data-splide-html-video={ lesson.src } data-lesson={ lesson.id }>
              <p>{ lesson.title }</p>
            </SplideSlide>
          ) ) }
        </Splide>
      ) }

      <p id="status" aria-live="polite">{ status }</p>
    </>
  );
}
```

Tested in production and development: "Play active" plays and the status follows `video:play`; the next arrow pauses it (`video:pause`); clicking another slide plays that one; with "Lock videos" checked the root has `is-video-disabled` and play requests do nothing; hiding and showing the carousel (lesson 3 never played) causes no error, and the remounted carousel plays. The same component with plain `Video` crashes on unmount.

### YouTube and Vimeo

```tsx
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css';
import { SafeVideo } from './SafeVideo';

const videos = [
  { provider: 'youtube', id: 'cdz__ojQOuU', title: 'Sample video 1' },
  { provider: 'youtube', id: 'https://www.youtube.com/watch?v=3GNQL3alB-Y', title: 'Sample video 2' },
  { provider: 'vimeo', id: 'https://vimeo.com/76979871', title: 'Sample video 3' },
] as const;

export default function VideoCarousel() {
  return (
    <Splide
      id="videos"
      options={ {
        label      : 'Videos',
        heightRatio: 0.5625, // 16:9; the video overlay fills the slide
        video      : {
          loop: true,
          mute: true,
          host: 'https://www.youtube-nocookie.com',
        },
      } }
      extensions={ { Video: SafeVideo } }
    >
      { videos.map( video => (
        <SplideSlide
          key={ video.id }
          data-splide-youtube={ video.provider === 'youtube' ? video.id : undefined }
          data-splide-vimeo={ video.provider === 'vimeo' ? video.id : undefined }
        >
          <p>{ video.title }</p>
        </SplideSlide>
      ) ) }
    </Splide>
  );
}
```

Tested with network access, in production and development: the YouTube API script was added once, clicking the first slide created a `youtube-nocookie.com/embed/cdz__ojQOuU?...loop=1&playlist=cdz__ojQOuU...&mute=1` iframe, and the Vimeo slide created a `player.vimeo.com/video/76979871?muted=1&loop=1` iframe. The docs and demo use the Vimeo video `215334213`, which no longer exists (oEmbed returns 404 and the slide fires `video:error`, checked 2026-11-25); `76979871` is Vimeo's own sample. Add preview images as children, for example `https://img.youtube.com/vi/<id>/hqdefault.jpg` for YouTube.

### Loop carousel whose slides change

```tsx
import { useState } from 'react';
import { Splide, SplideSlide } from '@splidejs/react-splide';
import '@splidejs/react-splide/css';
import '@splidejs/splide-extension-video/dist/css/splide-extension-video.min.css';
import { SafeVideo } from './SafeVideo';

const extensions = { Video: SafeVideo };

export default function Playlist() {
  const [ clips, setClips ] = useState( [ 'clip-1', 'clip-2', 'clip-3' ] );

  return (
    <>
      <button id="add" type="button" onClick={ () => setClips( list => [ ...list, `clip-${ list.length + 1 }` ] ) }>
        Add clip
      </button>
      <Splide
        // A new key remounts the carousel, so loop clones and players are rebuilt from scratch.
        key={ clips.join( '|' ) }
        id="playlist"
        options={ { label: 'Playlist', type: 'loop', heightRatio: 0.5625, video: { mute: true } } }
        extensions={ extensions }
      >
        { clips.map( clip => (
          <SplideSlide key={ clip } data-splide-html-video="/01.mp4">
            <p>{ clip }</p>
          </SplideSlide>
        ) ) }
      </Splide>
    </>
  );
}
```

Tested: after "Add clip", the new slide and the clones all play. Without the `key`, React Splide refreshes the same instance, and the rebuilt clones show a play button that does nothing. The remount resets the carousel to the first slide; pass `options.start` from state if you need to keep the position.

## React notes

- **Use SafeVideo** whenever a carousel can contain an HTML video (see above). Without it, development mode renders nothing and production crashes on unmount.
- **Events through `onMounted`.** Video events fire only after playback starts, so `onMounted` (inside `mount()`) and a ref-based `useEffect` both work; `onMounted` also re-registers on the new instance after StrictMode's remount. Handlers are captured once: use state setters, or a ref for other changing values (`references/reactivity.md`).
- **SSR.** The package imports safely on the server. Nothing runs until mount, so the server HTML is your slides with their `data-splide-*` attributes, and the overlay and players appear after hydration.
- **Script side effects.** The YouTube API script stays in `<head>` after the carousel unmounts (it is added once per page). When the module loads in the browser, the bundled Vimeo library turns any `[data-vimeo-id]` / `[data-vimeo-url]` elements elsewhere on the page into Vimeo players, and it adds a `message` listener that resizes Vimeo embeds.
- **Changing videos.** A new `data-splide-*` value on an existing slide is ignored by its player. Give the slide (or the carousel) a new `key`.
- **Loop and dynamic lists.** Remount with a `key` (third example). For `type: 'slide'` carousels a plain refresh works when you only append slides.

## Caveats

- **YouTube URL format.** The YouTube video URL must include a `v=` query parameter. Embed URLs and short URLs are not accepted, and the source shows what happens:
  - `https://youtu.be/ID` and `https://www.youtube.com/shorts/ID` have no `?` or `#`, so parsing throws a `TypeError` during mount, which in React unmounts the tree.
  - `https://www.youtube.com/embed/ID?...` and share links such as `https://youtu.be/ID?si=...` parse to no ID, so that slide silently never plays.
  - Use the bare ID when in doubt.
- **`file://` breaks YouTube.** The API script URL is built as `location.protocol + '//www.youtube.com/player_api'`. Dev servers serve `http://localhost`, which is fine.
- **Give slides a size.** `.splide__video` is absolutely positioned at 100% of the slide or container. Set `heightRatio` (for example `0.5625` for 16:9), `height` or `fixedHeight`, or size the slide or container in CSS. A `height` in `<SplideSlide style>` is overwritten by Splide's layout.
- **Autoplay policies.** `video.autoplay` and Intersection's `inView: { video: true }` start playback without a user gesture. Browsers block that unless the video is muted, so set `video.mute: true`.
- **Vimeo.** Hiding the controls requires the video owner to have a PRO account. An unknown Vimeo ID or URL fires `video:error`, and the Vimeo library also reports an uncaught promise rejection (`"<url>" was not found.`) in the console (checked).
- `disable( true )` also stops the automatic pause on move, so a video that is already playing keeps playing while the user changes slides.
- With the Grid extension, the result depends on the key order in `extensions`. Video reads the attributes from the slides that exist when it mounts. With `{ Grid, Video }` it sees only the pages Grid generates, so attributes on the grid cells are ignored. With `{ Video, Grid }` it creates players on the original slides before Grid moves them into cells, and clicking a cell plays its video (checked), but `Video.play( index )` no longer reaches those players, because Grid's pages take over the indexes.
