---
layout: post
title: "Linux on Dell Chromebook 11"
description: "How to make any Linux 4.15 distribution run smoothly on Dell Chromebook 11 2015 (3120, Candy)"
comments: true
category: custom_firmware
icon: assets/icons/icon-laptop.svg
keywords: ["dell", "chromebook", "linux", "4.15", "ubuntu", "kubuntu", "galliumos", "chtmax98090"]
lang: en
---

This is a how-to to make any Linux > 4.15 distribution run smoothly on Dell Chromebook 11 (3120 codename *candy*).

<figure class="right-image">
  <img src="{{site.url}}/assets/images/chromebook-linux/chromebook-kubuntu.png" alt=""/>
  <figcaption>Dell Chromebook 11 running KUbuntu</figcaption>
</figure>

A Linux distribution for Chromebooks based on XUbuntu exists, you can check out [GalliumOS](https://galliumos.org/). But you might want more upstream distributions.
Since Linux 4.15 (Ubuntu > 18.04), the hardware has been supported officially, but some stuff needs post-installation configuration.

I have tested XUbuntu, Ubuntu, and KUbuntu and I end up using **KUbuntu** as my daily driver. The Intel OpenGL driver makes Plasma desktop far more fluid than Gnome Shell for some reasons.

This article describes some post-installation specific configuration.

> **WARNING:**
> THIS METHOD OF HACKING IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. If you lose your data, brick your device, any other damage or anything else happens (e.g. your cat eats your dog), it is YOUR PROBLEM and YOUR RESPONSIBILITY. Your device warranty will be void and it will be impossible to go back.

* TOC
{:toc}

## Some useful links

* [MrChromebox.tech](https://mrchromebox.tech/#fwscript): custom UEFI coreboot firmware able to run Windows/Linux natively. But won't boot the preinstall Chrome OS directly.
* [Chrome OS devices on ArchWiki](https://wiki.archlinux.org/index.php/Chrome_OS_devices): I don't recommend using SeaBIOS if you don't want to keep ChromeOS since MrChromebox made a FULL UEFI ROM (see above).

## Chromebook keyboard layout

Chromebooks have a special keyboard layout optimized for web browsing.

<figure>
  <img src="{{site.url}}/assets/images/chromebook-linux/keyboard.png" alt=""/>
  <figcaption>QWERTY US Chromebook layout</figcaption>
</figure>

X11 does include a specific layout for Chromebook keyboards. To activate it, run :
```bash
localectl set-x11-keymap fr chromebook
```

Right `alt` will behave as a modifier to access `F1` to `F10`, `PageUp`, `PageDown`, `CapsLock`, `Delete`...

<figure>
  <img src="{{site.url}}/assets/images/chromebook-linux/keyboard-modifier.png" alt=""/>
  <figcaption>Layout when the modifier is pressed</figcaption>
</figure>

## Touchpad waking up from suspend

When the lid is closed, the screen can push the touchpad and wake the device up.

To check if you are concerned with this issue, execute

```bash
cat /proc/acpi/wakeup
```

If the line starting with `TPAD` reads `enabled` then your device will be wakened up with a touchpad click.

### Disabling touchpad waking up from suspend

The method consists in printing *TPAD* in /proc/acpi/wakeup every boot. 

One ArchLinux you can follow [the ChromeOS ArchWiki article](https://wiki.archlinux.org/index.php/Chrome_OS_devices#Fixing_suspend).

On Ubuntu/Debian-based distributions, this can be done with the following steps.

* Make a script called `acpi_wakeup` containing
  ```bash
  #!/bin/sh
  printf "TPAD" > /proc/acpi/wakeup
  ```
* Copy the script to system services directory
  ```bash
  sudo cp acpi_wakeup /etc/init.d/acpi_wakeup
  ```
* Make it executable
  ```bash
  sudo chmod 755 /etc/init.d/acpi_wakeup
  ```
* Tell Ubuntu to start it at boot
  ```bash
  sudo update-rc.d acpi_wakeup defaults
  ```

## Audio: Alsa Use Case Manager definitions

This section is based on [this ArchLinux bug thread](https://bugs.archlinux.org/task/48936).

The Intel Atom Processor Audio Controller *chtmax98090* works natively under Linux 4.15 but it isn't properly configured out of the box. This results in no correct mixer control making sound unusable.

To make audio works you can use [plbossart's Alsa Use Case Manager configuration](https://github.com/plbossart/UCM) :

* Clone locally the repository (you may need to install *git* versioning tool)
  ```bash
  git clone https://github.com/plbossart/UCM.git
  ```
* Copy the UCM (Use Case Manager) definitions
  ```bash
  sudo cp -rv UCM/chtmax98090 /usr/share/alsa/ucm
  ```
* Kill and reinit audio servers
  ```bash
  alsactl kill quit
  alsactl init
  pulseaudio --kill
  pulseaudio --start
  ```

Now audio should work fine.

## Power button

You may want to disable power button due to the horrible placement. Please refer to your desktop environment configuration.

## Bonus: Use the LED with a Python script

Dell ships the Chromebook with an integrated activity RGB led.

You can talk to the microcontroller driving the led with the following Python script adapted from [Can Bülbül's candy-led project](https://github.com/hgeg/candy-led).


```python
from random import randrange
import sys
from time import sleep

# Available colors
colors = {
    'red': 0x01,
    'green': 0x02,
    'blue': 0x03,
    'yellow': 0x04,
    'magenta': 0x05,
    'cyan': 0x06,
    'white': 0x07,
    'black': 0x08
}


def changeColor(color='white'):
    """ Change Dell Chromebook 11 led color """

    # Checksum function
    chsum = lambda b0, b1, b3: (21 * b0**2 + 19 * b1 - 3 * b3) % 255

    # Build command
    cmd = bytearray.fromhex('ff' * 64)
    cmd[0] = 0x11
    cmd[1] = colors[color]
    cmd[3] = randrange(255)
    cmd[2] = chsum(cmd[0], cmd[1], cmd[3])

    # Send command
    try:
        with open('/dev/hidraw0', 'wb') as d:
            d.write(cmd)
    except IOError:
        print("Can't access " + devpath)
        raise


for c in colors.keys():
    changeColor(c)
    sleep(0.1)

changeColor('black')
```

To be able to change led color as an user in group *plugdev*, you may add an udev rule by writting in `/etc/udev/rules.d/40-candy-led.rules`:
```
ATTRS{idVendor}==\"04d8\",ATTRS{idProduct}==\"0b28\",MODE=\"0660\",GROUP=\"plugdev\"
```

<figure>
  <img src="{{site.url}}/assets/images/chromebook-linux/rgb.png" alt="" style="max-width:220px" />
  <figcaption>Available colors, so RGB!</figcaption>
</figure>

