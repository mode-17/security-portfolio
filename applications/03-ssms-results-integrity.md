# 03 — SSMS: tamper-evident academic records

**Type:** security application · **My role:** design and implementation · **Source:** private
**Stack:** Node.js, Express, PostgreSQL, JWT, vanilla ES modules

---

## The problem

Academic results are a high-value target for insider modification: a grade change is a
database `UPDATE`, and most systems have no way to distinguish an authorised correction from
a quiet edit. The requirement was not "encrypt the grades". It was "make an unauthorised
change detectable, and make the detection independent of the person who made the change".

## Design

Every result is signed on write and re-verified on every read:

```
payload   = student_id : course_code : ca_score : exam_score : final_score : lecturer_id
signature = HMAC-SHA256(payload, server_only_key)
```

- The key never leaves the server.
- Verification uses `crypto.timingSafeEqual`, so the check does not leak how much of a
  candidate signature matched.
- Scores are normalised to two decimals before signing, so `70` and `70.00` cannot produce
  two different valid signatures.
- Every read re-verifies. A direct database edit produces a signature mismatch on the next
  page load, and the mismatch is logged as `TAMPER_DETECTED`.

## Authorization model

JWT with role-scoped claims and server-side checks, not UI gating:

| Role | Read | Write |
|---|---|---|
| Admin | all results, audit log | user and registration management |
| Lecturer | results for assigned courses | create/edit, which re-signs |
| Student | own published results only | none |
| Registrar | enrollment records | approve/reject registrations |

The student scope is derived from the token subject, not from a client-supplied student ID.
That is the difference between a role check and an object-level check, and it is where most
academic systems leak.

## Audit trail

An append-only `audit_logs` table records logins, failures, result creation and edits,
tamper detections, views, registration decisions, password resets and reveals. Application
code performs inserts only — no update or delete path exists. The admin view supports
filtering by user, action, date and tamper flag, plus aggregate statistics.

An audit log that the application can rewrite is not an audit log. The constraint is enforced
in code and should be enforced in the database with a restricted role as well.

## What is deliberately not claimed

- HMAC proves **integrity and origin**, not confidentiality. A database dump still exposes
  grades.
- A compromised server holds the key and can re-sign anything. This defends against database
  access and insider edits through the application, not against full server compromise.
- The hash is not a digital signature: there is no third-party verifier. Moving to asymmetric
  signing would let an external registrar verify without holding the signing key.

## Security controls in the stack

`helmet` for headers, `bcryptjs` for password hashing, JWT with expiry, CORS restricted to
the frontend origin, parameterised SQL through `pg`, input validation on score ranges
(0–100, CA 0–30, exam 0–70), and role middleware applied at the route layer.

## What I would improve

- Asymmetric signing with a published public key, so verification does not require the secret.
- Database-level append-only enforcement (revoked `UPDATE`/`DELETE` grants on the audit table).
- Hash-chaining the audit rows, as in A-WIDS, so deletion of the tail is also detectable.

## What this demonstrates

Threat-model-driven design: the control was chosen for the specific attacker (insider with
database access), the scope check was placed at the object level rather than the UI, and the
write-up states where the guarantee ends.
