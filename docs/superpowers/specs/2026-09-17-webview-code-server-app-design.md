# code-server WebView Android App — Design

## Purpose

A minimal Android app that displays `http://192.168.0.2:3000` (a code-server /
VS Code Web instance on the local network) full-screen in a `WebView`. The app
is a kiosk-style display shell — no browser chrome, no navigation, no
settings.

## Constraints

- Local network only. The target host has no TLS; the app must allow
  cleartext (non-HTTPS) HTTP traffic.
- VS Code Web is a desktop-oriented single-page app, so the app should
  maximize usable screen space.

## Non-goals

- No address bar, tabs, bookmarks, or any browser-style navigation UI.
- No settings screen or configurable server URL (hardcoded for this version).
- No retry/reconnect logic for drops that happen after a successful load.
- No automated test suite — this is a UI-only shell with no business logic.

## Architecture

Single `Activity` (`MainActivity`) hosting a single `WebView` that fills the
screen. No other screens or components.

## Components

**`MainActivity`**
- Hosts the `WebView`, configures it, and loads the target URL on create.
- Locks orientation to landscape (`android:screenOrientation="landscape"` in
  the manifest).
- Enables immersive full-screen mode (hides status bar and navigation bar,
  sticky immersive) on create and on window focus regain.
- Overrides back-button handling to consume the event and do nothing.

**WebView configuration**
- JavaScript enabled, DOM storage enabled (required by VS Code Web).
- Wide viewport / overview mode tuned for a desktop-width layout rather than
  a mobile one.
- Cookies and local storage persist normally (default `WebView` behavior) so
  the code-server session survives app restarts.
- `WebViewClient` overrides `onReceivedError` / `onReceivedHttpError` for the
  main frame only.

**Manifest**
- `android:usesCleartextTraffic="true"` at the application level (simplest
  option since the app only ever talks to one local, non-HTTPS host).
- `INTERNET` permission.
- Landscape lock and fullscreen/no-title theme.

## Data flow

1. `MainActivity.onCreate` configures the `WebView` and calls
   `loadUrl("http://192.168.0.2:3000")`.
2. On success, the page renders normally; all further interaction happens
   inside the page (it's an SPA, so in-page navigation doesn't trigger
   further `WebViewClient` page-load callbacks in practice).
3. On failure of the initial load (`onReceivedError`/`onReceivedHttpError`
   for the main frame), show a short notification (Toast or minimal dialog)
   describing the failure, then call `finish()` to close the app.

## Error handling

- Scoped to the **initial load only**. Once the initial page has loaded
  successfully, no reconnect/retry logic is added — transient drops are left
  to the WebView's own default behavior (typically an in-page error until
  the user or page recovers).
- Failure path: notify (Toast/dialog) → `finish()`. No retry button, no
  auto-retry loop.

## Interaction

- Hardware back button: fully suppressed (event consumed, no navigation, no
  app exit, no default `Activity` back behavior).
- No other custom interactions.

## Testing

No automated tests (no business logic to unit test; this is a thin
configuration shell). Manual verification on a device/emulator on the same
LAN as the code-server host:

- App launches, loads the code-server UI, screen is landscape and
  full-screen (no status/nav bars).
- Turning off the code-server host (or being off the LAN) before launch
  triggers the notify-then-exit path.
- A drop *after* a successful load does not crash or exit the app.
- Hardware back button does nothing at any point.
