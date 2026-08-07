---
publish: true
created: 2026-07-29
modified: 2026-08-07T09:58:55.985Z
---

# Installing Ubuntu 26.04 on an HP EliteDesk 705 G4

Refurbished HP EliteDesk mini PCs are affordable machines with great Linux compatibility. They are suitable for home lab servers running services like [Hermes agent](https://hermes-agent.nousresearch.com), [Home Assistant](https://www.home-assistant.io) or [Coolify](https://coolify.io).

This guide covers a clean Ubuntu Desktop 26.04 LTS installation on the following system:

- HP EliteDesk 705 G4 DM 35W
- AMD Ryzen 5 PRO 2400GE with integrated Radeon Vega graphics
- 4K monitor connected through DisplayPort

## 1. Download and verify Ubuntu

Download `ubuntu-26.04-desktop-amd64.iso` from:

https://releases.ubuntu.com/26.04/

Verify the image on macOS:

```sh
shasum -a 256 ~/Desktop/ubuntu-26.04-desktop-amd64.iso
```

Expected SHA-256:

```text
487f87faaf547ea30e0aba4d5b53346292571256b25333a978db1692bcee9dd2
```

## 2. Create the installation USB

Identify the USB device:

```sh
diskutil list
```

Unmount it and write the image, replacing `diskN` with the correct device:

```sh
diskutil unmountDisk /dev/diskN
sudo dd if=~/Desktop/ubuntu-26.04-desktop-amd64.iso of=/dev/rdiskN bs=4m status=progress
sync
diskutil eject /dev/diskN
```

The `rdisk` device provides faster raw access on macOS.

> [!warning]
> Writing the image erases the selected USB drive. Verify the device identifier before running `dd`.

## 3. Prepare the firmware

Open the HP firmware setup with `F10`.

Configure:

- Legacy Support: **Disabled**
- Secure Boot: **Disabled**

With Legacy Support disabled, HP automatically uses an **All UEFI** Option ROM policy. The corresponding setting may therefore be hidden or locked.

The original Q26 firmware on this machine was version `02.08.00`, dated 2019-06-25. Update it through **Check HP.com for BIOS Updates**. HP firmware package `02.24.01` supports the EliteDesk 705 G4 DM:

https://ftp.hp.com/pub/softpaq/sp155501-156000/sp155509.html

Keep the computer connected to power and do not interrupt the update. Recheck the firmware settings afterward because an update may reset them.

## 4. Boot the installer

1. Insert the installation USB.
2. Start the computer and press `Esc`.
3. Press `F9` to open the boot-device menu.
4. Select the UEFI USB device.
5. Choose **Try or Install Ubuntu**.
6. Follow the installer and select the required disk layout, encryption, user account, and time zone.

## Installer freeze

With the original 2019 firmware, the Ubuntu 26.04 live environment consistently hard-froze before displaying the installer. The loading animation stopped, systemd output ceased during startup, and the keyboard became unresponsive. Recovery required holding the power button.

The ISO checksum and USB layout were valid. The appropriate first corrective action is updating the Q26 firmware, confirming UEFI-only boot, and retrying the standard installer entry without temporary kernel parameters.

## After installation

Run all available updates and verify:

- Ethernet and Wi-Fi
- Bluetooth
- DisplayPort video and audio
- 4K resolution and scaling
- Suspend and restart
- Firmware updates

Next step: configure remote desktop: [[remote-desktop-ubuntu-macos]]
