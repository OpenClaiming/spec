# §2 Canonicalization

## 2.1 Overview

Before a Layer 1 claim can be signed or verified, its JSON must be transformed into a canonical byte sequence. OpenClaiming uses the **JSON Canonicalization Scheme (JCS)** defined in [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785). JCS eliminates representational ambiguity so that two parties produce identical bytes from the same logical document.

## 2.2 Input and output

**Input:** A JSON object from which the `sig` field has been removed (or was never present).

**Output:** A UTF-8 byte sequence with no byte-order mark.

## 2.3 Normative algorithm

Implementations MUST produce canonical output according to the following rules. These rules are drawn from RFC 8785 §3.

### 2.3.1 Primitive values

| JSON type | Canonical representation |
|---|---|
| `null` | `null` |
| `true` | `true` |
| `false` | `false` |
| Number | See §2.3.2 |
| String | See §2.3.3 |

### 2.3.2 Number serialization

Numbers MUST be serialized according to the IEEE 754 double-precision rules of RFC 8785 §3.2.2.3:

- Integers in the range [−2⁵³, 2⁵³] MUST be represented without a decimal point or exponent: e.g., `0`, `42`, `-7`.
- Numbers outside that range, or numbers with fractional parts, MUST be represented in a form that round-trips through IEEE 754 double: use the shortest decimal representation that uniquely identifies the value.
- `NaN` and `Infinity` are not valid JSON and MUST NOT appear.
- Negative zero (`-0`) MUST be serialized as `0`.

### 2.3.3 String serialization

Strings MUST be enclosed in double quotation marks (`"`). Within the string:

- U+0022 (`"`) MUST be escaped as `\"`
- U+005C (`\`) MUST be escaped as `\\`
- U+0000–U+001F (control characters) MUST be escaped as `\uXXXX` using four uppercase hex digits
- U+007F–U+FFFF MAY be represented as literal UTF-8 or as `\uXXXX` escapes; both are valid JCS
- Surrogate pairs (U+D800–U+DFFF) MUST be encoded as `\uXXXX\uXXXX` pairs

### 2.3.4 Array serialization

An array is serialized as `[` followed by each element serialized according to these rules, separated by `,`, followed by `]`. The order of elements MUST be preserved.

### 2.3.5 Object serialization

An object is serialized as `{` followed by its key-value pairs, followed by `}`. Key-value pairs MUST be ordered lexicographically by their key's Unicode code-point sequence (i.e., by the UTF-16 code units of the key string, as specified in RFC 8785 §3.2.3). Each pair is `"<key>":<value>`, separated from the next pair by `,`. There is no whitespace between any tokens.

### 2.3.6 Whitespace

No whitespace (spaces, tabs, newlines, carriage returns) MAY appear between any tokens in the canonical output. Whitespace inside string values is preserved verbatim.

## 2.4 Signing and verification use

When creating a claim:
1. Construct the JSON object with all fields **except** `sig`.
2. Apply the canonicalization algorithm to produce the byte sequence `M`.
3. Sign `M` according to §3.

When verifying a claim:
1. Parse the JSON object.
2. Remove the `sig` field to produce the unsigned object.
3. Apply the canonicalization algorithm to produce `M`.
4. Verify the signature from `sig` against `M` according to §3.

## 2.5 Test vectors

The following test vectors MUST be reproduced exactly by a conformant implementation.

### Vector 1: simple object

Input (any equivalent JSON representation):
```json
{
  "ocp": 1,
  "exp": 0,
  "iss": "data:key/es256;base64,test",
  "nbf": 0,
  "stm": {},
  "sub": "data:key/es256;base64,test"
}
```

Canonical output (UTF-8 bytes, shown as a string):
```
{"exp":0,"iss":"data:key/es256;base64,test","nbf":0,"ocp":1,"stm":{},"sub":"data:key/es256;base64,test"}
```

Note: keys are in lexicographic order: `exp` < `iss` < `nbf` < `ocp` < `stm` < `sub`.

### Vector 2: nested object and string escaping

Input:
```json
{
  "b": {"z": 1, "a": 2},
  "a": "hello\nworld"
}
```

Canonical output:
```
{"a":"hello\nworld","b":{"a":2,"z":1}}
```

Explanation: top-level keys `a` < `b`; nested keys `a` < `z`; `\n` is the two-character escape sequence (not a literal newline).

### Vector 3: number edge cases

Input:
```json
{
  "zero": 0,
  "neg_zero": -0,
  "big": 9007199254740992,
  "frac": 1.5
}
```

Canonical output:
```
{"big":9007199254740992,"frac":1.5,"neg_zero":0,"zero":0}
```

Note: `-0` serializes as `0`; `9007199254740992` is exactly representable as an integer.
