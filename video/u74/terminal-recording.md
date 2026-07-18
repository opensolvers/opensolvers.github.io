# Terminal recording guide — VisionFive 2

Record **1080p**, **60 fps**, terminal font **18–20 pt**, **dark background**.  
Target clip length after edit: **12–14 s** (storyboard shot 5).

## What the viewer should understand

One unchanged HPL binary; only the BLAS backend changes via FlexiBLAS.

## Pre-flight (off camera)

```bash
# EESSI on riscv64 — adjust paths to your install
export EESSI_VERSION_OVERRIDE=2025.06-001
source /cvmfs/dev.eessi.io/latest/init/bash  # or your CVMFS init

module load HPL/2.3-foss-2025b
module load FlexiBLAS  # if separate module
# Register U74 backend if built via easyconfigs#26436
# flexiblas add U74 /path/to/libopenblas.so
```

## On-camera sequence (type slowly; pause 1 s between blocks)

### Block A — context (3 s)

```bash
uname -m && lscpu | grep -E 'Model name|CPU\(s\):'
module list 2>&1 | tail -5
```

### Block B — FlexiBLAS swap (4 s)

```bash
flexiblas list
export FLEXIBLAS=OpenBLAS
export OPENBLAS_CORETYPE=RISCV64_GENERIC   # baseline label on screen
echo "Backend: generic (before)"

# cut in editor to patched backend:
export FLEXIBLAS=U74   # or path to your registered backend name
echo "Backend: U74 tuned (after)"
```

### Block C — HPL headline (5 s)

Run a **short** xhpl if time allows (N=8000 completes faster than N=10000), or show a previous log:

```bash
# Option 1 — live (may take minutes; prefer pre-captured log for demo)
OMP_NUM_THREADS=1 mpirun -np 4 xhpl 2>&1 | tee /tmp/hpl-u74.log

# Option 2 — show stored result lines only (recommended for 12 s clip)
grep -E 'T/V|WR11C2C|GFLOPS|PASSED|FAILED' /tmp/hpl-u74.log | tail -6
```

**Lines to highlight in post:** GFLOP/s rate and PASSED residual.

## If no hardware available

Use a **placeholder terminal clip**:

```bash
# Simulated output for edit mock — DO NOT present as live benchmark
cat << 'EOF'
HPLinpack 2.3 -- High-Performance Linpack benchmark
WR11C2C      126.00       5.2800e+04       5.28e+00
EOF
```

Label mock footage clearly in edit notes; replace with real capture before publish.

## Numbers to match voiceover (from site)

| Metric | Before | After |
| ------ | ------ | ----- |
| HPL GFLOP/s | 3.13 | 5.28 |
| HPL time (N=10000, 2×2) | 213 s | 126 s |
| Speedup | — | 1.69× |
| DGEMM 1-core | ~1.4 | 1.77 GFLOP/s |
| DGEMM 4-core | — | 6.31 GFLOP/s |

## Recording tools

- **macOS:** OBS Studio or QuickTime (crop to terminal window)
- **Linux on VF2:** `asciinema rec u74-demo.cast` then convert, or OBS
- Hide hostname/user if desired; prompt `opensolvers@visionfive2:~$` is fine
