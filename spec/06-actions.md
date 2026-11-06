# §6 Actions Extension (EVM)

## 6.1 Overview [informative]

The actions extension enables a group of signers to authorize a governance operation on a `ControlContract` without requiring each signer to submit an individual on-chain transaction. All signers sign the same EIP-712 `Action` struct off-chain. A relayer (or Safebox agent) collects the signatures and submits them in a single transaction. The `OpenClaiming` contract verifies all signatures, then forwards the operation to the `ControlContract` as each valid signer.

The pattern is composable with existing governance contracts that implement the invoke/endorse/execute quorum flow.

## 6.2 EIP-712 domain

```
name:              "OpenClaiming.actions"
version:           "1"
chainId:           <the chain's EIP-155 chain ID>
verifyingContract: 0x99996a51cc950d9822D68b83fE1Ad97B32Cd9999
```

Domain separator:

```
keccak256(
    abi.encode(
        keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
        keccak256("OpenClaiming.actions"),
        keccak256("1"),
        block.chainid,
        address(this)
    )
)
```

## 6.3 Action struct

```
Action {
    address authority        // semantic authority behind the claim (iss)
    address subject          // the ControlContract to invoke and endorse
    address contractAddress  // target contract ControlContract will ultimately call
    bytes4  method           // 4-byte ABI selector of the target method
    bytes32 paramsHash       // keccak256(params) where params is ABI-encoded calldata arguments
    uint256 minimum          // minimum number of endorsements for quorum
    uint256 fraction         // fractional quorum threshold out of 1e10 (e.g. 5000000000 = 50%)
    uint256 delay            // seconds that must elapse after quorum before execution; 0 = immediate
    uint256 nbf              // not-before timestamp (Unix seconds); 0 = no lower bound
    uint256 exp              // expiry timestamp (Unix seconds); 0 = no expiry
}
```

### 6.3.1 Type hash

```
ACTIONS_TYPEHASH = keccak256(
    "Action(address authority,address subject,address contractAddress,bytes4 method,bytes32 paramsHash,uint256 minimum,uint256 fraction,uint256 delay,uint256 nbf,uint256 exp)"
)
```

This string MUST be reproduced exactly (including field order).

### 6.3.2 Struct hash

```
structHash = keccak256(
    abi.encode(
        ACTIONS_TYPEHASH,
        a.authority,
        a.subject,
        a.contractAddress,
        a.method,
        a.paramsHash,
        a.minimum,
        a.fraction,
        a.delay,
        a.nbf,
        a.exp
    )
)
```

### 6.3.3 Signing digest

```
digest = keccak256(
    abi.encodePacked(
        bytes2(0x1901),
        domainSeparator,
        structHash
    )
)
```

All signers sign the same digest. The digest does not change based on the number of signers. Each signer independently signs it and produces an independent signature.

## 6.4 Params hash

The `paramsHash` commits all signers to exact calldata without storing the params on-chain:

```
paramsHash = keccak256(params)
```

where `params` is a raw `bytes` value — the ABI-encoded calldata arguments to the target method (without the method selector). The raw `keccak256` of the bytes is used directly (not `keccak256(abi.encode(bytes))` — there is no extra wrapping).

The executor MUST supply the plaintext `params` bytes alongside signatures. The contract recomputes `keccak256(params)` and rejects any call where the supplied bytes do not hash to `a.paramsHash`.

## 6.5 Signature format

Actions extension signatures use the same format as payments extension signatures (§5.6): 65 bytes, layout r||s||v, with the same validation rules including low-s enforcement.

The key difference in the multisig flow: each `signatures[i]` MUST correspond to `signers[i]`. If `ecrecover(digest, signatures[i])` does not equal `signers[i]`, **reject** immediately with `InvalidSignature` (the multi-signer execute path is strict; it does not silently skip invalid signatures as the view/preflight functions do).

## 6.6 Invoker

One signer is designated as the **invoker**: the address that calls `ControlContract.invoke()`. The invoker MUST hold the invoke role in the `ControlContract`'s associated `Community` contract.

The invoker MUST appear in the `signers[]` array with a valid signature. If the invoker is not found among the verified signers, the contract **rejects** with `InvokerNotInSigners`.

## 6.7 Execution flow

`actionsExecute(a, params, signers[], signatures[], minValid, invoker)`:

1. Validate temporal constraints from a.nbf and a.exp.
2. Validate `keccak256(params) == a.paramsHash`. If not, **reject** (`ParamsHashMismatch`).
3. Validate `len(signers) > 0 && len(signers) == len(signatures)`. If not, **reject** (`InvalidSignerCount`).
4. For each `i` in `signers[]`:
   - Skip if `signers[i] == address(0)` or `len(signatures[i]) == 0`.
   - Compute `recovered = ecrecover(digest, signatures[i])`. If it throws or returns `address(0)`, **reject** (`InvalidSignature`).
   - If `recovered ≠ signers[i]`, **reject** (`InvalidSignature`).
   - If `signers[i]` is not already in the valid set, add it.
