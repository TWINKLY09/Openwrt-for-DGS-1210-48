# TROUBLESHOOTING & PROCEDURES — D-Link DGS-1210-48 D1

Every problem encountered during this project, how to reproduce it, and the fix,
organized by category. Each entry is self-contained — you shouldn't need to read
the whole file to use one section.

---

## Table of Contents

1. [UART & Serial Console](#1-uart--serial-console)
2. [TFTP Server Setup](#2-tftp-server-setup)
3. [Flash Dumping via U-Boot](#3-flash-dumping-via-u-boot)
4. [Flash Dump Reconstruction](#4-flash-dump-reconstruction)
5. [Firmware/Rootfs Extraction](#5-firmwarerootfs-extraction)
6. [Toolchain Setup](#6-toolchain-setup)
7. [Kernel Build — Perl/Locale Issues](#7-kernel-build--perllocale-issues)
8. [Kernel Build — The Memory Overlap Bug](#8-kernel-build--the-memory-overlap-bug)
9. [Kernel Build — Archive Re-extraction Trap](#9-kernel-build--archive-re-extraction-trap)
10. [BusyBox Cross-Compilation](#10-busybox-cross-compilation)
11. [Booting to a Shell](#11-booting-to-a-shell)
12. [Docker for Vintage Toolchains](#12-docker-for-vintage-toolchains)

---

## 1. UART & Serial Console

### Setup
- **Voltage:** 3.3V only — never 5V, will damage the board
- **Settings:** 115200 baud (default) or 230400 baud (faster, for large dumps), 8N1, no flow control
- **Tool:** minicom

### Procedure
```bash
minicom -b 115200 -D /dev/ttyUSB0
```
Inside minicom:
- `Ctrl-A W` — disable line wrap (required before any large capture, e.g. flash dump)
- `Ctrl-A L` — enable logging to file (prompts for filename)
- `Ctrl-A P` — change baud rate on the fly (needed if you change U-Boot's `baudrate` env var mid-session)

### Common problem: garbled/no output
**Cause:** wrong baud rate, or TX/RX swapped.  
**Fix:** try 115200 first (U-Boot default), verify TX/RX orientation, confirm 3.3V logic level.

---

## 2. TFTP Server Setup

### Install
```bash
sudo apt install tftpd-hpa
```

### Config (`/etc/default/tftpd-hpa`)
```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure --create"
```

### Start & verify
```bash
sudo systemctl enable --now tftpd-hpa
sudo systemctl status tftpd-hpa
```

### Network interface setup
Dedicate a separate ethernet interface to the switch — don't touch your main
internet connection (works fine alongside Wi-Fi).

```bash
ip link show                                    # find the right interface
sudo ip addr add 192.168.1.27/24 dev enpXsX      # match U-Boot's serverip
sudo ip link set enpXsX up
```
No gateway, no DNS needed — this interface only talks to the switch.

### Common problem: `Filename 'X'. TFTP error 'Access violation'`
**Cause:** file not readable by the `tftp` user, or missing entirely.  
**Fix:**
```bash
sudo cp yourfile /srv/tftp/
sudo chmod 644 /srv/tftp/yourfile
```

### Common problem: firewall blocking transfer
```bash
sudo ufw allow 69/udp
```

---

## 3. Flash Dumping via U-Boot

### Why this method
This U-Boot build (1.1.4) has no `sf` (SPI flash direct access) and no `tftpput`
(upload). The NOR flash is memory-mapped, so `md.b` (memory display) over UART is
the only exfiltration path.

### Procedure
```
md.b 0xB9000000 0x1000000
```
(Address confirmed via `bdinfo` → `flashstart`. Size confirmed via `bdinfo` →
`flashsize`, here 16MB = `0x1000000`.)

- At 115200 baud: ~70 minutes
- At 230400 baud: ~35 minutes (change first with `setenv baudrate 230400`,
  then immediately switch minicom's baud rate too, `Ctrl-A P`)

### Common problem: `Ctrl-C` doesn't stop `md.b`
**Known limitation of this U-Boot build.** Options: let it finish (recommended —
you want the dump anyway), or power-cycle if you must abort. No flash writes occur
during a read, so this is always safe.

### Common problem: session appears to hang or logs stop
Check that:
- `Ctrl-A L` (logging) was active *before* starting `md.b`
- `Ctrl-A W` (line wrap off) was set — wrapped long lines corrupt the log format
- The PC didn't sleep/suspend mid-capture (disable power management for the duration)

---

## 4. Flash Dump Reconstruction

### Tool
`tools/uboot_mdb_to_bin.py` — parses the raw UART capture text and rebuilds the
binary flash image.

### Usage
```bash
python3 tools/uboot_mdb_to_bin.py capture.txt flash_dump.bin \
  --start-addr b9000000 --skip-errors --verbose
```

### Expected output
```
[INFO] 1047419 lignes parsées, 16777216 octets (16384.0 KB)
[OK]   Binaire écrit : flash_dump.bin (16777216 octets / 16.00 MB)
[OK]   Taille exacte 16MB — dump complet ✓
```

### Common problem: gap/padding warnings
```
[WARN] Ligne N: gap de 0x... octets → rembourrage avec 0xFF
```
Normal — a few dropped bytes during UART transit is expected at high baud rates.
The script pads with `0xFF`, which is also flash's natural erased-state value, so
this is harmless in practice for a handful of gaps.

### Verification
```bash
binwalk flash_dump.bin
```
Expect to see: U-Boot at `0x0`, uImage header + gzip kernel around `0x80000`,
JFFS2 filesystem starting around `0x280000`.

---

## 5. Firmware/Rootfs Extraction

### binwalk version issues
The `binwalk` 2.x from `apt`/older `pip` installs is frequently broken on modern
Python. Use `python3 -m binwalk` instead of the bare `binwalk` command if the
latter throws an `ImportError` traceback.

### Extracting the kernel
```bash
dd if=flash_dump.bin of=kernel.gz bs=1 skip=$((0x80040)) count=<size-from-binwalk>
gunzip -c kernel.gz > vmlinux.bin
```
Use `bs=65536` instead of `bs=1` for speed on large extractions (JFFS2 rootfs, etc.)
— the `bs=1` byte-at-a-time approach is extremely slow (minutes vs. seconds).

### Extracting the rootfs (JFFS2)
```bash
pip install jefferson --break-system-packages
jefferson rootfs.jffs2 -d rootfs_extracted
```
More reliable than mounting via `mtdram`/`mtdblock` kernel modules, which
frequently fail with `superblock` errors on modern kernels for old JFFS2 images.

### Common problem: `jefferson: command not found` despite `pip install` succeeding
**Cause:** installed to `~/.local/bin`, not in `PATH`.
```bash
export PATH=$PATH:~/.local/bin
# or call directly:
~/.local/bin/jefferson rootfs.jffs2 -d rootfs_extracted
```

---

## 6. Toolchain Setup

### Extracting D-Link's GPL toolchain
```bash
mkdir -p build/toolchain
cp -r gpl-sources/qca-music_toolchain build/toolchain
```

### Critical: hardcoded path requirement
The vendor Makefiles (`makefile.compile`) hardcode the toolchain path:
```makefile
OS_TOOL_CHAIN_PATH := /opt/qca-music_toolchain/build_mips/staging_dir/usr
```
**You must make this exact path exist**, via symlink:
```bash
sudo mkdir -p /opt
sudo ln -s ~/DLINK-SWITCH/build/toolchain /opt/qca-music_toolchain
```

### Environment variables (for manual builds outside the vendor Makefile)
```bash
export PATH=$PATH:/opt/qca-music_toolchain/build_mips/staging_dir/usr/bin
export CROSS_COMPILE=mips-linux-uclibc-
export ARCH=mips
```
Verify:
```bash
mips-linux-uclibc-gcc --version
# should print: gcc version 4.3.3
```

### Common problem: `cannot execute binary file`
**Cause:** toolchain binaries are 32-bit x86 (`ELF 32-bit LSB executable, Intel
80386`), and your host is 64-bit without multilib support.  
**Fix:**
```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libc6:i386 libstdc++6:i386 zlib1g:i386 libncurses5:i386
```

---

## 7. Kernel Build — Perl/Locale Issues

### Problem: `Can't use 'defined(@array)'` at `timeconst.pl` line 373
**Cause:** `kernel/timeconst.pl` uses Perl syntax that modern Perl (5.28+) rejects
as a fatal error instead of a warning.

**Fix:** patch the specific line:
```bash
sed -i 's/!defined(@val)/!@val/' kernel/timeconst.pl
```

**Critical gotcha:** the top-level vendor Makefile does `rm -rf linux-2.6.31;
tar -xzf linux-2.6.31.tar.gz` **on every single build invocation**. Patching the
extracted directory alone gets silently wiped on the next `make`. You must patch
**inside the `.tar.gz` archive itself** — see [Section 9](#9-kernel-build--archive-re-extraction-trap).

### Problem: `mkimage: invalid entry point -n`
**Cause:** the vendor Makefile extracts `Entry Point` via
`readelf -a | grep "Entry" | cut -d":" -f 2`. On a non-English system locale,
`readelf`'s field labels are translated (e.g. French: "Adresse du point d'entrée"),
so `grep "Entry"` matches nothing, producing an empty argument that breaks the
`mkimage` invocation.

**This error is harmless and expected** at this specific step — the kernel itself
(`vmlinux`) has already compiled successfully by this point; only the vendor's own
automatic `uImage` packaging step fails. Generate the `uImage` manually instead:

```bash
LC_ALL=C readelf -h vmlinux | grep "Entry point address"
# always force LC_ALL=C for any readelf/mkimage command from now on
```

---

## 8. Kernel Build — The Memory Overlap Bug

**This was the hardest bug of the whole project — most time-consuming to diagnose.**

### Symptom
```
Uncompressing Kernel Image ... Error: inflate() returned -3
GUNZIP ERROR - must RESET board to recover
```

### False leads (don't waste time here)
- ❌ gzip/zlib version mismatch — ruled out by testing multiple gzip versions,
  including a period-correct `gzip 1.3.12` (2007) via a CentOS 6 Docker container.
  Same failure every time, regardless of which gzip produced the file.
- ❌ FNAME header flag (`-n` gzip option to strip it) — the original factory
  firmware's compressed kernel *also* has the FNAME flag (confirmed via binwalk:
  `has original file name: "vmlinux.bin"`), so this was never the actual cause.
- ❌ Corrupted kernel `.config` or bad compilation — the exact same failure
  reproduces with a completely stock, unmodified kernel build.

### Real cause
The compressed uImage was loaded via `tftpboot` to the **same RAM address**
(`0x80002000`) that the uImage header specifies as the **decompression output**
`Load Address`. As U-Boot decompresses, the growing output overwrites the tail
of the still-unread compressed input mid-stream — the gzip stream corrupts itself.

### Fix
Load the compressed image to a **separate scratch address** with enough headroom
that decompressed output never catches up to unread input. Leave `Load Address`/
`Entry Point` in the uImage header unchanged (still `0x80002000`/kernel entry).

```
tftpboot 0x81000000 your-kernel-uimage
bootm 0x81000000
```

`bootm` reads the header, sees `Load Address: 0x80002000`, and decompresses
*from* `0x81000000` *into* `0x80002000` — no overlap, no corruption.

### Related symptom this same bug also caused
Complete silent early-boot failure with **zero console output**, showing only:
```
## Transferring control to Linux (at address ...) ...
## Giving linux memsize in bytes, 134217728

Starting kernel ...

## Control returned to monitor - resetting...
```
This looked like a CPU-specific early-boot crash (and was diagnosed at length via
raw UART probes injected directly into `arch/mips/kernel/head.S`, bypassing
`printk` entirely) — but once traced back, it was the exact same memory-overlap
bug, just manifesting differently depending on `-C gzip` vs `-C none` (uncompressed)
image type. Fixing the load address fixed both symptoms simultaneously.

---

## 9. Kernel Build — Archive Re-extraction Trap

### The trap
Any patch applied to files under `linux-2.6.31/` (the extracted directory) is
**silently discarded** the next time `make` runs from the vendor's top-level
Makefile, because it does:
```makefile
compile:
    rm -rf linux-$(OS_VER)
    tar -zxvf linux-$(OS_VER).tar.gz
```

### Correct patching procedure
Always patch **inside a copy of the `.tar.gz`**, then re-pack and overwrite:

```bash
rm -rf /tmp/rebuild
mkdir /tmp/rebuild
cd /tmp/rebuild
tar -xzf /path/to/linux-2.6.31.tar.gz.orig    # use a known-clean backup, not the working copy

# apply your patch(es) here, e.g.:
sed -i 's/!defined(@val)/!@val/' linux-2.6.31/kernel/timeconst.pl

# verify before repacking — don't skip this
grep -c "defined(@" linux-2.6.31/kernel/timeconst.pl    # expect 0

tar -czf linux-2.6.31.tar.gz linux-2.6.31
mv linux-2.6.31.tar.gz /path/to/linux/linux-2.6.31.tar.gz
```

### Always keep a pristine backup
Before applying *any* patch, save an untouched copy:
```bash
cp linux-2.6.31.tar.gz linux-2.6.31.tar.gz.orig
```
This saved significant time later when a multi-patch attempt got corrupted by
repeated/duplicated `sed` insertions (see next point) — being able to restart from
a guaranteed-clean archive rather than trying to "un-patch" a messy file was much
faster.

### Trap: repeated command execution silently duplicating patches
Running the same `sed -i '/pattern/r insertfile'` command **more than once**
inserts the content multiple times, since the pattern still matches after the
first insertion. This happened during iterative debugging (re-running a block of
commands from scrollback without noticing it had already succeeded) and produced
a corrupted, triplicated `head.S` that failed to compile correctly.

**Prevention:** always verify a patch's line count *immediately* after applying
it (`grep -c "unique-string" file`), before moving on to the next step. Don't
assume a command "probably didn't run twice" — check.

**Recovery:** don't try to manually de-duplicate a corrupted file. Restart from
the `.orig` backup and reapply cleanly — much faster and more reliable.

---

## 10. BusyBox Cross-Compilation

### Version
`busybox-1.19.4` — chosen to match `BUSYBOX := busybox-1.19.4` referenced in the
vendor's own `makefile.sdk`, for maximum compatibility with this toolchain/kernel.

### Build
```bash
cd build
wget https://busybox.net/downloads/busybox-1.19.4.tar.bz2
tar xjf busybox-1.19.4.tar.bz2
cd busybox-1.19.4

export PATH=$PATH:/opt/qca-music_toolchain/build_mips/staging_dir/usr/bin
export CROSS_COMPILE=mips-linux-uclibc-
export ARCH=mips

make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config   # static linking required
make -j$(nproc)
make install
```

### Common problem: `make defconfig` becomes interactive, asking Y/n for every option
Happens with some `make`/`kconfig` version combinations on modern hosts against
this old BusyBox. Just answer through it (defaults are fine — mostly `Y`), or
retry with `yes "" | make defconfig` to auto-accept.

### Common problem: `ubi_tools.c` fails to compile
```
miscutils/ubi_tools.c:63:26: error: mtd/ubi-user.h: No such file or directory
```
**Cause:** this toolchain's kernel headers don't include UBI support headers.
**Fix:** not needed for a debug shell — disable in `.config`:
```bash
sed -i \
  -e 's/CONFIG_UBIATTACH=y/# CONFIG_UBIATTACH is not set/' \
  -e 's/CONFIG_UBIDETACH=y/# CONFIG_UBIDETACH is not set/' \
  -e 's/CONFIG_UBIMKVOL=y/# CONFIG_UBIMKVOL is not set/' \
  -e 's/CONFIG_UBIRMVOL=y/# CONFIG_UBIRMVOL is not set/' \
  -e 's/CONFIG_UBIRSVOL=y/# CONFIG_UBIRSVOL is not set/' \
  -e 's/CONFIG_UBIUPDATEVOL=y/# CONFIG_UBIUPDATEVOL is not set/' \
  .config
```
Note the option names are `CONFIG_UBIATTACH` etc. (per-applet), **not** a single
`CONFIG_UBI_TOOLS` — a wrong guess at the option name will silently do nothing.

### Verification
```bash
file _install/bin/busybox
# expect: ELF 32-bit MSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, stripped
```

---

## 11. Booting to a Shell

### The working, reproducible procedure
```
setenv bootargs console=ttyS0,115200 root=31:03 rootfstype=jffs2 init=/bin/sh mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs) mem=128M
port enable
tftpboot 0x81000000 <your-uImage>
bootm 0x81000000
```

### Why `init=/bin/sh`
Bypasses `/sbin/init` → `rcS` → the proprietary `DGS12XX` application entirely.
Since the real stock JFFS2 rootfs mounts successfully (it's the actual flash
content, untouched), this drops straight into a BusyBox `ash` shell as root,
no password needed.

### Common problem: `VFS: Unable to mount root fs on unknown-block(31,3)`
**Cause:** `mtdparts=...` was omitted or shortened in a custom `bootargs`. Without
it, the kernel never creates the MTD partitions, so `root=31:03` (major:minor for
`/dev/mtdblock3`) points at nothing.  
**Fix:** always keep the full `mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),
2048k(uImage),-(roofs)` string in `bootargs`, even when adding other custom
parameters like `init=/bin/sh`.

### Common problem: `/bin/sh: can't access tty; job control turned off`
Harmless — just `ash` reporting no job control on this console. The shell works
normally right after; this is not an error to troubleshoot further.

### Common problem: precompiled `dgs_xal.ko`/`dgs_drv.ko` fail with `unknown symbol`
```
insmod: can't insert 'dgs_xal.ko': unknown symbol in module, or unknown parameter
```
**Cause:** these `.ko` files were compiled against the *exact* modversion CRCs of
the factory kernel build. A self-compiled kernel — even from identical source —
has different CRCs, so the module loader rejects the symbols as mismatched.  
**Not a bug to fix for shell access** — the rootfs mounts and a shell works fine
without these modules loading. Only matters if you want the stock `DGS12XX`
management application to run under a custom kernel, which would require either
an exactly-matching `.config`, or rebuilding these modules from source (not
available — only precompiled binaries were published by D-Link).

### Undo / reset instructions
None of the above ever writes to flash. `setenv` without `saveenv` only affects
the current U-Boot session's RAM copy of the environment.
```
reset
```
or a power cycle restores the original `bootargs`/`bootcmd` from the flash-stored
environment automatically — no lasting changes, always safe to experiment.

---

## 12. Docker for Vintage Toolchains

### Why
Reproducing period-correct tool behavior (e.g. `gzip` from ~2007–2011, matching
the actual era of this vendor toolchain) is sometimes necessary to rule out
version-mismatch hypotheses, even when — as in this project's case — it later
turns out not to be the real root cause.

### Procedure
```bash
docker pull centos:6
docker run --rm centos:6 gzip --version
# gzip 1.3.12 (2007) — matches the toolchain's era
```

### Using it to compress a file
```bash
docker run --rm -v /host/path:/work centos:6 gzip -f /work/somefile.bin
```

### Common problem: `bash -c "cmd1; cmd2"` inside `docker run` produces no output at all
Seen when combining a volume mount with a semicolon-separated inline script —
troubleshoot by testing each piece separately:
```bash
docker run --rm centos:6 echo "test"              # confirm container runs at all
docker run --rm -v /host:/work centos:6 ls /work   # confirm mount works
docker run --rm centos:6 which gzip                # confirm tool exists
```
Isolating each step individually is much faster than debugging a combined
multi-command invocation blind.

### Note: CentOS 6 is EOL
The image still pulls fine from Docker Hub as of this writing, but its internal
`yum` repos point to now-defunct mirrors. If you need to `yum install` anything
beyond what's already in the base image, repoint to the CentOS Vault first:
```bash
sed -i 's/mirror.centos.org/vault.centos.org/g; s/^#.*baseurl=http/baseurl=http/g; s/^mirrorlist=http/#mirrorlist=http/g' /etc/yum.repos.d/*.repo
```

---

## Quick Reference — Full Working Sequence, Start to Shell

For a fresh checkout, the complete sequence that's known to work:

```bash
# 1. Toolchain
mkdir -p ~/DLINK-SWITCH/build/toolchain
cp -r ~/DLINK-SWITCH/gpl-sources/qca-music_toolchain ~/DLINK-SWITCH/build/toolchain
sudo ln -s ~/DLINK-SWITCH/build/toolchain /opt/qca-music_toolchain
export PATH=$PATH:/opt/qca-music_toolchain/build_mips/staging_dir/usr/bin
export CROSS_COMPILE=mips-linux-uclibc-
export ARCH=mips

# 2. Kernel source, patched
cd ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL/linux
cp linux-2.6.31.tar.gz linux-2.6.31.tar.gz.orig    # backup before first patch
# (apply timeconst.pl patch inside the archive per Section 9)

# 3. Build
cd ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL
make    # will "fail" at the final mkimage step due to locale bug — that's fine, vmlinux is built

# 4. Manual uImage generation
mips-linux-objcopy -S -O binary --remove-section=.reginfo --remove-section=.mdebug \
  linux/linux-2.6.31/vmlinux linux/linux-2.6.31/vmlinux.bin
docker run --rm -v $(pwd)/linux/linux-2.6.31:/work centos:6 gzip -f /work/vmlinux.bin
ENTRY=$(LC_ALL=C readelf -h linux/linux-2.6.31/vmlinux | grep "Entry point address" | awk '{print $NF}')
pkgs/sdk/asdk_branch_rc_patch_0.9.7.253/system/cpusub/music/linux/filesystem/tools/mkimage \
  -A mips -O linux -T kernel -C gzip -a 0x80002000 -e $ENTRY -n "Linux Kernel Image" \
  -d linux/linux-2.6.31/vmlinux.bin.gz image/knl_img_custom

# 5. Deploy
sudo cp image/knl_img_custom /srv/tftp/
sudo chmod 644 /srv/tftp/knl_img_custom

# 6. Boot (on the switch's UART console)
#    setenv bootargs console=ttyS0,115200 root=31:03 rootfstype=jffs2 init=/bin/sh mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs) mem=128M
#    port enable
#    tftpboot 0x81000000 knl_img_custom
#    bootm 0x81000000
```
