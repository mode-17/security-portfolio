# 03 — Digital-asset exchange · mapping an auth surface behind two bot-defence layers

**Platform:** Bugcrowd · **Window:** 2026 Q4 · **Outcome:** no report — focus surface
unreachable on test accounts · **Focus:** auth mapping, IDOR/BOLA, mass assignment, SSRF

> Target name withheld. Third-party vendor names are mentioned only where the vendor is
> already visible to any visitor of the product.

---

## Why this target

A crypto exchange engagement with a financial focus area — deposits, withdrawals, KYC,
referral and reward logic. Financial flows are the highest-severity class in most crypto
programs, and they are almost always gated behind identity verification, which is exactly
where test-account planning decides whether an engagement is viable.

## Scope and constraints

- A managed program with a target list; the trading product lives on a subdomain of a
  separate marketing domain. I documented the interpretation and flagged it for the report
  rather than assuming.
- Two test accounts, both created on the trading product.
- The program requires created content to be deleted within a fixed window.
- **Test accounts start with zero balance, zero reward points, and no verification.**

That last line is the whole story of this engagement.

## Recon

**Reachability.** The exchange's main site was unreachable from my egress (timing out at the
edge). The trading product was reachable but sat behind a **Vercel security checkpoint** that
returned `429` to `curl` and to Postman, and to headless automation.

**Getting in legitimately.** A headed real browser completed the checkpoint, and I captured
the challenge flow: `429` → a WASM challenge → a challenge-response POST to a well-known
endpoint → `200`. No bypass, no trick — the platform's own challenge, solved by a real
browser. Automation after that reused the captured session state.

**Frontend.** Next.js with Turbopack, and an auth stack built on a third-party passwordless
identity provider proxied through the product's own subdomain.

**Auth model.** Email OTP (6 digits), no password login at all, plus Google and wallet-based
options. Device-intelligence fingerprinting ran on the client. Both accounts were created and
authenticated successfully.

**API map.** With a valid session, I walked the product routes read-only and recorded every
in-scope XHR/fetch with method, status, request body and a truncated response body. That
produced the authenticated API map and a Postman collection, and it is the artifact that made
every later test a single request rather than a browser session.

## What I tested

**Authorization.** User-object routes were self-scoped; there was no user-by-id route to
abuse. Favourites were derived from the session with no client-supplied identifier. The
Colosseum (rewards) leaderboard validated its parameters strictly.

**Mass assignment.** A settings `PATCH` accepted a payload padded with `email`, `role`,
`kycStatus`, `id`, and `privyId` alongside the legitimate field. The server returned `200`
and applied **only** the legitimate field. The extra keys were ignored — the correct
behaviour, recorded as a controlled negative with the before/after payloads.

**SSRF.** An avatar URL field was the natural candidate. The allowlist rejected a
credential-style host, an `@`-suffix host, a prefix-match host, a trailing dot, and an
alternate port and scheme — an exact host match. A chain-identifier parameter was also probed:
extra characters were parsed leniently in one case but the value was resolved numerically, so
no URL was ever constructed from it. Both negatives recorded.

**RPC surface.** Unauthenticated access to the product's RPC and data endpoints returned
`401`. An authenticated RPC call with an unsupported method returned a method-not-found
error, i.e. a method allowlist rather than a passthrough.

**Error-based injection.** Quote and parenthesis payloads against the leaderboard parameters
produced `400` or an edge `403`. No error-based SQLi signal.

## Why there is no report

The remaining focus-area surface — financial flows, KYC, guild authorization, referral abuse
— was **unreachable**. Creating a guild required 500 reward points; the accounts had none.
Onboarding required real deposits or trades, which the program does not permit. Both accounts
were also flagged by the platform's own anti-abuse controls (multi-account, tampering),
further restricting what they could do. The OTP paths needed mailbox access that the
automation did not have.

A skipped test is worth stating explicitly: the product let users publish a share link with
no delete path. The program required created content to be removed within five minutes, so
producing a stored-content test there risked leaving undeletable public content behind. I
skipped it and wrote down why. The program's rule outranks the test.

## Outcome

**Closed with no report filed.** Every control that I could reach behaved correctly, and the
surface that mattered was behind an eligibility gate I could not legitimately satisfy.

The valuable output is not a finding, it is the retained kit: a mapped authenticated API
surface, a 41-request Postman collection, request/response captures for two identities, and a
set of reusable tools for browser-session capture and read-only API mapping.

## What I would do differently

**Treat eligibility as a prerequisite with a deadline.** I should have identified the
points/balance requirement during scope review and either resolved it with the program or
declared the financial surface out of reach — before spending sessions mapping an API I could
not fully test.

**Ask the program about subdomain interpretation first.** The product I tested was not
literally in the targets table. I documented the reasoning and would have flagged it in any
report, but a one-line question to the program at the start is cheaper than a report that
gets closed on scope.

## Tools used

Playwright (headed, proxied, with per-identity persistent profiles), Burp Suite Professional,
Postman (API map and re-runnable collection), and custom read-only API-mapping scripts with
an in-scope host allowlist.
