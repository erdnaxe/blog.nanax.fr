---
categories:
- Hardware
date: "2024-01-28"
title: "Write-up of Long Range 2 (RealWorldCTF 2024)"
description: "Write-up of Long Range 2 challenge, RealWorldCTF 2024."
keywords: ["RealWorldCTF", "lora", "esp32", "meshtastic", "aes"]
---

Official challenge prompt:

> Of late, whispers doth persist behind mine back. Yesterday, under the studio tower, a peculiar contraption was found by me. I am most intrigued to discover the content of their discourse.

The challenge [attachment](https://github.com/chaitin/Real-World-CTF-6th-Challenges/releases/download/x/LongRange2_2bf203ac61f7a3c5f24ee3a250d8abd6.zip) contains two files:

  * `flash_dump` (8.0MiB, sha256: `508d328f855d5398aab38cc93bc66bec91dffd2bfff3691c55b096a6d273d972`)
  * `486_375MHz-1MSps-1MHz.wav` (293MiB, sha256: `1c60c7a45a4d1c279ca334339eecb51043b91b6714dda8382ecdcd3e7d4370f3`)

## `flash_dump` forensics analysis

```
$ file ./flash_dump
./flash_dump: DOS executable (COM), start instruction 0xe903023f bc983c40
```

```
$ binwalk ./flash_dump

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
88595         0x15A13         Neighborly text, "neighbors a simple (0 id) broadcast"
109607        0x1AC27         HTML document header
[...]
131048        0x1FFE8         Neighborly text, "neighborinfo message!shPacket to JSON"
135499        0x2114B         Unix path: /home/runner/.platformio/packages/framework-arduinoespressif32/libraries/Wire/src/Wire.cpp
```

From `file` and `binwalk` outputs, we observe that `flash_dump` is a dump of an **Espressif ESP32 flash**.
`file` misidentifies the flash dump as a DOS executable.
Let's use [`file` upstream definitions for ESP-IDF](https://github.com/file/file/blob/FILE5_45/magic/Magdir/firmware#L71-L133) with `binwalk` to retrieve the partition table. We copy and adapt the rules in a [`esp32.magic`](/assets/images/hardware-longrange2/esp32.magic) file.
Running `binwalk` with these definitions gives useful information:
```
$ binwalk ./flash_dump -m ./esp32.magic

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
32768         0x8000          ESP-IDF partition table entry, label: "nvs", NVS data, offset: 0x9000, size: 0x5000
32800         0x8020          ESP-IDF partition table entry, label: "otadata", OTA selection data, offset: 0xE000, size: 0x2000
32832         0x8040          ESP-IDF partition table entry, label: "app", OTA_0 app, offset: 0x10000, size: 0x250000
32864         0x8060          ESP-IDF partition table entry, label: "flashApp", OTA_1 app, offset: 0x260000, size: 0xA0000
32896         0x8080          ESP-IDF partition table entry, label: "spiffs", SPIFFS partition, offset: 0x300000, size: 0x100000
65536         0x10000         ESP-IDF application image for ESP32-S3, project name: "arduino-lib-builder", version esp-idf: v4.4.4 e8bdaf9198, compiled on Apr 20 2023 00:44:44, IDF version: v4.4.4, entry address: 0x40377680
```

Nice! So it is a **ESP32-S3 firmware using `arduino-lib-builder` and ESP-IDF partitions**.
Let's dump some partitions:
```
$ dd if=./flash_dump of=./app bs=1 skip=65536 count=2424832
2424832+0 records in
2424832+0 records out
2424832 bytes (2.4 MB, 2.3 MiB) copied, 3.95738 s, 613 kB/s
$ dd if=./flash_dump of=./flashApp bs=1 skip=2490368 count=655360
655360+0 records in
655360+0 records out
655360 bytes (655 kB, 640 KiB) copied, 1.08508 s, 604 kB/s
$ dd if=./flash_dump of=./spiffs bs=1 skip=3145728 count=1048576
1048576+0 records in
1048576+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 1.70655 s, 614 kB/s
$ file ./app ./flashApp ./spiffs
./app:      ESP-IDF application image for ESP32-S3, project name: "arduino-lib-builder", version esp-idf: v4.4.4 e8bdaf9198, compiled on Apr 20 2023 00:44:44, IDF version: v4.4.4, entry address: 0x40377680
./flashApp: ISO-8859 text, with very long lines (65536), with no line terminators
./spiffs:   data
```

{{< figure src="/assets/images/hardware-longrange2/meme-1.png" alt="Slicing ESP-IDF partitions" >}}

### OTA_0 `'app'` partition: Meshtastic firmware

Let's try a "strings-grep" quickwin:
```
$ strings -n10 ./app | grep ".cpp"
[...]
src/main.cpp
/src/mesh/Channels.cpp
/src/mesh/SX126xInterface.cpp
B/src/mesh/MeshModule.cpp
[...]
/.pio/libdeps/tbeam-s3-core/NimBLE-Arduino/src/NimBLECharacteristic.cpp
/.pio/libdeps/tbeam-s3-core/NimBLE-Arduino/src/NimBLEClient.cpp
[...]
```

Bingo! The firmware is using [PlatformIO](https://platformio.org/) ("pio") build environment with `tbeam-s3-core` library.
This means that the board might be a **LILYGO® TTGO T-Beam S3 Core**.

This board is equipped with a **Semtech SX1262 Lora transceiver**, confirmed by the presence of `SX126xInterface.cpp` file in `./app` strings.

Searching for found strings on GitHub leads to [Meshtastic firmware](https://github.com/meshtastic/firmware).
We confirm that the dumped firmware is Meshtastic with another epic strings-grep:
```
$ strings -n10 ./app | grep -i meshtastic | head
meshtastic.org
Meshtastic Android, iOS,
Visit meshtastic.org
http://meshtastic.local
meshtastic_NodeInfoLite* NodeDB::getMeshNodeByIndex(size_t)
Meshtastic_%02x%02x
meshtastic_TelemetrySensorType_BME680
meshtastic_TelemetrySensorType_BME280
meshtastic_TelemetrySensorType_BMP280
meshtastic_TelemetrySensorType_INA260
```

We also confirm the partition table layout using [upstream Meshstatic `partition-table.csv`](https://github.com/meshtastic/firmware/blob/v2.2.19.8f6a283/partition-table.csv).

**The `'app'` partition contains Meshtastic for T-Beam S3 Core.**
We don't know if Meshtastic has been modified or its version, but let's move on to the other files.

*Note: the author initially used [ghidra-esp32-flash-loader](https://github.com/dynacylabs/ghidra-esp32-flash-loader) patched with `esp32s3_rom.elf` and `esp32s3.svd` to load the flash dump in Ghidra 11.0, but this was overkill for this challenge.*

### OTA_1 `'flashApp'` partition: empty

```
$ hexdump -C ./flashApp
00000000  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
000a0000
```

This partition is empty.

### `'spiffs'` partition: dumping Meshtastic configuration

`spiffs` partition is usually used to hold [a SPIFFS filesystem in ESP-IDF projects](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/storage/spiffs.html).
We also identify some HTML/CSS/JS files in the partition:
```
$ strings -n10 ./spiffs
index-4c6bb78f.css.gz
maplibre-gl-282f41f2.js.gz
index-a77a21e0.js.gz
favicon.ico.gz
site.webmanifest.gz
```

*Note: The files are gziped on the filesystem to save space. This is usual for embedded webservers as HTTP body responses can use gzip.*

[mkspiffs](https://github.com/igrr/mkspiffs) fails to extract this partition.
Digging in Meshtastic code sources reveals that they are using `LittleFS`.
We use [littlefs-python](https://github.com/jrast/littlefs-python) to walk the filesystem and extract files in `prefs/` folder:
```
$ python3 -i examples/walk.py --img-filename=./spiffs
root . dirs ['prefs', 'static'] files []
root ./prefs dirs [] files ['channels.proto', 'config.proto', 'db.proto']
root ./static dirs [] files ['.gitkeep', 'Logo_Black.svg.gz', 'Logo_White.svg.gz', 'apple-touch-icon.png.gz', 'favicon.ico.gz', 'icon.svg.gz', 'index-4c6bb78f.css.gz', 'index-a77a21e0.js.gz', 'index-fc2e9253.js.gz', 'index.html.gz', 'maplibre-gl-282f41f2.js.gz', 'robots.txt.gz', 'site.webmanifest.gz']
>>> d= fs.open("./prefs/db.proto", "rb").read()
>>> open("./prefs/db.proto", "wb").write(d)
>>> d= fs.open("./prefs/config.proto", "rb").read()
>>> open("./prefs/config.proto", "wb").write(d)
>>> d = fs.open("./prefs/channels.proto", "rb").read()
>>> open("./prefs/channels.proto", "wb").write(d)
```

{{< figure src="/assets/images/hardware-longrange2/meme-2.png" alt="This is not SPIFFS" >}}

From the extensions, we expect Protobuf. This hypothesis is quickly confirm using `protoc --decode_raw < prefs/db.proto`.
To make sense of these structure, we need to decode them with some Protobuf definition.
Searching `'config.proto'` in Meshtastic leads to `./src/mesh/NodeDB.cpp` which calls `loadProto` with:

  * `/prefs/db.proto` as `meshtastic_DeviceState`,
  * `/prefs/config.proto` as `meshtastic_LocalConfig`,
  * `/prefs/channels.proto` as `meshtastic_ChannelFile`.

Let's dump the Protobuf data using the definitions found in `./protobufs/` git submodule:
```
$ protoc meshtastic/*.proto --decode=meshtastic.DeviceState < ../../prefs/db.proto > ../../prefs/db
$ protoc meshtastic/*.proto --decode=meshtastic.LocalConfig < ../../prefs/config.proto > ../../prefs/config
$ protoc meshtastic/*.proto --decode=meshtastic.ChannelFile < ../../prefs/channels.proto > ../../prefs/channels
```
*Note: these proto files contain a version number, so you can checkout an older Meshtastic version if needed. In this challenge, the proto files use version 22 matching Meshtastic 2.2.19.*

From the [device state `"db"` file](/assets/images/hardware-longrange2/protodump_db.txt), we learn that this is a **dump of Peter "c0c4" node (id:`fa6dc0c4`)**.
The device is a LILYGO® TTGO T-Beam S3 Core.
Peter node communicates with **Adam node (id:`fa6c4bbc`)** which uses a [Heltec WiFi LoRa 32(V3)](https://heltec.org/project/wifi-lora-32-v3/).

From the [local config `"config"` file](/assets/images/hardware-longrange2/protodump_config.txt), we discover that the LoRa radio is configured with the default Meshtastic preset.
The region is set to China (CN). `./src/mesh/RadioInterface.cpp` gives some LoRa parameters:
```C
default: // Config_LoRaConfig_ModemPreset_LONG_FAST is default.
  bw = (myRegion->wideLora) ? 812.5 : 250; // 250 in CN
  cr = 5;
  sf = 11;
  break;
```

The LoRa transmission uses a **bandwidth (BW) of 250kHz**, a **coding rate (CR) of 5** and a **spreading factor (SF) of 11**.

From the [`"channels"` file](/assets/images/hardware-longrange2/protodump_channels.txt), we learn the existence of a "Buddies" channel **using the key `cef8db8e8e6017fd6dcca21db8a1476d451480acd7f4f9f769a763f528c011f7`**.
[Meshtastic encryption](https://meshtastic.org/docs/overview/encryption) is **AES256-CTR**, so this key will be useful to decrypt the LoRa messages.

## `486_375MHz-1MSps-1MHz.wav` analysis: LoRa hell

```
$ file 486_375MHz-1MSps-1MHz.wav
486_375MHz-1MSps-1MHz.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 24 bit, stereo
```

Opening the file in Audacity reveals 16 blocks that could be radio transmission at 486.375MHz.

### Initial reconnaissance (Inspectrum)

Let's convert this file to a more standard IQ file using Python Numpy/Scipy:
```python
from scipy.io import wavfile
import numpy as np
samplerate, data = wavfile.read('./486_375MHz-1MSps-1MHz.wav')
signal = data[:,0] + data[:,1] * 1j
signal = signal.astype(np.complex64)
signal /= (2**32)//2
signal.tofile("./486_375MHz-1MSps-1MHz.complex64")
```

Opening `./486_375MHz-1MSps-1MHz.complex64` in Inspectrum reveals some chirps:

{{< figure src="/assets/images/hardware-longrange2/inspectrum.png" alt="486_375MHz-1MSps-1MHz.complex64 in Inspectrum" >}}

LoRa uses [8 up-chirp symbols then 2 down-chirp symbols to start a packet](https://www.sghoslya.com/p/lora-is-chirp-spread-spectrum.html), this confirms that we are dealing with LoRa at 486MHz!

*Note: some groups of chirps have a lower SNR than others. This could be explained by a transmission between 2 pairs, one being farther away.*

### LoRa demodulation and decoding using rpp0/gr-lora

We make a GnuRadio Companion schema using https://github.com/rpp0/gr-lora.

{{< figure src="/assets/images/hardware-longrange2/gnuradio-1.png" alt="GnuRadio schema with rpp0/gr-lora" >}}

Running this schema outputs 16 LoRa frames:
```
2d31e0bc4b7d8cd14eadb8364b980fc90d041af98a5a202fb93a88bce2f91c56a954ecdc36dbfe760dfab9a626f06bf82b3e
1b31e0c4c07c3c5a4fadc5b570900ec9ac6587bf16a017ca4f801d571334d5bd
493120c4c07c3c5a4fad599a57982fcd073170f34113cacd96abb9578e0efae2e5402846c0a1846e222de52685ed385f48483b5016a0f89eb8ccae0ec47cd7c81b75434599406b58809f6b4d6a2f
1b31e0bc4b7d8cd14eadd31c8d902ec9062a2e63dbf9178893d565d7195f2d6c
5f3000ffffcf8cd1cead6d2b5b900fcd14bd64fa23ecc8d9004a475d91c5312c3327a0bf3cc4504c5c702baccd645b80557237c13ce6d805c09983d18fb9c3db1e2b965348e9d339b902aed0a2d74fc9edbeea9583c0187f89676e9ac0f53d7a56ada86c
5f3000ffffcf8cd1cead6d2b5b810fcd14bd64fa23ecc8d9004a475d91c5312c3327a0bf3cc4504c5c702baccd645b80557237c13ce6d805c09983d18fb9c3db1e2b965348e9d339b902aed0a2f74fc9e5b4669583c4d0660727ec471b6b6299a1631c98
2d31e0ffffdfbc5a4fad614f55900fcdf731cd478c03fdbead48c28dbee19522376621e839756c126cbeb9a62620fdc81a5b
2d31e0ffffdfbc5a4fad614f55810fcdf731cd478c03fdbead48c28dbee19522376621e8396764124510b9a6662062ca1376
1f3140ffffdfbc5a4fad38fb6c9022cd648624f12bba42e31951453c2ca2d47e1683a5d4
1f3140ffffdfbc5a4fbd38fb6c9122cd648624f12bba42e31951453c4fb6dc7e932509a0
683060ffffcf8cd1cebd67ee4b9003cdee75b99e8a80c34ab309225dec208d6b952f1a75a1dca08b200bd134f76b92fbccceec1a67d95c98d32e72c0d187c9eb150cd4c7f50011f61ee28203556d6d9d1f35a1c712f9b8d838bbd16670f2ed8fea78b7ee472190851e2e6b2fbf
683060ffffcf8cd1cead67ee4b9103cdee75b99e8a80c34ab309225dec208d6b952f1a75a1dca08b200bd134f76b92fbccceec1a67d95c98d32e72c0d187c9eb150cd4c7f50011f61ee28203556d6d9d1f35a1c712f9b9fcac4ec16f5a385a5511c14d1367011c5ba5a2f9e449
223170ffffcf8cd1cebdd0786b9002c9ed6b7021378a6c2169435722be7df5247e88cca38be45f
223170ffffcf8cd1cebdd1786b9102c9ed6b7021378a6c2169435722befdf5976a88cce3ab9439
293140ffffcf8cd1cead7a6e799023c935726e033012cd5d63e1bcfaaba66c37e2f508cf4a154b3ab20c034ce36b
293140ffffcf8cd1cebd7a6e799123c935726e033012cd5d63e1bcfaaba66c37e2f508cf4a404d3ab2923f6e9b6b
```

The 3 first bytes are a LoRa header [encoding the length, coding rate and presence of CRC](https://github.com/tapparelj/gr-lora_sdr/blob/9e22055f83fdd4cd43a2bbe51309d630c3acfa90/lib/header_decoder_impl.cc#L133). This means that **Meshtastic activates explicit LoRa headers**.

*Protip: the last nibble of LoRa header is 0, so it can be easily spotted in a frame in hexadecimal format.*

Last 2 bytes are CRC16-CCITT. Checking the CRC16 of these frames reveals that they are corrupted.
Let's try other GNURadio LoRa implementations that are better with low SNR.

### LoRa demodulation and decoding using jkadbear/gr-lora

We make a GnuRadio Companion schema using https://github.com/jkadbear/gr-lora.

*Note: to run the Docker image, `QT_X11_NO_MITSHM=1` environment is needed inside the container, else gnuradio-companion might crash on start if the host is using a recent amdgpu drm module.*

We use a Pyramid demodulator as this seems to require fewer parameters tweaking.

{{< figure src="/assets/images/hardware-longrange2/gnuradio-pyramid.png" alt="GnuRadio schema with jkadbear/gr-lora" >}}

Running this schema outputs 14 LoRa frames:
```
2d31e0bc4b6cfac4c06dfa6626d2020b08a792b378fb6377d7e054d74f671ec02df18c7d0466c931bb22400fc9ec25c8c97300
1b31e0c4c06dfabc4b6cfa295d91380308a79212197e5e99471a63330fd5205d01
493120c4c06dfabc4b6cfaa641be1e0b08a792b94d9911171ca9be1547440ff1d6ce4602e2d2a411afe6dae9f20cdbd05ee5c43982cd6c69c13194f958f7d90c85b3280cc7627395667ec8be5b5e00
5f3000ffffffffc4c06dfaa3f30f120308a792aac18d79d83c56bc10345aea0fcc829141291c6301cc6e2315f6a10f7220b95df188c5e50becec89cb54b5406ec183c1b7bf775e6bff150daa09c83d6c984f70c9e2dc26972f84e05cbae75a8951d4959f01
5f3000ffffffffc4c06dfaa3f30f120208a792aac18d79d83c56bc10345aea0fccc09141291c62a5cc6e2315b5a90f7220b81dfb88c5f52bace489cb7421406ec19383bfbf775e7b5b150daa098abd6d984f5009f2dc26870f44e05cbae6ba8951d49f6300
2d31e0ffffffffbc4b6cfaedbb631c0308a792c94d34c567d13989479d376c8a0cacdc658d12564a93ef7400254437aca82601
2d31e0ffffffffbc4b6cfaedbb631c0208a792c94d34c567d13989479d376c8a0cacdc658d12564a93ef7400254437ac81a801
1f3140ffffffffbc4b6cfaa620df2d0304a7925afacd43977600361b8cc3f50fbd5d22a001
1f3140ffffffffbc4b6cfaa620df2d0204a7925afacd43977600361b8cc3f50fbd5dc0b001
683060ffffffffc4c06dfa8899c2020304a7925001502e35d73ab793c3e9ad6acc9da6a4955bc56b8054e9989d76f5355cf2868b90bffae321d51077be4e1774fc074f63a4c0af6ba38f33a829b0784bdadbcc738b16e930e741c48f4d6c0cab012e5605122dce40eaeb7b8dd001
683060ffffffffc4c06dfa8899c202020427925001402c7dd73ab7b2c3e5ad6adc9faea0955be46f8854e998fc7ef5355cd2029b90bffae02dd51077bfc81774fc074f6604c0af6a438f33a829d0f94bdadbcd138316e930a5c0c48f4d6c8dad012e5655182dce40fa6d7b192500
223170ffffffffc4c06dfa5f8a54220204a792531f89a36fea3018c9ceb7e71fa3cd7271ed9f4b01
293140ffffffffc4c06dfa86cc4a300304a7920b0e97b3a161aab9a502b145f06b98ef6a740782311ac653737f6501
293140ffffffffc4c06dfa86cc4a300204a7920b0e97b3a161aab9a502b145f06b98ef6a740782311ac65373197001
```

*Note: jkadbear/gr-lora adds an extra byte at the end of the frame to indicate whether frame CRC is correct (`01`) or incorrect (`00`).*

Although this demodulator is receiving fewer frames, their integrity seems higher: **the CRC is valid for most of these frames**.
Let's ignore corrupted frames for now.

*Note: some frames seem to be transmitted twice: this is expected for a mesh, with the hop count decreasing at each retransmission.*

{{< figure src="/assets/images/hardware-longrange2/meme-3.png" alt="LoRa demodulation with GnuRadio" >}}

### Decoding Meshtastic header and payload

After a bit of searching, we found [a discourse thread explaining the packet structure](https://meshtastic.discourse.group/t/meshtastic-lora-packet-size/7953/8):

```
       16 symbols      3 bytes     max 255 bytes          2 bytes
     ┌─────────────┬───────────────┬─────────────────┬───────────┐
     │ (Preambule) │  Lora Header  │     Payload     │Payload CRC│
     └─────────────┴┬─────────────┬┴┬───────────────┬┴───────────┘
  ┌─────────────────┘             │ └──┐            └──────────────┐
  │1 byte 3bit 1bit 1 byte   4bit │    │ 16 bytes    max 239 bytes │
 ┌┴──────┬────┬───┬────────┬──────┴┐  ┌┴───────────┬───────────────┴┐
 │Length │FEC │Has│ Header │Padding│  │ Meshtastic │   Meshtastic   │
 │       │CR  │CRC│  CRC   │       │  │   Header   │ Payload (Proto)│
 └───────┴────┴───┴────────┴───────┘  └┬──────────┬┴────────────────┘
                      ┌────────────────┘          └──────────────────────┐
                      │4 bytes  4 bytes  4 bytes  1 byte  1 byte  2 bytes│
                     ┌┴───────┬────────┬────────┬───────┬───────┬────────┴┐
                     │  Dest  │  From  │ PktId  │ Flags │ Hash  │  Align  │
                     └────────┴────────┴────────┴───────┴───────┴─────────┘
```

The goal is to decode the encrypted Meshtastic payload.
Meshtastic uses AES256-CTR and we already know the key from the flash dump.
AES in counter mode also require knowning a nonce to decrypt.
We find the nonce generation function in the source code:
```C
// ./src/mesh/CryptoEngine.cpp

/**
 * Init our 128 bit nonce for a new packet
 */
void CryptoEngine::initNonce(uint32_t fromNode, uint64_t packetId)
{
    memset(nonce, 0, sizeof(nonce));

    // use memcpy to avoid breaking strict-aliasing
    memcpy(nonce, &packetId, sizeof(uint64_t));
    memcpy(nonce + sizeof(uint64_t), &fromNode, sizeof(uint32_t));
}
```


Using this knowledge, we build the following Python script to decode our frames:
```python
def dec(nonce, payload):
    key = bytes.fromhex("cef8db8e8e6017fd6dcca21db8a1476d451480acd7f4f9f769a763f528c011f7")
    nonce = bytes.fromhex(nonce)
    cipher = AES.new(key, AES.MODE_CTR, nonce=b"", initial_value=nonce)
    print(cipher.decrypt(bytes.fromhex(payload)).hex())

dec("a620df2d 00000000 bc4b6cfa 00000000", "5afacd43977600361b8cc3f50fbd5d")
# here please
dec("8899c202 00000000 c4c06dfa 00000000", "5001502e35d73ab793c3e9ad6acc9da6a4955bc56b8054e9989d76f5355cf2868b90bffae321d51077be4e1774fc074f63a4c0af6ba38f33a829b0784bdadbcc738b16e930e741c48f4d6c0cab012e5605122dce40eaeb7b")
# alright alright. the key is rwctf{No_h0p_th1s_tim3_c831bcad725935ba25c0a3708e49c0c8}
dec("5f8a5422 00000000 c4c06dfa 00000000", "531f89a36fea3018c9ceb7e71fa3cd7271ed")
# keep it secret
dec("86cc4a30 00000000 c4c06dfa 00000000", "0b0e97b3a161aab9a502b145f06b98ef6a740782311ac65373")
# I'll see you tomorrow
```

The flag is `rwctf{No_h0p_th1s_tim3_c831bcad725935ba25c0a3708e49c0c8}` !
