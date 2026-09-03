# Checkmark Burst

Green checkmark-in-circle with a 13-point starburst — a "success" icon.

- **Source:** screen recording of a LottieFiles preview animation (no direct asset URL was found for this one)
- **Method:** measured from video, hand-rebuilt as Lottie JSON — see `CLAUDE.md` for the full recreation pipeline
- **Loop:** 2.57s @ 60fps, 154 frames
- **Timeline:** 0.4s blank → 0.5s circle stroke draws on → 0.18s checkmark draws on with a strong ease-out → burst starts as the checkmark hits 90% complete (no dead pause) → 0.7s starburst (13 dots ease-spawn then fly out while shrinking/fading) → 0.8s hold → cuts back to blank at the loop point
- Took 4 review rounds to get right — see the "First case" section of `CLAUDE.md` for what each round caught (including two real lottie-web keyframe bugs, now fixed for good in the shared build script)
