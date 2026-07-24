# X/P-Chain Address - Native Avalanche Address Derivation

**Phase:** 1 - Architecture + Cryptography + Wallet  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.1.0-dev  
**SDK file:** `lib/src/crypto/address/xp_address.dart`  
**Tests:** `test/src/crypto/address/xp_address_test.dart`


## Overview

X-Chain and P-Chain use a native Avalanche address format based on
Bitcoin's approach - hashing the public key and encoding with Bech32.
This is completely different from C-Chain (EVM) which uses keccak256.

```
PublicKey (secp256k1)
    -> toCompressed() - 33 bytes (with 0x02 or 0x03 prefix)
    -> SHA256(33 bytes) - 32 bytes
    -> RIPEMD160(32 bytes) - 20 bytes (raw address)
    -> Bech32 encode(20 bytes, hrp) - bech32 string
    -> chain prefix + bech32
X-avax1pvujlvryqevdu5cgc3nal55x5f8z39cw6t5p37  (X-Chain mainnet)
P-avax1pvujlvryqevdu5cgc3nal55x5f8z39cw6t5p37  (P-Chain mainnet)
X-fuji1pvujlvryqevdu5cgc3nal55x5f8z39cwkes7ap   (X-Chain Fuji)
```

**Official sources:**
- [Avalanche cryptographic primitives](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)
- [BIP-173 - Bech32 standard](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)
- [Avalanche Support - What is Bech32?](https://support.avax.network/en/articles/4587392-what-is-bech32)


## Decision 1: Compressed Public Key (33 bytes) as SHA256 Input

**Why:**
X/P-Chain address derivation uses the **compressed** public key
(33 bytes) as input to SHA256. This is the opposite of C-Chain which
uses the raw uncompressed key (64 bytes) as input to keccak256.

Using the wrong encoding would produce a completely different and
incompatible address. The compressed key includes the `0x02` or
`0x03` prefix byte that indicates the parity of the Y coordinate.

**Official source:**
[Avalanche cryptographic primitives](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)

> "The 33-byte compressed representation of the public key is hashed
> with sha256 once. The result is then hashed with ripemd160 to yield
> a 20-byte address."

**Implementation:**
```dart
factory XPAddress.fromPublicKey(PublicKey publicKey) {
  final compressed     = publicKey.toCompressed(); // 33 bytes
  final sha256Result   = _sha256(compressed);      // 32 bytes
  final ripemd160Result = _ripemd160(sha256Result); // 20 bytes
  return XPAddress._(ripemd160Result);
}
```


## Decision 2: SHA256 then RIPEMD160 (Bitcoin-style)

**Why:**
Avalanche follows Bitcoin's approach for hashing public keys.
Applying two different hash functions provides defense in depth:
if one hash function is broken in the future, the other still
protects the address. This is the same rationale Bitcoin uses.

Both `SHA256Digest` and `RIPEMD160Digest` are available in
`pointycastle` - no new dependencies needed.

**Official source:**
[Avalanche cryptographic primitives](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)

> "Avalanche follows a similar approach as Bitcoin and hashes the
> ECDSA public key."

**Implementation:**
```dart
static Uint8List _sha256(Uint8List data) {
  final digest = SHA256Digest();
  return digest.process(data);
}

static Uint8List _ripemd160(Uint8List data) {
  final digest = RIPEMD160Digest();
  return digest.process(data);
}
```

**Official example (from Avalanche docs):**
```
Public Key (33-byte compressed):
  0x02b33c917f2f6103448d7feb42614037d05928433cb25e78f01a825aa829bb3c27

SHA256(Public Key):
  0x28d7670d71667e93ff586f664937f52828e6290068fa2a37782045bffa7b0d2f

RIPEMD160(SHA256):
  0xe8777f38c88ca153a6fdc25942176d2bf5491b89 (20 bytes)
```


## Decision 3: Bech32 Implemented from Scratch

**Why:**
No Dart/Flutter library for Bech32 was available in pub.dev at
implementation time that was compatible with the Avalanche address
format and `pointycastle ^4.0.0`. Implementing Bech32 from scratch
per BIP-173 avoids adding an unverified dependency to the SDK and
gives full control over the encoding.

The implementation covers:
- `_convertBits`: converts 8-bit bytes to 5-bit groups
- `_polymod`: computes the BCH checksum
- `_hrpExpand`: expands the HRP for checksum computation
- `_createChecksum`: produces the 6-character checksum
- `_bech32Encode`: assembles the final address string

**Official source:**
[BIP-173 - Bech32 standard](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)

**Implementation:**
```dart
static const String _charset = 'qpzry9x8gf2tvdw0s3jn54khce6mua7l';

static const List<int> _generator = [
  0x3b6a57b2, 0x26508e6d, 0x1ea119fa, 0x3d4233dd, 0x2a1462b3,
];

static String _bech32Encode(String hrp, Uint8List data) {
  final converted = _convertBits(data, 8, 5);
  final checksum  = _createChecksum(hrp, converted);
  final combined  = [...converted, ...checksum];
  final result    = StringBuffer(hrp)..write('1');
  for (final value in combined) {
    result.write(_charset[value]);
  }
  return result.toString();
}
```


## Decision 4: AvalancheNetwork Enum for HRP

**Why:**
The Human-Readable Part (HRP) of the Bech32 address is different
per network. An enum prevents using incorrect HRP strings and makes
network-specific address generation explicit and compile-time safe.

| Network | HRP | X-Chain prefix | P-Chain prefix |
|---|---|---|---|
| Mainnet | `avax` | `X-avax1` | `P-avax1` |
| Fuji Testnet | `fuji` | `X-fuji1` | `P-fuji1` |

**Official source:**
[Avalanche cryptographic primitives](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)

> "A human-readable part (HRP). On Mainnet this is avax."

**Implementation:**
```dart
enum AvalancheNetwork {
  mainnet('avax'),
  fuji('fuji');

  const AvalancheNetwork(this.hrp);
  final String hrp;
}
```


## Decision 5: X-Chain and P-Chain Share the Same 20-byte Address

**Why:**
X-Chain and P-Chain use the same derivation path
(`m/44'/9000'/0'/0/n`) and the same address algorithm. The only
difference is the chain prefix (`X-` vs `P-`). This means the
same public key produces the same 20-byte address for both chains,
just displayed with different prefixes.

**Official source:**
[How are C, X, and P-Chain addresses derived?](https://support.avax.network/en/articles/6097243-how-are-c-x-and-p-chain-addresses-derived)

> "X and P addresses will share the same address space"

**Implementation:**
```dart
String xChainAddress({AvalancheNetwork network = AvalancheNetwork.mainnet}) =>
    'X-${_bech32Encode(network.hrp, _bytes)}';

String pChainAddress({AvalancheNetwork network = AvalancheNetwork.mainnet}) =>
    'P-${_bech32Encode(network.hrp, _bytes)}';
```

**Verification:**
```dart
test('X-Chain and P-Chain share same bech32 data', () {
  final address = XPAddress.fromPublicKey(pk.publicKey);
  // Strip chain prefix - everything after "X-" and "P-" is equal
  expect(address.xChainAddress().substring(2),
      equals(address.pChainAddress().substring(2)));
});
```


## Decision 6: Bech32 Address Length is Always 45 Characters

**Why:**
The address length is deterministic:
- 20 bytes -> `_convertBits(8, 5)` -> 32 five-bit groups
- 32 data chars + 6 checksum chars = 38 chars
- `X-` (2) + HRP `avax` (4) + `1` separator (1) + 38 = 45 chars

This is verified in the test suite to catch any Bech32 encoding
bugs early.

**Verification:**
```dart
test('mainnet address has correct total length', () {
  // X- (2) + avax1 (5) + 32 data + 6 checksum = 45
  expect(address.xChainAddress().length, equals(45));
});
```


## Decision 7: toString() Returns X-Chain Mainnet Address

**Why:**
Unlike `PrivateKey`, `Mnemonic`, `Seed`, and `HDWallet` which
return `[REDACTED]`, an `XPAddress` is **public information**.
`toString()` returns the X-Chain mainnet address as the default
representation - the most commonly used form.

**Implementation:**
```dart
@override
String toString() =>
    'XPAddress(X-${_bech32Encode(AvalancheNetwork.mainnet.hrp, _bytes)})';
```


## Key Difference vs C-Chain (EVM)

| | C-Chain (EVM) | X-Chain / P-Chain |
|---|---|---|
| Public key input | Raw uncompressed 64 bytes | Compressed 33 bytes |
| Hash algorithm | keccak256 | SHA256 -> RIPEMD160 |
| Output | 20 bytes | 20 bytes |
| Encoding | EIP-55 hex with `0x` | Bech32 with chain prefix |
| Example | `0x71C7656EC7ab...` | `X-avax1pvujlvry...` |
| Derivation path | `m/44'/60'/0'/0/n` | `m/44'/9000'/0'/0/n` |

---

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| Public key input | `toCompressed()` 33 bytes | Avalanche spec requirement |
| First hash | SHA256 | Bitcoin-style double hash |
| Second hash | RIPEMD160 | Defense in depth |
| Bech32 | Implemented from scratch | No compatible Dart library |
| HRP | `AvalancheNetwork` enum | Compile-time safety per network |
| X/P sharing | Same 20-byte address | Same path, different prefix only |
| Address length | Always 45 chars | Deterministic from algorithm |
| toString() | X-Chain mainnet address | Address is public information |
