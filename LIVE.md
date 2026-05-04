# LIVE.md — Session log: KR260 PetaLinux + Kria App + UIO development setup

End-state snapshot of an interactive session that took the KR260 design from
"just an XSA" to "running custom bitstream with userspace UIO access via
`make quick`". Captures the final layout, what's running where, and the
non-obvious things that broke along the way.

---

## What was accomplished, in one paragraph

Built a PetaLinux 2024.1 project from `vivado/kr260_petalinux.xsa`, generated
a flashable SD image and a 820 MB cross-development SDK, packaged the same
bitstream as a Kria App (`.bit.bin` + `.dtbo` + `shell.json`) for runtime
loading, deployed it to the booted KR260, debugged why UIO didn't bind
(turned out the kernel is running factory QSPI firmware, not our SD build,
and `uio_pdrv_genirq` needs `of_id=generic-uio`), set up persistent
permissions (udev rule) and SSH keys for a one-command edit/build/deploy/run
cycle.

---

## Repository layout (final)

```
~/work/kr260_petalinux/
├── CLAUDE.md                    # project guidance (updated with overview, XSA, fpganow link, etc.)
├── README.md                    # design doc + recreation + PetaLinux section + GPIO/IIC peripherals
├── LIVE.md                      # this file
├── .gitignore                   # ignores petalinux/ and sdk/
│
├── vivado/                      # Vivado source (existing)
│   ├── kr260_petalinux.tcl
│   ├── kr260_petalinux.xsa      # hardware handoff (committed)
│   ├── ip/
│   └── constraints/
│
├── petalinux/                   # gitignored, ~30 GB
│   ├── images/linux/
│   │   ├── BOOT.BIN              (7.9 MB)
│   │   ├── petalinux-sdimage.wic (6.1 GB, flashable)
│   │   ├── system.dtb            → system-zynqmp-sck-kr-g-revB.dtb
│   │   ├── Image, system.dtb, image.ub, ramdisk.cpio.gz.u-boot, ...
│   │   └── sdk.sh                (820 MB cross-dev SDK installer)
│   └── project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
│       (5 FIFOs overridden to compatible="generic-uio")
│
├── sdk/                         # gitignored, 5.1 GB extracted SDK
│   ├── environment-setup-cortexa72-cortexa53-xilinx-linux
│   ├── sysroots/cortexa72-cortexa53-xilinx-linux/
│   └── version-cortexa72-cortexa53-xilinx-linux
│
├── kria_app/                    # runtime-loadable Kria App
│   ├── INSTALL.md                # full install + load procedure
│   ├── Makefile                  # bootgen .bit→.bit.bin, dtc .dtso→.dtbo, package
│   ├── kr260_petalinux.bif       # bootgen input
│   ├── kr260_petalinux.dtso      # DT overlay (5 FIFOs + 2 DMAs as generic-uio,
│   │                             #  GPIO/I²C/XXV-eth with proper drivers)
│   ├── shell.json                # Kria manifest (XRT_FLAT)
│   └── build/kr260_petalinux/    # gitignored output package
│       ├── kr260_petalinux.bit.bin   (6.6 MB raw bitstream)
│       ├── kr260_petalinux.dtbo      (7.4 KB compiled overlay)
│       └── shell.json
│
└── src/                         # userspace app skeleton
    ├── Makefile                  # build / deploy (scp) / run (ssh) / debug (gdbserver) targets
    ├── uio_test.c                # finds uioN by base addr 0xA00A0000, mmaps, prints registers
    └── uio_test                  # cross-compiled aarch64 ELF
```

---

## Dev-machine state

- **SDK installed** at `~/work/kr260_petalinux/sdk/` (5.1 GB).
  Activate per shell: `. ~/work/kr260_petalinux/sdk/environment-setup-cortexa72-cortexa53-xilinx-linux`
  This sets `$CC = aarch64-xilinx-linux-gcc 12.2.0`, `$SDKTARGETSYSROOT`, and
  cross-compile `$LDFLAGS`/`$CFLAGS`.
- **SSH key** `~/.ssh/id_kr260` (ed25519, no passphrase, comment "kr260-app-dev").
- **`~/.ssh/config`** appended with:
  ```
  Host kr260-201
      HostName 192.168.1.201
      User petalinux
      IdentityFile ~/.ssh/id_kr260
      StrictHostKeyChecking accept-new
  ```
  (the existing `Host kr260` for .198 was left alone)

---

## KR260 board state

- **Boot source**: factory firmware on the SOM (QSPI/eMMC), **not** our SD-card
  build. Confirmed by `md5sum /sys/firmware/fdt` on target (`a83b1de2…`)
  vs our build (`b2fc79af…`). The user's SW1 boot-mode pins are likely set
  to QSPI rather than SD.
