# Security Portfolio — Modestu Osinachi

Offensive security work: authorized bug bounty engagements, the tooling I build to run
them, and the methodology behind both.

This is not a list of wins. It is a record of how I scope, test, validate, and report —
including the engagements that produced no finding and the ones I closed on purpose.

---

## Read this first

- **Everything here is redacted.** Client and program names are withheld under program
  rules and safe-harbour terms. No live credentials, session material, other users' data,
  or unpatched exploitation detail is published here.
- **Results are stated honestly.** A duplicate is called a duplicate. A closed engagement
  is called closed. Negative results are included because they are part of the work.
- **Nothing here is a turnkey exploit.** The goal is to show judgement and method, not to
  hand over a working attack against a live system.

---

## Engagements

| # | Sector | Platform | Window | Focus | Outcome | Write-up |
|---|---|---|---|---|---|---|
| 01 | Global apparel retail | HackerOne | 2026 Q4 | GraphQL BFF, step-up auth, OTP attempt controls | 1 report — closed duplicate | [01](engagements/01-retail-otp-step-up.md) |
| 02 | Online travel | HackerOne | 2026 Q4 | Access control, DOM XSS, postMessage | No report — negatives documented | [02](engagements/02-travel-access-control.md) |
| 03 | Digital-asset exchange | Bugcrowd | 2026 Q4 | Auth surface, IDOR/BOLA, mass assignment, SSRF | No report — surface unreachable | [03](engagements/03-digital-asset-auth-surface.md) |
| 04 | Second-hand marketplace | Bugcrowd | 2026 Q4 | BOLA/IDOR, upload, wallet logic | Closed — eligibility gate | [04](engagements/04-marketplace-authz.md) |
| 05 | Global payments network | HackerOne | 2026 Q4 | 169-asset scope analysis, surface ranking | Closed before traffic | [05](engagements/05-payments-scoping.md) |

Platforms: HackerOne (`kmode001`), Bugcrowd (`ModestuOsinachi`).

---

## Applications

Security applications I designed and built. Source is private; these are architecture and
design notes with the limitations stated.

| # | Application | Security idea | Stack |
|---|---|---|---|
| 01 | [A-WIDS](applications/01-awids-wireless-ids.md) | Wireless IDS with threat scoring, active deception and a hash-chain log | Python, Scapy, Flask |
| 02 | [SCKMS](applications/02-sckms-key-lifecycle.md) | Cryptographic key lifecycle with rules + ML anomaly detection | Python, Flask, scikit-learn |
| 03 | [SSMS](applications/03-ssms-results-integrity.md) | Tamper-evident academic records via HMAC-signed results | Node.js, PostgreSQL |
| 04 | [GhostPixel](applications/04-ghostpixel-steganography.md) | Encrypted, expiring steganography entirely in the browser | Vanilla JS, Web Crypto |
| 05 | [3SS](applications/05-3ss-security-services-platform.md) | Security services platform with on-chain milestone escrow | Flask, Redis, Solidity |
| 06 | [FUTO NIDS lab](applications/06-futo-ids-placement.md) | Why sensor placement decides what a WAF misses | Docker, Suricata, ModSecurity |
| 07 | [Laudate](applications/07-laudate-ticketing.md) | Payment reconciliation and ticket issuance state machine | Node.js, PostgreSQL, Paystack |

Index and rationale: [`applications/`](applications/).

---

## Tooling

| Project | Why it exists |
|---|---|
| [engagement-harness](https://github.com/mode-17/engagement-harness) | Multi-identity requests with a hard scope guard and per-program headers |
| [signed-api-client](https://github.com/mode-17/signed-api-client) | Signed API client for IAM privilege-testing surfaces the vendor CLI hides |

Design notes and the decisions behind each one: [`projects/`](projects/).

---

## Method

[`METHODOLOGY.md`](METHODOLOGY.md) is the short version of how I run an engagement:

1. Eligibility and scope before any traffic
2. Blueprint the application into components, actors, and data flows
3. Write a hypothesis-driven test matrix
4. Execute with a control-first identity matrix
5. Validate against a fixed gate before writing anything
6. Report for the fix, then record the outcome

The templates I use are in [`templates/`](templates/):
[scope](templates/scope.md) · [blueprint](templates/blueprint.md) ·
[test plan](templates/test-plan.md) · [finding](templates/finding.md).

---

## Principles

**Scope is law.** A scope guard is not a suggestion in a README; it is a check in the tool
that refuses the request. When an in-scope page pulls its API from an out-of-scope host, the
API does not become in scope.

**Control before conclusion.** Two responses differing is a hypothesis. A finding needs a
clean control, a reproducible request, and a measured claim.

**Negatives are deliverables.** The most useful note in an engagement folder is often the
one that says "this was tested and it is safe, here is the evidence" — it stops the next
session from burning hours on the same idea.

**Stop at proof.** Demonstrate access with the minimum evidence. Do not enumerate, do not
exfiltrate, do not leave test content behind.

---

## Authorisation

All work described here was performed under published bug bounty programs or written
authorization, using accounts I created and control. Testing was limited to in-scope assets.
Where a program required an identifier header, it was sent on every request. Where a program
required test content to be deleted within a fixed window, the test was skipped if deletion
could not be guaranteed.

If you believe something in this repository crosses a disclosure boundary, open an issue or
contact me through GitHub and I will remove it.
