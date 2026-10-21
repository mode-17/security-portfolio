# 06 — FUTO NIDS lab: sensor placement and layered detection

**Type:** lab / demonstration environment · **My role:** design and build · **Source:** private
**Stack:** Docker Compose, Suricata, OWASP ModSecurity CRS, DVWA, Kali

---

## The question

A network detection rule is only as good as the sensor's position. The same attack, against
the same vulnerable application, produces different evidence depending on whether the sensor
sees the traffic before or after the WAF, and whether the WAF is even in the path.

Most "install Suricata" tutorials skip that. The lab exists to make it visible.

## Topology

```
attacker (Kali) ──► WAF (ModSecurity CRS) ──► target (DVWA)
                        │                          │
                   NIDS "strategic"           NIDS "poor"
                   (sees filtered traffic)    (attached to target)
```

Two Suricata sensors on the same bridge network, deliberately placed:

- **Poor placement** — attached to the target container's network namespace. It sees what
  reaches the application, which is exactly the traffic the WAF already allowed.
- **Strategic placement** — attached to the WAF. It sees the request before filtering, which
  includes the attacks the WAF blocks.

Both sensors run the same rule set, so the only variable is position.

## Detection rules

A small, readable rule set rather than a full ruleset, so each alert can be explained:

| SID | Signal |
|---|---|
| 1000001 | SQL injection pattern (`UNION SELECT`) in the URI |
| 1000004 | XSS pattern (`<script>`) in the URI |
| 1000002 | Web traffic on 80/8080 |
| 1000003 | ICMP |

The point is not coverage. It is that each row maps to a scenario a non-technical audience can
follow: attack arrives, WAF acts, sensor sees or does not see.

## The four demo scenarios

1. **No security** — the application is reachable and compromised.
2. **WAF only** — the attack is blocked, and the sensor behind the WAF has nothing to show.
3. **NIDS only** — the attack succeeds and the sensor records it, but nothing stops it.
4. **WAF + NIDS, correctly placed** — the attack is blocked *and* recorded.

Scenario 2 is the teaching moment: a blocked attack that leaves no evidence is a detection
blind spot. "The WAF stopped it" and "we know it happened" are different controls, and an
incident review needs the second.

## Honest result

This is a demonstration environment, not a measurement study. The logs from the two placements
are kept for comparison, but the run counts are small and the value is the topology and the
scenario narrative, not a detection-rate claim. Presenting a tiny sample as a performance
result would be exactly the kind of overclaim the rest of this portfolio avoids.

## Operating details

- One-command bring-up; the demo script waits for Suricata's engine start and for the
  attacker container to have `curl` before running a scenario.
- Logs are mounted per sensor so the poor/strategic comparison survives a restart.
- The attacker container is isolated on an internal bridge, not exposed to the host.

## What I would improve

- Add a controlled traffic generator and repeat each scenario enough times to report detection
  counts with an interval, not an anecdote.
- Add an evasion scenario (encoding, case variation, chunked bodies) to show where the rules
  stop working.
- Forward both sensors to a single collector to demonstrate correlation across placement.

## What this demonstrates

Detection engineering judgement: the sensor's position is part of the control, a blocked
attack still needs evidence, and a lab is only useful if it can explain *why* a control
failed — not just that an alert fired.
