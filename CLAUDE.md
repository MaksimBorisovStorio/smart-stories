# Smart Stories — Mobile Photo Album Prototype

Interactive HTML prototype of a mobile photo-album feature. Two views — a
"Smart Stories" carousel that picks a trip, and a "Trip to Berlin" photo
album laid out by scene, not by date — both rendered from a **single
HTML file** as a JS-driven SPA.

## Files

| Path | Purpose |
| --- | --- |
| `stories.html` | The only real page. Contains **both** views (`.view-stories` and `.view-album`) and swaps between them via `history.pushState`/`popstate`. |
| `index.html` | Thin redirect to `stories.html#album` for backward-compat with any saved links. |
| `Trip to Berlin/` | 96 source photos. 5 are excluded from the album (see "Excluded photos" below). |
| `assets/` | Carousel card images (`<name>.png`) + their unstyled photos (`<name>-background.png`) + `icon/*.svg` (tab bar + eye/hide-eye). |
| `serve_https.py` | Local HTTPS server for gyro testing — see "Phone testing over HTTPS". |
| `localhost+2.pem` / `localhost+2-key.pem` | mkcert-generated cert/key. **gitignored** — never commit. |

No build step, no dependencies. Plain HTML + CSS + vanilla JS.

## Running

The prototype is deployed to **GitHub Pages**:

- **Live URL:** https://maksimborisovstorio.github.io/smart-stories/stories.html
- **Repo:** https://github.com/MaksimBorisovStorio/smart-stories (public)
- Push to `main` → Pages rebuilds in ~30s.

Pages is the easiest path for iPhone testing because it serves real HTTPS
with a trusted cert — no warnings, gyroscope works, "Add to Home Screen"
gives a full standalone PWA.

For local development:

```bash
cd "/Users/mborisov/Desktop/test/smart stories"
python3 -m http.server 8765
```

- Desktop: <http://localhost:8765/stories.html>
- Phone on same Wi-Fi: `http://<mac-lan-ip>:8765/stories.html` — gyroscope
  **won't work** here (iOS DeviceOrientation requires HTTPS).

### Phone testing over HTTPS (for gyroscope / DeviceOrientation)

iOS Safari requires a secure context for the DeviceOrientation API. Two
options that bypass cert/network restrictions on the corporate Mac:

1. **Use GitHub Pages** (preferred). Push to `main` and test against the
   live URL.
2. **Local HTTPS with mkcert** (for offline dev):

   ```bash
   brew install mkcert
   mkcert localhost 127.0.0.1 <mac-lan-ip>   # one-time per IP
   python3 serve_https.py 8765
   ```

   Safari shows "Connection Not Private" because `mkcert -install` is
   blocked on this Mac (no sudo). Tap **Show Details → visit this
   website**. The connection is still HTTPS — Safari treats it as a
   secure context after the bypass, which is all the
   DeviceOrientation API needs.

**Avoid:**
- `cloudflared tunnel --url` — the corporate firewall blocks
  `argotunnel.com:7844` on both UDP and TCP, so the tunnel never
  establishes a data plane.
- `mkcert -install` — requires sudo, denied on the work Mac.

## Architecture — single-page app

Both views live in `stories.html`:

```
<body>
  <div class="card-bg-stack">…</div>      <!-- position: fixed halo (stories view) -->
  <div class="view view-stories">…</div>  <!-- carousel + tab bar -->
  <div class="view view-album" hidden>…</div>  <!-- album page -->
</body>
```

View switching:

- `showAlbum(true)` → `history.pushState({view:"album"}, "", baseUrl + "#album")` + flip the `hidden` attribute on both view containers.
- Back button (in the album header) calls `history.back()` → `popstate` fires → views swap back.
- `location.hash === "#album"` on first load → seed history so the OS back gesture lands on stories instead of escaping the PWA.

**Why SPA, not two HTML files?** iOS standalone PWAs have a known bug where navigating to a *different* HTML file kicks the user out to Safari. Keeping everything in one file with JS-driven view switching avoids this entirely.

## stories.html — the carousel view

