# OpenMANET → Rock 2F: Gap Analysis & Porting Roadmap

## Current State Summary

**What you have working:**
- OpenWrt 24.10 booting on Rock 2F (RK3528 backported to kernel 6.6)
- MM6108 HaLow mesh via SPI (802.11s + SAE, 921 MHz, 27 dBm)
- AIC8800D80 WiFi 6 AP (management access, bridged to LAN)
- batman-adv on bat0 (BATMAN_IV, 10.41.0.0/16)
- Custom init scripts for driver ordering, GPIO reset, USB path fix
- All runtime configs applied manually post-flash

**What the original OpenMANET firmware has that you're missing:**

| Component | Original OpenMANET | Your Port | Gap |
|-----------|-------------------|-----------|-----|
| openmanetd | Go daemon, auto-everything | Not present | **Critical** |
| BATMAN version | BATMAN_V | BATMAN_IV | Medium |
| Alfred | Distributed data store | Not present | **Critical** (openmanetd needs it) |
| Bridge model | Single br-ahwlan (bat0+eth0+AP) | Separate br-lan + bat0 | Medium |
| Per-node DHCP | Each node serves 16 leases | Single DHCP on br-lan | Medium |
| LuCI web UI | Custom OpenMANET fork | Not present | Low priority |
| mDNS (avahi) | Hostname resolution across mesh | Not present | Low |
| GPSd | Location + range testing | Not present | Low |
| Protobuf node sync | Via Alfred | Not present | Comes with openmanetd |
| Build integration | Board config, zero post-flash | Manual post-flash steps | **Critical** |
| MediaMTX | Video streaming (optional pkg) | Not present | Optional |
| Tailscale | VPN tunnel (optional pkg) | Not present | Optional |

---

## Phase 1 — Add Rock 2F as an OpenMANET Board Target

**Goal:** Build from the OpenMANET firmware repo directly, producing a Rock 2F image with all packages baked in.

The OpenMANET firmware repo (`github.com/OpenMANET/firmware`) is a modified OpenWrt buildroot with a `boards/` directory and a setup script. You need to add a Rock 2F board definition.

### 1.1 Clone and understand the structure

```bash
git clone https://github.com/OpenMANET/firmware.git openmanet-firmware
cd openmanet-firmware
git checkout 24.10
```

Study the existing board definitions:
```bash
ls boards/
# You'll see directories like ekh-bcm2711 (RPi4 + Seeed SPI)
# Each board has a .config fragment and possibly overlay files
```

Look at one existing board to understand the pattern:
```bash
cat boards/ekh-bcm2711/.config
# This is a defconfig fragment selecting target, packages, and options
```

### 1.2 Create the Rock 2F board definition

```bash
mkdir -p boards/rock2f-spi
```

Create `boards/rock2f-spi/.config` — this should select:
- Target: `rockchip/armv8`
- Device: `radxa_rock-2f`
- All the OpenMANET packages (openmanetd, alfred, batman-adv, morse stack, LuCI, avahi, etc.)
- Your custom packages (kmod-aic8800-usb, aic8800-firmware)

**Key entries to include:**
```
CONFIG_TARGET_rockchip=y
CONFIG_TARGET_rockchip_armv8=y
CONFIG_TARGET_rockchip_armv8_DEVICE_radxa_rock-2f=y

# Morse HaLow stack
CONFIG_PACKAGE_kmod-morse=y
CONFIG_PACKAGE_dot11ah=y
CONFIG_PACKAGE_morse-fw-6108=y
CONFIG_PACKAGE_morse-bcf-info=y
CONFIG_PACKAGE_morse-board-config=y
CONFIG_PACKAGE_morse-regdb=y
CONFIG_PACKAGE_morsecli=y
CONFIG_PACKAGE_netifd-morse=y
CONFIG_PACKAGE_hostapd_s1g=y
CONFIG_PACKAGE_wpa_supplicant_s1g=y

# AIC8800 WiFi (your custom packages)
CONFIG_PACKAGE_kmod-aic8800-usb=y
CONFIG_PACKAGE_aic8800-firmware=y

# OpenMANET core
CONFIG_PACKAGE_openmanetd=y
CONFIG_PACKAGE_alfred=y
CONFIG_PACKAGE_batctl-full=y
CONFIG_PACKAGE_kmod-batman-adv=y

# Network services
CONFIG_PACKAGE_avahi-daemon=y
CONFIG_PACKAGE_avahi-utils=y
CONFIG_PACKAGE_dnsmasq-full=y

# LuCI
CONFIG_PACKAGE_luci=y
# (add any custom OpenMANET LuCI apps from the packages feed)

# GPS (if using the HAT GPS module)
CONFIG_PACKAGE_gpsd=y
CONFIG_PACKAGE_gpsd-clients=y

# Utilities
CONFIG_PACKAGE_curl=y
CONFIG_PACKAGE_jq=y
CONFIG_PACKAGE_tcpdump=y
```

