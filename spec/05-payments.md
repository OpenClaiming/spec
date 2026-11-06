# §5 Payments Extension (EVM)

## 5.1 Overview [informative]

The payments extension enables a signer to authorize token transfers on EVM-compatible chains without granting standing access to their wallet. The signer commits to a bounded authorization (payer, token, recipients, ceiling, time window) off-chain. An executor — which may be an AI agent, a relayer, or any smart contract caller — presents the signed authorization and a chosen recipient to the `OpenClaiming` contract, which verifies the signature on-chain and executes the transfer if all constraints pass.

The extension uses EIP-712 typed structured data hashing so wallets (MetaMask, Ledger, etc.) can display the authorization fields in a human-readable form before the user signs.

## 5.2 Contract deployment

The canonical verifier and executor for the payments extension is the `OpenClaiming` contract, deployed immutably at:

```
0x99996a51cc950d9822D68b83fE1Ad97B32Cd9999
```

on all supported EVM chains. The behavior defined in this section is normative and matches the deployed bytecode.

## 5.3 EIP-712 domain

The payments extension uses the following EIP-712 domain:

```
name:              "OpenClaiming.payments"
version:           "1"
chainId:           <the chain's EIP-155 chain ID>
verifyingContract: 0x99996a51cc950d9822D68b83fE1Ad97B32Cd9999
```

The domain separator is:

```
keccak256(
    abi.encode(
        keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
        keccak256("OpenClaiming.payments"),
        keccak256("1"),
        block.chainid,
        address(this)        // the OpenClaiming contract address
    )
)
```

The domain separator is computed at verification time using the current chain ID and the contract's own address. It is not a constant.

## 5.4 Payment struct

A payment authorization is represented as the following typed struct:

```
Payment {
    address payer          // whose tokens will be transferred
    address token          // ERC-20 contract address; address(0) = native coin
    bytes32 recipientsHash // keccak256(abi.encode(address[])) of the authorized recipient set
    uint256 max            // claim-level spending ceiling; 0 = unlimited
    uint256 line           // payment line bucket ID
    uint256 nbf            // not-before timestamp (Unix seconds); 0 = no lower bound
    uint256 exp            // expiry timestamp (Unix seconds); 0 = no expiry
}
```

### 5.4.1 Type hash

```
PAYMENTS_TYPEHASH = keccak256(
    "Payment(address payer,address token,bytes32 recipientsHash,uint256 max,uint256 line,uint256 nbf,uint256 exp)"
)
```

This string MUST be reproduced exactly (including field order) when computing the type hash.

### 5.4.2 Struct hash

```
structHash = keccak256(
    abi.encode(
        PAYMENTS_TYPEHASH,
        p.payer,
        p.token,
        p.recipientsHash,
        p.max,
        p.line,
        p.nbf,
        p.exp
    )
)
```

### 5.4.3 Signing digest

```
digest = keccak256(
    abi.encodePacked(
        bytes2(0x1901),
        domainSeparator,
        structHash
    )
)
```

This is the standard EIP-712 digest construction. `0x1901` is the two-byte prefix `\x19\x01`.

## 5.5 Recipients hash

The `recipientsHash` commits the payer to an authorized set of recipient addresses without storing that set on-chain.

```
recipientsHash = keccak256(abi.encode(recipients))
```

where `recipients` is an `address[]` array. `abi.encode` on a dynamic array encodes the array offset, length, and padded elements as defined by the Solidity ABI specification.

An executor MUST supply the plaintext `recipients` array alongside the signature when calling the contract. The contract recomputes the hash and rejects any call where the supplied array does not hash to `p.recipientsHash`.

## 5.6 Signature format

Payments extension signatures use secp256k1 and are exactly 65 bytes in the following layout:

```
sig[0:32]   r   (32 bytes, big-endian)
sig[32:64]  s   (32 bytes, big-endian)
sig[64]     v   (1 byte)
```

Signature validation MUST proceed as follows:

1. If `len(sig) ≠ 65`, **reject** (`InvalidSignatureLength`).
2. Extract r, s, v from `sig`.
3. If v ∈ {0, 1}, normalize: v += 27.
4. If v ∉ {27, 28}, **reject** (`InvalidSignatureV`).
5. If `uint256(s) > SECP256K1N_OVER_2`, **reject** (`InvalidSignatureS`).  
   `SECP256K1N_OVER_2 = 0x7fffffffffffffffffffffffffffffff5d576e7357a4501ddfe92f46681b20a0`
6. Compute `signer = ecrecover(digest, v, r, s)`.
7. If `signer == address(0)`, **reject** (`InvalidSignature`).
8. For single-signer flows, if `signer ≠ p.payer`, **reject** (`PayerMismatch`).

The low-s check in step 5 prevents signature malleability (EIP-2). Implementations that produce signatures SHOULD use a deterministic signing algorithm (RFC 6979) and SHOULD produce low-s signatures. Implementations that accept signatures MUST reject high-s signatures.

## 5.7 Line accounting

The `OpenClaiming` contract maintains a spending ledger:

```
lines[payer][lineId] → { max: uint256, spent: uint256, open: bool }
```

