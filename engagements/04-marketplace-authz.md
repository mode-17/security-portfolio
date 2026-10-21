# 04 — Second-hand marketplace · a 14-test BOLA/IDOR matrix, blocked by a country gate

**Platform:** Bugcrowd · **Window:** 2026 Q4 · **Outcome:** closed before submission —
eligibility gate at the target, not a target weakness · **Focus:** BOLA/IDOR, upload
handling, wallet and checkout logic

> Target name withheld. This write-up is included because the failure mode is instructive,
> not because it produced a finding.

---

## Why this target

A large marketplace with an explicit business-logic and financial-impact focus, and a
session/token architecture worth understanding. Marketplaces have the classic
multi-user-object shape: items, closets, favourites, wallets, orders, messages, each owned by
someone.

## Scope and constraints

- Three production web assets, web only.
- An anti-bot layer with a captcha on the login path; login was completed by hand.
- Throttling required; no rate-limit testing.
- Own accounts only.

## Recon

**Session lifecycle.** Login exchanged credentials for short-lived and long-lived HttpOnly
tokens on the primary host. The access token was a JWT carrying a subject and an
authentication-method claim. Expiry triggered a refresh, and a failed refresh caused the
client to call an expiry endpoint and drop the session.

The most useful negative came out of this: **an expired or failed session behaves exactly like
anonymous** — the same `403`/`user_login_required` responses as no session at all. Cookie
presence is not proof of authentication, and any access-control test that assumed otherwise
would have produced a false positive.

**Data model.** The public profile object exposed identity, location, verification and
moderation flags — no email, phone or address fields. The interesting boundaries were the
authenticated ones: verification state, security settings, wallet, and messaging.

**Route map.** Built from the bundles and a crawl of the unauthenticated surface, split into
components with the authenticated area marked as blocked pending a logged-in session.

## The test matrix

Fourteen rows, each with a component, class, exact test, expected-safe result, priority and
status. Examples of the shape:

| Class | Test | Expected-safe |
|---|---|---|
| BOLA | read another user's profile and closet with my session | public fields only |
| BOLA | read another user's verification/2FA/security state | denied |
| BOLA | attempt an email change on another user's account | denied, no state change |
| IDOR | favourite another user's item | no private data returned |
| Logic | wallet: derive ownership, tamper a balance identifier | balance tied to the token |
| Logic | checkout/offer: client-supplied price and quantity | server recomputes totals |
| Upload | image upload content-type, extension, polyglot | unsafe types rejected |
| Forced browsing | staff-only routes as a member | `401`/`403`/`404` |
| Logic | tracking numbers: sequential identifiers | no other user's parcel data |

Execution rules were fixed in advance: control before every tampered request, three
identities where a third role appears, unique markers on anything stored, no state-changing
test without a reversible plan.

## What happened

**The gate.** Selling required phone verification. The phone endpoint was evaluated
server-side by country, and the test number's country was rejected outright — while the same
endpoint accepted a supported-country number. That is a correct implementation of a country
restriction, and it is not a vulnerability. It also happened to close off the entire
financial path: wallet, checkout, offers, and the seller-side tests were all gated behind it.

**Account state.** One account was suspended during testing by the platform's own controls,
which left it usable only as a cross-account resource owner for authorization tests, and the
other as a verified-but-phone-blocked member.

**What still ran.** The unauthenticated and non-phone-gated rows: public profile reads,
upload handling (a plain image was accepted and re-encoded server-side — the behaviour you
want), the messaging and stats surface, and forced browsing of staff-only route strings found
in the bundle. Nothing reportable.

## Outcome

**Closed, no reports submitted.** The honest conclusion is that the target's phone-verification
gate is server-side and correct, and that correctness is what stopped the engagement.

## What this engagement demonstrates

Two things I would want a hiring reader to take from this:

1. **Eligibility gates are part of recon.** Test-account capability is a constraint that
   determines the achievable test surface. Discovering it on day one and declaring the
   ceiling beats discovering it on day four after building a matrix you cannot run.
2. **I document blocked, not just found.** The test matrix here is mostly `BLOCKED-PHONE`
   and `PENDING` with reasons. That record is why the engagement closed cleanly instead of
   drifting into repeated probing of the same gate.

## Tools used

Burp Suite Professional, a headed browser session for the captcha-protected login, a
per-identity `curl` wrapper with a scope guard, and Playwright profiles for session capture.
