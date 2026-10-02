# BetterYoutubeMusic — Firefox Extension Spec

## 1. Overview

**BetterYoutubeMusic** is a Firefox extension that replaces the solid dark background of
YouTube Music (`https://music.youtube.com/*`) with a mystical night-sky
wallpaper. The app's background surfaces are made transparent so the wallpaper
shows through behind the existing UI.

The extension is **purely cosmetic**. It changes how the background looks and
nothing else.

## 2. Goals

- Show a fixed, full-viewport night-sky image behind all of YouTube Music.
- Make YouTube Music's opaque background layers transparent (or translucent) so
  the image is visible.
- Keep all text, icons, album art, and controls readable.
- Work across every YouTube Music view: Home, Explore, Library, search results,
  artist/album/playlist pages, and the Now Playing (player) page.

## 3. Non-Goals

The extension must **not**:

- Change playback, audio, queue, ads, recommendations, or any other behavior.
- Read, store, or send any user data, listening history, or account info.
- Make network requests (the image ships inside the extension).
- Change layout, sizing, spacing, fonts, or the colors of foreground elements
  (text, buttons, icons, thumbnails).
- Affect any site other than `music.youtube.com`.
- Provide a settings UI, popup, or options page (v1).

## 4. Background Image

- The wallpaper is `images/background.png` (a crescent moon over a night-sky
  landscape), supplied by the project owner. The 740×493 original was
  upscaled (Lanczos + light sharpening) and center-cropped to 1920×1080.
- The image is **bundled locally** in the extension. It is never fetched from
  a remote URL at runtime.
- A true high-resolution source can be dropped in at the same path with no
  code changes.

## 5. Technical Design

### 5.1 File Structure

```
betteryoutubemusic/
├── manifest.json
├── content.js
├── betteryoutubemusic.css
├── images/
│   └── background.png
├── icons/
│   ├── icon-48.png
│   └── icon-96.png
└── spec.md            (not packaged)
```

### 5.2 Manifest (Manifest V3)

```json
{
  "manifest_version": 3,
  "name": "BetterYoutubeMusic",
  "version": "1.0.0",
  "description": "Replaces the YouTube Music background with a night-sky wallpaper.",
  "icons": {
    "48": "icons/icon-48.png",
    "96": "icons/icon-96.png"
  },
  "content_scripts": [
    {
      "matches": ["https://music.youtube.com/*"],
      "css": ["betteryoutubemusic.css"],
      "js": ["content.js"],
      "run_at": "document_start"
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["images/background.png"],
      "matches": ["https://music.youtube.com/*"]
    }
  ],
  "browser_specific_settings": {
    "gecko": {
      "id": "betteryoutubemusic@local",
      "strict_min_version": "142.0",
      "data_collection_permissions": {
        "required": ["none"]
      }
    }
  }
}
```

- **No `permissions`** and **no `host_permissions`** are needed beyond the
  content script match pattern.
- No background script, popup, or options page.

### 5.3 content.js

Its only job is to give the stylesheet the extension-internal URL of the image,
because the `moz-extension://<uuid>/` origin differs per install.

```js
document.documentElement.style.setProperty(
  "--bym-bg-image",
  `url("${browser.runtime.getURL("images/background.png")}")`
);
```

It must not touch the DOM in any other way, listen to events, or talk to the
page's scripts.

### 5.4 betteryoutubemusic.css

#### 5.4.1 Wallpaper layer

Put the image on the root so it sits behind everything:

```css
html,
body {
  background: #000 var(--bym-bg-image) center / cover no-repeat fixed !important;
}
```

- `background-attachment: fixed` keeps the wallpaper still while content
  scrolls.
- `#000` is the fallback color if the image fails to load.

#### 5.4.2 Transparent app surfaces

Override YouTube Music's background theme variables and the main container
elements. Selectors to target (these must be checked against the live DOM
during implementation, because YouTube Music changes its markup over time):

| Area                    | Selector(s)                                                   |
| ----------------------- | ------------------------------------------------------------- |
| App root                | `ytmusic-app`, `ytmusic-app-layout`, `#layout`                 |
| Top nav bar             | `ytmusic-nav-bar`, `#nav-bar-background`                       |
| Side guide / drawer     | `#guide-wrapper`, `#guide-content`, `ytmusic-guide-renderer`, `#mini-guide-background` |
| Browse pages            | `ytmusic-browse-response`, `#contents`, `ytmusic-section-list-renderer` |
| Search results          | `ytmusic-search-page`, `ytmusic-tabbed-search-results-renderer` |
| Artist/album headers    | `ytmusic-immersive-header-renderer`, `ytmusic-detail-header-renderer`, `ytmusic-responsive-header-renderer` |
| Now Playing page        | `ytmusic-player-page`, `#player-page`, `#side-panel`, `ytmusic-tab-renderer` |
| Bottom player bar       | `ytmusic-player-bar`, plus the `#player-bar-background` layer behind it (see note below) |

Theme variables to override on `ytmusic-app` / `:root`:

```css
:root,
ytmusic-app {
  --ytmusic-background: transparent !important;
  --ytmusic-general-background-a: transparent !important;
  --ytmusic-general-background-b: transparent !important;
  --ytmusic-general-background-c: transparent !important;
}
```

