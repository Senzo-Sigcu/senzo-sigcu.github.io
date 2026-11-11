---
layout: page
title: "Social Engineering — Credential Harvesting, from the Defender's Side"
platform: Kali (lab, self-targeted)
skills: [Vulnerability Identification, Log Analysis/Detection, Reporting]
date: 2026-11-07
writeup: true
---

> Scaffold for the social-engineering lab, written **defensively on purpose.** The skill worth showing is understanding *how* a credential-harvesting attack works so you can detect and defeat it — not producing a phishing kit. Everything here is framed around recognition, indicators and controls. No cloned-page HTML, no live harvester config, no reusable template is published, and the only "victim" is my own lab browser. Replace each `[FILL IN: ...]` with your own run.

## Overview

The Social-Engineer Toolkit (SET) automates the mechanics of a credential-harvesting attack: clone a login page, host it, and capture whatever a victim types. Running it once in a closed lab — against myself — is the fastest way to understand exactly what the attack produces and, more importantly, what it leaves behind for a defender to catch. That defender's view is the entire point of this write-up.

- **Attack host:** [FILL IN: Kali 2026.x]
- **"Victim":** [FILL IN: my own second VM / browser on the same host-only network — no third party, ever]
- **Goal:** understand the attack mechanics well enough to document detection and user-facing and technical controls
- **Dates:** [FILL IN]

**Scope and ethics (not optional for this lab).** Social-engineering techniques are only lawful against targets you own or are explicitly authorised to test. This exercise was confined to my own isolated lab with myself as the sole subject. Run against anyone else, the same actions would be a criminal offence under South Africa's Cybercrimes Act (Act 19 of 2020) and ECT Act. That line — ownership, authorisation, contained scope — is what this write-up is really about.

## Executive Summary

[FILL IN: 3–5 sentences for a non-technical reader. How convincing a cloned login page is, how little skill it takes to stand one up, and the two or three controls that actually blunt it (MFA, user reporting, and the technical signals below). Lead with the risk to people, not the tooling.]

## How the Attack Works (mechanics, not a recipe)

Enough to prove understanding, deliberately *not* a step-by-step you could reuse against a real target:

- **Clone:** [FILL IN: describe in prose that SET copies a legitimate login page's appearance. Do **not** paste the cloned HTML, the target URL, or the exact menu path/option numbers as a recipe.]
- **Host:** [FILL IN: it serves that page from the attacker's address, typically over plain HTTP — a key tell, covered below.]
- **Lure:** [FILL IN: the link would reach a victim via email/SMS/chat in a real attack. Describe the social pressure used — urgency, authority — since that's what detection training targets.]
- **Harvest:** [FILL IN: submitted credentials are written to the attacker's console/log while the victim is bounced to the real site. Describe this; don't publish a capture showing real-looking credentials.]

> ⚠️ **Not published:** the cloned page source, the harvester output file, the exact SET option sequence, and any screenshot of captured credentials. A redacted screenshot of the *attacker console state* (no credentials, no target URL) is the most I'd put here.

## Why It's Convincing (the vulnerability)

[FILL IN: this lab's "vulnerability" is human and procedural, not a CVE. Name the weaknesses the attack exploits — visual trust, look-alike domains, urgency, password reuse, absence of MFA — and rate the organisational risk on likelihood × impact per the [Methodology]({{ '/methodology/' | relative_url }}). Likelihood is usually High (cheap, scalable); impact depends entirely on whether MFA and monitoring are in place.]

| Weakness exploited | Likelihood | Impact | Risk | Note |
|--------------------|------------|--------|------|------|
| User trusts visual appearance | [H] | [FILL IN] | [FILL IN] | [FILL IN] |
| Password reuse across sites | [FILL IN] | [H] | [FILL IN] | [FILL IN: one captured credential unlocks others] |
| No MFA on the real service | [FILL IN] | [H] | [FILL IN] | [FILL IN: MFA is the control that most reduces impact] |

## Detection Evidence

The heart of the write-up. What would catch this attack, from the user's screen to the network.

- **In the URL bar:** [FILL IN: the harvester is typically plain HTTP and on an IP or look-alike domain — no valid TLS cert for the real brand. This is the single clearest user-facing tell; a browser's "Not secure" warning is a detection signal.]
- **On the wire:** [FILL IN: an HTTP `POST` of form fields to an unexpected host. In a pcap, Follow-TCP-Stream on that POST shows the submitted values in cleartext — demonstrate this with *your own throwaway test input*, never real credentials.]
- **At the email gateway:** [FILL IN: look-alike sender domain, link mismatch between display text and href, newly-registered domain — what a mail filter or a trained user flags before the click.]
- **At the identity layer:** [FILL IN: a successful login from an unusual location/device shortly after the phish; impossible-travel; MFA prompts the user didn't initiate. These catch the attack *after* the credential is stolen but *before* it's abused — which is why MFA is the backstop.]

> 📸 **Screenshots to capture (safe):** the browser "Not secure" / cert warning on the harvester; a pcap POST stream showing **test** input you typed yourself; an email-header analysis showing the domain mismatch.

## Recommendations

| Risk | Control |
|------|---------|
| User submits credentials to a clone | [FILL IN: security-awareness training focused on URL/TLS checks; a one-click "report phishing" button] |
| Stolen credential is reused | [FILL IN: enforce MFA everywhere; block password reuse; unique passwords via a manager] |
| Lure reaches the inbox | [FILL IN: mail filtering, DMARC/DKIM/SPF, link rewriting/sandboxing, new-domain blocking] |
| Compromise goes unnoticed | [FILL IN: impossible-travel and anomalous-login alerting at the IdP] |

## Lessons Learned

- **What worked:** [FILL IN: e.g. running it against myself made the plain-HTTP tell obvious in a way reading about it never did.]
- **What was slow or clumsy:** [FILL IN].
- **What you'd do differently:** [FILL IN].
- **The detection gap you noticed:** [FILL IN: usually that *technical* controls can't fully close a *human* vulnerability — MFA and reporting culture carry the weight. Say what you observed.]
- **One thing you now understand that you didn't before:** [FILL IN: e.g. why MFA is framed as the control that limits blast radius rather than one that prevents the click.]
