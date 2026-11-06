# §1 Core Claim Format

## 1.1 Overview [informative]

An OpenClaim is a JSON object signed by an issuer to attest a statement about a subject. The format is intentionally minimal: seven top-level fields, one of which is the signature itself. The statement payload (`stm`) is schema-flexible and defined by higher-level profiles and extensions built on this core.

## 1.2 Top-level fields

An OpenClaim MUST be a JSON object. It MUST contain the following fields:

| Field | Type | Description |
|---|---|---|
| `ocp` | integer | Specification version. MUST be `1`. |
| `iss` | string | Issuer identifier. See §1.3. |
| `sub` | string | Subject identifier. See §1.3. |
| `stm` | object | Statement payload. Schema is profile-defined. |
| `nbf` | integer | Not-before timestamp (Unix seconds). `0` means no lower bound. |
| `exp` | integer | Expiry timestamp (Unix seconds). `0` means no expiry. |
| `sig` | string | Base64-encoded ECDSA signature. See §3. |

An OpenClaim MUST NOT contain fields other than those listed above at the top level, unless a future version of this specification explicitly permits additional fields.

> **Note:** Extensions such as the payments and actions EVM extensions define their own signing formats (EIP-712) and do not use this JSON structure directly. The core claim format is used for off-chain capability grants, identity claims, and delegation chains.

## 1.3 Identifier format

The `iss` and `sub` fields identify the issuer and subject respectively.

**Key identifiers:** When the identifier refers to a cryptographic key, it MUST be a key reference in one of two forms:

1. **Inline (non-revocable):**  
   ```
   data:key/es256;base64,<base64-encoded-public-key-bytes>
   ```
   The public key bytes are the uncompressed P-256 point: a 65-byte sequence beginning with `0x04` followed by the 32-byte X coordinate and 32-byte Y coordinate, all base64-encoded (standard alphabet, no padding required, but padding is accepted).

2. **URL-based (revocable):**  
   An HTTPS URL that resolves to a JSON document. The URL form enables key revocation: a verifier that cannot fetch or parse the URL MUST reject the claim. The referenced document format is defined in §3.4.

Both forms use ES256 (ECDSA P-256 + SHA-256). Key agility (additional algorithms) is reserved for future versions.

**Arbitrary identifiers:** When the subject or issuer is not a cryptographic key (e.g., a domain name, an email address, an Ethereum address), the field MUST be a URI. For Ethereum addresses, use the CAIP-10 format: `eip155:<chainId>:<address>` with the address in EIP-55 checksum form.

## 1.4 Statement (`stm`)

The `stm` field MUST be a JSON object. Its contents are defined by the profile or extension that the claim implements. This specification makes no normative requirements on `stm` content; profile documents define those requirements.

## 1.5 Temporal constraints

Let *now* be the current time as a Unix timestamp (integer seconds since 1970-01-01T00:00:00Z UTC).

- If `nbf` is non-zero, a verifier MUST reject the claim if *now* < `nbf`.
- If `exp` is non-zero, a verifier MUST reject the claim if *now* > `exp`.
- A verifier SHOULD allow a clock skew tolerance of no more than 60 seconds when evaluating `nbf` and `exp`. Implementations that require stronger freshness guarantees SHOULD use a smaller tolerance or zero.
- A claim where `exp` ≤ `nbf` (and both are non-zero) is malformed; a verifier MUST reject it.

## 1.6 Claim creation algorithm

To create a valid OpenClaim, an issuer MUST:

1. Construct a JSON object with all required fields (`ocp`, `iss`, `sub`, `stm`, `nbf`, `exp`). Do **not** include `sig`.
2. Canonicalize the JSON object according to §2.
3. Sign the canonical bytes according to §3.
4. Encode the signature as base64 (standard alphabet; padding OPTIONAL).
5. Add the `sig` field to the JSON object with the encoded signature as its value.
6. The resulting JSON object is the OpenClaim.

The canonical form used in step 2 covers the claim **without** the `sig` field. Adding `sig` afterward does not change the canonical form.

## 1.7 Claim verification algorithm

To verify an OpenClaim, a verifier MUST:

1. Parse the input as a JSON object. If parsing fails, **reject**.
2. Check that `ocp` equals `1`. If not, **reject** (unknown version).
3. Check that all required fields (`iss`, `sub`, `stm`, `nbf`, `exp`, `sig`) are present. If any are missing, **reject**.
4. Resolve the issuer's public key from the `iss` field (see §3.4).
5. Remove the `sig` field from the object to produce the unsigned claim.
6. Canonicalize the unsigned claim according to §2.
7. Decode the `sig` field value from base64 to obtain the raw signature bytes.
8. Verify the signature against the canonical bytes using the issuer's public key according to §3.3. If verification fails, **reject**.
9. Apply temporal constraint checks from §1.5. If any check fails, **reject**.
10. Apply any additional profile-specific checks. If any fail, **reject**.
11. **Accept** the claim.

## 1.8 Multi-signer claims [informative]

The reference implementations (Q_OpenClaim.php, OpenClaim.js) support multi-signer claims where more than one party signs the same statement. In this variant:

- The `key` field is an **array** of key URI strings (lexicographically sorted) rather than a single string.
- The `sig` field is a parallel **array** of base64-encoded signatures, index-aligned with `key`.
- The `iss` field may be a semantic identity string (e.g., a domain, an Ethereum address in CAIP-10 form, or any URI identifying the issuer conceptually) rather than a cryptographic key URI. The cryptographic keys are carried in `key[]`.

Verification policy for multi-signer claims: a claim is valid if at least `minValid` of its entries in `key[]` have a valid corresponding entry in `sig[]`. The default policy is `minValid = 1` (any one valid signature suffices).

> **Status note:** The normative core format in §1.2 describes the single-signer case. Multi-signer behavior is specified informally here and in the reference implementations. A future revision may normatively specify the multi-signer wire format.

## 1.9 Immutability

A claim is immutable once signed. The `sig` field covers all other top-level fields. Any modification to `ocp`, `iss`, `sub`, `stm`, `nbf`, or `exp` after signing invalidates the signature.
