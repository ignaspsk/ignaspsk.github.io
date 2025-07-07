---
title: "How to install GrapheneOS in Fedora Linux"
date: 2025-05-22
draft: false
description: "example description"
tags: ["fedora", "android", "grapheneos"]
showTableOfContents: true
---
## Introduction
Right of the bat, there is no official install method for Fedora Workstation as there is for Ubuntu or Debian. But it's possible, and it is not difficult to do. In this guide I will show you how to install [GrapheneOS](https://grapheneos.org) in Fedora Linux environment.
Official guide can be found [here](https://grapheneos.org/install/cli).
## Prerequesites

- Carrier lock free Google Pixel device.
- Up-to-date Fedora Workstation installation

## Enabling OEM unlocking
Before continuing, you need to connect your device to internet at least once. This is needed for stock OS to check whether the device was sold as locked by a carrier.
In settings, go over to **About phone** and press **Build Number** repeatedly, until you see message "You're now a developer" pop up.
Then in settings, go to **System** > **Developer Options**, and below find the **OEM unlocking** option and toggle it.

## Installing fastboot
In the official Fedora 42 repository, required version of `android-tools` is now available! All you need to do is just get it is just:
```bash
sudo dnf install android-tools
```
Then check the version. Must be at least `35.0.1`

## Booting into the bootloader interface
Next, reboot the device and begin holding both power and volume down buttons at the same time. Wait until you enter the bootloader interface.
After that, run this to see if Fedora detects your Pixel device:
```bash
sudo fastboot devices
```

## Unlocking the bootloader
Unlock it to enable installing custom OS and firmware. Beware that by doing so all data on the device will be wiped.
```bash
sudo fastboot flashing unlock
```

## Getting openssh
OpenSSH is used to verify the download of OS beyond the security offered by HTTPS.
```bash
sudo dnf install openssh-clients
```

## Downloading OS images
Follow the instructions:
```bash
curl -O https://releases.grapheneos.org/allowed_signers
```
To download the factory image for your specific pixel, visit [releases page](https://grapheneos.org/releases) and find your device model codename. For example, to download the `VERSION` release for a device with the codename `DEVICE_NAME`:
```bash
curl -O https://releases.grapheneos.org/DEVICE_NAME-install-VERSION.zip
curl -O https://releases.grapheneos.org/DEVICE_NAME-install-VERSION.zip.sig
```
Just replace it with what you need.
Next, verify downloaded images:
```bash
ssh-keygen -Y verify -f allowed_signers -I contact@grapheneos.org -n "factory images" -s DEVICE_NAME-install-VERSION.zip.sig < DEVICE_NAME-install-VERSION.zip
```
Don't forget to replace `DEVICE_NAME` and `VERSION` with yours.
## Flashing OS image
To extract factory images:
```bash
bsdtar xvf DEVICE_NAME-install-VERSION.zip
```
And run the installation script that you have extracted together with factory image.
```bash
cd DEVICE_NAME-install-VERSION
chmod +x flash-all.sh
sudo ./flash-all.sh
```
## Locking the bootloader
After the flashing's complete, lock the bootloader by running:
```bash
fastboot flashing lock
```

Now you have successfully installed GrapheneOS in Fedora 👍

Cover background by [vishnumaiea](https://unsplash.com/@vishnumaiea) on [Unsplash](https://unsplash.com/photos/black-and-white-computer-keyboard-pfR18JNEMv8)
