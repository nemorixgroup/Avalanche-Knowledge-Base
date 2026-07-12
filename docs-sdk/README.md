# SDK Technical Decisions

This section documents the implementation decisions, technical 
rationale, and key references behind 
**[avalanche_flutter_sdk](https://github.com/nemorixgroup/avalanche-flutter-sdk)**, the first native Flutter/Dart SDK for the Avalanche network.

Every decision documented here is grounded in official Avalanche 
sources. Where relevant, implementation code and test vectors 
are included to make each decision fully verifiable.


## Why This Documentation Exists

Building a production-quality SDK for Avalanche requires making 
decisions that are not always obvious - which cryptographic library 
to use, why a specific encoding standard was chosen, how an 
algorithm was verified against the official specification. 

This documentation answers those questions for:

- **Contributors** who want to understand the rationale before 
  submitting a pull request
- **Developers** who want to verify that the SDK follows official 
  Avalanche standards
- **Auditors** who need a clear trail from specification to 
  implementation


## Structure

    docs-sdk/
      README.md          <- You are here
      phase1/            <- Architecture + Cryptography + Wallet
      phase2/            <- C-Chain Core (EVM)
      phase3/            <- Data API (Glacier) + WebSocket
      phase4/            <- P-Chain + X-Chain
      

## Phase 1 - Architecture + Cryptography + Wallet

| Feature | Description | Status |
|---------|-------------|--------|
| [secp256k1](./phase1/secp256k1/) | Key generation, curve parameters, library selection, security decisions | ✅ Done |
| [CB58](./phase1/CB58/) | Encoding standard, checksum algorithm, Base58 alphabet, test vector | ✅ Done |
| [BIP-39](./phase1/BIP-39/) | Mnemonic generation, EN/ES wordlists, entropy | 🔄 Next |
| [HD Derivation](./phase1/HD-derivation/) | BIP-44 path, key derivation from mnemonic | ⏳ Pending |
| [EVM Address](./phase1/EVM-address/) | keccak256 hash, EIP-55 checksum, C-Chain format | ⏳ Pending |
| [X/P-Chain Address](./phase1/XP-address/) | sha256 + ripemd160 + CB58, Bech32 format | ⏳ Pending |

## Phase 2 - C-Chain Core (EVM)

| Feature | Description | Status |
|---------|-------------|--------|
| [C-Chain EVM](./phase2/C-Chain-EVM/) | JSON-RPC client, endpoint selection | ⏳ Pending |
| [EIP-1559](./phase2/EIP-1559/) | Transaction format, gas fee model | ⏳ Pending |
| [ERC-20](./phase2/ERC-20/) | Token transfers, ABI encoding | ⏳ Pending |
| [Gas Estimation](./phase2/gas-estimation/) | eth_estimateGas, buffer strategy | ⏳ Pending |

## Phase 3 - Data API / Glacier

| Feature | Description | Status |
|---------|-------------|--------|
| [Glacier API](./phase3/Glacier-API/) | REST client design, pagination | ⏳ Pending |
| [WebSocket](./phase3/WebSocket/) | Stream design, auto-reconnect | ⏳ Pending |

## Phase 4 - P-Chain + X-Chain

| Feature | Description | Status |
|---------|-------------|--------|
| [P-Chain](./phase4/P-Chain/) | Staking, validators, delegation | ⏳ Pending |
| [X-Chain](./phase4/X-Chain/) | UTXO model, native asset transfers | ⏳ Pending |
| [Cross-Chain](./phase4/cross-chain/) | Export/Import atomic transactions | ⏳ Pending |

---

## Related

- [avalanche_flutter_sdk](https://github.com/nemorixgroup/avalanche-flutter-sdk) - the SDK itself  
- [Avalanche Knowledge Base](../README.md) - official source documentation
