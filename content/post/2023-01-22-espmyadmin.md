---
categories:
- Hardware
date: "2023-01-22"
title: "Write-up ESPMyAdmin (Insomni'hack Teaser 2023)"
description: "Write-up of the ESPMyAdmin of Insomni'hack Teaser CTF 2023."
keywords: ["ESP32", "ESP32-S3", "CTF", "2023", "MAC", "AES-XTS", "Ghidra"]
---

## Official description

> The only prototype of our brand new IoT device was stolen with the laptop
> containing the application source code... ;(
>
> And of course we had no backup ;( ;(
>
> For some reasons, the device is still online here, can you help us recover the
> secret value ?
>
> All we can provide is this logic analyzer capture.

We are given a `capture.dsl` file and a URL `https://espmyadmin.insomnihack.ch/`.

## Exploration

### Web service

We open `https://espmyadmin.insomnihack.ch/` and **we are greeted with a secret
value prompt** and a link to `https://espmyadmin.insomnihack.ch/config`.
We open `https://espmyadmin.insomnihack.ch/config` and we get some information:

  * **the chip is an [ESP32-S3](https://www.espressif.com/en/products/socs/esp32-s3)**,
  * encryption is set to `Enabled (DEVELOPMENT) - AES-256`, `FLASH_CRYPT_CONFIG=0xf`,
  * Wi-Fi MAC is `7C:DF:A1:E0:71:F1`,
  * then we get eFuses values, including `XTS_AES_256_KEY_1` and
    `XTS_AES_256_KEY_2`.

{{< figure src="/assets/images/hardware-espmyadmin/espinfo.webp" title="https://espmyadmin.insomnihack.ch/config web page" >}}

The ESP32-S3 is a newer variant of the ESP32. These chips use Tensilica processors with Xtensa instruction set.

Encryption is activated, it's a good time to open the manufacturer documentation:
<https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/security/flash-encryption.html>.
We learn that:

  * two 256-bit key blocks are used for `XTS_AES_256` mode,
  * the partition table should look like:
    ```
    I (62) boot: ## Label            Usage          Type ST Offset   Length
    I (69) boot:  0 nvs              WiFi data        01 02 0000a000 00006000
    I (76) boot:  1 storage          Unknown data     01 ff 00010000 00001000
    I (84) boot:  2 factory          factory app      00 00 00020000 00100000
    I (91) boot:  3 nvs_key          NVS keys         01 04 00120000 00001000
    ```
  * a tool `espsecure.py` (in [esptool](https://github.com/espressif/esptool))
    is able to encrypt and decrypt flash data using the `--aes_xts` flag
    (AES-XTS mode).

### capture.dsl

After a quick search online, we learn that `.dsl` files are
[DSView](https://www.dreamsourcelab.com/download/) traces.

We open it using DSView, and we get 4 logic signals: `CS`, `MISO`, `MOSI` and
`CLK`. This is an SPI interface. The chip select `CS` is pulled down when there
is activity on `MISO` and `MOSI`, so it is active-low.

Let's make the hypothesis this is a SPI flash.
We click `Decode`, then select `SPI flash` as a protocol and run the decoder.
After one minute of decoding, we confirm our hypothesis as there is no decoder
error, **it is indeed an SPI flash!**

{{< figure src="/assets/images/hardware-espmyadmin/dsview.webp" title="Decoding of SPI flash data in DSView" >}}

As hinted by the challenge prompt, **the goal seems to be to recover a secret from
the encrypted flash using one SPI capture.**

## Proposed solution

We build an encrypted flash dump from the SPI capture.
Then, we use `espsecure.py`, `XTS_AES_256_KEY_1` and `XTS_AES_256_KEY_2` to
decrypt the flash partitions.
Finally, we open the factory app partition in Ghidra and reverse the secret
value.

### Building encrypted flash dump from logic analyser capture

We extract the SPI flash commands from DSView using a CSV export.
Then we use the following Python script to reassemble a flash image:
```Python
flash = bytearray(0x200000)

with open("decoder--230121-130641.csv") as f:
    for line in f.readlines():
        if "I/O read (addr" in line:
            # SPI read command, extract fields from CSV line
            line = line.strip()
            _, data = line.split("I/O read (addr ", 1)
            addr, data = data.split(", ", 1)
            n_bytes, data = data.split(" bytes): ", 1)
            addr = int(addr, 16)
            n_bytes = int(n_bytes)
            data = bytes.fromhex(data)

            assert addr+n_bytes < len(flash)
            flash[addr:addr+n_bytes] = data

with open("spi_flash.bin", "wb") as f:
    f.write(flash)
```

We plot the entropy of the resulting image using `binwalk -E spi_flash.bin`:

{{< figure src="/assets/images/hardware-espmyadmin/entropy_flash_enc.svg" title="SPI flash entropy" >}}

We observe a large block starting at 0x20000.
According to the default partition table, this is the factory app partition.
As the entropy is almost 1, we confirm that the partition is encrypted.
Let's extract this encrypted partition:

```
dd if=spi_flash.bin of=app.enc bs=131072 skip=1
```

We now have the encrypted factory app partition.

### Decrypting factory app partition

As we were not sure of the key order or endianness, we made a Python script
to quickly try a set of key and plot the entropy.
If the output entropy is low, then the key is the right one.
```Python
# All efuses
keys = [
    bytes.fromhex("262C43B0D0EF9FFAE8135D7C7CDE16C95934E442EF342B8D41C7A585ABA30518"),
    bytes.fromhex("3D8653CF6648A5A77C1629A29ACB3F908A7F1D444BDE52C5748214795B461DF3"),
    bytes.fromhex("63ED844D77B80F1B50D68ED184323281438A96FC4F6971EDD912E02F79DFF4F4"),
    bytes.fromhex("5C6D737B33CAB17A93C0E3C9DDFEDAE8CB6494CEAB95A1774B69D56B1329CC5B"),
    bytes.fromhex("2047374EAF86A26D3209E6E503B4FD1160CE4B7E34FE6387A5F2FD04F372EADD"),
    bytes.fromhex("261C69AB6A28873F5296DC54D92B684FD003D93B8DC493AF5A20322CFF2D7E4F"),
]

# The final key and endianness that makes the entropy go low
key = keys[1] + keys[0]
key = key[::-1]

with open("key.bin", "wb") as f:
    f.write(key)
```

The decrypt and plot entropy using:
```
espsecure.py decrypt_flash_data --aes_xts -a 0x20000 -k key.bin -o app.dec app.enc
binwalk -E app.bin
```

We now have the decrypted factory app partition.

### Reversing the secret value

Running `strings` on the app partition reveals some useful strings such as
`Wrong secret`, `Correct secret !`, `secret=([A-Za-z0-9]{32})` and
`Your flag is :`.

As of January 2023, Ghidra does not officially support Xtensa instruction set,
nor loading directly a ESP32 flash partition, but there are community
contributions for these!
We follow this guide to set up Ghidra: <https://olof-astrand.medium.com/analyzing-an-esp32-flash-dump-with-ghidra-e70e7f89a57f>.

**Note that you should use ESP32-S3 SVD and `esp32s3_rev0_rom.elf` ROM available at <https://github.com/espressif/svd/> and <https://github.com/espressif/esp-rom-elfs/releases>**.

After running the automatic analysis in Ghidra, we look for cross-references to
the previous found strings. After some reversing, we find the problematic
`memcmp`:

```C
undefined4 FUN_42007114(void) {
  // [...]

  if (iVar5 < 1) {
    // [...]
  } else {
    iVar5 = FUN_42063344(http_smth, s_secret_([A_Za_z0_9]_32_)_3c082f7c, 1);
    if (iVar5 == 0) {
      // [...]
      iVar5 = FUN_42064de4(http_smth, secret_value, 2, puVar3,0);
      if (iVar5 == 0) {
        // [...]
        FUN_420275d4(mac_addr, 1);  // Get device MAC address 1, 7C:DF:A1:E0:71:F1
        FUN_42007990(7, (int)auStack225, 0, 0x100);
        MD5Init(md5ctx);
        MD5Update(md5ctx, mac_addr, 6);
        MD5Final(md5ctx, auStack65);
        // [...]
        aes_cbc_decrypt(auStack292,0);
        memset_0_0x58(md5ctx);
        memset_0_0x22(auStack292);
        iVar5 = __call_memcmp(local_a1, _secret_value, __n_04);
        if (iVar5 == 0) {
          // [...] XOR then triggers "Your flag is :"
          return extraout_a3;
        }
      }
      // [...] triggers "Wrong secret"
      len_str = __call_strlen((char *)wrong_secret_str);
      http_somethg0(extraout_a2_02, wrong_secret_str, len_str, puVar3, puVar4, unaff_a15);
      http_somethg1(http_args, wrong_secret_str, len_str, puVar3, puVar4, unaff_a15);
      uVar1 = extraout_a4;
    } else {
      uVar1 = 0;
    }
  }
  return uVar1;
}
```

We wrote the following script to retrieve the secret, then the flag:

```Python
from Crypto.Cipher import AES
from hashlib import md5
from pwn import xor

# Extracted from reverse
a = bytes.fromhex("929f92e98af63982791127612d1146ddba0ef666b2774e362e7ac8c3e5ce47cd")
b = bytes.fromhex("2338031428451c0c0b3a404a2f31552b2d26083c43611b27380504553b714918")

# EFUSE_BLK7
key = bytes.fromhex("5C6D737B33CAB17A93C0E3C9DDFEDAE8CB6494CEAB95A1774B69D56B1329CC5B")

# MAC address MD5
iv = md5(bytes.fromhex('7CDFA1E071F1')).digest()

cipher = AES.new(key, AES.MODE_CBC, iv=iv)
x = cipher.decrypt(a)
print(xor(b, x))
```

We get the flag: `INS{e$P_Fl4sh_3ncrypt1on_5uck$!}`
