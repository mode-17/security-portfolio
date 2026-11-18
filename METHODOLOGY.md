# Methodology

How I run an authorized engagement. This is the version I actually follow, not the
idealised one — including the parts that exist because something went wrong.

---

## 0. Eligibility before traffic

Before the first request I answer five questions in writing:

1. **What exactly is in scope?** Asset list copied from the program page, including path
   and wildcard qualifiers. Anything not listed is out of scope, including hosts that an
   in-scope page happens to load.
2. **What is out of scope by class?** Most programs exclude DoS, rate-limit issues on
   non-authentication endpoints, self-XSS, clickjacking without impact, missing headers,
   and known-vulnerable components without proof of exploitation. Rate-limit findings on
   *authentication* endpoints are often explicitly in scope — read the carve-out.
3. **How must traffic be identified?** A required header, identifier string, or user-agent
   token. This goes into the harness so it is attached automatically and cannot be forgotten.
4. **What are the test-account rules?** Which identity the program permits, whether
   plus-addressing is accepted, and whether content created during testing must be deleted
   within a fixed window.
5. **Can I reach the surface at all?** If the program needs a licence, a verified phone
   number, a funded balance, or a supported country, that is a prerequisite, not a detail.
   Resolve it before blueprinting, or declare the engagement blocked.

If any of these is unknown, the engagement stops here. Sending traffic blind is how
researchers get suspended from programs.

---

## 1. Blueprint

The unit of work is a **component**, not a URL. Each component gets a short record:

- Type: page, API, or both
- Routes and backing APIs observed in real traffic
- Roles that can reach it
- Data objects it touches
- Inputs the client controls, and what validation was observed
- Server-side decisions derived from client input, and whether the check is enforced
- Output sinks and their encoding
- Hypotheses, each with a class and a priority
- Unknowns and the test that resolves them

This is written down. A blueprint in someone's head is a blueprint that gets re-derived
every session.

### Where hypotheses come from

- A parameter that names an object (`id`, `uuid`, `memberId`) that the client also stores
- A role boundary the UI implies but the API may not enforce
- An action the UI gates client-side (a disabled button, a hidden route)
- A state machine step that can be re-entered out of order
- A vendor default that the product relies on

---

## 2. Test matrix

Each hypothesis becomes one row: ID, component, class, exact test, expected-safe result,
priority, status, evidence. Priority follows expected payout, not curiosity — object-ID
endpoints on user data first, then multi-user features, then role/permission endpoints,
then everything else.

Rules for the matrix:

- One hypothesis, one test. No shotgun scripts.
- Every test names its control.
- Every state-changing test names its reversal plan, or it does not run.
- A blocked test is recorded as blocked with the reason, not deleted.

---

## 3. Execution

The core loop is an **identity matrix**: take one request, drive it with every identity I
hold, and diff the responses.

```
control:  same identity, same request      → expected baseline
test:     other identity, same request     → what should be denied
clean:    no session / wrong token         → proves the check exists at all
```

Findings live in the delta between control, test, and clean. Three habits that prevent
false positives:

- **Diff the body, not the status code.** A `200` with a byte-identical body is not an
  access-control bug. A `403` that leaks a field is.
- **Marker discipline.** Any value I plant gets 8+ random characters so I can prove it
  reached a sink rather than colliding with existing content.
- **Reproduction sample size.** Timing and rate-limit claims need a real sample. One slow
  response is noise; n=80 with a stable distribution is a measurement.

---

## 4. Validation gate

Nothing is written up until it passes all of these:

1. **Reproducible** — from a clean session, with the exact steps.
2. **Controlled** — the clean/no-auth variant behaves differently, and I have recorded it.
3. **In scope** — the asset and the class are both permitted, and I can quote the clause.
4. **Impact is demonstrable** — I can state what an attacker gains. "Could potentially" is
   not an answer.
5. **Not a duplicate root cause** — the same missing check on three paths is one finding.
6. **Not a known class the program rejects** — checked against the program's named-known
   list and disclosed-report history.
7. **Worth the triager's time** — if I hesitate on this one, I kill it and move on.

A failed gate gets written into the notes as a retraction with the reason. Retractions are
cheaper than a bad signal.

---

## 5. Reporting

A finding is a bug report, not a story:

- Title states the weakness and the impact
- Asset, endpoint, method
- Identities involved and who *should* have access
- Copy-pasteable steps
- The controlled checks that prove the finding
- Impact in terms of data or action gained
- Remediation that a developer can implement

Evidence handling: cookies redacted, other users' data never included, screenshots cropped
to the minimum that proves the point. Raw evidence stays out of version control.

---

## 6. Record keeping

Per engagement: `scope.md`, `blueprint.md`, `test-plan.md`, `notes.md`, `findings/`,
`submissions.txt`. Notes hold leads, dead ends, hypotheses, and the reason each negative was
ruled out. The submission tracker holds report IDs so chained findings can cross-reference.

The value shows up on the second session and in the write-up afterwards — the negative list
is usually the most reused artifact.

---

## What I changed after getting it wrong

- **First-engagement lesson:** I sat on a validated finding while polishing it. It was
  closed as a duplicate of a report filed days earlier. Now: submit the moment the
  validation gate passes, then improve the write-up.
- **Novelty gate:** added a duplicate check against the program's disclosed history and
  named-known classes *before* testing, not after.
- **Eligibility gates are recon:** a test account that cannot reach the target surface is a
  blocking finding about the engagement, not a reason to keep probing.
- **Negatives need evidence:** "I tested it and it's fine" is worthless six weeks later
  without the request, the response, and the date.
- **A preflight is a prior, not a verdict.** I scored a contract program 16/20 from the
  outside, then scored it 13/20 after blueprinting reachability from the inside, and stopped
  with zero dollars spent. Blueprint gets veto power over preflight, permanently.
- **A primitive is not an impact.** Recovering a live production client secret from a shipped
  APK is a real result and a non-finding. "What does it open, and can I show it" is a separate
  gate that has to be passed on its own evidence.
- **Read the mapping function before the error message.** A `catch`-all that folds every
  unexpected exception onto one code will point you at the wrong subsystem for hours. The
  handler is always more informative than the string.
- **Control your own instrumentation.** Any behaviour observed with hooks, breakpoints or
  proxies attached has to reproduce without them. A minimal page is the cheapest control in
  browser testing and the one most often skipped.
- **A documented defence is a legitimate reason to stop.** If a uniform response removes the
  oracle a test depends on, spending a day confirming the oracle is absent is a day not spent
  where a differential is still possible.
- **Publish the redaction rules, not just the redactions.** Sector, platform, window and
  vulnerability class combine to identify a target even when the name is withheld. Narrow the
  window to the year and keep the mechanism generic.
