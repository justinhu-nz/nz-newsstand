# NZ Newsstand

`NZ Newsstand` is a small static web app that presents selected New Zealand newspaper editions as a visual "newsstand". It renders the papers onto a wooden shelf background, lets the user browse by edition date, page through individual papers, and open a fullscreen reader with cover-first spread navigation.

The project is intentionally lightweight:

- No framework
- No build step
- No backend
- One main document: `index.html`
- One image asset: `background-single.png`

## What The Site Does

The app chooses a set of newspapers for a selected New Zealand date, loads page images from a remote image endpoint, and places those papers into a shelf-style scene.

Main interactions:

- Choose a date with the date picker
- Move day-by-day with `Prev Day`, `Today`, and `Next Day`
- Use the settings icon to hide or restore front pages for the currently selected date lineup
- Page through each visible paper independently
- Use left/right keyboard arrows on the main shelf to move all visible papers together
- Click a paper cover/page to open the fullscreen reader
- Use left/right buttons or keyboard arrows in fullscreen mode

## Edition Rules

The app uses a fixed mapping between day type and the editions shown:

- Weekdays:
  - `New Zealand Herald`
  - `The Post`
  - `The Press`
  - `Otago Daily Times`
- Saturdays:
  - `Weekend Herald`
  - `The Post`
  - `The Press`
  - `Otago Daily Times`
- Sundays:
  - `Herald on Sunday`
  - `Sunday Star-Times`

These mappings are defined in `EDITION_CONFIG` inside `index.html`.

## Timezone Behavior

The app treats "today" as New Zealand time using `Pacific/Auckland`.

That matters because:

- The selected date should match the intended local edition date in New Zealand
- The `Today` button should not depend on the viewer's local timezone
- Weekday / Saturday / Sunday edition selection must follow NZ calendar boundaries

## External Data Source

Newspaper pages are not stored in this repository. The app constructs image URLs for a remote endpoint:

- Host: `https://t.prcdn.co`
- Path: `/img`
- Query parameters:
  - `file`
  - `page`
  - `scale`

The `file` parameter is built from:

- edition id
- issue date in `YYYYMMDD`
- a fixed trailing identifier segment used by the source system

Edition ids can be numeric (for example `1126`) or alphanumeric (for example `9hym` for `Otago Daily Times`).

The URL generation happens in `buildFrontPageUrl()`.

## Image Scaling Rules

The first page is treated as the cover and requests a higher scale:

- Cover page: `scale=100`
- Interior pages: `scale=70`

`Otago Daily Times` uses the same cover scale but overrides interior pages to `scale=71`.

This keeps the front page sharper while avoiding unnecessarily heavy requests for every subsequent page.

If an interior page fails at its configured scale, the app retries that page three times, reducing the scale by 10 for each attempt, before reporting it as unavailable. For example, a configured scale of `70` tries `70`, `60`, `50`, then `40`. Cover pages continue to use a single high-quality request.

## UI Structure

The page can be understood in three layers.

### 1. Page Chrome

The top control bar contains:

- previous day button
- date picker
- today button
- next day button

### 2. Shelf Scene

The main scene is a decorative background using `background-single.png` plus layered gradients and shadows. On desktop, papers are positioned into fixed "slots" on the shelf. Phones use a separate two-axis reader instead of the shelf (see `Responsive / Mobile Behavior`).

### 3. Paper Cards

Each paper card contains:

- the current loaded page image
- a fallback layer for loading/unavailable states
- a toast layer for transient status messages
- individual prev/next page controls

## Fullscreen Reader Behavior

The fullscreen reader is intentionally separate from the shelf card behavior.

On the shelf:

- each paper is displayed as a single page
- per-paper controls move one page at a time

In fullscreen:

- page 1 is shown as a standalone cover
- after that, navigation switches to facing-page spreads
- examples:
  - `1`
  - `2-3`
  - `4-5`
  - `6-7`

This allows the shelf to stay simple while the reader behaves more like an opened newspaper.

The fullscreen reader remains a desktop interaction. Phones use the dedicated single-page mobile reader described below.

## Responsive / Mobile Behavior

The layout has three general breakpoints at `1080px`, `860px`, and `520px`, plus a dedicated mobile-mode boundary at `700px`. Coarse-pointer devices also keep mobile mode in short landscape viewports. The desktop shelf and fullscreen spread behavior remain separate and unchanged.

### Viewport And Device Fit

- `viewport-fit=cover` is set so the page can extend under the notch / Dynamic Island and home indicator.
- A `theme-color` (`#3a1b0b`) and `color-scheme: dark` keep the browser chrome consistent with the dark scene.
- The app shell pads itself with `env(safe-area-inset-*)` so controls never sit under system UI.
- Mobile height is synchronized to `window.visualViewport.height` (with `100dvh` as the CSS fallback) so expanded iOS Safari controls cannot cover the bottom of the newspaper or its page controls.

