# 07 — Laudate: payments, tickets and check-in

**Type:** production web application · **My role:** design and implementation · **Source:** private
**Stack:** Node.js, Express, PostgreSQL, Sequelize, Paystack, bcrypt, JWT, rate limiting

---

## The problem

An event needs to sell tickets, reconcile two payment paths (automated gateway and manual bank
transfer), issue a unique ticket, and validate it at the door — including the difference
between a regular and a VIP ticket. The hard part is not the payment page. It is the state
machine between "money arrived" and "this person is allowed in", especially when one of the
paths is a human confirming a bank transfer.

## Flow

```
register → pay
            ├── Paystack  → webhook confirmed → ticket issued
            └── bank transfer → admin verifies → ticket issued
                                                        ↓
                                        check-in / check-out at the door
```

## Security and correctness decisions

- **Ticket issuance is server-side and post-reconciliation.** No ticket is generated on the
  client, and no ticket is issued on an unconfirmed payment. The redirect back from the
  gateway is not treated as proof of payment; the server confirms it.
- **Manual transfers are a separate, audited state.** An admin verification action is required,
  and it is not the same code path as the automated confirmation. Mixing them is how free
  tickets happen.
- **Ticket type is derived from the amount paid** (regular vs VIP), not from a client-supplied
  field.
- **Door validation is a server lookup**, not a signature the ticket holder can generate.
  Check-in and check-out are explicit transitions, so a ticket cannot be reused indefinitely.
- **Auth:** bcrypt password hashing, JWT sessions, admin routes gated server-side.
- **Abuse controls:** `express-rate-limit` on the public endpoints, `helmet` for headers, CORS
  restricted to the frontend, parameterised queries via Sequelize.
- **Performance:** indexes on role, email, user ID and status; pagination at 30 per page;
  indexed search for door lookup so validation stays fast when the queue is long.

## What I would improve

- Move manual verification to a two-person rule with an immutable audit entry — one admin
  confirming money is a single point of failure.
- Add idempotency on the payment webhook, so a retried delivery cannot issue two tickets.
- Store ticket validation as a signed, rotating code as a fallback for a door with no network,
  while keeping the server as the authority.
- Add reconciliation reporting: payments received vs tickets issued vs tickets scanned.

## What this demonstrates

Payment-state correctness: the difference between a redirect, a webhook and a human
confirmation, and why treating them as equivalent is a security bug. It is also the project
closest to the kind of business-logic surface I test on bug bounty programs — building it made
the failure modes obvious.
