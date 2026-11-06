# Glossary

This glossary defines terms used throughout the OpenClaiming specification. Where a term is used before it is defined in a document, its definition here applies.

---

**Canonical form**  
The unique byte representation of a JSON document produced by the JSON Canonicalization Scheme (RFC 8785). Used as the input to signing and verification.

**Capability claim**  
An OpenClaim whose `stm` asserts a permission or delegated capability. The issuer grants the capability to the subject.

**Chain (delegation chain)**  
A sequence of OpenClaims where each claim's issuer was itself the subject of a capability grant from the previous claim. Authority flows from a trusted root through the chain to the leaf claim.

**Claim**  
A signed JSON document conforming to the OpenClaim format defined in §1. Sometimes called an *OpenClaim* to distinguish it from other credential formats.

**Digest**  
A fixed-length hash of a message, used as the input to a signature algorithm. In Layer 1 (base claims), the digest is a SHA-256 hash of the JCS-canonical claim bytes (with `sig` removed). In Layer 2 (EVM extensions), the digest is an EIP-712 typed-data hash.

**Domain separator**  
In EIP-712, a hash that encodes the name, version, chain ID, and verifying contract address of a signing domain. Prevents replay of signatures across domains or chains.

**EIP-712**  
Ethereum Improvement Proposal 712 — Typed structured data hashing and signing. Defines how structured data is hashed for signing such that wallets can display human-readable fields before the user signs.

**EIP-2771**  
Meta-transaction standard. The `OpenClaiming` contract uses EIP-2771 to forward calls to `ControlContract` and `IncomeContract` with the original human signer's address appended to calldata, so target contracts' `_msgSender()` returns the signer rather than `OpenClaiming`.

**ES256**  
The JSON Web Algorithm identifier for ECDSA with P-256 and SHA-256. The algorithm used for Layer 1 (base claim) signatures.

**exp**  
Expiry. A Unix timestamp (seconds since 1970-01-01T00:00:00Z). A claim with a non-zero `exp` MUST be rejected if the current time is after `exp`.

**Issuer**  
The entity that signs an OpenClaim and thereby asserts the statement. Identified by the `iss` field.

**JCS**  
JSON Canonicalization Scheme. RFC 8785. Produces a deterministic byte representation of a JSON document. Required before hashing any Layer 1 claim.

**Line**  
A named payment bucket tracked by the `OpenClaiming` contract. Each line has a maximum spending ceiling (`max`) and a running total (`spent`). Lines are addressed by `(payer, lineId)`.

**nbf**  
Not Before. A Unix timestamp. A claim with a non-zero `nbf` MUST be rejected if the current time is before `nbf`.

**OpenClaim**  
See *Claim*.

**P-256**  
The NIST elliptic curve over a 256-bit prime field, also known as secp256r1 or prime256v1. Used for Layer 1 signatures.

**Payer**  
In the payments extension, the Ethereum address whose tokens are transferred when a payment claim executes. Must have opened a line before execution.

**secp256k1**  
The elliptic curve used by Ethereum. Used for Layer 2 (EVM extension) signatures.

**sig**  
The `sig` field of an OpenClaim containing the base64-encoded ECDSA signature. MUST be excluded from the claim JSON before canonicalization and signing.

**stm**  
Statement. The payload of an OpenClaim — a JSON object whose schema is defined by the type of claim being made.

**Subject**  
The entity the claim refers to. Identified by the `sub` field. May be the same as the issuer.

**Trusted root**  
In a delegation chain, the first issuer whose key is accepted as authoritative by the verifier without further verification.

**Verifier**  
An implementation that checks whether a claim or chain of claims is valid according to this specification.