### 1.3 Merge your BSP patches into the firmware repo

You need to bring over all your kernel/DTS work:

```
# Copy from your existing build tree into the OpenMANET firmware tree
target/linux/rockchip/patches-6.6/
  032-19, 032-20, 032-21, 032-22    (clock/reset/pinctrl)
  070-01 through 070-22              (RK3528 DTS)
  071, 072                           (Rock 2A/2F board DTS)
  073                                (USB + WiFi power + SPI0 + MM6108)
  160-01 through 160-04             (USB2 PHY)
  163-02                             (USB DTS nodes)
  164                                (USB power domain removal)

target/linux/rockchip/files-6.6/
  include/dt-bindings/clock/rockchip,rk3528-cru.h
  include/dt-bindings/reset/rockchip,rk3528-cru.h
  include/dt-bindings/power/rockchip,rk3528-power.h

target/linux/rockchip/config-6.6    (add CLK_RK3528 + PINCTRL_RK3528)
target/linux/rockchip/image/armv8.mk (Rock 2A/2F image defs)
target/linux/rockchip/image/rock-2.bootscript

package/kernel/aic8800-usb/
package/firmware/aic8800-firmware/
```

**Important:** The OpenMANET firmware repo may already have rockchip target support for other boards or its own kernel version. You'll need to check whether its kernel patches conflict with yours and merge carefully. If the OpenMANET 24.10 branch uses a different kernel patchlevel than yours (6.6.102 vs your 6.6.127), you may need to rebase.

### 1.4 Create the files overlay

Instead of applying post-flash manual steps, bake everything into the image via the `files/` directory. Create board-specific overlay files:

```
boards/rock2f-spi/files/
├── etc/
│   ├── init.d/
│   │   ├── morsechipreset          # Your GPIO reset + driver ordering script
│   │   └── fix-wifi-path           # AIC8800 USB path fix
│   ├── hotplug.d/
│   │   └── net/
│   │       └── 99-batadv           # Auto-attach wlan0 to bat0
│   ├── modules.d/
│   │   ├── morse.disabled          # Prevent kmodloader auto-load
│   │   └── aic8800-usb.disabled
│   ├── config/
│   │   ├── wireless                # Pre-configured radios
│   │   └── network                 # Pre-configured network (see Phase 3)
│   └── rc.local                    # BCF symlink fix if not handled elsewhere
└── lib/
    └── firmware/
        └── morse/
            └── bcf_default.bin → bcf_fgh100mhaamd.bin
```

### 1.5 Wire up the setup script

Study `scripts/openmanet_setup.sh` and add a case for your board:

```bash
# You want to be able to run:
./scripts/openmanet_setup.sh -i -b rock2f-spi
```

This typically copies `boards/rock2f-spi/.config` into `.config`, runs `make defconfig`, and may apply board-specific patches or feeds.

---

## Phase 2 — Get openmanetd Running

**This is the most important missing piece.** openmanetd is the daemon that makes OpenMANET self-configuring.

### What openmanetd does

