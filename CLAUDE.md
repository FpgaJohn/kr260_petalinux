# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AMD KR260 Starter Kit project (part `xck26-sfvc784-2LV-c`) implementing a 10G Ethernet data path (XXV Ethernet) with scatter-gather DMA, AXI FIFOs, an NI FPGA IP wrapper (`NiFpgaIPWrapper_poc_ip`), and a `my_state` accumulator IP exposed through paired AXI-GPIO blocks.

The repo spans the full stack — Vivado block design, PetaLinux build, cross-development SDK, runtime-loadable Kria App overlay, userspace UIO test app, and a Vitis bare-metal workspace — but only `vivado/`, `kria_app/`, `src/`, `LIVE.md`, and `README.md` are git-tracked. `petalinux/` and `sdk/` are gitignored; `vitis/` and the BSP tarball are currently untracked.

## Repository Layout

```
kr260_petalinux/
├── CLAUDE.md, README.md, LIVE.md
├── vivado/                # Vivado 2024.1 block design + IP + constraints + exported XSA (tracked)
├── kria_app/              # Runtime overlay package: .bit.bin + .dtbo + shell.json (tracked)
├── src/                   # Userspace UIO C app + cross-compile/deploy/run Makefile (tracked)
├── petalinux/             # PetaLinux 2024.1 project tree, ~30 GB (gitignored)
├── sdk/                   # Extracted PetaLinux cross-dev SDK, 5.1 GB (gitignored)
├── vitis/                 # Vitis 2024.1 platform + bare-metal `c_hw_one` app (untracked)
└── xilinx-kr260-starterkit-v2024.1-05230256.bsp   # 1.7 GB BSP (untracked)
```

`LIVE.md` is the most up-to-date end-to-end session log — read it for the full state of the dev machine, the booted KR260, and the gotchas. `README.md` has the canonical address map and PetaLinux build commands. `kria_app/INSTALL.md` documents the overlay load procedure.

## Toolchain

All flows use the **Xilinx 2024.1 stack**: Vivado 2024.1, Vitis 2024.1, PetaLinux 2024.1. Source the tools before any `petalinux-*` / `bootgen` / `dtc` invocation:

```bash
source /tools/Xilinx/PetaLinux/2024.1/settings.sh
```

Userspace cross-compile uses the **Yocto SDK** (`cortexa72-cortexa53-xilinx-linux`, ARMv8/aarch64, GCC 12.2.0). Source it per shell:

```bash
source ~/work/kr260_petalinux/sdk/environment-setup-cortexa72-cortexa53-xilinx-linux
```

This sets `$CC`, `$SDKTARGETSYSROOT`, and the cross-compile flags that `src/Makefile` requires.

## High-Level Hardware Architecture

PS-side AXI master `zynq_ultra_ps_e_0/M_AXI_HPM0_FPD` fans out through `slave_axi_mux` to every PL peripheral. DMAs reach DDR via `SAXIGP0` (HPC0).

- **Main 10G data path:** `axi_dma_0` ↔ CDC FIFOs (`tx_data_fifo` / `rx_data_fifo`, independent-clock mode) ↔ `axis2xgmii` / `xgmii2axis` ↔ XXV Ethernet ↔ SFP. The CDC FIFOs cross between `pl_clk0` and the XXV TX/RX clocks.
- **Echo loopback path** (test instrumentation): `axi_dma_echo` + `axi_dma_fifo_echo` + `axi_fifo_echo` form a self-contained PS-driven loopback, independent of the Ethernet path.
- **NiFpga IP monitor streams:** `NiFpgaIPWrapper_poc_ip` exposes three RX-only `axi_fifo_mm_s` instances — `axi_fifo_mdebug`, `axi_fifo_debug`, `axi_fifo_cmd`. Each presents an AXI4-Lite status port (Mem0) and an AXI4-Full data port (Mem1) to the PS.
- **`my_state` accumulator** (custom IP, `vivado/ip/my_state.v`): 64-bit accumulator driven from `axi_gpio_control` (ch1 2-bit opcode = noop/add/reset, ch2 32-bit addend) and read back via `axi_gpio_value` (two 32-bit input channels). The opcode is edge-triggered — pulse `0 → opcode → 0`. `src/uio_test.c` is the working reference.

Full address map and FIFO configuration are in `README.md`.

## Three Deployment Paths

When the user's request is about getting the design onto the board, pick the matching path — they are not interchangeable.

