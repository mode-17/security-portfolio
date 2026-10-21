# 02 — SCKMS: cryptographic key lifecycle with hybrid detection

**Type:** security application · **My role:** design and implementation · **Source:** private
**Stack:** Python, Flask, scikit-learn, SQLite, Server-Sent Events

---

## The problem

Key management fails quietly. A key outlives its cryptoperiod, a nonce gets reused, the same
key encrypts far more data than intended, and nothing in the application notices because the
application is not looking at key *usage*. SCKMS treats a key as an asset with a lifecycle and
watches how it is actually used.

## The loop

```
CONTROL → MONITOR → DETECT → RESPOND
```

- **Control** — generate high-entropy AES-256 keys, register them, enforce a cryptoperiod.
- **Monitor** — ingest usage metadata: algorithm, key size, plaintext and ciphertext length,
  entropy, execution time, nonce-reuse flag, key age, usage count.
- **Detect** — a rule engine for known-bad conditions, plus a behavioural model for patterns
  the rules do not encode.
- **Respond** — rotate, block, revoke, blacklist, or replace, with a remediation summary the
  operator can act on.

## Detection design

**Rules** handle the unambiguous cases, because a rule is explainable and testable:

| Condition | Signal |
|---|---|
| Nonce reuse | catastrophic for stream/GCM modes |
| Key age > cryptoperiod | lifecycle violation |
| Usage count above threshold | key overuse |
| Entropy below floor | weak or degraded ciphertext |
| Sessions per key above threshold | key sharing across contexts |

**The model** handles the rest: a classifier over the same nine features, trained to separate
normal usage from timing-anomaly and low-entropy patterns. It is a behavioural second opinion,
not an oracle.

That ordering is deliberate. Rules run first so the system never blocks on a model, every
ML-driven decision is accompanied by the rule status, and a model failure degrades to
rule-only rather than to no detection.

## The result I will not oversell

The training report shows 1.00 accuracy, precision, recall and F1 on a held-out set. That
number is a property of the **synthetic dataset**, not evidence of real-world detection
quality. The generator produces separable classes, so a clean split is unsurprising.

What the project actually demonstrates is the pipeline: feature schema, offline training,
runtime inference, rule/model arbitration, response mapping, and an audit trail. The honest
next step is evaluation on telemetry from a real application, where the interesting number
will be false positives per day, not F1.

## Supporting engineering

- **SSE stream** for live dashboard updates without polling.
- **Immutable audit log** of key events for compliance review.
- **Automated lifecycle actions** — rotation and revocation are state transitions with an
  audit entry, not ad-hoc deletes.
- **Remediation summaries** written for a human operator, because "status: BLOCKED" is not
  actionable on its own.

## What I would improve

- Replace synthetic evaluation with real telemetry and report precision/recall at a fixed
  false-positive budget.
- Make the model's decision boundary inspectable per event (feature contributions), not just
  the class.
- Add key ceremony and custody rules — generation is only one part of key management.

## What this demonstrates

Cryptographic lifecycle thinking, a detection architecture that fails safe, honest
measurement of an ML component, and the operational surface (audit, response, remediation)
that turns a detector into something a team can run.
