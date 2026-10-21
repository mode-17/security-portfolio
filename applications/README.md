# Applications

Security applications I designed and built. Source is private; these are architecture and
design notes, not code drops. Walkthroughs available on request.

| # | Application | Domain | Stack |
|---|---|---|---|
| 01 | [A-WIDS](01-awids-wireless-ids.md) | Wireless intrusion detection + active deception | Python, Scapy, Flask, SQLite |
| 02 | [SCKMS](02-sckms-key-lifecycle.md) | Cryptographic key lifecycle + ML anomaly detection | Python, Flask, scikit-learn, SQLite |
| 03 | [SSMS](03-ssms-results-integrity.md) | Tamper-evident academic records | Node.js, Express, PostgreSQL, vanilla JS |
| 04 | [GhostPixel](04-ghostpixel-steganography.md) | Client-side encrypted steganography | Vanilla JS, Web Crypto |
| 05 | [3SS](05-3ss-security-services-platform.md) | Security services platform with on-chain escrow | Flask, Redis, PostgreSQL, Solidity, web3.py |
| 06 | [FUTO NIDS lab](06-futo-ids-placement.md) | Sensor placement and layered detection | Docker, Suricata, ModSecurity, DVWA |
| 07 | [Laudate](07-laudate-ticketing.md) | Payment and ticketing with check-in | Node.js, Express, PostgreSQL |

## Why these belong in a security portfolio

Offensive testing and security engineering are the same skill read in two directions. To find
a broken authorization check you have to know how one is supposed to be built; to build a
tamper-evident record you have to know how it will be attacked.

Each of these projects makes a specific security claim — a hash chain detects tampering, a
signed payload detects modification, a threat score responds to a deauth flood — and the
write-up says what the claim actually covers and where it stops. That last part matters more
than the feature list.
