---
categories:
- Software
date: "2016-08-24"
title: "Ramips device porting on OpenWRT"
description: "A step-by-step guide to explain how you can add a new ramips (Ralink and MediaTek) target on OpenWRT"
keywords: ["ramips", "openwrt"]
aliases: ["/openwrt-add-ramips-target/"]
---

After porting two **ramips** target on OpenWRT (*SFR Minihub* and *D-Link DCS5020L A1*), I decided to write this guide to help beginners. It explains how OpenWRT structures devices and focuses on ramips target specialties.

These instructions were written in August 2016, and at the time of writing the official Wiki was out of date.

> **WARNING:**
> THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. If you lose your data, brick your device, any other damage or anything else happens (e.g. your cat eats your dog), it is YOUR PROBLEM and YOUR RESPONSIBILITY. Your device warranty is most probably void after installing this.

## First step: The Device-tree file

![Device-tree logo](/assets/images/openwrt-add-ramips-target/devicetree.png)

The hardest part about adding a target to OpenWRT is writing a device-tree file. When a kernel-panic occurred, It may mainly come from an error in this file.

All the device-tree files for ramips architecture are located in "**[source root]/target/linux/ramips/dts/**".

I recommend you to copy a device-tree from a very similar device (same partition layout and size, same CPU) and then remove all the GPIO LEDs and buttons settings.

Double check everything: the reference of the SPI chip, the firmware partition size...
If something is misconfigured, OpenWRT will warn you about the error in the build, or in the dmesg (written on Serial during boot).

For the LEDs and buttons, you can add them later with a working OpenWRT and a simple method described [here](https://wiki.openwrt.org/doc/hardware/port.gpio#finding_gpio_pins_on_the_pcb).

For the **SFR Minihub**, I copied "*ALL0256N-4M.dts*" and I had only to change the partition table to make it boot.

The **SFR Minihub device-tree file** is [on my GitHub](https://raw.githubusercontent.com/erdnaxe/SFR-Minihub-hacking/master/SFR-MINIHUB.dts).

The name you give to the file and the names inside are really important. With these names, Linux can identify on which hardware it is running on and then load the correct scripts. See other device-trees to understand how it's defined.

## Second step: image creation

We need to edit the **Makefile** describing image creation process to build our own OpenWRT sysupgrade image.

The file to edit depends of the processor, for example : **[source root]/target/linux/ramips/image/rt305x.mk**.

In this file you will need to add your device to **Image/Build/Profile/Default** list :

``` make
define Image/Build/Profile/Default
  # [...]
    $(call Image/Build/Profile/SL-R7205,$(1))
    $(call Image/Build/Profile/SFR-MINIHUB,$(1))  # New line for new target
    $(call Image/Build/Profile/UR-326N4G,$(1))
  # [...]
endef
```

Then somewhere in the Makefile, you need to add this line (change *Default4M* with the corresponding size of your SPI) :

``` make
Image/Build/Profile/SFR-MINIHUB=$(call BuildFirmware/Default4M/$(1),$(1),sfr-minihub,SFR-MINIHUB)
```

Now you should be able to follow the official instructions about how to build OpenWRT (select default target in the menuconfig). It should compile and generate the following file: "**[source root]/bin/ramips/bin/ramips/openwrt-ramips-rt305x-DEVICE-squashfs-sysupgrade.bin**".

Now you should be able to flash this firmware image in U-boot and boot on a working OpenWRT! But the network and led configurations should not be working...

## Third step: scripts configuration

You should now be able to fill the LEDs and buttons GPIO section in your DTS (go back to the first step).

After, you need to edit the following files :

* **[source root]/target/linux/ramips/base-files/lib/ramips.sh** to set the code name for your device (see the device-tree names),
* **[source root]/target/linux/ramips/base-files/lib/upgrade/platform.sh** to allow the OpenWRT upgrade on your device,
* **[source root]/target/linux/ramips/base-files/etc/diag.sh** to set the boot led (useful to go in failsafe mode),
* **[source root]/target/linux/ramips/base-files/etc/board.d/01_leds** to set Wifi, Ethernet leds... (optionally)
* **[source root]/target/linux/ramips/base-files/etc/board.d/02_network** to set the switch configuration of your device.

**Now you should be able to recompile OpenWRT (just do another `make`) and everything should be working.** I really recommend you to add LuCI to your build to get an awesome web interface.

