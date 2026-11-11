---
layout: page
title: "Weaponisation — Payload Crafting with msfvenom"
platform: Kali / Metasploitable 2
skills: [Payload Selection, Vulnerability Identification, Log Analysis/Detection, Reporting]
date: 2026-11-09
writeup: true
---

> Scaffold for the weaponisation lab. This is the one write-up where **what you don't publish matters as much as what you do.** Document the *reasoning* — why a payload shape fits a target and a constraint — not a working, reusable binary. Every generated artefact below is deliberately **redacted**: I show the decision, the defender's view, and enough of the command to prove I understand it, never a copy-paste weapon. Replace each `[FILL IN: ...]` with your own run.

## Overview

Weaponisation is the kill-chain stage between *finding* a way in and *delivering* it: taking a confirmed vulnerability and pairing it with a payload that actually fits the target's architecture, the network path back, and the level of access you need. This lab is about making those choices deliberately and being able to defend them — and about seeing what the resulting artefact looks like to a defender.

- **Attack host:** [FILL IN: Kali 2026.x]
- **Target context:** [FILL IN: the host/architecture the payload is *for* — e.g. Metasploitable 2, Linux i686]
- **Goal:** craft a payload matched to a specific target + constraint, host a handler, and document the detection surface
- **Dates:** [FILL IN]

## Executive Summary

[FILL IN: 3–5 sentences. What payload shape did the scenario call for and why; what an attacker gains if it lands; and the single most useful defensive takeaway — typically that a correctly-matched payload is trivial to generate but leaves a recognisable signature that endpoint controls can catch. Lead with the risk, keep it jargon-light.]

## Payload Selection (the reasoning)

This is the core of the write-up. For the scenario, walk the decision tree out loud:

- **Platform & architecture:** [FILL IN: which `--platform` and `--arch` the target requires, and how you confirmed it from enumeration — a payload built for the wrong arch simply won't run.]
- **Staged vs stageless:** [FILL IN: staged (`meterpreter/reverse_tcp`) pulls the rest over the network and is smaller on disk; stageless (`meterpreter_reverse_tcp`) is self-contained and survives a flaky link. State which you chose and why.]
- **Reverse vs bind:** [FILL IN: a reverse shell calls back to you — needs the target able to reach your LHOST, and it's what you want behind NAT/firewall; a bind shell listens on the target — needs *you* able to reach *it*. Which does the network path allow?]
- **Format:** [FILL IN: ELF / raw / the format the delivery method requires, tied back to the Exploitation path it feeds.]
- **Encoding / evasion:** [FILL IN: note briefly whether any encoder was used and that modern AV is not fooled by classic encoders — this sets up the Detection section honestly rather than overclaiming evasion.]

## Generation (redacted artefact)

The command *shape* — enough to show it's understood, with the payload type, LHOST/LPORT and output deliberately held back:

```
msfvenom -p <platform>/<payload — e.g. meterpreter/reverse_tcp> \
         LHOST=<REDACTED> LPORT=<REDACTED> \
         -f <format> -o <REDACTED-filename>
```

> ⚠️ **Not published:** the completed command line, the payload file, its hash, and any screenshot showing the raw bytes or a working filename. The lab deliverable is the decision-making and the detection view, not a ready-to-run binary. If a marker needs the full artefact, it lives in the offline submission, not on this public site.

→ [FILL IN: record only non-weaponisable facts — e.g. the generated file size and that generation succeeded. A screenshot here, if any, should show the *terminal confirming success*, not the payload contents or a reusable filename.]

## Delivery / Handler (context only)

[FILL IN: describe how the payload would be delivered in the scenario and the matching handler (`exploit/multi/handler` with the same payload and LPORT), WITHOUT publishing a live listener config tied to a real address. The teaching point: the handler's payload setting must exactly match the one baked into the artefact, or the session never completes — a common failure mode worth stating.]

> 📸 **Screenshots to capture (safe ones only):** the handler *waiting* (no address visible); the decision notes. Do **not** screenshot a caught session that exposes a reusable config.

## Impact / What It Demonstrates

[FILL IN: if the payload lands, what does the attacker get, and at what privilege? Frame as consequence — "an interactive session as <user> on <host>, from which the post-exploitation steps in the Metasploitable write-up follow." Link the two labs so the kill chain reads as one story.]

## Detection Evidence

The defender's half — and the honest counterweight to the "evasion" temptation above.

- **On disk:** [FILL IN: what an EDR/AV signature matches on a stock msfvenom payload — the template stub, known byte patterns. State plainly that classic encoders do **not** defeat modern detection.]
- **On the wire:** [FILL IN: the staging traffic and the reverse connection to an unusual high port — what a NIDS or egress-filtering rule would flag. In a pcap, the callback to LPORT is the artefact.]
- **On the host at runtime:** [FILL IN: the spawned process, unusual parent-child relationships, a process making an outbound connection it never normally would — what process-monitoring/Sysmon-equivalent would surface.]
- **The gap:** [FILL IN: where detection is genuinely hard and what control closes it — e.g. default-deny egress filtering catches the callback even when the binary evades AV.]

> 📸 **Screenshots to capture:** an AV/EDR alert firing on the artefact (great evidence, fully safe to publish); the egress/IDS alert on the callback.

## Recommendations

| Risk | Fix |
|------|-----|
| Payload executes on endpoint | [FILL IN: application allow-listing; EDR with behavioural detection, not signatures alone] |
| Callback reaches attacker | [FILL IN: default-deny egress filtering; alert on outbound to high/unusual ports] |
| Delivery succeeds | [FILL IN: the delivery-channel control — attachment filtering, macro policy, user training — tied to how the scenario delivered it] |

## Lessons Learned

- **What worked:** [FILL IN: e.g. deciding reverse-vs-bind from the network diagram *before* generating anything meant no wasted rebuilds.]
- **What was slow or clumsy:** [FILL IN].
- **What you'd do differently:** [FILL IN].
- **The detection gap you noticed:** [FILL IN: the real lesson here is usually how little classic encoding helps and how much egress filtering does — say what you observed.]
- **One thing you now understand that you didn't before:** [FILL IN: e.g. why the handler payload must match the artefact exactly, or why "undetectable" is a marketing word, not an engineering one.]
