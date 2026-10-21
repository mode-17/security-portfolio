# Disclosure and content policy

This repository contains redacted write-ups of authorized security testing. It is published
for professional portfolio purposes.

## What is deliberately not here

- Client, program, and vendor names (unless the vendor is already visible to any visitor)
- Live credentials, session tokens, cookies, or API keys
- Other users' personal data
- Request/response captures containing identifiers
- Working exploit payloads against systems that may still be vulnerable
- Anything under an NDA or a program rule that prohibits disclosure

## If you believe something here crosses a boundary

Open an issue on this repository or contact me through GitHub, and describe the specific
content. I will remove or redact it while the concern is reviewed. Disclosure concerns take
priority over portfolio completeness.

## Reporting a vulnerability in my own projects

For the tooling repositories linked from here, use GitHub's private vulnerability reporting
on the relevant repository, or open an issue if the report does not contain sensitive detail.
Please do not test against infrastructure you do not own.

## Authorization statement

All engagements referenced here were performed under published bug bounty programs or written
authorization, on accounts I created and control, against in-scope assets only. Where a
program required an identifier header, it was sent on every request. Where a program required
created content to be deleted within a fixed window, the test was skipped if deletion could
not be guaranteed.
