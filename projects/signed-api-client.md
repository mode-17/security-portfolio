# Signed API client — design notes

Repo: [mode-17/signed-api-client](https://github.com/mode-17/signed-api-client)

## The problem

A cloud provider's API v2 rejects every standard authentication scheme with
`403 invalid authorization scheme`. It uses a custom AWS-SigV4-style HMAC header instead:

```
Authorization: EXO2-HMAC-SHA256 credential=<KEY>,expires=<UNIX>,signature=<B64>
```

The vendor CLI signs transparently, but it does not expose every operation. Specifically, it
does not expose the ones that matter for an IAM privilege-escalation assessment:

- `POST /iam-role/{id}/assume`
- `PUT /iam-role/{id}:policy`
- `POST /iam-organization-policy:reset`

If the endpoints that implement role assumption and policy rewriting are unreachable, the
assessment cannot answer the question it was asked. So the signing scheme had to be
reimplemented.

## How the scheme was recovered

It was read out of the vendor SDK's request-signing function, then verified against a
signature captured from the real CLI. The two matched byte-for-byte, which is the only reason
I trusted the implementation:

```
canonical = "\n".join([
    "<METHOD> <escaped-path>",
    "<body>",              # empty string when there is no body
    "<query-values>",      # values of single-valued params, names sorted A-Z
    "",                    # headers are not signed
    "<expires-unix>",
])
signature = base64(HMAC-SHA256(secret, canonical))
```

And the header, where order matters:

```
EXO2-HMAC-SHA256 credential=<KEY>[,signed-query-args=<n1>;<n2>],expires=<UNIX>,signature=<B64>
```

Two details that are easy to get wrong and produce a valid-looking `403`:

- `signed-query-args` only appears when there are query parameters, and it lists **names
  only**, semicolon-joined, alphabetically sorted.
- Only single-valued query parameters are signed, and their **values** are concatenated in
  name-sorted order with no separator.

## What the client does

- Signs an arbitrary method, path, body and query set
- Selects credentials by **named identity**, so a low-privilege session and an owner session
  can be exercised against the same endpoint — the whole point of a privilege-escalation test
- Reads credentials from a gitignored `.env`, never from the repo

## Why this is in the portfolio

Because it is the difference between a scanner report and an assessment. A scanner would have
reported the API as "authentication not understood". Reimplementing the scheme from the SDK
and validating it against a captured CLI signature turned an unreachable surface into a
testable one, and the identity-switching design is what makes A-vs-B privilege testing
possible on it.

It is also a reminder that the interesting part of API testing is often not the vulnerability
class — it is getting to the point where the question can be asked at all.

## Limitations

- The signing implementation is tied to the vendor's SDK version at the time. If the SDK
  changes its canonical form, the signature breaks and the failure looks like `403`.
- The base region endpoint is configured in the script, not discovered.
- It signs what you tell it to sign. Destructive methods are possible; the README says so and
  the tool does not stop you. Scope discipline is the operator's job.
