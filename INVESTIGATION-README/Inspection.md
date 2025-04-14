# Firmware Inspection Summary (firmware.img)

This document summarizes the inspection findings after extracting `firmware.img` using `binwalk -e`.

## Inferred Development Environment & Tools

*   **OS:** **Tina Linux (Neptune, 5C1C9C53)** (Confirmed by `/etc/banner`). Based on OpenWrt.
*   **Build System:** **Tina Linux SDK** for the `sun8iw19p1` platform / Kernel 4.9.x.
*   **Base System:** OpenWrt (Foundation for Tina Linux).
*   **SoC:** **Allwinner sun8iw19p1** platform (ARM Cortex-A53). Correlates with **R329** family (based on Tina SDK docs and specific R329 guides showing kernel 4.9 and the Neptune banner).
*   **Bootloader:** U-Boot 2018.05.
*   **Kernel:** Linux **4.9.118**.
*   **Toolchain:** GCC **6.4.1** (OpenWrt/Linaro GCC 6.4-2017.11).
*   **Filesystem Tools:** `mksquashfs` (XZ), `mkfs.jffs2`, UBI tools.
*   **Compression:** `xz`.
*   **Packaging:** **Tina SDK `pack` command** (Confirmed by R329 Tina build guides).
*   **Debugging:** ADB daemon likely present (`adbd` service referenced in related forum posts).

## Analysis of Cloned R329 Tina SDK

*   **Cloning:** Successfully cloned the SDK components for the R329 platform (Kernel 4.9.x) from `dl.linux-sunxi.org` into the `R329/` directory. The SDK is structured as multiple Git repositories (`build.git`, `config.git`, `lichee/linux-4.9.git`, `lichee/brandy-2.0/u-boot-2018.git`, etc.).
*   **Packaging Script:** Located the main firmware image assembly script at `R329/projects/scripts/pack_img.sh`. This script orchestrates the packaging process based on input parameters (chip, platform, board).
*   **Configuration Files:** Identified the types and locations of key configuration files used by `pack_img.sh`:
    *   `image[_<platform>].cfg`: Defines image contents (found under `device/config/...`).
    *   `sys_partition[_<storage_type>].fex`: Defines MTD partition layout (found under `device/config/...`).
    *   `toc0.fex` / `toc1.fex`: Defines Bootloader/TOC config (found under `device/config/...`).
*   **Build Tools:** Confirmed presence of standard tools (`mkimage`, `mtd-utils`, `squashfs`) within `R329/projects/tools/` and source for helper tools like `merge_package` in `R329/projects/tools/pack-bintools/`.

## Device Specific Information

*   **Product Software:** `TF790 11.3 2024-01-19 LG` (From `/data/res/param/config_devmng.xml`).

## Known Credentials / Access Methods

*   **Default Root Credentials:** Found on similar Tina Linux devices: `root` / `tc310` (Hash: `$1$7ceqN2Tq$oWMQfYiVzLYccX59mc7C81`).
*   **Potential Access Vectors:**
    *   Serial Console (`ttyS0` likely, leading to `tina login:` prompt).
    *   Direct SPI NOR flash reading (e.g., via CH341a programmer).
    *   Allwinner FEL Mode (USB VID `1f3a` PID `efe8`, potentially requires shorting SPI flash pins).
    *   ADB (If `adbd` service is enabled).

## Potential Issues & Quirks

