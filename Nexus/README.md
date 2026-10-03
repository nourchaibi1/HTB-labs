<div align="center">

# HTB — Nexus

**Easy · Linux · Hack The Box**

![Status](https://img.shields.io/badge/status-done-green)
![Difficulty](https://img.shields.io/badge/difficulty-easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)

</div>

---

## 📋 Machine Information

| Field          | Value           |
| -------------- | --------------- |
| **Target IP**  | `10.129.89.224` |
| **Hostname**   | `nexus.htb`     |
| **OS**         | Linux           |
| **Difficulty** | Easy            |
| **Status**     |  Rooted        |

---

##  Table of Contents

* [1. Reconnaissance](#1-reconnaissance)

  * [Port Scanning](#port-scanning)
  * [Virtual Host Enumeration](#virtual-host-enumeration)
* [2. Task Answers](#2-task-answers)
* [3. Git Repository Enumeration](#3-git-repository-enumeration)
* [4. Identifying Krayin Version](#4-identifying-krayin-version)
* [5. Initial Foothold](#5-initial-foothold)
* [6. Getting Access as jones](#6-getting-access-as-jones)
* [7. User Flag](#7-user-flag)
* [8. Privilege Escalation](#8-privilege-escalation)
* [9. Git Path Traversal](#9-git-path-traversal)
* [10. From File Write to Root](#10-from-file-write-to-root)
* [11. Root Flag](#11-root-flag)
* [12. Key Takeaways](#12-key-takeaways)

---

# 1. Reconnaissance

## Port Scanning

I started with an Nmap scan to identify the available TCP services:

```bash
nmap -sT 10.129.89.224
```

The scan showed:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

So there were two open TCP ports:

* `22` — SSH
* `80` — HTTP

### Task 1 — Open TCP Ports

**Question:** How many open TCP ports are listening on Nexus?

**Answer:**

```text
2
```

---

## Virtual Host Enumeration

The web server redirected requests to:

```text
nexus.htb
```

I added the hostname to `/etc/hosts`:

```bash
echo "10.129.89.224 nexus.htb" | sudo tee -a /etc/hosts
```

I then used FFUF to enumerate virtual hosts:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-H "Host: FUZZ.nexus.htb" \
-u http://nexus.htb
```

One of the interesting results was:

```text
git.nexus.htb
```

I added this hostname to `/etc/hosts` and continued enumerating the Git service.

---

# 2. Task Answers

## Task 2 — Hiring Manager Email

While browsing the main website, I found a job application section containing the hiring manager's email address.

**Answer:**

```text
j.matthew@nexus.htb
```

---

## Task 3 — Git Subdomain

The virtual host enumeration revealed:

```text
git.nexus.htb
```

**Answer:**

```text
git
```

---

## Task 4 — DB_PASSWORD

The exposed Git repository contained useful deployment information.

**Answer:**

```text
N27xh!!2ucY04
```

---

## Task 5 — Krayin CRM Version

After reviewing the repository and correlating the deployment information with the application's release history, I identified the Krayin CRM version as:

```text
2.2.0
```

**Answer:**

```text
2.2.0
```

---

# 3. Git Repository Enumeration

After accessing:

```text
git.nexus.htb
```

I found a repository named:

```text
krayin-docker-setup
```

I started reviewing the repository files to look for configuration information and possible credentials.

There was a `.env` file, but the required password was not present in the current version.

The repository only had a small commit history, so I decided to inspect the previous commit as well.

The older commit contained information that was no longer visible in the current version.

I found:

```text
DB_PASSWORD=N27xh!!2ucY04
```

This was an important finding because the recovered credential could also be useful for accessing the application.

### Why Git History Matters

Removing sensitive information from the latest version of a repository does not necessarily remove it from previous commits.

This was a good reminder to always inspect Git history when an exposed repository is part of the attack surface.

---

# 4. Identifying Krayin Version

The repository contained deployment information related to the Krayin CRM application.

The Docker configuration used:

```text
latest
```

instead of specifying an exact application version.

To identify the version, I correlated the deployment timeline with the Krayin release history.

The version identified for the target was:

```text
Krayin CRM 2.2.0
```

Knowing the exact version made the vulnerability research much more focused.

---

# 5. Initial Foothold

## CVE-2026-38526

After identifying the application and version, I searched for vulnerabilities affecting:

```text
Krayin CRM 2.2.0
```

The relevant vulnerability was:

```text
CVE-2026-38526
```

During further enumeration, I found that the Krayin administration interface was hosted on:

```text
billing.nexus.htb
```

I added the hostname to `/etc/hosts`.

The credentials recovered earlier from the Git repository were:

```text
Username: j.matthew@nexus.htb
Password: N27xh!!2ucY04
```

I used them to access the administration dashboard.

---

## Exploiting the Upload Functionality

The vulnerable functionality was related to the TinyMCE upload handler.

Using Burp Suite, I intercepted the upload request and sent it to the vulnerable endpoint:

```http
POST /admin/tinymce/upload
```

The file was sent with an image MIME type:

```text
Content-Type: image/png
```

The server accepted the upload and returned the location of the uploaded file.

I then prepared a listener on my machine:

```bash
nc -lvnp 4444
```

After requesting the uploaded PHP file, the payload executed on the target and connected back to my machine.

This resulted in a shell running as:

```text
www-data
```

At this point, I had my initial foothold on the machine.

---

# 6. Getting Access as `jones`

After obtaining access as `www-data`, I started checking the application files for configuration information.

The Krayin installation was located at:

```text
/var/www/krayin
```

I checked the `.env` file:

```bash
cat /var/www/krayin/.env
```

Among the database configuration values, I found:

```text
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```

The interesting part was that this password was also valid for the local `jones` account.

I tested the credential with:

```bash
su - jones
```

Using:

```text
y27xb3ha!!74GbR
```

The login succeeded.

I confirmed the account with:

```bash
whoami
```

Output:

```text
jones
```

### Password for `jones`

```text
y27xb3ha!!74GbR
```

---

# 7. User Flag

With access to the `jones` account, I connected through SSH:

```bash
ssh jones@10.129.89.224
```

Then I checked the user's files:

```bash
ls
cat user.txt
```


---

# 8. Privilege Escalation

After obtaining access as `jones`, I started enumerating the machine for possible privilege-escalation vectors.

One useful command was:

```bash
systemctl list-timers --all
```

This revealed:

```text
gitea-template-sync.timer
```

The timer triggered:

```text
gitea-template-sync.service
```

I inspected the service:

```bash
systemctl cat gitea-template-sync.service
```

The important section was:

```ini
[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
```

The service was therefore running the synchronization script as `root`.

---

# 9. Git Path Traversal

I then inspected the synchronization script:

```bash
cat /etc/gitea/template-sync.py
```

The script processed file paths obtained from a Git repository.

One important operation was:

```python
target = os.path.join(stage_path, filepath)
```

The `filepath` value was not properly validated.

This meant that a Git path containing traversal sequences such as:

```text
../../
```

could potentially escape the intended staging directory.

The script also relied on Git tree information:

```bash
git ls-tree -r HEAD
```

I investigated whether a Git tree could contain a path that would cause the synchronization process to write outside its expected directory.

To work with the Git tree directly, I used:

```bash
git mktree
git commit-tree
```

The resulting path traversal could reach:

```text
/etc/cron.d/
```

The synchronization logs confirmed that the traversal path was being processed.

---

# 10. From File Write to Root

Because the synchronization service was running as `root`, the path traversal resulted in an arbitrary file write with root privileges.

The writable location:

```text
/etc/cron.d/
```

could then be used to add a cron job.

After cron executed the job, I verified root-level execution with:

```bash
id
```

The result was:

```text
uid=0(root) gid=0(root) groups=0(root)
```

The complete privilege-escalation chain was:

```text
jones
   ↓
Gitea template repository
   ↓
Malicious Git tree
   ↓
Path traversal
   ↓
Root-owned synchronization service
   ↓
Arbitrary file write
   ↓
/etc/cron.d/
   ↓
Cron
   ↓
root
```

This was the most interesting part of the machine because the privilege escalation required chaining several findings together rather than relying on a single obvious misconfiguration.

---

# 11. Root Flag

After obtaining root-level command execution, I accessed:

```text
/root/root.txt
```

### Final Attack Chain

```text
HTTP
  ↓
Virtual Host Enumeration
  ↓
git.nexus.htb
  ↓
Git History
  ↓
Recovered Credentials
  ↓
Krayin CRM 2.2.0
  ↓
CVE-2026-38526
  ↓
www-data
  ↓
Credential Reuse
  ↓
jones
  ↓
gitea-template-sync.timer
  ↓
Git Path Traversal
  ↓
Root Arbitrary File Write
  ↓
Cron
  ↓
root
```

---

# 12. Key Takeaways

Some of the main things I learned from this machine:

* Enumerate virtual hosts when a web server redirects to a hostname.
* Always inspect Git history when investigating an exposed repository.
* Identifying the exact software version makes vulnerability research easier.
* Credentials found in application configuration files may be reused by local accounts.
* Systemd timers are worth checking during Linux privilege-escalation enumeration.
* Git-controlled paths should always be validated before being processed by privileged scripts.
* A path traversal vulnerability can become much more serious when combined with a root-owned service.
* Privilege escalation often comes from chaining several smaller findings together.

---

## 🛠️ Tools Used

* Nmap
* FFUF
* cURL
* Git
* Burp Suite
* Netcat
* SSH
* Linux/systemd
* Python

---

<div align="center">

**HTB — Nexus**

🏴‍☠️ **Machine Rooted**

</div>
