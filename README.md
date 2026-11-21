# Droidian (API 28) for Huawei Honor 9 Lite (LLD-L31 / `hi6250`)

Unofficial community port of **Droidian (Halium 9 / API 28)** for the **Huawei Honor 9 Lite (`hi6250` / Kirin 659)** running **EMUI 9.1** vendor firmware.

Because this port is still incomplete and **EMUI 9.1 on Kirin 659 lacks standard `boot` and `ramdisk` partitions** (requiring the Halium initramfs to be unpacked directly onto the `system` partition while mounting `rootfs.img` from `f2fs` `userdata`), this adaptation is maintained as a standalone repository rather than an upstream Droidian device package.

---

## Upstream Sources & Release Artifacts

### Upstream Components Used
* **Kernel (`4.9.337`):** Built from [Iceows/android_kernel_huawei_hi6250 (`Phenix-4.9.337`)](https://github.com/Iceows/android_kernel_huawei_hi6250/tree/Phenix-4.9.337)
* **Rootfs (`API 28`):** Adapted from the latest `arm64` API 28 rootfs at [droidian-images/droidian Releases](https://github.com/droidian-images/droidian/releases)
* **Initramfs:** Adapted from [Halium/initramfs-tools-halium (`continuous`)](https://github.com/Halium/initramfs-tools-halium/releases/tag/continuous)

### Files Included in This Release

| File | Description |
| :--- | :--- |
| `Phenix_Kirin65x-4.9.337-permissive-genroot.img` | Prebuilt permissive kernel image for EMUI 9.1 (`kernel` partition) |
| `Phenix_Kirin65x-4.9.337-permissive-genroot-cmdline.txt` | Kernel command-line arguments embedded in the boot image |
| `halium.config` | Kernel `defconfig` used to build the kernel image (alognside `phenix_defconfig` from Iceows repo) |
| `halium_EMUI9.1-initramfs.tar` | Customized Halium initramfs (with `f2fs` userdata, `mmcblk0p59` fallback, SELinux masking, and `vendor`/`odm`/`product` mounts) |
| `rootfs-hi6250-release.img` | Pre-patched Droidian API 28 ext4 rootfs image |
| `TWRP-9.1.img` | TWRP recovery image compatible with EMUI 9.1 partition layout |

---

### Repository Files

| File | Description |
| :--- | :--- |
| `hi6250-droidian-adaptation.tar.gz` | Overlay archive of all custom services, scripts, udev rules, configs, patched binaries, and cached `.deb` packages applied to `rootfs.img` |
| `libselinux.so.1.orig` / `libselinux.so.1.patched` | Original and binary-patched `libselinux.so.1` (`/usr/lib/aarch64-linux-gnu/libselinux.so.1`) |
| `patch_libselinux.so.1.py` | Binary patcher script that replaces the `selinuxfs` string with `selinuxNO` in `libselinux.so.1` so `systemd` boots cleanly with a permissive SELinux kernel |
| `mtp-configfs.orig` / `mtp-configfs.patched` | Original and patched `/usr/sbin/mtp-configfs` script adapted for Huawei's `hisi-usb-otg` USB gadget controller |

---

## What Works & Included Adaptation Tweaks

* **Display & Compositor:** Phosh / `phoc` running over `HWCOMPOSER-1` at `2x` UI scaling (`xwayland=false`).
* **CPU, GPU & Memory Tuning:**
  * Screen-aware Mali-T830 GPU frequency scaling (`680 MHz` minimum when screen is ON for smooth 60fps UI, dropping to `0` / `120 MHz` idle when screen is OFF to prevent overheating).
  * `interactive` CPU governor across all cores and full-RAM `lz4` ZRAM swap (`zram-swap.service`, eMMC `/swapfile` removed).
  * Disabled background `tracker-miner-fs-3`, `gnome-software` autostart, `ofono2mm`, and Android's runaway `activity_recognition` HAL.
* **Power Button, Volume Keys & Flashlight:**
  * `huawei-powerkey.service` intercepts `hisi_on` (`KEY_POWER`) so pressing Power toggles the LCD backlight (`0` ↔ `2000`), locks Phosh, and drops GPU clocks without triggering `hwcomposer@2.2`'s unrecoverable deep-sleep state.
  * Hardware Volume Up / Down keys control PulseAudio volume.
  * `huawei-flashlightd.py` provides `org.droidian.Flashlightd` directly over `/sys/class/leds/torch/brightness` so Phosh's Quick Settings Flashlight toggle works.
* **Wi-Fi & Networking:**
  * Managed by `iwd` + `iwgtk` (`wpa_supplicant` masked to prevent Hi1102 `p2p-dev-wlan0` kernel deadlocks).
  * Android `PARANOID_NETWORK` groups (`aid_inet`, `aid_net_raw`) and `APT::Sandbox::User "root"` configured so `apt`, browsers, and networking work out of the box.
* **USB Networking:** USB RNDIS Gadget (`10.15.19.82`) enabled via `mtp-configfs@rndis.service`.

---

## Known Limitations & Potential Fixes

| Feature | Current Status | Root Cause & Possible Fix |
| :--- | :--- | :--- |
| **Internal Speaker & 3.5mm Audio** | **Not working** (routed to safe software sink to avoid kernel panics) | Huawei's 64-bit `/vendor/lib64/hw/audio.primary.hisi.so` uses proprietary EMUI 9.1 C struct offsets that cause `module-droid-card` to `SIGSEGV`, while opening raw Linux ALSA (`hw:0,x`) panics the kernel because Huawei's `hi6555c` HiFi DSP mailbox isn't initialized. **Possible fixes:** Use Bluetooth (`A2DP`) audio or a USB-C DAC adapter; or write a `libhybris` C struct shim / HIDL `android.hardware.audio@4.0` bridge for `audio.primary.hisi.so`. |
| **Camera** | **Not working** | `/android/system/lib64/libdroidmedia.so` links against Android 10's `libstagefright_ccodec.so` instead of Android 9's `libstagefright_omx.so`. **Possible fix:** Rebuild `droidmedia` specifically against Android 9 (`pie` / API 28) headers/libraries. |
| **Wayland Screenshots** | **Blank / white PNGs** | `composer@2.2-service` holds `/dev/fb0` open exclusively (blocking `fbgrab`) and Mali-T830 Android blobs do not expose `EGL_EXT_image_dma_buf_import` for `wlroots` screencopy. **Possible fix:** Patch `hisifb` (`hisi_fb_open` / `hisi_fb_read`) in the kernel to allow read-only `/dev/fb0` access while `composer@2.2` is active. |
| **Cellular / Mobile Data** | **Untested / Disabled** | `ofono2mm` was masked due to high CPU polling load on Kirin RIL. **Possible fix:** Unmask `ofono2mm.service` and configure `ofono` / `ril` quirks for Huawei Balong/Kirin modem. |

---

## Installation Guide

> **Warning:** Back up all important data and partitions before proceeding. Ensure your bootloader is unlocked and your device is running **EMUI 9.1** firmware.

### 1. Boot or Flash TWRP for EMUI 9.1
Install the latest TWRP for EMUI 9.1 (or use `TWRP-9.1.img` from this release):
```bash
fastboot flash recovery_ramdisk TWRP-9.1.img
```
* **Note:** Touchscreen input may not work inside TWRP on some Honor 9 Lite panels—perform all recovery steps via **`adb shell`**.

### 2. Back Up Your Current Kernel
In TWRP (`adb shell`), back up your stock `kernel` partition to your PC before flashing:
```bash
adb shell "dd if=/dev/block/bootdevice/by-name/kernel of=/tmp/kernel-backup.img"
adb pull /tmp/kernel-backup.img ./
```

### 3. Flash the Halium Kernel
Reboot into bootloader (`adb reboot bootloader`) and flash the permissive kernel image:
```bash
fastboot flash kernel Phenix_Kirin65x-4.9.337-permissive-genroot.img
```

### 4. Enter TWRP Recovery
Unplug the USB cable, run `fastboot reboot` (or hold **Volume Up + Power** immediately as the screen turns off) until the phone boots into TWRP, then reconnect USB.

### 5. Unpack `halium_EMUI9.1-initramfs.tar` Onto `system`
Because EMUI 9.1 has no separate `ramdisk` partition, the initramfs lives on the `system` partition:
```bash
# Check the exact block device for system
adb shell "ls -lah /dev/block/bootdevice/by-name/system"

# Mount system, wipe existing Android system files (keeping lost+found), and unpack initramfs
adb push halium_EMUI9.1-initramfs.tar /tmp/
adb shell "mkdir -p /tmp/sys && mount /dev/block/bootdevice/by-name/system /tmp/sys"
adb shell "find /tmp/sys -mindepth 1 -maxdepth 1 ! -name 'lost+found' -exec rm -rf {} +"
adb shell "tar -xvf /tmp/halium_EMUI9.1-initramfs.tar -C /tmp/sys"
adb shell "umount /tmp/sys"
```

### 6. Copy & Resize `rootfs-hi6250-release.img` on `/data`
`/data` (`f2fs`) is normally already mounted in TWRP. Remove old Android data directories while preserving Huawei encryption/storage folders, copy the rootfs image, and expand it (typically ~18 GB depending on internal storage):
```bash
# Clean /data while preserving essential folders
adb shell "find /data -mindepth 1 -maxdepth 1 \
  ! -name 'unencrypted' \
  ! -name 'sec_storage_data' \
  ! -name 'media' \
  ! -name 'cache' \
  ! -name 'lost+found' \
  -exec rm -rf {} +"

# Push rootfs image to /data/rootfs.img
adb push rootfs-hi6250-release.img /data/rootfs.img

# Check and resize rootfs.img (e.g., 18G)
adb shell "e2fsck -f -y /data/rootfs.img && resize2fs /data/rootfs.img 18G"
```

### 7. Reboot & Connect to Wi-Fi
Unmount any remaining temporary mounts and reboot:
```bash
adb shell "sync"
adb reboot
```
* **First Boot:** Unplug the USB cable during initial boot if the device stalls at the Huawei logo, then plug it back in once Phosh loads (~30–45 seconds). Default Droidian PIN/password is `1234`.
* **Wi-Fi:** Open **iwgtk** from the Phosh app grid (or run `iwctl` in Terminal) to scan and connect to Wi-Fi.
* **Troubleshooting Boot Issues:**
  * If the kernel fails to find the initramfs on `system`, repack `Phenix_Kirin65x-4.9.337-permissive-genroot.img` with `root=/dev/mmcblk0pXX` (matching your `system` block device) added to `Phenix_Kirin65x-4.9.337-permissive-genroot-cmdline.txt`.
  * If `userdata` fails to mount early in boot, verify your `userdata` partition node in TWRP (`ls -lah /dev/block/bootdevice/by-name/userdata`) and update the fallback path (`mmcblk0p59`) inside `scripts/halium` in `halium_EMUI9.1-initramfs.tar`.

---

## Rescue Telnet Access & Security Hardening

This image includes an active **root Telnet rescue service (`droidian-telnet.service`)** listening on **`10.15.19.82:23`** over USB RNDIS (`rndis0`) so you can recover or debug the system if Phosh or SSH is unreachable.

### Connecting Over USB
1. Plug the phone into your PC via USB after Phosh boots.
2. Assign an IP in the `10.15.19.0/24` subnet to the newly detected USB Ethernet interface on your PC:
   ```bash
   sudo ip addr add 10.15.19.100/24 dev <usb0_or_enp0s...>
   sudo ip link set <usb0_or_enp0s...> up
   ```
3. Connect as root:
   ```bash
   telnet 10.15.19.82
   ```

### Disabling Telnet (Recommended After Setup)
Because Telnet transmits unencrypted root sessions over the USB network interface, disable it once you have verified SSH (`ssh droidian@10.15.19.82`) or local Terminal access:
```bash
sudo systemctl disable --now droidian-telnet.service
```

---

## Credits & Acknowledgments

* Kernel source by Iceows (`https://github.com/Iceows/android_kernel_huawei_hi6250/tree/Phenix-4.9.337`)
* Rootfs and mobile stack by the Droidian Project (`https://github.com/droidian-images/droidian`)
* Initramfs base by the Halium Project (`https://github.com/Halium/initramfs-tools-halium`) 
* Debugging, rootfs adaptation scripts, hardware daemons, and documentation created with assistance from Google Gemini.
