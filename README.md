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
the full patch set (7.2-rc5): **108 patch files, 96 applied via `source=`** (the
other 12 are diagnostics kept on purpose, not built). These are byte-identical
to the Kali port's `kernel/patches/`; only the userland and a handful of config
symbols differ (see below). By subsystem:

- **0001-0024**: device DTS + shared-driver fixes (display, GPU, WiFi/BT, USB/OTG,
  charger, battery/JEITA, thermal, ramoops).
- **0025-0028, 0042-0049, 0058-0088**: **IPA / mobile data + interconnect +
  remoteproc/glink**. The core fix was the IPA v4.5+ register layout (the
  shared-SRAM window moved from +0x7000 to +0x10000); later patches add the
  SM6375 interconnect provider, NoC path voting, pd-mapper, the APSS watchdog and
  the glink/remoteproc reliability fixes. Needs `CONFIG_INTERCONNECT_QCOM_SM6375`.
  GSI firmware is loaded by the AP over TrustZone (`qcom,gsi-loader = "self"`,
  PAS id 15, `ipa_fws.mdt` from the modem/NON-HLOS partition).
- **0032-0036, 0059, 0095**: **audio** — APR services, LPASS macros + SoundWire +
  LPI pinctrl, the sound card, LPASS codec v2.2, the drvdata-before-clocks oops
  fix, and the soundwire wake IRQ.
- **0051-0057, 0089-0092**: **camera** — the FAN53870 camera PMIC
  (`CONFIG_REGULATOR_FAN53870`), CAMSS + CCI, the S5KJN1 rear sensor, FastRPC, and
  the flash-as-torch. (Groundwork; capture is not done yet.)
- **0062-0065, 0093-0094, 0097-0098, 0108**: **display / GPU** — LP-mode
  brightness, the 770/840 MHz GPU steps, selectable 48/60/90/120 Hz, and the DSI
  lane-underflow modeset recovery.
- **0096, 0099-0100, 0106-0109**: **battery / charger / regulator rails** —
  discharge-vs-full, the charger interrupt, and the Sony→Motorola rail voltages.
- **0101-0105, 0110-0113**: **NFC** — the s3fwrn5 (S3NRN4V) reader and the MIFARE
  listen/eSE routing.
- **0061, 0067-0068, 0049**: reliability — cpuidle, ramoops ECC/firmware regions,
  the watchdog, and the removed-region size.

Only **three config symbols** are added on top of the pmOS base to enable the new
hardware: `CONFIG_INTERCONNECT_QCOM_SM6375=y` (modem data),
`CONFIG_REGULATOR_FAN53870=m` (camera PMIC) and `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y`
(lets mismatched/out-of-tree modules load). Everything else the pmOS base already
enabled.

**Kali-only extras stay out of this pmOS port**: the NetHunter config stack (USB
Wi-Fi injection, SDR, BadUSB HID, CAN, NFS server, extended netfilter) and the
Kali userland. This keeps pmOS a clean, pentest-free build with the same working
hardware. Add `kernel/config/nethunter-config.fragment` from the Kali repo only
if you want those features.

## Not yet / pending
- Internal WiFi monitor mode: not possible (WCN3990 firmware). Use a USB adapter.
- Camera capture (PMIC + pipeline groundwork done, no image yet), GPS.
  Sensors and NFC now have kernel support via the patches above.

## src/
`src-postmarketos-*-v45.tar.gz` was the original 26-patch baseline. The live
tree under `aports/` is now the full 108-file set and is what to build from;
the tarball is only the historical starting point.

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
Config and the full 108-file patch set live under `aports/linux-motorola-rhodep/`.
Critical gotchas: NO `MODULE_COMPRESS` (hangs boot), FLAT Image not gz (the
Motorola bootloader resets on a self-decompressing image), do not mix config and
logic changes in one step.

## Difference from the Kali kernel
Same kernel and the **same 108-file patch set** (byte-identical). The only
difference is config: Kali carries ~43 active NetHunter pentest symbols (USB
Wi-Fi injection, SDR, BadUSB HID, CAN, NFS server, extended netfilter) that this
pmOS config deliberately omits. Both now share the three hardware-enabling
symbols (`INTERCONNECT_QCOM_SM6375`, `REGULATOR_FAN53870`,
`MODULE_ALLOW_BTF_MISMATCH`). So a pmOS build from this tree gets all the same
working hardware as Kali — modem/data, audio, display/GPU, camera groundwork,
NFC, sensors — without the pentest tooling. Verified: this tree builds clean with
pmbootstrap.

## pmaports MR
`postmarketOS/pmaports!9234`, fork `d4rks1d33/pmaports` branch `motorola-rhodep`,
single commit `motorola-rhodep: new device`. The MR is based on THIS port (pmOS),
NOT the NetHunter one. The pmOS `src/` has the config WITHOUT NetHunter (clean
baseline for the MR).