- **App loaded**: `kr260_petalinux` (XRT_FLAT) installed at
  `/lib/firmware/xilinx/kr260_petalinux/`, currently the active slot 0 app.
  Replaced the factory `k26-starter-kits` app via `xmutil unloadapp`/`loadapp`.
- **FPGA state**: `operating` (our bitstream live).
- **`/dev/uio*` after `loadapp` + `modprobe uio_pdrv_genirq of_id=generic-uio`**:
  | UIO | Peripheral | Address |
  |---|---|---|
  | uio0–uio3 | axi-pmon (PS perfmon, pre-existing) | ffa00000 etc. |
  | uio4 | axi_dma_0 (10G ethernet DMA) | a0020000 |
  | uio5 | axi_fifo_mdebug | a0050000 |
  | uio6 | axi_fifo_debug | a0070000 |
  | uio7 | axi_fifo_cmd | a00a0000 |
  | uio8 | axi_dma_echo | a00d0000 |
  | uio9 | axi_dma_fifo_echo | a00e0000 |
  | uio10 | axi_fifo_echo | a0100000 |
- **Permissions**: `/dev/uio*` is `crw-rw---- root:petalinux` thanks to
  `/etc/udev/rules.d/99-uio.rules`:
  ```
  SUBSYSTEM=="uio", KERNEL=="uio*", GROUP="petalinux", MODE="0660"
  ```
  The `petalinux` user is in the `petalinux` group; `sudo` is no longer
  required to mmap UIO devices.
- **`~/.ssh/authorized_keys`** contains the `id_kr260.pub` key, so passwordless
  ssh works from the dev machine via the `kr260-201` alias.

---

## The workflow now in place

In any new shell:

```bash
source ~/work/kr260_petalinux/sdk/environment-setup-cortexa72-cortexa53-xilinx-linux
cd ~/work/kr260_petalinux/src
make quick      # cross-compile + scp + ssh-and-run
```

Output:
```
Found /dev/uio7 (size=0x10000) for axi_fifo_cmd @ 0xA00A0000

axi_fifo_cmd registers:
  ISR  (0x00): 0x00900000
  IER  (0x04): 0x00000000
  TDFV (0x0C): 0x00000000  (TX vacancy, 32-bit words)
  RDFO (0x1C): 0x00000000  (RX occupancy, 32-bit words)
```

For step-through debugging:
```
make debug      # launches gdbserver on KR260
# in second shell (with SDK env sourced):
$GDB src/uio_test -ex "target remote kr260-201:2345"
```

---

## Build commands used (for reference)

### PetaLinux project
```bash
source /tools/Xilinx/PetaLinux/2024.1/settings.sh
cd ~/work/kr260_petalinux
petalinux-create --type project --source ../xilinx-kr260-starterkit-v2024.1-05230256.bsp --name petalinux
cd petalinux
petalinux-config --get-hw-description=../vivado --silentconfig
petalinux-build
petalinux-package --boot --u-boot --fpga --force
petalinux-package --wic                         # ⚠ 2024.1 dropped --disk-name and --bootfiles
petalinux-build --sdk                           # produces sdk.sh
```

### Kria App package
```bash
cd ~/work/kr260_petalinux/kria_app
make
# produces build/kr260_petalinux/{kr260_petalinux.bit.bin,kr260_petalinux.dtbo,shell.json}
```

### App load on KR260
```bash
sudo mv kr260_petalinux /lib/firmware/xilinx/
sudo xmutil listapps
sudo xmutil unloadapp
sudo xmutil loadapp kr260_petalinux
sudo modprobe -r uio_pdrv_genirq                # required for generic-uio binding!
sudo modprobe uio_pdrv_genirq of_id=generic-uio
ls /dev/uio*
```

### SDK install
```bash
~/work/kr260_petalinux/petalinux/images/linux/sdk.sh -y -d ~/work/kr260_petalinux/sdk
```

---

## Non-obvious things that broke (and how)

### 1. Built SD image vs running kernel — the md5 didn't match
After `dd`-ing the SD card and booting, `/sys/firmware/fdt` md5'd to a
*different* value than our packaged DTB. The running tree had only 4 PL
peripherals (a stock Kria 10GbE design with `xlnx,eth-dma` compatible) instead
of our 13. Conclusion: the K26 SOM's BootROM follows the boot-mode pins; if
SW1 isn't on SD, it falls back to QSPI/eMMC factory firmware. The `xlnx,eth-dma`
compatible was the smoking gun — that string doesn't exist in our `pl.dtsi`.

The pivot: instead of fighting boot mode, use the **Kria App / overlay** flow.
That's what `kria_app/` is for — same bitstream, same DT nodes, just packaged
as a runtime-loadable overlay against the factory rootfs.

