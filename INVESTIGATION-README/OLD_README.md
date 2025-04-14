# K2 Pro Firmware Analysis Notes

This document summarizes the analysis performed on the firmware image extracted into the `_firmware.img.extracted` directory.

## Environment & Tools

*   **OS:** WSL Ubuntu (on Windows)
*   **Key Tools Used:** `binwalk`, `jefferson` (for JFFS2), `sasquatch` (for SquashFS)

## Firmware Type

The firmware was identified as an **embedded Linux system**. Key indicators included:

*   **Filesystems:** Presence of `531800.squashfs` (SquashFS) and `1031C00.jffs2` (JFFS2).
*   **Kernel Directory:** A directory named `4.9.118/` suggesting Linux kernel v4.9.118 components.
*   **Init System:** Files following System V init script naming conventions (e.g., `S10udev`, `K80udev`).
*   **Configuration:** Standard Linux configuration files (`.rules` for udev, `.conf` for ALSA, etc.).

## Finding the Boot Logo

The boot logo was not found as a standard image file within the primary extracted filesystems (`squashfs-root` or `jffs2-root`).

1.  **Configuration Hint:** The file `_firmware.img.extracted/otacfg.xml` contained a line indicating a dedicated partition for the logo:
    ```xml
    <item partition="bootlogo" mainType="12345678" subType="BOOTLOGO_FEX0000" upgrade="1" mode="0"/>
    ```
2.  **Binary Analysis:** The `binwalk` tool was used to analyze the large binary blobs:
    *   `_firmware.img.extracted/29E320`: Appeared to contain kernel/driver-related data, no image signatures found.
    *   `_firmware.img.extracted/29E243.xz`: Although `unxz` reported errors, `binwalk` identified embedded SquashFS, JFFS2 filesystems, and importantly, **JPEG image data** starting at **decimal offset 14434749** (hex `0xDC41BD`).
3.  **Extraction:** The `dd` command was used to extract the JPEG data from the offset identified by `binwalk`:
    ```bash
    dd if=./_firmware.img.extracted/29E243.xz of=./_firmware.img.extracted/potential_bootlogo.jpg bs=1 skip=14434749
    ```
4.  **Result:** The extracted file `_firmware.img.extracted/potential_bootlogo.jpg` was confirmed to be the correct boot logo.

## Finding the Shutdown Logo

The shutdown logo was found as a standard PNG file within the extracted SquashFS filesystem.

1.  **Search:** The `find` command was used to locate files named `shut_down.png`:
    ```bash
    find ./_firmware.img.extracted -name shut_down.png -ls
    ```
2.  **Result:** Multiple instances were found, likely duplicates from the extraction process. The primary location within the main filesystem appears to be:
    *   `_firmware.img.extracted/squashfs-root/data/res/pic/shut_down.png`

## Summary of Key Files

*   **Boot Logo:** `_firmware.img.extracted/potential_bootlogo.jpg` (Extracted from `_firmware.img.extracted/29E243.xz` @ offset `14434749`)
*   **Shutdown Logo:** `_firmware.img.extracted/squashfs-root/data/res/pic/shut_down.png`
*   **Firmware Structure Hint:** `_firmware.img.extracted/otacfg.xml`
*   **Root Filesystem (Read-Only):** `_firmware.img.extracted/531800.squashfs` (mounted at `_firmware.img.extracted/squashfs-root/`)
*   **Data Filesystem (Writable):** `_firmware.img.extracted/1031C00.jffs2` (mounted at `_firmware.img.extracted/jffs2-root/`)
*   **Likely Kernel Image/Component:** `_firmware.img.extracted/29E320`
*   **Logo/Filesystem Container:** `_firmware.img.extracted/29E243.xz`

## Tutorial: Finding the Boot Logo (Detailed Steps)

This section details the steps taken to locate the boot logo image within the `firmware.img` file.

1.  **Initial Analysis & Filesystem Check:**
    *   The firmware was identified as embedded Linux.
    *   A search for common image files (`.png`, `.jpg`, etc.) and names (`logo`, `boot`, `splash`) within the extracted `squashfs-root` and `jffs2-root` directories did not reveal the main boot logo (using `find`).

2.  **Configuration File Hint (`otacfg.xml`):**
    *   The file `_firmware.img.extracted/otacfg.xml` (found during initial exploration or extracted by `binwalk -e`) contained a critical line:
      ```xml
      <item partition="bootlogo" ... />
      ```
    *   This indicated that the boot logo resides in a separate, dedicated partition or data blob named `bootlogo`, not as a regular file in the main filesystems.

3.  **Scanning Binary Blobs (`binwalk`):**
    *   The next step was to scan the large binary files extracted by `binwalk` (or present in the original firmware image) for image signatures. The most likely candidates were the files corresponding to the data chunks identified by `binwalk` before the main filesystems.
    *   The command `binwalk firmware.img` (or `binwalk _firmware.img.extracted/29E243.xz` if analyzing the previously extracted chunk) was used.
    *   `binwalk`'s output for the relevant data chunk (in this case, contained within `29E243.xz` or the corresponding section of `firmware.img`) showed:
      ```
      DECIMAL       HEXADECIMAL     DESCRIPTION
      --------------------------------------------------------------------------------
      ... (other data)
      14434749      0xDC41BD        JPEG image data, EXIF standard
      ... (other data)
      ```
    *   This identified JPEG data starting at byte offset `14434749` within that specific chunk (`29E243.xz`).

