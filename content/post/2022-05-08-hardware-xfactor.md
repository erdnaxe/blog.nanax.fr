---
categories:
- Hardware security
date: "2022-05-08"
title: "Write-up X-Factor (FCSC 2022)"
description: "Write-up of the X-Factor challenge of French Cybersecurity Challenge 2022."
keywords: ["FCSC", "2022", "x-factor"]
---

## Official description

> Un commanditaire vous a demandé de récupérer les données ultra secrètes d'une entreprise concurrente. Vous avez tenté plusieurs approches de recherche de vulnérabilités sur les serveurs exposés qui se sont malheureusement révélées infructueuses : les serveurs de l'entreprise ont l'air solides et bien protégés. L'intrusion physique dans les locaux paraît complexe vu tous les badges d'accès nécessaires et les caméras de surveillance.
> 
> Une possibilité réside dans l'accès distant qu'ont les employés de l'entreprise à leur portail de travail collaboratif : l'accès à celui-ci se fait via deux facteurs d'authentification, un mot de passe ainsi qu'un token physique à brancher sur l'USB avec reconnaissance biométrique d'empreinte digitale. Même en cas de vol de celui-ci, il sera difficile de l'exploiter. Installer un malware evil maid sur un laptop de l'entreprise n'est pas une option : ceux-ci sont très bien protégés avec du secure boot via TPM, et du chiffrement de disque faisant usage du token.
> 
> Mais tout espoir n'est pas perdu ! Vous profitez du voyage en train d'un des employés et de sa fugace absence au wagon bar pour brancher discrètement un sniffer USB miniaturisé sur le laptop. Vous glissez aussi une caméra cachée au dessus de son siège qui n'a pu capturer que quelques secondes. Vous récupérez la caméra et le sniffer furtivement après sa séance de travail : saurez-vous exploiter les données collectées pour mener à bien votre contrat ?
> 
> Pour obtenir le flag de `X-Factor 1/2`, vous devez vous logguer avec login et mot de passe. Puis avec le deuxième facteur d'authentification pour obtenir le flag pour `X-Factor 2/2`.
> 
> SHA256(`capture_USB.pcapng`) = 1543448477f925070b598f306b59a97610ba013...
>
> SHA256(`login_password.mkv`) = 0b1361a0261a2cdb13d47d4629c8ec93dc5a7e8...

## X-Factor 1/2 recap

`X-Factor 1/2` is a "misc" challenge that was scored by 207 people for 140
points. It consists in getting credentials from `login_password.mkv`.

```bash
mkdir -p login_password_video/
ffmpeg -i login_password.mkv login_password_video/frame_%04d.png
feh login_password_video/  # then use arrows to jump to frames
```

From this video, we get:

> Login: `john.doe@hypersecret`
>
> Password: `jesuishypersecretFCSC2022`
>
> URL: <https://x-factor.france-cybersecurity-challenge.fr/login>

![Login page on Firefox 99](/assets/images/hardware-xfactor/login.png)

After login in, we get a flag.
We were able to recover the password because of the login page JavaScript slowly
replacing clear password characters with "`*`" asterisks.

## Exploration

![Two-factor login page](/assets/images/hardware-xfactor/login2.png)

After login we are prompted for a second factor token through the browser
Universal 2nd Factor (U2F) API.
*The name of the challenge now makes sense.*

### Web page exploration