- **Cards:** 310 × 465, `border-radius: 24px`, inner 1 px white border at 20 % opacity. Non-active scale 0.93, active scale 1.0.
- **Scroll snap:** `scroll-snap-type: x mandatory; scroll-snap-stop: always`.
- **Halo backdrop** (`.card-bg-stack`):
  - **`position: fixed`** at the viewport top with a large `transform: translate(-50%, -180px)` — pulls the blurred top edge off-screen so the area under the status bar reads as solid halo color rather than a faded edge.
  - Two layered `.card-bg` divs crossfade on active-card change.
  - `apple-mobile-web-app-status-bar-style="black-translucent"` is required for the halo to actually paint under the status bar in standalone PWA mode. "default" reserves an opaque strip there.
- **Gyroscope tilt:** `setupGyroTilt()` IIFE at the end of the script.
  - First user click anywhere → calls `DeviceOrientationEvent.requestPermission()` if available (iOS 13+); otherwise just attaches the listener.
  - rAF loop lerps `latestG`/`latestB` toward smoothed values and writes `--ry` / `--rx` per card; the active card uses `ACTIVE_AMP 1.4`, neighbors `0.65`.
  - Each card's transform is **one chain**: `perspective(500px) rotateY(--ry) rotateX(--rx) scale(--sc)`. Separate `scale:` property + `transform:` would render isometric — `perspective()` must be inside the same transform as the rotations.
  - Active-scale transition uses **`@property --sc { syntax: "<number>" }`** so `transition: --sc 0.35s` interpolates smoothly even while rAF writes `--ry`/`--rx` each frame.
  - Each card also has a holographic-sheen `::before` with `mix-blend-mode: screen` whose `background-position` follows tilt.
  - Respects `prefers-reduced-motion: reduce` — bails out of both the listener and the rAF loop.

## Album view (Trip to Berlin)

### Data model

`chapters` array of `{ title, meta, tiles[] }`. Each tile: `{ main, stack? }`.
- `main` — file name in `Trip to Berlin/`, becomes the cover.
- `stack` (optional) — visually identical re-uploads of the same scene with different Flickr IDs. Rendered as paper layers behind the cover with a `+N` badge.

### Layout

`gridClass(count)` picks columns: 1→1col, 2→2col, 3→3col, 4→2×2, 5→2×2 + spanning, 6→3×2, ≥9→3×3. Photos preserve original aspect ratio via the per-tile `.frame` wrapper. Each frame has a deterministic random rotation `(-2.5°…+2.5°)`.

### Sticky header

The in-app nav (`< Trip to Berlin 2026 [eye]`) uses `position: sticky; top: 0`. `.phone` uses `overflow-x: clip` (not `hidden`) because `overflow: hidden` breaks `position: sticky` in descendants.

### Rubber-band scroll offset

Continuous scroll-driven offset (not an IntersectionObserver entrance). Each `.tile` below the viewport bottom is `translateY(+36 px)`, easing to `0` with a brief negative-overshoot bounce (`easeOutBack`) as it crosses the bottom edge. One rAF-throttled scroll handler powers it and the subtle frame-tilt.

### Preview overlay

- **Single tile** → centered preview at up to 90 vw / 75 vh. FLIP-style scale from the source tile's position.
- **Stack tile** → horizontal swiper with `scroll-snap-type: x mandatory`.
- Bottom action bar: edge-to-edge (`left: 14px; right: 14px`) with `flex: 1` buttons so they split evenly. Hide button has the hide-eye icon; Close is text-only.
- Esc / backdrop tap dismisses.

### Edit mode

- **Long-press (~480 ms)** any tile → edit mode. Slight haptic via `navigator.vibrate(12)`.
- Tiles jiggle (`tile-wiggle-a` / `-b` keyframes alternating per nth-child, randomized negative delay per tile).
- Each tile shows a red circular **hide-eye** button (was a cross); tap to hide.
- **Drag-arm dwell** (`DRAG_ARM_MS = 200`): once you're already in edit mode and put a finger on a tile, drag isn't activated for the first 200 ms. Quick swipes within that window are treated as scroll — fixes the scroll-vs-reorder conflict.
- Stack swiper: in edit mode, each photo gets its own hide-eye button. Removing the last photo of a stack hides the whole source tile.
- Tapping empty grid area exits edit mode; the overlay close button does *not*.

### Hide / show-hidden

- Hiding a tile adds `.tile-hidden` (was inline `display: none`). The `.removing` class plays the fade+scale animation, then we swap to `.tile-hidden`.
- The eye toggle in the album header (`#toggleHiddenBtn`, was the "add photos" slot) sets `body.show-hidden`:
  - `body.show-hidden .tile.tile-hidden { display: flex; opacity: 0.35; filter: grayscale(0.4); }` — hidden tiles re-appear semi-transparent.
  - Their hide button is suppressed in this mode (can't re-hide what's already hidden).
  - The button icon swaps between **eye** (off) and **hide-eye** (on) via CSS selectors on `aria-pressed`.
