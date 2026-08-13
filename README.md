> 🇫🇷 **Note :** Une traduction française complète est disponible en bas de ce document.

---

# D-Link DGS-1210-48 (Hardware Revision D1) — Reverse Engineering

> Full reverse engineering of the D-Link DGS-1210-48 Gigabit Ethernet Switch (hardware revision D1), including flash dump, firmware extraction, hardware identification, and OpenWrt portability assessment.

**Status:** Research in progress — contributions welcome  
**Author:** Clément  
**License:** CC BY-SA 4.0

---

## Table of Contents

- [Project Goals](#project-goals)
- [Hardware](#hardware)
- [Firmware](#firmware)
- [Boot Process](#boot-process)
- [Flash Layout](#flash-layout)
- [Tools & Methods](#tools--methods)
- [Findings](#findings)
- [OpenWrt Portability](#openwrt-portability)
- [How to Contribute](#how-to-contribute)
- [French Translation](#traduction-française)

---

## Project Goals

1. Document the hardware architecture of the DGS-1210-48 D1 for the community
2. Assess OpenWrt portability
3. Fix known bugs in the stock firmware (management IP instability)
4. Provide a solid base for anyone wanting to work on this device

---

## Hardware

### Component List

| Component | Identification | Confidence | Method |
|-----------|---------------|------------|--------|
| **CPU** | MIPS 24Kc V8.5, codename "Music" (confirmed booting at ~266 BogoMIPS) | ✅ Confirmed (live boot) | Custom-built kernel booted to a live shell — `/proc/cpuinfo` reports `system type: Atheros Music`, `cpu model: MIPS 24Kc V8.5`. Commercial chip identity still unconfirmed, see [Open Questions](#open-questions) |
| **Switch fabric revision** | "Vivo" B0 | ✅ Confirmed (live register read) | Direct read of `REG_SWITCH_GLB_CTRL_0` (`0x18800000`) from a custom debug kernel returned `0x02022002`; masked with `0xff` per GPL source (`SWITCH_GLB_CTRL_0_REV_ID_MASK`) matches `MUSIC_VIVO_REV_ID_B0` (`0x2`) exactly |
| **Switch fabric (master)** | Likely QCA8519-AC2C, codename "Vivo" (48-port GbE class) | 🟢 Strong candidate | `product_id` strings `"UM8719A"`/`"UM8719B"` in GPL source name the *board*, not necessarily the chip; MikroTik CRS226/CRS210 product pages list QCA8519-AC2C alongside an on-board CPU, matching this board's combined CPU+switch memory layout. See [Open Questions](#open-questions) |
| **Board ODM** | Alpha Networks Incorporation | ✅ Confirmed | Copyright header in GPL source (`ver_uboot.h`) |
| **Flash** | Macronix MX25L12835FMI-10G — SPI NOR, 128 Mbit / 16 MB | ✅ Confirmed (chip marking) | PCB inspection ("Mxix" marking) + `bdinfo` + binwalk |
| **RAM** | Nanya NT5CB128M8FN-DH — DDR3-1600 SDRAM, 1 Gbit / 128 MB | ✅ Confirmed (chip marking) | PCB inspection + `bdinfo` U-Boot |
| **I2C mux** | NXP PCA9545A (4-channel) | ✅ Confirmed (chip marking) | PCB inspection + symbol `DRV_CH_Select_PCA9545` in `dgs_drv.ko` |
| **GPIO expander** | NXP PCA9555 ×2 (16-bit I2C I/O expander) | ✅ Confirmed (chip marking) | PCB inspection + symbol `DRV_IF_Get_PCA9555_LED_MODE` in `dgs_drv.ko` |
| **CPLD** | Unknown model | ✅ Confirmed (present) | Symbols `DRV_CPLD_*` in `dgs_drv.ko` |
| **EEPROM** | Atmel/Microchip AT24C series (I2C serial EEPROM) | ✅ Confirmed (chip marking) | PCB inspection ("ATMLH548" marking) + symbol `DRV_EEPROM_*` in `dgs_drv.ko` |
| **RTC** | Unknown model (I2C) | ✅ Confirmed (present) | Symbol `DRV_RTC_Init` in `dgs_drv.ko` |
| **Temp. sensor** | Microchip TCN75AVOA (I2C digital temperature sensor) | ✅ Confirmed (chip marking) | PCB inspection + symbols `DRV_THM_*` in `dgs_drv.ko` |
| **Ports** | 48× GbE RJ45 (1-48) + 4× SFP (49-52) | ✅ Confirmed | Stock config |
| **Fan control** | PWM output via MOSFET or TI-style LED/power driver IC (not a dedicated fan controller chip) | 🟡 Probable | Symbol `DRV_FAN_*` in `dgs_drv.ko` exposes get/set status only — consistent with simple PWM duty-cycle control rather than a smart fan controller with its own protocol |
| **Watchdog** | Integrated | ✅ Confirmed | Symbol `DRV_WD_*` in `dgs_drv.ko` |
| **Monostable** | 74HC123D ×4 (dual retriggerable monostable multivibrator) | ✅ Confirmed (chip marking) | PCB inspection |
| **Logic gate** | 74HC08D (quad 2-input AND gate) | ✅ Confirmed (chip marking) | PCB inspection |
| **UART** | ttyS0, 115200 baud, 3.3V | ✅ Confirmed | Direct access |

### Power Management

Identified via PCB chip marking inspection:

| Component | Role |
|-----------|------|
| Richtek RT8120B | Synchronous PWM buck controller — drives MOSFETs to step down and regulate a DC rail |
| ITE IT76820M | DC-DC converter / power management IC — supplies a regulated voltage rail to the board |
| UTC 2SB772L | PNP power transistor (-30V, -3A) — intermediate-power load switching or regulation |
| Niko-Sem/UBIQ QM3004D (marked "M3004D") | N-channel power MOSFET (30V, high current) — switching stage in the board's voltage converters |



```
┌─────────────────────────────────────────────────────┐
│              AMD Alchemy Au1210 (CPU)                │
│         MIPS 4Kc · Big Endian · ~400MHz             │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ NOR Flash│  │ DDR3 RAM │  │   UART ttyS0     │  │
│  │   16MB   │  │  DDR     │  │   115200 baud    │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │              MDIO Bus                        │   │
│  │    ┌────────────────────────────────────┐   │   │
│  │    │  QCA8519/QCA8719 (Switch Fabric)   │   │   │
│  │    │  48× GbE (ports 1-48) + 4× SFP (ports 49-52) │   │   │
│  │    └────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │              I2C Bus                         │   │
│  │  ┌──────────┐                               │   │
│  │  │ PCA9545A │ (4-channel I2C mux)           │   │
│  │  │  (mux)   │                               │   │
│  │  └────┬─────┘                               │   │
│  │  ┌────┴──────────────┐                      │   │
│  │  │ PCA9555 ×2        │ (GPIO expanders)     │   │
│  │  │ CPLD              │ (LEDs, FAN, power)   │   │
│  │  │ EEPROM            │                      │   │
│  │  │ RTC               │                      │   │
│  │  │ Temp. sensor      │                      │   │
│  │  └───────────────────┘                      │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

> **Note on I2C:** The PCA9545 mux was not detected by U-Boot `iprobe` because the mux is not initialized at U-Boot stage. The downstream devices (PCA9555, CPLD, EEPROM, RTC) are only accessible once the mux is configured by `dgs_drv.ko`.

---

## Firmware

### Software Stack

| Component | Value |
|-----------|-------|
| U-Boot | 1.1.4 (Jan 29 2013) |
| Kernel | Linux 2.6.31 (compiled Jan 22 2016) |
| BSP | Atheros "Music" ASDK 0.9.7.253 |
| Compiler | GCC 4.3.3 |
| Rootfs | JFFS2 (big endian) |
| Init system | BusyBox + custom D-Link init scripts |

### Kernel Modules (proprietary)

| Module | Role |
|--------|------|
| `dgs_xal.ko` | Hardware Abstraction Layer (XAL) — char device major 60 |
| `dgs_drv.ko` | Main hardware driver (I2C, GPIO, CPLD, EEPROM, FAN, SFP...) |
| `sdk_um_uk_if.ko` | Atheros SDK userspace/kernelspace interface — MDIO access to switch fabric |

### Startup Sequence

```
U-Boot 1.1.4
    └── bootm 0xb9080000
        └── Linux 2.6.31
            └── /etc/rc.d/rcS
                ├── mount -a
                ├── ifconfig eth0 up
                ├── /etc/rc.d/rc.network  (udhcpc or static IP)
                ├── telnetd
                └── /native/app_fs/autorun.sh
                    ├── S10dgs_xal  → insmod dgs_xal.ko
                    ├── S20dgs_drv  → insmod dgs_drv.ko
                    ├── S30dgs_sdk  → insmod sdk_um_uk_if.ko
                    └── S40dgs_app  → extract & run DGS12XX binary
```

---

## Boot Process

### UART Access

- **Connector:** Internal header on PCB
- **Voltage:** 3.3V (do NOT use 5V adapters)
- **Settings:** 115200 baud, 8N1, no flow control
- **Prompt:** `music>`

### U-Boot Environment

```
ipaddr=192.168.1.10
serverip=192.168.1.27
bootcmd=bootm 0xb9080000
bootdelay=4
oem_vendor=QCA
asdk_ver=0.9.7.253
```

### Default Credentials

| Service | Username | Password |
|---------|----------|----------|
| Web UI / CLI | `admin` | `admin` |
| Telnet | `admin` | `admin` |

---

## Flash Layout

Confirmed at boot from kernel MTD output:

```
0x000000000000-0x000000040000 : "u-boot"       (256 KB)
0x000000040000-0x000000080000 : "u-boot-env"   (256 KB)
0x000000080000-0x000000280000 : "uImage"       (2 MB)
0x000000280000-0x000001000000 : "roofs"        (13.5 MB)
```

> **Note:** The partition name `roofs` (instead of `rootfs`) is a typo in the original D-Link/Atheros BSP source code. It has been preserved as-is in this documentation.

### Flash Dump

A full 16MB flash dump was obtained via U-Boot `md.b` command captured over UART at 230400 baud, then reconstructed using a custom Python script.

```bash
# In U-Boot
md.b 0xB9000000 0x1000000

# On PC (after UART capture)
python3 uboot_mdb_to_bin.py capture.txt flash_dump.bin \
  --start-addr b9000000 --skip-errors --verbose
```

The reconstruction script is available in [`tools/uboot_mdb_to_bin.py`](tools/uboot_mdb_to_bin.py).

---

## GPL Source Code

D-Link publishes GPL-licensed source code for the DGS-1210 series. The exact package matching this device's firmware (v4.10.023, hardware revision D1) was located in D-Link's public S3 bucket:

```
https://dlink-gpl.s3.amazonaws.com/GPL1300053/DGS-1210-48_D1_GPL%20source%20code_for_FW_v4.10.023.tar.gz
```

> Note: the archive is bzip2-compressed despite the `.tar.gz` extension. Rename to `.tar.bz2` before extracting, or use `tar -xjf`.

### What's included

- Full Linux 2.6.31 kernel source (`linux/linux-2.6.31.tar.gz`)
- Atheros Switch SDK (ASDK) patched sources, branch `asdk_branch_rc_patch_0.9.7.253`, under `pkgs/sdk/`
- Low-level platform drivers for the "Music" CPU core: MDIO, GMAC/PHY, SPI, I2C, GPIO, UART, watchdog, interrupt controller, flash/OEM address tables — all with full `.c`/`.h` source
- Board-specific config for `b2b_test` (matches `oem_product=b2b_test` seen in this device's U-Boot environment)
- The `qca-music_toolchain.tar.bz2` cross-compilation toolchain (CentOS 6.3 32-bit host required per D-Link's build instructions)

### What's NOT included

- **U-Boot source is absent.** Only a version header (`inc/ver_uboot.h`, copyright Alpha Networks Incorporation, 2012) is present — no bootloader source tree. D-Link may be in violation of GPL obligations for the U-Boot patches visible in this firmware (custom `dltftp` command, MDIO-aware boot code, etc.), since U-Boot itself is GPL-licensed.
- **`dgs_drv.ko` and `dgs_xal.ko` sources are absent.** Only precompiled `.ko` binaries are shipped (`pkgs/drv/app_fs/bin/dgs_drv.ko`, `pkgs/xal/app_fs/bin/dgs_xal.ko`). These modules likely link against the GPL kernel but their own source was not published — this is the same limitation Ghidra-based reverse engineering was already working around.

### Key findings from the GPL source

**MDIO transaction layer** (`driver/cpusub/music/mdio/music_mdio_base.c`) — confirms the exact protocol used to talk to the switch fabric:
```c
sa_i32_t music_mdiom_switch_write(sa_u32_t addr, sa_u32_t data);
sa_i32_t music_mdiom_switch_read(sa_u32_t addr, sa_u32_t *data);
sa_i32_t music_mdiom_soc_write(sa_u32_t addr, sa_u32_t data);
sa_i32_t music_mdiom_soc_read(sa_u32_t addr, sa_u32_t *data);
```
MDIO register base: `0x180a0000`. Transactions are routed to either the external switch fabric (`MDIOM_CTRL_DEV_SEL_SWITCH`) or the SoC's own internal PHY, selected via a control register bit.

**Memory-mapped switch core** (`driver/flashoem/music_flash_oem.c`) — the switch-master chip's registers are also directly memory-mapped, not solely accessible via MDIO:

```
Switch Core:   0x18800000, 8 MB
Capwap:        0x18140000, 256 KB
CPU Register:  0x18000000, 1152 KB
SPI:           0x1f000000, 16 MB
```

This board's address table is `address_table_b2b_test`, confirming `oem_product=b2b_test` from U-Boot maps directly to this hardware profile in the SDK.

**Existing upstream Linux support for the QCA8xxx family** — while QCA8519/QCA8719 itself has no known open source driver, the Linux kernel does have a mainline DSA driver (`drivers/net/dsa/qca8k.c`) for the closely related `qca8327`, `qca8334`, and `qca8337` switch chips. Given they share the same vendor and switch fabric generation, this driver is a plausible starting point for reverse-engineering or adapting a QCA8519/QCA8719 driver, rather than building one from scratch.



| Step | Tool | Notes |
|------|------|-------|
| UART capture | minicom 230400 baud | Log to file with Ctrl-A L |
| Flash dump reconstruction | Custom Python script | See `tools/` |
| Firmware analysis | binwalk 3.x | |
| Rootfs extraction | jefferson 0.4.7 | JFFS2 extractor |
| Binary analysis | Ghidra (Java 21) | MIPS 32-bit big endian |
| Symbol extraction | `nm`, `strings`, `objdump` | |

---

## Findings

### Known Bug — Management IP Instability

The switch occasionally stops responding on its management IP while continuing to forward traffic normally. Root cause analysis:

- `udhcpc` is launched in background (`-b` flag) with no restart mechanism
- If the DHCP lease expires or `udhcpc` crashes, the management IP is lost
- The `DGS12XX` application manages the IP stack internally — a crash or memory leak causes the management interface to become unresponsive

**Workaround:** Configure a static IP via CLI:
```
config ipif_cfg System ipaddress 192.168.X.X/24 vlan default
config ipif System dhcp disable
save
```

### I2C Bus Architecture

The PCA9545 mux was not visible from U-Boot `iprobe` (all addresses return FAIL). This is because U-Boot does not initialize the mux. All downstream I2C devices are only accessible after `dgs_drv.ko` is loaded and configures the mux channels.

### MDIO Switch Access

The switch fabric (QCA851x) is accessed via MDIO through three kernel functions:
- `music_mdio_switch_read` / `music_mdio_switch_write` — main switch access
- `music_mdio_capwap_read` / `music_mdio_capwap_write` — capwap-specific access
- `music_mdiom_soc_read` / `music_mdiom_soc_write` — SoC internal MDIO

These are exported kernel symbols called by `sdk_um_uk_if.ko`.

---

## Open Questions

- **Is the switch-master chip a QCA8519 or QCA8719?** GPL source (`music_flash_oem.c`) contains board `product_id` strings `"UM8719A"`/`"UM8719B"`, and the factory `DGS12XX` binary's detection list includes both. A later cross-reference tips the balance toward **QCA8519-AC2C**: MikroTik's own product pages for the CRS226-24G-2S+IN and CRS210-8G-2S+IN list `Switch chip model: QCA8519-AC2C` alongside a "400MHz CPU" spec on the same device — suggesting QCA8519-AC2C may be a combined switch+CPU part rather than a pure switch fabric IC, which would explain why "Music" (CPU) and "Vivo" (switch core) share the same base memory region on this board. Not yet settled with full certainty — see next point.
- **Is "Music" a real, physically distinct MIPS core, or is it integrated into the same silicon as the switch-master chip?** Live register read from `0x18800000` (`REG_SWITCH_GLB_CTRL_0`) returned `0x02022002`; masked with `0xff` per GPL source this equals `MUSIC_VIVO_REV_ID_B0` (`0x2`), confirming the "Vivo" codename and revision B0 with certainty — but this alone doesn't prove whether "Music" (CPU) and "Vivo" (switch core) are the same physical die or two adjacent chips. The heatsink covering this chip has not yet been removed (epoxy-glued, high risk); a MikroTik CRS210/CRS226 teardown photo showing the QCA8519-AC2C package for visual comparison would help settle this without further risk to this board.
- **Why does the PRId table match AMD Alchemy Au1210?** Most likely explanation: the "Music" core was designed to expose a MIPS 4Kc-compatible PRId for toolchain/kernel compatibility — though a live-booted custom kernel now reports the real core as `MIPS 24Kc V8.5` via `/proc/cpuinfo`, so the PRId table match was likely coincidental rather than deliberate spoofing.
- **Does Alpha Networks Incorporation have public documentation for the "Music"/"Vivo" platform?** They are the ODM that physically designed this board (confirmed via GPL source copyright headers). No public datasheet has been located yet.

---

## Custom Kernel Build & Live Boot — SUCCESS

**Status: a self-compiled Linux 2.6.31 kernel, built from the official GPL sources with the official `qca-music_toolchain`, boots this exact device end-to-end and reaches a live root shell.**

This is the single most important result of this project: proof that the toolchain, kernel source, and hardware understanding documented here are all correct and sufficient to produce working, booting firmware for this device.

### Build Environment

- Host: x86_64 Linux
- Toolchain: `qca-music_toolchain` (from GPL package), `mips-linux-uclibc-gcc 4.3.3`, must be symlinked/copied to `/opt/qca-music_toolchain` (hardcoded path in `makefile.compile`)
- Kernel source: `linux-2.6.31.tar.gz` from `dgs-1210-4.10.023_48_GPL` (firmware v4.10.023, D1)
- Board config: `b2b_test_defconfig` (matches `oem_product=b2b_test` from U-Boot env)

### Problems Encountered & Fixed

1. **`kernel/timeconst.pl` fails on modern Perl** — `Can't use 'defined(@array)'`. Fix: `sed -i 's/!defined(@val)/!@val/' kernel/timeconst.pl`. Must be patched *inside* `linux-2.6.31.tar.gz` itself, since the top-level Makefile does `rm -rf linux-2.6.31; tar -xzf linux-2.6.31.tar.gz` on every build, silently discarding changes made only to the extracted directory.

2. **`readelf`/locale bug in the vendor Makefile** — the official `linux/makefile` pipes `readelf -a | grep "Entry"`, but on a system with a non-English locale, `readelf`'s output is translated (e.g. French: "Adresse du point d'entrée") and the grep matches nothing, silently producing an empty `-e` argument for `mkimage` (`invalid entry point -n`). Fix: `LC_ALL=C readelf -h vmlinux | grep "Entry point address"`, or generate the uImage manually instead of relying on the vendor Makefile's own `mkimage` invocation.

3. **`inflate() returned -3` — Uncompressing Kernel Image fails in U-Boot.** This was **not** a gzip/zlib version incompatibility (multiple gzip versions were tried, including compiling with the same 2007-era `gzip 1.3.12` found in a CentOS 6 container, which reproduced factory-identical compressed size but still failed). The real cause: **the compressed uImage was being loaded via `tftpboot` to the same RAM address (`0x80002000`) that the header specifies as the *decompression* `Load Address`.** As U-Boot decompresses, the growing output overwrites the tail of the still-unread compressed input, corrupting the gzip stream mid-decompression. **Fix: load the compressed image to a separate scratch address with no overlap** (e.g. `tftpboot 0x81000000 <file>` instead of `tftpboot 0x80002000 <file>`), leaving the `Load Address`/`Entry Point` in the image header unchanged. `bootm 0x81000000` then correctly decompresses from `0x81000000` into `0x80002000` without self-corruption.

4. **Kernel panics before any console output (`Control returned to monitor - resetting...`)** — diagnosed by adding raw UART debug probes (bypassing `printk`, which isn't available until console init) directly in `arch/mips/kernel/head.S` at `kernel_entry`, before and after the CPU-specific `kernel_entry_setup` macro. Root cause turned out to be identical to #3 (memory overlap) — once fixed, boot proceeded normally and the probes confirmed `prom_init()` was reached correctly all along.

5. **`VFS: Unable to mount root fs`** when testing with a simplified `bootargs` — omitting `mtdparts=...` from the custom kernel command line means the kernel never creates the MTD partitions, so `root=31:03` points at nothing. Fix: always keep the full `mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs)` string alongside any other custom bootargs.

6. **BusyBox 1.19.4 (matching `BUSYBOX := busybox-1.19.4` in `makefile.sdk`) fails to compile `miscutils/ubi_tools.c`** against this toolchain's kernel headers (`mtd/ubi-user.h` missing, several `UBI_*` symbols undeclared). Not needed for a debug shell — disable in `.config`: `CONFIG_UBIATTACH`, `CONFIG_UBIDETACH`, `CONFIG_UBIMKVOL`, `CONFIG_UBIRMVOL`, `CONFIG_UBIRSVOL`, `CONFIG_UBIUPDATEVOL`.

### Working Boot Procedure

```
# In U-Boot, override init to bypass the (symbol-incompatible) proprietary
# DGS12XX application and drop straight to a shell, using the REAL stock
# rootfs already present in flash:

setenv bootargs console=ttyS0,115200 root=31:03 rootfstype=jffs2 init=/bin/sh mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs) mem=128M
port enable
tftpboot 0x81000000 <custom-uImage>
bootm 0x81000000
```

This mounts the actual factory JFFS2 rootfs read-only from flash (`root=31:03`) and drops into a live BusyBox/ash shell as root — no password needed, no flash writes involved, fully reversible with a power cycle (the custom `bootargs` only exists in RAM for that boot; `saveenv` was never called).

> Note: the stock `dgs_xal.ko`/`dgs_drv.ko` kernel modules fail to load against this self-built kernel (`unknown symbol alpha_inter_ver`) because they were compiled against the exact module version CRCs of the factory kernel build. This is expected and harmless for debug/shell access — building a matching kernel `.config` bit-for-bit identical to the factory one (or rebuilding these modules from a full source tree, if one can be obtained) would be required to get the stock `DGS12XX` application running normally under a custom kernel.

### Confirmed via Live Shell

```
/proc/cpuinfo:
  system type       : Atheros Music
  cpu model         : MIPS 24Kc V8.5
  BogoMIPS          : 266.24

/proc/mtd:
  mtd0: 00040000 00001000 "u-boot"
  mtd1: 00040000 00001000 "u-boot-env"
  mtd2: 00200000 00001000 "uImage"
  mtd3: 00d80000 00001000 "roofs"

Switch fabric register (0x18800000): 0x02022002
  → masked & 0xff = 0x02 = MUSIC_VIVO_REV_ID_B0 (confirmed via GPL source)
```



### Assessment

| Item | Status | Notes |
|------|--------|-------|
| "Music" CPU core support in mainline kernel | ❌ None | Not a recognized upstream MIPS platform; GPL source only covers kernel 2.6.31 |
| U-Boot source | ❌ Not published by D-Link | Only a version header is included in the GPL package — likely a GPL compliance gap |
| QCA8519/QCA8719 switch fabric driver | ❌ None (chip-specific) | 🟡 But mainline `drivers/net/dsa/qca8k.c` supports the related qca8327/qca8334/qca8337 family — plausible adaptation base |
| MDIO transaction protocol | ✅ Fully documented | Complete C source in GPL package (`music_mdio_base.c`) |
| Memory-mapped switch register space | ✅ Documented | `0x18800000`, 8 MB — see GPL source `music_flash_oem.c` |
| PCA9545/PCA9555 drivers | ✅ Mainline | Standard Linux I2C drivers |
| NOR flash driver | ✅ Mainline | Standard MTD, memory-mapped |
| Custom-built kernel boots on real hardware | ✅ **Proven** | Self-compiled 2.6.31 kernel reaches a live root shell — see [Custom Kernel Build & Live Boot](#custom-kernel-build--live-boot--success) |

### What Needs to be Done

1. **Backport the "Music" CPU platform code to a modern kernel** — the 2.6.31 GPL source (`kernel/arch/setup.c`, `platform.c`, and the full `driver/cpusub/music/` tree) gives a complete, now boot-*proven* reference for UART, interrupt controller, GMAC, MDIO, SPI, and I2C init. This is a porting/adaptation task, not blind reverse engineering — and the fact that the 2.6.31 code boots real hardware end-to-end validates that the reference is accurate.
2. **Adapt or reverse-engineer a QCA8519/QCA8719 DSA driver** — start from mainline `qca8k.c` (qca8327/8334/8337 family) and cross-reference against the GPL `music_mdiom_switch_read/write` calls and the memory-mapped switch-core register space to identify chip-specific differences. The switch-core register base is confirmed live (`0x18800000` reads `0x02022002`, decoding to revision "Vivo" B0).
3. **Create a board DTS** for the DGS-1210-48 D1, using the confirmed flash layout and the `address_table_b2b_test` memory map from the GPL source.
4. **Handle peripheral drivers** — PCA9545, PCA9555, CPLD, RTC (mostly available upstream).
5. **Rebuild `dgs_xal.ko`/`dgs_drv.ko` from source, or reimplement their functionality**, since the precompiled factory `.ko` files only load against the exact kernel build they were compiled for (confirmed: `unknown symbol alpha_inter_ver` when loaded against a self-built kernel of the same source version).
6. **U-Boot: no existing base to restore.** Since D-Link did not publish U-Boot sources and no upstream MIPS "Music" platform exists, options are: (a) request U-Boot GPL sources directly from D-Link/Alpha Networks, (b) keep using the stock U-Boot 1.1.4 purely as a RAM-loader via `tftpboot`+`bootm` (proven reliable throughout this project) and never touch flash, or (c) write a minimal U-Boot port from scratch using the GPL kernel source as a hardware-init reference.

### Realistic Timeline

- Linux minimal boot (no switch, RAM-only via stock U-Boot): **done** — see [Custom Kernel Build & Live Boot](#custom-kernel-build--live-boot--success).
- Full OpenWrt with all 48 ports: still likely 4-8 months — shorter than originally estimated since the MDIO protocol is now fully documented from source *and* proven working on real hardware, but the QCA8519/QCA8719 switch driver and CPU platform port to a modern kernel both still need to be written.

---

## How to Contribute

This project needs help with:

- **Hardware:** CPLD and RTC exact model still unidentified (heatsink-covered CPU/switch chips also still unconfirmed by physical decap) — most other PCB components are now identified by chip marking, see the [Hardware](#hardware) table
- **Kernel:** Experience with backporting a vintage MIPS platform driver to a modern kernel — the 2.6.31 reference now boots real hardware, so this is adaptation work with a proven target, not exploratory porting
- **Reverse engineering:** Cross-referencing the GPL kernel MDIO source against a QCA8xxx family datasheet (qca8327/8334/8337 public documentation) to identify QCA8519/QCA8719-specific register differences
- **GPL compliance:** Anyone willing to formally request U-Boot GPL sources from D-Link/Alpha Networks
- **Testing:** Anyone with the same hardware (DGS-1210-48, revision D1) — the [boot procedure](#working-boot-procedure) documented here should be directly reproducible

Feel free to open issues or pull requests. Discussions in English or French are welcome.

---

---

# Traduction française

> 🇬🇧 The English version above is the primary reference.

## D-Link DGS-1210-48 (Révision matérielle D1) — Rétro-ingénierie

Rétro-ingénierie complète du switch Gigabit D-Link DGS-1210-48 (révision matérielle D1), incluant le dump flash, l'extraction du firmware, l'identification du hardware, et l'évaluation de la portabilité OpenWrt.

## Objectifs

1. Documenter l'architecture hardware du DGS-1210-48 D1 pour la communauté
2. Évaluer la faisabilité d'un portage OpenWrt
3. Corriger les bugs connus du firmware stock (instabilité de l'IP de management)
4. Fournir une base solide pour quiconque voudrait travailler sur cet appareil

## Hardware

### Liste des composants

| Composant | Identification | Certitude | Méthode |
|-----------|---------------|-----------|---------|
| **CPU** | MIPS 24Kc V8.5, nom de code "Music" (confirmé au boot, ~266 BogoMIPS) | ✅ Confirmé (boot réel) | Kernel custom compilé, bootant jusqu'à un shell — `/proc/cpuinfo` rapporte `system type: Atheros Music`, `cpu model: MIPS 24Kc V8.5`. Identité commerciale de la puce toujours non confirmée |
| **Révision switch fabric** | "Vivo" B0 | ✅ Confirmé (lecture registre en direct) | Lecture directe de `REG_SWITCH_GLB_CTRL_0` (`0x18800000`) depuis un kernel de debug custom : `0x02022002`. Masqué avec `0xff` selon le code source GPL (`SWITCH_GLB_CTRL_0_REV_ID_MASK`), correspond exactement à `MUSIC_VIVO_REV_ID_B0` (`0x2`) |
| **Switch fabric (maître)** | Probablement QCA8519-AC2C, nom de code "Vivo" | 🟢 Candidat fort | Les chaînes `"UM8719A"`/`"UM8719B"` du code GPL nomment probablement la *carte*, pas la puce ; les fiches produit MikroTik CRS226/CRS210 listent QCA8519-AC2C avec un CPU embarqué, cohérent avec ce board |
| **ODM de la carte** | Alpha Networks Incorporation | ✅ Confirmé | En-tête copyright dans le code source GPL (`ver_uboot.h`) |
| **Flash** | Macronix MX25L12835FMI-10G — SPI NOR, 128 Mbit / 16 Mo | ✅ Confirmé (marquage puce) | Inspection PCB ("Mxix") + `bdinfo` + binwalk |
| **RAM** | Nanya NT5CB128M8FN-DH — DDR3-1600, 1 Gbit / 128 Mo | ✅ Confirmé (marquage puce) | Inspection PCB + `bdinfo` U-Boot |
| **Mux I2C** | NXP PCA9545A (4 canaux) | ✅ Confirmé (marquage puce) | Inspection PCB + symbole `DRV_CH_Select_PCA9545` dans `dgs_drv.ko` |
| **Expandeur GPIO** | NXP PCA9555 ×2 (16 bits I2C) | ✅ Confirmé (marquage puce) | Inspection PCB + symbole `DRV_IF_Get_PCA9555_LED_MODE` dans `dgs_drv.ko` |
| **CPLD** | Modèle inconnu | ✅ Confirmé (présent) | Symboles `DRV_CPLD_*` dans `dgs_drv.ko` |
| **EEPROM** | Atmel/Microchip série AT24C (EEPROM série I2C) | ✅ Confirmé (marquage puce) | Inspection PCB ("ATMLH548") + symbole `DRV_EEPROM_*` dans `dgs_drv.ko` |
| **RTC** | Modèle inconnu (I2C) | ✅ Confirmé (présent) | Symbole `DRV_RTC_Init` dans `dgs_drv.ko` |
| **Capteur temp.** | Microchip TCN75AVOA (capteur température I2C) | ✅ Confirmé (marquage puce) | Inspection PCB + symboles `DRV_THM_*` dans `dgs_drv.ko` |
| **Ports SFP** | 4× SFP (ports 49-52) | ✅ Confirmé | Symboles `DRV_SFP_*` + config stock |
| **Contrôle FAN** | Sortie PWM via MOSFET ou puce driver LED/puissance type TI (pas de puce contrôleur dédiée) | 🟡 Probable | Symbole `DRV_FAN_*` dans `dgs_drv.ko` n'expose que get/set du statut — cohérent avec un simple contrôle PWM du duty-cycle plutôt qu'un contrôleur intelligent avec son propre protocole |
| **Watchdog** | Intégré | ✅ Confirmé | Symbole `DRV_WD_*` dans `dgs_drv.ko` |
| **Monostables** | 74HC123D ×4 | ✅ Confirmé (marquage puce) | Inspection PCB |
| **Porte logique** | 74HC08D (quadruple porte ET) | ✅ Confirmé (marquage puce) | Inspection PCB |
| **UART** | ttyS0, 115200 baud, 3.3V | ✅ Confirmé | Accès direct |

#### Alimentation

| Composant | Rôle |
|-----------|------|
| Richtek RT8120B | Contrôleur PWM buck synchrone — pilote des MOSFETs pour abaisser/réguler une tension DC |
| ITE IT76820M | Convertisseur DC-DC / gestion d'énergie — fournit une ligne d'alimentation régulée |
| UTC 2SB772L | Transistor de puissance PNP (-30V, -3A) |
| Niko-Sem/UBIQ QM3004D (marqué "M3004D") | MOSFET canal N de puissance (30V, courant fort) |

## Firmware

### Stack logicielle

| Composant | Valeur |
|-----------|--------|
| U-Boot | 1.1.4 (29 jan 2013) |
| Kernel | Linux 2.6.31 (compilé 22 jan 2016) |
| BSP | Atheros "Music" ASDK 0.9.7.253 |
| Compilateur | GCC 4.3.3 |
| Rootfs | JFFS2 (big endian) |
| Init | BusyBox + scripts D-Link custom |

## Disposition de la flash

Confirmée au boot depuis la sortie MTD du kernel :

```
0x000000000000-0x000000040000 : "u-boot"       (256 Ko)
0x000000040000-0x000000080000 : "u-boot-env"   (256 Ko)
0x000000080000-0x000000280000 : "uImage"       (2 Mo)
0x000000280000-0x000001000000 : "roofs"        (13,5 Mo)
```

> **Note :** Le nom de partition `roofs` (au lieu de `rootfs`) est une faute de frappe dans le code source D-Link/Atheros d'origine.

## Bug connu — Instabilité de l'IP de management

Le switch cesse parfois de répondre sur son IP de management tout en continuant à commuter le trafic normalement.

**Cause probable :** `udhcpc` est lancé en arrière-plan sans mécanisme de redémarrage. Si le bail DHCP expire ou si le processus plante, l'IP de management est perdue.

**Contournement :** Configurer une IP statique via le CLI :
```
config ipif_cfg System ipaddress 192.168.X.X/24 vlan default
config ipif System dhcp disable
save
```

## Compilation du kernel custom & boot réel — SUCCÈS

**Un kernel Linux 2.6.31 auto-compilé, construit depuis les sources GPL officielles avec la `qca-music_toolchain` officielle, boote entièrement sur ce switch et atteint un vrai shell root.**

C'est le résultat le plus important de ce projet : la preuve que la toolchain, les sources kernel, et la compréhension hardware documentées ici sont correctes et suffisantes pour produire un firmware fonctionnel sur cet appareil.

### Problèmes rencontrés et corrigés

1. **`kernel/timeconst.pl` plante avec un Perl moderne** — corrigé en retirant le `defined()` autour d'un tableau. Doit être patché *à l'intérieur* de l'archive `linux-2.6.31.tar.gz` elle-même, car le Makefile racine fait `rm -rf` + ré-extraction à chaque build.

2. **Bug de locale sur `readelf` dans le Makefile vendor** — sur un système en français, `readelf -a | grep "Entry"` ne trouve rien (le texte est traduit), produisant un argument `-e` vide pour `mkimage`. Corrigé avec `LC_ALL=C`.

3. **`inflate() returned -3` à la décompression du kernel dans U-Boot** — **ce n'était pas un problème de version gzip/zlib** (testé avec plusieurs versions, y compris un `gzip 1.3.12` de 2007 compilé dans un conteneur CentOS 6). La vraie cause : **l'image compressée était chargée via `tftpboot` exactement à l'adresse RAM que le header spécifie comme adresse de *décompression*.** Pendant la décompression, la sortie grandissante écrase la fin des données compressées pas encore lues, corrompant le flux gzip en plein milieu. **Correction : charger l'image compressée à une adresse RAM séparée**, sans chevauchement (`tftpboot 0x81000000` au lieu de `0x80002000`), en laissant le `Load Address`/`Entry Point` du header inchangés.

4. **Kernel panic silencieux avant tout affichage console** — diagnostiqué en ajoutant des sondes UART brutes directement dans `arch/mips/kernel/head.S`. La cause s'est révélée identique au point 3 (chevauchement mémoire) — une fois corrigé, le boot a repris normalement.

5. **`VFS: Unable to mount root fs`** avec un `bootargs` simplifié — omettre `mtdparts=...` empêche la création des partitions MTD. Toujours garder la chaîne `mtdparts=` complète.

6. **BusyBox 1.19.4 échoue à compiler `ubi_tools.c`** contre les headers kernel de cette toolchain — désactivé dans `.config` (non nécessaire pour un shell de debug).

### Procédure de boot fonctionnelle

```
setenv bootargs console=ttyS0,115200 root=31:03 rootfstype=jffs2 init=/bin/sh mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs) mem=128M
port enable
tftpboot 0x81000000 <custom-uImage>
bootm 0x81000000
```

Ceci monte le vrai rootfs JFFS2 d'usine en lecture seule depuis la flash (`root=31:03`) et lance un shell BusyBox/ash root en direct — sans mot de passe, sans écriture flash, entièrement réversible par un simple redémarrage.

> Note : les modules `dgs_xal.ko`/`dgs_drv.ko` d'origine échouent au chargement contre ce kernel auto-compilé (`unknown symbol alpha_inter_ver`) car ils ont été compilés contre les CRC de version de modules exacts du kernel d'usine. C'est normal et sans conséquence pour un accès shell de debug.

### Confirmé via le shell en direct

```
/proc/cpuinfo :
  system type       : Atheros Music
  cpu model         : MIPS 24Kc V8.5
  BogoMIPS          : 266.24

/proc/mtd :
  mtd0: 00040000 00001000 "u-boot"
  mtd1: 00040000 00001000 "u-boot-env"
  mtd2: 00200000 00001000 "uImage"
  mtd3: 00d80000 00001000 "roofs"

Registre switch fabric (0x18800000) : 0x02022002
  → masqué & 0xff = 0x02 = MUSIC_VIVO_REV_ID_B0 (confirmé via le code source GPL)
```



Le code source GPL officiel de D-Link (firmware v4.10.023, trouvé sur le bucket S3 public de D-Link) a changé la donne : le cœur CPU "Music" n'a jamais eu de support upstream — ce n'est pas un vrai AMD Alchemy Au1210 mais un cœur MIPS propriétaire lié à la plateforme Atheros Switch SDK. En revanche, le protocole MDIO complet est maintenant documenté en code source (plus besoin de reverse engineering Ghidra pour cette couche), et une base de driver DSA existe déjà dans le kernel mainline pour la famille QCA8xxx apparentée (`qca8327`/`qca8334`/`qca8337`).

**Ce qui est réaliste :**
- Linux minimal qui boote (sans switch, en RAM via U-Boot stock) : quelques semaines, désormais moins risqué grâce au code source GPL qui couvre toute l'init hardware bas niveau
- OpenWrt complet avec les 48 ports : 4-8 mois — plus court qu'estimé initialement, mais le driver switch QCA8519/QCA8719 et le portage de la plateforme CPU restent à écrire
- **U-Boot reste un point bloquant** : D-Link n'a pas publié ses sources U-Boot (probable manquement GPL), et il n'existe aucune base upstream pour le cœur "Music"

## Comment contribuer

Ce projet a besoin d'aide pour :
- **Hardware :** Le CPLD et la RTC restent non identifiés (la puce CPU sous le dissipateur aussi) — la plupart des autres composants du PCB sont maintenant identifiés par marquage, voir le tableau [Hardware](#hardware)
- **Kernel :** Expérience avec le portage de plateformes MIPS embarquées, idéalement avec de l'expérience driver switch/DSA
- **Reverse engineering :** Croiser le code source MDIO du kernel GPL avec une datasheet de la famille QCA8xxx (documentation publique qca8327/8334/8337) pour identifier les différences spécifiques à la puce
- **Conformité GPL :** Toute personne prête à demander formellement les sources U-Boot GPL à D-Link/Alpha Networks
- **Tests :** Toute personne possédant le même matériel (DGS-1210-48, révision D1)

Les discussions en français ou en anglais sont les bienvenues.
