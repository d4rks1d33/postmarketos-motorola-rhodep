# postmarketOS port — motorola-rhodep (STABLE)

Alpine + Phosh on mainline kernel 7.2-rc5. This is the stable port.

## img/ — versions (all kernel 7.2.0-rc5)

| Ver | boot.img | What it adds | Notes |
|---|---|---|---|
| v42 | boot-v42-IPA.img | first IPA attempt (node misplaced) | historical |
| v43 | boot-v43-DOCKER-IPA.img | Docker (partial) + IPA | historical |
| v44 | (apk only) | IPA with mem fix | **BOOTLOOP** — do not use (IPA hangs without interconnect) |
| **v45** | boot-v45-DOCKER.img | **Docker OK + IPA disabled** | **CURRENT STABLE** |

Each version: `linux-motorola-rhodep-vNN.apk` + `modules-vNN.tar.gz`
(+ `device-motorola-rhodep-vNN.apk` where applicable).

## Works
Boot, UFS, display, touch, accelerated GPU, WiFi+BT, battery+charging,
thermal, USB host/gadget+SSH, buttons, **vibrator**, **Docker**,
**mobile data (IPA)**, **audio**.

## Kernel patches (shared with the Kali port)
The kernel is the base both ports share. `aports/linux-motorola-rhodep/` carries
the full patch set (7.2-rc5):
- 0001-0024: device DTS + shared-driver fixes (display, GPU, WiFi/BT, USB/OTG,
  charger, battery/JEITA, thermal, ramoops).
- 0025-0026: **IPA / mobile data**. The fix was the IPA v4.5+ register layout
  (the shared-SRAM window moved from +0x7000 to +0x10000 relative to the register
  base); the old window made the first SRAM access reset the SoC with no log.
  GSI firmware is loaded by the AP over TrustZone (`qcom,gsi-loader = "self"`,
  PAS id 15, `ipa_fws.mdt` from the modem/NON-HLOS partition).
- 0008: ramoops moved to `0xaf000000` (the address the Motorola bootloader
  actually preserves across a warm reset; the generic `0xd0000000` node is
  deleted on Motorola boards, which is why nothing was ever recovered from it).
- 0032-0036: **audio** — APR services, LPASS macros + SoundWire + LPI pinctrl,
  the Motorola rhodep sound card, and the LPASS codec v2.2. Needs
  `CONFIG_SM_LPASSCC_6115=m` (enabled in the config).

Kali-only extras (NetHunter tools, BTF-mismatch config, rtl8188eus, phone apps)
are NOT here — those belong to the Kali userland, a different OS. Only the shared
kernel work is mirrored into this pmOS port.

## Not yet / pending
- Internal WiFi monitor mode: not possible (WCN3990 firmware). Use a USB adapter.
- Sensors, GPS, NFC, camera: pending.

## src/
`src-postmarketos-*-v45.tar.gz` = 26 patches + APKBUILD + config **without**
the NetHunter features. This is the "clean" baseline for the pmaports MR. It is
extracted under `aports/` in this repo so it can be browsed/edited directly.

## Installation (ALWAYS via apk, NOT `fastboot flash boot` by hand)
This device has `deviceinfo_flash_kernel_on_update="true"`: the running DTB/boot
comes from the **kernel apk installed in the rootfs** (`/boot/dtbs/...`), and
boot-deploy reflashes `boot_a` when the apk is installed. Flashing a boot.img by
hand is ONLY for rescue and gets overwritten by the next `apk add`.
```
# from the host (phone IP, over USB gadget 172.16.42.1 or WiFi):
scp linux-motorola-rhodep-vNN.apk modules-vNN.tar.gz user@172.16.42.1:/tmp/
# on the phone:
sudo rm -rf /usr/lib/modules/7.2.0-rc5           # gotcha: remove the old ones first
sudo tar -xzf /tmp/modules-vNN.tar.gz -C /
sudo depmod -a 7.2.0-rc5
sudo apk add --allow-untrusted /tmp/linux-motorola-rhodep-vNN.apk   # reflashes boot itself
sudo reboot
# verify: 0 empty modules and 0 .ko.zst
```
Rescue via fastboot: `fastboot flash boot_a boot-v45-DOCKER.img` (boots, but
reinstall the apk afterwards to keep things consistent).

## How to build (summary)
```
cd ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/linux-motorola-rhodep
# edit patches/config, then ALWAYS:
pmbootstrap checksum linux-motorola-rhodep
pmbootstrap build --force linux-motorola-rhodep
# apk lands in ~/.local/var/pmbootstrap/packages/edge/aarch64/
# build boot.img (FLAT Image): sh scripts/make-boot-from-apk.sh <base.img> <new.img>
# extract modules: verify 0 .ko.zst!
```
Config and 26 patches: `src/src-postmarketos-*-v45.tar.gz` (or under `aports/`).
Critical gotchas: NO `MODULE_COMPRESS` (hangs boot), FLAT Image not gz (the
Motorola bootloader resets on a self-decompressing image), do not mix config and
logic changes in one step.

## Difference from the Kali kernel
Kali uses the SAME kernel + one config change: `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y`
(+ ~58 NetHunter symbols). That BTF fix is HARMLESS for pmOS, so the config could
be unified into a single kernel for both ports. pmOS v45 does not have it today;
if pmOS is rebuilt, consider adding it to converge.

## pmaports MR
`postmarketOS/pmaports!9234`, fork `d4rks1d33/pmaports` branch `motorola-rhodep`,
single commit `motorola-rhodep: new device`. The MR is based on THIS port (pmOS),
NOT the NetHunter one. The pmOS `src/` has the config WITHOUT NetHunter (clean
baseline for the MR).
