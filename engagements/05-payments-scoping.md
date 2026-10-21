# 05 — Global payments network · 169 assets, ranked, and closed before traffic

**Platform:** HackerOne · **Window:** 2026 Q4 · **Outcome:** closed at the researcher's
request before any request was sent · **Focus:** scope analysis and surface ranking

> Target name withheld. Included because knowing when *not* to test is part of the job.

---

## Why this target

A payments-network program with a large, explicitly enumerated scope — 169 assets, including
merchant platforms, sandbox payment gateways, admin portals, open-banking consoles, partner
APIs, and regional registration flows. A scope that large with per-asset severity caps is a
planning problem before it is a testing problem.

## What I did

### 1. Parsed the scope into a machine-readable list

Built the asset list from the program page export into a structured file, split by asset type
and severity cap, so that the scope guard in my tooling and the test plan referenced the same
source of truth. Both apex and `www` entries were treated as distinct assets where the program
listed them that way, rather than assumed duplicates.

### 2. Ranked the surface before touching it

The ranking was driven by *access path*, not by hostname count:

| Rank | Surface type | Why it ranks | Blocker |
|---|---|---|---|
| 1 | Merchant platform with explicit path scope | transaction/reporting objects with client-supplied merchant identifiers | test merchant account |
| 2 | Self-service payment sandbox | classic transaction and API-key scope bugs | documented sandbox sign-up |
| 3 | **Admin portal** in scope | BFLA / auth bypass against a privileged interface | none known — recon only |
| 4 | Hosted checkout sandbox | amount, currency and merchant tampering | sandbox credentials |
| 5 | Open-banking console | consent and account-access objects across users | self sign-up |
| 6 | Partner payments APIs | object ownership across demo accounts | self sign-up |

Then the long tail: regional registration variants mapped to the auth component they share,
wildcard DNS surfaces reserved for takeover and host-header recon, and the ~70 marketing
country domains last, because they are mostly static and CDN-served — their only value is
the explicitly scoped registration and gateway rows.

### 3. Wrote the hypotheses before testing

Three hypotheses were written down for the top-ranked surface, each with the class, the test,
and the control:

- **Cross-merchant object access** — reporting, invoicing and merchant-management paths that
  take merchant or transaction identifiers from the client. Test: create objects under
  identity A, request them as identity B, control with no session.
- **BFLA on configuration APIs** — key management, configuration and preference endpoints
  reachable by a plain merchant user. Test: low-privilege session against a privileged
  endpoint, compared with an admin response.
- **Onboarding IDOR** — registration and KYC templates. Test: does an identifier created by A
  resolve for B, and are template objects shared or per-tenant.

### 4. Confirmed the prerequisites

The program required a daily check-in from the testing IP for eligibility, and several of the
top surfaces needed sandbox merchant accounts created through vendor documentation. I
identified the documented sign-up path for the first surface and flagged the second as not
yet located. This is the part of engagement setup that decides whether week one is productive.

## Outcome

**Closed at my request before any traffic was sent.** My focus shifted to active engagements
with a better return, and I preferred to close it cleanly than run it half-attended.

What I retained is the deliverable that took the work: a ranked, severity-annotated surface
map, a prerequisite list with the exact sign-up paths, and three pre-written hypotheses that
make this a same-day restart rather than a from-scratch start.

## Why this is in the portfolio

Because the interesting part of a 169-asset scope is not the number of assets. It is
recognising that:

- A ranking driven by access path beats a ranking driven by host count.
- The admin portal sitting in scope matters more than the seventy marketing domains.
- Prerequisites — check-ins, sandbox accounts, daily eligibility windows — are engagement
  blockers that belong in the plan, not surprises in week two.
- Closing an engagement deliberately, with the artifacts preserved and the reason written
  down, is a professional decision and not a failure.

## Tools used

Scope export parsing into a structured asset file, a scope-guard wrapper that refuses any host
not in that file, and Burp Suite Professional for the planned proxy capture.
