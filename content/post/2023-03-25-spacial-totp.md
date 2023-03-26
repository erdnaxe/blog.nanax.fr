---
categories:
- Hardware
date: "2023-03-25"
title: "Write-up Spacial TOTP (Insomni'hack 2023)"
description: "Write-up of the Spacial TOTP challenge of Insomni'hack CTF 2023."
keywords: ["ESP32", "M3Core2", "CTF", "2023", "HMAC", "TOTP", "Ghidra", "Spacial"]
---

## Official description

> I sealed my master phassphrase on this device and protected it using my own TOTP algorithm. Can you recover it ?
>
> Once ready, come to the organizers desk to validate your solution on the device. (No connection to the device allowed)

We are given a `challenge.elf` file.

## Exploration

### The `challenge.elf` file

First, let's confirm it is an ELF file:
```
$ file challenge.elf
challenge.elf: ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, with debug_info, not stripped
```

We note that it is not stripped and **includes debug information**. The architecture is unusual, it uses **Tensilica Xtensa** instructions set.
We now look at some strings in the file:
```
$ strings -n10 challenge.elf
esp-idf: v4.4.3 6407ecb3f8
arduino-lib-builder
Dec 20 2022
[...]
axp: esp32 power voltage was set to 3.35v
[...]
M5Core2 initializing...
[...]
/home/azox/.arduino15/packages/m5stack/hardware/esp32/2.0.6/cores/esp32/esp32-hal-uart.c
/home/azox/git_repos/INS23_Spatial_TOTP/firmware/src/ins23_spatial_totp
/home/azox/Arduino/libraries/M5Core2/src/AXP192.cpp
/home/azox/Arduino/libraries/M5Core2/src/M5Core2.cpp
/home/azox/Arduino/libraries/M5Core2/src/M5Display.cpp
/home/azox/Arduino/libraries/M5Core2/src/M5Touch.cpp
/home/azox/Arduino/libraries/M5Core2/src/RTC.cpp
/home/azox/Arduino/libraries/M5Core2/src/Speaker.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/CommUtil.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/In_eSPI.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/M5Button.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/M5Timer.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/MPU6886.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/PointAndZone.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/Sprite.cpp
/home/azox/Arduino/libraries/M5Core2/src/utility/quaternionFilters.cpp
/home/azox/Arduino/libraries/TOTP_library/src/TOTP.cpp
/home/azox/Arduino/libraries/TOTP_library/src/sha1.cpp
```

From these strings, we known that the ELF file is a firmware for the
[M5Core2](https://github.com/m5stack/M5Core2).
It is based on an ESP32 and is using the Arduino framework with the
[TOTP library](https://github.com/lucadentella/TOTP-Arduino).

### The mysterious device

{{< figure src="/assets/images/hardware-spacial-totp/device_locked.jpg" title="Device locked" >}}

We get a look at the device near the organizers desk.
There is no keyboard to enter the TOTP.
After moving the device around, we notice that we can only enter **6 digits
between 0 and 3** by rotating the device around two axes.

## Proposed solution

We start by opening the firmware in Ghidra, then we reverse the TOTP code
computation and extract its HMAC key.

### Firmware analysis

As of March 2023, Ghidra does not officially support Xtensa instruction set,
but a community processor definition exists at <https://github.com/Ebiroll/ghidra-xtensa>.
After installing this processor, Ghidra is able to import the firmware successfully.

As it is using the Arduino framework, we find a `setup()` and `loop()` functions.
The setup function initializes the M5Core2, an EEPROM, a MPU6886 inertial
measurement unit, a TFT screen and [an RTC](https://en.wikipedia.org/wiki/Real-time_clock).

The loop function starts by getting a timestamp in seconds, then calls `TOTP::getCode(&totp, timestamp)`.
It then transforms the output into a `code_sequence`.
The remaining of the function displays information on the screen and manage
the sequence input and verification.

### TOTP HMAC key extraction

Using Ghidra and [TOTP library source code](https://github.com/lucadentella/TOTP-Arduino), we reimplement `TOTP::getCode` in Python.
Then we implement the same transformation to get a TOTP code with 6 digits between 0 and 3.

The TOTP library needs a HMAC key. By looking at `&totp` structure references,
we find `_GLOBAL__sub_I_prev_state` function that sets the key to `eed2976a1dcb29e02e42` and the timestep to `60`:
```C
int _GLOBAL__sub_I_prev_state(int param_1) {
  TOTP(&totp, hmacKey, 10, 60);
  return param_1;
}
```

We can now write the following Python script:
```Python
import datetime
from Crypto.Hash import HMAC, SHA1

def getCodeFromSteps(steps):
    """Python translation of the getCodeFromSteps() function"""
    hmac_key = bytes.fromhex("eed2976a1dcb29e02e42")
    _byteArray = bytearray(8)
    _byteArray[4] = (steps >> 0x18) & 0xFF
    _byteArray[5] = (steps >> 0x10) & 0xFF
    _byteArray[6] = (steps >> 8) & 0xFF
    _byteArray[7] = steps & 0xFF
    h = HMAC.new(hmac_key, digestmod=SHA1)  # initHmac(&Sha1,_hmacKey,_keyLength);
    h.update(bytes(_byteArray))  # write((Print *)&Sha1,_byteArray,8);
    puVar7 = h.digest()  # puVar7 = resultHmac(&Sha1);
    bVar1 = puVar7[0x13]
    _offset = bVar1 & 0xf
    _truncatedHash = 0
    iVar5 = 0
    iVar3 = 3
    while True:
        uVar4 = _truncatedHash << 8
        iVar6 = (bVar1 & 0xf) + iVar5
        _truncatedHash = uVar4
        iVar5 = iVar5 + 1
        bVar2 = puVar7[iVar6]
        _truncatedHash = bVar2 | uVar4
        if iVar3 == 0:
            break
        iVar3 = iVar3 + -1
    uVar4 = (bVar2 | uVar4 & 0x7fffffff) % 1000000
    return uVar4

# This is the datetime showed on device screen 
timestamp_sec = int(datetime.datetime(2023, 3, 24, 20, 5, 17).timestamp())
print(f"{timestamp_sec=}")

# Compute TOTP::getCode(&totp, timestamp)
totp = getCodeFromSteps(timestamp_sec // 60)
print(f"{totp=}")

# Compute code_sequence
code_sequence = bytearray(6)
for i in range(6):
    code_sequence[i] = (totp >> ((i & 0xf) << 1)) & 3
print(f"{code_sequence=}")

# Print sequence
for c in code_sequence:
    if c == 3:
        print("up")
    elif c == 2:
        print("down")
    elif c == 1:
        print("right")
    else:
        print("left")
```

We then run this script and enter the sequence:

{{< figure src="/assets/images/hardware-spacial-totp/device_unlocked.jpg" title="Device unlocked and displaying the flag" >}}

We get the following flag: `INS{0r13nt3d_T1m3d_P455wd}`.
