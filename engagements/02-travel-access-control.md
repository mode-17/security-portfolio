# 02 — Online travel · access control and DOM sinks on three in-scope URLs

**Platform:** HackerOne · **Window:** 2026 Q4 · **Outcome:** no report — every lead
controlled to a negative · **Focus:** IDOR/BOLA, DOM XSS, cross-origin messaging

> Target name withheld. Asset-level detail (paths, hostnames) is omitted because the
> program did not approve disclosure.

---

## Why this target

Three URL assets in scope, all product pages, all with a Critical ceiling and an explicit
IDOR → Critical severity note. A narrow scope with a high ceiling is ideal: the work is
depth on a small surface instead of breadth on a large one.

## Scope and constraints

- **Exactly three URL assets**, no wildcards. Anything else — including the API host the
  in-scope pages call — is out of scope.
- A research identifier had to appear on every request.
- Aggressive bot defence on the login path; login was completed by hand in a real browser.
- Self-registered accounts only, no other users' data.

## Setting up the identity matrix

Two accounts (A and B) were created with different roles at the point the program required,
each with its own browser profile and session. The harness enforced:

- A per-identity cookie jar and bearer token
- The mandatory research identifier on every request
- **A scope guard that refused any URL outside the three assets** — this fired more than
  once, which is the point of having it

Sessions were captured from a headed browser through a proxy, then replayed with `curl` so
that every test request was reproducible from a log rather than from browser clicks.

## The access-control work

Every data-bearing request was driven with four identities: A, B, anonymous, and a clean
request with no session at all. The clean request is what proves a check exists; A-vs-B is
what proves it is enforced per object rather than globally.

Two outcomes worth recording:

- **A token claim that looked like an IDOR selector was not one.** Both accounts' bearer
  tokens shared the same value in a wallet-related claim, which is exactly the shape of a
  cross-account object reference. Measured behaviour said otherwise: the servers derived
  ownership from the authenticated subject, and swapping the client-supplied identifier did
  not change whose data came back. Recorded as a controlled negative with the evidence,
  which closed the lead permanently.
- **The data-bearing APIs were on a different host than the pages that used them.** The
  program's in-scope list named the pages only. Reading the application's own bundle told me
  where the APIs lived, but "the page uses it" is not "the program scoped it". I did not
  test them, and I said so in the notes rather than quietly probing. That is the right call
  even when it costs the engagement its best surface.

## The client-side work

The client-side pass was static-first, then dynamic:

1. **Static.** Enumerated the loaded bundles and mapped URL-parameter readers, then narrowed
   to every HTML sink (`innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`,
   `eval`, `Function`, `setAttribute`, navigation and `window.open`, `postMessage`).
2. **Dynamic.** Instrumented those sinks *before* page script executed, then ran five passes:
   baseline, all query parameters with unique markers, hash fragment, authenticated session,
   and an attacker-controlled `Referer`. **Zero dangerous sink hits** in every pass. The only
   marker propagation observed was an analytics library writing the page URL into its own
   storage — not a sink.
3. **Reflection check.** Query parameters were echoed into a server-rendered JSON blob inside
   a `<script>` block. That is normally where JSON-in-script breaks out — but the encoder
   escaped the characters that would matter, so no breakout, and none of the redirect-style
   parameters produced a `Location` header. Not XSS, not an open redirect. Recorded as a
   negative.
4. **`Referer` handling.** A crafted `Referer` set a host value in that same server-rendered
   blob. It appeared in no client bundle the page loaded and had no client-side sink. Without
   a sink there is no finding, so it stayed a note.

### The cross-origin messaging lead

An analytics-devtools bridge accepted `postMessage` commands with **no origin check**, and
the in-scope booking page was frameable. On its own, that reads like a finding.

It was not. The responses were posted back to the page's own window rather than to the
opener, so a parent frame could not read them, and the reachable effect was toggling an
in-page debug flag. No exfiltration path, no impact. Recorded as a negative with the
reasoning, because "missing origin check" without a reachable effect is the kind of finding
that gets closed as informational and costs researcher signal.

Frameability was assessed separately: the cashback pages set `SAMEORIGIN`, and a real
browser refused the frame — the clickjacking angle was dead for the authenticated surface.
The frameable page was public and pre-authentication with no sensitive action reachable
from it. Informational at best, and the program excludes clickjacking without impact.

## Outcome

**No report.** The honest summary is that this target was well built: object ownership was
enforced server-side, the client-side sinks were not reachable from attacker input, the
JSON-in-script encoder did its job, and the messaging bridge had no exfiltration path.

## What this engagement demonstrates

- **Deliberate non-testing.** The highest-value API surface was out of scope. Not touching it
  is a decision worth showing, not a gap in the write-up.
- **Control-first discipline.** The shared wallet claim was a plausible IDOR and it was
  killed by measurement, not by intuition.
- **Negatives with evidence.** Every dead end above is stored with the request, the response,
  and the date, so a future session does not re-litigate it.

## Tools used

Burp Suite Professional, a custom instrumented-sink probe (static analysis of the bundles +
runtime sink hooks), a per-identity `curl` wrapper with a scope guard, and headed browser
sessions for the challenge-protected login.
