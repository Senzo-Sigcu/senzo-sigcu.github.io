---
layout: page
title: "Detecting the vsftpd Compromise: A Blue-Team View"
platform: Metasploitable 2
skills: [Log Analysis/Detection, Network Forensics, Incident Response, Reporting]
date: 2026-11-13
writeup: true
---

This write-up is the other half of the [Metasploitable full compromise]({{ '/writeups/metasploitable-2/' | relative_url }}). There, I attacked a host to root. Here, I re-ran the same attack and reconstructed it from the defender's chair, using only the network capture and the host logs, the way a SOC analyst reconstructs an intrusion from the evidence it leaves behind. The point is to show the same event from the other side: what the attack looked like to a defender, and which signals would have caught it.

## The attack, in one line

An unauthenticated attacker at `192.168.56.102` triggered the vsftpd 2.3.4 backdoor (CVE-2011-2523) on the target, gained a root shell, and created a persistence account. None of that requires the attacker's view to detect. All of it left a trail.

## The analyst's approach

I worked the evidence the way a SOC triages a suspected intrusion:

1. **The network trigger:** what started it, captured on the wire.
2. **The origin:** which host, confirmed from a service log.
3. **The action:** what the attacker did once in, confirmed from a second, independent log.
4. **Correlation:** tying the timestamps and the source together into one story.

The strength of the case comes from two independent sources agreeing, not from any single line.

## 1. The network trigger

A packet capture during the attack shows the backdoor being triggered in plain text on the FTP control channel (port 21):

![FTP TCP stream showing the backdoor trigger]({{ '/writeups/detection-metasploitable/images/01-ftp-trigger-stream.png' | relative_url }})

```
220 (vsFTPd 2.3.4)
USER sFyJA:)
331 Please specify the password.
PASS RrlHt7
```

The `USER` value ending in `:)` is the literal backdoor trigger. The banner `220 (vsFTPd 2.3.4)` names the vulnerable build. This is the single most detectable artefact of the whole attack, and it points straight at a detection rule (see below).

The filtered capture shows the same exchange in the packet list, followed by the reverse shell opening on port 4444, all between `192.168.56.102` and the target:

![Filtered packet list showing the trigger and the reverse shell]({{ '/writeups/detection-metasploitable/images/02-packet-list-filtered.png' | relative_url }})

A note on the shell channel: following the port-4444 stream returns binary, not text, because it carries a compiled Meterpreter payload.

![Reverse shell stream is opaque binary]({{ '/writeups/detection-metasploitable/images/03-reverse-shell-opaque.png' | relative_url }})

That is itself a finding: you cannot read the shell traffic, so detection has to come from the FTP trigger and the host logs, not from inspecting the shell. Signature tools that only read payloads will miss it; the behaviour around it is what gives the attacker away.

## 2. The origin (vsftpd.log)

The target's FTP service log records the connections, stamped with the attacker's IP and the time:

![vsftpd.log showing the connection from the attacker]({{ '/writeups/detection-metasploitable/images/04-vsftpd-log.png' | relative_url }})

```
Thu Nov 26 03:02:04 2026 [pid 4788] CONNECT: Client "192.168.56.102"
Thu Nov 26 03:02:04 2026 [pid 4790] CONNECT: Client "192.168.56.102"
```

Two connections from `192.168.56.102` at `03:02:04`. The exploit opens a pair of FTP connections to trigger and then use the backdoor, which is exactly what this shows. This is the *who* and *when*: an external host reaching the FTP service at the moment of compromise.

## 3. The action (auth.log)

The authentication log records what the attacker did once inside, the creation of an unauthorised account:

![auth.log showing the account creation]({{ '/writeups/detection-metasploitable/images/05-authlog-account.png' | relative_url }})

```
Nov 26 03:05:10 metasploitable useradd[4806]: new group: name=analyst1, GID=1008
Nov 26 03:05:10 metasploitable useradd[4806]: new user: name=analyst1, UID=1004, GID=1008, home=/home/analyst1, shell=/bin/bash
```

An account named `analyst1` was created at `03:05:10`. Nothing on the host authorised this. Unauthorised account creation is one of the highest-value detections a defender has, because it is both rare in normal operation and a direct sign of an attacker establishing persistence.

## 4. Correlation

Neither log tells the whole story alone. `vsftpd.log` shows an external host connected but not what it did; `auth.log` shows an account appeared but not who created it. Read together, on the same timeline, they are conclusive:

| Time | Source | Event |
|------|--------|-------|
| 03:02:04 | vsftpd.log | FTP connection from `192.168.56.102` |
| 03:02 (capture) | network | `USER ...:)` backdoor trigger, reverse shell to 4444 |
| 03:05:10 | auth.log | unauthorised account `analyst1` created |

An unknown external host reached the FTP service, triggered a known backdoor, and three minutes later an unauthorised account appeared on the host. That is a complete intrusion reconstructed from two independent sources plus the capture, which is exactly what an incident report needs.

## Detection rules this suggests

- **Network signature:** alert on any FTP `USER` command whose value contains `:)`. This is a reliable, low-false-positive signature for the vsftpd 2.3.4 backdoor, because no legitimate username ends that way. (Snort/Suricata content match on the FTP `USER` argument.)
- **Host rule:** alert on account or group creation, by watching `/var/log/auth.log` for `useradd` / `groupadd`, or the audit subsystem for writes to `/etc/passwd` and `/etc/shadow`. This should be a standing alert in any SOC.
- **Context:** an inbound connection to a legacy FTP service from an unexpected host, especially one followed by an outbound connection to an unusual high port (4444 here), is worth correlating.

## Indicators of compromise

| Type | Value |
|------|-------|
| Source IP | `192.168.56.102` |
| Network signature | FTP `USER` argument ending in `:)` |
| Vulnerable service | vsftpd 2.3.4 (CVE-2011-2523) |
| Reverse shell | outbound TCP to port 4444 |
| Persistence | local account `analyst1` (UID 1004) |
| Exposure | `/etc/shadow` read; `root` uses weak `$1$` MD5-crypt hashing |

## Recommendations

| Finding | Control |
|---------|---------|
| Backdoored vsftpd 2.3.4 | Remove or replace with a current signed build; restrict or retire FTP in favour of SFTP. |
| Cleartext FTP credentials | FTP transmits logins in plain text (visible above); move to an encrypted protocol. |
| Unauthorised account creation went unalerted | Add a standing alert on `useradd`/`groupadd` and on changes to `/etc/passwd`. |
| Backdoor trigger undetected | Deploy the FTP `USER ...:)` network signature on the IDS. |
| Weak password hashes (`$1$`) | Migrate to yescrypt or SHA-512 and force a reset. |
| Reverse shell reached the attacker | Apply default-deny egress filtering; alert on outbound connections to unusual high ports. |

## Lessons learned

- **Two weak sources beat one strong one.** Neither log was conclusive alone. The confidence came from correlating the origin (`vsftpd.log`) with the action (`auth.log`) on a shared timeline. That correlation habit is the core of the analyst's job.
- **Some of the attack is invisible, and that tells you where to look.** The shell channel was opaque binary. Rather than a dead end, that pointed detection toward the parts that are not hidden: the plaintext trigger and the host logs.
- **The loudest event is the persistence step.** The scan and the backdoor are quiet; the account creation is unmissable if you are watching for it. A defender's effort is best spent alerting on the actions an attacker cannot avoid leaving behind.
