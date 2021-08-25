---
categories:
- Software
date: "2017-12-24"
title: "Archos 50 Diamond debricking"
description: "Debricking of an Archos 50 Diamond (ac50da) using Fastboot."
keywords: ["android", "debrick", "archos", "diamond", "50", "ac50da"]
aliases: ["/debrick-archos-50-diamond/"]
---

## Initial step: How are we going to debrick?

This guide works if Android doesn't boot (or freeze), and no recovery is accessible.

**The global idea is:**

* **Flash TWRP** via Fastboot,
* Reboot into TWRP and backup then do a cleanup of the device,
* **Flash Firmware** (screen, recovery...) update via TWRP,
* Reboot into the freshly flashed stock recovery,
* Flash the official *update.zip* taken from Archos Firmware Support to get back Android.

**You will need:**

* An Archos 50 Diamond (ac50da),
* A MicroSD Card,
* A cool MicroUSB cable of course.

> **WARNING:**
> THIS METHOD OF HACKING IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. If
> you lose your data, brick your device, any other damage or anything else
> happens (e.g. your cat eats your dog), it is YOUR PROBLEM and YOUR
> RESPONSIBILITY. Your device warranty is most probably void after doing this.

## First step: How to go into Fastboot mode?

![Green Screen](/assets/images/debrick-archos-50-diamond/archos_green.png)

Disconnect the USB cable (or the device might enter a strange mass storage mode).

Even if the device isn't off, to go into Fastboot mode:

* Press `Volume Up` and `Power` and hold firmly,
* The screen will blink and you will have a **green screen** with "Fastboot" written at the top,
* **Just after the screen turns green, release the buttons**, or you will enter recovery (red screen).

Now you can plug your favorite USB cable and use Android Fastboot tool.

**Under Linux**, fastboot may be in the repository. **Under Windows**, you can follow [this thread](https://forum.xda-developers.com/showthread.php?t=2588979).

## Second step: TWRP Flashing

Download the latest official TWRP recovery for the Archos 50 Diamond: [https://eu.dl.twrp.me/ac50da/](https://eu.dl.twrp.me/ac50da/).

In Fastboot mode (see first step), flash this recovery: `fastboot flash recovery twrp-3.2.1-0-ac50da.img`

![Red Screen](/assets/images/debrick-archos-50-diamond/archos_red.png)

Now reboot the device with the same method as the first step, **but keep buttons pressed until a red screen appears**. You will be granted by TWRP home screen, WAHOO!

Now do some basic stuff like:

* Put an SD Card and create a full backup of the device,
* Charge your device,
* Format userdata partition (optional)...

## Third step: Stock firmware and recovery flashing

First of all, download Archos 50 Diamond Stock Firmware [on Archos website](http://update.archos.com/AFMv1/storage/files/full/ac50da/20150420.124756/update.zip).

If you try to flash Archos' *update.zip* directly in TWRP it will fail. So we need to clean up this zip:

* Create a copy of *update.zip* called *update-twrp.zip*,
* In the zip, edit *META-INF/com/google/android/updater-script* and remove all lines starting with `symlink(` (line 6 to 19).

**Now flash *update-twrp.zip* via TWRP.**

## Final step: Stock Android flash

Put Archos' *update.zip* on a MicroSD card (Fat32 formatted) and put the SD in the phone.

Reboot into the new stock recovery (first step with red screen). And select *update.zip*.

Wait while it is flashing. Then select reboot, and you can take a break (it really takes > 5min for the first Android boot).

**And voilà!** You should have a fully working phone.

