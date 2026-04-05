# OpenMANET on Radxa Rock 2F — Complete Porting Guide

## Project Overview

This document describes the complete process of porting [OpenMANET](https://openmanet.github.io/docs/) — a Raspberry Pi–based MANET (Mobile Ad-Hoc Network) radio system built on Wi-Fi HaLow (802.11ah) — to the Radxa Rock 2F single-board computer based on the Rockchip RK3528A SoC.

The port required three major phases of work:

1. **Phase 1 — RK3528 SoC support in OpenWrt 24.10**: Backporting clock, reset, pinctrl, USB PHY, and device tree support from upstream kernel 6.12–6.18 into kernel 6.6.127.
2. **Phase 2 — HaLow radio integration**: Adding the Morse Micro MM6108 SPI device to the board device tree, integrating the morse-feed and openmanet package feeds, and configuring the mesh stack.
3. **Phase 3 — Runtime integration**: Fixing driver load ordering, GPIO reset sequencing, BCF (Board Configuration File) selection, AIC8800 WiFi AP coexistence, and batman-adv mesh routing.

### Hardware

- **SBC**: Radxa Rock 2F (RK3528A quad-core Cortex-A53, LPDDR4)
- **HaLow module**: Seeed Wio-WM6108 (Quectel FGH100M-H with Morse Micro MM6108 chipset) on Seeed WM1302 Pi HAT, connected via SPI on the 40-pin GPIO header
- **Onboard WiFi**: AIC8800D80 (USB, connected internally via FE1.1s hub on EHCI bus)
- **Storage**: microSD card + onboard eMMC

### Software

- **Base OS**: OpenWrt 24.10-SNAPSHOT (kernel 6.6.127)
- **HaLow driver**: Morse Micro morse-feed (driver v1.16.4)
- **Mesh stack**: wpa_supplicant_s1g (802.11s) + batman-adv (BATMAN_IV)
- **WiFi AP**: AIC8800D80 driver (hostapd, 5 GHz HE80)

### Build Host

- Ubuntu 24.04 or later
- Standard OpenWrt build dependencies

---

## 1. Build Environment Setup

### 1.1 Clone and prepare

```bash
git clone https://git.openwrt.org/openwrt/openwrt.git openwrt-24.10
cd openwrt-24.10
git checkout openwrt-24.10
```

### 1.2 Feed configuration

The file `feeds.conf.default` must include the morse-feed and openmanet feeds in addition to the standard OpenWrt feeds:

```
src-git packages https://git.openwrt.org/feed/packages.git;openwrt-24.10
src-git luci https://git.openwrt.org/project/luci.git;openwrt-24.10
src-git routing https://git.openwrt.org/feed/routing.git;openwrt-24.10
src-git telephony https://git.openwrt.org/feed/telephony.git;openwrt-24.10
src-git morse https://github.com/MorseMicro/morse-feed.git^263b4c4ed8ae1683a180ce9cd28d3516c40dbce5
src-git openmanet https://github.com/OpenMANET/packages.git^f38f030beb67eb8da1e29cc76d9d08e668ed1a42
```

Install feeds:

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

**Note**: A recursive dependency warning between `netifd`, `wifi-scripts`, `kmod-cfg80211`, `kmod-morse`, `hostapd_s1g`, and `netifd-morse` will appear during `make defconfig`. This is harmless and does not affect the build.

---

## 2. RK3528 SoC Support — Kernel Backports

The RK3528 SoC was not supported in mainline Linux 6.6. All SoC support had to be backported from newer kernel versions (6.12–6.18) using patches from an older OpenWrt snapshot tree that had kernel 6.12 support.

### 2.1 Clock, reset, and pinctrl driver patches

Without these drivers, the kernel cannot initialize any peripheral — not even the UART — causing a silent hang at "Starting kernel..."

All patches are placed in `target/linux/rockchip/patches-6.6/`:

| Patch | Description | Notes |
|-------|-------------|-------|
| `032-19-v6.15-clk-rockchip-Add-PLL-flag-ROCKCHIP_PLL_FIXED_MODE.patch` | Adds PLL FIXED_MODE flag | Prerequisite for RK3528 clock driver. Applied cleanly. |
| `032-20-v6.15-clk-rockchip-Add-clock-controller-driver-for-RK3528-SoC.patch` | Complete `clk-rk3528.c` clock controller driver | Most critical patch. Applied cleanly (mostly new files). |
| `032-21-v6.15-clk-rockchip-rk3528-Add-reset-lookup-table.patch` | Reset controller lookup table (`rst-rk3528.c`) | **Required manual fix**: the `clk.h` hunk failed because 6.6 only has `rk3588_rst_init` (no `rk3576_rst_init`). Fixed by adjusting line offset and changing context from `rk3576_rst_init` to `rk3588_rst_init`. |
| `032-22-v6.15-pinctrl-rockchip-Add-support-for-RK3528.patch` | RK3528 pinctrl support in `pinctrl-rockchip.c` | Applied cleanly. |

**Patches intentionally excluded:**
- `032-26` (SD/SDIO tuning clocks in GRF) — Failed to apply; requires `enum rockchip_grf_type` members not present in 6.6. Not needed for basic operation.
- `032-27` (slab.h header fix) — Depends on 032-26. Removed.

### 2.2 Kernel config additions

Added to `target/linux/rockchip/config-6.6`:

```
CONFIG_CLK_RK3528=y
CONFIG_PINCTRL_RK3528=y
```

These must be present or the build will prompt interactively and fail in automated builds.

### 2.3 DT-binding header files

Placed in `target/linux/rockchip/files-6.6/include/dt-bindings/`:

- `clock/rockchip,rk3528-cru.h` — Clock IDs
- `reset/rockchip,rk3528-cru.h` — Reset IDs
- `power/rockchip,rk3528-power.h` — Power domain IDs

These headers are required by `rk3528.dtsi` and do not exist in upstream Linux 6.6.

### 2.4 RK3528 device tree patches

Series `070-01` through `070-22` add the base RK3528 DTS nodes (GPIO, UART, SDMMC, SDHCI, I2C, SPI, PWM, DMA, SARADC, GMAC, GPU, power controller, etc.). These were already present from the OpenWrt 24.10 snapshot tree.

Additional board-level patches:

| Patch | Description |
|-------|-------------|
| `071-arm64-dts-rockchip-Add-Radxa-ROCK-2A-2F-Makefile.patch` | Adds Rock 2A/2F to the DTS build Makefile |
| `072-v6.18-arm64-dts-rockchip-Add-Radxa-ROCK-2A-2F.patch` | Board-level DTS for Rock 2A and 2F |

---

## 3. USB PHY Support

The RK3528's USB2 PHY driver did not exist in Linux 6.6. Without it, USB host ports cannot function and the onboard AIC8800 WiFi chip (connected via internal USB) cannot be detected.

### 3.1 USB PHY driver patches

| Patch | Description |
|-------|-------------|
| `160-01-phy-rockchip-inno-usb2-Simplify-rockchip-usbgrf-handling.patch` | Simplifies GRF handling |
| `160-02-phy-rockchip-inno-usb2-Add-clkout_ctl_phy-support.patch` | Adds clock output control |
| `160-03-phy-rockchip-inno-usb2-Add-support-for-RK3528.patch` | RK3528 USB2 PHY support |
| `160-04-local-phy-rockchip-inno-usb2-clkout-helper.patch` | Clock output helper |
| `160-04-local-phy-rockchip-inno-usb2-rk3528-compat.patch` | RK3528 compatibility fixes |

**Important**: All five patches must be enabled together. The compat patch causes a build error (`rk3528_usb2phy_tuning defined but not used`) when the other 160-xx patches are disabled.

### 3.2 USB DTS nodes

| Patch | Description |
|-------|-------------|
| `163-02-arm64-dts-rockchip-Add-USB-nodes-for-RK3528.patch` | Adds USB controller and PHY nodes to `rk3528.dtsi` |

Adds `usb_host0_xhci` (DWC3), `usb_host0_ehci` (EHCI), `usb_host0_ohci` (OHCI), and `usb2phy` nodes, all with `status = "disabled"` (enabled in the board DTS).

### 3.3 USB power domain removal

| Patch | Description |
|-------|-------------|
| `164-arm64-dts-rockchip-Remove-USB-power-domains-RK3528.patch` | Removes `power-domains` from USB nodes |

The RK3528 power domain controller is not fully functional in this backport. Enabling the power domain DTS patch (070-24) caused all devices to fail with "deferred probe timeout." Removing `power-domains` from USB nodes is safe because Rockchip SoCs default power domains to ON at boot.

---

## 4. Board DTS Patch — 073 (The Critical Patch)

**File**: `target/linux/rockchip/patches-6.6/073-arm64-dts-rockchip-Enable-USB-for-Rock-2.patch`

This patch modifies `rk3528-rock-2.dtsi` to enable all hardware needed for OpenMANET. It is the single most important customization for the Rock 2F.

### What it adds:

#### WiFi power regulator fix
```dts
regulator-always-on;
regulator-boot-on;
```
Added to the `vcc_wifi` regulator node so the AIC8800D80 WiFi chip receives power at boot without requiring GPIO toggling from userspace.

#### USB controller enablement
Enables `usb2phy`, `usb2phy_host`, `usb2phy_otg`, `usb_host0_ehci`, `usb_host0_ohci`, and `usb_host0_xhci` nodes.

#### SPI0 controller with MM6108 device
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

**Critical detail — Reset GPIO polarity**: `GPIO_ACTIVE_LOW` means the driver asserts LOW to reset and releases to HIGH for normal operation. This was initially set to `GPIO_ACTIVE_HIGH` which kept the MM6108 in permanent reset, causing SPI probe failure (errno -5, MISO reads all `0xFFFFFFFF`). The polarity was discovered by manual testing with `sysfs` GPIO control.

#### GPIO pin mapping (Rock 2F 40-pin header)

| Function | GPIO | Pin # | RK designation |
|----------|------|-------|----------------|
| SPI0_MOSI | GPIO4_B2 | 19 | Alt function 2 |
| SPI0_MISO | GPIO4_B3 | 21 | Alt function 2 |
| SPI0_CLK | GPIO4_B4 | 23 | Alt function 2 |
| SPI0_CSN0 | GPIO4_B6 | 24 | Alt function 2 |
| Reset | GPIO4_B7 | 11 | GPIO, pull none |
| Wake | GPIO4_B0 | 16 | GPIO, pull none |
| Busy | GPIO4_B1 | 18 | GPIO, pull down |
| IRQ | GPIO4_B5 | 29 | GPIO, pull up |

#### Pinctrl entries
Adds `morse-mm6108` pinctrl group with entries for reset, wake, busy, and IRQ pins.

---

## 5. Boot Script

### Problem

The default OpenWrt boot script uses `console=ttyS2,1500000` which is correct for most Rockchip boards but wrong for the Rock 2F which uses `ttyS0` at `0xff9f0000`. This caused the console to stop working after earlycon handoff.

### Solution

**File**: `target/linux/rockchip/image/rock-2.bootscript`

```bash
# Power on WiFi GPIO4_A4 (GPIO4 base 0xffb20000, bit 4)
mw.l 0xffb20000 0x00100010

# WIFI_REG_ON GPIO1_A6 (GPIO1 base 0xffaf0000, bit 6)
mw.l 0xffaf0000 0x00400040

sleep 1

part uuid ${devtype} ${devnum}:2 uuid
setenv bootargs "console=ttyS0,1500000 earlycon=uart8250,mmio32,0xff9f0000 root=PARTUUID=${uuid} rw rootwait"
load ${devtype} ${devnum}:1 ${kernel_addr_r} kernel.img
bootm ${kernel_addr_r}
```

The boot script also toggles WiFi power GPIOs at the U-Boot level using memory-mapped register writes, which is necessary because the `vcc_wifi` regulator driver isn't available in U-Boot.

**File**: `target/linux/rockchip/image/armv8.mk`

Added `BOOT_SCRIPT := rock-2` to both Rock 2A and Rock 2F device definitions.

---

## 6. AIC8800D80 WiFi Driver Package

### Package: `package/kernel/aic8800-usb/`

Custom OpenWrt package that builds the AIC8800D80 USB WiFi driver as an out-of-tree kernel module. The driver source is placed in `/tmp/aic8800-src/` on the build host before compilation.

The driver required backport header compatibility fixes to build against the mac80211 6.12.6 backport headers used in OpenWrt 24.10.

### Package: `package/firmware/aic8800-firmware/`

Firmware binaries for the AIC8800D80 chip.

### Firmware overlay: `files/vendor/etc/firmware`

Symlink pointing to `/lib/firmware/aic8800_fw/USB/aic8800D80` — required because the AIC8800 driver looks for firmware in `/vendor/etc/firmware/`.

---

## 7. Runtime Configuration — Overlay Files

All runtime configuration files are placed in the `files/` directory of the OpenWrt build tree and are baked into the firmware image.

### 7.1 Init script: `files/etc/init.d/morsechipreset`

This script replaces the morse-feed's default `morsechipreset` (which relies on `gpio-line-names` in the device tree — not yet implemented for RK3528).

```bash
#!/bin/sh /etc/rc.common
START=09

boot() {
    # 1. Reset MM6108 via GPIO 143 (GPIO4_B7, pin 11)
    # LOW to reset, HIGH to run
    logger -t morsechipreset "Resetting MM6108..."
    echo 143 > /sys/class/gpio/export 2>/dev/null
    echo out > /sys/class/gpio/gpio143/direction 2>/dev/null
    echo 0 > /sys/class/gpio/gpio143/value
    sleep 1
    echo 1 > /sys/class/gpio/gpio143/value
    sleep 1

    # 2. Load morse FIRST so it registers wlan0
    logger -t morsechipreset "Loading morse driver..."
    modprobe dot11ah
    modprobe morse country=US bcf=bcf_fgh100mhaamd.bin spi_clock_speed=10000000 \
        enable_ps=0 enable_dynamic_ps_offload=0
    sleep 2

    # 3. Load AIC8800 SECOND (gets a separate interface)
    logger -t morsechipreset "Loading AIC8800 driver..."
    modprobe aic_load_fw
    modprobe aic8800_fdrv ps_on=N dpsm=N ap_uapsd_on=N
    sleep 3

    # 4. Disable USB autosuspend
    echo -1 > /sys/module/usbcore/parameters/autosuspend 2>/dev/null

    logger -t morsechipreset "All drivers loaded"
}

start() {
    boot
}
```

**Why driver load ordering matters**: If the AIC8800 loads first, it creates `wlan0`. When netifd then applies the morse mesh config to `wlan0`, the AIC8800 receives a `MESH_START_REQ` command it doesn't support, causing a kernel crash (`scheduling while atomic` in `aicwf_rxbuff_enqueue`). Loading morse first ensures it owns `wlan0` for the mesh, and AIC8800 gets a separate interface (`phy1-ap0`).

**Why modules.d auto-loading must be disabled**: OpenWrt's `kmodloader` processes `/etc/modules.d/*` before any `/etc/rc.d/` init scripts run. The morse module would be loaded before our GPIO reset script, causing SPI probe failure. Both `morse` and `aic8800-usb` modules.d entries are renamed to `.disabled` by the first-boot script.

### 7.2 WiFi path fixer: `files/etc/init.d/fix-wifi-path`

The AIC8800D80 chip is connected via USB through an internal hub. During firmware upload, the USB device disconnects and re-enumerates, sometimes on a different USB bus number (`usb3` vs `usb4`). This causes the `path` in the wireless config to mismatch on subsequent boots.

```bash
#!/bin/sh /etc/rc.common
START=18

boot() {
    sleep 3
    for phy in /sys/class/ieee80211/phy*; do
        devpath=$(readlink -f "$phy/device")
        case "$devpath" in
            *ff100000.usb*)
                relpath=${devpath#*/platform/}
                relpath="platform/${relpath}"
                current=$(uci -q get wireless.radio1.path)
                if [ "$current" != "$relpath" ]; then
                    uci set wireless.radio1.path="$relpath"
                    uci commit wireless
                    logger -t fix-wifi-path "Updated radio1 path to $relpath"
                fi
                ;;
        esac
    done
}

start() {
    boot
}
```

### 7.3 batman-adv hotplug: `files/etc/hotplug.d/net/99-batadv`

Automatically adds `wlan0` to the batman-adv `bat0` mesh interface when it comes up, and sets the MTU to 1532 (batman-adv requires 32 bytes of overhead).

```bash
[ "$ACTION" = "add" ] && [ "$INTERFACE" = "wlan0" ] && {
    sleep 2
    ip link set wlan0 mtu 1532 2>/dev/null
    batctl meshif bat0 if add wlan0 2>/dev/null
    logger -t batadv "Added wlan0 to bat0 with MTU 1532"
}
```

### 7.4 IPv6 disable: `files/etc/sysctl.d/99-disable-ipv6.conf`

The AIC8800 driver has a bug where IPv6 MLD (Multicast Listener Discovery) packets trigger `rwnx_send_mesh_proxy_add_req` in atomic context, causing a kernel crash. Disabling IPv6 at the kernel level prevents this.

```
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
```

### 7.5 First-boot configuration: `files/etc/uci-defaults/99-openmanet`

This script runs once on first boot and configures the wireless and network subsystems. It auto-detects radios by type rather than hardcoding radio numbers (which are dynamically assigned by `wifi config`).

Key actions:
- Identifies the morse radio (type `morse`) and AIC8800 radio (type `mac80211`, path contains `usb`)
- Disables the dot11ah mac80211 shim radio (SPI path, not the morse type)
- Configures the morse radio for 802.11s mesh on bat0 (channel 38, 921 MHz, SAE encryption)
- Configures the AIC8800 radio as an AP on the LAN bridge (SSID "OpenMANET-AP", WPA2-PSK)
- Creates batman-adv `bat0` interface (BATMAN_IV routing)
- Creates mesh bridge with IP 10.41.254.1/16
- Sets LAN IP to 192.168.1.1
- Disables IPv6
- Enables init scripts
- Disables auto-loading of morse and AIC modules from `/etc/modules.d/`
- Fixes the BCF symlink: `bcf_default.bin → bcf_fgh100mhaamd.bin`

### 7.6 Firmware overlay: `files/vendor/etc/firmware`

Symlink to `/lib/firmware/aic8800_fw/USB/aic8800D80` — the AIC8800 driver looks for firmware files in this non-standard path.

---

## 8. BCF (Board Configuration File) Selection

The BCF controls the RF front-end configuration of the Morse Micro chip, including power amplifier enable, TX power calibration, and frequency band settings.

### The problem

The morse-feed ships `bcf_default.bin` as a symlink to `bcf_failsafe.bin` — a minimal failsafe configuration with the **power amplifier disabled**. With this BCF, the driver reports 920.5 MHz operation and transmits beacons, but no actual RF signal is emitted (invisible on SDR).

### The solution

The Seeed Wio-WM6108 uses the **Quectel FGH100M-H** transceiver. The correct BCF is **`bcf_fgh100mhaamd.bin`** (1251 bytes, CRC32 `0x941b2a82`).

With the correct BCF:
- TX power increases from 0 dBm to **27 dBm** (matching OpenMANET's documented ~27 dBm)
- Beacons are visible on SDR at 921 MHz
- The RF front-end (PA) is properly enabled

The first-boot script (`99-openmanet`) changes the symlink:
```bash
ln -sf /lib/firmware/morse/bcf_fgh100mhaamd.bin /lib/firmware/morse/bcf_default.bin
```

**Note**: The `bcf` module parameter in `/etc/modules.d/morse` is ignored by the driver when it falls back to `bcf_default.bin`. The symlink approach is the only reliable method.

**Other BCFs available but incorrect for this hardware**: `bcf_mf08651_us.bin` (for the MF08651 reference board, not the Quectel FGH100M-H), `bcf_failsafe.bin` (PA disabled).

---

## 9. Seeed WM1302 HAT — Hardware Integration Notes

### HAT bootloop issue

The WM1302 HAT, when fully seated on the 40-pin header, causes the Rock 2F to bootloop. The root cause is a **3.3V brownout** — the combined current draw of the MM6108, GPS module (L76KB), ATECC608B crypto chip, and HAT voltage regulator (MP2161GJ) exceeds what the Rock 2F can supply during boot.

Additional contributing factors:
- **Pin 7 (GPIO4_A6 / UART1_TX)**: HAT's GPS 1PPS signal interferes with UART1 initialization
- **Pin 8/10 (UART2_TX/RX)**: GPS module sends NMEA data during boot
- **Pin 27/28 (I2C1)**: HAT's ID EEPROM conflicts with the board's own EEPROM at address `0x50`

### Working configuration

The HAT boots reliably when the non-essential pin conflicts are mitigated. The brownout is the primary issue and may require:
- An external 3.3V supply for the HAT
- Cutting traces to the GPS module on the HAT
- Staggered power-on

---

## 10. Network Architecture

```
┌─────────────────────────────────────────┐
│            Radxa Rock 2F                │
│                                         │
│  ┌─────────┐     ┌──────────────────┐   │
│  │ AIC8800 │     │ MM6108 HaLow     │   │
│  │ WiFi AP │     │ 921 MHz mesh     │   │
│  │ 5 GHz   │     │ wpa_supplicant   │   │
│  │phy1-ap0 │     │ _s1g (wlan0)     │   │
│  └────┬────┘     └────────┬─────────┘   │
│       │                   │ MTU 1532    │
│   br-lan              bat0 (batman-adv) │
│   192.168.1.1/24       10.41.254.1/16   │
│       │                   │             │
│  ┌────┴───────────────────┴─────┐       │
│  │     Kernel routing/bridging  │       │
│  └──────────────────────────────┘       │
└─────────────────────────────────────────┘
          │                    │
     WiFi clients         HaLow mesh
     (phones, laptops)    (other OpenMANET nodes)
```

- **WiFi AP** (`phy1-ap0`): SSID "OpenMANET-AP", WPA2-PSK, 5 GHz HE80, bridge to `br-lan`
- **HaLow mesh** (`wlan0`): SSID "OpenMANET", SAE encryption, 921 MHz (channel 38, 2 MHz BW), batman-adv on `bat0`
- **Mesh network**: 10.41.0.0/16 (compatible with OpenMANET addressing)
- **Management network**: 192.168.1.0/24

---

## 11. Build Configuration (.config)

Key packages that must be selected:

```
CONFIG_PACKAGE_kmod-morse=y
CONFIG_PACKAGE_hostapd_s1g=y
CONFIG_PACKAGE_netifd-morse=y
CONFIG_PACKAGE_kmod-batman-adv=y
CONFIG_PACKAGE_batctl-full=y
CONFIG_PACKAGE_kmod-aic8800-usb=y
CONFIG_PACKAGE_aic8800-firmware=y
```

---

## 12. Building the Firmware

```bash
cd ~/openwrt-rock2f/openwrt-24.10

# Clean kernel (required if DTS patches changed)
make target/linux/clean V=s

# Build
make -j$(nproc) V=s 2>&1 | tee /tmp/build-openmanet.log

# Output images
ls -lh bin/targets/rockchip/armv8/openwrt-*rock-2f*
```

### Flashing

```bash
dd if=bin/targets/rockchip/armv8/openwrt-rockchip-armv8-radxa_rock-2f-squashfs-sysupgrade.img \
   of=/dev/sdX bs=4M status=progress
```

---

## 13. Boot Sequence (What Happens at Power-On)

1. **U-Boot**: Loads `boot.scr` from partition 1
2. **boot.scr**: Toggles WiFi power GPIOs via MMIO writes, sets bootargs with `console=ttyS0,1500000`, loads kernel
3. **Kernel boot**: RK3528 clock/pinctrl/USB drivers initialize, serial console on ttyS0
4. **kmodloader** (`/etc/modules-boot.d/*`): Loads basic modules (GPIO, USB, MMC)
5. **`/etc/init.d/morsechipreset` (START=09)**: Resets MM6108 via GPIO 143 (LOW→HIGH), loads `dot11ah` + `morse` driver with correct BCF, then loads `aic_load_fw` + `aic8800_fdrv`
6. **`/etc/init.d/fix-wifi-path` (START=18)**: Detects AIC8800 USB path and updates wireless config
7. **kmodloader** (`/etc/modules.d/*`): Loads remaining modules (morse and AIC modules.d entries are `.disabled`, so they're skipped)
8. **netifd**: Brings up wireless interfaces — morse mesh on `wlan0`, AIC AP on `phy1-ap0`
9. **hotplug** (`99-batadv`): Adds `wlan0` to `bat0` batman-adv interface

---

## 14. Known Issues and Workarounds

### 14.1 AIC8800 WiFi latency spikes

Periodic ~500ms latency spikes on the WiFi AP link. Root cause is the AIC8800 driver's power management or USB scheduling. Partially mitigated by:
- Setting `ps_on=N dpsm=N ap_uapsd_on=N` module parameters
- Disabling USB autosuspend

The spikes are independent of the HaLow radio and occur even with morse disabled.

### 14.2 AIC8800 IPv6 MLD crash

The AIC8800 driver crashes when IPv6 MLD multicast packets trigger `rwnx_send_mesh_proxy_add_req` in atomic context. Fixed by disabling IPv6 system-wide via sysctl.

### 14.3 USB path changes between boots

The AIC8800's USB bus number changes between boots (usb3 vs usb4) because the chip disconnects during firmware upload and re-enumerates on a different bus. Fixed by the `fix-wifi-path` init script.

### 14.4 chipreset.sh requires gpio-line-names

The morse-feed's `chipreset.sh` expects `MM_RESET` in the device tree's `gpio-line-names` property. This has not been implemented for RK3528. The custom `morsechipreset` init script bypasses this by using direct sysfs GPIO control.

### 14.5 SPI clock speed

The DTS sets `spi-max-frequency` to 10 MHz. The default driver speed of 50 MHz causes SPI protocol errors (errno -71) with the HAT's signal integrity. The module parameter `spi_clock_speed=10000000` is passed at modprobe time.

---

## 15. Quick Rebuild Reference

### Full clean rebuild:
```bash
make target/linux/clean
make -j$(nproc) V=s
```

### Rebuild kernel only:
```bash
make target/linux/clean
make target/linux/compile -j1 V=s
```

### Rebuild AIC driver only:
```bash
make package/kernel/aic8800-usb/clean V=s
make package/kernel/aic8800-usb/compile -j1 V=s
```

### Find output images:
```bash
ls -lh bin/targets/rockchip/armv8/openwrt-*rock-2f*
```

### Manual U-Boot boot (if boot.scr fails):
```
load mmc 1:1 0x02000000 /kernel.img
setenv bootargs earlycon=uart8250,mmio32,0xff9f0000 console=ttyS0,1500000n8 root=/dev/mmcblk1p2 rootfstype=squashfs rootwait
bootm 0x02000000#config-1
```

---

## 16. Files Summary

### Kernel patches (`target/linux/rockchip/patches-6.6/`)

| Patch | Description |
|-------|-------------|
| 032-19 | PLL FIXED_MODE flag |
| 032-20 | RK3528 clock controller driver |
| 032-21 | RK3528 reset lookup table (manually fixed for 6.6) |
| 032-22 | RK3528 pinctrl support |
| 070-01 through 070-22 | RK3528 DTS nodes |
| 071 | Rock 2A/2F DTS Makefile |
| 072 | Rock 2A/2F board DTS |
| **073** | **USB + WiFi power + SPI0 + MM6108 + GPIO pinctrl** |
| 160-01 through 160-04 | USB2 PHY driver |
| 163-02 | USB DTS nodes for RK3528 |
| 164 | Remove USB power-domains |

### DT-binding headers (`target/linux/rockchip/files-6.6/`)

| File | Description |
|------|-------------|
| `include/dt-bindings/clock/rockchip,rk3528-cru.h` | Clock IDs |
| `include/dt-bindings/reset/rockchip,rk3528-cru.h` | Reset IDs |
| `include/dt-bindings/power/rockchip,rk3528-power.h` | Power domain IDs |

### Image files (`target/linux/rockchip/image/`)

| File | Description |
|------|-------------|
| `armv8.mk` | Added `BOOT_SCRIPT := rock-2` to Rock 2A/2F |
| `rock-2.bootscript` | Custom boot script with ttyS0 console and WiFi GPIO power |

### Overlay files (`files/`)

| File | Description |
|------|-------------|
| `etc/init.d/morsechipreset` | GPIO reset + ordered driver loading |
| `etc/init.d/fix-wifi-path` | Dynamic USB path for AIC8800 |
| `etc/hotplug.d/net/99-batadv` | Auto-add wlan0 to bat0 |
| `etc/uci-defaults/99-openmanet` | First-boot wireless/network config |
| `etc/sysctl.d/99-disable-ipv6.conf` | Disable IPv6 (AIC8800 MLD crash fix) |
| `vendor/etc/firmware` | Symlink to AIC8800 firmware path |

### Packages

| Package | Location | Description |
|---------|----------|-------------|
| kmod-aic8800-usb | `package/kernel/aic8800-usb/` | WiFi driver |
| aic8800-firmware | `package/firmware/aic8800-firmware/` | WiFi firmware |

### Build config

| Setting | Value |
|---------|-------|
| `CONFIG_CLK_RK3528` | y |
| `CONFIG_PINCTRL_RK3528` | y |
| `CONFIG_PACKAGE_kmod-morse` | y |
| `CONFIG_PACKAGE_hostapd_s1g` | y |
| `CONFIG_PACKAGE_netifd-morse` | y |
| `CONFIG_PACKAGE_kmod-batman-adv` | y |
| `CONFIG_PACKAGE_batctl-full` | y |

---

## 17. Hardware Reference

### WiFi connection path
```
RK3528A USB20_HOST1 → FE1.1s Hub (U2002) → Port 4 → AIC8800D80 (U2202)
```

### WiFi power control
- `VCC_WIFI` (3.3V): GPIO4_A4 (gpio132) — power to AIC8800
- `WIFI_REG_ON_H`: GPIO1_A6 (gpio38) — chip enable/reset

### HaLow SPI connection
```
RK3528A SPI0 → 40-pin GPIO header → WM1302 HAT → mini-PCIe → Wio-WM6108 (FGH100M-H/MM6108)
```

### Serial console
- UART0 at `0xff9f0000`, 1500000 baud, 8N1
- Directly on board header (no USB-serial converter needed)
- **Conflict**: HAT's GPS module uses the same pins (8, 10) for UART; serial console unavailable when HAT is fully seated

---

## 18. Future Improvements

1. **Add `gpio-line-names` to RK3528 DTS** so the morse-feed's native `chipreset.sh` works without our custom script
2. **Fix AIC8800 driver atomic context bug** to allow IPv6
3. **Investigate AIC8800 latency spikes** — may be USB hub scheduling or driver power management
4. **Add support for `openmanetd`** — requires SSH keys for private git submodules during build
5. **Upstream RK3528 support** to OpenWrt mainline so future updates don't require manual backporting
6. **Test with multiple mesh nodes** to verify batman-adv routing over HaLow links
