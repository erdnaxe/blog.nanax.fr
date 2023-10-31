---
categories:
- System administration
date: "2023-10-31"
title: "Setup of Yubikey PIV applet for age and SSH"
description: "Quick guide to correctly setup Yubikey PIV applet to use age and SSH."
keywords: ["Yubikey", "PIV", "applet", "age", "SSH"]
---

## Yubikey PIV applet factory reset

> **Warning:**
> This step will erase all the keys in the PIV applet on your Yubikey and restore default PIN, PUK and management key.
> It does not reset the other applets such as FIDO.

Install Yubikey Manager CLI (ykman), then reset your PIV applet:

```
$ ykman piv reset -f
Resetting PIV data...
Success! All PIV data have been cleared from the YubiKey.
Your YubiKey now has the default PIN, PUK and Management Key:
  	PIN:	123456
  	PUK:	12345678
  	Management Key:	010203040506070801020304050607080102030405060708
```

## Initial setup and age key setup

Now that the Yubikey PIV applet is back to factory settings, we want to make some configuration changes:

  - change default PIN and PUK,
  - configure PIN and touch policies,
  - migrate to a PIN-protected management key.

[age-plugin-yubikey](https://github.com/str4d/age-plugin-yubikey) proposes to configure all these with a wizard.
It also generates a key in slot 1 (82) for encryption purposes with age.

```
$ age-plugin-yubikey
✨ Let's get your YubiKey set up for age! ✨

This tool can create a new age identity in a free slot of your YubiKey.
It will generate an identity file that you can use with an age client,
along with the corresponding recipient. You can also do this directly
with:
    age-plugin-yubikey --generate

If you are already using a YubiKey with age, you can select an existing
slot to recreate its corresponding identity file and recipient.

When asked below to select an option, use the up/down arrow keys to
make your choice, or press [Esc] or [q] to quit.

🔑 Select a YubiKey: Yubico YubiKey FIDO+CCID 00 00 (Serial: REDACTED)
🕳️  Select a slot for your age identity: Slot 1 (Empty)
📛 Name this identity [age identity TAG_HEX]: PASSAGE
🔤 Select a PIN policy: Once   (A PIN is required once per session, if set)
👆 Select a touch policy: Always (A physical touch is required for every decryption)
Generate new identity in slot 1? yes


Enter PIN for YubiKey with serial REDACTED (default is 123456): [hidden]

✨ Your YubiKey is using the default PIN. Let's change it!
✨ We'll also set the PUK equal to the PIN.

🔐 The PIN is up to 8 numbers, letters, or symbols. Not just numbers!
❌ Your keys will be lost if the PIN and PUK are locked after 3 incorrect tries.

Enter current PUK (default is 12345678): [hidden]
Choose a new PIN/PUK: [hidden]

✨ Your YubiKey is using the default management key.
✨ We'll migrate it to a PIN-protected management key.
... Success!
👆 Please touch the YubiKey

📝 File name to write this identity to: age-yubikey-identity-REDACTED.txt

✅ Done! This YubiKey identity is ready to go.

🔑 Here's your shiny new YubiKey recipient:
  age1yubikey1REDACTED

Here are some example things you can do with it:

- Encrypt a file to this identity:
  $ cat foo.txt | rage -r age1yubikey1REDACTED -o foo.txt.age

- Decrypt a file with this identity:
  $ cat foo.txt.age | rage -d -i age-yubikey-identity-REDACTED.txt > foo.txt

- Recreate the identity file:
  $ age-plugin-yubikey -i --serial REDACTED --slot 1 > age-yubikey-identity-REDACTED.txt

- Recreate the recipient:
  $ age-plugin-yubikey -l --serial REDACTED --slot 1

💭 Remember: everything breaks, have a backup plan for when this YubiKey does.
```

You may verify that the Yubikey PIV applet is now correctly configured:

```
$ ykman piv info
PIV version:              5.4.3
PIN tries remaining:      3/3
Management key algorithm: 3
Management key is stored on the YubiKey, protected by PIN.
CHUID: No data available
CCC:   No data available
Slot 82:
  Algorithm:   ECCP256
  Subject DN:  CN=PASSAGE,OU=0.3.0,O=age-plugin-yubikey
  Issuer DN:   CN=PASSAGE,OU=0.3.0,O=age-plugin-yubikey
  Serial:      REDACTED
  Fingerprint: REDACTED
  Not before:  2022-12-31T16:33:31
  Not after:   9999-12-31T23:59:59
```

## SSH key setup

Let's generate a SSH key in slot 9a with a PIN and touch policy:

```
$ ykman piv keys generate -a ECCP256 --pin-policy ONCE --touch-policy ALWAYS 9a public.pem
Enter PIN:
$ ykman piv certificates generate -s "CN=SSH" 9a public.pem
Enter PIN:
Touch your YubiKey...
$ ssh-keygen -D /usr/lib/opensc-pkcs11.so
ecdsa-sha2-nistp256 <public SSH key>
```

To use this key, you need to hint the provider to your SSH client.
You may add to `~/.ssh/config`:

```
PKCS11Provider /usr/lib/opensc-pkcs11.so
```

Now everything should be ready to go!
**Please make sure you have a backup plan if you lose the Yubikey or forget its PIN.**
