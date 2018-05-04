---
layout: post
title: "OpenWRT on the SFR Minihub"
description: "A step-by-step guide to explain how I managed to flash OpenWRT on my SFR Minihub"
comments: true
category: custom_firmware
keywords: ["sfr", "minihub", "openwrt"]
icon: assets/icons/icon-parameters.svg
cover: 2016-08-22-openwrt-for-sfr-minihub.png
lang: en
---

The SFR Minihub stock firmware is very restrictive. This guide helps you to install a fully working OpenWRT system on it.

If you're willing to repeat this on your own device, you can clone [this repository](https://github.com/erdnaxe/SFR-Minihub-hacking) and [OpenWRT trunk sources](https://dev.openwrt.org/wiki/GetSource) to have all the files ready.
There are also precompiled files if compilation isn't your thing.

> **WARNING:**
> THIS METHOD OF HACKING IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. If you lose your data, brick your device, any other damage or anything else happens (e.g. your cat eats your dog), it is YOUR PROBLEM and YOUR RESPONSIBILITY. Your device warranty will be void and it will be impossible to go back.

* TOC
{:toc}

## What's inside the Minihub?

Out of the box, the SFR Minihub is a hub which connects to a network (on the WAN port) and emits its own WPA Wifi with SFR OpenVPN network. It was created to be WDS capable: you can pair minihubs together easily to extend the network.

Inside, you will discover these chips :

* A standard Ramips SoC: **Ralink RT3050F**,
* A little amount of ram: **EtronTech EM63A165TS-6G**,
* A SPI flash : **MX 25L3205D**.

The board has got 3 LEDs and 2 buttons controlled with GPIOs on the SoC. It's only powered via a USB cable.

## The eRcOmM hell : Boot-loader hacking

![sercomm-dalek]({{ site.baseurl }}/assets/images/openwrt-for-sfr-minihub/sercomm-dalek.png){: .right-image}

This device got the same boot-loader signature verification as the [Linksys WAP4410n](https://wiki.openwrt.org/toh/linksys/wap4410n#ercomm_hell_and_mtd_specialities). There are two methods to flash a working OpenWRT: you flash a very special OpenWRT with the signature but with lower free space, or you change the boot-loader. I decided to apply the second method.

If you already block your device by flashing an unsigned image, you can ignore the eRcOmM error by pressing **CTRL+C** to go to the U-Boot menu (you have to repeat this each boot).

### First step: serial access

To get a working serial to update U-Boot, I used a **5V UART<->USB chip** (you can pick an Arduino).

To take apart the Minihub, I use something very flat and push it through the side of the board, then the top piece can be removed.

The UART headers are just behind the Ethernet port. I soldered **RX**, **TX** and **GND** and connect with a baud-rate of **57600** : `screen /dev/ttyUSB0 57600`

### Second step : boot-loader configuration

In the beginning of the boot log, U-Boot print information such as the size of the RAM, the chip connected on port 5... I used all these information to configure my U-Boot build.

You can download the latest Ralink U-boot source code here : [ralink_sdk on GitHub](https://github.com/stevenylai/ralink_sdk) or [Ralink U-boot on my GitHub](https://github.com/erdnaxe/SFR-Minihub-hacking/raw/master/Ralink-Uboot.tar.bz2).

To build U-boot, you will need a GCC build-root. It can be downloaded here : [buildroot-gcc342.tar.bz2](https://github.com/kinger1172/ralink_rt5350/blob/master/SDK_4_0_0_0/RT288x_SDK/toolchain/buildroot-gcc342.tar.bz2) or [Ralink buildroot on my GitHub](https://github.com/erdnaxe/SFR-Minihub-hacking/raw/master/buildroot-gcc342.tar.bz2).

I did these steps to configure my U-Boot :

1. Go in the Uboot directory with a shell and execute `make menuconfig`. You have to specify the following parameters :
 *  **Cross Compiler path** : the path to the extracted *buildroot-gcc342.tar.bz2* + */bin*,
 * **Chip Type** : *ASIC*,
 * **Chip ID** : *RT3052* (yeh RT3050 is a variant),
 * **Port 5 Connect to** : *None*,
 * **Flash Type** : *SPI*,
 * **DRAM Component** : *256Mb*,
 * **DRAM Bus** : *16bits*
2. Then select **Exit** and save the configuration (a file named *.config* will be created).

Because the partition layout is a bit different I edited **include/configs/rt2880.h** and edit line **317** like this :

``` c
#define CFG_BOOTLOADER_SIZE 0x2fd70
#define CFG_CONFIG_SIZE 0x10000
#define CFG_FACTORY_SIZE 0x290
```

Then I inverted **factory** and **config**, edit line **321** like this :

``` c
#define CFG_ENV_ADDR        (CFG_FLASH_BASE + CFG_BOOTLOADER_SIZE + CFG_FACTORY_SIZE)
#define CFG_FACTORY_ADDR    (CFG_FLASH_BASE + CFG_BOOTLOADER_SIZE)
```

### Third step : boot-loader compilation and installation

Create the image :

1. Execute `make` (this is too fast for a coffee break),
2. At the end make will say that **uboot.img** is the correct file.

I used a **TFTP** server to send U-Boot :

1. Install a **TFTP server** (*tftp-hpa* on ArchLinux),
2. Copy **uboot.img** to the tftp server directory (*/srv/tftp/* on ArchLinux),
3. Launch **tftpd** daemon (*systemctl start tftpd*),
4. Set a manual IP address on Ethernet, for example : **192.168.1.2** for your computer ip, **255.255.255.0** for the netmask and **192.168.1.1** for the gateway.

Then I booted the device with serial console and select **option 9** in the boot menu, then enter these to set the location of the U-boot to flash :

* Device IP : **192.168.1.1**,
* Server IP : **192.168.1.2**,
* Filename : **uboot.img**.

It will flash and reboot the device after. Do not touch anything during the flash!

## Free your device!

If you want to put OpenWRT, you can read my other article about how to compile OpenWRT for a new ramips target.

To flash a new system, copy "**bin/ramips/openwrt-ramips-rt305x-sfr-minihub-squashfs-sysupgrade.bin**" to your TFTP server directory (rename it **openwrt.bin** for easier flashing).

In U-boot boot menu, select **option 2** and flash the file. HORRA!

*[SoC]: System on Chip

