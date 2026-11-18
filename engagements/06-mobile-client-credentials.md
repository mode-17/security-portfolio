# 06 — Philippine payments platform · a recovered production client secret that was not a vulnerability

**Platform:** YesWeHack · **Window:** 2026 · **Outcome:** closed, no report — the recovered
credential had no reachable impact · **Focus:** API misconfiguration, Android client
credentials, OIDC

> Target name withheld. This write-up is included because it is the cleanest example I have of
> a real technical result that correctly failed the impact gate — and because "recovered a
> secret" is the most over-reported non-finding in mobile testing.

---

## Why this target

A large payments platform with an API-misconfiguration-only program: six API hosts, three app
stores, and an explicit exclusion list that rules out most of the usual low-grade material
(info disclosure without impact, missing headers, version leaks, user enumeration,
rate-limit findings, mobile client hardening).

That exclusion list is the interesting part. It meant the only class worth pursuing was
**misconfiguration with reachable impact**, and the only assets that could produce it were the
APIs behind the mobile client.

## Scope and constraints

- Read-only whenever possible. No accounts, no orders, no PII touched.
- One class only: API misconfiguration.
- Any recovered credential validated **against the token endpoint only** — a hard rule fixed
  before the extraction started, not after.

## Phase 1 — the API surface

Three gateway stacks were distinguishable from headers and error shapes alone: a CDN into an
API gateway into an Istio/Envoy mesh on the main host, a legacy Kong gateway on a second, and
an API-gateway-plus-custom-authorizer front on a third. A fourth host was a standards-based
OIDC server. Two more were Spring Boot services.

Public OpenAPI documents were served without authentication on two of the hosts, plus OIDC
discovery and JWKS. That gave a route inventory for free and let the matrix be written from
the spec instead of from crawling.

The negatives from this phase are worth keeping, because they are the rows every engagement
re-tests:

| Attempt | Result |
|---|---|
| Gateway token bypass: case variation, duplicate slash, dot segment, encoded slash, traversal | rejected |
| Verb tampering on authenticated routes | rejected |
| `X-Forwarded-For` / `X-Original-URL` / `X-Internal` trust | no effect |
| Actuator on two Spring services | `health` + `info` only |
| Kong admin API / any production Kong key | not exposed; all published keys were sandbox |
| SigV4 bypass on the billing route | no |
| OIDC authorizer bypass on the token-op host | no; unsigned and algorithm-confusion JWTs both rejected |
| Dynamic client registration | `403` |

One host resolved to a load balancer in a region this egress could not reach on 80/443. That
is a reachability constraint, not a finding, and it was recorded as such rather than probed
repeatedly.

**Cost of phase 1:** about a day, and it produced an accurate map of what was hardened.

## Phase 2 — the client credential

The only live lead at the end of day one was a client-credentials request contract recovered
from the in-scope Android build: how the app obtains an app-level token, and where it keeps
the `client_id` and `client_secret` it sends.

The extraction is the part worth writing down, because it is a pattern rather than a target
detail:

1. A JNI library exposed getters that each returned a 172-character base64 blob from
   `.rodata`.
2. The app "decrypted" each blob with `Cipher("RSA/ECB/PKCS1Padding")` — using an RSA
   **public** key.
3. A public-key operation is not encryption. It is `c^e mod n`: anyone with the public key can
   perform it. What looked like a decryption routine was `pow(c, 65537, n)` followed by
   PKCS#1 v1.5 padding strip.
4. The modulus and exponent were themselves returned by another getter, in the clear.

So the "encrypted" material was obfuscation with the key to the obfuscation shipped alongside
it. Offline, with no device and no app running, the blobs resolved to: the client id, an
88-character client secret, an HMAC key for a request-signing header, and four certificate
pins.

The secret, the HMAC key and the pins were **not** written into the engagement repo. Only the
method and the client id, which is public by design.

### Validation

Per the pre-fixed rule, the credential went to the token endpoint and nowhere else:

```
POST /auth/v3/{vendor}/accesstoken/noexpiry
grant_type=client_credentials&client_id=<public>&client_secret=<recovered>

200 → {"token_type":"Bearer","expires_in":899,
       "scope":"open_platform_token.write internal"}
```

A real production JWT. Two claims were worth pausing on: the token was issued by the
production OIDC issuer, and its scope string contained `internal`.

### Then I tested what it could reach

This is the step that decides whether the engagement has a finding, and it is the step people
skip.

- Without the app's anti-abuse headers the endpoint returns `403/-250`. The headers include an
  HMAC over the request using the recovered key — so the credential alone is not enough; the
  signing scheme has to be reproduced too.
- With the token, the reachable surface was application configuration. Non-sensitive
  configuration. No customer data, no transaction path, no administrative action.

The `internal` scope string was misleading — it named the issuing surface, not an authorisation
boundary.

## Outcome

**Closed with no report.** The recovered material is real: a production client secret and a
signing key, extractable offline by anyone who downloads the APK. The program's rules do not
reward exposed credentials without impact, and there was no impact — the token opened a
configuration endpoint.

Filing it would have been a report that says "a secret is public" with no answer to "and
therefore what." That is the exact shape of report that gets closed as informational and
lowers the researcher's signal on the program.

## What this engagement demonstrates

1. **A primitive is not an impact.** Recovering a live production secret from a shipped app is
   a strong technical result and a weak security result until you can show what it opens.
   The two are separate gates, and passing the first does not imply the second.
2. **"Encrypted" in a client is obfuscation.** Any key material shipped with the client is
   recoverable. If the only thing between an attacker and a secret is a public-key operation,
   there is nothing between them. That is worth writing up as guidance regardless of whether
   it pays.
3. **Read the exclusion list before the code.** The program told me in advance that client
   hardening and impact-free secret exposure were out of scope. Knowing that on day one
   changed the order of work — the API surface came first, the APK only to explain a contract
   the API surface had already revealed.

## Tools used

Public OpenAPI and OIDC discovery documents, `jadx` for the DEX, `objdump`/`readelf` for the
native library, Python for the modular-exponentiation step, and a single `curl` to the token
endpoint with the signing header reproduced.
