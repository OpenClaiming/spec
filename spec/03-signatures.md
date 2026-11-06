# §3 Signatures

This section defines Layer 1 signatures for the base OpenClaim format. EVM extension signatures (secp256k1 / EIP-712) are defined in §5 and §6.

## 3.1 Algorithm

Layer 1 claims MUST be signed using **ES256**: ECDSA with P-256 (secp256r1) and SHA-256.

The signing process for a claim is:

1. Produce the canonical byte sequence `M` by applying §2 to the claim with `sig` removed.
2. Compute the digest `H = SHA-256(M)`.
3. Sign `H` with the issuer's P-256 private key using ECDSA, producing the integer pair (r, s).
4. Produce a signature byte sequence. Two formats are in use:
   - **DER** (ASN.1 SEQUENCE of r and s integers) — produced by `openssl_sign()` in PHP and by Node.js `crypto.sign("sha256", data, ecKey)`.
   - **IEEE P1363 / raw** (64 bytes: r||s, each 32-byte big-endian) — produced by `SubtleCrypto.sign({name:"ECDSA",hash:"SHA-256"}, key, data)` in browsers and Node 18+.
   Both formats are accepted by the reference implementations. Implementors sharing signatures across runtimes MUST agree on a common format; IEEE P1363 is preferred for new implementations because it has a fixed 64-byte length and is natively supported by SubtleCrypto.
5. Base64-encode the signature bytes (standard alphabet; padding OPTIONAL) to produce the `sig` field value.

> **Compatibility note:** `crypto.sign` in Node.js produces DER for EC keys by default; `subtle.sign` always produces IEEE P1363. Cross-platform implementations MUST handle both formats when verifying.

## 3.2 Signature verification

To verify a Layer 1 claim signature:

1. Produce `M` by applying §2 to the claim with `sig` removed.
2. Compute `H = SHA-256(M)`.
3. Decode the `sig` field from base64 to obtain `sigBytes`. If decoding fails, **reject**.
4. Determine the signature format:
   - If `sigBytes` is exactly 64 bytes: treat as IEEE P1363 (r||s).
   - If `sigBytes` starts with `0x30` (DER SEQUENCE tag): treat as DER-encoded ECDSA signature.
   - Otherwise, **reject**.
5. Verify the ECDSA signature over the canonical bytes using the issuer's P-256 public key. Use the appropriate verification primitive for the detected format. If verification fails, **reject**.

Low-s enforcement is NOT required for Layer 1 signatures (P-256 signatures are not subject to the malleability concerns that motivate low-s enforcement in secp256k1).

## 3.3 Key format

### 3.3.1 Inline key (non-revocable)

An inline key is embedded in a key-bearing field as a data URI:

```
data:key/es256;base64,<pubkey>
```

`<pubkey>` is the base64 encoding (standard alphabet, padding OPTIONAL) of the **SPKI DER encoding** of the P-256 public key. For P-256, the SPKI structure is:

```
SEQUENCE {
  SEQUENCE { OID ecPublicKey, OID prime256v1 }
  BIT STRING { 0x00, 0x04, X(32 bytes), Y(32 bytes) }
}
```

The resulting DER is 91 bytes. As a construction shortcut, implementations MAY assemble SPKI from the raw 65-byte uncompressed point (04||X||Y) by prepending the fixed 27-byte P-256 SPKI prefix:

```
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE
```

(base64 of the prefix up to and including the leading `0x04` of the point; then append X||Y, 64 bytes).

This is the format produced by `SubtleCrypto.exportKey("spki", key)`, by Node.js `createPublicKey(pem).export({type:"spki",format:"der"})`, and by OpenSSL `openssl_pkey_get_details()['key']` after PEM stripping.

Inline keys are non-revocable: once a claim is signed with an inline key, no mechanism exists to invalidate it other than expiry via the `exp` field.

### 3.3.2 URL-based key (revocable)

A URL-based key is specified as an HTTPS URL in the `iss` or `sub` field:

```
https://example.com/.well-known/ocp-key.json
```

The URL MUST use the HTTPS scheme. The URL MUST NOT use a data URI. When resolving a URL-based key, an implementation MUST:

1. Perform an HTTPS GET request to the URL.
2. If the response status is not 200, **reject** the claim (key is considered revoked or unavailable).
3. Parse the response body as a JSON object.
4. The object MUST have an `active` field set to `true`. If `active` is `false` or absent, **reject** (key revoked).
5. The object MUST have a `key` field containing an inline key string in the `data:key/es256;base64,...` format defined in §3.3.1.
6. Use the key from the `key` field for signature verification.

**Key document format:**

```json
{
  "ocp": 1,
  "active": true,
  "key": "data:key/es256;base64,<pubkey>"
}
```

Implementations that perform claim verification SHOULD cache resolved keys with a TTL derived from HTTP cache-control headers, but MUST NOT cache a key with `active: false`.

Implementations that cannot perform HTTPS requests (e.g., air-gapped environments) MUST reject any claim whose `iss` is a URL unless the key is provided through an out-of-band trust store.

## 3.4 Key resolution

Given an `iss` or `sub` value, a verifier resolves the public key as follows:

1. If the value begins with `data:key/es256;base64,`: parse as an inline key (§3.3.1).
2. If the value begins with `https://`: resolve as a URL-based key (§3.3.2).
3. Otherwise, the verifier MUST either:
   - Reject the claim (strict mode), or
   - Resolve the key via an application-specific trust store (permissive mode).

Verifiers implementing strict mode MUST reject any `iss` or `sub` value that is not in one of the two recognized forms.

## 3.5 Key generation

Implementations generating ES256 keypairs MUST use a cryptographically secure random number generator. P-256 key generation MUST follow FIPS 186-5 or an equivalent standard. Keys MUST be at least 256 bits of effective security (P-256 provides this by construction).

Private keys MUST NOT be transmitted over any channel or logged. When stored, private keys SHOULD be protected by hardware (HSM, TEE, or platform secure enclave).
