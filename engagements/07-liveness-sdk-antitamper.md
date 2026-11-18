# 07 — Identity-verification SDK · four leads, one breakpoint, and a defence that was working

**Platform:** HackerOne · **Window:** 2026 · **Outcome:** closed, no report — two info-level
observations, one dead lead, one lead blocked by an environment the vendor deliberately rejects
· **Focus:** browser SDK internals, liveness/tamper controls, token handling

> Target name withheld. This is the write-up I would point to for "how I decide a target is
> actually defended rather than assume I have not tried hard enough."

---

## Why this target

A browser-side identity-verification SDK: the component that opens a camera, guides a
document or face capture, and returns a verdict the relying party trusts. It is an unusually
high-value component — the whole point is that its output cannot be produced by a script.

The engagement carried four pre-existing leads into the session. The question was not "is
there something here" but "which of these four survives contact with the actual code."

## Scope and constraints

- Browser SDK and its API surface only. No biometric media retained; captures were generated
  locally from a test video and discarded.
- 12-hour budget. ~10 used.
- Anti-tamper behaviour is expected. The vendor rejecting an automated client is the product
  working, and that had to be distinguished from a bypass at every step.

## The four leads and what happened to them

| ID | Lead | Verdict |
|---|---|---|
| F-13 | Capture verdict computed client-side in the container format | **Blocked / untested** — no capture ever completes, so there is no container to diff |
| F-03 | JWT algorithm confusion or `kid` abuse | **Dead** — `alg:none`, HS256 with empty key, HS256 with a public constant as key, ES384 with a bad signature, and a valid-shape token with a future `exp` were all rejected `401` |
| F-16 | Page freeze on a near-miss configuration token | **Confirmed behaviour, informational** — deliberate anti-tamper |
| F-17 | Capture aborts with a generic encryption error | **Confirmed root cause, informational** — the anti-tamper path firing, with a misleading message |

Four leads, zero submittable findings. The value of the session is in *why* each died, so
that is what follows.

## F-03: the token layer was not the weak point

Algorithm confusion is worth trying and cheap to disprove. Every variant returned the same
`401 Invalid token` from the predict endpoint — no differential, no partial acceptance, no
error-shape difference between "wrong algorithm" and "wrong key." A defence that returns an
identical response for every forgery attempt is doing its job.

Recorded as a dead lead with the full request set, so nobody re-runs it next quarter.

## F-17: the error message was not the error

The capture aborted with a specific numeric code that reads like an encryption failure. That
is the natural thing to chase. It is the wrong thing to chase.

Reading the mapping function first — before attaching any instrumentation — showed the code is
a **fallback label**. Any exception that is not already a typed vendor error is replaced with a
pre-built "encryption error" constant. The message describes the label, not the cause.

The actual cause was recovered by pausing on the exception in an **unmodified** bundle. The
arguments at the callback were:

```
(sessionId, LivenessError(<same code>), [], Error("Virtual camera"))
```

The engine detected the synthetic capture device, reported it as a generic liveness error, and
closed the session. The error text was a red herring produced by a `catch`-all mapping.

**Why this matters:** if the misleading message had been reported, it would have been a
correct-looking bug report about error handling with no security content. Reading the mapping
function cost ten minutes and prevented a bad submission.

## The gate: a canary, not a check

The SDK would not run at all in an automated browser. The cause was a value written into a
page-level variable that the SDK compares against a server-provided token:

- clean environment → one value
- this host → a different value
- anything else → the SDK evaluates a tight infinite loop and the page hangs

That is a canary on a server-issued token. Normalising the write let the flow proceed, but the
capture still did not complete — the canary is a gate, not the defence.

**The important control:** every "bypass" result was reproduced on a minimal page carrying only
the gate accessor, with no instrumentation hooks. A gate fix that only works in a harness is
not a fix; a behaviour that persists on a bare page is a property of the target. That
distinction is the difference between an environmental artefact and a finding.

## The tamper check fails on its own

Separately from any hook, the SDK's integrity check threw:

```
TypeError: Function.prototype.toString requires that 'this' be a Function
```

This persisted with the instrumentation disabled and on the minimal page. So the tamper check
is fragile in this environment independently of what the researcher is doing — which is a real
observation about robustness, but not exploitable, and not security-relevant on its own.

## Outcome

**Closed, no report.** Two informational observations (a misleading error label, a freeze on a
near-miss configuration), one dead lead with a full forgery matrix, one lead blocked because
the vendor's virtual-camera rejection is exactly the defence that was supposed to be bypassed.

The honest summary: **the anti-tamper worked.** A high-value component that ships a deliberate
anti-automation layer, and this session did not get through it.

## What this engagement demonstrates

1. **A defence working is not a bug.** The most common way to turn this engagement into a bad
   report would have been to file the virtual-camera rejection as "liveness bypassable" or the
   freeze as "DoS." Both would have been wrong, and both would have been closed.
2. **Read the code before believing the error.** A `catch`-all that maps every unexpected
   exception onto one code will send you in the wrong direction for hours. The mapping
   function is always more informative than the message.
3. **Control your own instrumentation.** Any result produced while hooks are attached has to be
   reproduced without them. A minimal page is the cheapest control in browser testing and the
   one most often skipped.
4. **Stopping is a deliverable.** Ten hours with no report beats forty hours with a fabricated
   one. The engagement folder records exactly what would need to change for the lead to reopen
   — a real camera on a clean browser, or a sandbox that the vendor does not reject.

## Tools used

Headless browser with CDP pause-on-exception and breakpoint capture, a local TLS origin for the
SDK host, unmodified and minimally-modified builds side by side, and a request set for the
token-forgery matrix.
