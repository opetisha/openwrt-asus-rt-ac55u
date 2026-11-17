# ASUS RT-AC55U — hardware notes (gathered from stock fw 3.0.0.4.382_52236)

- SoC: QCA9558 rev 0 (MIPS 74Kc 720 MHz, DDR 600 MHz), 128 MiB RAM
- Flash: Winbond NAND 0xef/0xf1 (128 MiB SLC, 2 KiB page, 128 KiB block, 64 B OOB), boots from NAND
- Bootloader: ASUS U-Boot 1.1.4-g5589c4e5 (2014-07-24), has TFTP Rescue Mode (accepts ASUS trx or full UBI image), dual image fallback
- Switch: AR8327 on eth0 (GMAC0, PHY id 004dd036); eth1 present (GMAC1)
- WiFi 2.4 GHz: QCA9558 WMAC (ath9k); 5 GHz: QCA988x 168c:003c on PCIe (ath10k)
- USB3: Etron EJ168 1b6f:7023 xHCI on 2nd PCIe bus
- LEDs: 19 power (active low), 20 WPS (low), 18 LAN (low), 15 WAN blue (high), 14 WAN red (high), 13 2.4G (high), 0 5G (high), 1 USB3 (high), 4 USB2 (high). Buttons: 17 reset, 16 WPS (both active low). ASUS nvram gpio encoding: 0x1000 = active low.
- LAN1-4 = switch ports 2-5, WAN = port 1; CPU: port 0 (SGMII, eth1, LAN), port 6 (RGMII, eth0, WAN)

## NAND layout (raw)
| offset      | size      | name       |
|-------------|-----------|------------|
| 0x0000000   | 0x00e0000 | Bootloader |
| 0x0100000   | 0x7e00000 | UBI_DEV    |
| 0x7f00000   | 0x0100000 | caldata (old raw ART copy) |

## UBI volumes in UBI_DEV (LEB 126976)
nvram (1 LEB), Factory (2), Factory2 (2), linux (397), linux2 (397), lsdk (42), jffs2 (123)
- `linux` = kernel uImage followed by squashfs (stock kernel finds rootfs at 0x14b140)
- `Factory` = ART: 2.4G cal @0x1000 (MAC <WAN MAC> @0x1002), 5G cal @0x5000 (MAC @0x5006), stored twice (2nd copy at +0x1f000)

Kernel cmdline (stock): `board=AP135 console=ttyS0,115200 root=/dev/mtdblock6 rootfstype=squashfs mem=128M ubi.mtd=UBI_DEV`



## Bootloader behaviour (verified by experiment, 2026-11-17)
U-Boot env: ipaddr=192.168.1.1 serverip=192.168.1.10. Menu at boot: 1=TFTP to RAM, 2=TFTP to flash, 3=boot flash, 4=CLI (`ath>`).
CLI has `ubi part|info|create|remove|write|read`, `tftpboot`, `bootm`, `nand`.

On every boot U-Boot checks the UBI layout:
- `linux` smaller than 0x3013000 → reads its content, removes `jffs2` + `linux`, recreates `linux` at 0x3013000 and writes content back, recreates `jffs2` with max size.
- missing `lsdk`/`jffs2` → tries to recreate; fails harmlessly ("No space left") when no free LEBs.
- extra volumes (e.g. `rootfs`, `rootfs_data`) are left alone.
- removing `Factory` is inhibited by U-Boot itself.

Boot (option 3): verify `linux2` (hdr + data CRC) → verify `linux`; if `linux` bad and `linux2` good → copies `linux2` over `linux` → boots `linux` (loaded at 0x81000000, load/entry must be 0x80060000).

Stock image: `RT-AC55U_3.0.0.4_382_52236-ga0f880d.trx` (md5 cd96849a23bc5ccd82141f27cda20ae0) == uImage(lzma kernel + squashfs), ASUS tail in ih_name: `03 00 00 04 "RT-AC55U"`, sn/en at 0x30, rootfs offset in last 3 bytes.

OpenWrt layout chosen: nvram, Factory, Factory2, linux (OpenWrt kernel, kept 0x3013000), linux2 (fallback image: stock fw, or the initramfs after a web UI install), rootfs, rootfs_data (max). `lsdk`/`jffs2` removed.
