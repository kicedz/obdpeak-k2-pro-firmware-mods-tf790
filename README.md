# OBDPEAK K2 Pro TF790 Firmware Boot Logo Modification Guide

## DISCLAIMER

**The contents of this repository, including any firmware files, tools, or derived data, are provided for educational, research, and investigation purposes only.**

*   **Ownership:** I do not claim ownership of any original firmware, proprietary tools, or third-party code included or referenced herein. All rights belong to their respective owners.
*   **No Warranty:** Use the information and files in this repository **at your own risk**. Modifying firmware can damage your device. There is no guarantee of functionality, and I am not responsible for any damage caused by using the contents of this repository.
*   **Compliance:** Ensure your use complies with all applicable laws and terms of service related to your device and its software.

---

## Required Tools & Resources

This guide requires the following tools. It is recommended to obtain them from their original sources:

*   **imgRePacker:** For unpacking/repacking Allwinner `.img` firmware files.
    *   [XDA Developers Thread (English)](http://forum.xda-developers.com/showthread.php?t=1753473)
    *   [4PDA Forum Thread (Russian)](http://4pda.ru/forum/index.php?showtopic=287496&view=findpost&p=11406874)
*   **(Optional) Image Editor:** Any tool capable of editing JPEG files and saving them with specific dimensions (e.g., GIMP, Photoshop).
*   **(Optional) Filesystem Tools (for deeper analysis):**
    *   **sasquatch:** Modified `unsquashfs` tool for non-standard SquashFS formats. [Source (devttys0)](https://github.com/devttys0/sasquatch)
    *   **jefferson:** JFFS2 filesystem extraction tool. [Source (onekey-sec)](https://github.com/onekey-sec/jefferson/)

**Firmware:** You need the original `firmware.img` file for your device. This guide does not provide the firmware file itself.

---

This guide provides step-by-step instructions on how to modify the boot logo within the `firmware.img` for K2 Pro devices (TF790 family, Allwinner R329/sun8iw19p1) using the `imgRePacker` tool.

**Disclaimer:** Modifying firmware can potentially brick your device. Proceed with caution and at your own risk. Ensure you have a backup of your original firmware.

**Tool Required:**
*   `imgRePacker` (v2.0.7.8 or similar) - Windows or Linux version. Can be found in the `imgrepacker/` directory of this project.
    *   Note: The Windows version (`imgRePacker.exe`) was successfully used under WSL (Windows Subsystem for Linux) in testing.

## Steps to Replace Boot Logo:

1.  **Prepare Environment:**
    *   Place your target `firmware.img` file in the main project directory.
    *   Ensure the `imgrepacker` tool (either Linux binary or Windows `.exe`) is present in the `imgrepacker/` subdirectory.
    *   If using WSL and the Windows `.exe`, ensure it has execute permissions: `chmod +x imgrepacker/imgRePacker.exe`.

2.  **Backup Original Firmware:**
    *   Before making any changes, create a backup of your original firmware:
      ```bash
      cp firmware.img firmware_original.img
      ```

3.  **Unpack Firmware:**
    *   Navigate into the `imgrepacker` directory and run the tool, pointing it to the firmware image in the parent directory:
      ```bash
      cd imgrepacker
      ./imgRePacker.exe ../firmware.img # Use ./imgrepacker for the Linux binary
      ```
    *   This will create a new directory named `../firmware.img.dump/` (i.e., `firmware.img.dump/` in the main project directory) containing the extracted firmware components (`.fex` files).

4.  **Identify & Prepare New Logo:**
    *   Locate the `bootlogo.fex` file inside the `firmware.img.dump/` directory. This is the current boot logo, typically a JPEG image.
    *   Create or prepare your new boot logo image.
    *   **CRITICAL:** Your new logo **must** have the **exact same dimensions (width x height)** as the original `bootlogo.fex`. You can check the dimensions of the original by copying it to a `.jpg` file (`cp firmware.img.dump/bootlogo.fex temp_logo.jpg`) and opening it in an image viewer or editor.
    *   It's also recommended to keep the file format as JPEG and the file size reasonably close to the original. Reducing JPEG quality (if needed to reduce file size) is preferable to changing dimensions.
    *   Save your prepared logo as a `.jpg` file (e.g., `new_bootlogo.jpg`).

5.  **Replace Logo in Dump Directory:**
    *   Copy your new logo file into the dump directory, renaming it to overwrite the original `bootlogo.fex`:
      ```bash
      # Ensure you are in the main project directory first (use 'cd ..' if still in imgrepacker/)
      cp new_bootlogo.jpg firmware.img.dump/bootlogo.fex
      ```

6.  **Repack Firmware:**
    *   Navigate back into the `imgrepacker` directory (if you left it) and run the tool again, this time pointing it to the dump directory:
      ```bash
      cd imgrepacker
      ./imgRePacker.exe ../firmware.img.dump # Use ./imgrepacker for Linux
      ```
    *   The tool will repack the contents of the dump directory. **Note:** This process will **overwrite** the `firmware.img` file in your main project directory.
    *   (Ignore potential `CMD.EXE`/`UNC path` warnings if running the `.exe` under WSL).

7.  **Rename Repacked Firmware:**
    *   Rename the newly created `firmware.img` to something descriptive to avoid confusion with the original backup:
      ```bash
      # Ensure you are in the main project directory first (use 'cd ..' if still in imgrepacker/)
      mv firmware.img firmware_newlogo.img
      ```

8.  **Done:**
    *   You now have:
        *   `firmware_original.img` (Your untouched backup)
        *   `firmware_newlogo.img` (The repacked firmware with your new logo)
    *   You can now attempt to flash `firmware_newlogo.img` to your device using appropriate tools (e.g., PhoenixCard).

---

*For more detailed background, analysis of the firmware structure, alternative tools, and SDK information, please see the documentation files within the `INVESTIGATION-README/` directory.*