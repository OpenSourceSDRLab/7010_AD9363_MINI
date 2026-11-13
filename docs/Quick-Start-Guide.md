---
title: 7010_AD9363_MINI Quick Start Guide
product: 7010_AD9363_MINI
doc_id: OSSDRLAB-7010_AD9363_MINI-QSG
version: 1.0
date: 2026-11-12
repo: https://github.com/OpenSourceSDRLab/7010_AD9363_MINI
---

<!-- ossl:header -->
<p><a href="https://www.opensourcesdrlab.com"><img src="../images/OpenSourceSDRLab-banner.jpg" alt="OpenSourceSDRLab" width="100%"></a></p>

# 7010_AD9363_MINI Quick Start Guide

| Product | Document | Version | Date |
|---|---|---|---|
| 7010_AD9363_MINI | OSSDRLAB-7010_AD9363_MINI-QSG | 1.0 | 2026-11-12 |
<!-- /ossl:header -->

## 1. Overview

The 7010-AD9363-MINI is a 35 mm × 50 mm software-defined-radio module. It combines a Xilinx Zynq-7010 SoC with 512 MB DDR3 and 32 MB Quad-SPI flash, and the Analog Devices AD9363 RF transceiver with two transmit and two receive channels (70 MHz – 6 GHz).

This guide shows how to run the bare-metal test application in this repository: the AD9363 transmits a test tone and the captured I/Q samples of both receive channels are printed on the serial console.

## 2. What you need

| Item | Notes |
|---|---|
| 7010-AD9363-MINI module | |
| Debug adapter | USB adapter with JTAG and UART on the 9-pin MX1.25 cable (shown on the product page), or any Xilinx-compatible JTAG cable plus a 3.3 V USB-UART adapter |
| Power supply | 5 V – 17 V, through the MX1.25 connector or the 40-pin FPC connector |
| RF cable and attenuator | IPEX-to-SMA cables and a 20 dB (or larger) attenuator for the TX1 → RX1 loop-back |
| PC | Windows or Linux with Vivado / Vitis 2021.1 (Lab Edition is enough for programming) and a serial terminal |
| Prebuilt files | `7010_AD9363_MINI_prebuilt_Vivado2021.1.zip` from the [Releases](../../../releases) page |
| Vivado / Vitis 2021.1 | download from AMD/Xilinx: <https://www.xilinx.com/support/download.html> |

## 3. Board connectors

| Connector | Function |
|---|---|
| TX1, TX2 (IPEX) | transmit outputs |
| RX1, RX2 (IPEX) | receive inputs |
| 9-pin MX1.25 (JP1) | supply input VIN, JTAG (TMS, TCK, TDI, TDO), UART (TXD, RXD), 3.3 V, GND |
| 40-pin FPC (JP2) | 15 LVDS pairs to the Zynq programmable logic, supply input VIN, GND |
| Button | at power-up the Zynq boots from the Quad-SPI flash; press the button to switch to JTAG boot |
| LEDs | power and user LEDs on the Zynq MIO pins |

See `hardware/7010_AD9363_MINI_Schematic.pdf` for the full schematic.

## 4. Running the test application

> **Tip:** The test application sets the transmit attenuation to almost 0 dB. Always put an attenuator of at least 20 dB between TX1 and RX1, or connect antennas instead of a cable.

### 4.1 Prepare

1. Unzip `7010_AD9363_MINI_prebuilt_Vivado2021.1.zip`. It contains `system_top.bit`, `system_top.xsa`, `fsbl.elf`, `AD936X.elf` and `BOOT.bin`.

2. Connect TX1 to RX1 through the attenuator.

3. Connect the debug adapter to the 9-pin connector and to the PC, then power the module.

4. Open a serial terminal on the adapter's UART port: **115200 baud, 8 data bits, no parity, 1 stop bit**.

### 4.2 Option A: write the flash (the module then starts the test by itself at power-up)

1. Press the button on the module to switch to JTAG boot.

