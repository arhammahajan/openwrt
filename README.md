![OpenWrt logo](include/logo.png)

OpenWrt Project is a Linux operating system targeting embedded devices. Instead
of trying to create a single, static firmware, OpenWrt provides a fully
writable filesystem with package management. This frees you from the
application selection and configuration provided by the vendor and allows you
to customize the device through the use of packages to suit any application.
For developers, OpenWrt is the framework to build an application without having
to build a complete firmware around it; for users this means the ability for
full customization, to use the device in ways never envisioned.

Sunshine!

---

## D-Link AQUILA PRO AI M30 A1 (`ubootmod` Merged Partitions)

This branch (`dlink-m30-custom`) is based on **OpenWrt 25.12** (`openwrt-25.12`) and adds support for the **D-Link AQUILA PRO AI M30 (A1)** with a merged partition layout (`ubootmod`), ported from kszaq's `openwrt-24.10-dlink-m30-cp-experiment` branch and adapted for OpenWrt 25.12.

### Background & Motivation

Stock D-Link M30 firmware splits the flash storage across two redundant UBI partitions (`ubi` at `0x580000` and `ubi1` at `0x3780000`), each sized at 50 MB (0x3200000). While the stock OpenWrt target preserves this layout, it severely restricts available user space for rootfs and installed packages to under 50 MB.

This custom `ubootmod` target squashes both consecutive 50 MB partitions into a single **100 MB** (`0x6400000` / 102,400 KiB) UBI partition, doubling the usable storage capacity.

### Summary of Changes over OpenWrt 25.12

| Component | Path | Description |
| :--- | :--- | :--- |
| **DTS Split** | [`target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1.dtsi`](target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1.dtsi) | Refactored shared MT7981B hardware configuration (Ethernet GMACs, MT7531 switch ports, SPI-NAND flash, GPIO keys, I2C LED controller, and WiFi nvmem cell bindings) into a reusable DTSI. |
| **Stock DTS** | [`target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1.dts`](target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1.dts) | Preserved the stock dual-partition DTS by including the DTSI and declaring the stock 50 MB `ubi` + `ubi1` partitions. |
| **Merged DTS** | [`target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1-ubootmod.dts`](target/linux/mediatek/dts/mt7981b-dlink-aquila-pro-ai-m30-a1-ubootmod.dts) | Added custom DTS declaring the merged 100 MB (`0x6400000`) `ubi` partition starting at `0x580000`. |
| **Target & Image Recipes** | [`target/linux/mediatek/image/filogic.mk`](target/linux/mediatek/image/filogic.mk) | Added target profile `dlink_aquila-pro-ai-m30-a1-ubootmod`: builds `sysupgrade.bin` and stock web recovery compatible `recovery.bin` (using `dlink-ai-recovery-header DLK6E6110001` padded to 51200k). |
| **Platform Upgrade Scripts** | [`target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh`](target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh) | Added `dlink_initial_setup()` called during initramfs pre-upgrade: updates `fw_setenv mtdparts` to register the 100 MB `ubi` partition and sets `fw_setenv mupgrade_en 0` to disable stock dual-boot / auto-upgrade fallback. |
| **Network Configuration** | [`target/linux/mediatek/filogic/base-files/etc/board.d/02_network`](target/linux/mediatek/filogic/base-files/etc/board.d/02_network) | Added `dlink,aquila-pro-ai-m30-a1-ubootmod` interface definitions (`lan1 lan2 lan3 lan4` + `internet`). |
| **MAC Address Hotplug** | [`target/linux/mediatek/filogic/base-files/etc/hotplug.d/ieee80211/11_fix_wifi_mac`](target/linux/mediatek/filogic/base-files/etc/hotplug.d/ieee80211/11_fix_wifi_mac) | Configured WiFi MAC extraction from `Odm` partition offset `0x81`. |
| **U-Boot Tools** | [`package/boot/uboot-tools/uboot-envtools/files/mediatek_filogic`](package/boot/uboot-tools/uboot-envtools/files/mediatek_filogic) | Added environment mapping for `/dev/mtd1` (offset `0x0`, size `0x40000`). |

### Partition Comparison

