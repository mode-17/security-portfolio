# Test plan — <target>

Columns: ID · component · class · exact test · expected-safe result · priority · status ·
evidence.

**Status values:** `PENDING` · `IN PROGRESS` · `DONE` · `BLOCKED-<reason>` · `N/A`.

A blocked row stays in the table with the reason. Deleting blocked tests is how the same
blocker gets rediscovered.

## Accounts

| Account | Role | Identifier | Notes |
|---|---|---|---|
| A | | | |
| B | | | |
| anon | — | — | |

## Rows

| ID | C-ID | Class | Test | Expected-safe | Priority | Status | Evidence |
|---|---|---|---|---|---|---|---|
| T-001 | | BOLA | | | P2 | PENDING | |

## Execution rules

- Control request before every tampered request; record both.
- A-vs-A is the control; A-vs-B is the test. Add a no-session request to prove the check
  exists at all.
- Unique marker (8+ random chars) on any value that gets stored.
- No state-changing test without a reversible plan.
- Never touch another user's data. Stop at proof.

## Negatives (tested and safe)

| ID | Test | Evidence | Date |
|---|---|---|---|
| | | | |