Before any payment claim can execute against a line, the payer (or the payer's `Ownable.owner()`) MUST open the line by calling `lineOpen(account, lineId, max)`.

**Spending capacity** is computed as:

```
claimRemaining = (p.max == 0) ? UINT256_MAX : max(0, p.max - line.spent)
lineRemaining  = (line.max == 0) ? UINT256_MAX : max(0, line.max - line.spent)
available      = min(claimRemaining, lineRemaining)
```

An execution attempt for `amount` MUST be rejected if `amount > available`.

**Update rule:** When a payment executes for `amount`:

```
lines[payer][lineId].spent += amount
```

This update MUST occur before the token transfer (checks-effects-interactions pattern). The line's `spent` total is never reset by the contract; only the payer can reset it by closing and reopening the line.

### 5.7.1 Line operations

| Function | Caller authorization | Effect |
|---|---|---|
| `lineOpen(account, line, max)` | `account` or `account.owner()` | Opens the line; sets or updates `max`. Does not reset `spent`. |
| `lineClose(account, line)` | `account` or `account.owner()` | Closes the line; `spent` is preserved. |
| `lineAvailable(account, line, claimMax)` | Anyone (view) | Returns available capacity, or 0 if line is closed. |

## 5.8 Execution

### 5.8.1 Single-signer execution

`paymentsExecute(p, recipients, sig, recipient, amount, incomeContract)`:

1. Validate signature: `ecrecover(digest, sig) == p.payer` (§5.6).
2. Validate temporal constraints from p.nbf and p.exp (§1.5 semantics).
3. Validate `keccak256(abi.encode(recipients)) == p.recipientsHash`.
4. Validate `recipient ∈ recipients`.
5. Validate line is open: `lines[p.payer][p.line].open == true`.
6. Validate `amount ≤ available` (§5.7).
7. Update `lines[p.payer][p.line].spent += amount`.
8. Transfer tokens (§5.9).
9. Emit `PaymentsExecuted(p.payer, p.token, recipient, p.line, amount, newSpent)`.

### 5.8.2 Multi-signer execution

`paymentsExecuteSignatures(p, recipients, recipient, amount, signers[], signatures[], minValid, incomeContract)`:

Steps 2–7 from §5.8.1 apply. Step 1 is replaced with:

1. For each `i` in `signers[]`:
   - Skip if `signers[i] == address(0)` or `signatures[i].length == 0`.
   - Skip if `signers[i]` was already counted (deduplication by address).
   - Attempt `ecrecover(digest, signatures[i])`. If it throws or returns `address(0)`, skip.
   - If the recovered address equals `signers[i]`, count this as a valid signature.
2. If the count of unique valid signatures < `minValid`, **reject** (`InvalidSignature`).

## 5.9 Token transfer

Two transfer paths exist:

**Direct ERC-20 (`incomeContract == address(0)`):**  
The contract calls `token.transferFrom(p.payer, recipient, amount)`. The payer MUST have approved the `OpenClaiming` contract for at least `amount` of `token`.

**IncomeContract (`incomeContract != address(0)`):**  
The contract calls `incomeContract.pay(recipient, amount)` forwarded as `p.payer` via EIP-2771 (§5.10). The payer MUST be registered as a manager on the `IncomeContract`.

**Native coin (`p.token == address(0)`):**  
`msg.sender` MUST equal `p.payer` (native coin cannot be delegated to a relayer). `msg.value` MUST equal `amount`. The contract sends the native coin to `recipient` via `call{value: amount}("")`.

## 5.10 EIP-2771 forwarding

When the contract makes a forwarded call on behalf of a signer:

```
target.call(abi.encodePacked(calldata, signer))
```

The signer's address is appended as the last 20 bytes of calldata. The target contract's `_msgSender()` MUST implement EIP-2771: it strips the last 20 bytes of calldata and returns that address as the message sender. The `OpenClaiming` contract MUST be registered as a `trustedForwarder` on any target contract it interacts with via this mechanism.

## 5.11 Pre-flight validation

`paymentsPreFlight(p, recipients, recipient, amount, signers[], signatures[], minValid)` is a view function that returns a `PreflightResult` struct:

```
PreflightResult {
    bool    valid           // true if all checks pass
    string  extension       // "payments"
    bytes32 digest          // the EIP-712 digest
    uint256 validSigCount   // unique valid signatures found
    bool    notYetValid     // true if now < p.nbf
    bool    expired         // true if now > p.exp
    bool    lineOpen        // true if line is open
    bool    capacityOk      // true if available >= amount
}
```

This function does NOT modify state. It MUST NOT be used as a substitute for execution-time validation.

## 5.12 Error conditions

| Error | Condition |
|---|---|
| `InvalidSignature` | ecrecover returned address(0) or recovered address does not match |
| `InvalidSignatureLength` | Signature is not exactly 65 bytes |
| `InvalidSignatureV` | v byte is not 27 or 28 after normalization |
| `InvalidSignatureS` | s > SECP256K1N_OVER_2 |
| `NotYetValid(nbf)` | block.timestamp < p.nbf |
| `Expired(exp)` | block.timestamp > p.exp |
| `LineNotOpen(account, line)` | Line is not open |
| `PaymentRecipientsHashMismatch` | keccak256(abi.encode(recipients)) ≠ p.recipientsHash |
| `InvalidRecipient(recipient)` | recipient not found in recipients array |
| `ClaimMaxExceeded(requested, available)` | amount > claimRemaining |
| `LineMaxExceeded(requested, available)` | amount > lineRemaining |
| `InsufficientCapacity(requested, available)` | amount > min(claimRemaining, lineRemaining) |
| `PayerMismatch(expected, actual)` | Recovered signer ≠ p.payer |
| `NativeCoinDelegationUnsupported` | p.token == address(0) and msg.sender ≠ p.payer |
| `NativeCoinValueMismatch(expected, actual)` | msg.value ≠ amount for native coin, or msg.value ≠ 0 for ERC-20 |
| `TransferFailed` | ERC-20 transferFrom or native coin transfer returned failure |
| `UnauthorizedLineOperator(account, caller)` | caller is not account and not account.owner() |