* **Stock Layout (`mt7981b-dlink-aquila-pro-ai-m30-a1.dts`)**:
  * `ubi`: `0x580000` - `0x3780000` (50 MiB)
  * `ubi1`: `0x3780000` - `0x6980000` (50 MiB, read-only)
* **OpenWrt Merged Layout (`mt7981b-dlink-aquila-pro-ai-m30-a1-ubootmod.dts`)**:
  * `ubi`: `0x580000` - `0x6980000` (100 MiB)

### Building the Firmware

1. Configure feeds:
   ```bash
   ./scripts/feeds update -a
   ./scripts/feeds install -a
   ```
2. Select target configuration in `make menuconfig`:
   * **Target System**: `MediaTek Ralink ARM`
   * **Subtarget**: `Filogic 820 / MT7981 boards`
   * **Target Profile**: `D-Link AQUILA PRO AI M30 (OpenWrt partition layout)`
3. Compile:
   ```bash
   make -j$(nproc)
   ```
4. Output images will be generated in `bin/targets/mediatek/filogic/`:
   * `openwrt-mediatek-filogic-dlink_aquila-pro-ai-m30-a1-ubootmod-initramfs-recovery.bin`: For initial installation via stock D-Link emergency recovery web interface.
   * `openwrt-mediatek-filogic-dlink_aquila-pro-ai-m30-a1-ubootmod-squashfs-sysupgrade.bin`: For subsequent sysupgrades from OpenWrt.

---

## Download

Built firmware images are available for many architectures and come with a
package selection to be used as WiFi home router. To quickly find a factory
image usable to migrate from a vendor stock firmware to OpenWrt, try the
*Firmware Selector*.

* [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

If your device is supported, please follow the **Info** link to see install
instructions or consult the support resources listed below.

## 

An advanced user may require additional or specific package. (Toolchain, SDK, ...) For everything else than simple firmware download, try the wiki download page:

* [OpenWrt Wiki Download](https://openwrt.org/downloads)

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case
sensitive filesystem required). Cygwin is unsupported because of the lack of a
case sensitive file system.

### Requirements

You need the following tools to compile OpenWrt, the package names vary between
distributions. A complete list with distribution specific packages is found in
the [Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
documentation.

```
binutils bzip2 diff find flex gawk gcc-6+ getopt grep install libc-dev libz-dev
make4.1+ perl python3.7+ rsync subversion unzip which
```

### Quickstart

1. Run `./scripts/feeds update -a` to obtain all the latest package definitions
   defined in feeds.conf / feeds.conf.default

2. Run `./scripts/feeds install -a` to install symlinks for all obtained
   packages into package/feeds/

3. Run `make menuconfig` to select your preferred configuration for the
   toolchain, target system & firmware packages.

4. Run `make` to build your firmware. This will download all sources, build the
   cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen
   applications for your target system.

### Related Repositories

The main repository uses multiple sub-repositories to manage packages of
different categories. All packages are installed via the OpenWrt package
manager called `opkg`. If you're looking to develop the web interface or port
packages to OpenWrt, please find the fitting repository below.

* [LuCI Web Interface](https://github.com/openwrt/luci): Modern and modular
  interface to control the device via a web browser.

* [OpenWrt Packages](https://github.com/openwrt/packages): Community repository
  of ported packages.

* [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically
  focused on (mesh) routing.

* [OpenWrt Video](https://github.com/openwrt/video): Packages specifically
  focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

### Documentation

* [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
* [User Guide](https://openwrt.org/docs/guide-user/start)
* [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
* [Technical Reference](https://openwrt.org/docs/techref/start)

### Support Community

* [Forum](https://forum.openwrt.org): For usage, projects, discussions and hardware advise.
* [Support Chat](https://webchat.oftc.net/#openwrt): Channel `#openwrt` on **oftc.net**.

### Developer Community

* [Bug Reports](https://bugs.openwrt.org): Report bugs in OpenWrt
* [Dev Mailing List](https://lists.openwrt.org/mailman/listinfo/openwrt-devel): Send patches
* [Dev Chat](https://webchat.oftc.net/#openwrt-devel): Channel `#openwrt-devel` on **oftc.net**.

## License

OpenWrt is licensed under GPL-2.0
