# 05 — 3SS: security services platform with on-chain milestone escrow

**Type:** full-stack application · **My role:** design and implementation · **Source:** private
**Stack:** Python/Flask, SQLAlchemy, Redis + RQ, PostgreSQL (Supabase), Solidity, web3.py,
vanilla JS, Render

---

## The problem

A small security consultancy needs the operational spine of a business: attract leads, qualify
them, run a project with visible milestones, and get paid per milestone without either side
having to trust the other's accounting. That is a different problem from a brochure site, and
it is where most small consultancies run on spreadsheets and DMs.

## What the platform does

### Lead pipeline

An asyncio/aiohttp collector pulls technical roles from several job APIs concurrently,
normalises them, and scores them against service keywords. A custom filter rejects
non-technical roles and classifies the rest into cybersecurity, web development, or both.
Work is decoupled through a Redis queue so extraction does not block processing, and a worker
persists normalised leads and generates context-aware outreach drafts.

The scoring is intentionally simple and inspectable. A lead-scoring model nobody can explain
gets ignored by the person who has to send the message.

### Project delivery

- **Admin dashboard** — clients, projects, lead review, project phases, revenue
- **Client portal** — milestone progress, updates, staging links
- **Roles** — `Flask-Login` with a hard admin/client split, enforced server-side

### Milestone payments

A Solidity `ProjectEscrow` contract with an explicit state machine:

```
AwaitingActivation → AwaitingDeposit → Deposited → Commenced
                   → MidpointReleased → Completed
```

Both admin and client must activate before a deposit is accepted. Each milestone release is a
separate on-chain event, and completion records a build hash and domain. There is an emergency
refund path. The contract has role modifiers (`onlyAdmin`, `onlyClient`) and emits an event
for every state transition, so the off-chain app reconciles to chain state rather than
maintaining a parallel truth.

That last part is the security design decision worth stating: **the chain is the source of
truth for money, the database is the source of truth for presentation.** Duplicating balances
in both places is how escrow systems drift and double-pay.

## Security controls

- Admin/client authorization enforced at the route layer, not by hiding navigation
- Session-based auth with server-side role checks on every mutating endpoint
- Environment-based secrets; no keys in the repository
- SQLAlchemy parameterisation throughout
- Web3 calls over a managed RPC provider; transaction hashes recorded for reconciliation
- DKIM-signed outbound email (`dkimpy`) so outreach does not land in spam or get spoofed

## Deployment

Render web service + background worker + Redis, Supabase Postgres with a transaction pooler,
Alembic migrations, and git-push deploys. The split between the web service and the worker is
what keeps the crawler from competing with request handling.

## What I would improve

- Contract audit and a formal invariant test suite (no double release, no release before
  deposit, refund only by the authorised path) before handling real funds.
- Idempotency keys on milestone release, so a retried web3 call cannot double-submit.
- Move outreach templates out of code and into a reviewed content store.
- Add structured audit logging for admin actions, matching the standard used in SSMS.

## What this demonstrates

Full-stack delivery under real constraints: async workers, a queue, a relational schema, a
smart-contract state machine, and a payment reconciliation model where the on-chain state is
authoritative. It is also the project that most directly connects the offensive work to a
business: the platform sells the testing service, and the escrow design is a lesson in not
trusting a single ledger.
