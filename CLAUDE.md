# Lottie recreation workflow & preferences

This repo holds Lottie animations pulled from LottieFiles preview pages that gate the actual download behind a paid plan. The point of this repo: get the same animation, either by finding the real source file directly or by rebuilding it from a screen recording, and never pay for what's already shown for free.

Each animation lives in `animations/<name>/` — the final `.json`, a `preview.gif`, and a short `NOTES.md`. This file is the shared context: read it before starting a new one.

## Step 0: check for a direct, unauthenticated asset URL before recreating anything

Always try this first. Before doing any measurement/rebuild work, check whether the preview page is already loading the real Lottie/dotLottie file from a public CDN URL with no auth — if so, that file *is* the deliverable, and no recreation is needed at all. This is not a paywall bypass: it's the same public URL the page's own preview player loads for any visitor. Treat it as in-bounds when (a) the URL requires no auth/cookies/tokens — it's a plain public asset URL — and (b) the page itself states the animation is free to use (e.g. "Free to use under the Lottie Simple License"). If either condition is unclear or the URL looks access-controlled, fall back to the recreation pipeline below instead.

How to find the URL:
1. Open the preview page with real browser automation (Claude in Chrome), not a raw `curl` — sites like lottiefiles.com bot-block direct `curl`, and working around that block is out of bounds. Switching to legitimate browser automation is the correct response, not header-spoofing.
2. Try the browser tool's network-request reader first, filtered for `lottie`/`.json`/`dotlottie`/`assets`. This sometimes misses the request due to a timing/attachment gap.
3. If that comes up empty, run `performance.getEntriesByType('resource')` via the page's own JS console/eval tool. This lists every resource the page has loaded since navigation start regardless of when you query it, and reliably surfaces the real asset URL (e.g. `https://assets-v2.lottiefiles.com/a/.../xxxxx.lottie`) even when the network-request reader missed it.
4. Download the file. A `.lottie` extension means dotLottie: it's a zip archive containing `manifest.json` (metadata) plus `animations/<id>.json` (the actual raw Lottie JSON) — unzip it to get the real file. A bare `.json` URL is already the raw Lottie file.
5. Sanity-check it's really the free asset: compare file size against what the page's own "download" UI shows (e.g. a "1.8 KB dotLottie" option with no Pro/crown badge), and confirm the page's stated license.
6. Clean up only what's needed for the intended use — e.g. strip a baked-in opaque background solid layer (`ty:1`) if the animation is meant to overlay transparently — and hand the file over as-is otherwise. Don't run it through the measure-and-rebuild pipeline; that's for when no such direct file exists.
7. Verify the clean-up: render it (lottie-web + Playwright, see below) over a checkerboard page background and confirm the checkerboard shows through everywhere except the actual artwork.

Only fall back to the screen-recording pipeline below when there's no accessible direct asset (e.g. the animation is genuinely gated, or the source is only a video with no source page).

## Why the recreation pipeline works (when Step 0 doesn't apply)
A screen recording is raster pixels with no vector/keyframe data. There's no automatic video→Lottie converter. This is a measure-then-rebuild process: extract the animation's real geometry and timing from the video, then hand-construct the Bodymovin/Lottie JSON to match, then apply the standing taste defaults (below) on top of the measured numbers.

