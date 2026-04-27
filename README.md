# KR260 PetaLinux Project

Vivado block design project for the KR260 Starter Kit targeting `xck26-sfvc784-2LV-c`.  
The design implements a 10G Ethernet data path with DMA, AXI FIFOs, and an NI FPGA IP wrapper.

---

## PS-Accessible FIFOs and DMAs

All peripherals below are mapped into the Zynq PS (`zynq_ultra_ps_e_0`) address space via `M_AXI_HPM0_FPD` through the `slave_axi_mux` interconnect.

Each `axi_fifo_mm_s` instance exposes two AXI ports:
- **S_AXI (Mem0)** — AXI4-Lite control/status registers
- **S_AXI_FULL (Mem1)** — AXI4-Full high-bandwidth data port

---

### AXI DMA Controllers

| Instance | Description | Base Address | High Address | Size | AXI Data Width |
|---|---|---|---|---|---|
| `axi_dma_0` | Main 10G Ethernet DMA | `0xA002_0000` | `0xA002_FFFF` | 64 KB | 64-bit |
| `axi_dma_echo` | Echo loopback DMA | `0xA00D_0000` | `0xA00D_FFFF` | 64 KB | 64-bit |

Both DMAs are configured identically:
- MM2S burst size: 64 beats
- S2MM burst size: 8 beats
- S2MM DRE: enabled
- Scatter-Gather length width: 16-bit
- DMA masters access DDR via `SAXIGP0/HPC0_DDR_LOW`: `0x0000_0000 – 0x7FFF_FFFF` (2 GB)

---

### AXI FIFO MM_S Instances

| Instance | Description | Interface | Base Address | High Address | Size |
|---|---|---|---|---|---|
| `axi_fifo_mdebug` | MDEBUG stream from NiFpga IP | S_AXI (Lite/Status) | `0xA005_0000` | `0xA005_FFFF` | 64 KB |
| `axi_fifo_mdebug` | MDEBUG stream from NiFpga IP | S_AXI_FULL (Data) | `0xA006_0000` | `0xA006_FFFF` | 64 KB |
| `axi_fifo_debug` | DEBUG stream from NiFpga IP | S_AXI (Lite/Status) | `0xA007_0000` | `0xA007_FFFF` | 64 KB |
| `axi_fifo_debug` | DEBUG stream from NiFpga IP | S_AXI_FULL (Data) | `0xA008_0000` | `0xA008_FFFF` | 64 KB |
| `axi_fifo_cmd` | CMD stream from NiFpga IP | S_AXI (Lite/Status) | `0xA00A_0000` | `0xA00A_FFFF` | 64 KB |
| `axi_fifo_cmd` | CMD stream from NiFpga IP | S_AXI_FULL (Data) | `0xA00B_0000` | `0xA00B_FFFF` | 64 KB |
| `axi_dma_fifo_echo` | DMA echo loopback FIFO | S_AXI (Lite/Status) | `0xA00E_0000` | `0xA00E_FFFF` | 64 KB |
| `axi_dma_fifo_echo` | DMA echo loopback FIFO | S_AXI_FULL (Data) | `0xA00F_0000` | `0xA00F_FFFF` | 64 KB |
| `axi_fifo_echo` | Self-loopback echo FIFO | S_AXI (Lite/Status) | `0xA010_0000` | `0xA010_FFFF` | 64 KB |
| `axi_fifo_echo` | Self-loopback echo FIFO | S_AXI_FULL (Data) | `0xA011_0000` | `0xA011_FFFF` | 64 KB |

All `axi_fifo_mm_s` instances are configured with:
- AXI4 data width: 64-bit (`C_S_AXI4_DATA_WIDTH = 64`)
- Data interface type: AXI4 (`C_DATA_INTERFACE_TYPE = 1`)
- TX control: disabled

`axi_fifo_mdebug`, `axi_fifo_debug`, and `axi_fifo_cmd` have TX data disabled (receive-only from the PS perspective — data flows in from the NiFpga IP).  
`axi_dma_fifo_echo` and `axi_fifo_echo` have TX data enabled (bidirectional).

---

### AXI Stream FIFOs (internal pipeline — no PS register map)

These are clock-domain-crossing FIFOs in the data path. They are not directly addressable by the PS.

| Instance | Description | Depth | Data Width | Source | Sink |
|---|---|---|---|---|---|
| `tx_data_fifo` | Ethernet TX CDC FIFO | 8192 | 64-bit | `axi_dma_0` MM2S | `axis2xgmii_0` |
| `rx_data_fifo` | Ethernet RX CDC FIFO | 2048 | 64-bit | `xgmii2axis_0` | `axi_dma_0` S2MM |

Both use independent clock mode (`FIFO_MODE = 2`) to cross between `pl_clk0` (PS clock) and the XXV Ethernet TX/RX clocks.

---

### Complete Address Map Summary

| Base Address | High Address | Instance | Port |
|---|---|---|---|
| `0xA002_0000` | `0xA002_FFFF` | `axi_dma_0` | S_AXI_LITE |
| `0xA005_0000` | `0xA005_FFFF` | `axi_fifo_mdebug` | S_AXI (Lite/Status) |
| `0xA006_0000` | `0xA006_FFFF` | `axi_fifo_mdebug` | S_AXI_FULL (Data) |
| `0xA007_0000` | `0xA007_FFFF` | `axi_fifo_debug` | S_AXI (Lite/Status) |
| `0xA008_0000` | `0xA008_FFFF` | `axi_fifo_debug` | S_AXI_FULL (Data) |
| `0xA00A_0000` | `0xA00A_FFFF` | `axi_fifo_cmd` | S_AXI (Lite/Status) |
| `0xA00B_0000` | `0xA00B_FFFF` | `axi_fifo_cmd` | S_AXI_FULL (Data) |
| `0xA00D_0000` | `0xA00D_FFFF` | `axi_dma_echo` | S_AXI_LITE |
| `0xA00E_0000` | `0xA00E_FFFF` | `axi_dma_fifo_echo` | S_AXI (Lite/Status) |
| `0xA00F_0000` | `0xA00F_FFFF` | `axi_dma_fifo_echo` | S_AXI_FULL (Data) |
| `0xA010_0000` | `0xA010_FFFF` | `axi_fifo_echo` | S_AXI (Lite/Status) |
| `0xA011_0000` | `0xA011_FFFF` | `axi_fifo_echo` | S_AXI_FULL (Data) |
