# Villa Serena

A scroll-driven, cinematic landing page for a luxury villa rental, built on **Wix Headless** (Astro, Wix-managed hosting). Two source videos were converted into WebP frame sequences that play back on `<canvas>` as you scroll — Apple-product-page style — with all business content (copy, amenities, stats, booking inquiries) served by Wix business services.

**Live site:** https://villa-sere-d1e1a6aa-arielh47.wix-site-host.com

Lighthouse: **96 performance · 100 accessibility · 100 best practices · 100 SEO**

## How it works

### Scroll animation

The page has two animation zones, one per source video:

- **Zone 1 — The Villa** (exterior / living spaces): hero, vision quote, living overlay
- **Zone 2 — The Kitchen**: kitchen hero, craft overlay, location finale

Each zone is a tall (`520vh`, `380vh` on mobile) section with a `position: sticky` full-viewport canvas. A `requestAnimationFrame` loop maps scroll progress to a frame index through:

1. **A dwell engine** — a Gaussian-density remap that makes the scroll "almost stop" at each content overlay and speed up between them, so text has natural reading pauses.
2. **LERP smoothing** — the drawn frame eases toward the target frame; the canvas only redraws when the rounded index changes.

Frames load progressively: every 4th frame first (behind a branded loader), the rest in batches. Zone 2 doesn't load at all until you scroll near it. Missing frames fall back to the nearest loaded neighbour, so there are never blank paints. `prefers-reduced-motion` gets a static, fully readable page.

### Frames

Each 10s / 24fps / 720p video became 120 frames × 2 sizes:

| Variant | Resolution | Quality | Payload per zone |
|---|---|---|---|
| Desktop | 1280×720 | q70 | ~4.5–5 MB |
| Mobile | 960×540 | q62 | ~2.8–3.1 MB |

Mobile additionally renders the canvas at 1× DPR, uses a snappier LERP and gentler dwell, and drops the film grain + `backdrop-filter` layers (the two most GPU-expensive effects on phones).

### Wix backend

| Concern | Service | Binding |
|---|---|---|
| Section copy (hero, quote, kitchen, location, CTA…) | Wix CMS — `villa-sections` | queried SSR via `@wix/data` |
| Amenities grid | Wix CMS — `amenities` | queried SSR |
| Hero stat bar | Wix CMS — `villa-stats` | queried SSR |
| Booking inquiries | Wix Forms | `POST /api/inquiry` → `submissions.createSubmission` |

SSR content is cached in-memory for 5 minutes ([src/lib/content.ts](src/lib/content.ts)) with hard-coded fallback copy if the CMS is unreachable. All copy is editable in the Wix dashboard (Content Manager) with no code changes — the site fetches it at request time.

Form leads land in the dashboard under **Forms & Submissions**. Note: the contact-mapped fields (name, email, phone) render in the dashboard UI; the custom fields (check-in/out, guests, message) are captured in every submission but only visible via API/export — a Wix Forms platform limitation.

## Project structure

```
src/
  pages/
    index.astro        # the entire landing page: markup, styles, animation engine
    api/inquiry.ts     # booking inquiry endpoint → Wix Forms
  lib/
    content.ts         # CMS queries + 5-min SSR cache
public/
  frames/zone1/{desktop,mobile}/   # 120 WebP frames each
  frames/zone2/{desktop,mobile}/
  gallery/                         # optimized gallery photos
astro.config.mjs       # Wix integrations (do not remove wixPages/checkOrigin)
wix.config.json        # Wix site link (siteId / appId)
```

`index.astro` is intentionally self-contained (inline styles + script): the page is a single crafted experience, not a component library.

## Development

Requires Node ≥ 20.11 and a Wix CLI login (`npx @wix/cli login`).

```bash
npm install --ignore-scripts   # --ignore-scripts skips sharp's optional native build
npm run dev                    # wix dev → http://localhost:4321
npm run build                  # wix build
npm run release                # publish to Wix hosting
```

Authentication is ambient on Wix-managed Astro — no client, no OAuth setup; `import { items } from "@wix/data"` just works, in SSR and API routes.

### Tuning the animation

All knobs live at the top of the `<script>` in `index.astro`:

| Constant | Effect |
|---|---|
| `LERP_FACTOR` | higher = frames follow scroll more tightly |
| `DWELL_PEAK` / `DWELL_WIDTH` | how strongly/widely scroll slows at content stops |
| `.anim-zone` height (CSS) | total scroll distance per zone |
| overlay `data-show` / `data-hide` | when each text overlay appears (0–1 zone progress) |

### Regenerating frames

Frames were extracted with ffmpeg + Pillow (WebP). To re-extract from new footage, produce evenly spaced frames named `frame-0001.webp … frame-0120.webp` at the two resolutions above and drop them into `public/frames/zone{1,2}/`.

## Performance notes

- Page CSS is inlined (zero render-blocking requests); Google Fonts load async with system-font fallback.
- First frame of zone 1 is preloaded per-breakpoint (`media` attribute on the preload links).
- The total frame payload (~5 MB if you scroll the whole page) is the inherent cost of the scroll-film genre — halve it by cutting to ~90 frames per zone or lowering WebP quality.
- Remaining Lighthouse headroom is Wix SSR response time (~0.9s TTFB), not client-side work (TBT 0ms, CLS 0).
