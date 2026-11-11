---
layout: page
title: Methodology
---

This page describes how I actually work a target, from first contact to final report. It's written for someone evaluating my process — a hiring manager, an audit lead, or future me — so it favours reasoning over tool lists. Edit it until it sounds like you, not like a template.

## Enumeration strategy

My first principle is that **enumeration is the engagement**. Exploitation is usually a short step once the groundwork is right, so I spend most of my effort here and treat every service as a question to be answered, not a port to be ticked off.

I work outside-in, in layers, and I don't skip ahead:

1. **Host discovery** — confirm what's actually live before touching ports, so I'm not scanning ghosts. On a local segment that's an ARP sweep; across a routed network, a light ping/TCP sweep.
2. **Port and service sweep** — a full port scan with Nmap first (I'd rather be slow and complete than fast and blind), then version detection on everything that answered, and a Greenbone/GVM vulnerability scan to turn versions into leads. The version string is the hinge: it's what turns "there's a web server" into "there's a specific build with a known weakness."
3. **Service-by-service deep dive** — I take each open service on its own terms. A web app gets content discovery and input review in Burp Suite; a file service gets a share and permission check; a database gets an auth check. One big scan is a starting point, never the finding.
4. **The human and configuration layer** — default or blank credentials, exposed admin interfaces, verbose banners, and anything a real operator left on by accident. This layer is where the easy, high-impact findings usually hide.

I keep a running notes file per target with one line per service — what it is, its version, and why it's worth (or not worth) a closer look. I don't move to exploitation until I can write that sentence for every open port. If I can't explain why I'm attacking something, I haven't enumerated it properly yet.

## Triage and prioritisation

Not every finding deserves equal attention, and a flat list of "vulnerabilities" helps no one decide what to do on Monday. I rate each finding on two axes and let the combination set the priority.

- **Likelihood** — how realistically can this be exploited? Network-reachable, unauthenticated, and backed by a public exploit is **High**. Needing local access, valid credentials, a specific configuration, or a chain of conditions pushes it **lower**.
- **Impact** — what does a successful exploit actually get the attacker? Root, domain admin, or access to sensitive data is **High**. A limited user shell, information disclosure, or denial of a single service is **lower**.

**Risk rating = likelihood × impact**, mapped onto a simple matrix:

| | Impact: Low | Impact: Medium | Impact: High |
|-----------------|-------------|----------------|--------------|
| **Likelihood: High** | Medium | High | Critical |
| **Likelihood: Medium** | Low | Medium | High |
| **Likelihood: Low** | Low | Low | Medium |

The rating is only half the job; the **justification** is the other half, and it's the part that shows judgement. For every finding I write one line explaining *why* I placed it where I did — "unauthenticated and network-reachable with a public exploit (High likelihood), yields root (High impact), therefore Critical." A rating without that sentence is just a colour; the sentence is what an auditor or a developer can argue with, trust, or act on.

I report findings in rating order, highest first, because the reader's attention is the scarcest resource in the whole engagement and it should land on what matters most.

## Reporting standard

A finding nobody can act on is wasted work. Every write-up on this site follows the same structure so a reader can move cleanly from **what** to **so what** to **now what**:

- **Executive summary first.** One paragraph a non-technical reader can act on — the overall risk and the single most important fix — before any tooling appears.
- **Each finding carries its own evidence.** The exact command and its output (or a screenshot), the risk rating, the one-line justification, and a concrete recommendation. "Patch it" is not a recommendation; "disable service X / rotate credential Y / add control Z" is.
- **A defender's view is mandatory, not optional.** Every write-up says what evidence the attack left and which log or alert would have caught it. Being able to describe the blue-team side is what separates a tester from someone who just runs tools.
- **Reproducibility.** Enough detail that someone else could repeat the path. If a result can't be reproduced, I treat it as unconfirmed and say so rather than overstating it.
- **No finding without a recommendation, and no recommendation without a reason.** The whole document should read as a chain of reasoning, not a dump of output.
