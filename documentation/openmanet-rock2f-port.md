# Porting OpenMANET to Radxa ROCK 2F — Complete Technical Documentation

## Table of Contents

1. Project Overview
2. Hardware Platform
3. Build Environment and Directory Structure
4. Phase 1 — OpenWrt 24.10 on RK3528
5. Phase 2 — AIC8800D80 Onboard WiFi Driver
6. Phase 3 — MM6108 HaLow SPI Integration
7. Phase 4 — OpenMANET Mesh Stack
8. Phase 5 — Runtime Configuration and Boot Automation
9. Complete File Inventory
10. Known Issues and Workarounds
11. Rebuilding from Scratch
12. Hardware Reference

---

## 1. Project Overview

### Goal

Port the OpenMANET firmware — originally designed for Raspberry Pi 4B — to the Radxa ROCK 2F single-board computer. The system must support:

- Morse Micro MM6108 HaLow (802.11ah) mesh networking at 920 MHz via SPI, using the Seeed Wio-WM6108 module on a WM1302 Pi HAT
- Batman-adv layer 2 mesh routing over the HaLow link
- Onboard AIC8800D80 WiFi 6 access point for management/client access
- Automatic boot with no manual intervention required

### Key Challenge

The RK3528 SoC was not supported in OpenWrt 24.10 or mainline Linux 6.6. All SoC support had to be backported from newer kernel versions (6.12–6.18). Additionally, the Seeed WM1302 HAT was designed for Raspberry Pi and required GPIO remapping, power management adjustments, and driver configuration changes to work on the Rock 2F.

### Software Versions

| Component | Version |
|-----------|---------|
| OpenWrt | 24.10-SNAPSHOT, r29143-214a657cfa |
| Linux kernel | 6.6.127 |
| Morse Micro driver | rel_1_16_4_2025_Sep_18 |
| Morse firmware | mm6108.bin |
| OpenMANET base | 24.10 branch |
| U-Boot | Rockchip vendor U-Boot (RK3528) |

---

## 2. Hardware Platform

### Radxa ROCK 2F

| Feature | Detail |
|---------|--------|
| SoC | Rockchip RK3528A |
| CPU | Quad-core ARM Cortex-A53 |
| RAM | LPDDR4 |
| Storage | eMMC + microSD |
| WiFi | AIC8800D80 (onboard, USB 2.0 via FE1.1s hub) |
| USB | 2× USB 2.0 Host, 1× USB 3.0 OTG (host mode) |
| GPIO | 40-pin header (Raspberry Pi compatible layout) |
| Serial Console | UART0 at 0xff9f0000, 1500000 baud, 8N1 |

### WiFi Connection Path

```
RK3528A USB20_HOST1 → FE1.1s Hub (U2002) → Port 4 (USB4_DP/USB4_DM) → AIC8800D80 (U2202)
```

WiFi power is controlled by two GPIOs:
- `VCC_WIFI` (3.3V) — GPIO4_A4 (gpio132) — regulator enable
- `WIFI_REG_ON_H` — GPIO1_A6 (gpio38) — chip enable/reset

### HaLow Module Stack

```
Seeed WM1302 Pi HAT → Seeed Wio-WM6108 mPCIe module → Quectel FGH100M-H → Morse Micro MM6108
```

The Wio-WM6108 module uses a mini-PCIe form factor inserted into the WM1302 HAT, which maps the signals to the 40-pin GPIO header. The module communicates via SPI.

### 40-Pin GPIO Mapping for MM6108

| Physical Pin | RK3528 GPIO | Function | Direction |
|-------------|-------------|----------|-----------|
| 19 | GPIO4_B2 (138) | SPI0_MOSI | Output |
| 21 | GPIO4_B3 (139) | SPI0_MISO | Input |
| 23 | GPIO4_B4 (140) | SPI0_CLK | Output |
| 24 | GPIO4_B6 (142) | SPI0_CSN0 | Output |
| 11 | GPIO4_B7 (143) | Reset (active low) | Output |
| 16 | GPIO4_B0 (136) | Wake | Output |
| 18 | GPIO4_B1 (137) | Busy | Input |
| 29 | GPIO4_B5 (141) | IRQ | Input |

