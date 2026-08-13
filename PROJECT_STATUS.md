# PROJECT STATUS — D-Link DGS-1210-48 D1 Reverse Engineering

Last updated: session covering UART access through custom kernel live boot.

---

## Summary

Full hardware/firmware reverse engineering is essentially complete. A self-compiled
Linux 2.6.31 kernel, built from D-Link's own published GPL source, boots this exact
device end-to-end and reaches a live root shell. OpenWrt porting has not started —
this document tracks what's done, what's in progress, and what's genuinely open.

---

## ✅ DONE

### Access & Extraction
- [x] UART physical access identified and working (3.3V, 115200/230400 baud, ttyS0)
- [x] U-Boot 1.1.4 shell access confirmed, full command inventory documented
- [x] Full 16MB NOR flash dumped via `md.b` + UART capture + custom reconstruction script
- [x] Flash dump validated against binwalk (u-boot, uImage, JFFS2 rootfs all located)
- [x] Rootfs extracted from flash dump (jefferson/JFFS2)
- [x] Factory `DGS12XX` application binary extracted from rootfs
- [x] D-Link official GPL source code located and downloaded (firmware v4.10.023, D1,
      exact match to hardware) — from D-Link's public S3 bucket
- [x] GPL source extracted: full kernel 2.6.31 source, ASDK platform driver source,
      board config (`b2b_test`), toolchain (`qca-music_toolchain`)

### Hardware Identification
- [x] CPU: MIPS 24Kc V8.5, codename "Music" — **confirmed live** via `/proc/cpuinfo`
      on a self-booted kernel (previous AMD Alchemy Au1210 hypothesis was wrong,
      based only on a PRId table coincidence)
- [x] Switch fabric slaves (×6): QCA8059-AL1C — confirmed by physical chip marking
      (one heatsink decapped)
- [x] Switch fabric master: strong candidate QCA8519-AC2C, codename "Vivo" — cross-
      referenced against MikroTik CRS226/CRS210 product pages; revision B0 confirmed
      by live register read (`0x18800000` = `0x02022002`)
- [x] Flash: Macronix MX25L12835FMI, SPI NOR 16MB — confirmed by chip marking
- [x] RAM: Nanya NT5CB128M8FN-DH, DDR3 128MB — confirmed by chip marking
- [x] I2C mux: NXP PCA9545A — confirmed by chip marking
- [x] GPIO expanders: NXP PCA9555 ×2 — confirmed by chip marking
- [x] EEPROM: Atmel/Microchip AT24C series — confirmed by chip marking
- [x] Temp sensor: Microchip TCN75AVOA — confirmed by chip marking
- [x] Logic ICs: 74HC123D ×4 (monostable), 74HC08D (AND gate) — confirmed by marking
- [x] Power management ICs identified: Richtek RT8120B, ITE IT76820M, UTC 2SB772L,
      Niko-Sem/UBIQ QM3004D
- [x] Board ODM identified: Alpha Networks Incorporation (GPL source copyright)
- [x] Full flash partition layout confirmed (u-boot / u-boot-env / uImage / roofs)

### Software / Kernel Build
- [x] Toolchain (`mips-linux-uclibc-gcc 4.3.3`) working, cross-compiles successfully
- [x] Full kernel 2.6.31 compiled from GPL source, unmodified board config
- [x] BusyBox 1.19.4 cross-compiled statically (for future initramfs use)
- [x] **Custom kernel boots on real hardware end-to-end, reaches live root shell**
- [x] Live shell confirmed: `/proc/cpuinfo`, `/proc/mtd` read directly from hardware
- [x] Switch-core register read live from kernel space (`0x18800000`)
- [x] Real stock JFFS2 rootfs mounts correctly under the custom kernel
- [x] MDIO transaction protocol fully documented from GPL source
      (`music_mdio_base.c`) — register base, read/write/poll sequence, all in clear C
- [x] Memory-mapped switch-core register region documented (`0x18800000`, 8MB)
- [x] Known stock-firmware bug diagnosed (management IP loss — `udhcpc` background
      process, no restart mechanism) with a known workaround (static IP)

