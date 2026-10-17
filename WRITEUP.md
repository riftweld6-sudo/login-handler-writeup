# The Login Handler That Believed Every Token

**Severity:** High · **Class:** Authentication bypass / identity-assertion validation · **CVE:** Not assigned (placeholder: CVE-202X-XXXXX)

> A sanitized case study. All domains, identities, and payloads are synthetic. The proof of concept is conceptual and omits actionable exploit material.

## TL;DR / Summary

A consumer login endpoint accepted a client-supplied identity assertion without verifying its signature, issuer, audience, or lifetime. When the request also satisfied the account's device-trust check, the endpoint issued a session for an existing account; when the email was unregistered, it created a new confirmed account. The root cause was a fallback provider branch that decoded the assertion and trusted its `email` claim.

## Technical Deep Dive

### The shape of the flow

The login service supports several identity providers. Each provider has a branch that validates its own assertion format, then maps the result to a local account. That is the intended design.

The problem was the branch for everything else.

When the `provider` value did not match one of the validated providers, the handler fell through to a default arm. That arm treated the assertion as if an earlier step had already verified it. It decoded the token, read the claims, and used the `email` claim to find or create an account.

No signature check. No issuer check. No audience check. No `exp`, `nbf`, or `iat` check. The token was not a credential; it was a suggestion.

### The vulnerable sequence

```text
Client -------------------- POST /v1/login -------------------> Login service
  |                                                               |
  |  provider: "unexpected-provider"                              |
  |  assertion: <client-supplied token>                           |
  |  email: victim@example.test                                   |
  |                                                               |
  |                                      decode(assertion)        |
  |                                      findByEmail(claims.email) |
  |                                      deviceTrusted(...)       |
  |                                                               |
  <---------------- 200 + session, if device is trusted ---------|
```

The important detail is the order. The handler asked "is this device trusted for this account?" before it had established that the assertion was trustworthy. Device trust became the only gate standing between a forged claim and a session.

### What the handler accepted

In a controlled lab, the fallback arm accepted assertions that a correct verifier would reject:

| Assertion variant | Correct result | Observed result |
| :--- | :--- | :--- |
| Signed with a key the server never checked | Reject | Accepted |
| `alg: none` | Reject | Accepted |
| Symmetric algorithm with an attacker-chosen key | Reject | Accepted |
| Expired `exp` | Reject | Accepted |
| `nbf` / `iat` in the future | Reject | Accepted |
| Wrong issuer or audience | Reject | Accepted |

The table is the whole bug in one place. The handler was not checking whether the token was valid for this service. It was checking whether the token contained an email address.

### The device-trust gate

The session was not unconditional. The service tied session issuance to a device fingerprint pair. For an existing account:

- A trusted device pair returned `200` and an authorization code.
- An unknown device pair returned `202` with a mailbox challenge and no session.
- Missing or empty device fields returned `400`.

That precondition matters. It means the finding is not "any unauthenticated request can take over any account." It is "an attacker who can satisfy the device-trust check can bypass the identity check entirely." An attacker with access to the account's mailbox, or with a previously trusted device, meets that condition. The device check narrowed the attack; it did not make the assertion valid.

This is the part teams often miss. Device trust is a risk signal, not an authentication factor. It can answer "have I seen this device before?" It cannot answer "did this identity provider actually vouch for this person?" Using it as a substitute for assertion verification inverts the trust model.

### Why the email claim is the wrong key

The handler mapped the assertion to an account by email. Email addresses are mutable, reusable, and often unverified. The stable identifier is the pair `(issuer, subject)`: the issuer says who made the claim, and the subject is the issuer's stable identifier for that account.

When email is the key, two failure modes appear:

1. An attacker can claim an existing email and, if the device check passes, land in that account.
2. An attacker can claim an unregistered email and cause the service to create a new, confirmed account before the real owner ever arrives.

Both are consequences of treating a self-asserted string as an identity.

## Synthetic Proof of Concept

The request below is conceptual. The assertion is a placeholder, not a working token. The domains and identities are synthetic.

```http
POST /v1/login HTTP/1.1
Host: api.example.test
Content-Type: application/json

{
  "provider": "unexpected-provider",
  "assertion": "<client-supplied identity assertion>",
  "email": "victim@example.test",
  "deviceFingerprint": "<trusted-device-fingerprint>",
  "deviceRequestId": "<trusted-device-request-id>"
}
```