- `refreshChapterCounts()` runs after every hide. The chapter meta line becomes e.g. `12 April · 3 photos · 2 hidden` (the `· N hidden` suffix only appears when at least one tile is hidden). The date prefix is stashed in `data-date` on `.chapter-meta` so the suffix can be rewritten without losing it.

### Suppressing iOS native long-press menu

Long-pressing an `<img>` in iOS Safari opens the "Save / Share / Look Up" sheet, which conflicts with our long-press-to-edit. Suppressed two ways:

- CSS: `-webkit-touch-callout: none; -webkit-user-select: none;` on `.tile, .tile img, .stack-layer, .overlay-photo, .overlay-photo img, .card, .card img`.
- JS: `document.addEventListener("contextmenu", e => { if (e.target.closest(".tile, .overlay-photo, .card")) e.preventDefault(); })`.

### Photo data — excluded files

These 5 files exist in `Trip to Berlin/` but are excluded from the album because they're screenshots / tickets, not trip photos:

- `train-itinerary_55200561407_o.jpg` — DB train ticket
- `the-adventure-begins_55201314168_o.jpg` — Lufthansa strike email
- `an-extremely-full-flight_55203127296_o.jpg` — seat-map screenshot
- `a-dinner-party-in-ossling_55215701166_o.jpg` — Google Maps directions
- `berlin_55217771172_o.jpg` — DB ticket search results

### Photo data — duplicate Flickr IDs

The source folder has many photos uploaded multiple times with different Flickr IDs (the 55221xxxxx and 55222xxxxx ranges are mostly re-uploads of earlier 55217xxxxx photos). The album deduplicates these by treating them as `stack[]` items behind a single cover. When adding new photos, sort by visual scene first, then by ID.

## Style tokens

Tokens are **scoped per view** (under `.view-stories` and `.view-album`) rather than at `:root`, because the two views differ in palette and the merged file would otherwise have conflicting `:root` rules.

Stories view:
```
--bg:       #f4f4f4
--accent:   #007377       (active tab color)
--radius:   24px          (card corner radius)
```

Album view:
```
--bg:       #f9f9f9       (page background)
--text:     #1a1a1a
--muted:    #8a8a8e
--accent:   #1f7a6b       (back arrow, eye toggle)
--radius:   10px          (photo frame corner radius)
--photo-shadow / --stack-shadow
```

Shared `.phone` rule has `background: var(--bg)` and the view-stories override makes it `transparent` so the halo can show edge-to-edge.

## Coding conventions used here

- Vanilla JS only, no frameworks. One big `<script>` block at the bottom of `stories.html`. No IIFEs except for the gyro/debug section.
- Render tiles/cards by mapping a JS array of definitions to HTML via `innerHTML` / `insertAdjacentHTML`. State lives on DOM `data-*` attributes (e.g. `data-photos="<encoded JSON array>"`, `data-date` on `.chapter-meta`).
- Animations: prefer CSS (`transition`, `@keyframes`) over JS timers. One rAF-throttled scroll handler shared across the rubber-band + frame-tilt effects; a separate rAF loop drives the gyro lerp.
- Compose transforms by putting all related operations in **one** `transform:` chain — separate `scale:`/`rotate:`/`translate:` properties compose *outside* the transform string, which can defeat `perspective()` or fight transitions.
- Use **`@property`** to register a custom property's type when you need to transition or animate it independently of a parent transform string.
- Respect `prefers-reduced-motion: reduce` everywhere — disables tile wiggle, gyro tilt, and other non-essential transitions.

## Debug overlay

Add `?debug` (or `#debug`) to the stories URL to render a small fixed-position black panel at the bottom showing live gyro state (`isSecureContext`, `prefers-reduced-motion`, `DeviceOrientationEvent.requestPermission` type, permission state, listener attached, event count, latest gamma/beta). Helpful when the prompt or tilt isn't behaving as expected on a specific device.

The `?debug` flag is parsed once on initial load and survives the `history.replaceState` that runs at startup (which preserves `location.search` — earlier versions stripped it, killing the flag).
