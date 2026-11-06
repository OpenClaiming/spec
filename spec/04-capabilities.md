# §4 Capability Claims

## 4.1 Overview [informative]

A capability claim is an OpenClaim (§1) whose `stm` grants a named permission from the issuer to the subject. Capabilities may be further delegated by the grantee, forming a delegation chain. Verifiers evaluate the entire chain to confirm authority originates from a trusted root.

A common use case: an author grants a publisher the right to serve their content on specific domains. The publisher presents this signed grant to a verifying party (a jet, an API, a contract) in lieu of a password or registration.

## 4.2 Capability statement structure

The `stm` of a capability claim MUST be a JSON object containing at least:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | REQUIRED | A URI identifying the capability type |
| `grant` | object | REQUIRED | The specific permissions being granted |

The `grant` object's schema is defined by the `type` URI. Verifiers that do not recognize a `type` MUST reject the claim.

**Example capability claim:**

```json
{
  "ocp": 1,
  "iss": "data:key/es256;base64,<author-pubkey>",
  "sub": "data:key/es256;base64,<publisher-pubkey>",
  "nbf": 1735689600,
  "exp": 1767225600,
  "stm": {
    "type": "https://openclaiming.org/caps/content-access/v1",
    "grant": {
      "allowedOrigins": ["https://publisher.example.com", "*.publisher.example.com"],
      "rateLimit": { "minutesPerDay": 480 },
      "priceRange": { "min": 0, "max": 100 },
      "irrevocableUntil": 1751328000
    }
  },
  "sig": "<base64-signature>"
}
```

## 4.3 Revocability

A capability grant is revocable by default. An issuer may make a grant irrevocable until a specified time by including the `irrevocableUntil` field in the `grant` object.

- **Revocable (default):** `irrevocableUntil` is absent or null. The issuer may revoke the grant at any time by updating the key document at the `iss` URL to `active: false`, or by any application-specific revocation mechanism.
- **Irrevocable until timestamp:** `irrevocableUntil` is a non-zero Unix timestamp. A verifier MUST honor the grant without checking for revocation until `now ≥ irrevocableUntil`. After that timestamp, the grant becomes revocable normally.

A verifier that encounters `irrevocableUntil` in the future MUST NOT check the issuer key URL for revocation during that period (even if the issuer's key URL returns `active: false`).

Inline keys (§3.3.1) are always non-revocable by mechanism. Authors who need revocability MUST use URL-based keys.

## 4.4 Delegation chains

A subject who has received a capability grant MAY re-delegate part or all of that capability by issuing their own OpenClaim with:
- `iss`: the delegating party's key
- `sub`: the new grantee's key
- `stm.grant`: permissions that are a subset of (or equal to) those received

A delegating claim MUST cite the parent claim that authorized the delegation. This citation is placed in the `stm` as `basis`:

```json
{
  "ocp": 1,
  "iss": "data:key/es256;base64,<publisher-pubkey>",
  "sub": "data:key/es256;base64,<sub-publisher-pubkey>",
  "nbf": 1735689600,
  "exp": 1751328000,
  "stm": {
    "type": "https://openclaiming.org/caps/content-access/v1",
    "basis": "sha256:<hex-sha256-of-canonical-parent-claim>",
    "grant": {
      "allowedOrigins": ["https://sub.publisher.example.com"],
      "rateLimit": { "minutesPerDay": 60 }
    }
  },
  "sig": "<base64-signature>"
}
```

**`basis` field:**  
The `basis` field MUST be a string of the form `sha256:<hex>` where `<hex>` is the lowercase hexadecimal encoding of the SHA-256 hash of the parent claim's canonical bytes (the JCS-canonical form of the parent with its `sig` field present — i.e., the bytes a verifier would hash to reference the parent unambiguously).

> **Status note:** The `basis` field is a proposed extension. Applications that do not use delegation chains MAY omit it.

## 4.5 Chain verification algorithm

To verify a delegation chain ending at a leaf claim `C_n`:

**Inputs:**
- The leaf claim `C_n` to be verified
- The chain of ancestor claims `[C_1, C_2, ..., C_{n-1}]` where `C_1` is the root (issued by the trusted root)
- A set of trusted root public keys `T`

**Algorithm:**

1. Verify `C_1` as a standalone claim (§1.7). If it fails, **reject**.
2. Check that the issuer of `C_1` is in `T`. If not, **reject**.
3. For each `i` from 2 to `n`:
   a. Verify `C_i` as a standalone claim (§1.7). If it fails, **reject**.
   b. Check that the issuer of `C_i` matches the subject of `C_{i-1}`. If not, **reject** (chain break).
   c. Compute the SHA-256 of the canonical bytes of `C_{i-1}` (with `sig` present). Check that `C_i.stm.basis` equals `sha256:<that hex>`. If not, **reject** (basis mismatch).
   d. Check that the grant in `C_i` does not exceed the grant in `C_{i-1}` (scope reduction). The rules for scope comparison are defined by the capability type. If `C_i` claims more than `C_{i-1}` granted, **reject**.
   e. The effective `exp` for `C_i` is `min(C_i.exp, C_{i-1}.exp)` (unless either is 0, meaning no expiry, in which case the other's value applies; if both are 0, there is no expiry).
4. Apply temporal constraints from §1.5 using each claim's own `nbf`/`exp` individually, and the effective `exp` from step 3e for the leaf.
5. **Accept** the chain.

## 4.6 Scope reduction

A delegation MUST NOT grant permissions exceeding those in the parent grant. Verifiers MUST enforce this. The comparison rules depend on the capability type:

- **Origin lists:** The delegated `allowedOrigins` MUST be a subset of the parent's `allowedOrigins`. Wildcard matching is defined by the capability type profile.
- **Rate limits:** The delegated `rateLimit` values MUST be ≤ the parent's values.
- **Price ranges:** The delegated `priceRange` MUST fall within the parent's range (delegated `min` ≥ parent `min`, delegated `max` ≤ parent `max`).
- **Time bounds:** The delegated `exp` MUST be ≤ the parent's `exp` (or equal if the parent has no expiry).

Verifiers that cannot compare a capability field because its type is unknown MUST reject the delegation.