Observed response when the device pair was already trusted:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "challengeType": 0,
  "authorizationCode": "<opaque-code>",
  "user": {
    "id": "<victim-user-id>",
    "email": "victim@example.test"
  }
}
```

The authorization code could then be exchanged for a session, and the session could read the profile. The same request from an unknown device returned `202` with a mailbox challenge; missing device fields returned `400`.

The vulnerable logic, reduced to pseudocode:

```js
// Vulnerable: decode is not verify
const claims = decode(assertion);
const user = await users.findByEmail(claims.email);

if (await deviceIsTrusted(user, deviceFingerprint)) {
  return issueSession(user);
}
```

A correct implementation verifies the assertion before it looks up an account:

```js
// Pseudocode. Use a maintained library and your provider's JWKS.
const { payload } = await verifyAssertion(assertion, {
  algorithms: ["RS256"],
  issuer: TRUSTED_ISSUER,
  audience: CLIENT_ID,
  clockTolerance: 60
});

const user = await users.findByProviderSubject(payload.iss, payload.sub);
```

The device check still belongs in the flow, but after verification and as an additional policy signal. It should never be the thing that makes an unverified assertion acceptable.

## Impact Analysis

**Existing accounts.** If the attacker can present a trusted device pair, or complete the mailbox challenge, the forged assertion produces a session for the existing account. The session can read the profile and act as the user. This is account takeover, with a real precondition.

**Unregistered accounts.** The same branch auto-provisions an account for an unregistered email and marks it confirmed. At the measured sample, the flow allowed one request per account and showed no rate limit. That is an account factory: attacker-controlled identities that look like normal confirmed users and can consume trial or promotional entitlements.

**Adoption pre-hijacking.** An attacker can create an account for an address they do not own. If the real owner later registers and resets the password, the attacker's device trust can survive the reset. A post-reset forged login from that device can then reach the adopted account silently. This is a persistence layer on top of the same root cause, not a separate primitive.

**Downstream systems.** A separate legacy storefront accepted the same forged identity and exposed the account's order history and profile. That is the second lesson: the login endpoint was not the only relying party. Every service that accepts the session must validate the assertion or trust only the session minted after validation.

## What This Does Not Prove

This case study does not show a universal, zero-click takeover of every account. The device-trust check was real and did block unknown devices. It does not show an MFA bypass; accounts with an additional factor took a different challenge path. It does not show production exploitation: the testing was staging-only, with self-owned accounts and no third-party data.

The severity is High because the precondition is achievable and the impact is a full session. It is not Critical on the available evidence. Precision here matters more than the biggest possible number.

## Remediation

1. **Verify before decode.** Treat every client-supplied assertion as untrusted input. Verify the signature with the issuer's current JWKS, enforce an algorithm allowlist, and reject `none`.
2. **Pin the context.** Check `iss` against the expected issuer and `aud` against your client ID. If the token can carry multiple audiences, check the authorized party as the specification requires.
3. **Enforce time.** Reject expired tokens, tokens that are not yet valid, and tokens issued outside an acceptable skew. Use a small clock tolerance, not a large one.
4. **Map by `(iss, sub)`.** Use the stable subject identifier. Treat email as profile data, not as a primary key.
5. **Delete the fallback.** Replace the default provider arm with an explicit allowlist. Unknown providers should fail closed, not fall into a permissive branch.
6. **Centralize verification.** Every login path, including new providers and legacy endpoints, must call the same verification function. A second path is a second chance to get it wrong.
7. **Keep device trust in its lane.** Apply device policy after identity verification. A trusted device can reduce friction; it cannot substitute for proof of identity.
8. **Fail closed on missing input.** Empty device fields, missing assertions, and malformed claims should produce a rejection, not a skipped check.
9. **Test the negative matrix.** Include self-signed, `alg: none`, wrong issuer, wrong audience, expired, future-dated, and wrong-subject assertions in CI. Test every provider branch, including the one you think is dead.
10. **Log and alert.** Record the rejection reason and the provider value. A spike in unknown-provider attempts is worth seeing.

## Disclosure Timeline & Note

| Time | Event |
| :--- | :--- |
| T0 | Report submitted with a controlled, self-owned reproduction |
| T+1 | Acknowledged by the program |
| T+2 | Fix deployed |
| T+3 | Retest confirmed the assertion was rejected |
| T+30 | Sanitized write-up published with permission |

Exact dates are withheld. The timeline is intentionally relative.

This write-up was responsibly disclosed to the affected program. All company names, domains, IP addresses, endpoints, user identifiers, and customer data have been removed or replaced with synthetic values such as `api.example.test`, `victim@example.test`, and `CVE-202X-XXXXX`. The proof of concept is conceptual and does not include an actionable, unremediated exploit payload. It is published with permission for educational purposes.

If you take one thing from this: a decoder is not a verifier. The login handler did not need a more complicated exploit. It needed one more check.
