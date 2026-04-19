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
| **CPU** | AMD Alchemy Au1210 (MIPS 4Kc, ~400MHz, big endian) | ✅ Confirmed | PRId table in `vmlinux.bin`, build path `cpusub/music` |
| **Switch fabric** | QCA8519 or QCA8719 (48-port GbE) | 🟡 Probable | Detection list in `DGS12XX` binary |
| **Flash** | NOR 16MB | ✅ Confirmed | `bdinfo` + binwalk |
| **RAM** | 128MB | ✅ Confirmed | `bdinfo` U-Boot |
| **I2C mux** | NXP PCA9545 (4-channel) | ✅ Confirmed | Symbol `DRV_CH_Select_PCA9545` in `dgs_drv.ko` |
| **GPIO expander** | NXP PCA9555 ×2 | ✅ Confirmed | Symbol `DRV_IF_Get_PCA9555_LED_MODE` in `dgs_drv.ko` |
| **CPLD** | Unknown model | ✅ Confirmed (present) | Symbols `DRV_CPLD_*` in `dgs_drv.ko` |
| **EEPROM** | Unknown model (I2C) | ✅ Confirmed (present) | Symbol `DRV_EEPROM_*` in `dgs_drv.ko` |
| **RTC** | Unknown model (I2C) | ✅ Confirmed (present) | Symbol `DRV_RTC_Init` in `dgs_drv.ko` |
| **Temp. sensor** | Unknown model | ✅ Confirmed (present) | Symbols `DRV_THM_*` in `dgs_drv.ko` |
| **Ports** | 48× GbE RJ45 (1-48) + 4× SFP (49-52) | ✅ Confirmed | Stock config |
| **Fan controller** | Unknown | 🟡 Probable | Symbol `DRV_FAN_*` in `dgs_drv.ko` |
| **Watchdog** | Integrated | ✅ Confirmed | Symbol `DRV_WD_*` in `dgs_drv.ko` |
| **Monostable** | 74HC123 ×4 | ✅ Confirmed (visual) | PCB inspection |
| **UART** | ttyS0, 115200 baud, 3.3V | ✅ Confirmed | Direct access |

### Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│              AMD Alchemy Au1210 (CPU)                         │
│         MIPS 4Kc · Big Endian · ~400MHz                       │
│                                                               │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────┐          │
│  │ NOR Flash   │  │128MB RAM │  │   UART ttyS0     │          │
│  │   16MB      │  │  DDR     │  │   115200 baud    │          │
│  └─────────────┘  └──────────┘  └──────────────────┘          │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐    │
│  │              MDIO Bus                                 │    │
│  │    ┌─────────────────────────────────────────────┐    │    │
│  │    │  QCA8519/QCA8719 (Switch Fabric)            │    │    │
│  │    │  48× GbE (ports 1-48) + 4× SFP (ports 49-52)│    │    │
│  │    └─────────────────────────────────────────────┘    │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐    │
│  │              I2C Bus                                  │    │
│  │  ┌──────────┐                                         │    │
│  │  │ PCA9545  │ (4-channel I2C mux)                     │    │
│  │  │  (mux)   │                                         │    │
│  │  └────┬─────┘                                         │    │
│  │  ┌────┴──────────────┐                                │    │
│  │  │ PCA9555 ×2        │ (GPIO expanders)               │    │
│  │  │ CPLD              │ (LEDs, FAN, power)             │    │
│  │  │ EEPROM            │                                │    │
│  │  │ RTC               │                                │    │
│  │  │ Temp. sensor      │                                │    │
│  │  └───────────────────┘                                │    │
│  └───────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
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

## Tools & Methods

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

## OpenWrt Portability

### Assessment

| Item | Status | Notes |
|------|--------|-------|
| CPU (Au1210) support in mainline kernel | ⚠️ Dropped | Removed ~2017, last in kernel ~4.x |
| Au1000 target in OpenWrt | ⚠️ Removed | Last commit: `e6f9a8e89b`, kernel 3.18 |
| Au1210 big-endian vs au1000 little-endian | ❌ Mismatch | Needs `ARCH=mips` not `mipsel` |
| QCA851x switch fabric driver | ❌ None | No open source driver exists |
| PCA9545/PCA9555 drivers | ✅ Mainline | Standard Linux I2C drivers |
| NOR flash driver | ✅ Mainline | Standard MTD |

### What Needs to be Done