### WM1302 HAT — Extra Pins That Cause Conflicts

The HAT connects additional devices beyond the MM6108 that interfere with Rock 2F boot:

| Pin(s) | HAT Device | Rock 2F Function | Problem |
|--------|-----------|-------------------|---------|
| 7 | GPS 1PPS output | UART1_TX_M0 | Pulses interfere with UART1 init |
| 8, 10 | GPS UART TX/RX | UART2_TX/RX_M0 | NMEA data corrupts UART2 |
| 3, 5 | ATECC608B I2C | I2C0_SDA/SCL | Auth chip pulls I2C bus |
| 27, 28 | HAT ID EEPROM | I2C1_SDA/SCL | Conflicts with onboard EEPROM at 0x50 |
| 12 | SX1302 power enable | GPIO1_B5 | HAT drives this for LoRa module power |
| 1, 17 | 3.3V power | 3.3V rail | Combined current draw causes brownout |

The brownout from combined 3.3V draw was confirmed as the primary cause of bootlooping when the HAT is fully seated. The MM6108 only needs the SPI pins (19,21,23,24) and GPIO control pins (11,16,18,29).

---

## 3. Build Environment and Directory Structure

### Build Host

Ubuntu 24.04, standard OpenWrt build dependencies installed.

### Working Tree

```
~/openwrt-rock2f/openwrt-24.10/          ← Main OpenWrt build tree
~/openwrt-rock2f/openwrt/                ← Reference tree (kernel 6.12, for backport patches)
~/openwrt-rock2f/openmanet-firmware/     ← OpenMANET firmware repo (feeds reference)
```

### Key Directories

```
target/linux/rockchip/
├── patches-6.6/              # Kernel patches (backports + custom)
├── files-6.6/                # Overlay files (dt-bindings headers)
├── config-6.6                # Kernel config additions
└── image/
    ├── armv8.mk              # Image definitions for Rock 2A/2F
    └── rock-2.bootscript     # Custom boot script

package/
├── kernel/aic8800-usb/       # Custom AIC8800 WiFi driver package
└── firmware/aic8800-firmware/ # Custom AIC8800 firmware package

feeds/
├── morse/                    # Morse Micro HaLow feed
└── openmanet/                # OpenMANET packages feed

files/
└── vendor/etc/firmware       # Symlink → /lib/firmware/aic8800_fw/USB/aic8800D80
```

---

## 4. Phase 1 — OpenWrt 24.10 on RK3528

### 4.1 Clock/Reset/Pinctrl Driver Backport

The RK3528 SoC had no clock controller, reset controller, or pinctrl driver in Linux 6.6. Without the clock driver, the kernel hangs silently at "Starting kernel..." because it cannot initialize any peripheral.

Patches copied from `~/openwrt-rock2f/openwrt/target/linux/rockchip/patches-6.12/` to `target/linux/rockchip/patches-6.6/`:

| Patch | Description | Notes |
|-------|-------------|-------|
| 032-19 | PLL FIXED_MODE flag | Prerequisite for RK3528 clock driver |
| 032-20 | RK3528 clock controller driver (clk-rk3528.c) | Most critical patch — without it nothing works |
| 032-21 | RK3528 reset lookup table (rst-rk3528.c) | Required manual fix: clk.h hunk context changed from rk3576 to rk3588 |
| 032-22 | RK3528 pinctrl support | Added to pinctrl-rockchip.c |

Patches NOT included (removed after testing):
- 032-26 (SD/SDIO tuning clocks) — requires GRF type members not in 6.6
- 032-27 (slab.h fix) — depends on 032-26

### 4.2 Kernel Config Additions

Added to `target/linux/rockchip/config-6.6`:

```
CONFIG_CLK_RK3528=y
CONFIG_PINCTRL_RK3528=y
```

### 4.3 DT-Binding Header Files

Added to `target/linux/rockchip/files-6.6/include/dt-bindings/`:

- `clock/rockchip,rk3528-cru.h` — Clock IDs
- `reset/rockchip,rk3528-cru.h` — Reset IDs
- `power/rockchip,rk3528-power.h` — Power domain IDs

These do not exist in upstream 6.6 but are required by rk3528.dtsi.

### 4.4 RK3528 Device Tree Patches

Patches 070-01 through 070-22 add the base device tree for the RK3528 SoC:

| Patch | Content |
|-------|---------|
| 070-01 | Base DT (rk3528.dtsi): CPU, interrupt controller, timer, CRU |
| 070-02 | Clock generators |
| 070-03 | UART clocks |
| 070-04 | Pinctrl and GPIO nodes |
| 070-05 | QoS register node |
| 070-06 | SCMI clock support |
| 070-07 | SARADC node |
| 070-09 | UART3 interrupt fix |
| 070-10 | DMA controller |
| 070-12 | I2C controllers |
| 070-13 | PWM nodes |
| 070-14 | SDMMC/SDIO controllers |
| 070-15 | GMAC nodes |
| 070-16 | Move pinctrl node outside soc |
| 070-17 | SPI nodes |
| 070-18 | Power controller |
| 070-19 | GPU node |
| 070-22 | Convert power domains to SCMI |

### 4.5 Rock 2A/2F Board Support

| Patch | Description |
|-------|-------------|
| 071 | DTS Makefile — adds Rock 2A and 2F to the build |
| 072 | Board DTS files — rk3528-rock-2a.dts, rk3528-rock-2f.dts, rk3528-rock-2.dtsi (common) |

### 4.6 USB PHY Driver Backport

The RK3528 USB2 PHY driver did not exist in Linux 6.6. Without it, USB host ports and onboard WiFi cannot function.

| Patch | Description |
|-------|-------------|
| 160-01 | Simplify rockchip-usbgrf handling |
| 160-02 | Add clkout_ctl_phy support |
| 160-03 | Add support for RK3528 |
| 160-04 (local, two parts) | clkout helper and rk3528 compat |

All five patches must be enabled together — 160-04 alone causes `rk3528_usb2phy_tuning defined but not used` build error.

### 4.7 USB DTS Nodes

| Patch | Content |
|-------|---------|
| 163-02 | Adds USB controller/PHY nodes to rk3528.dtsi: usb_host0_xhci (DWC3), usb_host0_ehci, usb_host0_ohci, usb2phy. All added with status="disabled". |

### 4.8 USB Power Domain Removal

| Patch | Description |
|-------|-------------|
| 164 | Removes power-domains property from all USB nodes (xhci, ehci, ohci, usb2phy). The RK3528 power domain driver is not fully functional in this backport — enabling the power domain DTS caused "deferred probe timeout" on all devices. Safe because power domains default to ON at boot. |

### 4.9 Board DTS Modifications (Patch 073)

The 073 patch enables hardware on the Rock 2 board and adds MM6108 SPI support. Current content:

**WiFi power regulator fix:**
```dts
&vcc_wifi {
    regulator-always-on;
    regulator-boot-on;
};
```
This ensures the WiFi regulator stays on during boot — without it, the AIC8800 powers off during kernel init.

**USB enables:**
```dts
&usb2phy { status = "okay"; };
&usb2phy_host { phy-supply = <&vcc5v0_usb20>; status = "okay"; };
&usb2phy_otg { status = "okay"; };
&usb_host0_ehci { status = "okay"; };
&usb_host0_ohci { status = "okay"; };
&usb_host0_xhci { dr_mode = "host"; status = "okay"; };
```

**SPI0 with MM6108:**
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

Key details:
- `spi-max-frequency = <10000000>` — 10 MHz. 50 MHz caused SPI protocol errors with jumper wires and the full HAT.
- `reset-gpios` uses `GPIO_ACTIVE_LOW` — the MM6108 resets when the pin is LOW and runs when HIGH. This was discovered through debugging: the original `GPIO_ACTIVE_HIGH` setting kept the chip in reset.

**GPIO pinctrl for MM6108:**
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