4.  **Extracting the Image Data (`dd`):**
    *   The `dd` command was used to precisely copy the bytes starting from the identified offset.
    *   Assuming the relevant data chunk was saved as `_firmware.img.extracted/29E243.xz`:
      ```bash
      dd if=./_firmware.img.extracted/29E243.xz of=./_firmware.img.extracted/potential_bootlogo.jpg bs=1 skip=14434749
      ```
      *   `if=...`: Input file containing the JPEG data.
      *   `of=...`: Output file to save the extracted JPEG.
      *   `bs=1`: Use a block size of 1 byte for accurate skipping.
      *   `skip=14434749`: Skip the specified number of bytes from the beginning of the input file.

5.  **Verification:**
    *   The resulting file, `_firmware.img.extracted/potential_bootlogo.jpg`, was opened with an image viewer and confirmed to be the boot logo.

---

## SDK Investigation (Part 1: Generic H6 SDK)

*   An Allwinner H6 SDK (`Allwinner-SDKs-H6-lichee-v1p1-linux-3.10`) based on the Lichee branch (v1.1) and Linux kernel 3.10 was examined.
*   **Conclusion:** This SDK is **not suitable** for the target firmware due to major mismatches in the target SoC (H6 vs. the firmware's likely `sun8iw19p1`) and Kernel Version (3.10 vs. the firmware's 4.9.118).
*   **Value:** Its structure (containing a `tools/pack/` directory and build scripts) provides a general example of how official Allwinner SDKs might be organized.

---

## SDK Investigation (Part 2: `yi-hack-Allwinner-v2` Project & Related Findings)

