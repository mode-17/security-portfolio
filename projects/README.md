# Projects

Design notes for the tooling I build or adapt when an engagement needs something the
off-the-shelf tools do not do.

| Project | Repo | Problem it solves |
|---|---|---|
| Engagement harness | [mode-17/engagement-harness](https://github.com/mode-17/engagement-harness) | Multi-identity requests with a scope guard that refuses out-of-scope hosts and per-program mandatory headers that cannot be forgotten |
| Signed API client | [mode-17/signed-api-client](https://github.com/mode-17/signed-api-client) | An API that rejects standard auth schemes, and an IAM privilege-testing surface the vendor CLI does not expose |

## The rule I apply before building anything

A tool has to earn its place. I build when one of these is true:

1. **A mistake is easy to make and expensive.** Forgetting a mandatory research header can
   get testing blocked, or worse, make traffic look unauthorised. That belongs in code, not
   in a checklist.
2. **The existing tool cannot reach the surface.** If the vendor CLI does not expose an
   endpoint, and the endpoint is the point of the engagement, the client has to be written.
3. **The work is repetitive and identical.** Capturing session cookies and bearer tokens by
   hand for every identity in every session is transcription, and transcription produces
   errors.

If none of those apply, I use what already exists.
