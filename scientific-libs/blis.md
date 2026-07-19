# BLIS

[BLIS](https://github.com/flame/blis) (BLAS-like Library Instantiation Software, FLAME group) on RISC-V — a **vector-vs-vector** DGEMM comparison against patched RVV **OpenBLAS** on the SpaceMiT X60.

BLIS ships hand-written RVV assembly level-3 microkernels under [`kernels/rviv/3/`](https://github.com/flame/blis/tree/master/kernels/rviv/3) (dynamic VLEN via `get_vlenb()`), selected by the **`rv64iv`** config target. This is not scalar-vs-vector — both libraries use RVV.

Benchmark source: [opensolvers/benchmarks/BLIS](https://github.com/opensolvers/benchmarks/tree/main/BLIS) — `build-blis.sh`, `run-ab.sh`, shared `bench_dgemm.c`, `verify_ctrsm.c`.

See also [BLAS (OpenBLAS)](blas.html) for the OpenBLAS fixes and verification suite that define the baseline here.

## Why link, not FlexiBLAS

Other BLAS-axis benchmarks ([HPL](../apps/hpl.html), [NumPy](numpy.html), [QE](../apps/qe.html)) swap backends at runtime via FlexiBLAS. **FlexiBLAS is not installed on the RV2**, so this A/B links the same unchanged `bench_dgemm.c` against each library in turn — still one variable (the BLAS implementation), identical `-O3 -march=rv64imafdcv_zvl256b`.

## Build & run

```bash
./build-blis.sh                    # ~10–20 min native; installs to $HOME/blis-install
BLIS_PREFIX=$HOME/blis-install \
OPENBLAS_LIB=$HOME/trsm-pr5830/libopenblas.a \
  ./run-ab.sh
```

Configure with **`rv64iv`** (RVV 1.0, dynamic VLEN=256) and **`--enable-threading=openmp`** — without OpenMP, DGEMM stays single-threaded (~2.7 GFLOP/s regardless of thread count). Do **not** use `sifive_rvv` (defaults to VLEN=128).

**Toolchain:** EESSI GCC **14.3.0** forced ahead of compat GCC 13.4.0; link with `-L$GCC14/lib -B$GCC14/lib` (see benchmarks README for the `libgomp.spec` trap).

## Correctness

| Check | Result |
| ----- | ------ |
| `verify_ctrsm` (BLIS `rv64iv`) | **2400 cases, 0 fails**, worst residual **2.55×10⁻⁷** (1- and 8-thread) |
| `C[0]` in DGEMM sweep | **245.24** identical for BLIS and OpenBLAS at every size — no NaN |

BLIS TRSM passes cleanly on X60 — unlike the stock OpenBLAS `_rvv_v1` TRSM VLEN bug caught by [`verify_ctrsm` in OpenBLAS/](https://github.com/opensolvers/benchmarks/tree/main/OpenBLAS).

## Performance — Orange Pi RV2 (X60, VLEN=256)

BLIS `061c2eb` (`rv64iv`, OpenMP) vs patched RVV OpenBLAS `0.3.33.dev` (`zvl128bp`), EESSI GCC 14.3.0. Square DGEMM, 3 reps, best GFLOP/s:

| Threads | N | BLIS | OpenBLAS | BLIS / OpenBLAS |
| ------: | --: | ---: | -------: | --------------: |
| 1 | 1024 | 1.99 | 2.13 | 0.93× |
| 1 | 2048 | 2.73 | 2.25 | **1.21×** |
| 1 | 4096 | 2.95 | 2.28 | **1.29×** |
| 8 | 1024 | 8.93 | 10.13 | 0.88× |
| 8 | 2048 | 9.60 | 10.83 | 0.89× |
| 8 | 4096 | 9.55 | 11.94 | 0.80× |

### Takeaways

- **Single thread, large N:** BLIS's RVV assembly microkernel **beats OpenBLAS by ~20–30%** once packing is amortized (N ≥ 2048). At N=1024 the two are within noise.
- **8 threads:** OpenBLAS scales slightly better (BLIS **0.80–0.89×**). Both get ~3.5–5× from 8 cores; BLIS's OpenMP path leaves headroom vs OpenBLAS threading.
- **Correctness first:** both backends numerically identical on DGEMM; BLIS TRSM verified independently.

## References

- BLIS RISC-V config: [`config_registry`](https://github.com/flame/blis/blob/master/config_registry) (`rv64iv`, `sifive_rvv`)
- RVV assembly kernels: [`kernels/rviv/3/`](https://github.com/flame/blis/tree/master/kernels/rviv/3)
- Prior RISC-V BLIS work: PR [#737](https://github.com/flame/blis/pull/737) (X280), [#832](https://github.com/flame/blis/pull/832), [#868](https://github.com/flame/blis/pull/868) (SG2042)
