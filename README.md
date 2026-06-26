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
- Saturdays:
  - `Weekend Herald`
  - `The Post`
  - `The Press`
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

The URL generation happens in `buildFrontPageUrl()`.

## Image Scaling Rules

The first page is treated as the cover and requests a higher scale:

- Cover page: `scale=100`
- Interior pages: `scale=70`

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

The main scene is a decorative background using `background-single.png` plus layered gradients and shadows. On desktop, papers are positioned into fixed "slots" on the shelf. On smaller screens, the layout switches to a horizontally scrollable row.

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
- edition mappings

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
- `buildFrontPageUrl()` if the remote source changes naming or query rules
- shelf and fullscreen keyboard handling if new interactions are added
- fullscreen spread logic if reader behavior changes again

## Limitations

- The app depends on a third-party remote image source
- There is no persisted state or saved reading position
- There are no automated tests in this repository
- The app assumes the remote image source supports direct image requests from the browser

## Suggested Next Improvements

- Add a small caption in fullscreen showing the active page or spread
- Add a README section documenting the known edition ids and where they came from
- Add a lightweight smoke test or browser check if the project grows further