2. In Vivado, open **Hardware Manager**, connect to the target, right-click the `xc7z010` device and choose **Add Configuration Memory Device**. Select `mt25ql256-qspi-x4-single` (32 MB Quad-SPI).

3. Program the configuration memory with:
   - Configuration file: `BOOT.bin`
   - Zynq FSBL: `fsbl.elf`
   - Operations: Erase, Program, Verify

4. When programming has finished, power-cycle the module. It boots from the flash and runs the test application.

### 4.3 Option B: run over JTAG (nothing is written to the flash)

1. Press the button on the module to switch to JTAG boot.

2. In Vitis 2021.1, create a platform from `system_top.xsa` and an application project, or use the XSCT console:

```
connect
targets -set -filter {name =~ "APU*"}
rst -system
source ps7_init.tcl          ;# from the .xsa (unzip it)
ps7_init
ps7_post_config
fpga -file system_top.bit
targets -set -filter {name =~ "*A9*#0"}
dow AD936X.elf
con
```

### 4.4 What you should see

The console prints the AD9363 initialisation messages and the result of the digital-interface tuning, then 2048 lines of the form

```
ADC1_I=...,ADC1_Q=...,ADC2_I=...,ADC2_Q=...,
```

followed by `Done.`. With the loop-back connected, the RX1 samples show the transmitted tone.

## 5. Reference designs in this repository

| Folder | Contents |
|---|---|
| `vivado/AD936X_PL/` | Analog Devices HDL `hdl_2021_r1`. The board's project is `projects/fmcomms2/zc702/` (re-targeted to the XC7Z010). |
| `vivado/AD936X_PS/AD936X/` | Analog Devices no-OS AD9361 driver and example (release `2021_R1`), configured for this board; LO frequency and other settings are in `src/main.c` |
| `vivado/AD936X_PS/system_top.xsa` | hardware platform exported from the FPGA design |

### Rebuilding from source

1. **FPGA.** The project file was edited in Vivado after it was created by the Analog Devices scripts, so open the `.xpr` instead of running the full `make` build:
   1. Install Vivado 2021.1 and set up the Analog Devices build environment (`make` on the `PATH`; on Windows see <https://wiki.analog.com/resources/fpga/docs/build>). Package only the IP cores the design uses, without running the full project build:

          cd vivado/AD936X_PL/projects/fmcomms2/zc702
          make lib

   2. Open `vivado/AD936X_PL/projects/fmcomms2/zc702/fmcomms2_zc702.xpr` in Vivado 2021.1, generate the bitstream and export the hardware (**File → Export → Export Hardware**, include bitstream).

2. **Application.** In Vitis 2021.1:
   1. Create a platform project from `system_top.xsa` (operating system `standalone`, processor `ps7_cortexa9_0`, with boot components so that the FSBL is generated).
   2. Create an empty bare-metal C application named `AD936X` on that platform. Delete the generated `src` folder and import the folders `ad9361`, `api`, `AXI_Core`, `iio`, `include`, `src`, `util` and `xilinx` from `vivado/AD936X_PS/AD936X/`. `src/lscript.ld` is the linker script (512 MB DDR3).
   3. Add the same eight folders as include directories (C/C++ Build Settings > ARM v7 gcc compiler > Directories). The linker libraries are the Vitis defaults.
   4. Build the **Release** configuration.

3. **Flash image.** Create `BOOT.bin` with **Create Boot Image** in Vitis (or `bootgen`) from `fsbl.elf`, `system_top.bit` and `AD936X.elf`, in this order.

## 6. Support

- Email: support@opensourcesdrlab.com
- Forum: <https://forum.opensourcesdrlab.com>
- Discord: <https://discord.gg/nnxY6DkmvT>
- Blog and user manuals: <https://blog.opensourcesdrlab.com>

<!-- ossl:footer -->
---
<sub>© 2026 OpenSourceSDRLab. All rights reserved. · <a href="https://www.opensourcesdrlab.com">www.opensourcesdrlab.com</a></sub>
<!-- /ossl:footer -->
