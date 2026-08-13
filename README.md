# Port postmarketOS — motorola-rhodep (ESTABLE)

Alpine + Phosh sobre kernel mainline 7.2-rc5. Este es el port estable.

## img/ — versiones (todas kernel 7.2.0-rc5)

| Ver | boot.img | Qué agrega | Notas |
|---|---|---|---|
| v42 | boot-v42-IPA.img | primer intento IPA (nodo mal ubicado) | histórico |
| v43 | boot-v43-DOCKER-IPA.img | Docker (parcial) + IPA | histórico |
| v44 | (solo apk) | IPA con mem fix | **BOOTLOOP** — no usar (IPA cuelga sin interconnect) |
| **v45** | boot-v45-DOCKER.img | **Docker OK + IPA disabled** | **ESTABLE ACTUAL** |

Cada versión: `linux-motorola-rhodep-vNN.apk` + `modules-vNN.tar.gz`
(+ `device-motorola-rhodep-vNN.apk` donde aplica).

## Funciona
Arranque, UFS, display, táctil, GPU acelerado, WiFi+BT, batería+carga,
térmico, USB host/gadget+SSH, botones, **vibrador**, **Docker**.

## No / pendiente
- Datos móviles: IPA presente pero `status=disabled` (falta driver interconnect
  SM6375; activarlo cuelga). Ver doc técnico §7.6.
- Monitor mode WiFi interno: inviable (firmware WCN3990). Ver §7.7.
- Audio, sensores, GPS, NFC, cámara: pendientes.

## src/
`src-postmarketos-*-v45.tar.gz` = 26 patches + APKBUILD + config **sin**
features NetHunter. Esta es la línea base "limpia" para el MR a pmaports.

## Instalación (SIEMPRE por apk, NO fastboot flash boot a mano)
Este device tiene `deviceinfo_flash_kernel_on_update="true"`: el DTB/boot que
corre viene del **kernel-apk instalado en el rootfs** (`/boot/dtbs/...`), y
boot-deploy reflashea `boot_a` al instalar el apk. Flashear boot.img a mano SOLO
sirve para rescate y lo pisa el proximo `apk add`.
```
# desde la Mac (IP del telefono, por USB gadget 172.16.42.1 o WiFi):
scp linux-motorola-rhodep-vNN.apk modules-vNN.tar.gz user@172.16.42.1:/tmp/
# en el telefono:
sudo rm -rf /usr/lib/modules/7.2.0-rc5           # gotcha: borrar viejos (README-KERNEL §5)
sudo tar -xzf /tmp/modules-vNN.tar.gz -C /
sudo depmod -a 7.2.0-rc5
sudo apk add --allow-untrusted /tmp/linux-motorola-rhodep-vNN.apk   # reflashea boot solo
sudo reboot
# verificar: 0 modulos vacios y 0 .ko.zst (ver README-KERNEL §4.4/§5)
```
Rescate por fastboot: `fastboot flash boot_a boot-v45-DOCKER.img` (arranca, pero
para dejarlo consistente reinstalar el apk despues). Ver README-KERNEL §12.

## Cómo se compila (resumen; detalle en README-rhodep-KERNEL.md §4)
```
cd ~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/linux-motorola-rhodep
# editar patches/config, luego SIEMPRE:
pmbootstrap checksum linux-motorola-rhodep
pmbootstrap build --force linux-motorola-rhodep
# apk queda en ~/.local/var/pmbootstrap/packages/edge/aarch64/
# armar boot.img (Image PLANO): cd /opt/postmarket/repo ; sh scripts/make-boot-from-apk.sh <base.img> <nuevo.img>
# extraer modulos: ver README-KERNEL §4.4 (verificar 0 .ko.zst!)
```
Config y 26 patches: `src/src-postmarketos-*-v45.tar.gz` o directo en el pmaports.
GOTCHAS criticos (README-KERNEL §5): NO `MODULE_COMPRESS` (cuelga arranque),
Image PLANO no gz (bootloader Motorola resetea), no mezclar cambios config+logica.

## Diferencia con el kernel de Kali
Kali usa el MISMO kernel + 1 cambio de config: `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y`
(+ ~58 simbolos NetHunter). Ese fix BTF es INOFENSIVO para pmOS -> se podria
unificar el config y usar un solo kernel para ambos ports. Hoy pmOS v45 NO lo
tiene; si se recompila pmOS, considerar agregarlo para converger.

## MR pmaports
`postmarketOS/pmaports!9234`, fork `d4rks1d33/pmaports` rama `motorola-rhodep`.
Commit unico `motorola-rhodep: new device`. Detalle en README-rhodep-KERNEL.md §11.
El MR se basa en ESTE port (pmOS), NO en el de NetHunter. El `src/` de pmOS tiene
la config SIN NetHunter (linea base limpia para el MR).
