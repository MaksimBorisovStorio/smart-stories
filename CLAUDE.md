# Smart Stories — Mobile Photo Album Prototype

Interactive HTML prototype of a mobile photo-album feature. Two pages: a "Smart
Stories" carousel that picks a trip, which opens a "Trip to Berlin" photo album
laid out by scene, not by date.

## Files

| Path | Purpose |
| --- | --- |
| `stories.html` | Entry page — card carousel of trips; tapping the Berlin card opens `index.html`. |
| `index.html` | Photo album for the "Trip to Berlin" story (chapter list, stacks, edit mode, etc.). |
| `Trip to Berlin/` | 96 source photos. 5 are excluded from the album (4 screenshots/tickets + 1 DB-search screenshot — see "Excluded photos" below). |
| `assets/` | Carousel card images (`<name>.png`) and their matching unstyled photos (`<name>-background.png`) plus `icon/*.svg` for the bottom tab bar. |

No build step, no dependencies. Plain HTML + CSS + vanilla JS.

## Running

```bash
cd "/Users/mborisov/Desktop/test/smart stories"
python3 -m http.server 8765
```

Then:
- Desktop: <http://localhost:8765/stories.html>
- Phone on same Wi-Fi: `http://<mac-lan-ip>:8765/stories.html`

`file://` works too but `Trip to Berlin/` contains a space, so encoding matters — image paths are URL-encoded in JS for that reason.

### Phone testing over HTTPS (for gyroscope / DeviceOrientation)

iOS Safari requires a secure context (HTTPS or `localhost`) for the
DeviceOrientation API used by the 3D card tilt. Plain HTTP from a LAN IP
won't trigger the iOS motion-permission prompt and orientation events
will not fire.

Local HTTPS setup using mkcert (no sudo needed for cert generation, so
this path works on locked-down work Macs where `mkcert -install` is
blocked):

```bash
brew install mkcert
# one-time per IP — the LAN IP may change if you switch networks
mkcert localhost 127.0.0.1 <mac-lan-ip>
python3 serve_https.py 8765
```

Open `https://<mac-lan-ip>:8765/stories.html` on the iPhone. Safari will
show a "Connection Not Private" warning because the mkcert root CA isn't
system-trusted (sudo required for that, intentionally blocked by auto-mode
and corporate policies on work Macs). Tap **Show Details → visit this
website**. The connection is still HTTPS — that's what the
DeviceOrientation API requires — so motion permission works after you
accept the warning.

Cloudflare quick tunnel (`cloudflared tunnel --url`) was the original
plan but is blocked by the corporate firewall: argotunnel.com:7844 is
unreachable on both UDP and TCP. Avoid for local testing.

## stories.html — the carousel page

