# OpenSolvers website — decisions log

Context file for structural and content decisions on [opensolvers.com](https://www.opensolvers.com) (`opensolvers/opensolvers.github.io`).

**Workflow:** commit and deploy directly to `main` via PR (no need to ask first).

---

## Site structure

| Area | Path | Purpose |
|------|------|---------|
| Homepage | `README.md` → `/` | Intro, scientific libs, apps, board summaries |
| Videos | `videos.md` → `/videos.html` | YouTube walkthroughs (`_data/videos.yml`) |
| Boards | `boards/` | Per-board hardware + benchmark notes |
| Apps | `apps/` | End-to-end application benchmarks (e.g. HPL) |
| Scientific libs | `scientific-libs/` | Library-level probes (BLAS, LAPACK, ELPA) |

## Navigation groups

1. **Home** · **Videos** · **YouTube** (external)
2. **Apps** — HPL, Quantum ESPRESSO, ONNX Runtime, GROMACS
3. **Scientific libs** — BLAS (incl. OpenBLAS verification), **BLIS**, NumPy, LAPACK, ELPA, MLAS, FFTW, ScaLAPACK
4. **Boards** — VisionFive 2, OrangePi RV2, BananaPi F3

Nav config: `_config.yml` (`navigation`, `navigation_boards`, `navigation_apps`, `navigation_scientific_libs`). Rendered in `_includes/header.html`. Cayman theme requires `_layouts/default.html` override to include the header.

## Videos

Catalog: `_data/videos.yml` (newest first). Rendered by `_includes/video-grid.html` on `videos.md`. Add a row per published YouTube video — no new page needed.

## SEO

| Item | Location |
|------|----------|
| Site `url`, `lang`, description | `_config.yml` |
| `{% seo %}` tags | `_layouts/default.html` (jekyll-seo-tag) |
| Organization + WebSite JSON-LD | `_includes/head-custom.html` |
| VideoObject JSON-LD | `_includes/video-grid.html` (per video) |
| `robots.txt` + sitemap | site root; sitemap from GitHub Pages / `url:` |
| Page titles/descriptions | YAML front matter on key pages |

## Content sources

| Source | Used for |
|--------|----------|
| [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks) | OpenBLAS (`OpenBLAS/`), **BLIS (`BLIS/`)**, HPL (`hpl/`), ELPA/ScaLAPACK (`elpa/`, `scalapack/`), QE/FFTW (`qe/`, `fftw/`), GROMACS (`gromacs/` incl. `rvv-backend/`), ONNX/MLAS (`onnx/`), NumPy (`numpy/`) |
| [EESSI/docs#818](https://github.com/EESSI/docs/pull/818) | VisionFive 2 / U74 OpenBLAS + HPL |
| [EESSI/docs#819](https://github.com/EESSI/docs/pull/819) | Orange Pi RV2 / X60 RVV `gemv_n` fix + HPL |
| [easyconfigs#26436](https://github.com/easybuilders/easybuild-easyconfigs/pull/26436) | U74 OpenBLAS package |
| [easyconfigs#26444](https://github.com/easybuilders/easybuild-easyconfigs/pull/26444) | X60 OpenBLAS package |

## Key numbering conventions

- **Before / After** tables — stock EESSI OpenBLAS 0.3.30 vs fixed EasyBuild module (FlexiBLAS swap, no app rebuild).
- **Orange Pi RV2 HPL:** stock EESSI = FAILED (`nan`) ~8.5 GFLOP/s; fixed peak = **10.53 GFLOP/s** (N=20000, 2×4). Scalar A/B baselines from `run-hpl-ab.sh`: **6.41** and **7.38 GFLOP/s** — these are *scalar* backends, not “after” results.
- **X60 DNF/FAIL** — OpenBLAS 0.3.30 RVV `gemv_n` bug; fixed in ≥ 0.3.31, EESSI bump ≥ 0.3.34 preferred.

## Changelog

| Date | Decision |
|------|----------|
| 2026-07-19 | Publish X60 gemv_n / HPL NaN video (`W_-8cKA-CCU`) — `_data/videos.yml`, homepage, RV2, HPL, BLAS |
| 2026-07-19 | Sync benchmarks PR #19–20: add **BLIS** page; expand **GROMACS** with RVV Force backend (3.31×) and 2026.3 autovec lever |
| 2026-07-19 | Nav: OpenSolvers title links home; Home/Videos/YouTube on separate row from Apps |
| 2026-07-19 | Sync benchmarks PR #18: merge OpenBLAS verification into BLAS page; remove `dgemm.html`; fix GitHub links |
| 2026-07-18 | Videos page + `_data/videos.yml`; U74 YouTube link; SEO (`url`, JSON-LD, robots.txt, page descriptions) |
| 2026-07-18 | Per-board compute-backend SVGs; fix invalid UTF-8 in VisionFive 2 / K1 diagrams |
| 2026-07-18 | U74 video production moved to private repo `opensolvers/u74-video` (removed from site) |
| 2026-07-18 | Sync from benchmarks: GROMACS app, ScaLAPACK lib, FFTW QE end-to-end (~0%), updated QE/elpa/RV2 |
| 2026-07-17 | Add FFTW `r5v` RVV page from `benchmarks/fftw`; cross-links from QE and RV2 |
| 2026-07-17 | Add MLAS (lib) and ONNX Runtime (app) pages from `benchmarks/onnx` |
| 2026-07-15 | Vector box: add ZVL256B and ZVL128B subtitles |
| 2026-07-15 | Compute-backends diagram: RV64GC / IME subtitles; U74 DGEMM under Scalar |
| 2026-07-15 | Homepage: compute-backends illustration (Scalar / Vector / Specific / GPU) |
| 2026-07-14 | Nav layout: 3 stacked rows — Apps (top), Scientific libs (middle), Boards (bottom) |
| 2026-07-14 | Add Quantum ESPRESSO app page from `opensolvers/benchmarks/qe` |
| 2026-07-14 | IME (X60 `smt.vmadot`) section on RV2 and BPI-F3 board pages |
| 2026-07-14 | Add dedicated DGEMM and NumPy scientific-lib pages from benchmarks repo |
| 2026-07-14 | BPI-F3 results from `opensolvers/benchmarks`: HPL 11.52 GFLOP/s, 3.7 GB RAM limit |
| 2026-07-12 | Homepage scope widened: scientific libs + apps, not HPL-only |
| 2026-07-12 | Add `website.md`; sync HPL/BLAS from `opensolvers/benchmarks`; add scientific libs LAPACK + ELPA |
| 2026-07-12 | Split nav: Boards / Apps / Scientific libs; move HPL to `apps/` |
| 2026-07-12 | Fix homepage RV2 numbers (FAILED → 10.53 GFLOP/s, not 7.38 native) |
| 2026-07-11 | Override Cayman `_layouts/default.html` so `_includes/header.html` renders |
| 2026-07-11 | Site focus: RISC-V learnings; tagline “RISC-V learnings and fun” |
