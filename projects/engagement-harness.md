# Engagement harness — design notes

Repo: [mode-17/engagement-harness](https://github.com/mode-17/engagement-harness)

A small request layer for engagements that have rules attached. Three problems recur across
programs, and all three are solved the same way every time: put the rule in the tool.

## Problem 1 — the mandatory header

Many programs require a research identifier on every request, in a specific header or
user-agent. It is the kind of requirement that is followed perfectly for the first hour and
forgotten at 2 a.m. on the request that matters.

**Fix:** the identifier is defined once in the config and attached by the wrapper. There is no
code path that sends a request without it.

## Problem 2 — out-of-scope hosts

An in-scope page almost always loads data from a host that is not in the scope table.
Convenience encourages testing it anyway; program rules and safe-harbour terms do not. A
promising API on an out-of-scope host is still out of scope, and a leaked token does not
extend the permission.

**Fix:** the wrapper parses every URL argument and refuses anything whose host is not in the
allowlist. The refusal is loud and non-zero-exit, so a script that pipes output stops rather
than silently continuing.

This is deliberately annoying. It fired during a real engagement on exactly the URLs I wanted
to test, which is the behaviour I wanted from it.

## Problem 3 — identity bookkeeping

Authorization testing is a diff: the same request under A, under B, and with no session. Doing
that by hand means juggling cookie jars, bearer tokens, and browser profiles until a session
gets crossed and a control becomes invalid.

**Fix:** identities are named in the config, each with its own cookie jar and optional token
file. The wrapper resolves the identity, attaches the right session, and writes the jar back
after the request. Cookies and tokens live in files that are gitignored, never in the repo.

## The browser half

Modern targets sit behind challenge layers — interactive edge challenges, device
fingerprinting, bot detection — that reject plain HTTP clients. The harness therefore has two
parts:

- **`hit`** — the `curl` layer for reproducible, loggable request/replay
- **`pw-session`** — a headed Playwright session through a proxy that signs in by hand,
  captures cookies, bearer tokens and storage, and records in-scope API traffic as JSONL

The browser half is deliberately **not** a login automator. Login is completed by a human in a
real window, because automating it is both fragile and, on some programs, indistinguishable
from the automated scanning the rules forbid. The tool captures the session a human created.

## Design decisions worth stating

- **Read-only by default.** The API-mapping helper only issues `GET`. Mutation is a deliberate
  act, never a side effect of a crawl.
- **Capture, do not transcribe.** Cookies, tokens and storage are dumped to files from the
  live session instead of being copied by hand.
- **Bot-defence aware.** Headed browser, realistic user-agent, tiny request volume. The goal
  is to read your own session's traffic, not to sweep the target.

## Limitations

- The scope guard parses URL arguments. A tool that hard-codes a host will not be caught by it —
  the guard is one layer of defence, not a complete one.
- The capture records request and response metadata, not response bodies for everything;
  bodies are added deliberately per engagement because they often contain other users' data.
- Nothing here replaces reading the program's rules. It encodes rules you have already read.
