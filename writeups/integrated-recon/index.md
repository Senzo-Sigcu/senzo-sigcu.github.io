---
layout: page
title: "Integrated Reconnaissance — OSINT, DNS Enumeration & Traffic Analysis"
platform: Metasploitable 2
skills: [Enumeration, Log Analysis/Detection, Reporting]
date: 2026-11-05
writeup: true
---

> Scaffold for the Integrated Recon lab. Write every section in your own words as your own engagement — the lab brief is your source, not your script. Replace each `[FILL IN: ...]` with your own commands, output and screenshots. Don't paste findings you didn't observe; where the lab is *built* to contain something (an open zone transfer, cleartext logins), confirm it with your own evidence before you write it up as fact.

## Overview

[FILL IN: One paragraph — a structured reconnaissance assessment of a legacy Linux server (Metasploitable 2) in an isolated host-only lab you control. Note the goal: correlate findings across four tools spanning passive OSINT, active DNS enumeration, and live traffic analysis, then report what the evidence actually shows. Say plainly this is your own lab VM, so there's no authorisation question.]

- **Target:** [FILL IN: Metasploitable 2 — your lab IP]
- **Attack host:** [FILL IN: Kali — your lab IP]
- **Tools:** SpiderFoot (OSINT), dnsenum (DNS enumeration), tcpdump (capture), Wireshark (analysis)
- **Scope / goal:** [FILL IN: enumerate the host and its DNS zone, capture live traffic, and identify sensitive data exposed in transit]

## Executive Summary

[FILL IN: 3–5 sentences a non-technical reader could act on. The headline here is usually two things: the DNS server hands over its entire internal zone to anyone who asks, and core services authenticate in cleartext — so an attacker on the segment learns the whole network map and can read credentials off the wire. State the overall risk and the single most important fix before any tooling.]

## Enumeration

Work tool by tool, and for each one record the command, the result, and a screenshot. Correlation is the point of this lab — no single tool maps the target alone.

**SpiderFoot (automated OSINT).** [FILL IN: the scan you configured against the target, and what it returned — open ports, banners, hostnames. If it returned little or nothing, that is still a finding: explain it by how your Kali box resolves DNS for a host-only lab.]

**dnsenum (DNS enumeration + zone transfer).** [FILL IN: your enumeration of the internal zone and the zone-transfer (AXFR) attempt. Record the records returned and the hostnames the transfer exposed. The lab target is deliberately configured to allow unrestricted transfers — confirm it from your own output rather than asserting it.]

**Correlation table.** [FILL IN: a small table tying the two together — which detail came from SpiderFoot, which from dnsenum. Show that the full picture needs both.]

**Screenshots to capture:** SpiderFoot scan config + results view; dnsenum output showing the record types; the AXFR transfer listing the hostnames.

## Vulnerability Identification

[FILL IN: Turn the above into ranked findings. For each, name the weakness, tie it to your evidence, and rate it on likelihood × impact with a one-line justification — see the [Methodology]({{ '/methodology/' | relative_url }}) page. The two the lab is built around are an unrestricted DNS zone transfer and cleartext authentication protocols; add only what you actually confirmed.]

| ID | Finding | Likelihood | Impact | Risk rating | Justification (likelihood × impact) |
|------|------------------------------------|------------|--------|-------------|--------------------------------------|
| F-01 | [FILL IN: e.g. unrestricted DNS zone transfer] | [L/M/H] | [L/M/H] | [Low/Med/High/Critical] | [FILL IN: e.g. "One unauthenticated query returns every host in the zone (High likelihood); full internal network map disclosed (High impact)."] |
| F-02 | [FILL IN: e.g. cleartext credentials over FTP/Telnet] | [L/M/H] | [L/M/H] | [Low/Med/High/Critical] | [FILL IN: tie the rating to the Follow-TCP-Stream evidence below] |

## Exploitation

[FILL IN: This lab is reconnaissance and traffic analysis, not remote-code exploitation — so "exploitation" here is capturing the data the weak configuration leaks. Describe the capture you ran: tcpdump filtering on the relevant ports while you drove FTP and Telnet logins, saved to a .pcap. Record the exact tcpdump command and interface. Note that the default lab credentials stay inside the lab.]

**Screenshots to capture:** the tcpdump capture command and running capture; confirmation that packets were written to the .pcap.

## Impact / Post-Exploitation

[FILL IN: What this access actually means. A full zone transfer gives an attacker the internal naming and addressing of every host — a target list for the next stage. Cleartext credentials captured on the wire give working logins with no cracking required. Frame it as consequence: "an attacker on this segment needs no exploit to map the network and authenticate."]

## Detection Evidence

This is the section that sets your portfolio apart — the defender's view of your own attack. Open the captures in Wireshark and show what a blue team would see.

[FILL IN:]
- **The zone transfer in the capture:** filter for AXFR (`dns.qry.type == 252`) and show the transfer packets over TCP 53. A zone transfer from an unexpected host is itself an indicator of compromise — say what a defender should alert on.
- **Cleartext credentials:** use Follow TCP Stream on the FTP and Telnet sessions to show the username and password in plain text, no decryption. Record the display filter you used.
- **Conversation timeline:** use Statistics → Conversations to build a timeline of every connection, and tie each one back to the tool that made it.

**Screenshots to capture:** the AXFR filter result; the Follow-TCP-Stream windows showing credentials in cleartext; the Conversations view.

## Recommendations

[FILL IN: One concrete fix per finding, highest risk first. For the zone transfer: restrict AXFR to authorised secondary servers only (`allow-transfer`), and alert on transfers from any other source. For cleartext protocols: disable FTP and Telnet entirely and replace them with SFTP and SSH, and monitor for any cleartext-auth traffic on the segment. Say what to change and why, not just "secure it."]

## Lessons Learned

Answer these properly — this is where your reasoning shows:

- **What worked:** [FILL IN: e.g. running tcpdump *before* the active tools so nothing was missed, or keeping a per-tool notes file.]
- **What was slow or clumsy:** [FILL IN: e.g. interface naming, a filter that caught zero packets, correlating two tools by hand.]
- **What you'd do differently:** [FILL IN: one concrete change to your process next time.]
- **The detection gap you noticed:** [FILL IN: e.g. how trivially the zone transfer and the cleartext creds show up in a capture — and what that says about why these protocols are retired in real networks.]
- **One thing you now understand that you didn't before:** [FILL IN: the actual learning, in a sentence.]
