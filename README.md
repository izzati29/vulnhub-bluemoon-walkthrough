# VulnHub BlueMoon - System Hacking Walkthrough

## Objective
Perform system hacking (reconnaissance, enumeration, exploitation, privilege escalation) on the VulnHub machine **BlueMoon** (https://www.vulnhub.com/entry/bluemoon-2021,679/) as part of a Week 3 System Hacking assignment.

## Tools Used
- Kali Linux (VirtualBox, Bridged Adapter)
- netdiscover
- nmap
- gobuster
- ftp client
- curl
- searchsploit
- exiftool, strings, binwalk

## Environment Setup
- Attacker machine: Kali Linux
- Target machine: BlueMoon (Debian 10)
- Both VMs configured with **Bridged Adapter** network mode so they are on the same LAN and can communicate.

---

## 1. Reconnaissance

### Host Discovery
```
sudo netdiscover -r 172.20.10.0/24
```
Target identified: **172.20.10.3** (MAC: 08:00:27:00:9e:b1 — VirtualBox virtual NIC)

![netdiscover result](screenshots/netdiscover.png)

### Port Scanning
```
nmap -sV -sC -Pn 172.20.10.3
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.3 |
| 22/tcp | open | ssh | OpenSSH 7.9p1 Debian 10+deb10u2 |
| 80/tcp | open | http | Apache httpd 2.4.38 (Debian) — title: "BlueMoon:2021" |

![nmap scan](screenshots/nmap-scan.png)

---

## 2. Enumeration

### FTP (Port 21)
```
ftp 172.20.10.3
```
Anonymous login **failed** (530 Permission denied) — anonymous access is disabled on this server.

![ftp attempt](screenshots/ftp-attempt.png)

### HTTP (Port 80)
Visited `http://172.20.10.3` in the browser. The page displayed:
> " -- Welcome -- "
> "Are You Ready To Play With Me .....!"

along with an image captioned "THE GAME BEGINS".

![web page](screenshots/web-page.png)

### Directory Enumeration
```
gobuster dir -u http://172.20.10.3 -w /usr/share/wordlists/dirb/common.txt
```
Found: `/index.html` (200); `/.hta`, `/.htaccess`, `/.htpasswd`, `/server-status` (403 Forbidden).

Re-ran including image/file extensions to search for hidden files:
```
gobuster dir -u http://172.20.10.3 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,jpg,jpeg
```
Found two unlisted, dot-prefixed image files:
- `/.blue.jpg`
- `/.starts.jpeg`

![gobuster result](screenshots/gobuster.png)

### Investigating the Hidden Image Files
Confirmed both files are served by the web server:
```
curl -I http://172.20.10.3/.blue.jpg
curl -I http://172.20.10.3/.starts.jpeg
```
Both returned `200 OK` (Content-Type: image/jpeg). `robots.txt` was not present (404).

Downloaded both files for offline analysis:
```
wget http://172.20.10.3/.blue.jpg
wget http://172.20.10.3/.starts.jpeg
```

Checked metadata:
```
exiftool .blue.jpg .starts.jpeg
```
No unusual metadata fields found (standard JPEG/JFIF headers only).

Searched for embedded strings (credentials, usernames, flags, keys):
```
strings .blue.jpg | grep -i -E "pass|user|flag|ssh|key"
strings .starts.jpeg | grep -i -E "pass|user|flag|ssh|key"
```
No matches found.

Checked for embedded/appended files:
```
binwalk .blue.jpg
binwalk .starts.jpeg
binwalk -e .blue.jpg
binwalk -e .starts.jpeg
```
Both files reported only as standard "JPEG image data, JFIF standard 1.01" with no embedded files detected by binwalk.

![file analysis - wget and exiftool](screenshots/file-analysis-1.png)
![file analysis - strings and binwalk](screenshots/file-analysis-2.png)

**Note:** A `steghide` check (`steghide info .blue.jpg`) was planned as a further step, since steghide-hidden data is not always detected by `strings`/`binwalk`, but this could not be completed due to a Kali package repository issue (404 errors) preventing installation in the available time.

### Exploit Research
```
searchsploit vsftpd 3.0.3
searchsploit OpenSSH 7.9
searchsploit Apache 2.4.38
```
- vsftpd 3.0.3: only a Remote DoS exploit listed (not useful for gaining access)
- OpenSSH 7.9: no direct exploits found
- Apache 2.4.38: no exploits directly applicable to this version/configuration

![searchsploit results](screenshots/searchsploit.png)

---

## 3. Exploitation (Not Completed)
No working exploit or valid credentials were found within the available time. The two hidden image files (`.blue.jpg`, `.starts.jpeg`) remain the most likely lead — their naming convention (hidden, dot-prefixed) suggests they were intentionally placed as a puzzle, but `strings` and `binwalk` analysis did not reveal hidden content, and a `steghide` check could not be performed due to a tooling/package repository issue.

---

## 4. Privilege Escalation (Not Completed)
Not reached — no initial shell access was obtained on the target.

---

## 5. Flag / Proof of Access (Not Completed)
Not reached.

---

## Conclusion
Reconnaissance and enumeration were completed successfully: three open services were identified (FTP 21, SSH 22, HTTP 80), anonymous FTP access was confirmed disabled (a good security practice), and extension-based directory brute-forcing on the web server uncovered two hidden image files not linked anywhere on the visible site. Metadata and string analysis of these files did not reveal obvious hidden data, and a deeper steganography check (steghide) could not be completed due to a package installation issue. Full exploitation, privilege escalation, and flag capture were not achieved within the available time. This exercise highlighted the value of extension-aware content discovery in finding non-obvious attack surface, and the importance of having analysis tools (steghide, etc.) pre-installed before starting time-boxed assessments.
