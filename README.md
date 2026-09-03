# Lottie.Animations

A personal collection of Lottie animations pulled from LottieFiles, without paying for the "free" ones LottieFiles gates behind a paid download.

## How

Two ways an animation ends up here, in order of preference:

1. **Use the real file directly.** Most LottieFiles preview pages load the actual animation file from a public, unauthenticated CDN URL to power their own preview player — the same file the paid "download" button would give you. Find that URL (usually visible in the page's network requests), grab it, and clean up only what's needed for the intended use (e.g. strip a baked-in opaque background so it composites transparently).
2. **Recreate it from a screen recording**, when no such direct file is available. Measure the animation's real geometry and timing from video, then hand-rebuild it as a Lottie JSON to match.

Full technical detail on both approaches — plus the two nasty Lottie/lottie-web keyframe bugs to avoid and the standing style preferences these all follow (easing, timing, overlap) — is in [`CLAUDE.md`](./CLAUDE.md). Read that before adding a new one.

## Animations

| | Name | Source | Notes |
|---|---|---|---|
| <img src="animations/checkmark-burst/preview.gif" width="120"> | [checkmark-burst](animations/checkmark-burst) | Recreated from a screen recording | 4 review rounds — see [NOTES.md](animations/checkmark-burst/NOTES.md) |
| <img src="animations/heart-burst/preview.gif" width="120"> | [heart-burst](animations/heart-burst) | Real file, found directly | Background layer stripped for transparency |
| <img src="animations/celebration-burst/preview.gif" width="120"> | [celebration-burst](animations/celebration-burst) | Real file, found directly | Used as-is, no cleanup needed |

Each animation's folder has the final `.json`, a `preview.gif`, and a `NOTES.md` with the specifics of how that one was made.

## Usage

Drop the `.json` into any Lottie player (`lottie-web`, `lottie-react`, After Effects via Bodymovin, etc.). Files with no background layer are already transparent; check each `NOTES.md` for anything animation-specific.