If we cancel the browser U2F prompt or use a random U2F token (such as [rust-u2f](https://github.com/danstiner/rust-u2f) emulator or a hardware token),
then we get `Bad second factor authentication!` error.

Let's look at the U2F login page source code and list points of interest:

  * `Check Token` button calls `beginAuthen('UezmElyJs4+StNBS<snip>')`
    on click, the parameter looks like base64 and does not seem to change.
  * `js/u2f-api.js` and `js/util.js` scripts are imported. This first script
    is a JavaScript polyfill by Google to get an U2F API and the second one
    contains some helper functions.
  * `beginEnroll`, `finishEnroll`, `beginAuthen` and `finishAuthen` JavaScript
    functions are defined.

At first glance `beginEnroll` and `finishEnroll` seems interesting as this may
allow us to register a new U2F token, but `/beginEnroll` and `/finishEnroll`
return HTTP 404 with the following error:

```
Enrollment is not allowed on this portal. Please contact your administrator
```

Let's analyse the login flow when `Check Token` is clicked:

  * `beginAuthen` is called with a `keyHandle` parameter,
  * HTTP GET request is sent to `/beginAuthen` with `keyHandle`,
  * On response, we get a `startAuthen` object from the server containing:
      - the expected U2F version (`startAuthen.version`),
      - a key handle (`startAuthen.keyHandle`),
      - an app identifier (`startAuthen.appId`),
      - a challenge (`startAuthen.challenge`).
  * The browser U2F API is called to sign the data in `startAuthen`,
  * After signing or failure, `finishAuthen` is called with U2F returned data,
  * HTTP GET request is sent to `/finishAuthen` with the returned data,
  * The user is redirected to `/check`.

Let's comment `location.assign('/check')` and print to console intermediate
data:

  * `startAuthen.version` is `U2F_V2`,
  * `startAuthen.keyHandle` is a constant web-safe base64[^websafeb64],
  * `startAuthen.appId` is `https://x-factor.france-cybersecurity-challenge.fr`,
  * `startAuthen.challenge` looks like a random web-safe base64 value.
    On closer inspection, it takes randomly one of the following values:

    ```
    L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI
    D5CxgaFPGIQu5fGYPEjo-YA9Dqd6y2PBoWP6p56TpFw
    9rlDOo98PIKIiubib97v4IDCJ1FBB2uRUhNgwH89wqw
    ```

[^websafeb64]: base64 variant in which (`/`, `+`, `=`) is replaced by (`_`, `-`, '').

### USB capture exploration

We are given a `capture_USB.pcapng` trace. Opening it directly in Wireshark
shows USB traffic. It starts with `GET DESCRIPTOR` exchanges containing
`idVendor=0x0000` and `idProduct=0x1337` and a USB HID configuration.
As the vendor identifier is null, we cannot determine which is exactly this
device, but it is a USB HID device and there is a high probability this is
an U2F token.

Wireshark is unable to dissect the following packets further than the USB URB
layer. Let's search for a U2F dissector.
We use [u2f_fido2_dissector.lua by Yuxiang Zhang](https://gist.github.com/z4yx/218116240e2759759b239d16fed787ca), but with the following extra line to detect
the custom U2F token:

```Lua
usb_table:add(0x00001337,ctap_proto) -- VID/PID of custom
```

Now we may start Wireshark with this extra dissector:

```bash
wireshark capture_USB.pcapng -X lua_script:u2f_fido2_dissector.lua
```

We now see CTAPHID layer with ISO 7816 APDU.

![ISO 7816 messages from USB capture](/assets/images/hardware-xfactor/wireshark.png)

We confirm that it is indeed an U2F token.
We download the [Universal 2nd Factor (U2F) Overview from fidoalliance.org](https://fidoalliance.org/specs/fido-u2f-v1.2-ps-20170411/FIDO-U2F-COMPLETE-v1.2-ps-20170411.pdf) and start
coloring packets with filters.
In previous screenshot, red packets are 0x6985 errors, meaning that the token
did not find user presence. Green packets are successful signatures.
Blue packets are version responses containing `U2F_V2`.

The dissector is not ideal and does not properly detect the signature message,
but we can use the hexadecimal dump to get these values.

After a bit of documentation reading and dissection, we recover 11 successful
requests/responses signatures. The request has the following structure:

  * Challenge parameter (32 bytes), varying but we see repetitions,
  * Application parameter (32 bytes), constant,
  * Key length \\(L\\) (1 byte), `0x40` constant,
  * Key handle (\\(L\\) bytes), constant.

The response has the following structure:

  * User presence (1 byte), `0x01` constant,
  * Counter (4 bytes), `0x00000000` **constant**,
  * An ASN.1 ECDSA signature of the concatenation of the application parameter,
    user presence, counter and challenge parameter.

The key handle is checked by the U2F token. If it does not satisfy the U2F
token, it returns an error rather than giving a signature.

**Bad ECDSA nonces?**
I lost a lot of time in this challenge [trying to attack the ECDSA nonce](https://blog.trailofbits.com/2020/06/11/ecdsa-handle-with-care/) by
making the assumption that because this may be a custom U2F token, it might
not correctly generated nonces.
This did not succeed.

**No counter?**
As the returned counter is `0x00000000`, it must also be `0x00000000` in the
signed message, else the server wouldn't be able to check the signature.
As the fidoalliance.org overview states, this counter is a mitigation against
replay attacks, and it should be increasing at each signature.

## Proposed solution

We are going to attack the login flow by replaying what we captured in the USB
trace, but first we need to understand how the challenge received on the device
is derived from the given server challenge.

### From client data to token challenge

We need a working U2F authentification flow to analyse the object passed to 
`finishAuthen`.

**Option A: using a patched hardware token.**
I modified my Ledger Nano S+ [U2F app](https://github.com/LedgerHQ/app-u2f/) to
skip a check on the key handle by:

  * removing
[lines 206-207](https://github.com/LedgerHQ/app-u2f/blob/561adacce62082875e89f241acbcb59a3c14639e/src/u2f_processing.c#L206-L207)
in `u2f_processing.c`,
  * putting random `ATTESTATION_KEY` and `ATTESTATION_CERT`,
  * renaming the app to `X-factor`,
  * doing some last minute hacks to make the app compile with the latest SDK,
    this was time inefficient.

**Option B: using an online U2F demo.**
By doing the registration and signing process on
<https://mdp.github.io/u2fdemo/> we are able to quickly get the generated
response.

From this working login flow we deduce that `finishAuthen` is called with this object:

```json
{
  "clientData": "eyJjaGFsbGVuZ2UiOiJMOHRzQ2tERXJSUH<snip>",
  "errorCode": 0,  // no error
  "keyHandle": "MiXECXjEbxAAe7QOH2gsiNiK7bXeuGJnLUGO7kbJ<snip>",
  "signatureData": "AQAAAAAwRQIhANFIGfidyE1ywJlF49C5dtD1<snip>"
}
```

The web-safe base64 decoding of `clientData` gives:

```json
{
  "challenge": "L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI",
  "origin": "https://x-factor.france-cybersecurity-challenge.fr",
  "typ": "navigator.id.getAssertion"
}
```

According to fidoalliance.org overview, the challenge signed by the device is the
SHA256 of `clientData`. We are now able to build correspondence between server
and device challenge:

```Python
import hashlib
import base64

def sha256(val: str) -> bytes:
    h = hashlib.sha256()
    h.update(val.encode())
    return h.hexdigest()

sha256('{"challenge":"L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI","origin":"https://x-factor.france-cybersecurity-challenge.fr","typ":"navigator.id.getAssertion"}')
```

```
D5CxgaFPGIQu5fGYPEjo-YA9Dqd6y2PBoWP6p56TpFw
-> 9e5e67419d90aa711dda3c361678a4cd8ddf835051bc7369ccf14bf9da7bcfb3

L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI
-> 8736c5b6cb8b27617a7ccbec9f599ba460eefee042fe25b9b2a673bf43ddb1e8

9rlDOo98PIKIiubib97v4IDCJ1FBB2uRUhNgwH89wqw
-> 153db9a93297ea0b55d3a3e0898213c251ec3cf3ae40c8d6518642fc8b64fd0e
```

*Oh surprise!* **We recognize challenges from the USB capture.**

*This section could have been done with another unpatched U2F token by changing
`u2f.sign` application identifier and key handler to match values working on
another website.*

### Replaying captured signature

Let's go back to the U2F login page and override `beginAuthen()` function by
putting this in the developer console:

```javascript
function beginAuthen() {
    $.getJSON(
        "/beginAuthen",
        { keyHandle: "" },
        function (startAuthen) {
            console.log(startAuthen.challenge)
        },
    );
}
```

Now we call `beginAuthen()` and get `L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI`
challenge.
From the last section, we know that this correspond to the signature
of `8736c5b6cb8b27617a7ccbec9f599ba460eefee042fe25b9b2a673bf43ddb1e8` in the USB
capture.

We manually forge the response:

```Python
import base64
import json

# From USB capture
challenge = "L8tsCkDErRPzV9SAOlOj2JzFMXAOjmUs7JnimkH9_gI"
signature = ("01000000003046022100f60af84cfc0f3d9f33ae8ad04b617ab7"
             "cf782f70cd083a71aad738b4e75d51c0022100b43a629349cea1"
             "3415263b86a1afc637cd4c92dc8673a360577311710582f9da")

# Craft response
clientData = base64.b64encode(('{"challenge":"' + challenge + '","origin":"https://x-factor.france-cybersecurity-challenge.fr","typ":"navigator.id.getAssertion"}').encode()).replace(b"/", b"_").replace(b"+", b"-").replace(b"=", b"").decode()
keyHandle = "MiXECXjEbxAAe7QOH2gsiNiK7bXeuGJnLUGO7kbJutdODZvuqV-T1TPpTVEVIrynmScyNOjQaRAUi0PSH8LUtQ"
signatureData = base64.b64encode(bytes.fromhex(signature)).replace(b"/", b"_").replace(b"+", b"-").replace(b"=", b"").decode()

print(json.dumps({
    "clientData": clientData,
    "errorCode": 0,
    "keyHandle": keyHandle,
    "signatureData": signatureData
}))
```

Then send the response in the browser console:
```
finishAuthen({"clientData": "eyJj<snip>RcQWC-do"})
```

![Successful login and a flag](/assets/images/hardware-xfactor/login_success.png)

This challenge was really well designed.
During this challenge, I learned:

  * A lot about U2F internals,
  * More about APDU and SmartCards,
  * ECDSA bad nonce attacks, even if this was not the solution,
  * What is web-safe base64 and how to build it using base64 Python module.

![X-factor app emulating a modified U2F_V2 token](/assets/images/hardware-xfactor/ledgerapp.jpg)
