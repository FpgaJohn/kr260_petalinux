# KR260 PetaLinux Project

Vivado block design project for the KR260 Starter Kit targeting `xck26-sfvc784-2LV-c`.  
The design implements a 10G Ethernet data path with DMA, AXI FIFOs, and an NI FPGA IP wrapper.

---

## Recreating the Project in Vivado 2024.1

### Prerequisites

- Vivado 2024.1 installed
- KR260 board files installed (the script targets `xck26-sfvc784-2LV-c`)

### Steps

**Option A — Vivado GUI**

1. Open Vivado 2024.1.
2. In the Tcl Console (bottom panel), navigate to the `vivado` directory:
   ```tcl
   cd {C:/work/kr260/kr260_petalinux/vivado}
   ```
3. Source the script:
   ```tcl
   source kr260_petalinux.tcl
   ```
4. Vivado will create the `kr260_starter_kit` project in the current directory and open it automatically.

**Option B — Vivado Tcl Shell (batch)**

1. Open a terminal and launch the Vivado Tcl shell:
   ```
   vivado -mode tcl
   ```
2. Navigate to the `vivado` directory and source the script:
   ```tcl
   cd {C:/work/kr260/kr260_petalinux/vivado}
   source kr260_petalinux.tcl
   ```

### After Project Creation

The script creates the block design but does not launch any runs. To generate the bitstream:

1. Generate the block design output products:
   **Flow Navigator → IP Integrator → Generate Block Design**
2. Run Synthesis:
   **Flow Navigator → Synthesis → Run Synthesis**
3. Run Implementation:
   **Flow Navigator → Implementation → Run Implementation**
4. Generate Bitstream:
   **Flow Navigator → Program and Debug → Generate Bitstream**

### Repository Structure

```
vivado/
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

### AXI GPIO and IIC Peripherals

| Instance | Description | Base Address | High Address | Size | Width / Direction |
|---|---|---|---|---|---|
| `axi_gpio_0` | Single-channel output | `0xA000_0000` | `0xA000_FFFF` | 64 KB | 2-bit output |
| `axi_gpio_1` | Single-channel output | `0xA009_0000` | `0xA009_FFFF` | 64 KB | 1-bit output |
| `axi_gpio_control` | Dual-channel output | `0xA004_0000` | `0xA004_FFFF` | 64 KB | ch1 2-bit out, ch2 32-bit out |
| `axi_gpio_value` | Dual-channel input | `0xA00C_0000` | `0xA00C_FFFF` | 64 KB | ch1 32-bit in, ch2 32-bit in |
| `axi_iic_0` | I²C controller (SFP module mgmt) | `0xA001_0000` | `0xA001_FFFF` | 64 KB | (IRQ 91) |

The GPIO IPs bind to the in-tree `xlnx,axi-gpio-2.0` driver (gpiolib). The IIC IP binds to `xlnx,axi-iic-2.1` (i2c-xiic) and is wired to the SFP module I²C bus on the SOM240 connector.

Pairings between `axi_gpio_control` / `axi_gpio_value` and the `my_state` custom IP (2-bit control input → 64-bit accumulator output) are inferred from port widths but should be confirmed against the block design.

---

### Complete Address Map Summary

| Base Address | High Address | Instance | Port |
|---|---|---|---|
| `0xA000_0000` | `0xA000_FFFF` | `axi_gpio_0` | S_AXI |
| `0xA001_0000` | `0xA001_FFFF` | `axi_iic_0` | S_AXI |
| `0xA002_0000` | `0xA002_FFFF` | `axi_dma_0` | S_AXI_LITE |
| `0xA003_0000` | `0xA003_FFFF` | `xxv_ethernet_0` | S_AXI |
| `0xA004_0000` | `0xA004_FFFF` | `axi_gpio_control` | S_AXI |
| `0xA005_0000` | `0xA005_FFFF` | `axi_fifo_mdebug` | S_AXI (Lite/Status) |
| `0xA006_0000` | `0xA006_FFFF` | `axi_fifo_mdebug` | S_AXI_FULL (Data) |
| `0xA007_0000` | `0xA007_FFFF` | `axi_fifo_debug` | S_AXI (Lite/Status) |
| `0xA008_0000` | `0xA008_FFFF` | `axi_fifo_debug` | S_AXI_FULL (Data) |
| `0xA009_0000` | `0xA009_FFFF` | `axi_gpio_1` | S_AXI |
| `0xA00A_0000` | `0xA00A_FFFF` | `axi_fifo_cmd` | S_AXI (Lite/Status) |
| `0xA00B_0000` | `0xA00B_FFFF` | `axi_fifo_cmd` | S_AXI_FULL (Data) |
| `0xA00C_0000` | `0xA00C_FFFF` | `axi_gpio_value` | S_AXI |
| `0xA00D_0000` | `0xA00D_FFFF` | `axi_dma_echo` | S_AXI_LITE |
| `0xA00E_0000` | `0xA00E_FFFF` | `axi_dma_fifo_echo` | S_AXI (Lite/Status) |
| `0xA00F_0000` | `0xA00F_FFFF` | `axi_dma_fifo_echo` | S_AXI_FULL (Data) |
| `0xA010_0000` | `0xA010_FFFF` | `axi_fifo_echo` | S_AXI (Lite/Status) |
| `0xA011_0000` | `0xA011_FFFF` | `axi_fifo_echo` | S_AXI_FULL (Data) |

---

## PetaLinux

Commands for generating a PetaLinux 2024.1 project for the KR260 from this repo's exported `kr260_petalinux.xsa`.

### Prerequisites

- PetaLinux 2024.1 installed at `/tools/Xilinx/PetaLinux/2024.1`
- KR260 starter-kit BSP at `~/work/xilinx-kr260-starterkit-v2024.1-05230256.bsp` (~1.7 GB, intentionally not checked into this repo)
- This Vivado project synthesized through bitstream so `vivado/kr260_petalinux.xsa` is current

### Path A — From the official KR260 BSP (recommended)

The BSP ships with a working PS-only XSA, so you can boot first and merge the custom XSA in afterwards.

```bash
# 1. Source the tools (required before any petalinux-* command)
source /tools/Xilinx/PetaLinux/2024.1/settings.sh

