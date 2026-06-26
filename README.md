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

## UI Structure

The page can be understood in three layers.

### 1. Page Chrome

The top control bar contains:

- previous day button
- date picker
- today button
- next day button

### 2. Shelf Scene

The main scene is a decorative background using `background-single.png` plus layered gradients and shadows. On desktop, papers are positioned into fixed "slots" on the shelf. On smaller screens, the layout switches to a horizontally scrollable, snap-aligned row and the decorative shelf is hidden (see `Responsive / Mobile Behavior`).

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

On phone-width screens the reader switches to a single-page model instead of facing-page spreads (see `Responsive / Mobile Behavior`).

## Responsive / Mobile Behavior

The layout has three breakpoints: `1080px`, `860px`, and `520px`, plus a dedicated reader breakpoint at `700px`. The phone experience (driven mainly by the `860px` and `700px` rules) differs from desktop in several deliberate ways.

### Viewport And Device Fit

- `viewport-fit=cover` is set so the page can extend under the notch / Dynamic Island and home indicator.
- A `theme-color` (`#3a1b0b`) and `color-scheme: dark` keep the browser chrome consistent with the dark scene.
- The app shell pads itself with `env(safe-area-inset-*)` so controls never sit under system UI.
- Heights use dynamic viewport units (`100dvh`, with a `100vh` fallback) so the layout is stable when the mobile URL bar collapses.

### Mobile Shelf

- The app shell becomes a `dvh`-height flex column: the control bar takes its natural height and the scene grows to fill the rest.
- The decorative shelf is removed on phones. `background-single.png`, the scene gradients (`::before` / `::after`), and the `.shelf-lip` are all hidden so papers read on the plain warm backdrop.
- Papers render in a horizontally scrollable row that uses CSS scroll-snap (`scroll-snap-type: x mandatory`, `scroll-snap-align: center`, `scroll-snap-stop: always`). Symmetric `padding-inline` lets the first and last paper center, and adjacent papers "peek" at the edges.
- The three day-navigation buttons share a single row (`flex: 1 1 0`) so the control bar stays two rows tall (date picker + buttons).

### Mobile Reader

- Below `700px` the reader shows one full-width page at a time instead of a two-page spread (two newspaper pages side-by-side are unreadable at phone widths). This is gated by `isMobileLightbox()`, which checks `matchMedia("(max-width: 700px)")`.
- `normalizeLightboxPage()`, `loadLightboxSpread()`, and `changeLightboxPage()` all branch on `isMobileLightbox()`: on phones, paging steps one page at a time; on wider screens it keeps the `1 -> 2-3 -> 4-5` spread stepping.
- A horizontal swipe pages the reader (touch handlers on the spread). Multi-touch gestures are ignored so pinch-to-zoom still works, and a swipe must be clearly horizontal to count.
- Reader controls are enlarged for touch (close `44px`, nav `52px`) and the nav buttons move to thumb-reachable bottom corners, all offset by safe-area insets.

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
- the `860px` / `700px` media queries and `isMobileLightbox()` together — the CSS breakpoint and the JS check share the `700px` value, so change them in step if the reader's mobile threshold moves

## Limitations

- The app depends on a third-party remote image source
- There is no persisted state or saved reading position
- There are no automated tests in this repository
- The app assumes the remote image source supports direct image requests from the browser

## Suggested Next Improvements

- Add a small caption in fullscreen showing the active page or spread
- Add a README section documenting the known edition ids and where they came from
- Add a lightweight smoke test or browser check if the project grows further