### 2. Overlay applied but UIO bindings didn't fire
After `xmutil loadapp kr260_petalinux` succeeded, `cat /sys/class/fpga_manager/fpga0/state`
said `operating` and `dmesg` showed all 13 PL nodes registered as platform
devices. But `/dev/uio*` only showed the 4 PS-side `axi-pmon` entries — none
of our 7 generic-uio nodes bound.

Cause: `uio_pdrv_genirq` ships with an **empty** `of_id` match table by default.
Without `of_id=generic-uio` (either as kernel cmdline or modprobe parameter),
nodes with `compatible = "generic-uio"` don't match the driver. The factory
Kria firmware doesn't set this param.

Fix: `modprobe -r uio_pdrv_genirq && modprobe uio_pdrv_genirq of_id=generic-uio`.
After that, all 7 expected UIO nodes appeared.

This setting is **not persistent** on the factory firmware. Open work item
below.

### 3. xxv-ethernet probe fails with `missing axistream-connected`
`dmesg` shows `xilinx_axienet a0030000.ethernet: missing axistream-connected
property` after the overlay applies. The xxv-ethernet driver requires a
phandle to a connected DMA via `axistream-connected = <&phandle>`, and the
overlay doesn't declare that linkage. Doesn't affect UIO/DMA functionality
but means the 10G interface isn't usable as a network device until the
overlay is fixed. Open work item.

### 4. PetaLinux 2024.1 changed `petalinux-package wic` argument syntax
Older docs recommend `--bootfiles "ramdisk... boot.scr Image system.dtb..." --disk-name "sda"`.
In 2024.1 those arguments are gone — `--disk-name` is unrecognized and
`--bootfiles` now copies into `/boot/` rather than the boot partition.
The default form `petalinux-package --wic` (no args) is what works.

### 5. `system-user.dtsi` `&label` overrides DID compile into our DTB
We confirmed `dtc -I dtb -O dts images/linux/system.dtb | grep generic-uio`
returned 5 matches matching our 5 overrides. So the override mechanism works
correctly — the issue was purely that we never actually booted that DTB
(see #1).

---

## Open work items / next steps

1. **Make `uio_pdrv_genirq of_id=generic-uio` persist across reboots.**
   Easiest: drop a file on the KR260:
   ```
   /etc/modprobe.d/uio.conf:  options uio_pdrv_genirq of_id=generic-uio
   ```
   Then either reboot or `modprobe -r && modprobe` to apply. Without this,
   every reboot requires manual reload.

2. **Fix xxv-ethernet binding in the overlay.** Add
   `axistream-connected = <&axi_dma_0>` (and `axistream-control-connected`
   if needed) to the `xxv_ethernet_0` node in `kria_app/kr260_petalinux.dtso`.
   Or override its compatible to `generic-uio` if you don't need the kernel
   network stack and just want raw register access.

3. **Decide the long-term boot story.** Two options:
   - **Stay on factory firmware** + Kria App overlay. Pros: don't touch the
     SOM, easy to swap apps. Cons: rootfs is the factory image, not yours;
     library skew with the SDK sysroot is possible.
   - **Switch SW1 to SD-boot** and `dd petalinux-sdimage.wic` to the card.
     Pros: rootfs matches SDK sysroot exactly; UIO match table can be set
     in the kernel cmdline via PetaLinux config. Cons: leaves the factory
     QSPI image alone but takes over fully when the card is present.

4. **Optionally bake the udev rule and modprobe.d into a future PetaLinux
   image.** Recipe paths under `project-spec/meta-user/recipes-bsp/`:
   - `udev/files/99-uio.rules`
   - `modprobe/files/uio.conf` (with `options uio_pdrv_genirq of_id=generic-uio`)
   So a fresh SD flash already has both set.

5. **Document the four undocumented PL peripherals.**
   `axi_iic_0`, `axi_gpio_0/1/control/value` were not in the original
   `README.md` address map but exist in the design. Already added in this
   session — confirm the inferred GPIO ↔ `my_state` IP pairing
   (`axi_gpio_control` 2-bit out + 32-bit out → control input;
   `axi_gpio_value` 32-bit in + 32-bit in → 64-bit accumulator readback)
   against the Vivado block design.

---

## Verification one-liner (run on KR260)

```bash
sudo xmutil listapps && echo && \
for i in /sys/class/uio/uio*; do
    printf "%s\t%-12s @ %s\n" "$(basename $i)" "$(cat $i/name)" "$(cat $i/maps/map0/addr)"
done && echo && \
echo "FPGA state: $(cat /sys/class/fpga_manager/fpga0/state)"
```

This shows: which app is loaded, full UIO map, FPGA state — the three things
worth checking after every reboot or app reload.
