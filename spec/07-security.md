# §7 Security Considerations

## 7.1 Signature malleability (EVM extensions)

The secp256k1 curve admits two valid signatures (r, s) and (r, n−s) for any message, where n is the curve order. Without mitigation, an adversary could modify a valid signature into an equally-valid one, breaking assumptions about signature uniqueness.

The `OpenClaiming` contract enforces the **low-s** constraint (EIP-2): any signature with `s > SECP256K1N_OVER_2` is rejected. This ensures each message has at most one accepted signature form.

Implementors producing signatures for the EVM extensions SHOULD use a deterministic signing algorithm (RFC 6979) and MUST produce a low-s signature (negate s if s > SECP256K1N_OVER_2 and add 1 to v — standard secp256k1 practice).

## 7.2 Replay attacks

### 7.2.1 Cross-chain replay (EVM)

The EIP-712 domain separator includes `chainId` and `verifyingContract`. A signature produced on chain A is not valid on chain B because the domain separator differs. Implementors MUST NOT reuse a signature across chains.

### 7.2.2 Cross-extension replay

The domain separator also encodes the extension name (`"OpenClaiming.payments"` vs `"OpenClaiming.actions"`). A Payment signature cannot be presented to an actions-related function and vice versa.

### 7.2.3 Claim replay (Layer 1)

The base OpenClaim format does not include a nonce. Replay within the claim's validity window is possible if a verifier does not maintain seen-claim state. Verifiers that need to prevent replay MUST maintain a set of accepted claim hashes and reject any claim whose canonical hash has been seen before.

Applications where replay within the validity window is acceptable (e.g., a bearer token that can be presented multiple times) need not track claim hashes.

## 7.3 Clock skew

Temporal constraints (`nbf`, `exp`) depend on accurate clocks. Implementations SHOULD allow a tolerance of no more than 60 seconds to accommodate minor clock drift between issuer and verifier. For EVM extensions, `block.timestamp` is used by the contract and is controlled by the block producer; it may be manipulated by up to ~15 seconds in practice on most EVM networks.

Issuers SHOULD set `nbf` at least 60 seconds in the past relative to expected verification time, and `exp` at least 60 seconds beyond the last intended use, to absorb normal clock skew.

## 7.4 Key revocation

**Inline keys** cannot be revoked by mechanism. Once a claim signed by an inline key is issued, the only way to invalidate it before its `exp` is for the verifier to maintain an application-level revocation list. Authors who require revocability MUST use URL-based keys.

**URL-based keys** are revocable by updating the key document at the URL to `active: false`. Verifiers MUST check the `active` field on each verification (possibly using HTTP caching for performance, but MUST NOT use a stale `active: true` after a cache miss on a fresh request returns `active: false`).

**irrevocableUntil** grants temporarily inhibit revocation. Verifiers MUST NOT check URL-based key revocation until `now ≥ irrevocableUntil`. If an issuer cannot revoke before that time, they accept the full risk of a compromised key during that window.

## 7.5 Delegation scope creep

A delegation that claims more permissions than the parent granted is a security violation. Verifiers MUST validate scope reduction at each link in a delegation chain (§4.6). An implementation that skips scope reduction checks could be presented with a chain where the leaf claims vastly greater permissions than the root granted.

## 7.6 ControlContract trust (actions extension)

The `OpenClaiming` contract must be registered as a `trustedForwarder` on any `ControlContract` or `IncomeContract` it interacts with. Any contract that trusts `OpenClaiming` as a forwarder thereby accepts that `OpenClaiming` can call it on behalf of any address, subject to the conditions the `OpenClaiming` contract enforces. Deployers MUST audit `OpenClaiming`'s enforcement logic before registering it as a trusted forwarder.

## 7.7 Front-running (EVM)

Payment and action submissions are publicly visible in the mempool. An adversary could:
- Observe a submission and copy it in a competing transaction (front-run). For payments, this is mitigated by the payer's signature constraining the recipient and amount.
- Observe a partial signature collection for an action and attempt to execute early. This is prevented by the `minValid` threshold; the contract rejects submissions below quorum.

## 7.8 Line exhaustion

A malicious relayer could exhaust a payment line by submitting many small transactions quickly. Authors SHOULD set appropriate `max` values on lines and claims. Rate limiting at the application layer (before submitting to the contract) is advisable for high-value lines.

## 7.9 P-256 security (Layer 1)

P-256 provides approximately 128 bits of security. This is considered adequate for the credential lifetimes typical in OpenClaim use cases (minutes to years). Implementations MUST use a cryptographically secure random source for key generation and nonce generation during signing.

## 7.10 Key document HTTPS security

URL-based key documents MUST be served over HTTPS with a valid certificate. An attacker who can intercept the HTTP response could serve a modified key document and thereby cause a verifier to accept a signature from an unauthorized key. Certificate pinning is RECOMMENDED for high-security deployments.

## 7.11 Parameter encoding for ControlContract

The `OpenClaiming` contract converts `method` (bytes4) and `params` (bytes) to lowercase hex strings before passing them to `ControlContract.invoke()`. This encoding is deterministic and reversible. `ControlContract` implementations MUST use the same encoding when reconstructing calldata from these strings, or the target call will execute with incorrect data.
