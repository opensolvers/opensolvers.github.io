# Storyboard — hybrid (motion + terminal)

**Total runtime:** ~75 s · **Aspect:** 16:9 · **No render yet**

Legend: **M** = motion (record `slides/index.html`) · **T** = terminal (record on VF2)

| # | Time | Type | Visual | Audio (script section) | Notes |
| - | ---- | ---- | ------ | ---------------------- | ----- |
| 1 | 0:00–0:06 | **M** | Slide 1 — title + tagline | “OpenSolvers benchmarks…” | Fade in from black |
| 2 | 0:06–0:14 | **M** | Slide 2 — board + Scalar highlight | “…VisionFive 2… Scalar path” | Pulse highlight on Scalar box |
| 3 | 0:14–0:28 | **M** | Slide 3 — Before 3.13 GFLOP/s | “Out of the box… three point one three” | Animate “Before” row |
| 4 | 0:28–0:38 | **M** | Slide 4 — stack diagram | “We built U74-tuned OpenBLAS…” | Hold while VO explains FlexiBLAS |
| 5 | 0:38–0:52 | **T** | Terminal: EESSI + FlexiBLAS + xhpl snippet | “Same HPL binary… six point three” | See `terminal-recording.md`; cut to best 12–14 s |
| 6 | 0:52–1:05 | **M** | Slide 5 — After 5.28 + speedup table | “End-to-end HPL… one hundred twenty-six” | Animate After row + 1.69× badge |
| 7 | 1:05–1:15 | **M** | Slide 6 — end card + links | “Same methodology… opensolvers dot com” | Hold 3 s after VO ends |

## Edit rhythm

- **Crossfade** M↔T at shot 4→5 and 5→6 (0.3 s).
- **Lower third** optional on terminal clip: `VisionFive 2 · EESSI 2025.06 · FlexiBLAS swap`.
- **Music:** optional subtle bed (−18 dB under VO); none in terminal clip.
- **Captions:** burn-in key numbers (3.13 → 5.28 GFLOP/s) on slide 5.

## Asset checklist before recording

- [ ] VisionFive 2 powered, EESSI/CVMFS working
- [ ] U74 OpenBLAS module or FlexiBLAS backend registered
- [ ] Terminal font ≥ 18 pt, dark theme, 1920×1080 display scaling 100%
- [ ] `slides/index.html` tested fullscreen in Chrome/Safari
- [ ] Mic test; room tone sample for noise reduction

## Deferred (not in this pass)

- Remotion project / automated render
- Background music file
- YouTube thumbnail
