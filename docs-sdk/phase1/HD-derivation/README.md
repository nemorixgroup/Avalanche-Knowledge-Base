# HD Key Derivation - BIP-32 + BIP-44

**Phase:** 1 - Architecture + Cryptography + Wallet  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.1.0-dev  
**SDK files:**
- `lib/src/crypto/hd/seed.dart`
- `lib/src/crypto/hd/hd_wallet.dart`

**Tests:**
- `test/src/crypto/hd/seed_test.dart`
- `test/src/crypto/hd/hd_wallet_test.dart`


## Overview

HD (Hierarchical Deterministic) wallets allow deriving millions of
private keys from a single seed. The process:

```
Mnemonic phrase
    -> PBKDF2-HMAC-SHA512 (Seed)
512-bit seed
    -> HMAC-SHA512(key="Bitcoin seed")
Master private key + Master chain code
    -> BIP-44 path derivation
Child private key (per chain, per index)
    -> PublicKey.fromPrivateKey()
Public key
    -> EvmAddress / XPAddress
Avalanche address
```

**Official sources:**
- [BIP-39 - Seed generation](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki#from-mnemonic-to-seed)
- [BIP-32 - HD wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
- [BIP-44 - Multi-account hierarchy](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
- [Avalanche derivation paths](https://support.core.app/en/articles/7004986)


## Part 1: Seed (PBKDF2-HMAC-SHA512)

### Decision 1: PBKDF2-HMAC-SHA512 with 2048 iterations

**Why:**
BIP-39 specifies PBKDF2-HMAC-SHA512 as the key derivation function
for converting a mnemonic to a seed. The parameters are fixed by
the spec: 2048 iterations, 64-byte output. The high iteration count
makes brute-force attacks computationally expensive.

**Official source:**
[BIP-39 - From mnemonic to seed](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki#from-mnemonic-to-seed)

> "To create a binary seed from the mnemonic, we use the PBKDF2
> function with a mnemonic sentence (in UTF-8 NFKD) used as the
> password and the string 'mnemonic' + passphrase (again in UTF-8
> NFKD) used as the salt. The iteration count is set to 2048 and
> HMAC-SHA512 is used as the pseudo-random function. The length of
> the derived key is 512 bits (64 bytes)."

**Implementation:**
```dart
static const int _iterations = 2048;
static const int seedLength  = 64;
static const String _saltPrefix = 'mnemonic';

static Uint8List _derive(String phrase, String passphrase) {
  final password = utf8.encode(_toNfkd(phrase));
  final salt = utf8.encode(_toNfkd(_saltPrefix + passphrase));

  final derivator = PBKDF2KeyDerivator(HMac(SHA512Digest(), 128))
    ..init(Pbkdf2Parameters(
      Uint8List.fromList(salt),
      _iterations,
      seedLength,
    ));

  return derivator.process(Uint8List.fromList(password));
}
```

### Decision 2: NFKD normalization for Spanish mnemonics

**Why:**
BIP-39 requires both the mnemonic (password) and the salt to be
in UTF-8 NFKD form before PBKDF2. For English mnemonics (pure ASCII)
this is a no-op. For Spanish mnemonics with accented characters,
NFC characters must be decomposed to NFKD before hashing - otherwise
the seed would differ from other wallets using the same Spanish phrase.

`WordlistEs.wordAt()` returns NFC (human-readable) - see BIP-39 docs.
`Seed._toNfkd()` converts back to NFKD before PBKDF2.

**Implementation:**
```dart
static String _toNfkd(String input) {
  return input
      .replaceAll('\u00e1', 'a\u0301') // á -> a + combining acute
      .replaceAll('\u00e9', 'e\u0301') // é -> e + combining acute
      .replaceAll('\u00ed', 'i\u0301') // í -> i + combining acute
      .replaceAll('\u00f3', 'o\u0301') // ó -> o + combining acute
      .replaceAll('\u00fa', 'u\u0301') // ú -> u + combining acute
      .replaceAll('\u00fc', 'u\u0308') // ü -> u + combining diaeresis
      .replaceAll('\u00f1', 'n\u0303'); // ñ -> n + combining tilde
}
```

### Decision 3: Seed.toString() Always Returns [REDACTED]

**Why:**
The seed is the root of the entire HD wallet. Any party with the seed
can derive every private key and address. Same sensitivity as the
mnemonic itself.

**Implementation:**
```dart
@override
String toString() => 'Seed[REDACTED]';
```

### Test vectors (Trezor official)

**Source:** [trezor/python-mnemonic/vectors.json](https://github.com/trezor/python-mnemonic/blob/master/vectors.json)
(passphrase = "TREZOR" for all vectors)

| Mnemonic | Expected seed (first 8 bytes) | Status |
|---|---|---|
| `abandon x11 + about` | `c55257c360c07c72...` | ✅ Verified |
| `legal winner...yellow` | `2e8905819b8723fe...` | ✅ Verified |
| `letter advice...above` | `d71de856f81a8acc...` | ✅ Verified |


## Part 2: HDWallet (BIP-32 + BIP-44)

### Decision 4: Master Key via HMAC-SHA512("Bitcoin seed")

**Why:**
BIP-32 specifies that the master key is derived from the seed using
HMAC-SHA512 with the string literal "Bitcoin seed" as the key. This
specific string is part of the standard - all compatible wallets
(Core Wallet, MetaMask, Ledger) use the same string.

**Official source:**
[BIP-32 - Master key generation](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#master-key-generation)

> "Generate a seed byte sequence S of a chosen length. Calculate
> I = HMAC-SHA512(Key = 'Bitcoin seed', Data = S)"

**Implementation:**
```dart
factory HDWallet.fromSeed(Seed seed) {
  final masterData = _hmacSha512(
    key: Uint8List.fromList('Bitcoin seed'.codeUnits),
    data: seed.bytes,
  );
  final masterKey       = masterData.sublist(0, 32);
  final masterChainCode = masterData.sublist(32, 64);
  return HDWallet._(masterKey, masterChainCode);
}
```

### Decision 5: Child Key Derivation - Hardened vs Normal

**Why:**
BIP-32 defines two types of child key derivation:

- **Hardened** (`i >= 0x80000000`): uses the private key as input.
  More secure - a compromised child key cannot derive sibling keys.
  Used for `purpose'`, `coin_type'`, and `account'` levels.

- **Normal** (`i < 0x80000000`): uses the public key as input.
  Allows public key derivation without the private key.
  Used for `change` and `index` levels.

**Official source:**
[BIP-32 - Child key derivation](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#child-key-derivation-ckd-functions)

**Implementation:**
```dart
Uint8List _deriveChildKey(Uint8List parentKey,
    Uint8List parentChainCode, int index) {
  final data = BytesBuilder();

  if (index >= _hardenedOffset) {
    // Hardened: 0x00 || ser256(kpar) || ser32(i)
    data
      ..addByte(0x00)
      ..add(parentKey);
  } else {
    // Normal: serP(point(kpar)) || ser32(i)
    final privKey = PrivateKey.fromHex(/* parentKey as hex */);
    data.add(privKey.publicKey.toCompressed());
  }

  // ser32(i) - 4 bytes big-endian
  data
    ..addByte((index >> 24) & 0xff)
    ..addByte((index >> 16) & 0xff)
    ..addByte((index >> 8) & 0xff)
    ..addByte(index & 0xff);

  final i = _hmacSha512(key: parentChainCode, data: data.toBytes());
  final il = i.sublist(0, 32);
  final ir = i.sublist(32, 64);

  // child_key = (parse256(IL) + parentKey) mod n
  final childKeyInt = (_bytesToBigInt(il) + _bytesToBigInt(parentKey))
      % _domain.n;

  return Uint8List.fromList([..._bigIntToBytes(childKeyInt), ...ir]);
}
```

### Decision 6: BIP-44 Derivation Paths for Avalanche

**Why:**
BIP-44 defines a 5-level path: `m / purpose' / coin_type' / account' / change / index`.
Avalanche uses two different coin types depending on the chain:

| Chain | Path | coin_type | Rationale |
|---|---|---|---|
| C-Chain (EVM) | `m/44'/60'/0'/0/n` | 60 | EVM-compatible, same as Ethereum |
| X-Chain | `m/44'/9000'/0'/0/n` | 9000 | Avalanche native |
| P-Chain | `m/44'/9000'/0'/0/n` | 9000 | Same path as X-Chain |

C-Chain uses `coin_type=60` (Ethereum) because it is 100% EVM-compatible.
This ensures keys derived for Avalanche C-Chain match those derived by
MetaMask and other Ethereum-compatible wallets using the same mnemonic.

X-Chain and P-Chain use `coin_type=9000` (Avalanche native), registered
in the SLIP-44 standard for Avalanche.

**Official source:**
[Core Wallet - Derivation paths](https://support.core.app/en/articles/7004986)

**Implementation:**
```dart
static const int _hardenedOffset = 0x80000000;
static const int _purpose        = 44 + _hardenedOffset; // 44'
static const int _coinTypeEvm    = 60 + _hardenedOffset; // 60'
static const int _coinTypeAvax   = 9000 + _hardenedOffset; // 9000'

PrivateKey derivePrivateKeyForCChain({int index = 0}) {
  return _derivePath([
    _purpose,
    _coinTypeEvm,
    0 + _hardenedOffset, // account 0'
    0,                   // change 0 (external)
    index,
  ]);
}

PrivateKey derivePrivateKeyForXPChain({int index = 0}) {
  return _derivePath([
    _purpose,
    _coinTypeAvax,
    0 + _hardenedOffset,
    0,
    index,
  ]);
}
```

### Decision 7: HDWallet.toString() Always Returns [REDACTED]

**Why:**
The HDWallet holds the master private key and chain code, from which
all child keys can be derived. Equivalent in sensitivity to the seed
itself.

**Implementation:**
```dart
@override
String toString() => 'HDWallet[REDACTED]';
```

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| Seed algorithm | PBKDF2-HMAC-SHA512, 2048 iter | BIP-39 spec requirement |
| Seed output | 64 bytes (512 bits) | BIP-39 spec requirement |
| NFKD normalization | Manual char substitution | Required for Spanish mnemonics |
| Master key | HMAC-SHA512("Bitcoin seed") | BIP-32 spec requirement |
| Hardened derivation | i >= 0x80000000 | purpose', coin_type', account' |
| Normal derivation | i < 0x80000000 | change, index |
| C-Chain path | m/44'/60'/0'/0/n | coin_type=60, EVM-compatible |
| X/P-Chain path | m/44'/9000'/0'/0/n | coin_type=9000, Avalanche native |
| Security | toString() = [REDACTED] | Master key = full wallet access |