### A. Kria App overlay (current default — factory firmware + runtime swap)
The board boots its factory K26 SOM firmware (QSPI/eMMC); we load our bitstream + DT overlay at runtime via `xmutil loadapp kr260_petalinux`. Build with `make` in `kria_app/`. See `kria_app/INSTALL.md` for the load procedure. This is what's currently working and what `src/uio_test.c` exercises.

### B. PetaLinux SD-card boot (full-stack)
Build a complete `petalinux-sdimage.wic` from `vivado/kr260_petalinux.xsa` and the official KR260 BSP, flash to SD, and switch SW1 boot mode to SD. Commands in `README.md` "PetaLinux" section. Note: `petalinux-package --wic` in 2024.1 dropped the `--disk-name` and `--bootfiles` arguments — use the no-argument form.

### C. Vitis bare-metal (`vitis/c_hw_one`)
Standalone C application (`dma.c`, `fifos.c`, `gpio.c`, `util.c`) running on the Cortex-A53 with FSBL on the same XSA. Useful for hardware bring-up before booting Linux. Currently untracked.

## Userspace Dev Workflow (path A)

After sourcing the SDK env (`. ~/work/kr260_petalinux/sdk/environment-setup-...`):

```bash
cd src
make quick      # cross-compile + scp + ssh-and-run
make debug      # launches gdbserver on the KR260 for remote debugging
```

The Makefile defaults to `KR260_HOST = kr260`. `~/.ssh/config` aliases `kr260-201` → `192.168.1.201` with key `~/.ssh/id_kr260` for passwordless login as `petalinux`.

## Two Runtime Gotchas Worth Knowing

These bit us in earlier sessions and are easy to repeat. Both are documented in detail in `LIVE.md` "Non-obvious things that broke".

1. **`/dev/uio*` doesn't appear after `xmutil loadapp` succeeds.** `uio_pdrv_genirq` ships with an empty `of_id` match table on factory Kria firmware. Reload it with the parameter set:
   ```bash
   sudo modprobe -r uio_pdrv_genirq
   sudo modprobe uio_pdrv_genirq of_id=generic-uio
   ```
   Make this persistent with `/etc/modprobe.d/uio.conf: options uio_pdrv_genirq of_id=generic-uio`.

2. **`/sys/firmware/fdt` md5 doesn't match the DTB you built.** The K26 SOM BootROM picks its boot source from SW1 pins; if SW1 isn't on SD, it boots factory QSPI regardless of what's on the SD card. Check with `xmutil listapps` and `cat /sys/class/fpga_manager/fpga0/state`. The "factory boot + Kria App overlay" path (A) sidesteps this entirely.

A `udev` rule on the target gives `petalinux`-group users mmap access to `/dev/uio*` without sudo:
```
/etc/udev/rules.d/99-uio.rules:
SUBSYSTEM=="uio", KERNEL=="uio*", GROUP="petalinux", MODE="0660"
```

## Recreating the Vivado Project

The `vivado/kr260_petalinux.tcl` script rebuilds the full project from checked-in sources. It must be sourced **from inside `vivado/`** (`origin_dir` defaults to `"."` and all paths are relative).

```bash
cd vivado
vivado -mode tcl
# in the Tcl shell:
source kr260_petalinux.tcl
```

The script creates the project and block design but does **not** launch synthesis, implementation, or bitstream generation — those are run manually from the GUI flow navigator. After bitstream, re-export the XSA (Vivado → File → Export → Export Hardware, *Include bitstream*) and re-commit `vivado/kr260_petalinux.xsa` so downstream PetaLinux/Kria-App builds pick up the change.

## Re-Export Workflow

The upstream sources live outside this repo, in the workspace's `fpganow/` tree (`fpganow/eth_pcs_pma_bare_2024.1/` and `fpganow/Market.Data.PoC/ip_export/`). The top-level workspace `CLAUDE.md` marks `fpganow/` as "Archived," but this project actively pulls from it on every re-export — treat `fpganow/` as the source-of-truth tree for the IP files listed below, even though no other active project uses it.

When the Vivado project is modified and `kr260_petalinux.tcl` is re-exported from Vivado, the TCL will contain absolute paths back to the original source locations on the developer's machine. Two things must be done:

### 1. Re-copy all source files

Copy the latest versions of all referenced files into the repo:

