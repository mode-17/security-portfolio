# 01 — Global apparel retail · step-up auth under a GraphQL BFF

**Platform:** HackerOne · **Window:** 2026 Q4 · **Outcome:** 1 report filed, closed as
duplicate · **Class:** CWE-307 (improper restriction of excessive authentication attempts)

> Target name withheld under program rules. Everything below is either visible to an
> unauthenticated user or described at the level of mechanism rather than payload.

---

## Why this target

The storefront is a large retail commerce platform with a modern front end bolted onto a
legacy commerce back end. That combination — a React micro-frontend shell, a GraphQL
back-end-for-frontend, and an older commerce engine underneath — is where access-control and
step-up-auth assumptions tend to disagree with each other.

## Scope and constraints

- One production web asset, self-registered test accounts only.
- Edge bot defence requiring a real browser session; plain `curl` and Postman got the
  client challenge, so all authenticated requests came from a headed browser through a
  proxy.
- A program identifier was attached to every request as a research header.
- Same architecture served a sibling brand; the program treats that as one root cause.

## Recon

**Fingerprint.** Edge client challenge cookies on HTML and API responses → CDN with
bot defence in front. Origin behaviours (`storeId` / `catalogId` / `langId` parameters,
`/shop/` routing, address-error bodies naming an internal service) identified the commerce
engine. Bundles showed a Next.js shell hosting webpack micro-frontends with an Apollo client
using a batched HTTP link.

**API style.** GraphQL BFF over `POST` with a **JSON array body** and an `x-operation-name`
header — a batching shape that most generic scanners do not send correctly, which is useful:
it means server-side behaviour is best tested by hand from real captured traffic.

**Auth model.** Email + password session, an optional "keep me signed in" soft-auth state,
step-up OTP for sensitive account changes, and a separate MFA flow for associate accounts.

**Data model.** Users own addresses, orders, and preferences. The interesting edges were the
step-up-gated mutations: the UI treats them as protected, so the question is whether the
server does too.

**Operation inventory.** ~150 operations extracted from the bundles. Introspection was
disabled — expected, and irrelevant, since the operations are in the client anyway.

## What I found

The OTP protecting an account email change had no attempt controls:

- No per-request or per-account attempt limit, lockout, cooldown, or escalating delay.
- No format validation — the server compared whatever it was given, so every value is a
  valid guess rather than a rejected input.
- **200 concurrent incorrect submissions completed in 2.86 s** with no throttling: every
  request processed, HTTP 200, no challenge. A 50-request burst completed in 1.38 s.

Two design details made it worse than a plain missing rate limit:

- The request ID was valid for **~5 minutes** (measured by polling: last accepted at
  23:06:55Z, first rejected at 23:07:15Z), and failed attempts did not extend it.
- Issuing a new request ID did **not** invalidate older ones, and issuance itself was
  unthrottled (10/10 concurrent sends succeeded). So an attacker can hold several valid
  windows open at once instead of working against a single timer.

At 35–70 guesses/second from one client, a 6-digit code is a matter of hours, not years.
Because the same verified step-up is what authorises replacing the account email, a
brute-forced code converts a stolen session — or a known password with no mailbox access —
into a persistent account takeover, with the original owner locked out of the recovery
channel.

## What I checked that behaves correctly

The reason I could be confident about the finding is the list of things that did not work:

- Anonymous submission with a valid request ID → `UNAUTHENTICATED`.
- A second account using the first account's request ID → `OTP_EXPIRED` (binding is
  enforced, so the ID is not a cross-account selector).
- A bogus request ID → `400 INVALID_INPUT` ("could not be decrypted") — no padding oracle
  behaviour visible.
- Empty code → `400 INVALID_INPUT`, not a silent guess.
- Soft-auth ("keep me signed in") sessions could not even request the step-up challenge.

I also completed the flow end-to-end with a code read from the mailbox I control, and
verified that the follow-up profile update succeeds immediately after a validated code and
fails without one. The update value was deliberately left equal to the current address so no
account state actually changed — the point was to prove the code, not to weaponise it.

Elsewhere in the same application, the address book and email-preference surfaces were
hardened: ownership was enforced server-side, changes were session-scoped, and I recorded
both as controlled negatives.

## Outcome

Reported with the full test record: the burst logs, the TTL measurement, the concurrent
issuance test, the positive validation, and the controlled checks. **Closed as a duplicate**
of an earlier report on the same root cause.

That is a real outcome and I am not dressing it up. The technical work was validated end to
end; someone else filed the same missing control first.

## What I would do differently

**Submit earlier.** I had a complete, reproducible finding and then spent another session
polishing supporting documentation before filing. The duplicate was filed days before mine.
The validation gate is the trigger to submit — further polish does not protect against being
second.

**Check novelty before testing, not after.** A pre-test pass over the program's disclosed
history and named-known classes would have told me a missing brute-force control on an auth
endpoint was a likely crowded lane, and I could have weighted my time differently.

## Tools used

Burp Suite Professional (proxy capture and replay), headed browser sessions for the
challenge layer, a small Python summariser for burst logs, and the
[engagement-harness](https://github.com/mode-17/engagement-harness) pattern for per-identity
requests with a mandatory research header.
