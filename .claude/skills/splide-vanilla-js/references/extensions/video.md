# Video extension

Source: https://splidejs.com/extensions/video/
Package: @splidejs/splide-extension-video@0.8.0 (peer: none declared; built against @splidejs/splide ^4.0.8, its devDependency; runtime dependency `@vimeo/player` ^2.17.1, bundled into the dist file; checked with 4.1.4)

Also checked: the 0.8.0 dist (`splide-extension-video.esm.js`, `dist/types/**`, `dist/css/splide-extension-video.min.css`). A headless Chrome run with core 4.1.4 confirmed the global, the generated DOM and the method list.

The extension assigns HTML, YouTube and/or Vimeo videos to slides and controls play and pause when the carousel moves. A video can fill the whole slide or only a part of it (the slide container).

## Load (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css">

<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.min.js"></script>
```

- The CSS is required. It positions the player and draws the play button. `dist/css/splide-extension-video.min.css` is the only CSS file; there is no unminified version.
- The script sets `window.splide.Extensions.Video` and adds to the shared object, so other extensions stay.
- Files in `dist/js/`: `splide-extension-video.min.js` (classic, 33 KB), `splide-extension-video.js` (unminified classic, 106 KB, + map), `splide-extension-video.esm.js` (ES module, 94 KB), `splide-extension-video.cjs.js` (CommonJS).
- Third-party code:
  - **Vimeo**: `@vimeo/player` v2.17.1 is bundled into every build. The Vimeo iframe itself loads from `player.vimeo.com` at play time. A URL value is first resolved through `https://vimeo.com/api/oembed.json`.
  - **YouTube**: the IFrame API is **not** bundled. When a carousel with a YouTube slide mounts, the extension adds `<script src="{location.protocol}//www.youtube.com/player_api">` to `<head>`, once per page. It skips this if `window.YT.Player` already exists or a script with that `src` is present. It wraps any existing `window.onYouTubeIframeAPIReady` and chains to it.
  - **HTML video**: a native `<video>` element; nothing is fetched until it plays.

## Mount

All loaded extensions:

```js
new Splide( '.splide', { heightRatio: 0.5625 } ).mount( window.splide.Extensions );
```

Selective:

```js
new Splide( '.splide', { heightRatio: 0.5625 } ).mount( { Video: window.splide.Extensions.Video } );
```

ES module. The build has no imports, so it needs no import map, but the CSS still needs the `<link>`:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css">
<script type="module">
  import { Splide } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.esm.js';
  import { Video } from 'https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.esm.js';

  new Splide( '.splide', { heightRatio: 0.5625 } ).mount( { Video } );
</script>
```

Keep the key `Video`. The Intersection extension looks it up as `splide.Components.Video`.

## Markup

Assign a video to a slide with one of these attributes. The value can be a URL or a video ID.

| Attribute | Value |
|---|---|
| `data-splide-html-video` | Path or URL of a video file. It becomes the `src` of a `<video>`. |
| `data-splide-youtube` | A YouTube video ID (`cdz__ojQOuU`) or a URL with a `v=` query parameter (`https://www.youtube.com/watch?v=cdz__ojQOuU`). Embed URLs and short URLs are not accepted (see [Caveats](#caveats)). |
| `data-splide-vimeo` | A numeric Vimeo ID (`76979871`) or a `vimeo.com` URL (`https://vimeo.com/76979871`). |

```html
<div class="splide">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" data-splide-html-video="path or URL to the video">
        <img src="preview01.jpg">
      </li>
      <li class="splide__slide" data-splide-youtube="https://www.youtube.com/watch?v=cdz__ojQOuU">
        <img src="preview02.jpg">
      </li>
      <li class="splide__slide" data-splide-vimeo="https://vimeo.com/76979871">
        <img src="preview03.jpg">
      </li>
    </ul>
  </div>
</div>
```

