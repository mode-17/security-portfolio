# Finding — <short name>

**Title:** <weakness> in <feature> allows <impact>
**Severity:** <requested> (CWE-<id>)
**Asset:** <in-scope asset>
**Endpoint / method:** `<method> <path>`
**Date:** <YYYY-MM-DD>
**Program:** <name> · **Researcher:** <handle / identifier>

---

## Summary

Two or three sentences: what the check should be, what it actually is, and what an attacker
gains.

## In-scope argument

Quote the clause that places this asset and this class in scope. Address any OOS clause a
triager could reasonably apply, before they apply it.

## Identities

| Identity | Should have access | Actually has access |
|---|---|---|
| | | |

## Steps to reproduce

Copy-pasteable, from a clean session.

1.
2.
3.

## Controlled checks

| Check | Result | What it proves |
|---|---|---|
| clean / no session | | the check exists at all |
| wrong identity | | per-object enforcement |
| bogus identifier | | no padding/decryption abuse |

## Evidence

- Request and response, with cookies and identifiers redacted
- Measurement detail (sample size, timing, counts) for any limit-related claim

## Impact

What data or action the attacker gains, and the chain to the worst realistic outcome. No
"could potentially".

## Remediation

Specific, implementable, and verifiable.

## Notes

- Why this is one finding and not a duplicate of the same root cause elsewhere.
- Anything intentionally not tested and why.
