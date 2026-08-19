# HOW TO GET WHERE I'M AT — Full Reproduction Guide

A linear, start-to-finish path from an unmodified DGS-1210-48 D1 to a live root
shell running a self-compiled kernel. This skips the dead ends and false leads
explored during the original research — for troubleshooting specific errors along
the way, see [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md).

**Time estimate:** a few hours if nothing goes wrong, spread over a couple of
sessions if you're doing the UART flash dump (that alone takes 35-70 minutes of
just waiting).

**Risk level:** low, as long as you follow the "never write to flash" rule below.

---

## Prerequisites

- A DGS-1210-48, hardware revision D1 (check the label on the back)
- A 3.3V USB-to-TTL UART adapter (**not** 5V — will damage the board)
- A Linux PC with a spare ethernet port
- Patience — several steps involve waiting on slow serial transfers

## The One Safety Rule

**Never run `erase`, `cp <ram> <flash>`, `dltftp`, or `saveenv` in U-Boot unless
you specifically mean to.** Everything in this guide — dumping, `setenv` without
`saveenv`, `tftpboot`+`bootm` — is RAM-only and 100% reversible with a power cycle.
If you stick to that, there is no way to brick the device following this guide.

---

## Step 1 — Get UART Access

1. Open the switch, locate the UART header on the PCB (internal, near the CPU
   area typically)
2. Connect your 3.3V USB-TTL adapter: TX↔RX, RX↔TX, GND↔GND (don't connect VCC —
   the board is self-powered)
3. Connect with minicom:
   ```bash
   minicom -b 115200 -D /dev/ttyUSB0
   ```
4. Power on the switch. You should see U-Boot output.

**Confirm you have a U-Boot shell:**
```
music>
```
If autoboot completes before you can interrupt it, power-cycle and press a key
during the boot countdown ("Hit any key to stop autoboot").

---

## Step 2 — Explore U-Boot (read-only, safe)

```
version
bdinfo
printenv
imls
```

Note down `flashstart`, `flashsize`, `ipaddr`, `serverip` from the output —
you'll need these later. On this exact hardware:
```
flashstart  = 0xB9000000
flashsize   = 0x01000000    (16MB)
```

---

## Step 3 — Set Up TFTP on Your PC

```bash
sudo apt install tftpd-hpa
sudo systemctl enable --now tftpd-hpa
```

Dedicate a network interface to the switch (doesn't interfere with Wi-Fi/main
internet):
```bash
ip link show                                     # identify your ethernet interface
sudo ip addr add 192.168.1.27/24 dev enpXsX       # match U-Boot's serverip
sudo ip link set enpXsX up
```

Connect the switch's ethernet port to this interface, then verify from U-Boot:
```
port enable
ping 192.168.1.27
```
Expect: `host 192.168.1.27 is alive`

---

## Step 4 — Dump the Flash

```
md.b 0xB9000000 0x1000000
```

Before running this: enable minicom logging (`Ctrl-A L`, name it `capture.txt`)
and disable line wrap (`Ctrl-A W`). This takes **~70 minutes at 115200 baud**
(or ~35 min if you first `setenv baudrate 230400` and switch minicom to match
with `Ctrl-A P`). `Ctrl-C` will not interrupt it — let it finish.

Once done, reconstruct the binary on your PC:
```bash
python3 tools/uboot_mdb_to_bin.py capture.txt flash_dump.bin \
  --start-addr b9000000 --skip-errors --verbose
```
Expect: `Taille exacte 16MB — dump complet ✓`

---

## Step 5 — Extract the Rootfs (optional but useful for exploration)

```bash
binwalk flash_dump.bin
# confirms rootfs (JFFS2) starts around offset 0x280000

dd if=flash_dump.bin of=rootfs.jffs2 bs=65536 skip=$((0x280000/65536))

pip install jefferson --break-system-packages
jefferson rootfs.jffs2 -d rootfs_extracted
```

You now have the factory rootfs contents to browse (`rootfs_extracted/`).

---

## Step 6 — Get D-Link's GPL Source Code

D-Link publishes GPL sources on a public S3 bucket. Find the exact package
matching your firmware version — for v4.10.023 D1:
```
https://dlink-gpl.s3.amazonaws.com/GPL1300053/DGS-1210-48_D1_GPL%20source%20code_for_FW_v4.10.023.tar.gz
```
(If your firmware version differs, search the bucket listing — see
`TROUBLESHOOTING.md` or the main README for the S3 listing procedure.)

```bash
mkdir -p ~/DLINK-SWITCH/gpl-sources
cd ~/DLINK-SWITCH/gpl-sources
curl -L -o gpl.tar.bz2 "https://dlink-gpl.s3.amazonaws.com/GPL1300053/DGS-1210-48_D1_GPL%20source%20code_for_FW_v4.10.023.tar.gz"

file gpl.tar.bz2   # will report "bzip2 compressed data" despite the .tar.gz extension
tar -xjf gpl.tar.bz2
```

Inside you'll find `dgs-1210-4.10.023_48_GPL.tar.bz2` (kernel + platform source)
and `qca-music_toolchain.tar.bz2` (cross-compiler). Extract both:
```bash
tar -xjf code_for_FW_v4.10.023/dgs-1210-4.10.023_48_GPL.tar.bz2
tar -xjf code_for_FW_v4.10.023/qca-music_toolchain.tar.bz2
```

---

## Step 7 — Set Up the Toolchain