*   **New Clues:** A GitHub issue log related to Yi camera hacking confirmed the presence of firmware using the exact same **Linux kernel 4.9.118** and **GCC 6.4.1** compiler.
*   **SoC Hint:** The boot log from the issue provided an OP-TEE version string (`sun8iw19p1_v0.6.0`), strongly suggesting the target device uses an SoC from the **Allwinner `sun8iw19p1` family**.
*   **Relevant Project:** The [yi-hack-Allwinner-v2](https://github.com/roleoroleo/yi-hack-Allwinner-v2) project targets these specific cameras and kernel versions.
*   **Applicable Toolchain:** This project points to a downloadable toolchain based on `lindenis-org/lindenis-v536-prebuilt`, providing the necessary `arm-sunxi-musl` GCC (likely version 6.4.1).
*   **Applicable Build/Pack Scripts:** The `yi-hack-Allwinner-v2` project contains custom build (`./scripts/compile.sh`) and packaging (`./scripts/pack_fw.all.sh`) scripts. These scripts likely handle the specific requirements for building and packaging firmware for this platform (e.g., layout, headers, checksums).
*   **Conclusion:** Leveraging the toolchain and adapting the build/packaging scripts from the `yi-hack-Allwinner-v2` project is the most promising approach for modifying and rebuilding the target `firmware.img`.

---

## SDK Investigation (Part 3: Tina Linux Confirmation & R329 Correlation)

*   **OS Confirmation:** The banner file (`_firmware.img.extracted/banner`) definitively identifies the OS as **Tina Linux (Neptune, 5C1C9C53)**, which is based on OpenWrt.
*   **SoC Platform:** The `sun8iw19p1` platform identifier (from OP-TEE logs) strongly correlates with the **Allwinner R329** family, based on Tina SDK documentation and build guides for R329 that show the same "Neptune" banner string and use the 4.9 kernel.
*   **Build Process:** Guides for Tina Linux R329 confirm the standard build includes the **`pack` command** (typically run via `make pack`) to assemble the final `firmware.img`.
*   **Boot Logo Location:** User reports for Tina Linux (Neptune) confirm the boot logo/animation resides in its **own MTD partition**, not the root filesystem, aligning with our earlier `binwalk`/`dd` extraction method.
*   **Lichee RV Clarification:** The "Lichee" name also refers to Sipeed's RISC-V based boards (e.g., Lichee RV using the D1 SoC). While these *can* run Tina Linux (Neptune), they are distinct from the target device's ARM Cortex-A53 (`sun8iw19p1`/R329) platform.
*   **Debugging:** Related forum posts mention `adbd`, suggesting ADB access might be available on the target device.
*   **Product Version:** The file `/data/res/param/config_devmng.xml` specifies the software version as `TF790 11.3 2024-01-19 LG`.
*   **Conclusion:** The target device runs **Tina Linux 4.9.118** on an **Allwinner R329-family SoC (`sun8iw19p1`)**. Modification requires the **official Tina Linux SDK** for this platform/kernel to use the correct **`pack` command** and configuration. The `yi-hack-Allwinner-v2` project remains the best reference/alternative for the toolchain and packaging logic.

---

## SDK Investigation (Part 4: R329 SDK Cloned & Inspected)

*   **SDK Acquired:** The necessary components for the Tina Linux SDK targeting the R329 platform (Kernel 4.9.x) were successfully cloned from `dl.linux-sunxi.org` into the `R329/` directory.
*   **Packaging Script Found:** The main script responsible for assembling the final firmware image was identified as `R329/projects/scripts/pack_img.sh`.
*   **Configuration Identified:** The locations of key configuration files (`image*.cfg`, `sys_partition*.fex`, `toc*.fex`) used by the packaging script were found within the `R329/projects/device/config/` subdirectories.
*   **Build Tools Confirmed:** Standard utilities (`mkimage`, `mtd-utils`, etc.) and helper tool sources (`pack-bintools`) are present in the SDK structure.
*   **Next Steps:** The focus now shifts to understanding the specific configuration files (`.cfg`, `.fex`) used for the target device within the SDK and how to use the `pack_img.sh` script with modified components (like the logos) to rebuild the firmware.

---

## Known Credentials & Potential Issues

*   **Default Root Credentials:** Based on findings from similar Tina Linux devices, the default login appears to be `root` with the password `tc310`.
*   **Potential Custom SquashFS:** Be aware that analysis of related devices ([Ref: EEVblog Forum](https://www.eevblog.com/forum/projects/custom-squashfs-format-help-hacking-chinese-crap-podofo-10-26-android-auto/msg5551831/#msg5551831)) has shown that some Tina Linux firmware may use a **non-standard SquashFS superblock header**. If standard `unsquashfs` (or tools like `binwalk -e` that use it) fail to extract the root filesystem, it might be necessary to **patch and recompile `squashfs-tools`** based on the specific header format found. This is especially critical if attempting to **repack** a modified filesystem.

---

## Alternative Tool: `imgRePacker`

*   A third-party tool, `imgRePacker` ([Ref: XDA Thread](https://xdaforums.com/t/tool-imgrep...suits-firmware-images-unpacker-packer.1753473)), exists specifically for unpacking/repacking Allwinner `.img` firmware.
*   It is confirmed to successfully **unpack** firmware from the same device family (TF790/sun8iw19), extracting components like `bootlogo.fex` and `rootfs.fex` ([Ref: XDA Post](https://xdaforums.com/t/need-help-wrong-firmware-mirror-dashcam-tf790.4692681/#post-89869517)).
*   This tool offers a potentially **simpler method** for unpacking and attempting to repack the firmware, especially for basic modifications like changing logos, compared to using the full Tina SDK.
*   **Repacking requires caution:**
    *   User reports indicate potential difficulties with repacking using this tool. The official Tina SDK `pack_img.sh` script remains the more robust (but complex) option.
    *   It is **unknown** if `imgRePacker` handles the potential custom SquashFS format.
    *   **IMPORTANT:** By default, `imgRePacker` overwrites the original firmware file when repacking (`firmware.img.dump` -> `firmware.img`). To avoid losing the original:
        1.  **Backup:** `cp firmware.img firmware_original.img`
        2.  **Modify:** Make changes within the `firmware.img.dump/` directory (e.g., replace `bootlogo.fex`).
        3.  **Repack:** Run `imgRePacker.exe firmware.img.dump` (or Linux equivalent).
        4.  **Rename:** `mv firmware.img firmware_modified.img` (or a more descriptive name like `firmware_newlogo.img`).
        5.  You now have `firmware_original.img` and `firmware_modified.img`.

---

## Alternative Tool: DragonFace for CDR v1.1

*   Another tool, **DragonFace for CDR v1.1** (a Windows GUI application found in [Allwinner-Homlet GitHub](https://github.com/Allwinner-Homlet/H6-BSP4.9-tools/blob/master/tools_win/DragonFaceForCDR_V1.1_20171009.rar)), has been reported to work with TF790 firmware ([Ref: XDA Post](https://xdaforums.com/t/need-help-wrong-firmware-mirror-dashcam-tf790.4692681/#post-89889550)).
*   This tool reportedly allows **opening the `.img` file, replacing the `bootlogo.jpg` via its interface, and saving/repacking a functional `.img` file** suitable for flashing.
*   **Limitation:** It does **not** seem to support modifying the rootfs partition directly.
*   **Use Case:** May be the **simplest method specifically for changing the boot logo**, but likely not suitable for other firmware modifications.

---

## Important Note: Zlink Activation Keys

*   **Warning:** Users have reported that flashing firmware backups via USB bootloader methods (e.g., Allwinner FEL mode using `xfel`) can **erase the unique keys** required for Zlink (CarPlay / Android Auto), causing these features to stop working.
*   **Potential Reactivation:** A potential method for reactivating Zlink using a specific Chinese Android app and a phone hotspot named "zlink" has been documented ([Ref: singah.tistory.com post](https://singah.tistory.com/4089)). The reliability and long-term viability of this method are unknown.
*   **Recommendation:** Be extremely cautious when flashing entire firmware backups, especially from other devices. Prioritize methods that update specific partitions if possible, or investigate backing up device-specific key partitions before flashing if feasible. 