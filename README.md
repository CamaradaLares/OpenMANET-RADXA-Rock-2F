# OpenMANET Firmware

A MANET (Mobile Ad-Hoc Network) is a self-forming wireless mesh where each node connects directly
without centralized infrastructure. This technology is especially useful for search and rescue,
disaster response, field operations, and any disconnected communications scenario. Designed to be
budget-friendly with excellent long-range performance using Wi-Fi HaLow (802.11ah) at sub-1 GHz
frequencies. The build integrates with ATAK over multicast but works equally well over standard IP
links.

---

## Supported Hardware

### Single Board Computers

| Device            | Status         | Onboard WiFi        | Notes                                              |
|-------------------|----------------|---------------------|----------------------------------------------------|
| Raspberry Pi 4    | ✅ Stable      | ✅ Working (SPI)    | Reference platform. Onboard WiFi in AP mode only   |
| Raspberry Pi CM4  | ✅ Stable      | ✅ Working (SPI)    | Onboard WiFi in AP mode only                       |
| Raspberry Pi 3B   | ✅ Stable      | ✅ Working (SPI)    | Onboard WiFi in AP mode only                       |
| Raspberry Pi 2W   | ✅ Stable      | ✅ Working (SPI)    | Onboard WiFi in AP mode only                       |
| **Radxa Rock 2F** | ⚠️ Beta        | ✅ Working (USB)    | AIC8800D80 onboard WiFi. See Rock 2F section below |

### HaLow Radio Modules

| Device              | Status     | Interface | Chipset | Notes                                  |
|---------------------|------------|-----------|---------|----------------------------------------|
| Wio-WM6108 + WM1302 | ✅ Tested  | SPI       | MM6108  | Best performing HaLow option currently |
| Silex SX-SDMAH      | ✅ Tested  | SDIO      | MM6108  | Low dBm, high noise floor              |
| Alfa AHPI6108E      | ✅ Tested  | SDIO      | MM6108  | Decent performance                     |
| TBD                 | ✅ Tested  | USB       | MM8108  | Great performance                      |

---

## Building OpenMANET Firmware

### System Requirements

Ubuntu 24.04 or later. Install build dependencies:

```bash
sudo apt update
sudo apt install build-essential clang flex g++ gawk gcc-multilib g++-multilib git gettext \
  libncurses5-dev libssl-dev python3-setuptools rsync unzip golang-go zlib1g-dev swig file wget \
  libnl-3-dev libnl-genl-3-dev libgps-dev libcap-dev pkg-config libopus-dev \
  libopusfile-dev portaudio19-dev net-tools libpcre3-dev libpcre3 upx-ucl
```

### Building for Raspberry Pi (reference platform)

```bash
./scripts/openmanet_setup.sh -i -b ekh-bcm2711
make download
make -j$(nproc)
```

Output image: `bin/targets/bcm27xx/bcm2711/`

### Building for Radxa Rock 2F

```bash
./scripts/openmanet_setup.sh -i -b rock2f-spi
make download
make -j$(nproc)
```

Output image: `bin/targets/rockchip/armv8/openmanet-*-radxa_rock-2f-squashfs-sysupgrade.img.gz`

For verbose output:
```bash
make -j$(nproc) V=s 2>&1 | tee build.log
```

---

## Radxa Rock 2F — Port Documentation

### Hardware

The Rock 2F is a compact SBC based on the Rockchip RK3528A quad-core Cortex-A53 SoC. It uses the
same 40-pin GPIO header as Raspberry Pi, which allows the Seeed WM1302 HaLow HAT to fit directly.

| Component       | Detail                                                               |
|-----------------|----------------------------------------------------------------------|
| SoC             | Rockchip RK3528A (quad-core Cortex-A53)                              |
| HaLow module    | Seeed Wio-WM6108 on WM1302 Pi HAT — connected via SPI0               |
| Onboard WiFi    | AIC8800D80 — USB 2.0 via internal FE1.1s USB hub                     |
| Serial console  | UART0 at `0xff9f0000`, 1500000 baud 8N1 (pins 8/10 on header)        |
| Storage         | Boot from microSD                                                    |

### Flashing

Flash the image to a microSD card (replace `/dev/sdX` with your card device):

```bash
gunzip -c openmanet-*-radxa_rock-2f-squashfs-sysupgrade.img.gz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
```

Insert the card into the Rock 2F and power on. The system configures itself on first boot.

### Default Access