5. If `len(validSet) < minValid`, **reject** (`InvalidSignature`).
6. If `invoker ∉ validSet`, **reject** (`InvokerNotInSigners`).
7. Call `a.subject.invoke(a.contractAddress, hexString(a.method), hexString(params), a.minimum, a.fraction, uint64(a.delay))` forwarded as `invoker` via EIP-2771 (§6.8).
8. Re-derive `invokeID = keccak256(abi.encodePacked(block.timestamp, block.prevrandao, invoker))`.
9. For each signer `s` in `validSet`, call `a.subject.endorse(invokeID)` forwarded as `s` via EIP-2771.
10. Emit `ActionsExecuted(a.authority, a.subject, a.contractAddress, a.method, invokeID)`.

### 6.7.1 Method and params encoding for ControlContract

The `ControlContract.invoke()` function accepts the method selector and params as hex-encoded strings (not raw bytes). The `OpenClaiming` contract converts:
- `a.method` (bytes4) → lowercase hex string without `0x` prefix, e.g. `a9059cbb`
- `params` (bytes) → lowercase hex string without `0x` prefix

Target ControlContract implementations MUST accept these hex-encoded string arguments and reconstruct the bytes internally.

## 6.8 EIP-2771 forwarding for actions

Identical to §5.10. For each forwarded call, the caller's address is appended as the last 20 bytes of calldata:

```
target.call(abi.encodePacked(calldata, signer))
```

This means:
- `invoke()` is called with the invoker's address appended → `ControlContract._msgSender()` returns `invoker`.
- Each `endorse()` is called with that endorser's address appended → `ControlContract._msgSender()` returns that endorser.

**Required change to ControlContract:** `generateInvokeID()` MUST use `_msgSender()` (not `msg.sender`) so the derived `invokeID` is based on the actual human signer (the invoker) rather than the `OpenClaiming` contract address. Without this change, the invokeID re-derivation in step 8 of §6.7 will not match.

## 6.9 InvokeID re-derivation

After calling `invoke()`, the `OpenClaiming` contract re-derives the invokeID using the same formula the `ControlContract` uses internally:

```
invokeID = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, invoker)))
```

This works because `ControlContract.generateInvokeID()` uses `_msgSender()` (the invoker, via EIP-2771) rather than `msg.sender`. Both sides compute the same hash from the same inputs in the same transaction.

## 6.10 Multisig collection [informative]

Signers may sign independently and in any order. A relayer or agent collects signatures asynchronously:

1. Construct the `Action` struct and compute the EIP-712 digest.
2. Request signatures from each signer. Each signer independently signs the same digest.
3. As each signature arrives, optionally call `actionsPreFlight()` (§6.11) to check whether `validSigCount` has reached `minValid`.
4. Once `minValid` signatures are collected, submit `actionsExecute()`.

No coordination between signers is required beyond agreeing on the `Action` struct before signing.

## 6.11 Pre-flight validation

`actionsPreFlight(a, params, signers[], signatures[], minValid, invoker)` returns a `PreflightResult` struct:

```
PreflightResult {
    bool    valid           // true if all checks pass
    string  extension       // "actions"
    bytes32 digest          // the EIP-712 digest
    uint256 validSigCount   // unique valid signatures found
    bool    notYetValid     // true if now < a.nbf
    bool    expired         // true if now > a.exp
    bool    lineOpen        // false (not applicable for actions)
    bool    capacityOk      // false (not applicable for actions)
}
```

The preflight function does NOT modify state and does NOT revert on individual bad signatures. It is safe to call repeatedly as signatures are collected. `valid` is true when: not notYetValid, not expired, paramsHash matches, invoker is among valid signers, and validSigCount ≥ minValid.

## 6.12 Error conditions

| Error | Condition |
|---|---|
| `InvalidSignature` | Signature fails validation or recovered address ≠ signer |
| `InvalidSignatureLength` | Signature is not exactly 65 bytes |
| `InvalidSignatureV` | v byte is not 27 or 28 after normalization |
| `InvalidSignatureS` | s > SECP256K1N_OVER_2 |
| `NotYetValid(nbf)` | block.timestamp < a.nbf |
| `Expired(exp)` | block.timestamp > a.exp |
| `InvokerNotInSigners` | invoker not found among valid unique signers |
| `InvalidSignerCount` | signers[] is empty, or len(signers) ≠ len(signatures) |
| `ParamsHashMismatch` | keccak256(params) ≠ a.paramsHash |

## 6.13 Fraction field

`fraction` is an integer denominator out of 1×10¹⁰. Examples:

| `fraction` value | Meaning |
|---|---|
| `0` | Fraction threshold disabled (use `minimum` only) |
| `3000000000` | 30% of eligible endorsers |
| `5000000000` | 50% |
| `10000000000` | 100% (unanimous) |

The `ControlContract` combines `minimum` and `fraction` to determine quorum. Meeting either threshold (the lower of the two) is NOT sufficient; both apply independently. The semantics of quorum computation are defined by the `ControlContract`, not by the `OpenClaiming` contract.
