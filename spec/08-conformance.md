# §8 Conformance

This section defines conformance classes for implementations of the OpenClaiming Protocol. An implementation claims conformance with a class by meeting all MUST-level requirements listed for that class.

## 8.1 Conformance levels

### 8.1.1 Layer 1 Issuer

An implementation that creates OpenClaims for use in the base claim format.

**MUST:**
- Produce claims with all required top-level fields (`ocp`, `iss`, `sub`, `stm`, `nbf`, `exp`) and no undocumented extra fields (§1.2).
- Set `ocp` to `1` (§1.2).
- Use valid `iss` and `sub` identifier formats (§1.3).
- Sign claims using ES256 (ECDSA P-256 + SHA-256) (§3.1).
- Apply JCS canonicalization per RFC 8785 before signing, with `sig` absent from the input (§2.3, §1.6).
- Encode signatures as 64-byte raw (r||s) base64 (§3.1).
- Use a cryptographically secure random number generator for key generation and signing nonces (§3.5).

**SHOULD:**
- Use deterministic signing (RFC 6979) for secp256k1 extensions (§7.1).
- Use URL-based keys when revocability is required (§3.3.2, §7.4).

### 8.1.2 Layer 1 Verifier

An implementation that validates OpenClaims.

**MUST:**
- Reject claims with unknown `ocp` values (§1.7).
- Reject claims missing any required field (§1.7).
- Canonicalize the claim (with `sig` removed) per RFC 8785 before verifying the signature (§2.3).
- Verify signatures using ES256 (§3.2).
- Decode the `sig` field as base64 and require exactly 64 bytes (§3.2).
- Apply temporal constraints from `nbf` and `exp` (§1.5, §1.7).
- Resolve inline keys per §3.3.1.
- Resolve URL-based keys per §3.3.2, including checking the `active` field.
- Reject any claim whose issuer key URL returns a non-200 response (§3.3.2).

**SHOULD:**
- Cache resolved URL-based keys per HTTP cache-control, but invalidate cached `active: true` responses after a cache miss (§7.4).
- Apply the clock skew tolerance defined in §1.5 and §7.3.

### 8.1.3 Capability Verifier

An implementation that validates capability claims and delegation chains. Extends Layer 1 Verifier.

**MUST:**
- Verify each claim in the chain individually as a Layer 1 Verifier (§4.5).
- Verify that each link's issuer matches the previous link's subject (§4.5).
- Verify the `basis` field at each delegation link (§4.4, §4.5).
- Enforce scope reduction at each delegation link according to the capability type's rules (§4.6).
- Apply `irrevocableUntil` semantics — do not check revocation for URL-based keys during an irrevocability window (§4.3).
- Reject capability types it does not recognize (§4.2).

### 8.1.4 EVM Payments Executor

An implementation that calls the payments extension functions on the `OpenClaiming` contract.

**MUST:**
- Construct the EIP-712 domain separator per §5.3, using the correct chain ID and contract address.
- Compute the struct hash per §5.4.2 using the exact field order in PAYMENTS_TYPEHASH.
- Compute the signing digest per §5.4.3.
- Produce 65-byte secp256k1 signatures in r||s||v layout (§5.6).
- Produce low-s signatures (§5.6, §7.1).
- Compute `recipientsHash = keccak256(abi.encode(address[]))` using Solidity ABI encoding (§5.5).
- Open payment lines before attempting execution (§5.7).

**SHOULD:**
- Call `paymentsPreFlight()` before submitting a transaction to surface errors without consuming gas (§5.11).
- Supply `incomeContract = address(0)` for direct ERC-20 transfers unless routing through an IncomeContract is intentional (§5.9).

### 8.1.5 EVM Actions Executor

An implementation that calls the actions extension functions on the `OpenClaiming` contract.

**MUST:**
- Construct the EIP-712 domain separator per §6.2.
- Compute the struct hash per §6.3.2 using the exact field order in ACTIONS_TYPEHASH.
- Compute the signing digest per §6.3.3.
- Produce 65-byte secp256k1 signatures in r||s||v layout per §5.6.
- Produce low-s signatures (§7.1).
- Compute `paramsHash = keccak256(params)` using raw bytes (not ABI-encoded bytes) (§6.4).
- Include the invoker address in the signers array with a valid signature (§6.6).
- Ensure the ControlContract has `OpenClaiming` registered as `trustedForwarder` before submitting (§7.6).
- Ensure `ControlContract.generateInvokeID()` uses `_msgSender()` not `msg.sender` (§6.8).

**SHOULD:**
- Call `actionsPreFlight()` to monitor signature collection progress before submitting (§6.11).

### 8.1.6 EVM Contract Verifier

The `OpenClaiming` contract itself, as the on-chain verifier. Conformance is established by the deployed bytecode. The source code and this specification are co-normative descriptions of the same behavior.

## 8.2 Test vectors

Conformance implementations SHOULD validate against the canonicalization test vectors in §2.5.

Additional test vectors for the EIP-712 digest computation (using known inputs to verify struct hash and domain separator computations) will be published in a companion document.

## 8.3 Interoperability note [informative]

The two layers (base claims and EVM extensions) use different signing algorithms and are interoperable at the application level, not at the cryptographic primitive level. A base claim signature (ES256 / P-256) cannot be verified by the `OpenClaiming` contract, which uses secp256k1 / `ecrecover`. Implementors building systems that use both layers MUST maintain separate signing keys for each layer.