### Documentation
- [x] Full bilingual (EN/FR) README with hardware table, firmware breakdown, boot
      process, flash layout, GPL source findings, kernel build log, OpenWrt
      assessment, open questions, contribution guide
- [x] Complete U-Boot command reference document
- [x] Flash dump reconstruction tool (`uboot_mdb_to_bin.py`)
- [x] OpenWrt forum post drafted, ready to publish

---

## 🔄 IN PROGRESS / PARTIALLY DONE

- [ ] **QCA8519-AC2C identity — not 100% certain.** Strong circumstantial case
      (MikroTik cross-reference + live revision-register read matching "Vivo" B0),
      but the chip itself has not been visually/physically confirmed (heatsink not
      removed — epoxy-glued, real risk of damage). A teardown photo comparison
      with a MikroTik CRS210/CRS226 would settle this without further risk.
- [ ] **initramfs boot path — built but not yet actually used for anything.**
      BusyBox statically compiled and packaged as `initramfs.cpio.gz`/`uInitrd`,
      but the kernel `.config` doesn't have `CONFIG_BLK_DEV_INITRD` enabled, so
      it's currently ignored at boot (`Initrd not found or empty`). Booting off
      the real stock rootfs with `init=/bin/sh` turned out to be a better/faster
      path to a shell, so this was never revisited. Would need re-enabling in
      `.config` and a kernel rebuild if a rootfs-independent shell is wanted later.
- [ ] **CPLD exact model — unidentified.** Present and functional (`DRV_CPLD_*`
      symbols confirm it's in active use), but no chip marking read yet.
- [ ] **RTC exact model — unidentified.** Same situation as CPLD.

---

## ❌ NOT STARTED

- [ ] No attempt yet to get `dgs_xal.ko`/`dgs_drv.ko` loading against the custom
      kernel (blocked on modversion CRC mismatch — would need either a kernel
      config that bit-for-bit matches the factory build, or rebuilding these
      modules from source, which isn't available — only precompiled `.ko` shipped)
- [ ] No attempt to write, adapt, or test any QCA8519/QCA8719 DSA driver
- [ ] No attempt to backport the "Music" platform init code to a modern kernel
      (5.x/6.x) — this is the actual core OpenWrt-porting task and hasn't begun
- [ ] No board DTS written
- [ ] No U-Boot GPL source request has been sent to D-Link/Alpha Networks
- [ ] No attempt to write a modern U-Boot port for this platform
- [ ] GitHub repo structure created locally but not yet confirmed pushed/public
      (verify before linking it from the forum post)
- [ ] OpenWrt forum post drafted but not yet published

---

## Key Artifacts Produced This Session

| Artifact | Location / Status |
|----------|-------------------|
| Full flash dump (16MB) | `flash_dump.bin` |
| Extracted stock rootfs | `rootfs_extracted/` |
| D-Link GPL source (v4.10.023, D1) | `gpl-sources/dgs-1210-4.10.023_48_GPL/` |
| Custom-compiled working kernel (uImage) | `build/dgs-1210-4.10.023_48_GPL/image/knl_img_switchreg` (or latest build) |
| Statically-linked BusyBox | `build/busybox-1.19.4/_install/` |
| Flash dump reconstruction script | `tools/uboot_mdb_to_bin.py` |
| README (bilingual) | ready for GitHub |
| U-Boot command reference | ready for GitHub `docs/` |
| OpenWrt forum post draft | ready to publish |

---

## Suggested Next Steps (in priority order)

1. **Push the GitHub repo** if not already done — everything above is ready to go in
2. **Publish the OpenWrt forum post** to start attracting outside help
3. **Attempt physical confirmation of the QCA8519-AC2C** via a teardown photo
   comparison (no further risk to the board)
4. **Re-enable `CONFIG_BLK_DEV_INITRD`** and confirm the initramfs path actually
   works, as a fallback shell method independent of the stock rootfs
5. **Start the real OpenWrt-porting work**: backporting "Music" platform init code
   to a modern kernel — this is the long pole and the item most likely to benefit
   from outside contributors found via the forum post
