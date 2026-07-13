# CB58 - Encoding Standard

**Phase:** 1 - Architecture + Cryptography + Wallet  
**Status:** ✅ Implemented and verified  
**SDK file:** `lib/src/crypto/cb58.dart`  
**Tests:** `test/src/crypto/cb58_test.dart`


## Overview

CB58 is the binary-to-text encoding format used across the Avalanche
ecosystem to represent keys, addresses, transaction IDs, chain IDs,
and other binary values in APIs and wallets. It is a Base58-encoded
string with a 4-byte SHA256 checksum appended to detect transcription
errors.

**Official sources:**
- [What is CB58? - Avalanche Support](https://support.avax.network/en/articles/4587395-what-is-cb58)
- [Cryptographic Primitives - Avalanche Docs](https://build.avax.network/docs/rpcs/other/standards/cryptographic-primitives)
- [Issuing API Calls - Avalanche Docs](https://docs.avax.network/docs/rpcs/other/guides/issuing-api-calls)

> "When transmitting information through external applications, the
> CB58 convention is required."


## Decision 1: Algorithm - SHA256 checksum of 4 bytes

**Why:**  
CB58 appends a 4-byte checksum to the data before Base58-encoding.
The checksum is the last 4 bytes of the SHA256 hash of the data.
This allows decoders to detect immediately if a key or address was
mistyped or corrupted - without this, a transcription error would
fail silently and potentially result in lost funds.

**Official source:**  
[What is CB58? - Avalanche Support](https://support.avax.network/en/articles/4587395-what-is-cb58)

> "CB58 is the concatenation of the data bytes and a checksum.
> The checksum is created by taking the last four bytes of the
> SHA256 hash of the data bytes. This concatenated output is then
> mapped to a base-58 string."

**Verified against AvalancheJS source code:**  
[avalanchejs/src/utils/base58.ts - Ava Labs](https://github.com/ava-labs/avalanchejs/blob/master/src/utils/base58.ts)

```typescript
// AvalancheJS reference implementation
export const base58check: BytesCoder = {
  encode(data) {
    return base58.encode(concatBytes(data, sha256(data).subarray(-4)));
  },
};
```

**Implementation:**
```dart
static const int _checksumLength = 4;

static Uint8List _sha256(Uint8List data) {
  final digest = SHA256Digest();
  return digest.process(data);
}

static String encode(Uint8List data) {
  final checksum = _sha256(data).sublist(32 - _checksumLength);
  final payload = Uint8List.fromList([...data, ...checksum]);
  return _base58Encode(payload);
}
```


## Decision 2: SHA256Digest from pointycastle

**Why:**  
`pointycastle` is already a dependency of `avalanche_flutter_sdk`
for secp256k1 cryptography. Reusing `SHA256Digest` from the same
library avoids adding a new dependency for a single hash function,
keeps the dependency tree minimal, and ensures consistency across
the crypto layer.

**Implementation:**
```dart
import 'package:pointycastle/export.dart' hide PublicKey;

static Uint8List _sha256(Uint8List data) {
  final digest = SHA256Digest();
  return digest.process(data);
}
```


## Decision 3: Base58 Alphabet - Bitcoin/Avalanche standard

**Why:**  
The Base58 alphabet excludes four visually ambiguous characters to
prevent transcription errors when copying keys or addresses manually:
`0` (zero), `O` (capital O), `I` (capital I), and `l` (lowercase L).
Avalanche uses the same alphabet as Bitcoin, confirmed directly from
the AvalancheJS source code.

**Verified against AvalancheJS:**  
[avalanchejs/src/utils/base58.ts - Ava Labs](https://github.com/ava-labs/avalanchejs/blob/master/src/utils/base58.ts)

**Implementation:**
```dart
/// The Base58 alphabet used by Bitcoin and Avalanche (CB58).
/// Excludes visually ambiguous characters: 0 (zero), O (capital o),
/// I (capital i), l (lowercase L).
static const String _alphabet =
    '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';
```


## Decision 4: Checksum verification in decode()

**Why:**  
The AvalancheJS reference implementation does not explicitly verify
the checksum on decode - it simply truncates the last 4 bytes.
However, the CB58 specification states that the checksum exists
precisely to detect errors. Verifying the checksum on decode is
the stricter and more correct behavior, and is the right choice
for a production SDK where a corrupted key could result in
loss of funds.

**AvalancheJS reference (does not verify):**
```typescript
decode(string) {
  return base58.decode(string).subarray(0, -4); // just truncates
},
```

**Our implementation (verifies):**
```dart
static Uint8List decode(String encoded) {
  final payload = _base58Decode(encoded);

  if (payload.length < _checksumLength) {
    throw ArgumentError(
      'Invalid CB58 string: payload too short to contain a checksum.',
    );
  }

  final data = payload.sublist(0, payload.length - _checksumLength);
  final embeddedChecksum = payload.sublist(
    payload.length - _checksumLength,
  );
  final expectedChecksum = _sha256(data).sublist(32 - _checksumLength);

  if (!_bytesEqual(embeddedChecksum, expectedChecksum)) {
    throw ArgumentError(
      'Invalid CB58 string: checksum mismatch. '
      'The string may have been mistyped or corrupted.',
    );
  }

  return data;
}
```


## Decision 5: Constant-time comparison via XOR

**Why:**  
Checksum comparison uses XOR accumulation instead of direct byte
equality. This prevents timing attacks where an attacker could infer
information about the expected checksum by measuring how long the
comparison takes. Each byte is XORed and the result accumulated -
a zero result means all bytes matched.

**Implementation:**
```dart
static bool _bytesEqual(Uint8List a, Uint8List b) {
  if (a.length != b.length) return false;
  var result = 0;
  for (var i = 0; i < a.length; i++) {
    result |= a[i] ^ b[i];
  }
  return result == 0;
}
```


## Decision 6: Leading zero bytes preservation

**Why:**  
Base58 has a special convention: each leading zero byte in the
input is represented as a leading `1` character in the output.
This must be handled explicitly because the BigInteger conversion
used for Base58 encoding loses leading zero bytes. Failing to
handle this would produce incorrect CB58 strings for data that
starts with `0x00`.

**Implementation:**
```dart
static String _base58Encode(Uint8List input) {
  // Count leading zero bytes - each becomes a leading '1'
  var leadingZeros = 0;
  while (leadingZeros < input.length && input[leadingZeros] == 0) {
    leadingZeros++;
  }

  // ... BigInteger conversion ...

  final buffer = StringBuffer()..write('1' * leadingZeros);
  for (final digit in digits.reversed) {
    buffer.write(_alphabet[digit]);
  }
  return buffer.toString();
}
```


## Bug found during development

**The `<=` vs `<` boundary error:**

During development, `decode()` used `<=` instead of `<` to validate
the minimum payload length:

```dart
// Bug: rejects valid CB58 of empty data (payload = 4 bytes = checksum only)
if (payload.length <= _checksumLength) { ... }

// Fix: only reject if strictly less than checksum length
if (payload.length < _checksumLength) { ... }
```

Empty data (0 bytes) is a valid CB58 input - its encoded form
contains only the 4-byte checksum. The `<=` check incorrectly
rejected this case. This was caught by the test suite before
any release.


## Reference test vector

Verified against `moreati/cb58ref` - a reference implementation of
CB58 for the AVA/Avalanche network:

```dart
// Source: https://github.com/moreati/cb58ref
// >>> cb58ref.cb58encode(b"Hello world")
// '32UWxgjUJd9s6Kyvxjj1u'

test('encodes "Hello world" to the known reference output', () {
  final data = Uint8List.fromList(utf8.encode('Hello world'));
  expect(CB58.encode(data), equals('32UWxgjUJd9s6Kyvxjj1u'));
});

test('decodes the reference CB58 string back to "Hello world"', () {
  final decoded = CB58.decode('32UWxgjUJd9s6Kyvxjj1u');
  expect(utf8.decode(decoded), equals('Hello world'));
});
```


## Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Checksum | SHA256(data)[-4:] | Official CB58 spec |
| SHA256 implementation | `pointycastle SHA256Digest` | Reuse existing dependency |
| Base58 alphabet | Bitcoin/Avalanche standard (58 chars) | Confirmed from AvalancheJS source |
| Checksum on decode | Verify (not just truncate) | Stricter than reference, safer for production |
| Byte comparison | XOR accumulation | Constant-time, prevents timing attacks |
| Leading zeros | Explicit `1` prefix per zero byte | Required by Base58 standard |

---

*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixpay)*
