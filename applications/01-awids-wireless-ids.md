# 01 — A-WIDS: wireless intrusion detection with active deception

**Type:** security application · **My role:** design and implementation · **Source:** private
**Stack:** Python, Scapy, Flask, SQLite, Chart.js

---

## The problem

Wireless attacks are cheap and loud, but most small networks have no visibility into layer 2.
A deauthentication flood does not appear in an HTTP access log. An EAPOL capture does not
trigger a firewall rule. The goal was a system that watches raw 802.11 management frames,
scores what it sees, and does something other than write a log line.

## Architecture

Four concurrent concerns, deliberately decoupled so packet capture never blocks on storage or
the UI:

| Thread | Job |
|---|---|
| Flask main | HTTP API, dashboard, Server-Sent Events |
| Hardware poll | Interface state and monitor-mode transitions |
| Sniffer | Scapy capture on the monitor interface; extraction only, pushes raw events |
| Scoring consumer | Device profiling, threat scoring, countermeasures, ledger writes, UI queue |

The event bus between them is a thread-safe in-memory queue. Capture drops when the consumer
is slow, so the sniffer is kept as close to "copy bytes and enqueue" as possible.

## The three parts that matter

### 1. Threat scoring is multi-factor, not a signature

A fixed rule ("more than N deauth frames") is easy to evade by slowing down. The score
combines:

- **Frequency** — packet rate in a rolling 5-second window
- **Proximity** — RSSI bands, because an attacker in the room is a different risk from one
  outside
- **Device history** — deviation from a per-device baseline persisted in SQLite

The output is a 0–100 score with an explainability breakdown in the UI. A score you cannot
explain is a score an operator will not trust, and an untrusted alert gets ignored.

### 2. Deception, not just detection

On a critical score the system crafts and injects fake EAPOL four-way handshake frames
(M1/M2 with invalid MICs) toward the attacker. The point is not to "hack back". It is to make
the attacker's capture useless and waste their time, while the operator gets a real signal.

This is the part I would want reviewed hardest: active countermeasures in a wireless
environment can affect third parties. It is gated by a configurable threshold, it is clearly
documented, and it is off unless the operator enables it. Defensive tooling that misbehaves
is worse than no tooling.

### 3. The log is a hash chain

Every alert is written with a SHA-256 hash over its content and the previous record's hash,
anchored at `GENESIS`. Verification walks the chain. Editing a row by hand breaks every
subsequent link, which turns "the database says so" into "the database can prove it was not
edited after the fact".

The limit is honest: a hash chain detects modification, not deletion of the tail, and not an
attacker who rewrites the whole chain from genesis. It raises the cost of tampering; it does
not make the log immutable.

## Operational details

- `preflight.sh` checks monitor mode and injection capability before launch, read-only.
- `restore_net.sh` restores host connectivity without a reboot — network manipulation has to
  be reversible.
- The interface selector protects the host uplink; the tool must not take the operator
  offline.
- The Kali bundle is self-contained so the tool can be moved to an assessment host.

## What I would improve

- The scoring weights are hand-tuned. A labelled capture set would let them be fitted and
  evaluated instead of argued about.
- The hash chain should periodically anchor to an external store to bound tail truncation.
- The deception module needs a formal third-party impact assessment before any use outside a
  controlled lab.

## What this demonstrates

Real-time packet capture and threading under load, a security decision model that is
explainable, active response with a stated blast radius, and a tamper-evident record — the
same primitives that show up in detection engineering and in testing whether a system's
integrity claims hold.