## Pipeline
1. `ffprobe` for fps/duration/resolution.
2. Coarse pass: extract frames at ~20fps, tile into a labeled contact sheet (PIL), find the loop period and rough phase structure.
3. Fine pass: re-extract at 30fps starting at a clean phase boundary (e.g. the blank frame) so one capture window holds a full loop.
4. Quantitative timeline: mask pixels by color, count per frame; for radial effects, bucket by distance-from-center and run `scipy.ndimage.label` to count/locate blobs. This gives exact phase-transition frames — don't eyeball it.
5. Geometry: bounding box + scanline thickness for stroke width/radius; distance-thresholding to separate compound-icon parts (e.g. checkmark = pixels inside the ring's inner radius), extreme points as path vertices; cluster centroids for burst-dot angle/distance/size.
6. Build the Lottie JSON programmatically using a `build_keys()`-style helper — see the two keyframe pitfalls below, both need to be handled correctly.
7. Render with the real engine: `lottie-web` (npm) + headless Chromium (Playwright), inline the JSON into the HTML (`fetch()` of a local file over `file://` can hang).
8. Verify motion is gradual, not just present: sample every frame across each animated segment and re-run the step-4 pixel/blob measurement on the *rendered* frames. A real draw-on/fly-out shows the number climbing frame over frame; a snap shows flat-then-jump.
9. Verify nothing is silently blanked: render at least one frame from inside every eased segment AND a frame from the final hold afterward, in the same page session, and confirm the hold frame still shows content (see pitfall #2 — this class of bug makes everything downstream of the poisoned frame render blank, with no console error).
10. Crop-and-resize side-by-side against equivalent original-video frames.
11. Deliver the `.json` and a preview GIF into `animations/<name>/` in this repo, plus a `NOTES.md` for that animation.

## Two lottie-web/Bodymovin keyframe pitfalls (both cost a full round of feedback to find on the first animation — check for these every time)

**Pitfall 1 — `"h":1` holds to the *next* keyframe, not from the previous one.** A hold keyframe's value stays flat all the way to the *next* keyframe's time, then jumps instantly — it does not mean "flat before this point, animated after." A common "flat at v0, then animate to v1" pattern needs `[hold(t0,v0), ease(t1,v0), ease(t2,v1)]` — marking the *middle* keyframe `h:1` instead holds flat all the way to t2 and snaps at the end.

**Pitfall 2 — a keyframe with only ONE of `i`/`o` silently corrupts ALL later rendering.** Giving a keyframe only "o" (because its incoming segment was a hold, so it seemed to need no "i") makes lottie-web's SVG renderer blank every shape on every frame evaluated *after* that one, for the rest of the session — no console error, no exception, just empty paths from then on. Fix: every non-hold keyframe always gets BOTH `i` and `o` together (even the last keyframe in an array, where it's unused but harmless) — never emit one alone.

Both pitfalls are covered by verification step 9 above (render a late hold frame after visiting eased frames, in-session, and confirm it isn't blank) — that check catches pitfall 2 immediately instead of costing a review round.

## Jeff's standing taste defaults (apply these from the FIRST draft, don't wait to be told)
- Every motion eases — no instant pops/snaps anywhere, including a burst/particle spawn, unless the source video demonstrably has a true hard cut (confirmed by identical pixel data frame-to-frame followed by a one-frame jump, not just "looks fast").
- A primary "draw" motion (stroke trim, path reveal) defaults to a pronounced ease-OUT (fast start, long soft stop), not a symmetric ease-in-out.
- Secondary/reactive motions (bursts, sparkles, follow-up flourishes) should start as the primary motion nears completion (~90% through), not after it fully finishes — no dead-air pause between "thing A is done" and "thing B starts." Overlap transitions.
- Literal measured timing from the source often reads as too fast/mechanical once rebuilt — default to ~20-25% slower than the raw measured durations for anything with visible motion (holds can stay at measured length).
- Double check any off-center compound shape (e.g. a checkmark inside a circle) against the true geometric center before calling it done — asymmetric shapes read as off-center more easily than expected.

## Preview GIF standard: white background

The delivered `preview.gif` for every animation renders on a solid white page background (`body{background:#fff}`), never a transparency checkerboard — checkerboard is a debugging aid, not a delivery format. Use it only for a one-off verification screenshot when checking that a background layer was actually removed (see Step 0, item 7); rebuild the actual `preview.gif` on white before it goes in `animations/<name>/`. This applies regardless of whether the underlying `.json` itself is transparent — transparency lives in the file's layers, not in how the preview happens to be rendered.

## Preview GIF pitfall: under-sampling makes a fine file look janky
The `celebration-burst` GIF preview looked choppy on first delivery — like it was running at ~5fps — even though the delivered `.json` was the untouched original, correctly authored at its real frame rate (70fps). The cause was the preview capture, not the animation: sampling every 4th frame and then displaying each captured frame for a fixed 120ms threw away most of the motion for anything moving fast (confetti pieces spinning/streaming quickly), so consecutive preview frames showed large jumps instead of smooth motion.

Fix: sample densely enough for the fastest motion in that specific animation (every 2nd frame is usually enough; drop to every frame for very fast/small motion), and set the GIF's per-frame duration to match real elapsed time (`step_frames / source_fr * 1000` ms), not a fixed guess like 120ms. Always sanity-check total GIF playback time against the source's real duration (`total_frames / fr` seconds) before sending — if they're off by more than ~10%, the preview is lying about speed and/or smoothness even when the underlying file is fine. This only affects the preview GIF; it never affects the delivered `.json`, but a janky preview understandably reads as "something's wrong with the animation" if not caught.

## Scope/limits
Well-suited to icon-style UI animations: strokes, fills, simple shape morphs, particle bursts, fades, scales. Much harder for organic motion, gradients, masks/mattes, or raster/photographic content — Lottie only expresses vector shape animation.

## Delivered so far
- **`animations/checkmark-burst/`** — full recreation pipeline, 4 review rounds, both keyframe pitfalls hit and fixed along the way.
- **`animations/heart-burst/`** — Step 0 shortcut, one pass, background layer stripped for transparency.
- **`animations/celebration-burst/`** — Step 0 shortcut, one pass, no cleanup even needed (no background layer to begin with).

See each animation's `NOTES.md` for specifics.