| Service     | Value                       |
|-------------|-----------------------------|
| WiFi AP     | SSID: `OpenMANET-AP` / Password: `openmanet123` |
| Mesh        | SSID: `OpenMANET` / SAE key: `openmanet123` / 921 MHz |
| Node IP     | `10.41.254.1`               |
| Web UI      | `http://10.41.254.1`        |
| SSH         | `ssh root@10.41.254.1` (no password by default) |

### HaLow Transmission Proof

The image below shows SDR++ receiving HaLow beacons from the Rock 2F + Seeed WM1302 HAT at
921 MHz — confirming that the radio is transmitting at full power with the correct BCF loaded.

![SDR++ spectrum showing HaLow beacons at 921 MHz](documentation/sdrpp-halow-beacons-921mhz.png)

> The Rock 2F transmits at **27 dBm** using the Quectel FGH100M-H front-end with the
> `bcf_fgh100mhaamd.bin` board configuration file. Beacons are visible at 920–921 MHz.

---

## ⚠️ Rock 2F — Known Issues and Caveats

### System stability — multiple reboots may be required

> **The Rock 2F port is currently in beta.** On first boot after flashing, the system may need
> **2 to 3 reboots** before all services stabilize. This is caused by a combination of:
> - The AIC8800 USB WiFi chip re-enumerating on a different USB bus number after firmware upload,
>   requiring the `fix-wifi-path` init script to update the stored path and restart wireless.
> - Driver load ordering between the morse HaLow driver and AIC8800 — if the order is wrong on
>   first boot, netifd may apply the mesh config to the wrong interface.
> - The batman-adv `bat0` interface requiring the HaLow mesh to be fully up before joining the
>   bridge.
>
> **After 2–3 reboots the system stabilizes and operates reliably.**

### WM1302 HAT — Bootloop with full seating

The WM1302 HAT was designed for Raspberry Pi. When fully seated on the Rock 2F 40-pin header it
causes a **3.3V brownout** at boot due to the combined current draw of the MM6108, GPS module
(L76KB), ATECC608B crypto chip, and HAT voltage regulator exceeding what the Rock 2F can supply
during boot.

**Additional conflicting pins:**

| Pin(s) | HAT signal | Rock 2F conflict |
|--------|-----------|-----------------|
| 8, 10  | GPS UART TX/RX | UART2 — NMEA data corrupts serial during boot |
| 7      | GPS 1PPS       | UART1_TX — pulses interfere with UART1 init  |
| 3, 5   | ATECC608B I2C  | I2C0 — crypto chip pulls the bus             |
| 27, 28 | HAT ID EEPROM  | I2C1 — conflicts with onboard EEPROM at 0x50 |

**Workaround:** Place tape or insulating material over pins 3, 5, 7, 8, 10, 27, 28 before seating
the HAT. This exposes only the SPI and MM6108 GPIO control pins that the HaLow driver needs. An
external 3.3V supply for the HAT also resolves the brownout.

### IPv6 disabled system-wide

The AIC8800D80 driver crashes in atomic context when IPv6 MLD multicast packets trigger its mesh
proxy code. IPv6 is disabled at the kernel level (`sysctl`) as a workaround. This affects all
interfaces, not just WiFi.

### HaLow frequency shown as 5 GHz in system tools

`iwinfo` and `iw` report the HaLow interface as operating on 5 GHz (e.g. channel 149 / 5745 MHz).
This is expected — the `dot11ah` shim maps 802.11ah sub-1 GHz channels to 5 GHz equivalents for
mac80211 compatibility. The actual RF frequency is 921 MHz. Use `morse_cli -i wlan0 channel` to
confirm the real operating frequency.

### BCF must be set correctly

The morse-feed ships `bcf_default.bin` pointing to `bcf_failsafe.bin` — a configuration with the
power amplifier disabled (0 dBm, no RF output). The Rock 2F port automatically symlinks
`bcf_default.bin` to `bcf_fgh100mhaamd.bin` on first boot, which is the correct BCF for the
Quectel FGH100M-H module in the Wio-WM6108 (27 dBm, PA enabled). If the symlink is missing, the
radio will appear to work but produce no detectable RF signal.

---

## Rock 2F — Porting Challenges

Porting OpenMANET to the Rock 2F required solving several non-trivial problems:

### 1. RK3528A had zero support in Linux 6.6

