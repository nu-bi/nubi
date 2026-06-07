---
name: nubi-illustrations
description: Produce and refine high-quality, on-brand SVG illustrations for the Nubi app (landing, docs, compare). Use whenever creating, fixing, or refining illustrations/graphics. Enforces a render → screenshot → critique → refine loop so illustrations are VERIFIED VISUALLY before shipping — never hand off SVG work to an agent that cannot see the result.
---

# Nubi Illustrations

Hand-authored SVG illustrations regressed repeatedly because they were written by
sub-agents that cannot SEE the rendered output. This skill fixes that: the orchestrator
(who can take screenshots) drives a tight visual loop. Sub-agents may write SVG **to spec**,
but every illustration is screenshotted, critiqued, and refined by the orchestrator until it
genuinely looks good — on BOTH light and dark surfaces.

## The loop (mandatory — do not skip)

1. **Render in isolation, large.** Use the gallery route `/dev/illustrations`
   (`src/pages/dev/IllustrationGallery.jsx`) which renders every illustration BIG, on both a
   light and a dark card, with a name label. Never judge an illustration only inside its
   cramped landing card.
2. **Screenshot** (dev server on :5173) with headless Chrome, then **crop each illustration**
   with PIL and **view it**:
   ```bash
   CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
   "$CHROME" --headless=new --disable-gpu --hide-scrollbars --window-size=1500,4000 \
     --virtual-time-budget=7000 --screenshot=/tmp/gallery.png "http://localhost:5173/dev/illustrations"
   # then crop per-tile with python3 + PIL and Read each crop
   ```
   New `import.meta.glob` files or config changes need a dev-server RESTART (kill :5173, relaunch).
3. **Critique** each against the checklist below. Be specific (e.g. "double-stroked frame",
   "points overflow the viewBox", "metaphor unclear").
4. **Refine** the offending file(s). Re-run 2–3. Loop until each passes the checklist on light
   AND dark. Only then move on.

## Brand + style spec

- **Palette** (from the logo, navy→teal): navy `#1b2363`, blue `#2456a6`, teal `#17b3a3`,
  cyan `#2dd4bf`. One linear gradient (navy→blue→teal) per file as the signature.
- **Surfaces**: illustrations render inside `bg-surface` cards — WHITE in light, dark-navy in
  dark. So: **transparent illustration background** (no own dark/black panel), and use brand
  MID-TONES + semi-transparent fills so shapes read on both. Never rely on pure-white or
  near-black fills for key shapes (they vanish in one mode).
- **Aesthetic**: think Stripe / Linear / Vercel — minimal, geometric, confident, airy. ONE
  clear metaphor per illustration. Generous negative space. Large focal element. No tiny text,
  no code snippets, no dashboards-inside-the-illustration, no "mini-screenshot" clutter.
- **Cohesion**: same stroke weight (1.5–2.5px, rounded caps), same corner radius, a recurring
  node-dot + connecting-line motif, consistent gradient direction. They are a family.

## Hard rules (these are the bugs that keep happening — enforce them)

1. **Single stroke per shape.** NEVER draw the same shape twice with two offset strokes — it
   reads as a misaligned/broken double border. One clean stroke. (Layering is allowed ONLY if
   perfectly concentric.)
2. **Stay inside the viewBox.** Every point/shape within ~6–94% of width and ~10–90% of height.
   Add a `<clipPath>` (rounded-rect inset) wrapping the content as a safety net so nothing ever
   spills out of the card. This was the WebGL overflow bug.
3. **Unique ids per file.** Prefix every gradient/clip/filter id with a per-file token
   (e.g. `kib-`, `wgl-`) — duplicate ids across files that co-render on one page corrupt fills.
4. **viewBox + responsive.** `viewBox="0 0 W H"`, `className` passthrough, `width="100%"
   height="auto"`, `preserveAspectRatio="xMidYMid meet"`.
5. **Works in light AND dark** — verify BOTH via the gallery, every time.

## Files

`src/components/illustrations/`: HeroIllustration, KernelInBrowser, WebGLPerf, EdgeCache,
EmbedAuth, LlmDashboards, ConnectorSdk. Keep export names stable (LandingPage imports them).

## Verify before done

- Gallery screenshot reviewed for ALL tiles, light + dark.
- Each passes the checklist (single stroke, contained, unique ids, clear metaphor, airy).
- `npm run build` green.
