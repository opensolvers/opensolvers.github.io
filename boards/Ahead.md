---
title: BeagleV-Ahead — T-Head C910
description: BeagleV-Ahead (TH1520) — quad Xuantie C910 at 1.85 GHz, 3-wide out-of-order, draft RVV 0.7.1 (xtheadvector, 128-bit), GhostWrite, plus the on-chip C906, 4 TOPS NPU, and BXM-4-64.
---

The BeagleV-Ahead is a BeagleBone-sized board on the T-Head **TH1520**. The CPUs that run Linux are four **Xuantie C910** cores. Beagle’s board manual clocks that cluster at **1.85 GHz**; the [product page](https://www.beagleboard.org/boards/beaglev-ahead) lists **2 GHz**. **4 GB** LPDDR4, **16 GB** eMMC. The factory image is still on the eMMC (**THEAD C910 Release Distro 1.1.2**, kernel `5.10.113-yocto-standard`, built 2023-06-10). Ubuntu 24.04.3 now boots from the SD card. How that boot and its Wi-Fi were brought up is below.

Product page: [BeagleV-Ahead](https://www.beagleboard.org/boards/beaglev-ahead) · [docs](https://docs.beagleboard.org/boards/beaglev/ahead/index.html).

## The CPU: Xuantie C910

C910 is T-Head’s first out-of-order application core, and the open RTL behind the TH1520 cluster (up to four cores sharing an L2). On this SoC that cluster is the whole Linux SMP set.

| | **C910** on the Ahead |
| --- | --- |
| Count | **4** (one cluster) |
| Clock | **1.85 GHz** in the board manual; product page says **2 GHz** |
| Pipeline | **3-wide**, **12-stage**, out-of-order issue and completion, in-order retirement; **8** execution ports |
| L1 | **64 KB** I + **64 KB** D per core |
| L2 | **1 MB** shared, inclusive |
| Scalar ISA | **RV64GC** |
| Vector | Draft **RVV 0.7.1**, exposed as **`xtheadvector`**, **128-bit** datapath |

Fetch is in order. Issue, execution, and completion are out of order, then retirement is in order again. The integer side has a dedicated branch port plus ALU ports; the FP/vector side is a dual pipe, and both pipes can do **128-bit** vector ops. The vector register file supplies the mask as well — RVV has no separate mask registers.

That vector unit is the part that does not match the rest of this site. X60 and X100 implement **ratified RVV 1.0** (`Zvl256b` on those chips, VLEN=256). C910 implements the **0.7.1 draft**. The encodings and the programming model differ (`xtheadvector` versus `v`). A binary built with `-march=rv64gcv` for the Orange Pi RV2 or the BPI-SM10 does not run on this core. Tooling has to target T-Head’s vector extension explicitly.

**GhostWrite** ([CVE-2024-44067](https://nvd.nist.gov/vuln/detail/CVE-2024-44067)) is a bug in that same vector unit: a faulty store uses a physical address, so an unprivileged process can write memory outside its own mapping. It is a hardware erratum on the C910 in the TH1520 (and on the related C920). The mitigation in later kernels is to **turn `xtheadvector` off**. The factory 5.10 image on this board predates that erratum. With the vector unit disabled, the C910 that remains is the 3-wide scalar RV64GC core — the 128-bit pipes stay dark.

## Other compute on the same chip

These are not the Linux CPUs. They sit beside the C910 cluster and have their own drivers.

| Block | What it is |
| --- | --- |
| **C906** | One smaller T-Head core for the **audio** subsystem. It does not join the 4-core SMP set. |
| **NPU** | **4 TOPS INT8** at 1 GHz. Vendor neural path (TensorFlow / ONNX / Caffe in T-Head’s stack), separate from the C910 vector pipes. |
| **GPU** | Imagination **BXM-4-64**. Beagle rates it around **50 GFLOPS**, with GLES / Vulkan / OpenCL via the usual PowerVR stack. Same GPU family as the [BPI-SM10](SM10.html), different SoC and driver story. |

## Where this sits next to the other boards

| Board | Application cores | Vector |
| --- | --- | --- |
| [VisionFive 2](VisionFive2.html) | 4× SiFive U74, in-order | none |
| **BeagleV-Ahead** | 4× C910, out-of-order | draft RVV 0.7.1, 128-bit |
| [Orange Pi RV2](RV2.html) / [BPI-F3](F3.html) | 8× X60 | RVV 1.0, VLEN=256, plus IME1 |
| [BPI-SM10](SM10.html) | 8× X100 + 8× A100 | RVV 1.0, VLEN=256 / 1024, plus IME2 |

## Ubuntu 24.04 from the SD card

The image is the Beagle [xuantie-ubuntu](https://openbeagle.org/beaglev-ahead/xuantie-ubuntu) console build, kernel `6.15.11-20251216+`, Ubuntu 24.04.3. Serial login is `beagle` / `beagle`, hostname `beaglev`, 115200 8N1. Write it to the SD card as a normal GPT image (boot ext4, then root ext4, extlinux `root=/dev/mmcblk1p3`). Leave sector 0 as the GPT. The SPL and the partition table cannot share that sector.

The ROM loads U-Boot from the eMMC boot partition. The Ubuntu U-Boot (`2020.01`, prompt `C910 Light#`, `bootdelay=2`) has `boot_targets=mmc1 mmc0`, so a plain reset tries the SD card and falls back to the factory eMMC root. Do not hold the SD button. To boot the factory system on purpose, stop autoboot and run `bootcmd_mmc0`.

Kernel 6.15 hangs at `smp: Bringing up secondary CPUs` when the secondary harts are started from the factory OpenSBI 0.9 image (85856 bytes on the eMMC). The boot that brings up all four CPUs loads the SD card’s `fw_dynamic.bin` (119104 bytes) from `mmc 1:2` to address 0 and still runs `bootslave` before distro boot. Hart 0 keeps reporting OpenSBI 0.9. That pair is what prints `smp: Brought up 1 node, 4 CPUs`.

`reboot` and `reboot -f` on this Ubuntu call SBI system reset and trap in OpenSBI 0.9 (`sbi_trap_error`). Use the reset button.

Kernel 6.15 includes the GhostWrite mitigation and hides `xtheadvector`. The GEMM numbers below were measured on the factory 5.10 kernel, which still exposes the extension.

## Wi-Fi and SSH

The onboard module is an AMPAK AP6203BM. The kernel sees it as SDIO `BCM43012/2` on `mmc@ffe70a0000`. The stock device tree leaves that slot looking removable, so the host never scans it. After the device-tree changes below, a reset prints `mmc2: new high speed SDIO card`.

On a copy of `/boot/firmware/th1520-beaglev-ahead.dtb`:

- Add `/wifi-pwrseq` (`compatible = "mmc-pwrseq-simple"`, `reset-gpios` GPIO2 pin 31 active-low, `post-power-on-delay-ms = <200>`).
- On `mmc@ffe70a0000`: `non-removable`, `no-sd`, `no-mmc`, `mmc-pwrseq`, `no-1-8-v`, and `max-frequency = <50000000>`.
- Delete `interrupts`, `interrupt-parent`, and `interrupt-names` from `wifi@1`.

`no-1-8-v` keeps the bus at 3.3 V high-speed. At 1.8 V DDR50 the card enumerates and then dies with `brcmf_sdio_htclk: HT Avail timeout` (`clkctl 0x50`). The stock host-wake pin is `SDIO1_DETN`, still muxed as SDIO. With that interrupt in the tree, `brcmfmac` waits for an out-of-band IRQ that never arrives (`dongle is not responding`). In-band SDIO interrupts work. Leave `function = "gpio"` off the MMC pin group: the GPIO controller and the MMC pinctrl then both claim GPIO2_31, and the SDIO host fails to probe.

linux-firmware’s `brcmfmac43012-sdio.bin` and the Cypress `cyfmac43012-sdio.bin` do not start this module. The vendor files already on the rootfs do. Copy them onto the names `brcmfmac` requests, including the board-specific `beagle,beaglev-ahead` names:

```sh
cp /lib/firmware/fw_bcm43013c1_ag.bin /lib/firmware/brcm/brcmfmac43012-sdio.bin
cp /lib/firmware/fw_bcm43013c1_ag.bin /lib/firmware/brcm/brcmfmac43012-sdio.beagle,beaglev-ahead.bin
cp /lib/firmware/nvram_ap6203bm.txt /lib/firmware/brcm/brcmfmac43012-sdio.txt
cp /lib/firmware/nvram_ap6203bm.txt /lib/firmware/brcm/brcmfmac43012-sdio.beagle,beaglev-ahead.txt
cp /lib/firmware/clm_bcm43013c1_ag.blob /lib/firmware/brcm/brcmfmac43012-sdio.clm_blob
```

A working probe logs `Firmware: BCM43012/2 wl0: Aug 19 2022 09:56:39 version 18.35.389.80` and creates `wlan0` with MAC `50:41:1c:cf:ce:ca`. These file copies and the device tree take effect on the next reset.

The image uses iwd. This kernel is built without `CONFIG_RFKILL`, so `/dev/rfkill` does not exist and iwd 2.20 exits immediately (`Module rfkill failed to start: -2`). A small preload supplies a dummy file descriptor for that open. The stock iwd unit is `DevicePolicy=closed` and only allows `/dev/rfkill`, so the drop-in also sets `DevicePolicy=auto`.

```sh
gcc -shared -fPIC -o /usr/local/lib/librfkill-shim.so rfkill-shim.c -ldl
mkdir -p /etc/systemd/system/iwd.service.d
printf '%s\n' '[Service]' \
  'Environment=LD_PRELOAD=/usr/local/lib/librfkill-shim.so' \
  'DevicePolicy=auto' \
  > /etc/systemd/system/iwd.service.d/rfkill.conf
printf '\n[General]\nEnableNetworkConfiguration=true\n' >> /etc/iwd/main.conf
systemctl daemon-reload
systemctl restart iwd
```

`rfkill-shim.c` interposes `open`, `open64`, and `ioctl`. For the path `/dev/rfkill` it returns one end of a `socketpair` and makes `ioctl` on that descriptor succeed. Every other path goes to libc.

Put the network in `/var/lib/iwd/<ssid>.psk`, mode `0600`:

```ini
[Security]
Passphrase=<the passphrase>
```

If the SSID has a trailing space, that space is part of the filename. iwd then associates and runs DHCP. On this desk the lease was `192.168.1.227/24`. `eth0` stays down until a cable is plugged in.

`ssh.socket` is already listening. `ufw allow OpenSSH` fails here (`Couldn't determine iptables version`) and is unnecessary. An `eessi` account in group `sudo`, with `eessi ALL=(ALL) NOPASSWD:ALL` and an `authorized_keys` entry, logs in with a key:

```sh
ssh -i ~/.ssh/id_rsa eessi@192.168.1.227
```

## Measured

[EESSI](../eessi.html) `2025.06-001` is mounted (`software.eessi.io` and `dev.eessi.io`). Init selects `riscv64/generic`. That tree is RVV 1.0, so it supplies the compatibility layer and the generic OpenBLAS, not a C910 vector build. Stock `HPL/2.3-foss-2025b` runs on it: N=2000, 2×2, **3.77 GFLOP/s**, residual PASSED. That problem fits in cache.

OpenBLAS **0.3.34** `TARGET=C910V` is a local build with EESSI GCC 14.3 and `-march=rv64gc_xtheadvector`. Upstream binutils names the 0.7 instructions `th.vle.v` and `th.vfmacc.vv`; the stock Xuantie march string does not assemble. Clock pinned at **1.848 GHz**. All-ones square GEMM, `C[0] = N` at every size. Full notes: [benchmarks/OpenBLAS](https://github.com/opensolvers/benchmarks/blob/main/OpenBLAS/README.md).

| | 1 core | 4 cores |
| --- | ---: | ---: |
| DGEMM N=2048 | 2.56 | 7.42 |
| DGEMM N=4096 | | **7.42** |
| SGEMM N=2048 | 5.72 | 16.69 |
| SGEMM N=4096 | | **16.58** |

GFLOP/s. Four cores reach about 2.7× one core. Single precision is a bit over twice double precision, which matches the 128-bit vector pipe.

CBLAS Level 2 (all four precisions) and complex Level 3 pass. Real and double Level 3: `trmm` and `trsm` pass; `gemm`, `symm`, `syrk`, and `syr2k` fail the netlib ratio on small shapes, including K=0. The all-ones GEMM sizes above still match.

## Related

- [BeagleV-Ahead](https://www.beagleboard.org/boards/beaglev-ahead) — product page
- [Beagle docs](https://docs.beagleboard.org/boards/beaglev/ahead/index.html)
- [GhostWrite](https://ghostwriteattack.com/) — CVE-2024-44067
- [Chips and Cheese: Xuantie C910](https://chipsandcheese.com/p/alibabat-heads-xuantie-c910) — pipeline and caches on the TH1520
- [xuantie-ubuntu pipelines](https://openbeagle.org/beaglev-ahead/xuantie-ubuntu) — Ubuntu 24.04 images
- [opensolvers/benchmarks](https://github.com/opensolvers/benchmarks)