### Two-Axis Mobile Reader

- The app shell becomes a `dvh`-height flex column: the control bar takes its natural height and the scene grows to fill the rest.
- The desktop shelf is hidden and replaced by a dedicated full-height mobile surface.
- A horizontal swipe moves one page backward or forward within the current newspaper.
- A vertical swipe moves between newspapers. Every newspaper occupies one mandatory vertical scroll-snap stop.
- Page position is stored independently for each newspaper, so returning vertically to a paper restores the page that was being read.
- After a page loads, the next page for that newspaper is prefetched into a mobile-only request cache so the following horizontal navigation can reuse an in-flight or completed load.
- Clearly horizontal gestures are required before paging; vertical and diagonal motion stays with the native vertical scroller, and multi-touch gestures are ignored.
- Native pinch zoom remains available. While magnified, visual-viewport height changes are ignored so Safari zooms the newspaper without reflowing the mobile shell, and any gesture that becomes multi-touch is cancelled as a page swipe.
- Each paper includes visible previous/next page buttons so gestures are not the only navigation method.
- Loading, unavailable-page, newspaper-position, and current-page states appear directly over the active paper.
- The three day-navigation buttons share a single row (`flex: 1 1 0`) so the control bar stays two rows tall (date picker + buttons).

### Touch Polish

- The blue tap-highlight is removed and `touch-action: manipulation` is set on buttons and paper images.
- `overscroll-behavior` is constrained to prevent page bounce / pull-to-refresh from interfering with shelf scrolling.

## Keyboard Shortcuts

### Main Shelf

When the fullscreen reader is closed:

- `Left Arrow`: move all visible papers back one page
- `Right Arrow`: move all visible papers forward one page

### Fullscreen Reader

When the fullscreen reader is open:

- `Left Arrow`: previous page or spread
- `Right Arrow`: next page or spread
- `Escape`: close fullscreen

The shortcuts are context-sensitive so the fullscreen reader takes priority when open.

## Error Handling

The app expects some requested pages to be unavailable and handles that gracefully.

Examples:

- If page 1 fails to load, the card shows `Issue unavailable`
- If a later page fails to load, the card keeps the last successful page and shows a transient message such as `Page X unavailable`
- An unavailable page does not block navigation: the page cursor advances past it, so the next action tries the following page instead of repeatedly retrying the missing one
- Fullscreen spread loading is tolerant of missing right-hand pages, so a spread can degrade to a single loaded left page if necessary

## Internal Architecture

Even though this is a single-file app, the JavaScript is organized into a few conceptual areas.

### Configuration

- timezone constants
- image scale constants
- slot layout definitions
- edition mappings, including optional per-edition scale overrides

### Date Utilities

Helpers convert between:

- browser `Date` objects
- NZ-local edition dates
- UTC-safe `YYYY-MM-DD` strings

### Shelf State

Each edition has a small state object that tracks:

- current page
- last successful page
- whether any page has loaded yet
- whether a request is currently in flight
- a request id for race protection
- an optional toast timer

Separately, the app stores per-edition visibility preferences in local storage so hidden front pages stay hidden until re-enabled.

### Fullscreen State

The fullscreen reader uses its own state so it can support spread navigation without changing shelf behavior.

It tracks:

- which edition is open
- which spread or cover page is active
- whether fullscreen content is currently loading
- a request id to ignore stale asynchronous responses

## Why Request IDs Exist

Image requests happen asynchronously. If the user navigates quickly, an older request could finish after a newer one. The request-id pattern prevents stale image loads from overwriting the latest intended view.

This pattern is used for:

- shelf page loading
- fullscreen spread loading
- mobile page loading

## File Overview

- `index.html`
  - all HTML markup
  - all CSS
  - all JavaScript
- `background-single.png`
  - decorative shelf background image

## Local Development

Because the project is static, local development is simple. Any basic static file server should work.

Examples:

```powershell
python -m http.server 8000
```

or

```powershell
npx serve .
```

Then open the served page in a browser.

## Maintenance Notes

If you change this app in the future, the main areas to review together are:

- `EDITION_CONFIG` if newspaper lineup changes
- `buildFrontPageUrl()` if the remote source changes naming, query rules, or edition-specific scale overrides
- shelf and fullscreen keyboard handling if new interactions are added
- fullscreen spread logic if reader behavior changes again
- the phone media query and `isMobileMode()` together — CSS and JavaScript share the mobile-mode boundary and should change in step

## Limitations

- The app depends on a third-party remote image source
- There is no persisted state or saved reading position
- There are no automated tests in this repository
- The app assumes the remote image source supports direct image requests from the browser

## Suggested Next Improvements

- Add a small caption in fullscreen showing the active page or spread
- Add a README section documenting the known edition ids and where they came from
- Add a lightweight smoke test or browser check if the project grows further
