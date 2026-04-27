# CLAUDE.md — KR260 PetaLinux Project

## Project Overview

Vivado 2024.1 block design project for the AMD KR260 Starter Kit, targeting part `xck26-sfvc784-2LV-c`.  
The design implements a 10G Ethernet data path (XXV Ethernet) with scatter-gather DMA, AXI FIFOs, and an NI FPGA IP wrapper (`NiFpgaIPWrapper_poc_ip`).

## Repository Structure

```
kr260_petalinux/
├── CLAUDE.md
├── README.md
└── vivado/
    ├── kr260_petalinux.tcl       # Vivado project recreation script
    ├── archive_project_summary.txt
    ├── ip/                        # Verilog/VHDL source files
    │   ├── axis2xgmii.v
    │   ├── xgmii2axis.v
    │   ├── xgmii_includes.vh
    │   ├── my_state.v
    │   ├── kr260_starter_kit_wrapper.v
    │   ├── NiFpgaAG_poc_ip.v
    │   └── NiFpgaIPWrapper_poc_ip.vhd
    └── constraints/               # XDC constraint files
        ├── default.xdc
        └── Xentech.xdc
```

## Re-Export Workflow

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

- `origin_dir` defaults to `"."` — the directory the script is sourced from. All file paths are relative to this, so the script must be sourced from within the `vivado/` directory.
- `orig_proj_dir` (line ~140) points to the original Vivado project location. It is **never used** to load any files — it is vestigial metadata and can be left as-is.
- `utils_1` fileset is empty in this project (the `.dcp` checkpoint files from the original project were intentionally dropped).
- The script creates the project but does **not** launch synthesis or implementation. Those must be run manually after sourcing.

## Key Design Details

- **Part**: `xck26-sfvc784-2LV-c`
- **Top module**: `kr260_starter_kit_wrapper`
- **PS interface**: `zynq_ultra_ps_e_0` — `M_AXI_HPM0_FPD` → `slave_axi_mux` → all peripherals
- **DMA HP port**: `SAXIGP0` (HPC0), DDR range `0x0000_0000–0x7FFF_FFFF`
- **PL clock**: `pl_clk0` drives all PS-side logic; XXV Ethernet TX/RX clocks drive the MAC-side FIFOs
- Full PS peripheral address map is documented in `README.md`