The image is the preview shown under the play button until the video starts. To place the video in only part of the slide, add the [slide container](https://splidejs.com/guides/structure/#slide-container). The video then covers the container instead of the slide:

```html
<li class="splide__slide" data-splide-html-video="path or URL to the video">
  <div class="splide__slide__container">
    <img src="preview01.jpg">
  </div>
  <div>
    Other contents
  </div>
</li>
```

Generated DOM (checked in Chrome). It is appended to the slide, or to its `.splide__slide__container` if present:

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

The player (iframe or `<video>`) is created on the **first play** of that slide, not at mount. Before that, only the preview and the button exist.

## Options

Pass a `video` object in the Splide options:

```js
new Splide( '.splide', {
  video: {
    loop         : true,
    mute         : true,
    playerOptions: {
      youtube: { /* YT.PlayerVars */ },
    },
  },
} );
```

| Option | Type | Default | Description |
|---|---|---|---|
| `autoplay` | `boolean` | `false` | Enables autoplay: the video in the active slide starts on `mounted`, `moved` and `scrolled`. In some browsers autoplay doesn't work for HTML video without the `mute` option, because of the autoplay policy (https://developer.chrome.com/blog/autoplay/). Unmuted YouTube/Vimeo autoplay is usually blocked too. |
| `disableOverlayUI` | `boolean` | `false` | Disables the overlay controls such as the play button. The button is hidden, but clicking the slide still toggles play/pause. |
| `hideControls` | `boolean` | `false` | Asks the player to hide its UI (YouTube `controls: 0`, Vimeo `controls: false`, `<video>.controls = false`). The provider decides what can be hidden. Vimeo lets only PRO users hide the controls; free users can't. |
| `loop` | `boolean` | `false` | Loops the video. For YouTube it sets `loop: 1` and `playlist: <id>`. |
| `mute` | `boolean` | `false` | Mutes the video (YouTube `mute: 1`, Vimeo `muted: true`, `<video>.muted = true`). |
| `volume` | `number` | `0.2` | Initial volume from `0.0` to `1.0`, clamped. YouTube gets `volume * 100` when the player is ready, and `<video>` gets `.volume`. For Vimeo the 0.8.0 source checks `!player.getMuted()` before setting the volume, but `getMuted()` returns a Promise, so the volume is never applied to Vimeo players. |
| `host` | `string` | not set | Changes the YouTube `host` option (since 0.8.0). Other providers ignore it. For example, `'https://www.youtube-nocookie.com'` turns on Privacy Enhanced Mode. |
| `playerOptions` | `{ youtube?: YT.PlayerVars, vimeo?: VimeoOptions, htmlVideo?: HTMLVideoOptions }` | not set | Overrides the player-specific options. They are merged last, over the values the extension derives from the options above. Splide just forwards them to the player and doesn't check what each one does. |

`playerOptions` keys:

| Key | Type | Reference |
|---|---|---|
| `youtube` | `YT.PlayerVars` | https://developers.google.com/youtube/player_parameters#Parameters |
| `vimeo` | `VimeoOptions` (`@vimeo/player` options) | https://github.com/vimeo/player.js#embed-options |
| `htmlVideo` | `HTMLVideoOptions` | listed below |

```js
{
  playerOptions: {
    youtube  : {},
    vimeo    : {},
    htmlVideo: {},
  }
}
```

YouTube `playerVars` the extension sets before `playerOptions.youtube` is merged: `controls` (0 or 1 from `hideControls`), `iv_load_policy: 3`, `loop`, `playlist` (only when looping), `rel: 0`, `autoplay: 0` and `mute`.

`HTMLVideoOptions`: the properties you can pass as `htmlVideo`. They are assigned as **properties** of the `<video>` element (`Object.assign`), not as attributes, so the names are camelCase. See MDN `HTMLVideoElement` and `HTMLMediaElement`.

`autoplay`, `controls`, `crossOrigin`, `currentTime`, `disablePictureInPicture`, `disableRemotePlayback`, `height`, `loop`, `muted`, `playbackRate`, `playsInline`, `poster`, `preload`, `width`, `volume`

```js
new Splide( '.splide', {
  video: {
    host: 'https://www.youtube-nocookie.com',
  },
} );
```

i18n: the play button's `aria-label` is "Play Video". Change it with the `playVideo` key of the core `i18n` option: `i18n: { playVideo: 'Video abspielen' }`.

## Methods

Reach the component as `splide.Components.Video`:

| Method | Signature | Description |
|---|---|---|
| `play()` | `play( index?: number ): void` | Plays the video in the slide at `index`, by default `splide.index`. It is not on the docs page, but it is in the typings (`VideoComponent`) and the source. |
| `pause()` | `pause(): void` | Pauses the playing video. Internally it calls `pause()` on every player. |
| `disable()` | `disable( disabled: boolean ): void` | Disables the play button. It does not hide the button; it adds the `is-video-disabled` class to the root element. While disabled, each player ignores `play()` and `pause()` calls from the extension, so clicks, moves and `Video.play()` do nothing. This method does not disable the HTML video, YouTube or Vimeo player UI. |

```js
// Disables the button.
splide.Components.Video.disable( true );

// Enables the button.
splide.Components.Video.disable( false );
```

## Events

Each handler receives the `Player` instance of the slide. Its public methods are `play()`, `pause()`, `isPaused()`, `disable( boolean )` and `destroy()`. The slide element is on the TypeScript-private field `player.slide`, which is readable in plain JS.

| Event | Arguments | Fired |
|---|---|---|
| `video:play` | `player: Player` | When the video starts. This is not always right after the play button is clicked, because there may not be enough buffer to start the video. |
| `video:pause` | `player: Player` | When the video is paused. |
| `video:ended` | `player: Player` | When the video ends. |
| `video:error` | `player: Player` | Not on the docs page. When the player reports an error (for example an invalid ID or a failed `<video>` load). The slide also gets `is-error` and the play button shows again. |
| `video:click` | `player: Player` | Not on the docs page. When the user clicks a slide that has a video, after play/pause is toggled. All other players on the carousel pause. |

```js
// `videos` is the Splide instance (avoid a global named `splide`, see Caveats)
videos.on( 'video:play', function ( player ) {
  console.log( 'playing in', player.slide );
} );
videos.on( 'video:ended', function () {
  videos.go( '>' ); // advance after a video finishes
} );
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

## Behavior

- Clicking a slide that has a video (or its container) toggles play/pause, and the button is part of that area. Other videos on the same carousel pause.
- A playing video pauses on `move` and `scroll`, and on the first `dragging` after a `drag`. With `autoplay`, the active slide's video starts again after `moved` or `scrolled`.
- On a play request, the button hides and the player wrapper shows. On pause (including the automatic pause on move), the wrapper is hidden but not destroyed, so the preview image and button come back. The next play resumes from the same position.
- After `refresh`, for example after `splide.add()`, new slides that have video attributes get players. `Video.play( index )` (and so Intersection's `video: true`) still uses the index each slide had when its player was created (`video.autoplay` is not affected; it compares slide elements). Checked: after `add( slide, 0 )` on two video slides, `play( 1 )` played the slide now at index 2, and `play( 2 )` played nothing. Appending at the end keeps the indexes; otherwise destroy and mount again.
- With `type: 'loop'`, clones copy the data attribute and get their own players. This holds only for the clones made at mount. Clones rebuilt later (by `refresh()`, or a breakpoint that changes the clone count) copy the already-initialized slide, including `splide__slide--has-video` and its overlay, so the extension skips them: they show the play button but get no player, and clicking them does nothing (checked).
- The options are read when each player is created (mount or refresh). Changing `splide.options = { video: ... }` later doesn't affect existing players.

## Examples

### YouTube, Vimeo and HTML video (official demo settings)

Serve this page over `http://` or `https://`, not `file://` (see [Caveats](#caveats)).

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Video carousel</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/css/splide.min.css">
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/css/splide-extension-video.min.css">
</head>
<body>
  <section id="videos" class="splide" aria-label="Videos">
    <div class="splide__track">
      <ul class="splide__list">
        <li class="splide__slide" data-splide-youtube="https://www.youtube.com/watch?v=cdz__ojQOuU">
          <img src="https://img.youtube.com/vi/cdz__ojQOuU/hqdefault.jpg" alt="Sample video 1">
        </li>
        <li class="splide__slide" data-splide-youtube="3GNQL3alB-Y">
          <img src="https://img.youtube.com/vi/3GNQL3alB-Y/hqdefault.jpg" alt="Sample video 2">
        </li>
        <li class="splide__slide" data-splide-vimeo="https://vimeo.com/76979871">
          <img src="preview-vimeo.jpg" alt="Sample video 3">
        </li>
        <li class="splide__slide" data-splide-html-video="https://interactive-examples.mdn.mozilla.net/media/cc0-videos/flower.mp4">
          <img src="preview-flower.jpg" alt="Sample video 4">
        </li>
      </ul>
    </div>
  </section>

  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide@4.1.4/dist/js/splide.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@splidejs/splide-extension-video@0.8.0/dist/js/splide-extension-video.min.js"></script>
  <script>
    // Not `var splide`: a global with that name overwrites window.splide.Extensions.
    var videos = new Splide( '#videos', {
      heightRatio: 0.5625,      // 16:9; the video overlay fills the slide
      cover      : true,        // turns the preview <img> into a background
      video      : {
        loop: true,
        // host: 'https://www.youtube-nocookie.com',
      },
    } );

    videos.on( 'video:play', function ( player ) { console.log( 'play', player.slide.id ); } );
    videos.on( 'video:error', function ( player ) { console.warn( 'video error', player.slide.id ); } );

    videos.mount( { Video: window.splide.Extensions.Video } );
  </script>
</body>
</html>
```

The docs and demo use the Vimeo video `215334213`, which no longer exists: Vimeo's oEmbed API returns 404 for it, and the slide fires `video:error` (checked 2026-11-25). The examples here use `76979871` (Vimeo's own sample video), which loads. The official demo also uses `lazyLoad: 'nearby'` with `<img data-splide-lazy="https://img.youtube.com/vi/<id>/hqdefault.jpg">` previews and `classes: { slide: 'splide__slide splide__slide--image' }` (a site-specific class).

### Video in part of the slide, muted autoplay, custom HTML video options

```html
<section id="lessons" class="splide" aria-label="Lessons">
  <div class="splide__track">
    <ul class="splide__list">
      <li class="splide__slide" data-splide-html-video="videos/lesson-1.mp4">
        <div class="splide__slide__container">
          <img src="videos/lesson-1.jpg" alt="Lesson 1 preview">
        </div>
        <p>Lesson 1: setting up.</p>
      </li>
      <li class="splide__slide" data-splide-html-video="videos/lesson-2.mp4">
        <div class="splide__slide__container">
          <img src="videos/lesson-2.jpg" alt="Lesson 2 preview">
        </div>
        <p>Lesson 2: first steps.</p>
      </li>
    </ul>
  </div>
</section>

<script>
  new Splide( '#lessons', {
    heightRatio: 0.5625,      // applied to .splide__slide__container when it exists
    video      : {
      autoplay     : true,
      mute         : true,
      hideControls : false,
      playerOptions: {
        htmlVideo: { playsInline: true, preload: 'metadata', playbackRate: 1.25 },
      },
    },
  } ).mount( { Video: window.splide.Extensions.Video } );
</script>
```

### External controls

```js
var videos = new Splide( '#videos', { heightRatio: 0.5625 } )
  .mount( { Video: window.splide.Extensions.Video } );

document.querySelector( '#play' ).addEventListener( 'click', function () {
  videos.Components.Video.play();          // the active slide
} );
document.querySelector( '#pause' ).addEventListener( 'click', function () {
  videos.Components.Video.pause();         // all players
} );
document.querySelector( '#lock' ).addEventListener( 'change', function ( e ) {
  videos.Components.Video.disable( e.target.checked );
} );
```

## Caveats

- **YouTube URL format.** The YouTube video URL must include a `v=` query parameter. Embed URLs and short URLs are not accepted, and the source shows what happens:
  - `https://youtu.be/ID` and `https://www.youtube.com/shorts/ID` have no `?` or `#`, so parsing throws a `TypeError` during `mount()` and the carousel breaks.
  - `https://www.youtube.com/embed/ID?...` and share links such as `https://youtu.be/ID?si=...` parse to no ID, so that slide silently never plays.
  - Use the bare ID when in doubt.
- **`file://` breaks YouTube.** The API script URL is built as `location.protocol + '//www.youtube.com/player_api'`, which becomes `file://www.youtube.com/...`. Test on a local server (`http://localhost`).
- **Give slides a size.** `.splide__video` is absolutely positioned at 100% of the slide or container. Set `heightRatio` (for example `0.5625` for 16:9), `height` or `fixedHeight`, or size the slide or container in CSS.
- **Autoplay policies.** `video.autoplay` and Intersection's `inView: { video: true }` start playback without a user gesture. Browsers block that unless the video is muted, so set `video.mute: true`.
- **Vimeo.** Hiding the controls requires the video owner to have a PRO account. When the script loads, the bundled Vimeo library also turns any `[data-vimeo-id]` / `[data-vimeo-url]` elements elsewhere on the page into Vimeo players, and it adds a `message` listener that resizes Vimeo embeds. An unknown Vimeo ID or URL fires `video:error`, and the Vimeo library also reports an uncaught promise rejection (`"<url>" was not found.`) in the console (checked).
- **Global names from the classic script.** `splide-extension-video.min.js` declares top-level `St`, `qt` and `e` and assigns a top-level `t`, which overwrites page globals with those names (core `splide.min.js` also assigns `t`). It also sets `window.VimeoPlayerResizeEmbeds_` and `window.VimeoSeoMetadataAppended`. The unminified `splide-extension-video.js` declares `_assertThisInitialized`, `_inheritsLoose` and `_setPrototypeOf` instead. If page scripts depend on a global `e` or `t`, use the ESM build, which runs in module scope.
- `disable( true )` also stops the automatic pause on move, so a video that is already playing keeps playing while the user changes slides.
- **`destroy()` throws when an HTML video never played.** The `<video>` element is created on the first play, but the HTML player's `destroy()` reads it unconditionally, so `splide.destroy()` throws `TypeError: Cannot read properties of undefined (reading 'addEventListener')` and leaves the carousel half destroyed: later players keep their overlay, and the core components are not destroyed (checked; a never-played YouTube slide and a played HTML video destroy cleanly, and Vimeo uses the same base `destroy()` as YouTube). If you destroy carousels (page transitions, re-initialization), mount this wrapper under the key `Video` instead (checked: no error, overlays and classes removed, mounting again on the same element works):

  ```js
  function SafeVideo( Splide, Components, options ) {
    var video = window.splide.Extensions.Video( Splide, Components, options );
    var destroy = video.destroy;

    video.destroy = function () {
      video.disable( true ); // players that destroy() skips then ignore clicks
      try {
        destroy();
      } catch ( e ) {
        // Finish the cleanup that the error interrupted.
        Splide.root.querySelectorAll( '.splide__video' ).forEach( function ( el ) {
          el.remove();
        } );
        Splide.root.querySelectorAll( '.splide__slide--has-video, .splide__slide__container--has-video' ).forEach( function ( el ) {
          el.classList.remove( 'splide__slide--has-video', 'splide__slide__container--has-video' );
        } );
      }
      Splide.root.classList.remove( 'is-video-disabled', 'is-playing' );
    };

    return video;
  }

  videos.mount( { Video: SafeVideo } ); // keep the key `Video`
  ```
- With the Grid extension, the result depends on the mount order. Video reads the attributes from the slides that exist when it mounts. With `{ Grid, Video }` it sees only the pages Grid generates, so attributes on the grid cells are ignored. With `{ Video, Grid }` it creates players on the original slides before Grid moves them into cells, and clicking a cell plays its video (checked), but `Video.play( index )` no longer reaches those players, because Grid's pages take over the indexes.
- Don't write `var splide = new Splide(...)` at the top level of a classic script. That global replaces the `window.splide` namespace, so `window.splide.Extensions` becomes `undefined` (verified). Use `const`/`let`, another name, or a function scope.
- With an AMD loader on the page (RequireJS), the UMD wrapper calls `define()` instead of setting the global. See `cdn.md`.