# 2. Create the project from the BSP (run from this repo's root)
cd ~/work/kr260_petalinux
petalinux-create --type project \
    --source ../xilinx-kr260-starterkit-v2024.1-05230256.bsp \
    --name petalinux

cd petalinux

# 3. (Optional) swap in this repo's XSA so the FPGA bitstream + device tree
#    match this Vivado design. Pass the *directory* containing the .xsa.
petalinux-config --get-hw-description=../vivado --silentconfig

# 4. Build (takes ~30–90 min the first time)
petalinux-build

# 5. Package a bootable SD-card image
petalinux-package --boot --u-boot --fpga --force
petalinux-package --wic \
    --images-dir images/linux/ \
    --bootfiles "ramdisk.cpio.gz.u-boot,boot.scr,Image,system.dtb,system-zynqmp-sck-kr-g-revB.dtb" \
    --disk-name "sda"
```

The resulting `images/linux/petalinux-sdimage.wic` is what you flash to the SD card (`dd` or BalenaEtcher).

### Path B — From scratch using the XSA only (no BSP)

Minimal project with no Kria-specific layers pre-loaded. You'll have to add KR260-specific device-tree fragments (SOM240 carrier, FAN/SFP GPIO pins, etc.) yourself.

```bash
source /tools/Xilinx/PetaLinux/2024.1/settings.sh

petalinux-create --type project --template zynqMP --name petalinux
cd petalinux
petalinux-config --get-hw-description=../vivado   # interactive menu — Save & Exit
petalinux-build
petalinux-package --boot --u-boot --fpga --force
```

### Updating the XSA after a Vivado change

If you re-export the XSA after modifying the block design, clear stale build artifacts before reconfiguring:

```bash
cd petalinux
petalinux-build -x mrproper
petalinux-config --get-hw-description=../vivado --silentconfig
petalinux-build
```

### Optional: customize kernel / rootfs

```bash
petalinux-config -c kernel    # kernel menuconfig
petalinux-config -c rootfs    # add packages, change init, etc.
```

---

## Live Configuration

* If nothing appears in /dev/uio or as the output of:

Run the following commands to bind the correct driver:
```
sudo modprobe -r uio_pdrv_genirq
sudo modprobe uio_pdrv_genirq of_id=generic-uio
ls /dev/uio*
```


Then everything appears:
```
for i in /sys/class/uio/uio*; do
  printf "%s\t%-12s @ %s\n" "$(basename $i)" "$(cat $i/name)" "$(cat $i/maps/map0/addr)"
done
```

Output:
```
uio0    axi-pmon     @ 0x00000000ffa00000
uio1    axi-pmon     @ 0x00000000fd0b0000
uio10   axi_fifo     @ 0x00000000a00e0000
uio11   axi_fifo     @ 0x00000000a0100000
uio2    axi-pmon     @ 0x00000000fd490000
uio3    axi-pmon     @ 0x00000000ffa10000
uio4    gpio         @ 0x00000000a0040000
uio5    axi_fifo     @ 0x00000000a0050000
uio6    axi_fifo     @ 0x00000000a0070000
uio7    axi_fifo     @ 0x00000000a00a0000
uio8    gpio         @ 0x00000000a00c0000
uio9    dma          @ 0x00000000a00d0000
```

* To make it stick across reboots:
  * But first you have to make it load the kr26_petalinux app by default

```
echo 'options uio_pdrv_genirq of_id=generic-uio' | sudo tee /etc/modprobe.d/uio.conf
```


---

## Live Validation

sudo xmutil listapps && echo && \
for i in /sys/class/uio/uio*; do
  printf "%s\t%-12s @ %s\n" "$(basename $i)" "$(cat $i/name)" "$(cat $i/maps/map0/addr)"
done

uio0    axi-pmon     @ 0x00000000ffa00000
uio1    axi-pmon     @ 0x00000000fd0b0000
uio10   axi_fifo     @ 0x00000000a0100000
uio2    axi-pmon     @ 0x00000000fd490000
uio3    axi-pmon     @ 0x00000000ffa10000
uio4    dma          @ 0x00000000a0020000
uio5    axi_fifo     @ 0x00000000a0050000
uio6    axi_fifo     @ 0x00000000a0070000
uio7    axi_fifo     @ 0x00000000a00a0000
uio8    dma          @ 0x00000000a00d0000
uio9    axi_fifo     @ 0x00000000a00e0000
