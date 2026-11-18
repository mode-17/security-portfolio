# 08 — Multi-chain stablecoin issuer · my preflight said GO, my blueprint said NO-GO

**Platform:** HackerOne · **Window:** 2026 · **Outcome:** NO-GO at blueprint — no test harness
built, no PoC written, ~30 read-only RPC calls · **Focus:** Solidity contracts, bridge
accounting, role design

> Target name withheld and all addresses replaced with placeholders. This is the case study I
> would put first in an interview, because the finding is that **my own gate reversed my own
> optimistic score before I spent anything**.

---

## Why this target

A fiat-backed stablecoin deployment across six chains: seven ERC-20 proxies, two bridge
contracts, four in-scope implementations, roughly 900 lines of own code on OpenZeppelin 5.x.
Contract programs are attractive because the code is the whole surface — no bot layer, no
account gates, everything readable.

## Step 1: the preflight said GO

The gate scored **16/20** on the Web3 asset: readable source, reachable value, no special
access needed to evaluate, documented payout bands. Verdict: **GO — Web3 surface only**, with
an explicit NO-GO on the web/app surface (two independent vetoes, so the web half was closed
before it started).

16/20 is a comfortable GO. I started planning a Foundry fork harness.

## Step 2: the blueprint disagreed

Blueprinting means reading every externally reachable function and asking who can call it. Two
of the preflight cells turned out to be wrong, and both were wrong in the optimistic direction:

| Dimension | Preflight | Corrected | Why |
|---|---|---|---|
| Value-surface reachability | 2 | **0** | No unprivileged address reaches a value-changing action. Every mint path is role-gated. The only anonymous call lets a caller burn **their own** tokens. |
| Duplicate risk | 1 | **0** | The single class a stranger can reach is quoted in the program policy as an accepted defence-in-depth item — already filed, already closed. |

Revised score ≈ **13/20**, below the GO band. Then a veto fired on top of it:

> **V5** — the only surface reachable from outside produces exactly the classes the program
> explicitly excludes: centralisation, key compromise, audit findings, registry pollution.

The preflight could not have caught this. V5 is a *composition* judgement: it needs the
reachability map from the blueprint to see that the one reachable surface and the one excluded
class list are the same set. That is the difference between a scope check and a blueprint.

## What the blueprint found (none of it reportable)

**Every privileged role is a hot EOA.** Admin, pauser, upgrader and fee-manager on all seven
tokens and both bridges resolve to a single externally-owned address with **zero code**. The
bridge-operator role is a second EOA. Exactly one address in the role table — the admin and fee
collector on one chain — is a contract: a 171-byte proxy, shaped like a multisig.

That is the most security-relevant fact about the deployment. It is also explicitly not
reportable, because the policy names centralisation and key compromise as accepted design
choices. A researcher who does not read the exclusion list files this and gets an N/A.

**The bridge trusts an operator and keeps no on-chain proof.** The fulfilment function takes a
token, recipient, amount, source chain id, source transaction hash and deposit id from a
privileged key, then mints against a daily cap. The replay guard proves only that *this key has
not used this string before* — not that a deposit burned, not that the amount matches the burn.

So the only place a double-mint could live is the off-chain operator. And the operator is
untestable by rule: verifying the fulfilment leg means depositing on a live chain, which the
program forbids touching real funds to do, or calling the operator service, whose plausible
hostnames return `403` from this egress.

**The on-chain code is clean on the classes that matter.** Six hypotheses, six controlled
negatives:

| ID | Checked | Result |
|---|---|---|
| N-1 | `initialize` reachable on 7 proxies × 6 chains | no — all have real `name()` and non-zero supply; the implementation constructor disables initializers |
| N-2 | `permit` replay across chains at the shared address | no — EIP-712 domain includes `chainid` |
| N-3 | `abi.encodePacked` collision in the replay key | no — three fixed 32-byte types |
| N-4 | Overflow in the daily-mint accumulator | no — 0.8.x checked math |
| N-5 | Reentrancy in deposit / fulfil | no — `nonReentrant`, flag set before the external call |
| N-6 | Fee plus full amount double-pulled from the depositor | no — `fee + (amount − fee) == amount` |

## Step 3: the permissionless surface, exhaustively

When the answer is "no", the useful artifact is the proof that the search was complete. Every
external function across the four implementations, classified by what an anonymous caller can
do:

| Entry point | Anonymous capability | Value impact |
|---|---|---|
| `transfer` / `transferFrom` / `approve` / `permit` | move **their own** tokens | none |
| `burn` / `burnFrom` | burn **their own** tokens, allowance-gated | none |
| Bridge deposit | burn **their own** tokens, pay the fixed fee | negative for the caller |
| Registry admin setters on a caller-supplied token | configure a token **they already control** | none — minting still needs a role |
| all `view` functions | read | none |

Everything that moves supply is behind a role held by the two EOAs or by the bridge contract
itself. There is no unprivileged path to value.

The griefing sweep — the usual home of a Medium — is also dead: the replay key is only writable
inside the operator-only function; the daily cap only advances in role-gated mints; front-running
a deposit is both caller-chosen-recipient and a named MEV exclusion; reentrancy in the registry
setter only reaches the attacker's own token. One genuine curiosity — `registerToken(address(0))`
makes a later validation branch unreachable dead code — is cosmetic.

## Step 4: the re-assessment, under pressure

The pushback was fair: a Critical is not mandatory, High, Medium and Low all pay. So the bar
was re-run.

It did not move, because **the bar was never what was binding.** The bands are: Critical up to
$25,000, High up to $5,000, **Medium up to $1,000**, **Low is swag, not cash**, with KYC above
$500. The true floor of the program is roughly $500–$1,000 plus an identity process. Lowering
the severity threshold buys nothing when the finding count is zero.

## Outcome

**NO-GO. Stopped.** No Foundry install, no fork harness, no PoC.

Total cost: 4 unauthenticated GETs for the preflight, ~30 read-only RPC calls, 4 explorer
fetches. Zero state-changing transactions, zero accounts, zero requests beyond the preflight
roots.

That is cheaper than one hour of a harness that would have found the same nothing.

## What this engagement demonstrates

1. **A preflight is a prior, not a verdict.** It scores the *asset* from the outside. The
   blueprint scores *reachability* from the inside, and it can overturn a GO. Building the
   reversal into the process is what makes the gate worth having.
2. **"Clean code" is a result.** Six controlled negatives on 900 lines is a real deliverable.
   It stops the next researcher — or the next me — from re-running the same six checks.
3. **Read the exclusion list as an attack-surface filter.** The most security-relevant fact
   here was a table of hot EOAs. Writing it up would have been an instant N/A. Knowing that
   *before* the blueprint is what turned a tempting report into a stop.
4. **Cost discipline is a skill.** The expensive failure mode on a contract program is a
   two-day fork harness in service of a hypothesis the policy already excludes.

## Tools used

Block explorer and public RPC endpoints for read-only state, and nothing else. The tooling
decision here was to spend an hour reading instead of a day installing.
