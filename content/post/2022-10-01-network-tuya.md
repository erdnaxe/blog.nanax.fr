---
categories:
- Hardware
date: "2022-10-01"
title: "Write-up Tuya (TEC Qualifiers 2023)"
description: "Write-up of the Tuya challenge of CTF Qualifiers for Team Europe Candidates 2023."
keywords: ["Tuya", "CTF", "2023", "team", "europe", "network", "wireshark", "UDP", "SSID"]
---

## Official description

> This is a network forensic challenge. Please analyze the provided network
> dump. During a forensics mission, CERT was able to identify suspicious traffic
> from a specific laptop. In fact, by investigating the laptop, it seems that it
> was compromised and a popular script was used in order to configure Tuya
> devices inside the internal network. Can you exfiltrate the SSID and password?

We are given a network capture trace `TuyaDevice.pcapng`, a flag format
`FLAG{SSID-Password}` and two hints: `Tuya Devices` and `Colin Kuebler`.

## Exploration

We open `TuyaDevice.pcapng` in Wireshark.

{{< figure src="/assets/images/network-tuya/wireshark.png" title="TuyaDevice.pcapng in Wireshark" >}}

As a reflex, let's start by looking for clear HTTP traffic using the `http`
filter. We observe the following:

  * A GET request to `/` which returns 200.
  * A GET request to `/favicon.ico` which returns 404. This is expected as most
    web browsers tend to try this path even if no favicon is specified.

By selecting the `GET / HTTP/1.1` request and then `Follow > HTTP Stream` we get
the following:

```
GET / HTTP/1.1
Host: 192.168.1.55:8000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Pragma: no-cache
Cache-Control: no-cache

HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.2
Date: Thu, 26 May 2022 16:38:33 GMT
Content-type: text/html; charset=utf-8
Content-Length: 573

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="__pycache__/">__pycache__/</a></li>
<li><a href="broadcast.py">broadcast.py</a></li>
<li><a href="crc.py">crc.py</a></li>
<li><a href="main.py">main.py</a></li>
<li><a href="multicast.py">multicast.py</a></li>
<li><a href="smartconfig.py">smartconfig.py</a></li>
</ul>
<hr>
</body>
</html>
```

The challenge hints that the laptop uses a "popular script [...] used in order
to configure Tuya". Searching for the files, we find
[tuya-convert GitHub repository](https://github.com/ct-Open-Source/tuya-convert/blob/v2.4.5/scripts/smartconfig/smartconfig.py).
This script enables users of older Tuya devices (such as light bulbs) to flash
them with an open-source firmware such as [Tasmota](https://tasmota.github.io/).

Looking at the source code of tuya-convert we understand that all the UDP
packets containing only 0s might be generated by this script:

```Python
class SmartConfigSocket(object):
    # [...]
    def send_broadcast(self, data):
        # NOTE: The number of 0s in an UDP packet corresponds to an item of data
        for length in data:
            self._socket.sendto(b'\0' * length, ('255.255.255.255', 30011))
            sleep(self._gap)

def smartconfig(password, ssid, region, token, secret):
    sock = SmartConfigSocket()
    token_group = region + token + secret
    broadcast_body = encode_broadcast_body(password, ssid, token_group)
    # [...]
    for i in range(10):
        # [...]
        sock.send_broadcast(broadcast_body)
```

## Proposed solution

We extract the UDP packets corresponding to a single run of `smartconfig`
function, then implement a parser to retrieve the SSID and password.

### Extraction of the encoded configuration data

*This step can be done using [scapy](https://scapy.net/) rather than exporting
a JSON from Wireshark.*

We isolate a single run of `smartconfig()` in the network trace by fixing
the UDP source port which is chosen at random on `SmartConfigSocket`
instantiation: `udp.srcport == 51099 && ip.dst == 255.255.255.255`.

We get 740 packets that we export using
`File > Export Packet Dissections > As JSON...` to `dump.json`.

Then, we can load the UDP packets data in Python and retrieve `data` array
using:
```Python
import json

with open("dump.json") as f:
    pkt_data = json.load(f)

# inverse of "for length in data: self._socket.sendto(b'\0' * length, ('255.255.255.255', 30011))"
# see https://github.com/ct-Open-Source/tuya-convert/blob/v2.4.5/scripts/smartconfig/smartconfig.py
broadcast_body = []
for pkt in pkt_data:
    payload = bytes.fromhex("".join(pkt["_source"]["layers"]["udp"]["udp.payload"].split(":")))
    broadcast_body.append(len(payload))
```

### Decoding of the configuration data

We implement the inverse of `encode_broadcast_body` of [smartconfig/broadcast.py](https://github.com/ct-Open-Source/tuya-convert/blob/v2.4.5/scripts/smartconfig/broadcast.py).

```Python
# Skip header
broadcast_body = broadcast_body[160:]

# Decode first call of sock.send_broadcast(broadcast_body) to retrieve `r`
broadcast_body.pop(0)  # e.append(length >> 4 | 16)
broadcast_body.pop(0)  # e.append(length & 0xF | 32)
broadcast_body.pop(0)  # e.append(length_crc >> 4 | 48)
broadcast_body.pop(0)  # e.append(length_crc & 0xF | 64)
r = bytearray()
for i in range(9):
    broadcast_body.pop(0)  # e.append(group_crc & 0x7F | 128)
    sequence = broadcast_body.pop(0) ^ 128  # e.append(sequence | 128)
    b = bytearray(4)
    b[0] = (broadcast_body.pop(0)) ^ 256
    b[1] = (broadcast_body.pop(0)) ^ 256
    b[2] = (broadcast_body.pop(0)) ^ 256
    b[3] = (broadcast_body.pop(0)) ^ 256
    assert i == sequence
    r += b

# Unpack `r`
print("Retrieved the following r:", r)
len_password = r.pop(0)
password = r[:len_password].decode()
r = r[len_password:]
print(f"{len_password=} {password=}")

len_token_group = r.pop(0)
token_group = r[:len_token_group].decode()
r = r[len_token_group:]
print(f"{len_token_group=} {token_group=}")

print("SSID:", r.decode())
```

We get:
```
Retrieved the following r: bytearray(b'\x0caebbcc123665\x0eUS000000000101CTF-AP\x00\x00')
len_password=12 password='aebbcc123665'
len_token_group=14 token_group='US000000000101'
SSID: CTF-AP
```

Using the flag format from this challenge description, we get:
`FLAG{CTF-AP-aebbcc123665}`

{{< figure src="/assets/images/network-tuya/bulb_disassembly.jpg" title="Disassembly of a Tuya light bulb" >}}