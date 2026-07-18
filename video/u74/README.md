# U74 progress video — hybrid production pack

**Status:** assets ready · **no render yet**  
**Language:** English  
**Format:** 16:9 · 1080p target · ~75 s  
**Style:** motion slides (screen-capture) + terminal clip(s)

## What’s in this folder

| File | Purpose |
| ---- | ------- |
| [`script.en.md`](script.en.md) | Voiceover with timestamps (~75 s) |
| [`storyboard.md`](storyboard.md) | Shot list: motion vs terminal, edit notes |
| [`terminal-recording.md`](terminal-recording.md) | Exact commands to record on VisionFive 2 |
| [`slides/index.html`](slides/index.html) | Branded slide deck — open in browser, record with OBS/etc. |

## How to produce (when ready to render)

1. **Record motion (~45 s)** — open `slides/index.html` fullscreen (1920×1080). Advance with → or Space; optional auto-play via `?autoplay=1`. Screen-record at 60 fps.
2. **Record terminal (~25 s)** — follow [`terminal-recording.md`](terminal-recording.md) on real VF2 hardware (or use placeholder if unavailable).
3. **Record voiceover** — read [`script.en.md`](script.en.md) in a quiet room; export WAV/MP3.
4. **Edit** — cut to [`storyboard.md`](storyboard.md); duck music under VO if used; end card = slide 6.
5. **Export** — H.264 1080p, ~8 Mbps; optional 9:16 centre crop for social.

## Site links to cite on end card

- [opensolvers.com](https://www.opensolvers.com)
- [VisionFive 2 board page](https://www.opensolvers.com/boards/VisionFive2.html)
- [EESSI/docs#818](https://github.com/EESSI/docs/pull/818)
- [easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436)

## Do not do yet

- Remotion / ffmpeg final render (explicitly deferred)
- Upload to YouTube or embed on site until review
