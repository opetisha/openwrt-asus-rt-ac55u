# OpenWrt for ASUS RT-AC55U

Unofficial OpenWrt **25.12.6** (ath79/nand, kernel 6.12) for the ASUS RT-AC55U, installable from the **stock ASUS web UI**. The RT-AC55U is not supported by upstream OpenWrt. This port adds it.

> ⚠️ Flashing third-party firmware is at your own risk. Everything below was tested on one RT-AC55U (stock firmware 3.0.0.4.382_52236), including install, upgrade and return to stock.

## What works

| | |
|---|---|
| SoC / RAM / flash | QCA9558 @ 720 MHz, 128 MiB, 128 MiB NAND |
| Ethernet | 4× LAN + WAN, gigabit (AR8337 switch, both CPU ports used) |
| Wi-Fi 2.4 GHz | ath9k (SoC), 2×2, factory calibration |
| Wi-Fi 5 GHz | ath10k-ct (QCA9882), 2×2 802.11ac, factory calibration |
| USB | USB 3.0 port (Etron EJ168) and USB 2.0 port |
| LEDs | all 9 LEDs |
| Buttons | reset, WPS |
| MAC addresses | same as stock (WAN `…F0`, LAN `…F4`) |
| Free space | ~13.8 MB for packages (UBIFS overlay) |

## Install (from stock firmware)

1. Download `openwrt-ath79-nand-asus_rt-ac55u-initramfs-factory.trx` and `openwrt-ath79-nand-asus_rt-ac55u-squashfs-sysupgrade.bin`, and check them against `sha256sums`.
2. In the ASUS web UI go to **Administration → Firmware Upgrade**, upload the `initramfs-factory.trx` and wait for the reboot.
3. Connect to a LAN port. OpenWrt runs from RAM at **http://192.168.1.1** (user `root`, no password).
4. Make it permanent: in LuCI go to **System → Backup / Flash Firmware → Flash new firmware image**, upload the `squashfs-sysupgrade.bin` and **untick "Keep settings"**. Or from SSH:
   ```sh
   sysupgrade -n /tmp/openwrt-ath79-nand-asus_rt-ac55u-squashfs-sysupgrade.bin
   ```
5. After the reboot, set a root password. Wi-Fi is disabled by default.

Later upgrades are normal OpenWrt sysupgrades of the `sysupgrade.bin`, and settings can be kept.

### Alternative: install over TFTP (serial console)
Stop autoboot with `4` at the bootloader menu, then:
```
tftpboot 0x81000000 openwrt-ath79-nand-asus_rt-ac55u-initramfs-factory.trx
bootm 0x81000000
```
The bootloader uses `ipaddr=192.168.1.1` and `serverip=192.168.1.10`. This boots OpenWrt from RAM without touching flash. Continue with step 4.

## Return to stock

Download the official firmware from ASUS (e.g. `RT-AC55U_3.0.0.4_382_52236-ga0f880d.trx`), copy it to `/tmp` and run:
```sh
sysupgrade -F -n /tmp/RT-AC55U_3.0.0.4_382_52236-ga0f880d.trx
```
This writes the stock image to both firmware slots and removes the OpenWrt volumes. On the next boot the ASUS bootloader recreates its own `lsdk`/`jffs2` volumes, so the flash layout is exactly as from the factory.

The bootloader's **Rescue Mode** (hold reset while powering on, then upload the stock image with ASUS Firmware Restoration) is never modified by this port and stays available.

## How it works

The ASUS bootloader (U-Boot 1.1.4) manages the UBI flash layout itself and enforces it on every boot. This port is designed around those rules:

| UBI volume | Use |
|---|---|
| `nvram`, `Factory`, `Factory2` | untouched (ASUS settings, Wi-Fi calibration, MACs) |
| `linux` | OpenWrt kernel. Must stay 48 MiB, otherwise the bootloader rebuilds it, so it is updated in place |
| `linux2` | fallback image. The bootloader copies it over `linux` if the kernel is ever corrupt |
| `rootfs`, `rootfs_data` | OpenWrt squashfs + overlay. They replace stock `lsdk`/`jffs2` and use all free space, so the bootloader cannot recreate those |

What else the port needed:
- The stock web UI checks an ASUS signature in the uImage name (product id, build numbers and a key byte derived from the image contents). `initramfs-factory.trx` carries it, built with OpenWrt's `asusuimage` (TRX v2).
- An lzma-loader wraps the kernel, because the 2014 bootloader cannot decompress large kernels.
- Calibration data and MACs are read from the `Factory` UBI volume via `CONFIG_MTD_UBI_NVMEM`.

The bootloader experiments are documented in [`HARDWARE.md`](HARDWARE.md).

## Building

Apply the patch to OpenWrt v25.12.6:
```sh
git clone -b v25.12.6 https://github.com/openwrt/openwrt.git && cd openwrt
git apply ../0001-ath79-add-support-for-ASUS-RT-AC55U.patch
./scripts/feeds update -a && ./scripts/feeds install -a
cat > .config <<'EOF'
CONFIG_TARGET_ath79=y
CONFIG_TARGET_ath79_nand=y
CONFIG_TARGET_ath79_nand_DEVICE_asus_rt-ac55u=y
CONFIG_TARGET_ROOTFS_INITRAMFS=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-ssl=y
EOF
make defconfig && make -j$(nproc)
```

## Known limitations
- About 20 MiB of the 128 MiB NAND is available to OpenWrt, because the bootloader insists on keeping two 48 MiB firmware slots.
- On every boot the bootloader prints a harmless `Create volume lsdk fail` (no free space, by design).