| Source (relative to `C:/work/fpganow/`) | Destination |
|---|---|
| `eth_pcs_pma_bare_2024.1/imports/xgmii_includes.vh` | `vivado/ip/` |
| `eth_pcs_pma_bare_2024.1/imports/axis2xgmii.v` | `vivado/ip/` |
| `eth_pcs_pma_bare_2024.1/imports/xgmii2axis.v` | `vivado/ip/` |
| `eth_pcs_pma_bare_2024.1/kr260_starter_kit.srcs/sources_1/new/my_state.v` | `vivado/ip/` |
| `eth_pcs_pma_bare_2024.1/kr260_starter_kit.srcs/sources_1/imports/kr260_starter_kit_wrapper.v` | `vivado/ip/` |
| `eth_pcs_pma_bare_2024.1/kr260_starter_kit.srcs/constrs_1/imports/kr260-xentech-10g/default.xdc` | `vivado/constraints/` |
| `eth_pcs_pma_bare_2024.1/kr260_starter_kit.srcs/constrs_1/imports/kr260-xentech-10g/Xentech.xdc` | `vivado/constraints/` |
| `eth_pcs_pma_bare_2024.1/archive_project_summary.txt` | `vivado/` |
| `Market.Data.PoC/ip_export/NiFpgaAG_poc_ip.v` | `vivado/ip/` |
| `Market.Data.PoC/ip_export/NiFpgaIPWrapper_poc_ip.vhd` | `vivado/ip/` |

### 2. Update path references in the TCL

The freshly exported TCL will contain paths like:
```
$origin_dir/../../fpganow/eth_pcs_pma_bare_2024.1/...
C:/work/fpganow/...
```

These must be replaced with local paths. There are **9 locations** to update:

| Location | Old pattern | New pattern |
|---|---|---|
| Header comments (file listing) | `C:/work/fpganow/...` | `ip/...` or `constraints/...` |
| `checkRequiredFiles` — local files | `$origin_dir/../../fpganow/eth_pcs_pma_bare_2024.1/...` | `$origin_dir/ip/...` or `$origin_dir/constraints/...` |
| `checkRequiredFiles` — remote files | `$origin_dir/../../fpganow/Market.Data.PoC/ip_export/...` | `$origin_dir/ip/...` |
| `add_files` for `sources_1` | `${origin_dir}/../../fpganow/Market.Data.PoC/ip_export/...` | `${origin_dir}/ip/...` |
| `import_files` for `sources_1` | `${origin_dir}/../../fpganow/eth_pcs_pma_bare_2024.1/...` | `${origin_dir}/ip/...` |
| File property set for `.vhd` | `$origin_dir/../../fpganow/Market.Data.PoC/ip_export/NiFpgaIPWrapper_poc_ip.vhd` | `$origin_dir/ip/NiFpgaIPWrapper_poc_ip.vhd` |
| `get_files` lookup for `.vh` | `"imports/xgmii_includes.vh"` | `"ip/xgmii_includes.vh"` |
| `constrs_1` imports (×2 files) | `${origin_dir}/../../fpganow/eth_pcs_pma_bare_2024.1/.../kr260-xentech-10g/...` and `"kr260-xentech-10g/..."` | `${origin_dir}/constraints/...` and `"constraints/..."` |
| BD source import block (absolute paths) | `C:/work/fpganow/...` | `[file normalize "${origin_dir}/ip/..."]` |

### Notes on the TCL structure

- `origin_dir` defaults to `"."` — the directory the script is sourced from. All file paths are relative to this, so the script must be sourced from within `vivado/`.
- `orig_proj_dir` (line ~140) points to the original Vivado project location. It is **never used** to load any files — vestigial metadata, leave as-is.
- `utils_1` fileset is empty (the `.dcp` checkpoints from the original project were intentionally dropped).
- The script creates the project but does **not** launch synthesis or implementation.
- `kr260_starter_kit_wrapper.v` was originally generated by Vivado **2022.1** (see its file header) and carried into the 2024.1 project. If Vivado 2024.1 regenerates it from the BD, the header and minor formatting will change — expected, not a toolchain mismatch.

## Key Design Details

- **Part**: `xck26-sfvc784-2LV-c`
- **Top module**: `kr260_starter_kit_wrapper`
- **PS interface**: `zynq_ultra_ps_e_0` — `M_AXI_HPM0_FPD` → `slave_axi_mux` → all peripherals
- **DMA HP port**: `SAXIGP0` (HPC0), DDR range `0x0000_0000–0x7FFF_FFFF`
- **PL clock**: `pl_clk0` drives all PS-side logic; XXV Ethernet TX/RX clocks drive the MAC-side FIFOs
- **Hardware handoff**: `vivado/kr260_petalinux.xsa` is the exported XSA — regenerate and re-commit whenever the BD changes, since downstream PetaLinux and Kria App builds consume it.
- Full PS peripheral address map is documented in `README.md`.