```bash
mkdir -p ~/DLINK-SWITCH/build
cp -r qca-music_toolchain ~/DLINK-SWITCH/build/toolchain

sudo mkdir -p /opt
sudo ln -s ~/DLINK-SWITCH/build/toolchain /opt/qca-music_toolchain

export PATH=$PATH:/opt/qca-music_toolchain/build_mips/staging_dir/usr/bin
export CROSS_COMPILE=mips-linux-uclibc-
export ARCH=mips

mips-linux-uclibc-gcc --version
# expect: gcc version 4.3.3
```

If you get `cannot execute binary file`, your host needs 32-bit library support:
```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libc6:i386 libstdc++6:i386 zlib1g:i386 libncurses5:i386
```

---

## Step 8 — Patch the Kernel Source (required for a modern build host)

```bash
cd ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL/linux
cp linux-2.6.31.tar.gz linux-2.6.31.tar.gz.orig    # keep a clean backup

mkdir -p /tmp/rebuild
cd /tmp/rebuild
tar -xzf ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL/linux/linux-2.6.31.tar.gz.orig

sed -i 's/!defined(@val)/!@val/' linux-2.6.31/kernel/timeconst.pl

# verify — must print 0
grep -c "defined(@" linux-2.6.31/kernel/timeconst.pl

tar -czf linux-2.6.31.tar.gz linux-2.6.31
mv linux-2.6.31.tar.gz ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL/linux/linux-2.6.31.tar.gz
```

**Why this is needed:** `timeconst.pl` uses Perl syntax modern Perl rejects.
The patch must go inside the `.tar.gz` itself, because the vendor build system
re-extracts it from scratch on every build.

---

## Step 9 — Build the Kernel

```bash
cd ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL
make 2>&1 | tee ~/DLINK-SWITCH/build/build.log
```

This will end with an error at the final `mkimage` step
(`invalid entry point -n`) — **that's expected and harmless**, it's a locale bug
in the vendor Makefile's own packaging step. The important thing, `vmlinux`, is
already built successfully by that point. Confirm:
```bash
ls -la linux/linux-2.6.31/vmlinux
```

---

## Step 10 — Package the Kernel Yourself (uImage)

```bash
cd ~/DLINK-SWITCH/build/dgs-1210-4.10.023_48_GPL

mips-linux-objcopy -S -O binary --remove-section=.reginfo --remove-section=.mdebug \
  linux/linux-2.6.31/vmlinux linux/linux-2.6.31/vmlinux.bin

gzip -f linux/linux-2.6.31/vmlinux.bin
# (a period-correct gzip via Docker/CentOS 6 is not actually required — this was
#  tested and ruled out as a variable; see TROUBLESHOOTING.md Section 8 for why)

ENTRY=$(LC_ALL=C readelf -h linux/linux-2.6.31/vmlinux | grep "Entry point address" | awk '{print $NF}')
echo "Entry: $ENTRY"   # sanity check — should print something like 0x80006210

pkgs/sdk/asdk_branch_rc_patch_0.9.7.253/system/cpusub/music/linux/filesystem/tools/mkimage \
  -A mips -O linux -T kernel -C gzip \
  -a 0x80002000 -e $ENTRY -n "Linux Kernel Image" \
  -d linux/linux-2.6.31/vmlinux.bin.gz \
  image/knl_img_custom

ls -la image/knl_img_custom
```

**Always use `LC_ALL=C` with `readelf`** — on a non-English locale, its output
is translated and the entry point extraction silently breaks.

---

## Step 11 — Deploy and Boot

```bash
sudo cp image/knl_img_custom /srv/tftp/
sudo chmod 644 /srv/tftp/knl_img_custom
```

On the switch's UART console:
```
setenv bootargs console=ttyS0,115200 root=31:03 rootfstype=jffs2 init=/bin/sh mtdparts=music-nor0:256k(u-boot),256k(u-boot-env),2048k(uImage),-(roofs) mem=128M
port enable
tftpboot 0x81000000 knl_img_custom
bootm 0x81000000
```

**Critical detail:** load the image to `0x81000000`, *not* `0x80002000` (which is
the kernel's own `Load Address`/decompression target). Loading to the same
address the decompressor writes to causes the compressed stream to overwrite
itself mid-decompression (`Error: inflate() returned -3`) — this was the single
hardest bug in this whole project to diagnose. See `TROUBLESHOOTING.md` Section 8
for the full story if curious.

**Critical detail #2:** keep the full `mtdparts=...` string in `bootargs`, even
though we're overriding `init=`. Without it, the kernel can't find the root
filesystem partition and panics with `VFS: Unable to mount root fs`.

---

## Step 12 — You Have a Shell

If everything above worked, you'll see the kernel boot log end with the real
factory JFFS2 rootfs mounting, and then a shell prompt:
```
VFS: Mounted root (jffs2 filesystem) readonly on device 31:3.
Freeing unused kernel memory: 132k freed
/bin/sh: can't access tty; job control turned off
/ # 
```

Try:
```
whoami
cat /proc/cpuinfo
cat /proc/mtd
```

**This is root, on the real stock rootfs, under a kernel you compiled yourself.**
No password, no flash writes, fully reversible — power-cycle the switch and it
boots back to the stock factory firmware exactly as before, since nothing above
ever touched flash.

---

## What's Next From Here

This guide gets you to a shell — it does not get you to a working OpenWrt port.
See `PROJECT_STATUS.md` for what's done vs. still open, and the main `README.md`'s
"OpenWrt Portability" section for the remaining work (backporting the platform
init code to a modern kernel, writing/adapting a switch fabric driver, etc.).
