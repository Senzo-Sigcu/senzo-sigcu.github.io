---
layout: page
title: "Phishing Analysis: Authenticated, Misaligned, and Still Hostile"
platform: Email security
skills: [Phishing Analysis, Email Security, Detection, Reporting]
date: 2026-11-14
writeup: true
---

A phishing email that reached a university mailbox, analysed from the defender's side. It is a useful specimen because it does something that trips up a lot of people: it **passes SPF and DKIM and is still phishing**. This write-up works the message the way a SOC analyst would, pulls the real headers to show exactly why the authentication result is misleading, and then maps why it got through to the control that should have stopped it.

This is an analysis of a received message. The prevention section is a set of recommended controls, not changes I implemented, as the mailbox sits on a managed email system I do not administer. Personal details, the live phishing token, and the sending platform are redacted; the impersonated brand is named because the impersonation is the point.

## The sample

```
From:     Disney Plus <noreply@disneyplus-accounts.com>
Reply-To: <redacted look-alike domain>
Subject:  Suspending your subscription
To:       <redacted university address>

Dear customer,
Your Disney Plus subscription is about to expire. There was a problem
auto-renewing due to an incorrect payment method...
[ Update my information ]
Please note that if you do not comply, you will be charged a processing
fee of 9.99 dollard.
```

## Verdict

Phishing, high confidence.

## Indicators, ranked

1. **Look-alike sender domain.** The real service sends from `disneyplus.com`. This came from `disneyplus-accounts.com`, a separate domain made to look trusted. The display name reads "Disney Plus," but the domain is the giveaway.
2. **The link does not go to the brand.** The "Update my information" button was wrapped in Microsoft Defender Safe Links. Decoding the wrapper, the true destination was a document-signing domain (`hxxps://secure[.]sign-doc[.]com/<token redacted>`) with no connection to the streaming service. Brand-versus-destination mismatch is one of the clearest phishing signals.
3. **Reply-To redirects elsewhere.** The `Reply-To` pointed to a different look-alike domain dressed up with `mandela` and `2fa` labels to appear official. Any reply would leave the brand entirely.
4. **Low-effort tells.** "9.99 **dollard**" (spelling), dollars where a South African account bills in Rand, and a generic "Dear customer" despite the header carrying the recipient's real name.

## Authentication analysis (the evidence)

This is the heart of it. Pulling the raw headers and reading the `Authentication-Results` line shows why "it passed authentication" means nothing here.

```
Authentication-Results: mx.microsoft.com;
   spf=pass   smtp.mailfrom=<sending-platform envelope domain>;
   dkim=pass  header.d=<a different signing domain>;
   dmarc=none action=none
From:      Disney Plus <noreply@disneyplus-accounts.com>
```

Read which domain each check actually authenticated:

- **SPF passed**, but for the **envelope** domain (the sending platform), not the From.
- **DKIM passed**, but for a **different signing** domain, again not the From.
- The **From** the user sees is `disneyplus-accounts.com`, a **third** domain that neither check authenticated.

Three different domains. The two green results are for domains the recipient never sees, and the domain they do see was never authenticated. This gap is called **alignment**, and closing it is the entire job of DMARC: it requires the domain that passed to match the visible From. Here neither aligns, so DMARC should fail the message.

But the result is `dmarc=none`. Confirming why, directly from DNS:

```
$ dig +short TXT disneyplus-accounts.com
"v=spf1 include:<sending-platform SPF> ~all"

$ dig +short TXT _dmarc.disneyplus-accounts.com
   (empty, no DMARC record)
```

The From domain has an SPF record (a soft `~all`), but **no DMARC record at all**. With no DMARC policy on the From domain, there is no rule to enforce alignment, so the misaligned message is delivered. Note also that the SPF record on the From domain was not even the one evaluated; SPF is checked against the envelope sender, which was a different domain. SPF keeps checking the envelope while the user's eyes are on the From.

**The lesson in one line:** SPF and DKIM passing is meaningless until you check whether they passed for the domain you actually see. Alignment is the thing, and DMARC is what enforces it. A reader who stops at "spf=pass, dkim=pass" and concludes "authenticated, safe" gets phished.

## How it should have been stopped

The message reached the Junk folder rather than being rejected, so the gateway processed it but did not block it. Mapping that to the layered controls:

| Layer | What it does | Why it did not stop this one |
|-------|--------------|------------------------------|
| SPF / DKIM | Prove a message came from some domain intact | They passed, for domains other than the visible From |
| DMARC | Enforce that the authenticated domain matches the From | No DMARC record on the From domain, so nothing to enforce |
| Secure email gateway | Filter on reputation, content, URLs | Caught enough to junk it and rewrite the link, not to reject |
| Impersonation protection | Flag look-alike domains and display-name spoofing | The control that should catch `disneyplus-accounts.com`; evidently not tuned for it |
| Reporting plus auto-remediation | Pull a reported email from all mailboxes | Would remove it once the first recipient reports it |
| MFA | Stop a stolen credential being used | The backstop if someone did submit their details |

A newly registered look-alike with no DMARC is genuinely hard to block before delivery, which is why defence in depth matters: no single layer is sufficient.

## Indicators of compromise

| Type | Value |
|------|-------|
| Sender (From) domain | `disneyplus-accounts.com` (look-alike, no DMARC) |
| Impersonated brand | Disney Plus |
| Phishing destination | `hxxps://secure[.]sign-doc[.]com` (defanged) |
| Authentication state | spf=pass, dkim=pass, dmarc=none, all misaligned with From |
| Lure | lapsed subscription / payment update, penalty threat |
| Delivery | reached Junk via Microsoft 365; link rewritten by Safe Links |

## Recommendations

| Finding | Control |
|---------|---------|
| Misaligned mail delivered | Treat `dmarc=none` with a non-aligned From as suspicious, not neutral; weight it toward quarantine. |
| Look-alike domain reached mailboxes | Tune impersonation protection to flag domains similar to protected brands; block the sender domain. |
| Credential-harvesting link | Keep URL rewriting and click-time scanning on; blocklist the destination domain. |
| Message would spread to peers | Ensure a one-click report button feeds the security team, with automatic removal from all mailboxes on confirmation. |
| Risk if a credential is entered | Enforce MFA everywhere so a stolen password alone is not enough. |
| Human susceptibility | Continue awareness training and simulations; the trained user is the last line against look-alike attacks. |

## Lessons learned

- **"Authenticated" is a trap unless you check alignment.** This message passed SPF and DKIM and is hostile, because both passed for domains other than the one in the From. The question is never "did it pass," it is "did it pass *for the domain I see*."
- **DMARC is the control that ties authentication to reality**, and its absence on the From domain is exactly what let the misalignment through. An analyst who reads `dmarc=none` as "fine" misses the point; here it is the reason the attack worked.
- **Prevention and analysis are two halves of email security.** Catching this one is a SOC task; stopping the next is an engineering task (tuning impersonation protection, weighting unaligned mail). Connecting the two, "it got through because this layer was not tuned, here is the fix," is what makes an analysis actionable.
