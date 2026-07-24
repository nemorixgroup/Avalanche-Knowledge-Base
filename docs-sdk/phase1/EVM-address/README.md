# EVM Address - C-Chain Address Derivation

**Phase:** 1 - Architecture + Cryptography + Wallet  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.1.0-dev  
**SDK file:** `lib/src/crypto/address/evm_address.dart`  
**Tests:** `test/src/crypto/address/evm_address_test.dart`


## Overview

The C-Chain is 100% EVM-compatible and uses the same address derivation
algorithm as Ethereum. The process:

```
PublicKey (secp256k1)
    -> toRawUncompressed() - 64 bytes (no 0x04 prefix)
    -> keccak256(64 bytes) - 32 bytes
    -> last 20 bytes - raw address
    -> EIP-55 checksum - mixed-case hex with 0x prefix
0x71C7656EC7ab88b098defB751B7401B5f6d8976F
```

**Official sources:**
- [Avalanche C-Chain cryptography](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)
- [EIP-55 - Mixed-case checksum address encoding](https://eips.ethereum.org/EIPS/eip-55)


## Decision 1: Raw Uncompressed Public Key as keccak256 Input

**Why:**
The EVM address derivation uses the raw uncompressed public key
(64 bytes, without the `0x04` prefix) as input to keccak256.
This is different from X/P-Chain which uses the compressed key
(33 bytes) as input to SHA256.

Using `toRawUncompressed()` (64 bytes) instead of `toUncompressed()`
(65 bytes with `0x04` prefix) is critical - including the prefix byte
would produce a completely different and incorrect address.

**Official source:**
[Avalanche cryptographic primitives](https://docs.avax.network/docs/rpcs/other/standards/cryptographic-primitives)

> "Avalanche nodes support the full Ethereum Virtual Machine (EVM)
> and precisely duplicate all of the cryptographic constructs used
> in Ethereum."

**Implementation:**
```dart
factory EvmAddress.fromPublicKey(PublicKey publicKey) {
  final raw  = publicKey.toRawUncompressed(); // 64 bytes, no 0x04
  final hash = _keccak256(raw);               // 32 bytes
  final bytes = hash.sublist(12);             // last 20 bytes
  return EvmAddress._(Uint8List.fromList(bytes));
}
```


## Decision 2: KeccakDigest(256) from pointycastle

**Why:**
Ethereum uses Keccak-256 (not the standardized SHA3-256). These are
different algorithms - using SHA3-256 would produce incorrect addresses.
`pointycastle` provides `KeccakDigest(256)` which implements the
original Keccak algorithm (pre-NIST standardization), matching
exactly what Ethereum uses.

**Implementation:**
```dart
static Uint8List _keccak256(Uint8List data) {
  final digest = KeccakDigest(256);
  return digest.process(data);
}
```

> **Note:** `KeccakDigest(256)` from pointycastle is Keccak-256
> (Ethereum), NOT SHA3-256 (NIST). They produce different outputs
> for the same input.


## Decision 3: EIP-55 Checksum via keccak256 of Lowercase Hex

**Why:**
EIP-55 defines a checksum for Ethereum addresses using mixed-case
hex characters. The checksum allows detecting mistyped addresses
without a separate checksum field. The algorithm:

1. Take the lowercase hex address (40 chars, no `0x`)
2. Compute `keccak256(lowercase_hex_address)` - treating the hex
   string as UTF-8 bytes
3. For each hex character at position `i`:
   - If it is a digit (`0-9`): keep as-is
   - If it is a letter (`a-f`): capitalize if nibble `i` of the
     keccak hash is `>= 8`, otherwise keep lowercase

**Official source:**
[EIP-55](https://eips.ethereum.org/EIPS/eip-55)

**Implementation:**
```dart
String get checksumAddress {
  final hex = _bytes
      .map((b) => b.toRadixString(16).padLeft(2, '0'))
      .join(); // lowercase, no 0x

  final hash = _keccak256(Uint8List.fromList(utf8.encode(hex)));
  final hashHex = hash
      .map((b) => b.toRadixString(16).padLeft(2, '0'))
      .join();

  final buffer = StringBuffer('0x');
  for (var i = 0; i < hex.length; i++) {
    final c = hex[i];
    if (int.tryParse(c) != null) {
      buffer.write(c); // digits unchanged
    } else {
      final nibble = int.parse(hashHex[i], radix: 16);
      buffer.write(nibble >= 8 ? c.toUpperCase() : c);
    }
  }
  return buffer.toString();
}
```

**Test vectors (EIP-55 official):**
```dart
// Source: eips.ethereum.org/EIPS/eip-55
test('vector 1', () {
  final addr = EvmAddress.fromHex(
    '0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed');
  expect(addr.checksumAddress,
      equals('0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed'));
});
```


## Decision 4: Two Address Formats - checksumAddress and lowercaseAddress

**Why:**
Both formats are needed in practice:

- `checksumAddress` - the standard format for displaying addresses
  to users, compatible with MetaMask, Core Wallet, and all EVM tools.
  Allows typo detection.
- `lowercaseAddress` - required by some JSON-RPC APIs and smart
  contract interactions that compare addresses as strings.
  Also used for equality comparison.

**Implementation:**
```dart
String get checksumAddress  => /* EIP-55 mixed-case with 0x */;
String get lowercaseAddress => '0x${_bytes.map(...).join()}';
```


## Decision 5: Case-insensitive Equality via lowercaseAddress

**Why:**
`0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed` and
`0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaed` are the same address.
Equality comparison normalizes to lowercase to avoid false negatives
when comparing addresses from different sources.

**Implementation:**
```dart
@override
bool operator ==(Object other) =>
    other is EvmAddress &&
    lowercaseAddress == other.lowercaseAddress;

@override
int get hashCode => lowercaseAddress.hashCode;
```


## Decision 6: toString() Returns checksumAddress

**Why:**
Unlike `PrivateKey`, `Mnemonic`, `Seed`, and `HDWallet` which always
return `[REDACTED]`, an EVM address is **public information** - it is
safe and intended to be shared. `toString()` returns the checksummed
address so logging or printing an `EvmAddress` always shows the
human-readable, checksum-verified format.

**Implementation:**
```dart
@override
String toString() => checksumAddress;
```


## Test Vectors

**EIP-55 official test vectors:**

| Input (any case) | Expected checksumAddress | Status |
|---|---|---|
| `0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaed` | `0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed` | ✅ |
| `0xfb6916095ca1df60bb79ce92ce3ea74c37c5d359` | `0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359` | ✅ |
| `0xdbf03b407c01e7cd3cbea99509d93f8dddc8c6fb` | `0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB` | ✅ |
| `0xd1220a0cf47c7b9be7a2e6ba89f429762e7b9adb` | `0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb` | ✅ |

Source: [eips.ethereum.org/EIPS/eip-55](https://eips.ethereum.org/EIPS/eip-55)

---

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| Public key input | `toRawUncompressed()` 64 bytes | EVM standard - no 0x04 prefix |
| Hash algorithm | `KeccakDigest(256)` (pointycastle) | Keccak-256, not SHA3-256 |
| Address bytes | Last 20 bytes of keccak hash | EVM standard |
| Checksum | EIP-55 mixed-case | Industry standard for EVM addresses |
| Formats | `checksumAddress` + `lowercaseAddress` | Different use cases |
| Equality | Lowercase comparison | Case-insensitive by definition |
| toString() | Returns `checksumAddress` | Address is public information |
