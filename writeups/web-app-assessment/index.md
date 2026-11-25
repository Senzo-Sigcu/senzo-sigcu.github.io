---
layout: page
title: "Web Application & Reconnaissance Assessment"
platform: scanme.nmap.org · Acunetix test sites · local lab
skills: [OSINT, Enumeration, Vulnerability Identification, Web Application Testing, Log Analysis/Detection, Reporting]
date: 2026-11-12
writeup: true
---

A full-surface assessment that moves from passive intelligence gathering, through active network reconnaissance, into web-application testing (intercepting proxy, SQL injection, XSS, CSRF) and a WPA2 handshake analysis. Each technique is paired with the defender's view: what it looks like from the other side, and which control stops it.

## Overview

- **Reconnaissance target:** `scanme.nmap.org`, the host the Nmap project publishes expressly for people to practise scanning against, legally.
- **Web-exploitation targets:** the Acunetix public vulnerable test sites (`testphp.vulnweb.com`, `testasp.vulnweb.com`), Google's `xss-game`, and a self-hosted CSRF demo on `127.0.0.1`.
- **OSINT target:** a live institutional website, used only for passive information gathering. The organisation and the specific findings are redacted here; this write-up shows the technique, not the target.
- **Attack host:** Kali Linux.
- **Goal:** exercise the web-application and reconnaissance toolchain end to end against authorised targets, and document the detection surface of each step.

