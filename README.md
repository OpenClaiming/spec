# OpenClaiming Protocol — Specification

**Status:** Working Draft  
**Version:** 1  
**Contract:** `0x99996a51cc950d9822D68b83fE1Ad97B32Cd9999` (all supported EVM chains, immutable)  
**Repository:** [github.com/OpenClaiming/Specification](https://github.com/OpenClaiming/Specification)

---

OpenClaiming is a two-layer authorization protocol for cryptographically signed, verifiable claims.

**Layer 1 — Base claims** defines a portable, chain-agnostic signed JSON format for identity, capability grants, and permission delegation. Claims are signed with ECDSA over P-256 using the JSON Canonicalization Scheme (RFC 8785). This layer works anywhere JSON can be passed.

**Layer 2 — EVM extensions** binds payment authorization and governance-action multisig to EVM-compatible blockchains via EIP-712. The canonical verifier is the immutable `OpenClaiming` contract deployed at the address above. EVM extension signatures use secp256k1, the native Ethereum curve.

## Documents

| # | Document | Covers |
|---|---|---|
| — | [Glossary](spec/glossary.md) | Defined terms used throughout |
| 1 | [Core Claim Format](spec/01-core.md) | Fields, semantics, creation, verification |
| 2 | [Canonicalization](spec/02-canonicalization.md) | RFC 8785 JCS — normative algorithm and test vectors |
| 3 | [Signatures](spec/03-signatures.md) | Key formats, ES256 signing and verification |
| 4 | [Capability Claims](spec/04-capabilities.md) | Permission grants, delegation chains, revocation |
| 5 | [Payments Extension](spec/05-payments.md) | EVM payment authorization (EIP-712) |
| 6 | [Actions Extension](spec/06-actions.md) | EVM governance multisig (EIP-712) |
| 7 | [Security Considerations](spec/07-security.md) | Threat model and mitigations |
| 8 | [Conformance](spec/08-conformance.md) | Requirements per conformance class |

## Conformance language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL in these documents are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

Sections marked **[informative]** do not contain normative requirements. All other sections are normative unless otherwise stated.

## Versioning

This specification is versioned as an integer. The `ocp` field in every claim carries the integer version at which that claim was constructed. The current version is **1**. Implementations MUST reject claims with an `ocp` value they do not recognize.

## Contract immutability

The EVM extensions are defined by the deployed bytecode of the `OpenClaiming` contract, which is not upgradeable. This specification is the human-readable normative description of that bytecode. In the event of a discrepancy, the deployed bytecode is authoritative for the EVM extension behavior.