*   **Custom SquashFS Format:** Analysis of similar Allwinner/Tina devices ([Ref: EEVblog Forum](https://www.eevblog.com/forum/projects/custom-squashfs-format-help-hacking-chinese-crap-podofo-10-26-android-auto/msg5551831/#msg5551831)) revealed a **non-standard SquashFS superblock structure** (specifically, the `inodes` field was relocated). This may require **patching and recompiling `squashfs-tools` (`unsquashfs`, `mksquashfs`)** for successful extraction and critically for **repacking** the root filesystem.

## Alternative Tool: `imgRePacker`

*   **Identification:** A third-party tool ([Ref: XDA Forums Thread](https://xdaforums.com/t/tool-imgrep...suits-firmware-images-unpacker-packer.1753473)) specifically designed for unpacking and repacking Allwinner LiveSuit/PhoenixSuit `.img` firmware files.
*   **Relevance:** Confirmed by user reports ([Ref: XDA Post on TF790](https://xdaforums.com/t/need-help-wrong-firmware-mirror-dashcam-tf790.4692681/#post-89869517)) to successfully **unpack** firmware for the TF790 (`sun8iw19`) device family, extracting components like `bootlogo.fex` and `rootfs.fex`.
*   **Functionality:** Aims to both unpack and repack `.img` files, understanding the internal structure based on `.fex` configuration files.
*   **Potential Advantages:** Offers a potentially simpler workflow for basic modifications (like logo swaps) compared to setting up and using the full Tina SDK.
*   **Repacking Caveats:** Repacking success is not guaranteed (user reports indicate potential issues). It's unknown if it handles the potential custom SquashFS header correctly.
*   **Recommendation:** Worth trying for unpacking and potentially simple repacking *before* resorting to the full SDK. Use the Tina SDK `pack_img.sh` as the more reliable (though complex) fallback for repacking.

### Modifying Boot Logo with imgRePacker (Successful Example)

1.  **Unpack:** Run `imgRePacker.exe firmware.img` (or the Linux equivalent `imgrepacker`) from its directory. This creates a `firmware.img.dump/` directory containing extracted `.fex` files.
2.  **Identify & Copy Logo:** The `bootlogo.fex` file within `firmware.img.dump/` is typically the boot logo (often identified as a JPEG by the tool). Copy this file out (e.g., `cp firmware.img.dump/bootlogo.fex extracted_bootlogo.jpg`).
3.  **Modify:** Edit the copied logo file (`extracted_bootlogo.jpg`) using an image editor. Ensure the dimensions and format (JPEG) are compatible.
4.  **Replace:** Copy the modified logo back into the dump directory, replacing the original: `cp extracted_bootlogo.jpg firmware.img.dump/bootlogo.fex`.
5.  **Repack:** Run `imgRePacker.exe firmware.img.dump` (or Linux equivalent) from its directory. This will use the contents of the `.dump` directory (including the modified logo) to create a new `firmware.img` file, overwriting the original.
6.  **Result:** The repacked `firmware.img` should now contain the modified boot logo. (Note: Running the Windows `.exe` under WSL may show warnings about UNC paths, which seem to be ignorable if the process completes.)

## Alternative Tool: DragonFace for CDR v1.1

*   **Identification:** A Windows GUI tool found within an Allwinner tools repository ([Ref: Allwinner-Homlet GitHub](https://github.com/Allwinner-Homlet/H6-BSP4.9-tools/blob/master/tools_win/DragonFaceForCDR_V1.1_20171009.rar)) designed for modifying Allwinner `.img` firmware.
*   **Relevance:** Reported by a user ([Ref: XDA Post on TF790](https://xdaforums.com/t/need-help-wrong-firmware-mirror-dashcam-tf790.4692681/#post-89889550)) to successfully **modify the boot logo** and **repack a working firmware `.img` file** for the TF790 device family, suitable for flashing with tools like PhoenixCard.
*   **Functionality (Boot Logo Focus):**
    *   Allows opening the `.img` firmware.
    *   Provides an interface to browse/extract/replace the `bootlogo.jpg`.
    *   Can save the changes back into a valid `.img` file.
*   **Limitations:**
    *   Reportedly **cannot modify the rootfs partition** directly via its interface.
    *   Bundles standard `mksquashfs`/`unsquashfs` tools, but integrating a modified rootfs back into the `.img` using DragonFace is unclear.
    *   Uses temporary directories for some operations.
    *   The tool version is relatively old (v1.1 from 2017).
*   **Recommendation:** Presents a potentially **very simple and effective method specifically for changing the boot logo**, especially if other methods fail or seem too complex. May not be suitable for rootfs or other advanced modifications.

## Important Note: Zlink (CarPlay/Android Auto) Activation

*   **Risk:** Flashing firmware backups (especially someone else's) via USB bootloader tools (like `xfel`) may **erase or overwrite unique device keys** required for Zlink (CarPlay/Android Auto) functionality, resulting in messages like "Unregistered" or "BTunregistered".
*   **Potential Reactivation Method:** A method involving a specific Chinese Android application and connecting the device to a phone hotspot named "zlink" (password: `12345678`) has been shared ([Ref: singah.tistory.com blog post](https://singah.tistory.com/4089)). This process reportedly allows re-downloading activation keys. The long-term validity of the provided app login details and keys is unknown.
*   **Caution:** Modify and flash firmware with awareness of this risk. Backing up original, unique device partitions (if possible via methods like FEL or serial console) might be advisable before flashing, although the specific location of Zlink keys isn't confirmed.

## Filesystem Structure (`squashfs-root`)

*   Standard OpenWrt/Linux layout (`/bin`, `/sbin`, `/etc`, `/lib`, `/usr`, `/var`, `/tmp`, etc.).
*   OpenWrt specific: `/rom`, `/overlay` directories present.
*   Custom directories:
    *   `/data`: Contains application data, resources (`res/`), device config (`device.xml`), OTA config (`otacfg.xml`).
    *   `/www`: Indicates presence of a web server.
*   Init Process: Standard OpenWrt/BusyBox init (`/etc/inittab`), `preinit` script, `/etc/init.d/` startup scripts.

## Key Configuration & Services (`etc/`)

*   **OpenWrt Config:** `openwrt_release`, `openwrt_version`, `opkg/`.
*   **Init Scripts (`etc/init.d/`):**
    *   Standard: `cron`, `network`, `ntpd`, `udev`.
    *   Core: `rc.modules`, `rc.preboot`, `rc.final`, `rcS`, `rcK`.
    *   Custom/App: `S00mpp`, `S00part`, `S00pqd`, `S01app` (likely main device functions, media processing, picture quality, main app).
    *   Hardware: `S11dev`, `S50usb`.
*   **Networking:** `udhcpd.conf`, `wpa_supplicant.conf`.
*   **Hardware/Platform Specific:** `asound.conf` (ALSA), `cedarx.conf` (Allwinner multimedia), `disp_firmware`, `sunxi-keyboard.kl`.
*   **Device Info:** `device_info`, `zj.ini`.

## Application Resources (`data/res/`)

*   Structured resource directory:
    *   `audio/`, `font/`, `language/`, `layout/`, `pic/`, `style/`, `config/`, `param/`.
*   Suggests a graphical application with localization and customizable UI elements.

## Storage Layout

*   **Read-Only Root:** SquashFS (`531800.squashfs`).
*   **Writable Data:** JFFS2 (`1031C00.jffs2`) - Likely used for `/overlay` and potentially other persistent storage.
*   **Boot Logo:** Stored as raw JPEG data within the `29E243.xz` chunk (offset `14434749`). **Confirmed by related user reports to be in a separate MTD partition.**
*   **Kernel:** `uImage` format, likely the uncompressed data corresponding to `29E320` (needs verification, uImage header found at `0x29BC00`).

## Insights from Related `yi-hack-Allwinner-v2` Unbricking Guide

While targeting IP cameras, the unbricking guide for the `yi-hack-Allwinner-v2` project provides relevant insights due to shared platform characteristics (likely `sun8iw19p1` SoC, U-Boot, SPI flash):

*   **MTD Partitioning Confirmed:** The guide confirms the use of the Linux MTD (Memory Technology Device) subsystem for partitioning the SPI NOR flash. A typical layout includes partitions for `uboot`, `boot` (kernel/dtb), `rootfs`, `home`/`data`, `env`, `mfg`, etc. The exact layout for the target device needs confirmation.
*   **Offsets & Sizes are Critical:** The process of calculating flash offsets by summing previous partition sizes and using exact data lengths for U-Boot commands (`sf erase`, `sf write`) highlights the requirement for precise layout information when reconstructing firmware.
*   **Packaging vs. Raw Flash:** The unbricking guide uses raw partition dumps (e.g., `mtdblock4.bin`). The target `firmware.img` appears to be a *packaged* update file containing multiple components, possibly compressed or archived together (e.g., the `29E243.xz` chunk holding the logo and filesystems). SDK tools or custom packaging scripts (like `yi-hack-Allwinner-v2`'s `pack_fw.all.sh`) are responsible for creating this packaged format correctly, embedding components according to the expected flash layout and potentially adding headers/checksums.
*   **U-Boot Access:** The guide demonstrates recovery/modification via U-Boot serial console access, suggesting a possible (though potentially difficult) access method for the target device.

## Overall Conclusion

The firmware is built using **Tina Linux (Neptune)**, an OpenWrt-based distribution tailored by Allwinner for the **sun8iw19p1 platform (likely R329 family)**. It uses Linux kernel **4.9.118** and standard components. Packaging and modification require the specific **Tina Linux SDK** for this platform and kernel version, using its **`pack` command**. The boot logo resides in a dedicated MTD partition. The `yi-hack-Allwinner-v2` project serves as a valuable reference for toolchain and packaging logic if the official SDK is unavailable. 