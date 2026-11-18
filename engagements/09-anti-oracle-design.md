# 09 — CI merge-queue SaaS · an anti-oracle design that killed the hypothesis in three hours

**Platform:** HackerOne · **Window:** 2026 · **Outcome:** paused after a research-only phase — 3
passive GETs, no accounts, no authenticated requests · **Focus:** API authorisation model,
credential delegation, OIDC

> Target name withheld. Short engagement, kept because it is the cheapest example I have of a
> vendor documenting a defence well enough that the preflight hypothesis died before it was
> tested.

---

## Why this target

A hosted merge-queue service: GitHub App, ~90 REST endpoints, a dashboard, and three distinct
credential types in play — a deprecated personal access token, a user-scoped API token issued by
the vendor, and an OIDC exchange for CI jobs.

That is a lot of authorisation surface for a small product, and the OIDC path is the kind of
thing that is often bolted on after the token model and not re-derived against it.

## Scope and constraints

- Two scoped API hosts and the dashboard.
- Two tenants required for any cross-tenant test: an organisation and a user account.
- Read-only until a hypothesis survived the blueprint. It never did.

## The preflight and what it got right

**CONDITIONAL, 14/20.** Good-faith clause rather than a scanner ban, no blanket state-change
ban, one vulnerability per report, first-report-wins, and a standard out-of-scope list. The
conditions were written down with an abandon trigger before any traffic.

## The design that ended it

The vendor documents an **anti-oracle** authorisation model. A missing permission, a request
naming a different organisation, and a request naming a repository that does not exist all
return the same response:

```
403 APPLICATION_KEY_REFUSED
```

with permission and expiry checked **before** any object lookup.

Read that as a researcher and the consequence is immediate: enumeration-based BOLA is dead. If
"you cannot see it" and "it does not exist" and "not yours" are indistinguishable, then no
response differential exists to enumerate against, and any object-reference test collapses into
a single bit.

The preflight had ranked cross-tenant object access as the primary hypothesis. The blueprint
demoted it — not because the code was tested and held, but because the *documented design*
removes the oracle that testing it would depend on.

**This is the useful part of the case study.** A blueprint reads documentation as evidence about
the target's design intent, and design intent can be enough to kill a hypothesis before a single
authenticated request is sent. That is not the same as trusting the docs — it means the docs
tell you where the differential *cannot* be, so effort goes where a differential is still
possible.

## Where the surface actually was

The credential layer, which the documentation does not describe as hardened. Three hypotheses
survived the blueprint:

| ID | Surface | Question |
|---|---|---|
| H-1 | Deprecated GitHub PAT accepted as an API credential | No published scope model — does it inherit the token's full GitHub reach, and is that reach re-checked per resource? |
| H-2 | Vendor user token | Documented as "reaches exactly what its owner's dashboard session reaches" — a delegation claim. Is the effective permission re-derived per request, or fixed at issue time? |
| H-3 | OIDC exchange | A one-hour token bound to a single repository. Is the binding derived from the verified `repository` claim, or from a caller-supplied value? |

H-2 is the interesting one. "Reaches what the session reaches" is a statement about *equivalence*
at issue time; whether it stays equivalent when the session's permissions change underneath it
is a separate question, and it is the kind of revocation gap that does not show up in a
functional test.

H-3 is the classic CI-trust shape: an OIDC token's whole security value is that the binding is
derived from a claim the issuer verified, not from anything the caller can influence.

## Why it stopped

All three remaining hypotheses are **credentialed** — the credential is the subject of the test,
so there is nothing to test without one. Both tenants were started, but the GitHub App was never
authorised on either, which means no installation id, no application key, and no vendor user
token existed to send.

The engagement was paused there rather than worked around. Creating a credential by another
route, or testing a delegation claim without the delegated credential, would have produced
speculation rather than evidence.

## Outcome

**Paused, research-only, closed cleanly.** Total cost: ~20 minutes of preflight and ~1 hour of
documentation and blueprinting. Three passive GETs to the two in-scope hosts. Zero
authenticated requests, zero accounts beyond the two tenant shells, zero target impact.

## What this engagement demonstrates

1. **A documented defence is a legitimate reason to drop a hypothesis.** Not because
   documentation is trustworthy, but because a uniform-response design removes the signal a test
   would need. Spending a day confirming an oracle does not exist is a day not spent on the
   credential layer.
2. **Read the credential model as a system.** Three credential types with three different scope
   models on one product means the interesting question is the seam between them — particularly
   where a delegation claim is asserted in prose rather than enforced in code.
3. **Killing an engagement for an hour of work is a good outcome.** The alternative on this
   target was a day of tenant setup for hypotheses that could not be tested, or worse, a
   speculative report about a delegation claim nobody had exercised.

## What would reopen it

Both GitHub Apps installed on two genuinely separate tenants (an organisation on each side —
a user-versus-org pair can fail a cross-tenant test for an uninteresting reason), both
credential types minted, and one least-privilege PAT on the first tenant. Then start at H-1.

## Tools used

Program policy and vendor documentation, three `curl` requests, and a blueprint. No scanner, no
authenticated traffic.