### 4.10 Boot Script

File: `target/linux/rockchip/image/rock-2.bootscript`

```
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

This script:
1. Powers on the WiFi chip via direct GPIO register writes in U-Boot (before the kernel has GPIO drivers)
2. Sets `earlycon` for immediate serial output during kernel init
3. Uses PARTUUID for root device to handle varying MMC numbering

Referenced in `target/linux/rockchip/image/armv8.mk` via `BOOT_SCRIPT := rock-2` for both Rock 2A and Rock 2F image definitions.

### 4.11 Image Definitions

Added to `target/linux/rockchip/image/armv8.mk`:
- Rock 2A definition with `BOOT_SCRIPT := rock-2`
- Rock 2F definition with `BOOT_SCRIPT := rock-2`

---

## 5. Phase 2 — AIC8800D80 Onboard WiFi Driver

### 5.1 Driver Package

Location: `package/kernel/aic8800-usb/`

The AIC8800D80 WiFi 6 chip is connected via the onboard USB hub. The driver required backport header fixes to compile against the mac80211 backports in OpenWrt 24.10.

### 5.2 Firmware Package

Location: `package/firmware/aic8800-firmware/`

Contains the firmware binaries for the AIC8800D80.

### 5.3 Firmware Path Symlink

Location: `files/vendor/etc/firmware`

This is a symlink pointing to `/lib/firmware/aic8800_fw/USB/aic8800D80`. The AIC8800 driver looks for firmware in `/vendor/etc/firmware/` by default.

### 5.4 Known AIC8800 Issues

**USB bus number changes between boots:** The AIC8800 disconnects and reconnects during firmware upload, causing it to enumerate on different USB buses (usb3 vs usb4). This breaks the OpenWrt wireless `path` config which uses the full USB path. Solved with the `fix-wifi-path` init script (see Section 8.3).

**Scheduling-while-atomic crash with batman-adv:** When the AIC8800 WiFi interface has batman-adv mesh proxy enabled, the driver's `rwnx_send_mesh_proxy_add_req` function is called from an atomic context (IPv6 MLD workqueue) and tries to sleep, causing a kernel BUG. Solved by ensuring batman-adv only runs on the morse HaLow interface, never on the AIC8800.

**Periodic ~500ms latency spikes:** The AIC8800 WiFi AP exhibits periodic latency spikes of 500-700ms. Power save parameters (`ps_on`, `dpsm`, `ap_uapsd_on`) are read-only at runtime and must be set at modprobe time, but the driver firmware appears to override some of them. This remains a known issue; reducing WiFi mode complexity (disabling HE/WiFi 6) may help.

---

## 6. Phase 3 — MM6108 HaLow SPI Integration

### 6.1 OpenMANET Feeds

Added to `feeds.conf.default`:
- `morse-feed` — contains kmod-morse SPI driver, hostapd_s1g, wpa_supplicant_s1g, netifd-morse, morse firmware, BCF files
- `openmanet` — contains openmanetd (could not compile due to private submodules), batman-adv, mesh config packages

### 6.2 Morse Micro Packages Installed

| Package | Description |
|---------|-------------|
| kmod-morse | SPI driver for MM6108 (kernel module) |
| dot11ah | 802.11ah shim layer for mac80211 |
| morse-fw-6108 | Firmware binary (mm6108.bin) |
| morse-fw-6108-tlm | Telemetry firmware |
| morse-bcf-info | BCF information utility |
| morse-board-config | Board configuration hotplug |
| morse-board-config-hotplug-model | Model-based BCF selection |
| morse-regdb | Regulatory database for S1G |
| morsecli (morse_cli) | Command-line management tool |
| netifd-morse | netifd wireless backend (morse.sh) |
| hostapd_s1g | 802.11ah hostapd |
| wpa_supplicant_s1g | 802.11ah wpa_supplicant for mesh |

### 6.3 Board Configuration File (BCF)

**Critical discovery:** The correct BCF for the Seeed Wio-WM6108 (Quectel FGH100M-H) is `bcf_fgh100mhaamd.bin`, NOT `bcf_mf08651_us.bin` (which is for the MF08651 reference board) and NOT `bcf_default.bin` (which symlinks to `bcf_failsafe.bin`).

The failsafe BCF has the power amplifier disabled, resulting in TX power of 0 dBm — the radio firmware reports transmission but no actual RF output is produced (invisible on SDR).

The correct BCF (`bcf_fgh100mhaamd.bin`) enables 27 dBm TX power, matching what OpenMANET achieves on the Raspberry Pi.

**Fix applied on the board:**
```bash
rm /lib/firmware/morse/bcf_default.bin
ln -s /lib/firmware/morse/bcf_fgh100mhaamd.bin /lib/firmware/morse/bcf_default.bin
```

### 6.4 Reset GPIO Polarity

The MM6108 reset pin behavior:
- LOW = chip in reset
- HIGH = chip running

The DTS originally had `GPIO_ACTIVE_HIGH` which caused the driver to assert HIGH for reset, keeping the chip running, and LOW for "active" which put it in reset — exactly backwards. Changed to `GPIO_ACTIVE_LOW` in the 073 patch.

Even with the correct polarity in the DTS, the `chipreset.sh` script from the morse-feed cannot find the reset GPIO because `gpio-line-names` is not defined in the device tree for GPIO4. The workaround is a custom init script that manually exports and toggles GPIO 143 (see Section 8.1).

### 6.5 S1G Channel Mapping

The dot11ah kernel module maps 802.11ah S1G sub-1GHz channels to equivalent 5GHz HT channels because mac80211 doesn't natively understand sub-1GHz frequencies. This means `iwinfo` and `iw` show the interface on 5 GHz (e.g., channel 149 / 5745 MHz) even though the actual RF is on 920-921 MHz.

The real frequency is confirmed via `morse_cli -i wlan0 channel` which shows the actual operating frequency.

| UCI Channel | S1G Center Freq | Bandwidth | mac80211 Mapped Channel |
|------------|-----------------|-----------|------------------------|
| 37 | 920.5 MHz | 1 MHz | 149 (5745 MHz) |
| 38 | 921.0 MHz | 2 MHz | 149 (5745 MHz) |

---

## 7. Phase 4 — OpenMANET Mesh Stack

### 7.1 Mesh Configuration

Since `openmanetd` could not compile (private git submodules), the mesh was configured manually via UCI:

**HaLow Mesh (radio2):**
- Mode: 802.11s mesh point
- Mesh ID: OpenMANET
- Encryption: SAE (WPA3)
- Key: openmanet123
- Channel: 38 (921 MHz, 2 MHz bandwidth)
- Country: US
- Network: bat0 (batman-adv)
- Mesh forwarding: disabled (batman-adv handles routing)

**WiFi AP (radio1):**
- Mode: AP (access point)
- SSID: OpenMANET-AP
- Encryption: WPA2-PSK
- Key: openmanet123
- Network: lan (br-lan bridge)

### 7.2 Batman-adv Configuration

```
network.bat0.proto='batadv'
network.bat0.routing_algo='BATMAN_IV'
network.bat0.gw_mode='off'
```

The bat0 interface gets a static IP via a separate network interface:
```
network.mesh.proto='static'
network.mesh.device='bat0'
network.mesh.ipaddr='10.41.254.1'
network.mesh.netmask='255.255.0.0'
```

The `wlan0` (HaLow mesh) interface is added to bat0 as a hard interface. Due to netifd not reliably attaching it automatically, a hotplug script handles this (see Section 8.4).

### 7.3 Network Architecture

```
End User Device
    │ (WiFi 5GHz, WPA2)
    ▼
