# Streambert Mobile Web Design

## Status

Approved direction: extend the existing React/Electron app with a browser mode for iPhone-sized screens. The first version is local-only: metadata, saved titles, history, and watch progress are stored independently in the phone browser. There is no account, backend, or cross-device synchronization.

## Goal

Allow a Streambert user to browse and watch movies, series, and anime from Safari on an iPhone with the same core flow as the laptop app: discover content, open a title, choose an episode or source, play it, and resume locally.

## Non-goals

- Cross-device account or progress synchronization.
- Replacing the Electron desktop player, download manager, Discord integration, or OS integrations.
- Adding a backend, database, authentication, or server-side proxy.
- Guaranteeing that every third-party player source permits iframe playback in every browser.
- Offline streaming or downloading on iPhone.

## Product behavior

### Browser setup

When running outside Electron, Streambert uses the existing TMDB setup flow but stores the long TMDB API Read Access Token in browser local storage. The setup screen must not call `window.electron` for external links or focus behavior. The token is validated against TMDB before the main app is shown.

The UI should make it clear that this is local to the current browser. A user entering the token on iPhone is not expected to see the laptop's token or library.

### Mobile navigation

At narrow widths, the desktop sidebar and custom titlebar are replaced with a mobile shell:

- A compact top bar with the Streambert mark, search affordance, and a simple back action when a detail page is open.
- A fixed bottom navigation bar with Home, Library, Downloads (only when supported/meaningful), and Settings. Search remains available from the top bar and can also be opened from Home.
- Safe-area padding using `env(safe-area-inset-top)` and `env(safe-area-inset-bottom)` so controls remain usable around the iPhone notch and home indicator.
- Minimum touch targets of approximately 44px and no hover-only interaction requirements.

The existing navigation state remains the source of truth. Mobile controls call the same `navigate` and search callbacks used by the desktop shell.

### Home and library

Existing home sections remain available, but mobile presentation changes to:

- Horizontal scroll rails for discovery rows to reduce vertical scanning and preserve poster size.
- A compact hero with readable title, metadata, and one primary Watch action. The overview is clamped more aggressively on narrow screens.
- Responsive poster grids for library/history and search results.
- No drag-and-drop requirement for saved items on touch devices.

The existing local storage keys and item shapes remain unchanged so the browser experience uses the same state model.

### Detail and playback

Movie and TV detail pages keep their existing metadata and controls, with the player moved near the top of the mobile detail flow. The browser player adapter renders a normal `iframe` for direct player URLs and resolved URLs. Electron continues to render the existing `<webview>` path.

The browser path must:

- Use a responsive 16:9 player container with a black background.
- Support native browser fullscreen when the embedded source allows it (`allowFullScreen` and appropriate iframe permissions).
- Show the existing loading, source selection, error, and episode selection states in touch-friendly controls.
- Avoid Electron-only progress probing. Progress is saved when the browser player exposes a same-origin video element or a supported postMessage integration; otherwise the UI records the title/episode as started and clearly avoids promising precise resume percentages.
- Hide pop-out/PiP controls that require `window.electron`.
- Preserve source switching and existing fallback/error behavior where it is browser-compatible.

Because third-party sources can reject framing or block browser playback, the player error state must offer a concise fallback message explaining that the source is unavailable in Safari and that another source can be tried.

### Desktop preservation

Electron behavior remains the default when `window.electron` exists. The custom titlebar, sidebar, downloads, native file handling, pop-out player, subtitle helpers, Discord presence, and desktop update flows remain available on desktop. Any browser-only code must be guarded by the existing `isElectron`/`window.electron` checks.

Desktop-only pages and actions should either be hidden in the mobile shell or show a short unavailable-in-browser message instead of throwing.

## Architecture

### Runtime mode

Add a small runtime-mode helper or shared constant based on `isElectron`. Use it at shell and player boundaries rather than scattering user-agent checks throughout the app.

### Shell boundary

Keep `App.jsx` responsible for page state, storage, and callbacks. Add a responsive shell layer around the existing page content that chooses `Sidebar`/`WindowTitlebar` for Electron desktop and a new mobile navigation component for narrow browser layouts. CSS media queries determine layout; runtime mode determines Electron-only behavior.

The new mobile navigation component should be presentational: it receives the current page and callbacks, and it does not own page state.

### Player boundary

Factor the repeated embedded-player element behavior in `MoviePage` and `TVPage` behind a small player-frame component or shared rendering helper. It should accept the same source URL, loading state, and visibility inputs, then render:

- `<webview>` with the current Electron attributes in Electron.
- `<iframe>` with `title`, `allow`, `allowFullScreen`, sandbox/permissions appropriate to the existing source, and the current URL in a browser.

The extraction should be narrow and avoid rewriting the surrounding movie/TV player state machines.

### Browser storage

Extend the existing storage abstraction so the browser can persist the TMDB token under the existing API-key key. Electron continues to use secure storage. This is local convenience storage, not a security boundary; the setup copy should avoid claiming that browser storage is encrypted.

## Data flow

1. App detects Electron or browser mode.
2. App loads the API token from secure storage in Electron or regular storage in a browser.
3. Setup validates the token and saves it through the storage abstraction.
4. Existing TMDB helpers fetch metadata and images.
5. Existing App callbacks update saved items, history, watched state, and progress in local storage.
6. Detail pages build the same player source URL as desktop.
7. The player boundary selects webview or iframe based on runtime mode.
8. Browser playback reports only capabilities it can observe; unsupported progress probing is treated as unavailable rather than as an error.

## Error handling

- Missing token: show the existing setup screen.
- Invalid/revoked token: use the existing auth-error flow and return the user to setup/settings.
- TMDB/network failure: preserve offline and retry states; avoid assuming Electron APIs.
- Embedded source refuses to load: show a player error with a source-switch action.
- Browser-only missing capability: hide or disable the affected action and explain it inline when needed.
- Narrow viewport or safe-area layout issues: keep content scrollable and ensure fixed navigation does not cover the last content row or player controls.

## Testing and verification

- Run the production Vite build.
- Verify Electron mode still builds with `<webview>` and desktop-only integrations guarded.
- Serve the built web assets and verify the browser setup screen does not throw when external links are clicked.
- Use a mobile viewport to verify Home, Search, Library, Movie, TV, Settings, back navigation, and safe-area spacing.
- Verify a movie and TV episode produce an iframe player in browser mode, including loading/error states and source switching.
- Verify token, saved item, history, watched, and progress persistence across a browser reload.
- Check that desktop-only downloads and pop-out actions are not exposed as broken controls in browser mode.

## Implementation shape

Likely touched areas:

- `src/App.jsx` for runtime-aware shell selection and browser token loading/saving.
- `src/components/SetupScreen.jsx` for browser-safe links/focus behavior and local-token copy.
- `src/components/Sidebar.jsx` or a new `src/components/MobileNav.jsx` for the mobile shell.
- `src/pages/MoviePage.jsx` and `src/pages/TVPage.jsx` for the shared player boundary.
- `src/utils/storage.js` for browser API-token persistence.
- `src/styles/global.css` for responsive layout, touch targets, safe-area padding, rails, and player sizing.
- Tests or lightweight verification fixtures if the repository has no existing test runner.

The implementation should favor a small number of focused changes over a second app or a broad rewrite of existing page logic.