Do **not** override `--ytmusic-brand-background-solid` / `-translucent`:
menus, the playlist dialog, toasts and upload panels use it for their
background. `--ytmusic-background` is also used as a *foreground* color by the
sidebar play button icon and as the search box background, so those two
consumers get the original color restored explicitly
(`--yt-sys-color-baseline--base-background`).

Element overrides use `background: transparent !important` (or
`background-color`).

**Shadow DOM note:** `ytmusic-app-layout` renders `#nav-bar-background`,
`#mini-guide-background` and `#player-bar-background` inside its shadow root,
so CSS selectors from the extension cannot reach them. Style them through the
custom properties they read, set on `ytmusic-app-layout` (custom properties
inherit into shadow roots):

- `--ytmusic-nav-bar`: top bar and collapsed side guide (shown on scroll).
- `--ytmusic-player-bar-background`: the solid layer behind the bottom
  player bar. Set it to `transparent` on `ytmusic-app-layout`, then back to
  the tint on `ytmusic-player-bar` itself.

#### 5.4.3 Readability layers (translucent, not fully transparent)

Some surfaces sit over busy parts of the image and need a dark, semi-opaque
tint so text stays readable:

| Surface                           | Treatment                                              |
| --------------------------------- | ------------------------------------------------------ |
| Top nav bar (when scrolled)        | `rgba(0, 0, 0, 0.45)` + `backdrop-filter: blur(12px)`   |
| Bottom player bar                  | `rgba(0, 0, 0, 0.55)` + `backdrop-filter: blur(12px)`   |
| Side guide (expanded)              | `rgba(0, 0, 0, 0.35)` + `backdrop-filter: blur(8px)`    |
| Menus, dropdowns, dialogs, popups  | **Leave unchanged** (keep their native opaque styling)  |

Optional global dim: a fixed `html::before` overlay of `rgba(0, 0, 0, 0.25)`
behind content, if the chosen image is too bright. Values are starting points
and should be tuned against the final image.

#### 5.4.4 Things that must stay untouched

- Popup menus (`tp-yt-paper-listbox`, `ytmusic-menu-popup-renderer`,
  `tp-yt-iron-dropdown`), dialogs, and toasts: keep opaque so they stay usable.
- `ytmusic-browse-response .background-gradient`: this element **wraps all
  page content** (header, tabs, shelves). Only strip its background; never
  hide it (`display: none` blanks the Home page).
- Thumbnails, album art, video player (`#song-video`, `video`): never made
  transparent.
- Foreground colors (text, icons, progress bar, accents).

## 6. Behavior Requirements

| ID   | Requirement                                                                 |
| ---- | --------------------------------------------------------------------------- |
| R1   | The wallpaper is visible on first load of any `music.youtube.com` page.      |
| R2   | The wallpaper stays applied after in-app (SPA) navigation without reload.    |
| R3   | The wallpaper is fixed; it does not scroll with content.                    |
| R4   | The wallpaper covers the full viewport at any window size, without stretching (`cover`). |
| R5   | There is no flash of the original dark background on load (CSS injected at `document_start`). |
| R6   | Playback, controls, search, and navigation work exactly as without the extension. |
| R7   | No network requests are made by the extension.                               |
| R8   | Disabling/removing the extension fully restores the original look (no leftover state). |
| R9   | Text contrast stays readable (target WCAG AA, 4.5:1, for main body text over the tinted surfaces). |

Because the stylesheet is declarative CSS injected by the content script,
R2 is satisfied automatically. No `MutationObserver` is needed.

## 7. Testing

Manual test pass in Firefox (load via `about:debugging` → *This Firefox* →
*Load Temporary Add-on* → select `manifest.json`):

1. Open `music.youtube.com`. The night-sky image is visible behind the UI.
2. Navigate to Home, Explore, Library, a search, an artist page, an album
   page, and a playlist. The background stays consistent everywhere.
3. Play a song, then open the Now Playing page (both Song and Video modes).
   Album art/video are intact; the panel background shows the wallpaper.
4. Open the three-dot menu on a track, the account menu, and the "Save to
   playlist" dialog. They are opaque and readable.
5. Collapse/expand the side guide. Resize the window down to a narrow width.
   No gaps or repeated tiles in the background.
6. Scroll a long page. The wallpaper stays fixed.
7. Check the Network tab in DevTools. There are no requests from the extension
   origin other than the local image.
8. Disable the extension and reload. The original YouTube Music look comes back.

Optional: run `npx web-ext lint` for manifest/AMO validation and
`npx web-ext run` for quick iteration.

## 8. Packaging & Distribution

- Build with `npx web-ext build --ignore-files spec.md` → `web-ext-artifacts/betteryoutubemusic-1.0.0.zip`.
- For permanent install outside AMO, the add-on must be signed (via AMO
  unlisted submission) or used in Firefox Developer Edition/Nightly with
  `xpinstall.signatures.required = false`.

## 9. Future Ideas (out of scope for v1)

- Options page to choose a custom image or upload your own.
- Slider for dim/blur strength.
- Toolbar toggle to turn the wallpaper on/off without disabling the add-on.
- Use the current album art (blurred) as a dynamic background.