[OpenMANET-AP] phy1-ap0 ──── br-lan (192.168.1.1/24)
    │
[batman-adv] bat0 ──── 10.41.254.1/16
    │
[HaLow Mesh] wlan0 ──── 921 MHz, 27 dBm
    │ (802.11s + SAE, via wpa_supplicant_s1g)
    ▼
Other OpenMANET Nodes
```

---

## 8. Phase 5 — Runtime Configuration and Boot Automation

### 8.1 Main Init Script: `/etc/init.d/morsechipreset`

```bash
#!/bin/sh /etc/rc.common
START=09

start() {
    # 1. Reset MM6108
    logger -t morsechipreset "Resetting MM6108..."
    echo 143 > /sys/class/gpio/export 2>/dev/null
    echo out > /sys/class/gpio/gpio143/direction 2>/dev/null
    echo 0 > /sys/class/gpio/gpio143/value
    sleep 1
    echo 1 > /sys/class/gpio/gpio143/value
    sleep 1

    # 2. Load morse FIRST so it gets wlan0
    logger -t morsechipreset "Loading morse driver..."
    modprobe dot11ah
    modprobe morse country=US bcf=bcf_fgh100mhaamd.bin spi_clock_speed=10000000 enable_ps=0 enable_dynamic_ps_offload=0
    sleep 2

    # 3. Load AIC8800 SECOND
    logger -t morsechipreset "Loading AIC8800 driver..."
    modprobe aic_load_fw
    modprobe aic8800_fdrv ps_on=N dpsm=N ap_uapsd_on=N
    sleep 3

    logger -t morsechipreset "All drivers loaded"
}
```

**Why START=09:** This runs before kmodloader (which processes `/etc/modules.d/` at around boot stage 20). The script must run first to reset the MM6108 chip and control driver load order.

**Why morse loads first:** The first wireless driver to load gets `wlan0`. If AIC8800 loads first and gets `wlan0`, netifd applies the mesh config to it instead of the morse device, causing the AIC driver to crash with `MESH_START_REQ` errors (the AIC8800 doesn't support mesh mode).

**Module parameters explained:**
- `bcf=bcf_fgh100mhaamd.bin` — correct BCF for Quectel FGH100M-H
- `spi_clock_speed=10000000` — 10 MHz SPI clock (50 MHz causes protocol errors)
- `enable_ps=0` — disable power save (reduces latency)
- `enable_dynamic_ps_offload=0` — disable dynamic power save offload

### 8.2 Disabled Module Auto-Loading

Both the morse and AIC8800 modules are disabled from kmodloader auto-loading because the init script handles loading in the correct order:

```
/etc/modules.d/morse.disabled
/etc/modules.d/zz-morse.disabled
/etc/modules.d/aic8800-usb.disabled
```

The original `/etc/modules.d/morse` file (now disabled) contains:
```
morse reattach_hw=0 bcf=bcf_fgh100mhaamd.bin country=US enable_sgi_rc=1 macaddr_suffix=
```

### 8.3 WiFi Path Fix Script: `/etc/init.d/fix-wifi-path`

```bash
#!/bin/sh /etc/rc.common
START=18

