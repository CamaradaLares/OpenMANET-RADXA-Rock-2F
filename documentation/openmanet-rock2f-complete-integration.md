# OpenMANET on Radxa Rock 2F — Complete Integration Documentation

> Kernel: Linux 6.6.102 | OpenMANET firmware: 24.10 (r28739-d9340319c6) | HaLow driver: morse_driver 1.16.4-gateworks

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Hardware](#2-hardware)
3. [Repository Structure](#3-repository-structure)
4. [Board Definition — boards/rock2f-spi/](#4-board-definition)
5. [Kernel BSP Patches](#5-kernel-bsp-patches)
6. [Morse HaLow Driver — SPI Changes](#6-morse-halow-driver-spi-changes)
7. [Runtime Overlay Files](#7-runtime-overlay-files)
8. [Network Architecture](#8-network-architecture)
9. [Build Instructions](#9-build-instructions)
10. [Boot Sequence](#10-boot-sequence)
11. [Problems Encountered and Solutions](#11-problems-encountered-and-solutions)
12. [Known Issues](#12-known-issues)
13. [Complete File Change Index](#13-complete-file-change-index)
14. [Future Work](#14-future-work)

---

## 1. Project Overview

This document describes the complete port of [OpenMANET](https://openmanet.github.io/docs/) to the
Radxa Rock 2F single-board computer (Rockchip RK3528A SoC). The port is implemented as a board
definition inside the OpenMANET firmware build tree (`openmanet-firmware/`), so that running
`./scripts/openmanet_setup.sh -b rock2f-spi` and `make -j$(nproc)` produces a ready-to-flash
firmware image with no manual post-flash configuration required.

### What was ported

| Component | Status |
|-----------|--------|
| RK3528A SoC clock/reset/pinctrl (backport from kernel 6.12–6.18) | Complete |
| RK3528A USB2 PHY driver (backport) | Complete |
| Radxa Rock 2F board device tree (USB, WiFi power, SPI0, MM6108) | Complete |
| Morse Micro MM6108 HaLow (SPI) — driver compile fix for kernel 6.6 | **Complete** |
| AIC8800D80 onboard WiFi AP driver | Complete |
| batman-adv mesh over HaLow backhaul | Complete |
| Ordered GPIO reset + driver loading init script | Complete |
| First-boot UCI configuration (network, wireless, firewall, DHCP) | Complete |
| LuCI web interface | Working (from openmanet feed) |

### Source trees used

```
rock2f-port-openmanet/
├── openmanet-firmware/          ← OpenMANET firmware build tree (main working tree)
│   ├── boards/rock2f-spi/       ← Rock 2F board definition (our addition)
│   ├── feeds/openmanet/         ← OpenMANET packages feed (symlink from feeds/)
│   ├── feeds/morse/             ← Morse Micro HaLow feed (symlink from feeds/)
│   └── target/linux/rockchip/  ← Rockchip kernel target with our BSP patches
└── documentation/               ← This folder
```

---

## 2. Hardware

### Radxa Rock 2F

| Feature | Detail |
|---------|--------|
| SoC | Rockchip RK3528A, quad-core ARM Cortex-A53, up to 1.8 GHz |
| RAM | LPDDR4 |
| Storage | eMMC + microSD (boot from microSD) |
| Onboard WiFi | AIC8800D80 — USB 2.0, connected internally via FE1.1s USB hub on `USB20_HOST1` |
| USB ports | 2× USB 2.0 Host, 1× USB 3.0 OTG (configured as host) |
| GPIO header | 40-pin, Raspberry Pi-compatible pin layout |
| Serial console | UART0 at physical address `0xff9f0000`, 1500000 baud 8N1 |

### WiFi chip connection path

```
RK3528A USB20_HOST1 → FE1.1s USB Hub (U2002) → Port 4 → AIC8800D80 (U2202)
```

WiFi chip power GPIOs (toggled via memory-mapped register writes in U-Boot):
- `VCC_WIFI` (3.3V rail): GPIO4_A4 — GPIO4 base `0xffb20000`, bit 4
- `WIFI_REG_ON_H` (chip enable): GPIO1_A6 — GPIO1 base `0xffaf0000`, bit 6

### Seeed WM1302 HAT + Wio-WM6108 Module

The HAT plugs onto the Rock 2F 40-pin GPIO header. It carries a Seeed Wio-WM6108 mPCIe module,
which contains the Quectel FGH100M-H radio with a Morse Micro MM6108 chipset. The SPI bus and
control GPIOs of the module are exposed via the HAT to the host SBC header.

#### GPIO mapping — MM6108 SPI and control pins

| Header Pin | RK3528 GPIO | GPIO# (sysfs) | Function | Direction |
|-----------|-------------|---------------|----------|-----------|
| 19 | GPIO4_B2 | 138 | SPI0_MOSI | Output |
| 21 | GPIO4_B3 | 139 | SPI0_MISO | Input |
| 23 | GPIO4_B4 | 140 | SPI0_CLK | Output |
| 24 | GPIO4_B6 | 142 | SPI0_CSN0 | Output |
| 11 | GPIO4_B7 | **143** | Reset (active-LOW) | Output |
| 16 | GPIO4_B0 | 136 | Wake | Output |
| 18 | GPIO4_B1 | 137 | Busy | Input |
| 29 | GPIO4_B5 | 141 | IRQ | Input |

**Reset pin polarity**: GPIO4_B7 (pin 11) is wired to MM6108 hardware reset. The chip is held in
reset when the pin is LOW and runs normally when the pin is HIGH. The DTS uses
`GPIO_ACTIVE_LOW` on `reset-gpios`, which correctly maps driver "assert reset" → pin LOW.

#### HAT pins that conflict with Rock 2F boot

The WM1302 HAT was designed for Raspberry Pi and several of its non-MM6108 circuits conflict with
Rock 2F peripherals:

| Pin(s) | HAT device | Rock 2F conflict |
|--------|-----------|-----------------|
| 1, 17 | 3.3V power | Combined 3.3V draw (MM6108 + GPS L76KB + ATECC608B + HAT regulator) causes brownout |
| 7 | GPS 1PPS | GPIO4_A6 / UART1_TX_M0 — pulses interfere with UART1 init |
| 8, 10 | GPS UART TX/RX | UART2_TX/RX_M0 — NMEA data corrupts UART2 |
| 3, 5 | ATECC608B I2C | I2C0_SDA/SCL — crypto chip pulls bus |
| 27, 28 | HAT ID EEPROM | I2C1_SDA/SCL — conflicts with onboard EEPROM at 0x50 |

The brownout from 3.3V current draw was confirmed as the primary cause of the HAT-seated bootloop.
Mitigations: external 3.3V for the HAT, or physical tape over conflicting pins (1, 3, 5, 7, 8, 10,
27, 28) to expose only the SPI and GPIO control pins.

---

## 3. Repository Structure

### openmanet-firmware directory layout (relevant to this port)

```
openmanet-firmware/
│
├── boards/
│   ├── common/                      # Shared base configs for all boards
│   ├── common_extras/               # Optional diffconfigs (-x flag)
│   └── rock2f-spi/                  ← Our board definition
│       ├── target_diffconfig        # Target + HaLow + AIC8800 package selection
│       ├── dev_diffconfig           # Developer tools (debug kernel, perf, etc.)
│       ├── utils_diffconfig         # Utilities (iperf, tcpdump, LuCI tools)
│       ├── rangetest_diffconfig     # Range test app config
│       ├── wireshark_diffconfig     # Wireshark capture config
│       └── files/                   # Runtime overlay (baked into squashfs)
│           └── etc/
│               ├── init.d/
│               │   ├── morsechipreset     # GPIO reset + ordered driver loading
│               │   └── fix-wifi-path     # AIC8800 USB bus path fix
│               ├── hotplug.d/net/
│               │   └── 99-batadv         # wlan0 → bat0, bat0 → br-ahwlan
│               ├── sysctl.d/
│               │   └── 99-disable-ipv6.conf  # AIC8800 MLD crash mitigation
│               └── uci-defaults/
│                   └── 99-rock2f-openmanet   # First-boot network/wireless config
│
├── feeds/
│   ├── morse/                       # Morse Micro HaLow stack
│   └── openmanet/
│       └── drivers/
│           └── morse_driver/
│               └── Makefile         ← Modified: SPI compile fix
│
├── target/linux/rockchip/
│   ├── patches-6.6/                 # All our kernel patches
│   ├── files-6.6/                   # DT-binding header files
│   ├── config-6.6                   # Kernel config additions
│   └── image/
│       ├── armv8.mk                 # Rock 2A/2F image definitions
│       └── rock-2.bootscript        # Custom boot script
│
├── package/
│   ├── kernel/aic8800-usb/          # AIC8800D80 driver package
│   └── firmware/aic8800-firmware/  # AIC8800 firmware binaries
│
└── feeds.conf.default               # Feed URLs (morse + openmanet feeds added)
```

### Build command

```bash
cd openmanet-firmware
./scripts/openmanet_setup.sh -b rock2f-spi -i
make -j$(nproc) V=s 2>&1 | tee /tmp/build.log
```

The setup script:
1. Concatenates `boards/common/*_diffconfig` + `boards/rock2f-spi/*_diffconfig` into `.config`
2. Runs `make defconfig` to expand
3. Copies `boards/rock2f-spi/files/` into `files/` (the squashfs overlay root)

### Output image

```
bin/targets/rockchip/armv8/openmanet-24.10-1.6.5-rockchip-armv8-radxa_rock-2f-squashfs-sysupgrade.img.gz
```

Compressed size: ~52 MB. Uncompressed: ~320 MB (256 MB rootfs partition + 16 MB kernel partition + overhead).

### Flash command

```bash
gunzip -c openmanet-24.10-*-radxa_rock-2f-squashfs-sysupgrade.img.gz | \
  dd of=/dev/sdX bs=4M status=progress
```

---

## 4. Board Definition

### boards/rock2f-spi/target_diffconfig

The primary board configuration. Controls target selection and package inclusion.

```makefile
CONFIG_TARGET_MULTI_PROFILE=y
CONFIG_TARGET_rockchip=y
CONFIG_TARGET_rockchip_armv8=y
CONFIG_TARGET_DEVICE_rockchip_armv8_DEVICE_radxa_rock-2f=y

CONFIG_VERSION_MANUFACTURER="Radxa"
CONFIG_VERSION_PRODUCT="ROCK 2F"

# Morse HaLow stack — SPI-connected MM6108 (Seeed WM1302 HAT)
CONFIG_MORSE_SPI=y                          # ← CRITICAL: enables SPI transport in morse driver
CONFIG_PACKAGE_kmod-morse=y
CONFIG_PACKAGE_morse-fw-6108=y
CONFIG_PACKAGE_morse-fw-6108-tlm=y
CONFIG_PACKAGE_morse-board-config=y
CONFIG_PACKAGE_morse-regdb=y
CONFIG_PACKAGE_morsectrl=y
CONFIG_PACKAGE_netifd-morse=y
CONFIG_PACKAGE_hostapd_s1g=y
CONFIG_PACKAGE_wpa_supplicant_s1g=y
CONFIG_PACKAGE_dot11ah=y

# AIC8800D80 — internal USB WiFi AP (out-of-tree driver)
CONFIG_PACKAGE_kmod-aic8800-usb=y
CONFIG_PACKAGE_aic8800-firmware=y

# Rootfs partition size (256 MB — squashfs is ~50 MB, leaves ~200 MB for overlay)
CONFIG_TARGET_ROOTFS_PARTSIZE=256

# Disable ext4 image (squashfs only)
CONFIG_TARGET_ROOTFS_EXT4FS=n
```

**Key point**: `CONFIG_MORSE_SPI=y` must be in `target_diffconfig` explicitly. The
`common_extras/spi_diffconfig` file that the OpenMANET build system provides also sets this flag,
but it also pulls in BCM/Raspberry Pi-specific SPI kernel modules that must not be selected on
Rockchip. Adding `CONFIG_MORSE_SPI=y` directly to `target_diffconfig` is the correct approach for
the Rock 2F.

---

## 5. Kernel BSP Patches

All patches reside in `target/linux/rockchip/patches-6.6/`. These patches backport RK3528A support
from upstream Linux 6.12–6.18 into kernel 6.6.102 (the version pinned by the OpenMANET firmware
tree at the time of this port).

### 5.1 Clock, Reset, and Pinctrl (032-xx series)

Without the RK3528 clock driver the kernel hangs silently at "Starting kernel..." because it cannot
initialize any peripheral (not even the UART serial console). These are the most critical patches.

| Patch | Description | Notes |
|-------|-------------|-------|
| `032-19-v6.15-clk-rockchip-Add-PLL-flag-ROCKCHIP_PLL_FIXED_MODE.patch` | PLL FIXED_MODE flag prerequisite | Applied cleanly |
| `032-20-v6.15-clk-rockchip-Add-clock-controller-driver-for-RK3528-SoC.patch` | Complete `clk-rk3528.c` driver | Applied cleanly (mostly new file) |
| `032-21-v6.15-clk-rockchip-rk3528-Add-reset-lookup-table.patch` | `rst-rk3528.c` reset controller lookup table | Required manual fix: `clk.h` hunk context references `rk3576_rst_init`, which does not exist in kernel 6.6. Fixed by changing context to `rk3588_rst_init`. |
| `032-22-v6.15-pinctrl-rockchip-Add-support-for-RK3528.patch` | RK3528 pinctrl support in `pinctrl-rockchip.c` | Applied cleanly |

Excluded:
- `032-26` — SD/SDIO tuning clocks via GRF; failed to apply because it requires `rockchip_grf_type`
  enum members not present in 6.6. Not needed for basic operation.
- `032-27` — slab.h header fix that depends on 032-26.

### 5.2 DT-Binding Header Files

Location: `target/linux/rockchip/files-6.6/include/dt-bindings/`

| File | Purpose |
|------|---------|
| `clock/rockchip,rk3528-cru.h` | Clock ID constants (e.g. `CLK_SPI0`) |
| `reset/rockchip,rk3528-cru.h` | Reset ID constants (e.g. `SRST_SPI0`) |
| `power/rockchip,rk3528-power.h` | Power domain ID constants |

These headers are required by `rk3528.dtsi` but do not exist in upstream Linux 6.6.

### 5.3 Kernel Config Additions

Added to `target/linux/rockchip/config-6.6`:

```
CONFIG_CLK_RK3528=y
CONFIG_PINCTRL_RK3528=y
```

These must be present; without them a non-interactive `make defconfig` run will silently skip them
and the SoC will fail to boot.

### 5.4 RK3528 Base Device Tree (070-xx series)

Patches 070-01 through 070-22 add the base SoC device tree (`rk3528.dtsi`). These were already
present from the OpenMANET firmware tree. Highlights:

| Patch | Content |
|-------|---------|
| 070-01 | CPU, interrupt controller, timer, CRU |
| 070-04 | Pinctrl and GPIO bank nodes |
| 070-14 | SDMMC/eMMC/SDIO controllers |
| 070-17 | SPI controller nodes (spi0–spi2) |
| 070-18 | Power controller (not fully functional — see Section 5.7) |
| 070-22 | Convert power domains to SCMI |

### 5.5 Rock 2A/2F Board DTS Patches

| Patch | Content |
|-------|---------|
| `071-arm64-dts-rockchip-Add-Radxa-ROCK-2A-2F-Makefile.patch` | Adds `rk3528-rock-2a.dtb` and `rk3528-rock-2f.dtb` to the DTS build Makefile |
| `072-v6.18-arm64-dts-rockchip-Add-Radxa-ROCK-2A-2F.patch` | Board DTS files: `rk3528-rock-2.dtsi` (common), `rk3528-rock-2a.dts`, `rk3528-rock-2f.dts` |

### 5.6 USB PHY Backport (160-xx and 163-02 series)

The RK3528 USB2 PHY driver does not exist in Linux 6.6. Without it, USB host ports cannot function
and the AIC8800D80 (connected via the internal USB hub) is never detected.

| Patch | Description |
|-------|-------------|
| `160-01-phy-rockchip-inno-usb2-Simplify-rockchip-usbgrf-handling.patch` | GRF handling simplification |
| `160-02-phy-rockchip-inno-usb2-Add-clkout_ctl_phy-support.patch` | Clock output control |
| `160-03-phy-rockchip-inno-usb2-Add-support-for-RK3528.patch` | RK3528 USB2 PHY support (core patch) |
| `160-04-local-phy-rockchip-inno-usb2-clkout-helper.patch` | Clock output helper function |
| `160-04-local-phy-rockchip-inno-usb2-rk3528-compat.patch` | RK3528 compatibility fixes |
| `163-02-arm64-dts-rockchip-Add-USB-nodes-for-RK3528.patch` | USB controller DTS nodes: `usb_host0_xhci` (DWC3), `usb_host0_ehci`, `usb_host0_ohci`, `usb2phy` — all with `status = "disabled"` (enabled in board patch 073) |

**Important**: All five 160-xx patches must be applied together. The compat patch (`160-04-local-...
rk3528-compat`) defines `rk3528_usb2phy_tuning()` which is referenced by the core USB2 PHY driver
file. If the earlier patches are absent, this function is defined but never called, causing a
`-Werror=unused-function` build failure.

### 5.7 USB Power Domain Removal (164)

| Patch | `164-arm64-dts-rockchip-Remove-USB-power-domains-RK3528.patch` |
|-------|-----------------------------------------------------------------|
| Content | Removes `power-domains` property from USB nodes (xhci, ehci, ohci, usb2phy) |
| Reason | The RK3528 power domain controller (from patch 070-18) is not fully functional in this backport. When `power-domains` was present in the USB nodes, all devices failed with `deferred probe timeout` — the power domain controller would never complete initialization. |
| Safety | Removing `power-domains` is safe because Rockchip SoCs default all power domains to ON at reset. USB works correctly without the power domain reference. |

### 5.8 Critical Patch — 073 (USB Enable + MM6108 SPI)

File: `target/linux/rockchip/patches-6.6/073-arm64-dts-rockchip-Enable-USB-for-Rock-2.patch`

This patch modifies `rk3528-rock-2.dtsi` and is the single most important board-specific
customization for the Rock 2F. It contains:

#### WiFi power regulator (always-on)

```dts
&vcc_wifi {
    regulator-always-on;
    regulator-boot-on;
};
```

Without this, the `vcc_wifi` regulator powers down during kernel init before the AIC8800 driver
loads, causing the WiFi chip to become inaccessible. Setting it always-on ensures power throughout
boot.

#### USB controller enable

```dts
&usb2phy       { status = "okay"; }
&usb2phy_host  { phy-supply = <&vcc5v0_usb20>; status = "okay"; }
&usb2phy_otg   { status = "okay"; }
&usb_host0_ehci { status = "okay"; }
&usb_host0_ohci { status = "okay"; }
&usb_host0_xhci { dr_mode = "host"; status = "okay"; }
```

#### SPI0 with MM6108 device node

```dts
&spi0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins &spi0_csn0>;
    #address-cells = <1>;
    #size-cells = <0>;

    mm6108: mm6108@0 {
        compatible = "morse,mm610x-spi";
        reg = <0>;
        spi-max-frequency = <10000000>;
        reset-gpios = <&gpio4 RK_PB7 GPIO_ACTIVE_LOW>;
        power-gpios = <&gpio4 RK_PB0 GPIO_ACTIVE_HIGH>,
                      <&gpio4 RK_PB1 GPIO_ACTIVE_HIGH>;
        spi-irq-gpios = <&gpio4 RK_PB5 GPIO_ACTIVE_HIGH>;
        status = "okay";
    };
};
```

Key design decisions:
- `spi-max-frequency = <10000000>` — 10 MHz. The default 50 MHz caused SPI protocol errors (errno
  -71, EAGAIN) on both jumper wires and through the HAT. 10 MHz is stable.
- `reset-gpios = GPIO_ACTIVE_LOW` — Chip resets when pin is LOW, runs when HIGH. Initially set to
  `GPIO_ACTIVE_HIGH` which kept the MM6108 in permanent reset (SPI probe returned errno -5, MISO
  reads all `0xFFFFFFFF`).

#### MM6108 pinctrl

```dts
&pinctrl {
    morse-mm6108 {
        morse_reset_pin: morse-reset-pin {
            rockchip,pins = <4 RK_PB7 RK_FUNC_GPIO &pcfg_pull_none>;
        };
        morse_wake_pin: morse-wake-pin {
            rockchip,pins = <4 RK_PB0 RK_FUNC_GPIO &pcfg_pull_none>;
        };
        morse_busy_pin: morse-busy-pin {
            rockchip,pins = <4 RK_PB1 RK_FUNC_GPIO &pcfg_pull_down>;
        };
        morse_irq_pin: morse-irq-pin {
            rockchip,pins = <4 RK_PB5 RK_FUNC_GPIO &pcfg_pull_up>;
        };
    };
};
```

### 5.9 Boot Script

File: `target/linux/rockchip/image/rock-2.bootscript`

```bash
# Power on WiFi: VCC_WIFI (GPIO4_A4) and WIFI_REG_ON (GPIO1_A6)
# using memory-mapped register writes before kernel GPIO drivers are available.
mw.l 0xffb20000 0x00100010
mw.l 0xffaf0000 0x00400040
sleep 1

part uuid ${devtype} ${devnum}:2 uuid
setenv bootargs "console=ttyS0,1500000 earlycon=uart8250,mmio32,0xff9f0000 root=PARTUUID=${uuid} rw rootwait"
load ${devtype} ${devnum}:1 ${kernel_addr_r} kernel.img
bootm ${kernel_addr_r}
```

Key point: `console=ttyS0` (UART0 at `0xff9f0000`). Other Rockchip boards typically use
`ttyS2` — using `ttyS2` on Rock 2F results in a silent boot after earlycon hands off.

Referenced in `target/linux/rockchip/image/armv8.mk` via `BOOT_SCRIPT := rock-2` for both Rock 2A
and Rock 2F image definitions.

---

## 6. Morse HaLow Driver — SPI Changes

### 6.1 Package origin

The OpenMANET firmware tree uses the **Gateworks fork** of the Morse Micro driver:

- Location in tree: `package/feeds/openmanet/morse_driver/`
- Source: `https://github.com/Gateworks/morse_driver.git`
- Version: `1.16.4-gateworks` (commit `dec5bc215b881c369ff530c440c212ec6dd13183`)

This is different from the upstream `feeds/morse/essentials/morse_driver/` package. The Gateworks
fork is what the OpenMANET feed installs.

### 6.2 Problem: CONFIG_MORSE_SPI not enabled

**Symptom**: The morse kernel module compiles without SPI support. No `morse_spi_*` symbols in
`morse.ko`. The Seeed WM1302 HAT is invisible — no SPI device probed.

**Root cause**: `CONFIG_MORSE_SPI=y` was absent from the build configuration. The openmanet
`common_extras/spi_diffconfig` provides this flag but also enables BCM-specific SPI kernel modules
(`CONFIG_SPI_BCM2835`, `CONFIG_SPI_BCM2835AUX`) that must not be enabled on Rockchip. Simply
appending the spi_diffconfig via `-x spi` would pull in those BCM modules.

**Fix**: Added `CONFIG_MORSE_SPI=y` directly to `boards/rock2f-spi/target_diffconfig`:

```
# Morse HaLow stack — SPI-connected MM6108 (Seeed WM1302 HAT)
CONFIG_MORSE_SPI=y
CONFIG_PACKAGE_kmod-morse=y
```

### 6.3 Problem: SPI_CONTROLLER_ENABLE_CS_GPIOD not defined in kernel 6.6

**Symptom**: Morse driver compile fails with:

```
morse_driver-1.16.4-gateworks/spi.c:1422:2: error: #warning "SPI_CONTROLLER_ENABLE_CS_GPIOD macro not defined" [-Werror=cpp]
```

**Root cause**: The macro `SPI_CONTROLLER_ENABLE_CS_GPIOD` (`BIT(9)` in the SPI controller flags)
was added to the Linux SPI core in kernel 6.9. The morse driver's `spi.c` contains:

```c
#if KERNEL_VERSION(6, 1, 21) <= LINUX_VERSION_CODE
  #ifdef SPI_CONTROLLER_ENABLE_CS_GPIOD
    ... use the macro ...
  #else
    #warning "SPI_CONTROLLER_ENABLE_CS_GPIOD macro not defined"
  #endif
#endif
```

With `-Werror=cpp` active (the standard for out-of-tree kernel module builds), this warning
promotes to an error and the compilation fails.

The seedio (Raspberry Pi 4) reference build avoids this via a bcm27xx-specific kernel patch
(`991-0007-spi-support-control-cs-pin-on-init.patch`) that adds the macro to the kernel SPI header.
That patch cannot be applied to the Rockchip target.

**Fix**: Added a compile-time define to `NOSTDINC_FLAGS` in the morse driver Makefile
(`package/feeds/openmanet/morse_driver/Makefile`):

```makefile
# SPI_CONTROLLER_ENABLE_CS_GPIOD was added in kernel 6.9. Define it for
# older kernels so the morse SPI driver compiles without -Werror=cpp failure.
# BIT(9) is unused in 6.6 SPI core so setting it is harmless.
NOSTDINC_FLAGS += -DSPI_CONTROLLER_ENABLE_CS_GPIOD='(1<<9)'
```

This defines the macro as `(1<<9)` (which equals `BIT(9)`) when compiling the morse driver. The
macro is never passed to the kernel SPI layer — it is only used inside the morse driver's
conditional code block to select a CS GPIO control code path that is safe on kernel 6.6 as well.

**Verification**: After the fix, `strings morse.ko | grep -i spi` shows:

```
morse_spi_find_response
morse_spi_bus_enable
morse_spi_cmd53_write
...
morse,mm610x-spi
morse,mm810x-spi
morse_spi_probe
```

### 6.4 Board Configuration File (BCF)

**Problem**: The morse-feed ships `bcf_default.bin` as a symlink to `bcf_failsafe.bin` — a minimal
configuration with the **power amplifier disabled**. With the failsafe BCF the driver loads and
reports 920.5 MHz operation, but no actual RF signal is emitted (visible on SDR as 0 dBm /
undetectable).

**Correct BCF**: The Seeed Wio-WM6108 module uses the Quectel FGH100M-H front-end. The correct BCF
is `bcf_fgh100mhaamd.bin` (size: 1251 bytes, CRC32: `0x941b2a82`). With this BCF:
- TX power: **27 dBm** (correctly matching OpenMANET's documented power level)
- PA enable: active
- Beacons visible on SDR at 921 MHz

**Fix**: Applied at first boot via `etc/uci-defaults/99-rock2f-openmanet`:

```bash
ln -sf /lib/firmware/morse/bcf_fgh100mhaamd.bin /lib/firmware/morse/bcf_default.bin
```

The `bcf=` module parameter in `/etc/modules.d/morse` is ignored by the driver when it falls
back to loading `bcf_default.bin`; the symlink approach is the only reliable method.

Also passed explicitly at modprobe time in `morsechipreset`:

```bash
modprobe morse country=US bcf=bcf_fgh100mhaamd.bin spi_clock_speed=10000000 ...
```

### 6.5 S1G Frequency Mapping

The dot11ah kernel shim maps 802.11ah sub-1 GHz channels to equivalent 5 GHz channels for
mac80211 compatibility. This means tools like `iwinfo` and `iw dev` show the interface on 5 GHz:

| UCI channel | Actual RF frequency | Bandwidth | mac80211 view |
|------------|---------------------|-----------|--------------|
| 37 | 920.5 MHz | 1 MHz | ch149 / 5745 MHz |
| 38 | **921.0 MHz** | 2 MHz | ch149 / 5745 MHz |

The real frequency is confirmed with:
```bash
morse_cli -i wlan0 channel
```

---

## 7. Runtime Overlay Files

All files in `boards/rock2f-spi/files/` are copied into the build root's `files/` directory by the
setup script and are baked into the squashfs rootfs. They overwrite any default files provided by
the OpenMANET feed packages.

### 7.1 /etc/init.d/morsechipreset (START=09)

This replaces the morse-feed's default `morsechipreset` init script, which calls
`/morse/scripts/chipreset.sh`. The upstream script:
1. Uses `gpiofind MM_RESET` — requires `gpio-line-names` in the device tree (not implemented for
   RK3528).
2. Unbinds/rebinds the mmc_host subsystem for SDIO reset — catastrophic on Rock 2F because it
   would unbind eMMC.

The custom script directly uses sysfs GPIO:

```bash
#!/bin/sh /etc/rc.common
START=09

boot() {
    # 1. Hardware reset MM6108 via GPIO 143 (GPIO4_B7, pin 11)
    echo 143 > /sys/class/gpio/export 2>/dev/null
    echo out > /sys/class/gpio/gpio143/direction 2>/dev/null
    echo 0 > /sys/class/gpio/gpio143/value    # Assert reset (LOW)
    sleep 1
    echo 1 > /sys/class/gpio/gpio143/value    # Release reset (HIGH)
    sleep 1

    # 2. Load morse FIRST (registers wlan0 for 802.11s mesh)
    modprobe dot11ah
    modprobe morse country=US bcf=bcf_fgh100mhaamd.bin spi_clock_speed=10000000 \
        enable_ps=0 enable_dynamic_ps_offload=0
    sleep 2

    # 3. Load AIC8800 SECOND (gets its own interface)
    modprobe aic_load_fw
    modprobe aic8800_fdrv ps_on=N dpsm=N ap_uapsd_on=N
    sleep 3

    # 4. Disable USB autosuspend
    echo -1 > /sys/module/usbcore/parameters/autosuspend 2>/dev/null
}

start() { boot; }
```

**Why driver load order is critical**: If `aic8800_fdrv` loads before `morse`, the AIC8800 claims
`wlan0`. When netifd then applies the HaLow mesh configuration to `wlan0`, it issues a
`MESH_START_REQ` command to the AIC8800 driver which does not support mesh mode. This triggers a
`scheduling while atomic` kernel BUG in `aicwf_rxbuff_enqueue`, crashing the system.

**Why START=09 and not modules.d**: OpenWrt's `kmodloader` processes `/etc/modules.d/*` before any
`/etc/rc.d/` init scripts run. Without the GPIO reset (LOW→HIGH sequence) the morse SPI driver
probe fails because the chip is still in reset from power-on. The auto-load entries for `morse` and
`aic8800-usb` are renamed to `.disabled` by the first-boot script so kmodloader skips them.

### 7.2 /etc/init.d/fix-wifi-path (START=18)

The AIC8800D80 performs a USB disconnect + re-enumerate during firmware upload. The chip can
re-appear on a different USB bus number (`usb3` vs `usb4`) between boots. OpenWrt stores the full
USB path in the wireless config; a stale path means the AIC8800 radio is not recognized and the
AP does not start.

```bash
#!/bin/sh /etc/rc.common
START=18

boot() {
    sleep 3
    for phy in /sys/class/ieee80211/phy*; do
        devpath=$(readlink -f "$phy/device")
        case "$devpath" in
            *ff100000.usb*)        # RK3528 USB20_HOST1 MMIO address
                relpath=${devpath#*/platform/}
                relpath="platform/${relpath}"
                current=$(uci -q get wireless.radio1.path)
                if [ "$current" != "$relpath" ]; then
                    uci set wireless.radio1.path="$relpath"
                    uci commit wireless
                fi
                ;;
        esac
    done
}

start() { boot; }
```

The script matches by the SoC's USB host controller MMIO address (`ff100000`) which is stable,
then reads the dynamic USB path from sysfs.

### 7.3 /etc/hotplug.d/net/99-batadv

```bash
# Attach wlan0 (HaLow) to batman-adv bat0 when it comes up
[ "$ACTION" = "add" ] && [ "$INTERFACE" = "wlan0" ] && {
    sleep 2
    ip link set wlan0 mtu 1532 2>/dev/null
    batctl meshif bat0 if add wlan0 2>/dev/null
    logger -t batadv "Added wlan0 to bat0 with MTU 1532"
}

# Bridge bat0 into br-ahwlan when batman-adv interface comes up
[ "$ACTION" = "add" ] && [ "$INTERFACE" = "bat0" ] && {
    sleep 1
    brctl addif br-ahwlan bat0 2>/dev/null
    logger -t batadv "Added bat0 to br-ahwlan"
}
```

The MTU of 1532 = standard 1500 bytes + 32 bytes batman-adv encapsulation overhead.

### 7.4 /etc/sysctl.d/99-disable-ipv6.conf

```
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
```

The AIC8800 driver's batman-adv mesh proxy code (`rwnx_send_mesh_proxy_add_req`) is invoked from
an IPv6 MLD (Multicast Listener Discovery) workqueue in atomic context and attempts to sleep,
causing a kernel BUG: `scheduling while atomic`. Disabling IPv6 at the sysctl level prevents MLD
processing and eliminates the crash.

### 7.5 /etc/uci-defaults/99-rock2f-openmanet

Runs once on first boot. Performs auto-detection of radios and applies the full OpenMANET network
configuration for the Rock 2F:

**Radio detection** (by type and USB path, not by radio number, which is dynamically assigned):
- Identifies the `morse` type radio → HaLow mesh on channel 38, SAE encryption, network `mesh0`
- Identifies the mac80211 USB radio (AIC8800) → AP on `ahwlan` bridge
- Disables dot11ah mac80211 shim radios (SPI path) — morse driver handles those internally

**Network architecture (OpenMANET single-bridge model)**:
```
br-ahwlan bridge device:
  - eth0 (wired port)
  - bat0 (added via hotplug — see 99-batadv)
  - phy1-ap0 (AIC8800 WiFi AP — network='ahwlan')

bat0:
  - proto: batadv
  - routing_algo: BATMAN_V
  - gw_mode: off

mesh0 (wlan0 → bat0):
  - proto: batadv_hardif
  - master: bat0
  - mtu: 1532
```

**DHCP**: Per-node 16 leases from `10.41.0.0/16` (`start=2, limit=16, leasetime=12h`)

**Module auto-load prevention**:
```bash
[ -f /etc/modules.d/morse ] && mv /etc/modules.d/morse /etc/modules.d/morse.disabled
for f in /etc/modules.d/*aic*; do
    [ -f "$f" ] && mv "$f" "${f}.disabled"
done
```

**BCF symlink fix**:
```bash
ln -sf /lib/firmware/morse/bcf_fgh100mhaamd.bin /lib/firmware/morse/bcf_default.bin
```

---

## 8. Network Architecture

### Diagram

```
End User Device (phone, tablet, laptop)
     │  WiFi 5 GHz (WPA2-PSK "OpenMANET-AP")
     ▼
┌────────────────────────────────────────────────┐
│              Radxa Rock 2F                     │
│                                                │
│  AIC8800D80                                    │
│  phy1-ap0 (WiFi AP)                           │
│       │                                        │
│  br-ahwlan  10.41.254.1/16                    │
│       │   (also has eth0, bat0 via hotplug)    │
│       │                                        │
│  bat0 (batman-adv, BATMAN_V)                   │
│       │                                        │
│  wlan0 (HaLow 802.11s, 921 MHz, 27 dBm)       │
│  MTU 1532 → bat0 hard interface                │
│                                                │
│  DHCP: 10.41.X.2–10.41.X.17 (16 leases)      │
│  DNS: dnsmasq on br-ahwlan                     │
└────────────────────────────────────────────────┘
          │  HaLow RF 920–921 MHz
          ▼
   Other OpenMANET Nodes (same mesh)
```

### Interface summary

| Interface | Type | IP | Purpose |
|-----------|------|----|---------|
| `wlan0` | Morse HaLow mesh (802.11s) | none | HaLow backhaul to bat0 |
| `bat0` | batman-adv | none (in br-ahwlan) | L2 mesh routing |
| `br-ahwlan` | Bridge (eth0 + bat0 + phy1-ap0) | 10.41.254.1/16 | Main management + mesh IP |
| `phy1-ap0` | AIC8800 WiFi AP | (in br-ahwlan) | Client access point |
| `eth0` | Ethernet | (in br-ahwlan) | Wired client access |

### Default credentials

| Service | Value |
|---------|-------|
| WiFi AP SSID | `OpenMANET-AP` |
| WiFi AP password | `openmanet123` |
| Mesh SSID / mesh_id | `OpenMANET` |
| Mesh encryption / key | SAE / `openmanet123` |
| Node IP | `10.41.254.1` (default, set by uci-defaults) |
| LuCI / SSH | `http://10.41.254.1` / `root` (no password by default) |

---

## 9. Build Instructions

### Prerequisites

```bash
# Ubuntu 24.04
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  gettext git libncurses-dev libssl-dev python3-distutils rsync unzip zlib1g-dev
```

### Full build from clean state

```bash
cd openmanet-firmware

# Initialize and run setup for Rock 2F SPI board
./scripts/openmanet_setup.sh -b rock2f-spi -i

# Full build (parallel)
make -j$(nproc) V=s 2>&1 | tee /tmp/build.log
```

The `-i` flag runs `scripts/feeds update -a && scripts/feeds install -a` before configuring.

### Rebuild after changing only the morse driver Makefile

```bash
cd openmanet-firmware
make package/feeds/openmanet/morse_driver/clean V=s
make package/feeds/openmanet/morse_driver/compile -j$(nproc) V=s
make package/index V=s
```

### Rebuild after changing kernel patches or DTS

```bash
cd openmanet-firmware
make target/linux/clean V=s
make -j$(nproc) V=s
```

### Rebuild after kernel .config goes stale (syncconfig failure)

If the kernel's `.config` in `build_dir/` gets out of sync (symptoms: build fails with
`Restart config... [NEW]` on kconfig symbols), clean the kernel config state:

```bash
KDIR=build_dir/target-aarch64_generic_musl/linux-rockchip_armv8/linux-6.6.102
rm -f "$KDIR/.config" "$KDIR/.config.prev" "$KDIR/.configured"
rm -f "$KDIR/include/generated/autoconf.h" "$KDIR/include/generated/rustc_cfg"
make -j$(nproc) V=s
```

### Output image

```
bin/targets/rockchip/armv8/openmanet-24.10-1.6.5-rockchip-armv8-radxa_rock-2f-squashfs-sysupgrade.img.gz
```

### Flash to microSD

```bash
gunzip -c openmanet-24.10-*-radxa_rock-2f-squashfs-sysupgrade.img.gz | \
  sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
```

Replace `/dev/sdX` with the microSD device. The image creates two partitions:
- Partition 1 (16 MB, type `0x83`): U-Boot + boot script + kernel
- Partition 2 (256 MB, type `0x83`): squashfs rootfs

---

## 10. Boot Sequence

```
Power on
  │
  ▼
U-Boot (Rockchip vendor)
  │  Loads boot.scr from partition 1
  │  Writes to GPIO MMIO registers: WiFi power on (GPIO4_A4, GPIO1_A6)
  │  Sets bootargs: console=ttyS0,1500000 earlycon=uart8250,mmio32,0xff9f0000
  │  Loads kernel.img + DTB, boots
  │
  ▼
Linux 6.6.102 (earlycon → ttyS0)
  │  RK3528 clock/reset/pinctrl drivers init → peripherals available
  │  USB2 PHY driver probes → USB host ports active
  │  AIC8800 detected on USB, firmware loaded → re-enumerates on USB bus
  │  SPI0 probed → morse,mm610x-spi device detected (chip still in reset)
  │
  ▼
OpenWrt init (procd)
  │
  ▼
kmodloader (modules-boot.d)
  │  Loads gpio-keys, USB storage, MMC, etc.
  │  morse.disabled + aic8800.disabled → skipped
  │
  ▼
/etc/init.d/morsechipreset  [START=09]
  │  Exports GPIO 143, drives LOW (reset), waits 1s
  │  Drives HIGH (release), waits 1s
  │  modprobe dot11ah
  │  modprobe morse country=US bcf=bcf_fgh100mhaamd.bin spi_clock_speed=10000000 ...
  │    → wlan0 registered (HaLow SPI interface)
  │  modprobe aic_load_fw
  │  modprobe aic8800_fdrv ps_on=N dpsm=N ap_uapsd_on=N
  │    → phy1-ap0 registered (USB WiFi AP)
  │  Disables USB autosuspend
  │
  ▼
/etc/init.d/fix-wifi-path  [START=18]
  │  Detects AIC8800 current USB path from sysfs
  │  Updates wireless.radio1.path in UCI if changed
  │
  ▼
kmodloader (modules.d)
  │  Processes remaining modules (morse/aic8800 entries are .disabled → skipped)
  │
  ▼
netifd
  │  Applies wireless config: morse radio (wlan0) → 802.11s mesh
  │  Applies wireless config: AIC radio (phy1-ap0) → AP
  │
  ▼
hotplug (99-batadv)
  │  On wlan0 up: set MTU 1532, batctl if add wlan0 to bat0
  │  On bat0 up: brctl addif br-ahwlan bat0
  │
  ▼
System ready
  │  HaLow mesh operational: 921 MHz, 27 dBm, 802.11s + SAE
  │  WiFi AP: OpenMANET-AP (WPA2, 5 GHz)
  │  batman-adv bat0: BATMAN_V, in br-ahwlan
  │  IP: 10.41.254.1/16 on br-ahwlan
```

---

## 11. Problems Encountered and Solutions

### 11.1 Silent boot hang ("Starting kernel...")

**Cause**: No RK3528 clock controller driver in Linux 6.6. Without clock driver, no peripheral
initializes, including UART → complete silence after earlycon.

**Fix**: Applied patches 032-19 through 032-22 (clock, reset, pinctrl backports). Added
`CONFIG_CLK_RK3528=y` and `CONFIG_PINCTRL_RK3528=y` to `target/linux/rockchip/config-6.6`.

### 11.2 AIC8800 WiFi not detected

**Cause**: No USB2 PHY driver for RK3528 in Linux 6.6.

**Fix**: Applied patches 160-01 through 160-04 (USB2 PHY backport) and 163-02 (USB DTS nodes).

### 11.3 All devices fail with "deferred probe timeout"

**Cause**: Power domain controller (patch 070-18) incomplete in backport. USB nodes had
`power-domains` references that caused probe deferral loops.

**Fix**: Patch 164 removes `power-domains` from all USB nodes. Safe because power domains default
to ON at reset.

### 11.4 MM6108 SPI probe failure (errno -5)

**Cause**: `reset-gpios = GPIO_ACTIVE_HIGH` in DTS kept MM6108 in reset (asserted LOW by driver,
but chip resets on LOW). SPI probe found only `0xFFFFFFFF` on MISO.

**Fix**: Changed to `GPIO_ACTIVE_LOW` in patch 073.

### 11.5 SPI protocol errors (errno -71) at 50 MHz

**Cause**: Signal integrity issues at 50 MHz on the HAT's PCB traces and through the mini-PCIe
connector.

**Fix**: `spi-max-frequency = <10000000>` (10 MHz) in DTS + `spi_clock_speed=10000000` at modprobe.

### 11.6 chipreset.sh fails with "No GPIO named MM_RESET"

**Cause**: The morse-feed's `chipreset.sh` uses `gpiofind MM_RESET` which requires
`gpio-line-names` in the GPIO controller's DTS node — not implemented for RK3528 GPIO4.

**Fix**: Custom `morsechipreset` init script that uses direct sysfs GPIO control (`/sys/class/gpio/143`).

### 11.7 Kernel crash — "scheduling while atomic" in AIC8800

**Cause**: IPv6 MLD multicast packets trigger `rwnx_send_mesh_proxy_add_req()` in the AIC8800
driver from atomic context (a workqueue with a spinlock held). The function calls `msleep()`,
which is illegal in atomic context.

**Fix**: Disable IPv6 system-wide via `sysctl.d/99-disable-ipv6.conf`. This prevents MLD
processing entirely and eliminates the trigger condition.

### 11.8 AIC8800 WiFi AP not starting after reboot

**Cause**: AIC8800 re-enumerates on a different USB bus number after firmware upload. OpenWrt's
wireless config stores the full USB bus path; a stale path means the radio is not found.

**Fix**: `fix-wifi-path` init script detects the current USB sysfs path by matching the SoC's
USB host controller MMIO address (`ff100000`) and updates the UCI config if it changed.

### 11.9 HaLow visible on 5 GHz (iwinfo/iw confusion)

**Cause**: dot11ah shim maps 802.11ah S1G channels to 5 GHz mac80211 channel equivalents. This is
by design; mac80211 does not natively support sub-1 GHz frequencies.

**Information**: Not a bug — use `morse_cli -i wlan0 channel` to see the actual 921 MHz frequency.

### 11.10 kmod-morse compiles without SPI support

**Cause**: `CONFIG_MORSE_SPI=y` was missing from `target_diffconfig`. The
`common_extras/spi_diffconfig` file provides it but also includes BCM-specific kernel modules
unsuitable for Rockchip.

**Fix**: Added `CONFIG_MORSE_SPI=y` directly to `boards/rock2f-spi/target_diffconfig`.

### 11.11 Morse driver compile error: SPI_CONTROLLER_ENABLE_CS_GPIOD

**Cause**: Macro added in kernel 6.9; driver emits `#warning` which `-Werror=cpp` promotes to
error. See [Section 6.3](#63-problem-spi_controller_enable_cs_gpiod-not-defined-in-kernel-66)
for full analysis.

**Fix**: `NOSTDINC_FLAGS += -DSPI_CONTROLLER_ENABLE_CS_GPIOD='(1<<9)'` in morse driver Makefile.

### 11.12 Kernel syncconfig interactive failure (Restart config... [NEW])

**Cause**: The kernel `.config` in `build_dir/` became stale (e.g., after manually editing files in
the build dir changed their timestamps, triggering a config rebuild). The new `.config.set`
generated by OpenWrt differed from `.config.prev` in a way that the cached `.config` (from an
earlier successful build) did not have certain symbols set (e.g., `CONFIG_ARCH_ACTIONS`). Running
`make syncconfig` non-interactively fails when new symbols are encountered without defaults.

**Fix**:
```bash
KDIR=build_dir/target-aarch64_generic_musl/linux-rockchip_armv8/linux-6.6.102
rm -f "$KDIR/.config" "$KDIR/.config.prev" "$KDIR/.configured"
rm -f "$KDIR/include/generated/autoconf.h" "$KDIR/include/generated/rustc_cfg"
make -j$(nproc) V=s
```

### 11.13 Kernel module build fails with CONFIG_ARM64_PA_BITS undeclared

**Cause**: Editing files in `build_dir/` (e.g., `spi.c`, `spi.h`) changes their timestamps, which
causes `make` to believe those files are newer than the kernel's compiled objects. This triggers
a rebuild of `gpio-button-hotplug` and other kernel modules, but outside of a full build the arm64
`asm/mmu.h` include chain is broken because the kernel's `autoconf.h` has been deleted/regenerated
mid-build.

**Fix**: Never edit files in `build_dir/` directly. All fixes should be made at the source level
(kernel patches, package Makefiles, overlay files). Always run the full `make -j$(nproc)` which
handles dependency ordering correctly.

### 11.14 Rock 2F bootloop with HAT fully seated

**Cause**: Combined 3.3V current draw (MM6108 + GPS L76KB + ATECC608B + HAT regulator) causes
brownout during boot. Additional contributing factors: GPS UART NMEA data on UART2 pins (8, 10)
and GPS 1PPS on UART1_TX (pin 7) during kernel init.

**Fix**: Physical mitigation — tape over pins 3, 5, 7, 8, 10, 27, 28 to expose only the SPI
and MM6108 control GPIOs, or supply the HAT from an external 3.3V source.

---

## 12. Known Issues

| Issue | Impact | Status |
|-------|--------|--------|
| `gpio-line-names` not defined for GPIO4 | Cannot use upstream `chipreset.sh` | Workaround: sysfs GPIO in custom init script |
| AIC8800 USB path change between boots | WiFi AP may not start on first boot after kernel update | Workaround: `fix-wifi-path` init script |
| AIC8800 IPv6 atomic crash | System crash if IPv6 is enabled | Fixed: IPv6 disabled via sysctl |
| AIC8800 ~500ms periodic latency spikes | WiFi AP responsiveness | Known; reducing HE/WiFi6 modes may help |
| HAT brownout with all devices seated | System fails to boot with full HAT | Workaround: physical pin isolation or external 3.3V |
| `openmanetd` not compiled | No automatic mesh IP addressing, no Alfred sync | Go submodule dependency issue; shell workaround available |
| BATMAN_V vs BATMAN_IV | Current build uses BATMAN_V; cannot mix with older BATMAN_IV nodes | Design decision — document node firmware compatibility |

---

## 13. Complete File Change Index

### New files added

```
boards/rock2f-spi/
├── target_diffconfig                    # Board config: target + packages (CONFIG_MORSE_SPI=y)
├── dev_diffconfig                       # Dev build options
├── utils_diffconfig                     # Utility packages
├── rangetest_diffconfig                 # Range test app
├── wireshark_diffconfig                 # Wireshark
└── files/
    └── etc/
        ├── init.d/
        │   ├── morsechipreset           # GPIO reset + ordered driver loading
        │   └── fix-wifi-path            # AIC8800 USB path fix
        ├── hotplug.d/net/
        │   └── 99-batadv               # HaLow → bat0 + bat0 → br-ahwlan
        ├── sysctl.d/
        │   └── 99-disable-ipv6.conf    # IPv6 disable (AIC8800 crash fix)
        └── uci-defaults/
            └── 99-rock2f-openmanet     # First-boot wireless/network configuration

target/linux/rockchip/patches-6.6/
├── 032-19-*.patch                       # PLL FIXED_MODE flag
├── 032-20-*.patch                       # RK3528 clock controller driver
├── 032-21-*.patch                       # RK3528 reset table (manual 6.6 fix)
├── 032-22-*.patch                       # RK3528 pinctrl
├── 071-*.patch                          # Rock 2A/2F DTS Makefile
├── 072-*.patch                          # Rock 2A/2F board DTS
├── 073-*.patch                          # USB + WiFi power + SPI0 + MM6108
├── 160-01-*.patch                       # USB2 PHY: GRF handling
├── 160-02-*.patch                       # USB2 PHY: clkout
├── 160-03-*.patch                       # USB2 PHY: RK3528 support
├── 160-04-local-...-clkout-helper.patch # USB2 PHY: clkout helper
├── 160-04-local-...-rk3528-compat.patch # USB2 PHY: RK3528 compat
├── 163-02-*.patch                       # USB DTS nodes
└── 164-*.patch                          # Remove USB power-domains

target/linux/rockchip/files-6.6/include/dt-bindings/
├── clock/rockchip,rk3528-cru.h         # Clock IDs (not in upstream 6.6)
├── reset/rockchip,rk3528-cru.h         # Reset IDs
└── power/rockchip,rk3528-power.h       # Power domain IDs

target/linux/rockchip/image/
└── rock-2.bootscript                    # Custom boot script (ttyS0, WiFi GPIO)

package/kernel/aic8800-usb/             # AIC8800D80 out-of-tree driver package
package/firmware/aic8800-firmware/      # AIC8800 firmware binaries
```

### Modified files

| File | Change |
|------|--------|
| `target/linux/rockchip/config-6.6` | Added `CONFIG_CLK_RK3528=y`, `CONFIG_PINCTRL_RK3528=y` |
| `target/linux/rockchip/image/armv8.mk` | Added Rock 2A and Rock 2F image definitions with `BOOT_SCRIPT := rock-2` |
| `feeds.conf.default` | Added `src-git morse` and `src-git openmanet` feed entries |
| `package/feeds/openmanet/morse_driver/Makefile` | Added `NOSTDINC_FLAGS += -DSPI_CONTROLLER_ENABLE_CS_GPIOD='(1<<9)'` |

---

## 14. Future Work

1. **Add `gpio-line-names` to RK3528 GPIO4 DTS** — allows the upstream
   `morse/scripts/chipreset.sh` to find `MM_RESET` by name, removing the need for the custom
   sysfs-based init script.

2. **Fix AIC8800 IPv6 atomic context bug** — patch `rwnx_send_mesh_proxy_add_req` to be
   atomic-safe, then remove the IPv6 sysctl disable.

3. **Compile openmanetd** — the Go daemon that provides automatic mesh IP addressing, Alfred node
   sync, and gateway discovery. Currently blocked by Go submodule dependency issues.

4. **Upstream RK3528 support to OpenWrt mainline** — so future OpenWrt updates include the SoC
   support and this port's BSP patches can be removed.

5. **Test multi-node mesh** — validate batman-adv routing, node addressing, and throughput across
   two or more Rock 2F nodes at 921 MHz.

6. **Investigate AIC8800 latency spikes** — determine if the ~500ms periodic latency is caused
   by the USB hub, the driver power management, or the AIC8800 AP mode implementation.

7. **GPS integration** — the WM1302 HAT includes an L76KB GPS module on UART2 (pins 8/10). Once
   the HAT pin conflicts are resolved (external 3.3V + pin isolation), `gpsd` can be configured
   to read `/dev/ttyS2`.