1. **Restore Au1210 big-endian support** in a modern kernel (backport from last known good state)
2. **Write or reverse-engineer a QCA851x DSA driver** — this is the hardest part, requires analysis of `dgs_drv.ko` and `sdk_um_uk_if.ko` in Ghidra
3. **Create a board DTS** for the DGS-1210-48 D1
4. **Handle peripheral drivers** — PCA9545, PCA9555, CPLD, RTC (mostly available upstream)
5. **Port U-Boot** or adapt existing Alchemy U-Boot support

### Realistic Timeline

- Linux minimal boot (no switch): a few weeks for someone familiar with MIPS BSP work
- Full OpenWrt with all 48 ports: 6-12 months, requires reverse engineering of proprietary MDIO driver

---

## How to Contribute

This project needs help with:

- **Hardware:** Photos of the PCB to identify CPLD, RTC, temp sensor references
- **Kernel:** Experience with AMD Alchemy / MIPS BSP porting
- **Reverse engineering:** Ghidra analysis of `dgs_drv.ko` and `sdk_um_uk_if.ko` to document the QCA851x MDIO protocol
- **Testing:** Anyone with the same hardware (DGS-1210-48, revision D1)

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
| **CPU** | AMD Alchemy Au1210 (MIPS 4Kc, ~400MHz, big endian) | ✅ Confirmé | Table PRId dans `vmlinux.bin`, chemin de compilation `cpusub/music` |
| **Switch fabric** | QCA8519 ou QCA8719 (48 ports GbE) | 🟡 Probable | Liste de détection dans le binaire `DGS12XX` |
| **Flash** | NOR 16MB | ✅ Confirmé | `bdinfo` + binwalk |
| **RAM** | 128MB | ✅ Confirmé | `bdinfo` U-Boot |
| **Mux I2C** | NXP PCA9545 (4 canaux) | ✅ Confirmé | Symbole `DRV_CH_Select_PCA9545` dans `dgs_drv.ko` |
| **Expandeur GPIO** | NXP PCA9555 ×2 | ✅ Confirmé | Symbole `DRV_IF_Get_PCA9555_LED_MODE` dans `dgs_drv.ko` |
| **CPLD** | Modèle inconnu | ✅ Confirmé (présent) | Symboles `DRV_CPLD_*` dans `dgs_drv.ko` |
| **EEPROM** | Modèle inconnu (I2C) | ✅ Confirmé (présent) | Symbole `DRV_EEPROM_*` dans `dgs_drv.ko` |
| **RTC** | Modèle inconnu (I2C) | ✅ Confirmé (présent) | Symbole `DRV_RTC_Init` dans `dgs_drv.ko` |
| **Capteur temp.** | Modèle inconnu | ✅ Confirmé (présent) | Symboles `DRV_THM_*` dans `dgs_drv.ko` |
| **Ports SFP** | 4× SFP (ports 49-52) | ✅ Confirmé | Symboles `DRV_SFP_*` + config stock |
| **Contrôleur FAN** | Inconnu | 🟡 Probable | Symbole `DRV_FAN_*` dans `dgs_drv.ko` |
| **Watchdog** | Intégré | ✅ Confirmé | Symbole `DRV_WD_*` dans `dgs_drv.ko` |
| **Monostables** | 74HC123 ×4 | ✅ Confirmé (visuel) | Inspection PCB |
| **UART** | ttyS0, 115200 baud, 3.3V | ✅ Confirmé | Accès direct |

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

## Portage OpenWrt

Le portage complet est techniquement possible mais ambitieux :
- Le support CPU Au1210 a été retiré du kernel mainline vers 2017
- La cible `au1000` OpenWrt a été supprimée (dernier commit : `e6f9a8e89b`, kernel 3.18)
- Il n'existe pas de driver open source pour le switch fabric QCA851x

**Ce qui est réaliste :**
- Linux minimal qui boote (sans switch) : quelques semaines
- OpenWrt complet avec les 48 ports : 6-12 mois, nécessite le reverse engineering du driver MDIO propriétaire

## Comment contribuer

Ce projet a besoin d'aide pour :
- **Hardware :** Photos du PCB pour identifier le CPLD, la RTC, le capteur de température
- **Kernel :** Expérience avec le portage AMD Alchemy / BSP MIPS
- **Reverse engineering :** Analyse Ghidra de `dgs_drv.ko` et `sdk_um_uk_if.ko`
- **Tests :** Toute personne possédant le même matériel (DGS-1210-48, révision D1)

Les discussions en français ou en anglais sont les bienvenues.