start() {
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
```

**Why this is needed:** The AIC8800 disconnects and reconnects during firmware upload, causing it to enumerate on different USB buses (usb3 or usb4) on each boot. OpenWrt's wireless config uses the full USB device path to identify radios, so a hardcoded path breaks when the bus number changes.

### 8.4 Batman-adv Hotplug Script: `/etc/hotplug.d/net/99-batadv`

```bash
[ "$ACTION" = "add" ] && [ "$INTERFACE" = "wlan0" ] && {
    sleep 2
    ip link set wlan0 mtu 1532 2>/dev/null
    batctl meshif bat0 if add wlan0 2>/dev/null
    logger -t batadv "Added wlan0 to bat0 with MTU 1532"
}
```

**Why this is needed:** The netifd batadv integration doesn't reliably attach wlan0 to bat0 automatically. This hotplug script fires when wlan0 appears, sets the MTU to 1532 (batman-adv overhead requires >1500), and adds it as a hard interface.

### 8.5 BCF Symlink

```
/lib/firmware/morse/bcf_default.bin → /lib/firmware/morse/bcf_fgh100mhaamd.bin
```

This ensures the correct BCF is loaded regardless of how the driver determines the BCF file name. The original symlink pointed to `bcf_failsafe.bin` which has the PA disabled.

---

## 9. Complete File Inventory

### Build Host — Modified/Created Files

#### Kernel Patches (`target/linux/rockchip/patches-6.6/`)

| Patch | Description |
|-------|-------------|
| 032-19 | PLL FIXED_MODE flag |
| 032-20 | RK3528 clock controller driver |
| 032-21 | RK3528 reset lookup table (manually fixed) |
| 032-22 | RK3528 pinctrl support |
| 070-01 through 070-22 | RK3528 DTS nodes |
| 071 | Rock 2A/2F DTS Makefile |
| 072 | Rock 2A/2F board DTS |
| 073 | USB + WiFi power + SPI0 + MM6108 + pinctrl |
| 160-01 through 160-04 | USB2 PHY driver |
| 163-02 | USB DTS nodes for RK3528 |
| 164 | Remove USB power-domains |

#### DT-Binding Headers (`target/linux/rockchip/files-6.6/`)

| File | Description |
|------|-------------|
| include/dt-bindings/clock/rockchip,rk3528-cru.h | Clock IDs |
| include/dt-bindings/reset/rockchip,rk3528-cru.h | Reset IDs |
| include/dt-bindings/power/rockchip,rk3528-power.h | Power domain IDs |

#### Image Files (`target/linux/rockchip/image/`)

| File | Description |
|------|-------------|
| armv8.mk | Added Rock 2A/2F image definitions with BOOT_SCRIPT |
| rock-2.bootscript | Custom boot script with WiFi GPIO init and serial console |

#### Kernel Config (`target/linux/rockchip/config-6.6`)

| Option | Value |
|--------|-------|
| CONFIG_CLK_RK3528 | y |
| CONFIG_PINCTRL_RK3528 | y |

#### Packages

| Package | Location |
|---------|----------|
| kmod-aic8800-usb | package/kernel/aic8800-usb/ |
| aic8800-firmware | package/firmware/aic8800-firmware/ |

#### Firmware Overlay

| Path | Type |
|------|------|
| files/vendor/etc/firmware | Symlink → /lib/firmware/aic8800_fw/USB/aic8800D80 |

### Target Board — Runtime Modifications

| File | Purpose |
|------|---------|
| /etc/init.d/morsechipreset | GPIO reset + driver load ordering (START=09) |
| /etc/init.d/fix-wifi-path | Dynamic AIC8800 USB path fix (START=18) |
| /etc/hotplug.d/net/99-batadv | Auto-attach wlan0 to bat0 with MTU 1532 |
| /lib/firmware/morse/bcf_default.bin | Symlink → bcf_fgh100mhaamd.bin |
| /etc/modules.d/morse.disabled | Disabled (init script handles loading) |
| /etc/modules.d/zz-morse.disabled | Disabled (init script handles loading) |
| /etc/modules.d/aic8800-usb.disabled | Disabled (init script handles loading) |
| /etc/config/wireless | Radio0-4 config (see Section 7.1) |
| /etc/config/network | LAN + bat0 + mesh interfaces |

---

## 10. Known Issues and Workarounds

### 10.1 AIC8800 Latency Spikes

The onboard AIC8800D80 WiFi AP exhibits periodic ~500ms latency spikes. Root cause appears to be firmware-level power management that cannot be fully disabled via driver parameters. The `ps_on=N` and `ap_uapsd_on=N` modprobe parameters are set but the driver firmware may override them.

Potential mitigation: reduce WiFi mode complexity by switching from HE80 (WiFi 6) to VHT80 (WiFi 5/AC).

### 10.2 Boot Requires GPIO Reset Script

The morse `chipreset.sh` script fails at boot because `gpio-line-names` is not defined for GPIO4 in the device tree. The workaround is the custom `morsechipreset` init script. A proper fix would add `gpio-line-names` to the GPIO4 node in the DTS so the upstream script works.

### 10.3 Driver Load Order is Critical

If the AIC8800 driver loads before the morse driver, the AIC gets `wlan0` and netifd applies the mesh config to it, causing a kernel crash. The init script controls this, but it requires disabling auto-loading via modules.d.

### 10.4 USB Path Changes Between Boots

The AIC8800 enumerates on usb3 or usb4 randomly due to the firmware upload disconnect/reconnect cycle. The `fix-wifi-path` script handles this dynamically.

### 10.5 Batman-adv MTU Warning

Batman-adv requires MTU > 1500 to avoid L2 fragmentation. The hotplug script sets wlan0 MTU to 1532. The warning in dmesg is non-critical — batman-adv will fragment if needed.

### 10.6 Duplicate Wireless Radios

The system generates radio0 (dot11ah mac80211 shim for SPI), radio1 (AIC8800), radio2 (morse type), and sometimes radio3/radio4 (AIC8800 duplicates from re-enumeration). Only radio1 (AP) and radio2 (mesh) should be enabled. All others must be disabled.

---

## 11. Rebuilding from Scratch

### 11.1 Full Clean Rebuild

```bash
cd ~/openwrt-rock2f/openwrt-24.10
make target/linux/clean
make -j$(nproc) V=s
```

### 11.2 Kernel Only

```bash
make target/linux/clean
make target/linux/compile -j1 V=s
```

### 11.3 AIC Driver Only

```bash
make package/kernel/aic8800-usb/clean V=s
make package/kernel/aic8800-usb/compile -j1 V=s
```

### 11.4 Find Output Images

```bash
ls -lh bin/targets/rockchip/armv8/openwrt-*rock-2f*
```

### 11.5 Flash to SD Card

```bash
dd if=bin/targets/rockchip/armv8/openwrt-rockchip-armv8-radxa_rock-2f-squashfs-sysupgrade.img of=/dev/sdX bs=4M status=progress
```

### 11.6 Manual U-Boot Boot (if boot.scr fails)

```
load mmc 1:1 0x02000000 /kernel.img
setenv bootargs earlycon=uart8250,mmio32,0xff9f0000 console=ttyS0,1500000n8 root=/dev/mmcblk1p2 rootfstype=squashfs rootwait
bootm 0x02000000#config-1
```

### 11.7 Post-Flash Setup

After flashing a new image, the following runtime modifications must be reapplied:

1. Replace `/etc/init.d/morsechipreset` with the version from Section 8.1
2. Create `/etc/init.d/fix-wifi-path` from Section 8.3
3. Create `/etc/hotplug.d/net/99-batadv` from Section 8.4
4. Disable auto-loading: `mv /etc/modules.d/morse /etc/modules.d/morse.disabled` and same for aic8800-usb
5. Fix BCF symlink: `rm /lib/firmware/morse/bcf_default.bin && ln -s bcf_fgh100mhaamd.bin /lib/firmware/morse/bcf_default.bin`
6. Configure wireless and network via UCI (see Section 7)
7. Enable init scripts: `/etc/init.d/morsechipreset enable && /etc/init.d/fix-wifi-path enable`

---

## 12. Hardware Reference

### Serial Console

- UART0 at 0xff9f0000
- Baud rate: 1500000
- Settings: 8N1
- Available on the board header (no USB-serial converter needed)
- Note: When the WM1302 HAT is fully seated, the serial pins are occupied. Bending the HAT pins or using jumper wires allows simultaneous HAT and serial access.

### GPIO Calculation

For Rockchip GPIO numbering: `GPIO_NUMBER = BANK * 32 + GROUP * 8 + BIT`

Where GROUP: A=0, B=1, C=2, D=3

Example: GPIO4_B7 = 4*32 + 1*8 + 7 = 143

### WiFi Power Control (U-Boot)

The boot script powers on WiFi via direct register writes because the kernel GPIO driver isn't loaded yet:

```
# GPIO4_A4 (VCC_WIFI enable) — GPIO4 data register low
mw.l 0xffb20000 0x00100010   # bit 4 set + write mask

# GPIO1_A6 (WIFI_REG_ON) — GPIO1 data register low  
mw.l 0xffaf0000 0x00400040   # bit 6 set + write mask
```

The Rockchip v2 GPIO write convention uses bits 16-31 as a write mask and bits 0-15 as data.
