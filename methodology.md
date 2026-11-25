---
layout: page
title: Methodology
---

This page documents the methodology applied to every assessment on this site, from initial reconnaissance through to the final report. It records process and reasoning rather than a list of tools, so that a reader can evaluate how an engagement is approached and why each decision is made.

## Enumeration strategy

Enumeration is treated as the core of the engagement. Exploitation is generally a short step once reconnaissance is thorough, so the majority of effort is spent at this stage, and each service is assessed on its own terms rather than recorded simply as an open port.

Enumeration proceeds outside-in, in defined layers, without skipping stages:

1. **Host discovery.** Live hosts are confirmed before any port scanning, to avoid probing addresses that are not in use. On a local segment this is an ARP sweep; across a routed network, a light ping or TCP sweep.
2. **Port and service sweep.** A full Nmap port scan is run first, with completeness prioritised over speed, followed by version detection on every responding service and a Greenbone/GVM vulnerability scan to convert versions into actionable leads. The service version is the decisive detail, as it distinguishes a generic service from a specific build with known weaknesses.
3. **Service-by-service review.** Each open service is examined individually. A web application receives content discovery and input review in Burp Suite; a file service is checked for share and permission weaknesses; a database is tested for weak authentication. A broad scan is a starting point, not a conclusion.
4. **Human and configuration layer.** Default or blank credentials, exposed administrative interfaces, verbose banners, and other operator misconfigurations are reviewed. This layer frequently yields the highest-impact findings for the least effort.

A per-target record is maintained with a single line for each service: its identity, its version, and the reason it does or does not warrant further investigation. Exploitation does not begin until that line can be written for every open port. A service that cannot be justified as a target has not yet been fully enumerated.

## Triage and prioritisation

Findings are not treated as equally important, as an undifferentiated list of vulnerabilities provides no basis for deciding what to remediate first. Each finding is rated on two axes, and the combination sets its priority.

- **Likelihood:** how realistically the weakness can be exploited. A network-reachable, unauthenticated weakness supported by a public exploit is rated High. A weakness requiring local access, valid credentials, a specific configuration, or a chain of conditions is rated lower.
- **Impact:** what successful exploitation grants. Root or administrative access, or access to sensitive data, is rated High. A limited user shell, information disclosure, or denial of a single service is rated lower.

**Risk rating = likelihood × impact**, mapped onto the following matrix:

| | Impact: Low | Impact: Medium | Impact: High |
|-----------------|-------------|----------------|--------------|
| **Likelihood: High** | Medium | High | Critical |
| **Likelihood: Medium** | Low | Medium | High |
| **Likelihood: Low** | Low | Low | Medium |

The rating is accompanied by a justification, which carries equal weight. Each finding includes one line stating why it was placed where it was, for example: "unauthenticated and network-reachable with a public exploit (High likelihood), yields root access (High impact), therefore Critical." The justification is what allows an auditor or developer to assess, challenge, or act on the rating.

Findings are presented in rating order, highest first, so that the most significant issues are addressed before less important ones.

## Reporting standard

A finding that cannot be acted upon has limited value. Every write-up follows a consistent structure, moving from the finding, to its significance, to the required action:

- **Executive summary first.** A single paragraph suitable for a non-technical reader, stating the overall risk and the most important remediation, before any technical detail is presented.
- **Evidence with each finding.** The exact command and its output or a screenshot, the risk rating, the one-line justification, and a specific recommendation. A recommendation specifies an action, such as disabling a named service, rotating a credential, or adding a control, rather than a generic instruction to patch.
- **A defender's view in every write-up.** Each finding states what evidence the activity left behind and which log entry or alert would detect it. Documenting the defensive perspective is a core part of the methodology, not an optional addition.
- **Reproducibility.** Sufficient detail is provided for the path to be repeated. A result that cannot be reproduced is recorded as unconfirmed rather than overstated.
- **A recommendation for every finding, and a reason for every recommendation.** The report is intended to read as a continuous line of reasoning rather than a collection of output.