Every active technique in this write-up was run against hosts that exist to be tested (scanme.nmap.org, the Acunetix sites, Google's XSS game) or against my own local machine. The OSINT section used only publicly served information. Nothing here was an unauthorised action against a third party.

## Executive Summary

The assessment demonstrates the full reconnaissance-to-exploitation path against web targets. Passive OSINT alone surfaced sensitive directory names and documents that an organisation had inadvertently exposed, before a single packet was sent. Active scanning then showed how service versions and a Denial-of-Service weakness (Slowloris, CVE-2007-6750) are identified, and how a stealth scan evades naive logging. Against intentionally vulnerable web applications, an intercepting proxy exposed credentials transmitted in clear text, and `sqlmap` confirmed and exploited SQL injection to enumerate databases and dump a user table. Reflected XSS and a forged cross-site request were demonstrated in controlled environments. The recurring theme is that most of these attacks are defeated by a small set of controls (input validation, TLS everywhere, anti-CSRF tokens, and egress/connection logging) applied consistently.

## OSINT / Passive Reconnaissance

Before touching the target, I built a picture from public sources only. Three techniques, each with a defensive takeaway (findings and the organisation are redacted):

- **Source and comment inspection**: the page source exposed HTML comments and meta tags revealing technologies in use. *Takeaway:* strip developer comments from production pages; they hand an attacker a technology map for free.
- **Archived history (Wayback Machine)**: older archived versions showed a different site structure and content no longer public. *Takeaway:* content is never truly deleted; assume anything once published is permanent.
- **Search operators (Google dorking)**: a `site:` query combined with a keyword surfaced sensitive policy documents and reporting pages. A read of `robots.txt` then advertised the exact private directory names the site was trying to hide. *Takeaway:* `robots.txt` is not an access control: listing a path there only tells an attacker where to look; use real server-side authorisation instead.

Combined, passive OSINT alone gives an attacker a detailed map of systems, technologies and sensitive paths before any active probing, which is precisely why it is hard to detect and must be managed at the source.

## Active Reconnaissance (scanme.nmap.org)

**Host discovery.** A simple ICMP ping confirmed the host was alive (4 sent, 4 received, 0% loss).

![ping confirming the host is alive]({{ '/writeups/web-app-assessment/images/01-ping-host-discovery.png' | relative_url }})

Interestingly, a probe-based discovery scan (`nmap -sn -PS443`) initially reported **0 hosts up**: the host answers ICMP echo but drops the probe used here. The lesson is to reach for `-Pn` (skip host discovery) when a host is known to be alive but appears "down", rather than concluding it is offline.

**Port scanning: connect vs stealth.** Two scan types were compared:

![TCP connect scan]({{ '/writeups/web-app-assessment/images/02-nmap-connect-scan.png' | relative_url }})

![TCP SYN stealth scan]({{ '/writeups/web-app-assessment/images/03-nmap-syn-stealth.png' | relative_url }})

The SYN scan (`-sS`) is called a *stealth* scan because it never completes the TCP three-way handshake: it sends a SYN, receives a SYN-ACK if the port is open, then immediately sends a RST to abort. Because the connection is never fully established, many older logging systems and firewalls never record it as a completed connection. (Timing is situational: here the connect scan actually finished faster; stealth is about detectability, not speed.)

**Service enumeration and banner grabbing.**

![nmap service and version detection]({{ '/writeups/web-app-assessment/images/04-nmap-service-version.png' | relative_url }})

`nmap -sV` identified port 80 as **Apache httpd 2.4.7 (Ubuntu)**, with 22 and 443 filtered. A manual banner grab with Netcat independently confirmed the same version string from the `Server:` header, a good habit, since cross-checking one tool against another guards against a single tool being fooled.

![netcat banner grab]({{ '/writeups/web-app-assessment/images/05-nc-banner-grab.png' | relative_url }})

**Vulnerability detection (non-intrusive).** An NSE vulnerability scan flagged the Slowloris DoS weakness:

![nmap vuln scan showing Slowloris]({{ '/writeups/web-app-assessment/images/06-nmap-vuln-slowloris.png' | relative_url }})

Slowloris (CVE-2007-6750) works by opening many connections and holding them open with partial requests, starving the server's connection pool and causing a Denial of Service. Importantly, the scanner only *reported* "LIKELY VULNERABLE": it detected without exploiting, and the server stayed up. `--script vuln` is safe reconnaissance, not an attack.

## Intercepting Proxy (Burp Suite)

An intercepting proxy sits between the browser and the server, letting you read, pause, modify and forward HTTP(S) requests before they reach their destination, an active man-in-the-middle position, in contrast to a passive sniffer like Wireshark, which only records and cannot alter traffic. Wireshark sees everything at the packet level; Burp works at the HTTP application layer.

Key findings from exercising Burp against test logins:

- **Credentials in clear text.** On a site served over plain HTTP, the intercepted `POST` showed the username and password in plain text in the request body. Even over HTTPS, Burp can still read the traffic because it acts as a trusted man-in-the-middle using its own certificate, which is exactly why TLS plus certificate awareness matters.
- **Repeater and IDOR testing.** Capturing a request once and replaying it in Repeater, changing a single parameter (for example `id=1` → `id=2`) and resending, is far faster than driving the browser for each attempt. This is the natural way to probe for IDOR (Insecure Direct Object Reference), where changing an identifier exposes another user's data.
- **GET vs POST.** A `GET` carries data in the URL (visible in history and address bar, length-limited); a `POST` carries it in the request body (used for forms and sensitive submissions). A proxy exposes both equally.

> *Note on evidence:* the Burp captures from this work are deliberately **not** published here: they contained a live session's own cookies and a real institutional portal. In a security write-up, redacting your own session tokens is part of the job. A clean re-capture against a deliberately vulnerable app (DVWA/Mutillidae) with dummy credentials would be the right way to add a screenshot later.

## Web Application Exploitation

All exploitation below was performed against the Acunetix public test sites, Google's XSS game, or a local demo. All are environments built to be attacked.

**SQL injection with sqlmap.** `sqlmap` confirmed an injectable parameter and fingerprinted the back-end database:

![sqlmap confirming an injectable parameter]({{ '/writeups/web-app-assessment/images/07-sqlmap-injectable.png' | relative_url }})

Pointed at a second target, it enumerated the full list of databases on a Microsoft SQL Server back end:

![sqlmap enumerating databases]({{ '/writeups/web-app-assessment/images/08-sqlmap-databases.png' | relative_url }})

and dumped a target table, retrieving its rows:

![sqlmap dumping a users table]({{ '/writeups/web-app-assessment/images/09-sqlmap-dump-users.png' | relative_url }})

Some fully-automated dumps hit repeated HTTP 500 responses on one target and did not complete, a realistic reminder that automation is not guaranteed, and that reading the tool's output matters as much as launching it.

**Reflected XSS.** Using Google's XSS training game, injecting a script payload into a reflected input field executed JavaScript in the page, confirmed by the alert dialog:

![XSS alert executing in the browser]({{ '/writeups/web-app-assessment/images/10-xss-alert.png' | relative_url }})

This is the core of cross-site scripting: input that is reflected into the page without encoding becomes executable code in the victim's browser.

**CSRF.** A cross-site request forgery was demonstrated safely on a local server: a forged form on `127.0.0.1` submitted a state-changing request to a target endpoint without the user's intent.

![local CSRF demonstration form]({{ '/writeups/web-app-assessment/images/11-csrf-demo-form.png' | relative_url }})

The demonstration shows why anti-CSRF tokens exist: without a secret, per-request token the server cannot distinguish a genuine user action from a forged one.

## WPA2 Handshake Analysis

Using a public sample capture, the WPA2 four-way handshake was isolated and analysed. Aircrack-ng read the capture, and Wireshark displayed the four EAPOL messages that make up the handshake:

![Wireshark showing the EAPOL four-way handshake]({{ '/writeups/web-app-assessment/images/12-wireshark-eapol.png' | relative_url }})

The four EAPOL frames are the material an attacker needs to attempt an offline dictionary attack against the pre-shared key, which is why a long, high-entropy WPA2 passphrase (or WPA3) is the real control, since the capture itself cannot be prevented.

## Detection Evidence

The defender's view of each stage:

- **OSINT** is effectively undetectable: it touches public sources and archives, not the target. The defence is at the source: don't publish what shouldn't be public.
- **Host discovery and scanning** generate connection attempts. A full connect scan completes handshakes and is logged; a SYN stealth scan does not, so detection has to come from an IDS watching for the scan *pattern* (many ports, one source, short window) rather than from connection logs alone.
- **SQL injection** leaves a clear trail in web-server and application logs: malformed parameters, SQL syntax in query strings, and bursts of near-identical requests from `sqlmap`. A WAF and parameterised queries both break it.
- **XSS and CSRF** show up as unusual payloads in request parameters and as state-changing requests lacking a valid token or with a mismatched `Referer`/`Origin`.
- **The WPA2 capture** is passive and silent on the wire; there is no host log of it, which is exactly why passphrase strength carries the defence.

## Recommendations

| Weakness | Control |
|----------|---------|
| Sensitive info exposed via OSINT | Strip production HTML comments; remove sensitive content rather than relying on `robots.txt`; enforce real server-side authorisation. |
| Outdated service versions (Apache 2.4.7) | Patch to current releases; suppress verbose version banners. |
| Slowloris DoS | Use connection-rate limiting and timeouts (e.g. `mod_reqtimeout`), or a reverse proxy / WAF that buffers slow requests. |
| Credentials in clear text | Enforce HTTPS everywhere (HSTS); never transmit credentials over HTTP. |
| SQL injection | Use parameterised queries / prepared statements; validate input; deploy a WAF; apply least-privilege database accounts. |
| Reflected / stored XSS | Context-aware output encoding; a Content Security Policy; input validation. |
| CSRF | Per-request anti-CSRF tokens; `SameSite` cookies; check `Origin`/`Referer`. |
| Weak WPA2 passphrase | Long, high-entropy passphrases or WPA3; the handshake capture cannot be prevented, only made useless. |

## Lessons Learned

- **What worked:** running each technique against purpose-built, authorised targets (scanme.nmap.org, the Acunetix sites, Google's XSS game) meant the full toolchain could be exercised safely and realistically.
- **What was slow or clumsy:** fully-automated `sqlmap` runs hit HTTP 500s on one target and stalled; next time I would tune the request rate and fall back to manual, targeted injection rather than relying on full automation.
- **On judgment:** the most valuable lesson was knowing what *not* to publish: my own Burp captures held live session cookies and a real portal, so they stay out. Handling evidence responsibly is part of the skill, not an afterthought.
- **The detection gap I noticed:** the quietest techniques (OSINT, a stealth scan, a passive handshake capture) are the ones a defender cannot catch after the fact, which is why prevention at the source matters more than detection for those stages.
- **One thing I now understand that I did not before:** how much an attacker can assemble from entirely public information before sending a single packet. Reconnaissance, not exploitation, is where an assessment is really won or lost.
