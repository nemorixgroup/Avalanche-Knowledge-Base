# secp256k1 - Key Generation and Curve Parameters

**Phase:** 1 - Architecture + Cryptography + Wallet  
**Status:** ✅ Implemented and verified  
**SDK files:**
- `lib/src/crypto/private_key.dart`
- `lib/src/crypto/public_key.dart`  
**Tests:** `test/src/crypto/private_key_test.dart`, `test/src/crypto/public_key_test.dart`


## Overview

secp256k1 is the elliptic curve used by Avalanche for all cryptographic
operations across C-Chain, P-Chain, and X-Chain. It is the same curve
used by Bitcoin and Ethereum. A private key is a 256-bit integer `d`
in the range `0 < d < n`, where `n` is the curve order. The public key
is the point `Q = d * G` on the curve, where `G` is the generator point.

**Official source:**
[Avalanche Cryptographic Primitives](https://build.avax.network/docs/rpcs/other/standards/cryptographic-primitives)

> "The Avalanche virtual machine uses elliptic curve cryptography,
> specifically secp256k1, for its signatures on the blockchain."


## Decision 1: Library Selection - pointycastle ^4.0.0

**Why:**  
Dart does not have a native secp256k1 implementation in its standard
library. A third-party package is required. `pointycastle` is the only
Dart cryptography library with native support for `ECDomainParameters('secp256k1')`,
active maintenance, and over 2 million downloads on pub.dev.

**Alternatives considered:**

| Library | Reason rejected |
|---------|----------------|
| `web3dart` crypto module | Ethereum-focused wrapper, not a standalone crypto library. Adds unnecessary EVM dependencies. |
| `dart-secp256k1` | Community package, unmaintained, no pub.dev publisher verification. |
| `cryptography` (package) | Does not expose raw secp256k1 ECDomainParameters needed for Avalanche-specific operations. |
| `pointycastle ^4.0.0` ✅ | Native `ECDomainParameters('secp256k1')`, `ECKeyGenerator`, `ECDSASigner`. Active maintenance. Verified publisher. |

**Official source:**  
[pointycastle on pub.dev](https://pub.dev/packages/pointycastle)

**Implementation:**
```dart
import 'package:pointycastle/export.dart' hide PublicKey;

static final ECDomainParameters _domainParams =
    ECDomainParameters('secp256k1');
```

> **Note:** `hide PublicKey` is required because `pointycastle` exports
> its own generic `PublicKey` class that conflicts with our
> `avalanche_flutter_sdk` `PublicKey` class.


## Decision 2: Secure Random Number Generation - FortunaRandom

**Why:**  
Private key generation requires a cryptographically secure
pseudo-random number generator (CSPRNG). Dart's `Random.secure()`
provides the entropy seed, but `pointycastle`'s `FortunaRandom` is
required as the actual CSPRNG for key generation via `ECKeyGenerator`.
Using a non-secure random source would make private keys predictable
and exploitable.

**Official source:**  
[pointycastle FortunaRandom](https://pub.dev/documentation/pointycastle/latest/impl.secure_random.fortuna_random/FortunaRandom-class.html)

**Implementation:**
```dart
static SecureRandom _buildSecureRandom() {
  final secureRandom = FortunaRandom();
  final random = Random.secure();
  final seed = Uint8List.fromList(
    List<int>.generate(32, (_) => random.nextInt(256)),
  );
  secureRandom.seed(KeyParameter(seed));
  return secureRandom;
}

factory PrivateKey.generate() {
  final secureRandom = _buildSecureRandom();
  final keyGenerator = ECKeyGenerator()
    ..init(
      ParametersWithRandom(
        ECKeyGeneratorParameters(_domainParams),
        secureRandom,
      ),
    );
  final pair = keyGenerator.generateKeyPair();
  final ecPrivateKey = pair.privateKey as ECPrivateKey;
  return PrivateKey._(ecPrivateKey.d!);
}
```


## Decision 3: Private Key Range Validation - 0 < d < n

**Why:**  
Not every 256-bit integer is a valid secp256k1 private key. The value
`d` must satisfy `0 < d < n`, where `n` is the curve order. A value of
`d = 0` has no corresponding public key. A value of `d >= n` wraps
around the curve and produces a non-unique key. Both cases are
cryptographically invalid and must be rejected on import.

**Curve order `n` - verified at runtime:**  
The value of `n` was verified directly from `pointycastle`'s
`ECDomainParameters('secp256k1').n` at runtime - not hardcoded from
an external source:

```
n = fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
```

**Implementation:**
```dart
factory PrivateKey.fromHex(String hex) {
  final cleanHex = hex.startsWith('0x') ? hex.substring(2) : hex;

  if (cleanHex.length != 64) {
    throw ArgumentError(
      'Invalid private key length: expected 64 hex characters '
      '(32 bytes), got ${cleanHex.length}.',
    );
  }

  final d = BigInt.parse(cleanHex, radix: 16);

  if (d <= BigInt.zero || d >= _domainParams.n) {
    throw ArgumentError(
      'Private key out of valid range for secp256k1.',
    );
  }

  return PrivateKey._(d);
}
```

**Boundary tests:**
```dart
// Verified against pointycastle's n at runtime
const curveOrderN =
    'fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141';
const curveOrderNMinusOne =
    'fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364140';

test('throws ArgumentError when d is zero', () {
  expect(() => PrivateKey.fromHex('0'.padLeft(64, '0')),
      throwsA(isA<ArgumentError>()));
});

test('throws ArgumentError when d equals curve order n', () {
  expect(() => PrivateKey.fromHex(curveOrderN),
      throwsA(isA<ArgumentError>()));
});

test('accepts d = 1 (minimum valid value)', () {
  expect(() => PrivateKey.fromHex('1'.padLeft(64, '0')), returnsNormally);
});

test('accepts d = n - 1 (maximum valid value)', () {
  expect(() => PrivateKey.fromHex(curveOrderNMinusOne), returnsNormally);
});
```


## Decision 4: Public Key Derivation - Q = d * G

**Why:**  
The public key is deterministically derived from the private key using
scalar multiplication on the secp256k1 curve: `Q = d * G`, where `G`
is the generator point defined in the curve parameters. This is the
standard derivation and has no alternatives - it is defined by the
secp256k1 specification.

**Implementation:**
```dart
PublicKey get publicKey {
  final point = _domainParams.G * _d;
  return PublicKey.fromEcPoint(point!);
}
```


## Decision 5: Two Public Key Encodings

**Why:**  
Avalanche uses secp256k1 for all three chains, but the address
derivation algorithm is different per chain. Each algorithm requires
a different encoding of the public key as its input:

| Chain | Address Algorithm | Required Input | Method |
|-------|------------------|----------------|--------|
| X-Chain / P-Chain | `sha256 → ripemd160` | Compressed (33 bytes, SEC1) | `toCompressed()` |
| C-Chain (EVM) | `keccak256` | Raw uncompressed (64 bytes, no `0x04` prefix) | `toRawUncompressed()` |

**Official source:**  
[Avalanche Cryptographic Primitives - Addresses](https://build.avax.network/docs/rpcs/other/standards/cryptographic-primitives#secp256k1-addresses)

> "The 33-byte compressed representation of the public key is hashed
> with sha256 once. The result is then hashed with ripemd160."
> (X-Chain / P-Chain address derivation)

**Implementation:**
```dart
/// Compressed (33 bytes) - input for X/P-Chain address derivation.
Uint8List toCompressed() {
  final encoded = _point.getEncoded();
  return Uint8List.fromList(encoded);
}

/// Uncompressed (65 bytes) - includes 0x04 prefix.
Uint8List toUncompressed() {
  final encoded = _point.getEncoded(false);
  return Uint8List.fromList(encoded);
}

/// Raw uncompressed (64 bytes, no 0x04 prefix) - input for C-Chain
/// EVM address derivation (keccak256 input).
Uint8List toRawUncompressed() {
  final uncompressed = toUncompressed();
  return Uint8List.sublistView(uncompressed, 1);
}
```

> **Note:** `getEncoded()` defaults to `true` (compressed) in
> `pointycastle`. The `false` argument is required explicitly
> for uncompressed encoding.


## Decision 6: Private Key toString() - Always [REDACTED]

**Why:**  
Private key material must never appear in logs, error messages,
stack traces, or debug output. If a developer accidentally logs
a `PrivateKey` instance, the actual key value must not be exposed.
This is a non-negotiable security requirement for any SDK that
handles cryptographic key material.

**Implementation:**
```dart
@override
String toString() => 'PrivateKey[REDACTED]';
```

**Security test:**
```dart
test('never exposes the private key value', () {
  final key = PrivateKey.fromHex('1'.padLeft(64, '0'));
  expect(key.toString(), equals('PrivateKey[REDACTED]'));
});

test('redacted string does not contain any hex of the key', () {
  final key = PrivateKey.generate();
  expect(key.toString().contains(key.toHex()), isFalse);
});
```

> `PublicKey.toString()` intentionally does NOT redact - public keys
> are safe to expose by definition.


## Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Crypto library | `pointycastle ^4.0.0` | Only Dart lib with native secp256k1 |
| CSPRNG | `FortunaRandom` + `Random.secure()` | Required for secure key generation |
| Range validation | `0 < d < n` | Spec requirement; `n` verified at runtime |
| Public key derivation | `Q = d * G` | Standard secp256k1 derivation |
| Compressed encoding | `toCompressed()` 33 bytes | X/P-Chain address derivation input |
| Raw uncompressed | `toRawUncompressed()` 64 bytes | C-Chain EVM address derivation input |
| toString() | Always `[REDACTED]` | Security: no key material in logs |

*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixpay)*