1. **Auto IP allocation** — on fresh boot, queries the mesh via Alfred asking all nodes to announce their IP and DHCP range, then picks an unused static IP and DHCP range
2. **Gateway discovery** — gateways announce themselves; openmanetd picks the best one (matching batman-adv's gateway selection) and updates the default route
3. **Node announcements** — periodically broadcasts node info (IP, hostname, GPS, signal quality, battery) via Alfred using protobuf encoding
4. **Conflict detection** — if an IP conflict is detected, picks a new address and reboots

### Dependencies

openmanetd is written in Go and lives at `github.com/OpenMANET/openmanetd`. It requires:

- **Alfred** (`alfred` + `batadv-vis`) — distributed data store over batman-adv
- **Protobuf definitions** from `github.com/OpenMANET/protobufs`
- **batman-adv** with BATMAN_V (see Phase 3)
- **batctl-full** (not batctl-default)
- **Go toolchain** in the OpenWrt build system

### Building openmanetd

The OpenMANET packages feed (`github.com/OpenMANET/packages`, branch 24.10) already has an OpenWrt Makefile for openmanetd at `openmanetd/`. This should handle the Go cross-compilation.

```bash
# In your feeds.conf.default, make sure you have:
src-git openmanet https://github.com/OpenMANET/packages.git;24.10

# Then:
./scripts/feeds update openmanet
./scripts/feeds install -a -p openmanet
```

**The problem you hit before** was private git submodules. Check whether the 24.10 branch of the packages repo has resolved this — the openmanetd repo itself is now public (GPL-3.0, 23 stars). If submodules still fail, you can:

1. Fork the packages repo
2. Update the openmanetd Makefile to point to the public `github.com/OpenMANET/openmanetd` repo
3. Ensure the Go module dependencies resolve (the protobuf definitions at `github.com/OpenMANET/protobufs` are also public)

### If openmanetd won't compile

As a temporary workaround, you can replicate its core behavior with shell scripts:

```bash
#!/bin/sh
# /etc/init.d/openmanet-autoconf (simplified openmanetd replacement)

# 1. Generate a deterministic IP from the HaLow MAC address
mac=$(cat /sys/class/net/wlan0/address 2>/dev/null)
octet3=$(echo "$mac" | cut -d: -f5 | xargs printf "%d\n" 2>/dev/null || echo 254)
octet4=1

ip="10.41.${octet3}.${octet4}"
dhcp_start=2
dhcp_limit=16

# 2. Configure bat0 IP
uci set network.mesh.ipaddr="$ip"
uci set network.mesh.netmask="255.255.0.0"
uci commit network

# 3. Configure DHCP for local clients
uci set dhcp.mesh=dhcp
uci set dhcp.mesh.interface='mesh'
uci set dhcp.mesh.start="$dhcp_start"
uci set dhcp.mesh.limit="$dhcp_limit"
uci set dhcp.mesh.leasetime='12h'
uci commit dhcp

/etc/init.d/network reload
/etc/init.d/dnsmasq restart
```

This is far less sophisticated than openmanetd (no conflict detection, no gateway announcement, no protobuf sync) but gets you a working multi-node mesh immediately.

---

## Phase 3 — Match the Network Architecture

### 3.1 Switch to BATMAN_V

The original OpenMANET uses BATMAN_V for improved link quality monitoring and future bonding support.

```
# /etc/config/network
config interface 'bat0'
    option proto 'batadv'
    option routing_algo 'BATMAN_V'    # Changed from BATMAN_IV
    option gw_mode 'off'
    option orig_interval '5000'
```

**Note:** All nodes in the mesh must use the same BATMAN algorithm. You can't mix BATMAN_IV and BATMAN_V nodes.

### 3.2 Restructure the bridge

OpenMANET uses a single bridge (`br-ahwlan`) that contains bat0, eth0, and the WiFi AP interface. This creates a flat L2 domain — any client connected to any node (via WiFi, Ethernet, or mesh) sees the same subnet.

```
# /etc/config/network

# The main bridge — everything goes here
config device
    option name 'br-ahwlan'
    option type 'bridge'
    list ports 'eth0'
    # bat0 and phy1-ap0 are added dynamically

config interface 'ahwlan'
    option device 'br-ahwlan'
    option proto 'static'
    option ipaddr '10.41.254.1'     # Fresh install default
    option netmask '255.255.0.0'

# batman-adv mesh
config interface 'bat0'
    option proto 'batadv'
    option routing_algo 'BATMAN_V'
    option gw_mode 'off'

# HaLow mesh backhaul (wlan0 → bat0)
config interface 'mesh0'
    option proto 'batadv_hardif'
    option master 'bat0'
    option mtu '1532'
```

Update the hotplug script to add bat0 to the bridge:
```bash
# /etc/hotplug.d/net/99-batadv
[ "$ACTION" = "add" ] && [ "$INTERFACE" = "bat0" ] && {
    sleep 1
    brctl addif br-ahwlan bat0 2>/dev/null
    logger -t batadv "Added bat0 to br-ahwlan"
}
```

### 3.3 Per-node DHCP

Each OpenMANET node runs its own DHCP server handing out 16 addresses from its local range. This means clients always get an IP even when the mesh gateway is unreachable.

```
# /etc/config/dhcp
config dhcp 'ahwlan'
    option interface 'ahwlan'
    option start '2'
    option limit '16'
    option leasetime '12h'
```

### 3.4 WiFi AP on the bridge

The WiFi AP needs to be on `br-ahwlan` instead of `br-lan`:

```
# /etc/config/wireless (radio1 — AIC8800 AP)
config wifi-iface 'default_radio1'
    option device 'radio1'
    option mode 'ap'
    option ssid 'OpenMANET-AP'
    option encryption 'psk2'
    option key 'openmanet123'
    option network 'ahwlan'          # Changed from 'lan' to 'ahwlan'
```

---

## Phase 4 — Alfred + Protobuf Node Discovery

### 4.1 Install Alfred

Alfred is the backbone for inter-node data exchange. It's available in the OpenWrt routing feed:

```
CONFIG_PACKAGE_alfred=y
CONFIG_PACKAGE_batctl-full=y
```

Alfred runs on bat0 and lets nodes publish/retrieve arbitrary key-value data. openmanetd uses it to broadcast protobuf-encoded node info.

### 4.2 Alfred init config

```bash
# /etc/config/alfred
config alfred 'alfred'
    option interface 'bat0'
    option mode 'secondary'    # 'primary' on one node for sync
    option batmanif 'bat0'
```

### 4.3 Protobuf definitions

The OpenMANET protobuf definitions (`github.com/OpenMANET/protobufs`) define the message format for node announcements. openmanetd handles encoding/decoding — you don't need to implement this manually if openmanetd compiles.

---

## Phase 5 — Supporting Services

### 5.1 mDNS (avahi)

Enables `manet01.local` hostname resolution across the mesh:

```
CONFIG_PACKAGE_avahi-daemon=y
CONFIG_PACKAGE_avahi-dbus-daemon=n
```

Configure to reflect mDNS across bridge interfaces:
```
# /etc/avahi/avahi-daemon.conf
[server]
host-name=manet01
enable-reflector=yes

[reflector]
enable-reflector=yes
```

### 5.2 GPSd (optional, if using HAT GPS)

Your WM1302 HAT has a GPS module on pins 8/10 (UART2). If you isolate the conflicting pins and wire GPS separately:

```
CONFIG_PACKAGE_gpsd=y
CONFIG_PACKAGE_gpsd-clients=y
```

Configure to read from the UART:
```bash
# /etc/config/gpsd
config gpsd
    option device '/dev/ttyS2'    # UART2 on RK3528
    option port '2947'
    option enabled '1'
```

**Caveat:** This only works if you solve the HAT pin conflict from Section 2 of your docs (pins 8/10 GPS vs UART2). The jumper wire approach may be needed.

### 5.3 LuCI Web Interface

The OpenMANET project has a custom LuCI fork (`github.com/OpenMANET/luci`) and custom LuCI apps in the packages feed (`packages/luci/`). Add to your board config:

```
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-theme-bootstrap=y
# Add any OpenMANET-specific LuCI apps from the packages feed
```

---

## Phase 6 — Eliminate Post-Flash Manual Steps

**This is what makes it a real firmware image vs a DIY setup.**

Every manual step from Section 11.7 of your docs needs to be baked into the build. Here's the mapping:

| Manual Step | Build Integration |
|-------------|-------------------|
| Replace morsechipreset | Put in `boards/rock2f-spi/files/etc/init.d/morsechipreset` |
| Create fix-wifi-path | Put in `boards/rock2f-spi/files/etc/init.d/fix-wifi-path` |
| Create 99-batadv hotplug | Put in `boards/rock2f-spi/files/etc/hotplug.d/net/99-batadv` |
| Disable module auto-load | Ship `.disabled` files in overlay |
| Fix BCF symlink | Patch the morse-board-config package or add symlink in overlay |
| Configure wireless/network | Ship pre-configured UCI files in overlay |
| Enable init scripts | Add `ENABLE=1` or create symlinks in overlay |

For init script auto-enable, add symlinks in the overlay:
```bash
# In your board files overlay
mkdir -p boards/rock2f-spi/files/etc/rc.d
cd boards/rock2f-spi/files/etc/rc.d
ln -s ../init.d/morsechipreset S09morsechipreset
ln -s ../init.d/fix-wifi-path S18fix-wifi-path
```

---

## Recommended Execution Order

| Step | Task | Effort | Blocks |
|------|------|--------|--------|
| **1** | Fork OpenMANET/firmware, create `boards/rock2f-spi/` | 1 day | — |
| **2** | Merge BSP patches + AIC8800 packages into the fork | 1-2 days | Step 1 |
| **3** | Bake all runtime configs into files overlay | 1 day | Step 2 |
| **4** | First build + test (should match your current state, zero manual setup) | 1 day | Step 3 |
| **5** | Switch BATMAN_IV → BATMAN_V, restructure bridge to br-ahwlan | 1 day | Step 4 |
| **6** | Add Alfred, test data distribution between 2+ nodes | 1 day | Step 5 |
| **7** | Get openmanetd compiling (Go cross-compile for aarch64) | 1-3 days | Step 6 |
| **8** | Test openmanetd auto-addressing with 2+ nodes | 1 day | Step 7 |
| **9** | Add mDNS, LuCI, GPSd | 1 day | Step 5 |
| **10** | Full integration test, submit PR to OpenMANET upstream | 1 day | All |

**Total estimated effort: 10-14 days**

---

## Architecture Comparison

### Original OpenMANET (Raspberry Pi)

```
End User Device (tablet, ATAK, laptop)
    │ WiFi 5GHz / Ethernet
    ▼
┌─────────────────────────────────────┐
│  br-ahwlan (flat L2 bridge)        │
│  10.41.X.1/16 + DHCP (16 leases)  │
│                                     │
│  ┌──────────┐ ┌──────┐ ┌────────┐  │
│  │phy1-ap0  │ │ eth0 │ │  bat0  │  │
│  │(WiFi AP) │ │      │ │(BATMANv│) │
│  └──────────┘ └──────┘ └───┬────┘  │
│                             │       │
│  ┌──────────────────────────┘       │
│  │ wlan0 (HaLow 921MHz, SPI)      │
│  │ 802.11s mesh + SAE              │
│  └──────────────────────────────── │
│                                     │
│  openmanetd ←→ alfred ←→ bat0     │
│  (auto-IP, gateway, node sync)     │
│                                     │
│  avahi (mDNS)  gpsd  LuCI         │
└─────────────────────────────────────┘
    │ HaLow RF (920-921 MHz)
    ▼
Other OpenMANET Nodes
```

### Your Current Port (Rock 2F)

```
End User Device
    │ WiFi 5GHz
    ▼
┌────────────────────────────────────┐
│  br-lan (192.168.1.1/24)          │
│  ┌──────────┐                      │
│  │phy1-ap0  │ (AIC8800 WiFi AP)   │
│  └──────────┘                      │
│                                    │
│  bat0 (10.41.254.1/16) ← separate │
│  ┌─────────────────────┐           │
│  │ wlan0 (HaLow, SPI)  │          │
│  │ 802.11s + SAE        │          │
│  └─────────────────────┘           │
│                                    │
│  morsechipreset (init script)      │
│  fix-wifi-path (init script)       │
│  99-batadv (hotplug)               │
│  ❌ No openmanetd / alfred / mDNS │
└────────────────────────────────────┘
```

### Target State (Rock 2F, Full OpenMANET)

Same as original but on Rock 2F hardware, with AIC8800 instead of RPi onboard WiFi, and RK3528-specific boot/GPIO handling.

---

## Key Risks & Mitigations

**1. openmanetd Go compilation**
- Risk: Go cross-compilation for aarch64 on OpenWrt can be finicky
- Mitigation: The OpenMANET packages feed already has the Makefile; also, the packages repo includes a custom Go toolchain at `lang/golang/`

**2. Kernel version mismatch**
- Risk: OpenMANET 24.10 may pin a different kernel patchlevel than your backport
- Mitigation: Check `target/linux/rockchip/Makefile` in both trees; you may need to adjust `LINUX_VERSION`

**3. BATMAN_V compatibility**
- Risk: Switching from IV to V requires all nodes to match
- Mitigation: Test with 2 Rock 2F boards first before mixing with RPi nodes

**4. AIC8800 driver in OpenMANET tree**
- Risk: Your custom driver packages won't exist in upstream OpenMANET
- Mitigation: Keep them in your fork; they're board-specific and won't affect RPi builds

**5. HAT pin conflicts remain**
- Risk: Full-seated HAT still bootloops
- Mitigation: Physical pin isolation (tape on pins 3,5,7,8,10,27,28) or custom breakout board
