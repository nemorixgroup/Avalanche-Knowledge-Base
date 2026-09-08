# EIP-1559 - AVAX Transfer Transaction

**Phase:** 2 - C-Chain Core (EVM  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.1.2-dev  
**SDK files:**
- `lib/src/crypto/ecdsa_signature.dart`
- `lib/src/crypto/private_key.dart` (signDigest method)
- `lib/src/chains/cchain/utils/rlp_encoder.dart`
- `lib/src/chains/cchain/transaction/avax_transfer_tx.dart`

**Tests:**
- `test/src/crypto/ecdsa_signature_test.dart`
- `test/src/chains/cchain/utils/rlp_encoder_test.dart`
- `test/src/chains/cchain/transaction/avax_transfer_tx_test.dart`

## Overview

EIP-1559 (EIP-2718 type 2) is the transaction format used on
Avalanche C-Chain for all AVAX transfers and contract interactions.
It replaces the legacy gas price model with a base fee + priority
fee (tip) model.

The full signing flow:

```
PrivateKey (secp256k1)
    -> signDigest(unsignedHash)
    -> EcdsaSignature (r, s, v)

AvaxTransferTransaction fields
    -> RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
            gasLimit, to, value, data, accessList])
    -> prepend 0x02 (EIP-2718 type byte)
    -> keccak256 -> unsignedHash (32 bytes)
    -> signDigest(unsignedHash) -> EcdsaSignature

RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
     gasLimit, to, value, data, accessList,
     signatureYParity, r, s])
    -> prepend 0x02
    -> hex encode
    -> raw transaction (0x02f86d...)
    -> eth_sendRawTransaction
    -> txHash
```

**Official sources:**
- [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)
- [EIP-2718 - Typed Transaction Envelope](https://eips.ethereum.org/EIPS/eip-2718)
- [RLP Encoding](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)
- [Avalanche C-Chain RPC](https://build.avax.network/docs/rpcs/c-chain)

## Part 1: RLP Encoding

### Decision 1: Implement RLP from scratch (no external library)

**Why:**
RLP (Recursive Length Prefix) is the serialization format for all
Ethereum and Avalanche C-Chain transactions. No Dart library with
sufficient adoption existed at implementation time - the best
candidate (`rlp ^2.0.0`) had only 184 downloads. Implementing RLP
from scratch (~80 lines) avoids adding an unverified dependency and
is consistent with the SDK philosophy of minimal, controlled
dependencies.

**Alternatives considered:**

| Library | Downloads | Reason rejected |
|---|---|---|
| `rlp ^2.0.0` | 184 | Too low adoption, unverified |
| `ethereum_util ^1.5.1` | Higher | Too large, potential conflicts |
| **From scratch** ✅ | N/A | ~80 lines, fully controlled |

**Official source:**
[Ethereum RLP encoding](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)

### Decision 2: RLP rules for each type

**Why:**
The RLP spec defines different encoding rules for each input type.
The key decisions are:

- **`BigInt.zero` encodes as empty bytes (`0x80`)** - per RLP spec,
  zero is the empty byte string, not `0x00`. This is critical for
  nonce=0 (new wallet) and data=[] (no contract call).
- **Single byte `< 0x80` encodes as-is** - no length prefix needed.
  This covers values 0x01 to 0x7f (e.g. `v=0` or `v=1`).
- **Negative values throw `ArgumentError`** - RLP does not support
  signed integers; all transaction fields are non-negative.

**Implementation:**
```dart
static Uint8List _bigIntToBytes(BigInt value) {
  if (value == BigInt.zero) return Uint8List(0); // 0x80 after prefix
  if (value < BigInt.zero) {
    throw ArgumentError('RlpEncoder: negative values not supported.');
  }
  final hex = value.toRadixString(16);
  final padded = hex.length.isOdd ? '0$hex' : hex;
  return Uint8List.fromList(
    List.generate(padded.length ~/ 2,
      (i) => int.parse(padded.substring(i * 2, i * 2 + 2), radix: 16)),
  );
}
```

**Verified against official Ethereum RLP vectors:**
```dart
test('encodes BigInt.zero as 0x80', () {
  expect(RlpEncoder.encode(BigInt.zero),
      equals(Uint8List.fromList([0x80])));
});

test('encodes nested empty lists correctly', () {
  // Official vector from ethereum.org RLP docs
  final result = RlpEncoder.encode([[], [[]], [[], [[]]]]);
  expect(result, equals(Uint8List.fromList(
    [0xc7, 0xc0, 0xc1, 0xc0, 0xc3, 0xc0, 0xc1, 0xc0])));
});
```

## Part 2: secp256k1 Signing (RFC 6979)

### Decision 3: RFC 6979 deterministic nonce via ECDSASigner

**Why:**
ECDSA signing requires a random nonce `k`. Using a truly random `k`
means the same transaction signed twice produces different signatures.
RFC 6979 derives `k` deterministically from the private key and the
message digest using HMAC-SHA256 - the same inputs always produce the
same signature. This is the standard used by Bitcoin, Ethereum, and
Avalanche.

`pointycastle`'s `ECDSASigner(null, HMac(SHA256Digest(), 64))`
implements RFC 6979 automatically.

**Official source:**
[RFC 6979 - Deterministic DSA/ECDSA](https://www.rfc-editor.org/rfc/rfc6979)

**Implementation:**
```dart
EcdsaSignature signDigest(Uint8List digest) {
  final signer = ECDSASigner(null, HMac(SHA256Digest(), 64))
    ..init(true, PrivateKeyParameter(ECPrivateKey(_d, _domainParams)));
  final sig = signer.generateSignature(digest) as ECSignature;
  // ...
}
```

**Verification:**
```dart
test('signing is deterministic (RFC 6979)', () {
  final key = PrivateKey.fromHex('1'.padLeft(64, '0'));
  final digest = Uint8List(32)..fillRange(0, 32, 0xab);
  final sig1 = key.signDigest(digest);
  final sig2 = key.signDigest(digest);
  expect(sig1.r, equals(sig2.r));
  expect(sig1.s, equals(sig2.s));
});
```

### Decision 4: Low-S normalization (signature malleability prevention)

**Why:**
For any ECDSA signature `(r, s)`, `(r, n - s)` is also a valid
signature. Without normalization, both forms would be accepted by
the network, allowing a third party to mutate a transaction's hash
without invalidating it (signature malleability). Ethereum and
Avalanche require `s <= n/2` (low-S). If `s > n/2`, we replace
`s` with `n - s` and flip the recovery id `v`.

**Official source:**
[EIP-2 - Homestead Hard-fork Changes](https://eips.ethereum.org/EIPS/eip-2)

> "All transaction signatures whose s-value is greater than
> secp256k1n/2 are now considered invalid."

**Implementation:**
```dart
final halfN = n >> 1;
var s = sig.s;
var v = _computeRecoveryId(digest, r, s);

if (s > halfN) {
  s = n - s;
  v = 1 - v; // flip recovery id when s is normalized
}
```

**Verification:**
```dart
test('s is always low-S (s <= n/2)', () {
  final n = BigInt.parse(
    'FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141',
    radix: 16);
  final key = PrivateKey.generate();
  final digest = Uint8List(32)..fillRange(0, 32, 0x01);
  final sig = key.signDigest(digest);
  expect(sig.s <= n >> 1, isTrue);
});
```

### Decision 5: Recovery id v is 0 or 1 (not 27/28)

**Why:**
Legacy Ethereum transactions used `v = 27 + recoveryId` (or higher
with EIP-155 replay protection). EIP-1559 transactions use the raw
recovery id: `0` or `1`. Using the wrong `v` value would produce a
transaction that nodes reject.

**Official source:**
[EIP-1559 specification](https://eips.ethereum.org/EIPS/eip-1559)

> "signatureYParity: The parity (0 for even, 1 for odd) of the
> y-value of the secp256k1 signature."

**Implementation:**
```dart
class EcdsaSignature {
  final BigInt r;
  final BigInt s;
  final int v; // 0 or 1 - NOT 27/28
}
```

### Decision 6: Recovery id computed by public key recovery

**Why:**
`pointycastle`'s `ECDSASigner` returns `(r, s)` but not the recovery
id. The recovery id is determined by trying `v = 0` and `v = 1`,
recovering the public key from each, and comparing with the expected
public key. The `v` that produces a match is the correct recovery id.

**Implementation:**
```dart
int _computeRecoveryId(Uint8List digest, BigInt r, BigInt s) {
  for (var yBit = 0; yBit < 2; yBit++) {
    final R = _decompressPoint(r, yBit, curve);
    if (R == null) continue;
    final Q = (R * s + G * (n - z % n)) * r.modInverse(n);
    if (Q == null) continue;
    if (_bytesAreEqual(Q.getEncoded(true), expectedPubKey)) return yBit;
  }
  throw StateError('Failed to compute recovery id.');
}
```

## Part 3: EIP-1559 Transaction

### Decision 7: Transaction format - EIP-2718 type 2

**Why:**
EIP-1559 transactions use the EIP-2718 typed transaction envelope.
The raw transaction is `0x02 || RLP([fields])` where `0x02` is the
type byte that distinguishes EIP-1559 from legacy transactions.
Without the type byte, nodes would interpret the transaction as
legacy format and reject it.

**Official source:**
[EIP-2718 - Typed Transaction Envelope](https://eips.ethereum.org/EIPS/eip-2718)

**Unsigned transaction (for signing hash):**
```
0x02 || RLP([
  chainId,               // 43113 Fuji / 43114 Mainnet
  nonce,
  maxPriorityFeePerGas,
  maxFeePerGas,
  gasLimit,              // 21000 for AVAX transfer
  to,                    // 20 bytes
  value,                 // Wei
  data,                  // Uint8List(0) for AVAX transfer
  accessList,            // [] for AVAX transfer
])
```

**Signed transaction (for broadcast):**
```
0x02 || RLP([
  ...same fields as unsigned...,
  signatureYParity,      // v: 0 or 1
  signatureR,            // r: BigInt
  signatureS,            // s: BigInt (low-S normalized)
])
```

**Implementation:**
```dart
String sign(PrivateKey privateKey) {
  final hash = unsignedHash();
  final sig = privateKey.signDigest(hash);

  final signedRlp = RlpEncoder.encode([
    BigInt.from(chainId), BigInt.from(nonce),
    maxPriorityFeePerGas, maxFeePerGas, gasLimit,
    _addressToBytes(to), value,
    Uint8List(0),   // data: empty for AVAX transfer
    <dynamic>[],    // accessList: empty
    BigInt.from(sig.v), sig.r, sig.s,
  ]);

  final rawTx = Uint8List(1 + signedRlp.length);
  rawTx[0] = 0x02; // EIP-2718 type byte
  rawTx.setAll(1, signedRlp);
  return '0x${rawTx.map((b) => b.toRadixString(16).padLeft(2, '0')).join()}';
}
```

### Decision 8: data and accessList are always empty for AVAX transfers

**Why:**
A simple AVAX transfer (no contract interaction) has:
- `data = Uint8List(0)` - empty byte array, not null
- `accessList = []` - empty list, not null

Both fields must be present in the RLP encoding - omitting them
would produce a malformed transaction.


## On-Chain Verification

First transaction from `avalanche_flutter_sdk` - Fuji Testnet:

```
TX Hash  : 0x43f9d7b50013a5b18b0c1b6eda82875a39f7e7a01d114fa62595ced6f52d1ff2
Block    : 58,211,707
Status   : Success ✅
From     : 0xe9d70EE1dfcEd40152eA3f030d0a3918F2EE81a8
To       : 0x0fEB475783E004621b9463Aae30F33665485FafA
Amount   : 0.001 AVAX
Gas used : 21,000 units
Time     : ~2 seconds (Snowman consensus)
Explorer : https://testnet.snowtrace.io/tx/0x43f9d7b50013a5...d1ff2
```

Core Wallet compatibility: same mnemonic -> same address ✅


## Summary

| Decision | Choice | Rationale |
|---|---|---|
| RLP library | From scratch | No reliable Dart library (184 downloads) |
| Zero encoding | Empty bytes (0x80) | RLP spec requirement |
| Signing | RFC 6979 (HMAC-SHA256) | Deterministic, standard for EVM |
| Low-S | s = n - s when s > n/2 | EIP-2 malleability prevention |
| Recovery id | 0 or 1 (not 27/28) | EIP-1559 signatureYParity |
| Recovery method | Public key recovery | pointycastle doesn't expose v |
| Tx format | 0x02 \|\| RLP([...]) | EIP-2718 type 2 |
| data field | Uint8List(0) | Required, empty for AVAX transfer |
| accessList | [] | Required, empty for AVAX transfer |
