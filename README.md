# Vulnerable Active Directory Lab — Exploit Chain Project

A self-built, intentionally vulnerable Active Directory environment demonstrating a complete attack chain from unauthenticated initial access to full Domain Admin compromise. Built for offensive security learning and portfolio documentation.

> ⚠️ **This lab is intentionally insecure.** It must only ever run in an isolated network with no exposure to production systems or the public internet. See [Safety & Isolation](#safety--isolation) below.

---

## Overview

This project stands up a small multi-host Active Directory domain (`lab.local`) with deliberately planted misconfigurations, then documents a full, real-world-style attack path against it:

```
Vulnerable web app (WEB01)
        │  unrestricted file upload → webshell
        ▼
Local SYSTEM on WEB01
        │  unquoted service path
        ▼
Domain Admin credential in LSASS
        │  Mimikatz memory dump
        ▼
SYSTEM on SRV01
        │  Pass-the-Hash
        ▼
svc_sql credentials
        │  Kerberoasting + offline cracking
        ▼
DOMAIN ADMIN
        │  DCSync (abused replication rights)
```

Full technique-by-technique documentation — including detection signatures and remediation for every stage — lives in [`AD_Exploit_Chain_Documentation.md`](./AD_Exploit_Chain_Documentation.md).

---

## Environment

| Host | Role | OS |
|---|---|---|
| DC01 | Domain Controller | Windows Server 2022 |
| WEB01 | Vulnerable web server (initial access) | Windows Server 2019/2022 |
| SRV01 | File/SQL server (lateral movement target) | Windows Server 2019/2022 |
| WKS01 | Workstation | Windows 10/11 |
| Kali | Attacker box | Kali Linux |

**Domain:** `lab.local` / NetBIOS `LAB`
**Hosting:** AWS EC2, isolated VPC (`10.10.0.0/24`), no public internet exposure on domain-joined hosts

---

## Intentional Vulnerabilities

| # | Vulnerability | Host | Purpose in chain |
|---|---|---|---|
| 1 | Unrestricted file upload (`upload.aspx`) | WEB01 | Initial access / RCE |
| 2 | Unquoted service path, writable service directory | WEB01 | Local privilege escalation → SYSTEM |
| 3 | Domain Admin credential left in LSASS memory | WEB01 | Credential dumping |
| 4 | NTLM authentication accepted (Pass-the-Hash viable) | SRV01 | Lateral movement |
| 5 | SPN registered on a weak-password account (`svc_sql`) | SRV01 / AD | Kerberoasting |
| 6 | Excessive AD replication rights (`Replicating Directory Changes[/All]`) granted to `svc_sql` | DC01 | Domain dominance via DCSync |

Full setup steps for seeding each vulnerability are in [`SETUP.md`](./SETUP.md) *(or see the provisioning section below)*.

---

## Repository / Project Contents

```
.
├── README.md                          # this file
├── AD_Exploit_Chain_Documentation.md  # full exploit chain writeup: commands,
│                                       #   evidence, detection, remediation
├── scripts/
│   ├── provision-ad-objects.ps1       # OU structure, svc_sql, jsmith, da_admin, DCSync grant
│   └── webshell.aspx                  # test webshell payload for WEB01
└── screenshots/                       # evidence captured during the attack walkthrough
```

---

## Quick Start

1. **Provision infrastructure** — VPC, subnet, security groups locked to your IP only, and 5 EC2 instances (DC01, WEB01, SRV01, WKS01, Kali).
2. **Promote DC01** to a domain controller for `lab.local`, point VPC DNS at it.
3. **Domain-join** WEB01, SRV01, WKS01.
4. **Snapshot** (`clean-domain-joined`).
5. **Seed vulnerabilities** — run `scripts/provision-ad-objects.ps1` on DC01, configure the unquoted service and vulnerable upload page on WEB01, and log in once as `da_admin` on WEB01 to seed the LSASS credential.
6. **Snapshot again** (`vulnerable-and-ready`).
7. **Run the attack chain** end-to-end from Kali, following [`AD_Exploit_Chain_Documentation.md`](./AD_Exploit_Chain_Documentation.md).
8. **Capture evidence** (command output, screenshots) into the documentation as you go.

---

## Tools Used

- **Recon/Enumeration:** BloodHound + SharpHound, WinPEAS
- **Local Privesc:** PowerUp.ps1
- **Credential Access:** Mimikatz
- **Lateral Movement:** Impacket (`psexec.py`), Rubeus
- **AD Exploitation:** Impacket (`GetUserSPNs.py`, `GetNPUsers.py`, `secretsdump.py`, `ticketer.py`), Hashcat

---

## Safety & Isolation

- All lab hosts sit in a dedicated VPC/subnet with **no public IPs** except an optional locked-down bastion.
- Security groups restrict RDP/WinRM to the operator's IP only — never `0.0.0.0/0`.
- Credentials, hashes, and passwords documented in this project (e.g., `Summer2024!`) are **lab-only test values** and are not reused anywhere else.
- The environment should be torn down (or instances deallocated/terminated) when not actively in use.
- Nothing in this repository is intended for use against systems the operator does not own or have explicit authorization to test.

---

## Purpose

This project was built as a hands-on learning exercise and portfolio piece covering:
- AD misconfiguration identification and abuse (Kerberoasting, ACL/rights abuse, credential exposure)
- Standard offensive tradecraft (webshells, local privesc, Pass-the-Hash, DCSync)
- Translating offensive findings into **defensive** value — every technique is paired with detection logic and remediation guidance in the main documentation.

---

*Educational / portfolio use only. Built and tested exclusively in an isolated, self-owned lab environment.*