The Rockchip RK3528A SoC was not supported in OpenWrt 24.10 or mainline Linux 6.6. Clock,
reset, pinctrl, and USB PHY drivers had to be backported from Linux 6.12–6.18 kernel patches.
Without the clock driver the kernel hangs silently after "Starting kernel..." with no output.

### 2. Morse SPI driver did not compile on kernel 6.6

The morse driver's `spi.c` references the macro `SPI_CONTROLLER_ENABLE_CS_GPIOD`, which was
added to the Linux SPI core in kernel 6.9. On kernel 6.6 the driver emits a `#warning` that
`-Werror=cpp` promotes to a build error. Fixed by defining the macro at compile time in the
morse driver Makefile (`NOSTDINC_FLAGS`).

### 3. SPI was never enabled in the build

`CONFIG_MORSE_SPI=y` was missing from the board configuration. The upstream
`common_extras/spi_diffconfig` that provides this flag also enables BCM/Raspberry Pi-specific
SPI kernel modules that must not be enabled on Rockchip. The flag was added directly to the
`rock2f-spi` board `target_diffconfig`.

### 4. MM6108 reset GPIO polarity was inverted

The initial DTS had `GPIO_ACTIVE_HIGH` on `reset-gpios`. This kept the MM6108 in permanent
reset — SPI probe returned errno -5 and MISO read all `0xFFFFFFFF`. The correct polarity is
`GPIO_ACTIVE_LOW` (LOW = reset, HIGH = run).

### 5. USB power domain caused deferred probe deadlock

Enabling the RK3528 power domain controller in the device tree caused every device on the
system to fail with "deferred probe timeout". The USB nodes were referencing power domains that
the incomplete backport could not satisfy. Resolved by removing `power-domains` from all USB
DTS nodes — safe because power domains default to ON at SoC reset.

### 6. AIC8800 USB path changes between reboots

The AIC8800D80 disconnects and re-enumerates during firmware upload, changing its USB bus number
(`usb3` → `usb4`). OpenWrt stores the full USB path in the wireless config, causing the AP radio
to become unrecognized after reboot. Solved with a dedicated init script that detects the current
USB path from sysfs and updates UCI if it changed.

### 7. Driver load order caused kernel crash

If the AIC8800 driver loads before the morse HaLow driver, it claims `wlan0`. When netifd then
applies the 802.11s mesh config to `wlan0`, the AIC8800 receives a `MESH_START_REQ` it does not
support, triggering a `scheduling while atomic` kernel BUG. The morse driver must always load
first. OpenWrt's `kmodloader` was bypassed with a custom `START=09` init script.

### 8. chipreset.sh incompatible with RK3528

The morse-feed's `chipreset.sh` script uses `gpiofind MM_RESET` to locate the reset GPIO by
name, requiring `gpio-line-names` in the device tree. This is not implemented for RK3528 GPIO4.
Additionally, the script unbinds the MMC host controller for SDIO reset — catastrophic on the
Rock 2F where the same controller manages eMMC. Replaced with a custom init script using direct
sysfs GPIO control.

---

## Roadmap

- [ ] **Fix first-boot stability** — eliminate the 2–3 reboot requirement. Root cause is a race
  between the AIC8800 USB re-enumeration, driver load ordering, and netifd interface
  initialization. Target: single clean boot with all services up.
- [ ] Add `gpio-line-names` to RK3528 GPIO4 device tree so the upstream morse `chipreset.sh`
  works natively without the custom sysfs workaround.
- [ ] Fix AIC8800 driver IPv6 atomic context crash to re-enable IPv6.
- [ ] Compile and integrate `openmanetd` — the Go daemon for automatic mesh IP addressing,
  gateway discovery, and Alfred node sync. Currently blocked by Go build dependencies.
- [ ] Switch from BATMAN_IV to BATMAN_V for improved link quality metrics and gateway selection.
- [ ] Investigate and fix AIC8800 WiFi AP periodic ~500ms latency spikes.
- [ ] Add GPS integration using the WM1302 HAT's onboard L76KB module (requires HAT pin
  isolation to avoid UART conflicts).
- [ ] Upstream RK3528A BSP patches to OpenWrt mainline.
- [ ] Test multi-node mesh with 2+ Rock 2F boards — validate batman-adv routing and throughput.

---

## Contributing

To contribute a custom package for OpenMANET, open a pull request in the
[OpenMANET Packages Repository](https://github.com/OpenMANET/packages).

For Rock 2F-specific issues, please open an issue in this repository with the output of
`logread` and `dmesg` from the affected boot.