- **Header** + **horizontal card carousel** + **bottom tab bar** (Home / Projects / Stories (active) / Basket / Account).
- Cards are **290 × 435**, `border-radius: 24px`, with `box-shadow: inset 0 0 0 1px rgba(255,255,255,0.2)` for the inner highlight.
- `scroll-snap-type: x mandatory; scroll-snap-stop: always` — one card centered per screen.
- A **rounded-rectangle blurred backdrop** sits behind the active card (sized slightly larger than the card, `filter: blur(51px) saturate(1.4)` so the rectangle's *own* edges feather out into a colored halo). Two layered `.card-bg` divs crossfade on card change.
- Active card is detected by finding the card whose center is closest to the carousel center; the bg crossfades on each new active.
- Tapping a **non-active** card scroll-snaps to it; tapping the **Berlin** card (which is an `<a href="index.html">`) navigates to the album.

## index.html — the album

### Data model

`chapters` array, each chapter has `tiles[]`. A tile is `{ main, stack? }`:
- `main` — file name in `Trip to Berlin/`, becomes the cover photo.
- `stack` (optional) — sibling photos that are visually identical (re-uploaded copies of the same scene with different Flickr IDs). Rendered as paper layers behind the cover with a `+N` badge.

### Layout

`gridClass(count)` picks columns: 1→1col, 2→2col, 3→3col, 4→2×2, 5→2×2 + spanning, 6→3×2, ≥9→3×3. Photos preserve original aspect ratio (`object-fit: contain`). Each frame has a deterministic random rotation `(-2.5°…+2.5°)`.

### Sticky header

Status-bar is the device's real one; the in-app nav (`< Trip to Berlin 2026 [+]`) is wrapped in `.header` with `position: sticky; top: 0`. Note: `overflow-x: clip` on `.phone` is used instead of `overflow-x: hidden` because `hidden` breaks `position: sticky` in descendants.

### Rubber-band scroll offset

Continuous, scroll-driven, **not** an IntersectionObserver entrance animation. Each `.tile` below the viewport bottom is `translateY(+36px)`, easing to `0` (with a brief negative-overshoot bounce courtesy of `easeOutBack`) as it crosses the bottom edge. Fast scrolling makes every tile bounce on its own — "sticky to rubber". Implemented with a single rAF-throttled scroll handler that also drives the subtle frame-tilt.

### Preview overlay

- Tapping a **single** tile opens a centered preview at up to 90vw / 75vh. FLIP-style animation scales from the source tile's position.
- Tapping a **stack** tile opens a horizontal swiper (`scroll-snap-type: x mandatory`) — first photo centered, swipe to step through.
- Bottom buttons: **Hide from story** (single mode only — removes the source tile) and **Close**. Esc / backdrop tap also dismisses.
- Photos that look like the cover but are duplicate Flickr IDs sit in `stack[]` and render as layered images behind the cover, not just colored rectangles.

### Edit mode

- **Long-press (~480 ms)** any tile → edit mode. Slight haptic via `navigator.vibrate(12)` where supported.
- All tiles **jiggle** (alternating CSS keyframes `tile-wiggle-a` / `tile-wiggle-b` per nth-child, with a small random negative delay per tile so the wave looks organic). 0.29s duration.
- Each tile shows a red **×** in the top-right; tap to hide that tile (fade + scale out).
- In a stack preview while edit mode is active, **each photo in the swiper has its own ×**. Tapping it removes that photo from the stack: `syncTileFromPhotos()` re-renders the source tile's cover, stack-layers, `+N` badge, and `is-stack` class in real time. If the last photo is removed, the whole tile is hidden.
- Tapping empty grid area exits edit mode; the overlay close button does *not* exit edit mode (you can keep editing across multiple tiles).

### Photo data — excluded files

These 5 files exist in `Trip to Berlin/` but are excluded from the album because they're screenshots or tickets, not real trip photos:

- `train-itinerary_55200561407_o.jpg` — DB train ticket
- `the-adventure-begins_55201314168_o.jpg` — Lufthansa strike email
- `an-extremely-full-flight_55203127296_o.jpg` — seat-map screenshot
- `a-dinner-party-in-ossling_55215701166_o.jpg` — Google Maps directions
- `berlin_55217771172_o.jpg` — DB ticket search results

### Photo data — duplicate Flickr IDs

The source folder has many photos uploaded multiple times with different Flickr IDs (the 55221xxxxx and 55222xxxxx ranges are mostly re-uploads of earlier 55217xxxxx photos). The album deduplicates these by treating them as `stack[]` items behind a single cover. If you add new photos, sort by visual scene first, then by ID.

## Style tokens (index.html)

```
--bg:       #f9f9f9       (page background)
--text:     #1a1a1a
--muted:    #8a8a8e
--accent:   #1f7a6b       (back arrow, add icon)
--radius:   10px          (photo frame corner radius)
```

## Style tokens (stories.html)

```
--bg:       #f4f4f4
--accent:   #007377       (active tab color)
--radius:   24px          (card corner radius)
```

## Coding conventions used here

- Vanilla JS only, no frameworks. Single `<script>` block at the bottom of each HTML.
- Render tiles/cards by mapping a JS array of definitions to HTML via `innerHTML`/`insertAdjacentHTML`. State lives on DOM `data-*` attributes (e.g. `data-photos="<encoded JSON array>"`).
- Animations: prefer CSS (`transition`, `@keyframes`) over JS timers; one rAF-throttled scroll handler shared across effects.
- All transforms that need to compose with other transforms are isolated to different elements (e.g., rotation on `.frame`, scroll-bounce translateY on `.tile`, wiggle replaces `.tile` transform during edit mode).
- Respect `prefers-reduced-motion` — disables wiggle and other non-essential transitions.
