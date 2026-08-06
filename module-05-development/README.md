# Module 05 - Avalanche Development
 
## Table of Contents
 
- [1. Developer ecosystem overview](#1-developer-ecosystem-overview)
- [2. AvalancheGo - the official node](#2-avalanchego---the-official-node)
- [3. RPC endpoints - public APIs](#3-rpc-endpoints---public-apis)
- [4. AvalancheGo APIs](#4-avalanchego-apis)
- [5. Avalanche CLI - primary tooling](#5-avalanche-cli---primary-tooling)
- [6. Official Ava Labs SDKs](#6-official-ava-labs-sdks)
- [7. Ethereum ecosystem tooling on Avalanche](#7-ethereum-ecosystem-tooling-on-avalanche)
- [8. Wallets](#8-wallets)
- [9. Recommended development workflow](#9-recommended-development-workflow)
- [10. Avalanche_Flutter_SDK](#10-avalanche-flutter-sdk)
- [11. Official sources](#11-official-sources)

 
## 1. Developer ecosystem overview
 
The Avalanche developer ecosystem has two main paths depending on the use case:
 
| Path | When to use | Main tools |
|------|-------------|-----------|
| **C-Chain / EVM** | dApps, Solidity smart contracts, DeFi, NFTs | Hardhat, Foundry, ethers.js, viem, Remix |
| **Avalanche L1s** | Sovereign blockchains, custom VMs, own tokenomics | Avalanche CLI, Subnet-EVM, AvalancheGo |
 
For developers integrating Avalanche into a client application, the primary entry point is the
**C-Chain via JSON-RPC** - exactly the same interface as Ethereum/Geth.
 
 
## 2. AvalancheGo - the official node
 
**AvalancheGo** is the official Avalanche node implementation, written in Go. It is the
software every validator runs and the one that exposes all network APIs.
 
| Parameter | Value |
|-----------|-------|
| Repository | https://github.com/ava-labs/avalanchego |
| Language | Go |
| Default HTTP port | 9650 |
| Default staking port | 9651 |
 
**Connect to Mainnet:**
 
```bash
./avalanchego
```
 
**Connect to Fuji Testnet:**
 
```bash
./avalanchego --network-id=fuji
```
 
**Run a single-node local network (development):**
 
```bash
./avalanchego --network-id=local --staking-enabled=false \
  --snow-sample-size=1 --snow-quorum-size=1
```
 
> Source: [Using Pre-Built Binary - build.avax.network](https://build.avax.network/docs/nodes/run-a-node/using-binary)
 
Once bootstrapping is complete, chain endpoints are available at:
 
```
localhost:9650/ext/bc/C/rpc   <- C-Chain (EVM)
localhost:9650/ext/bc/P       <- P-Chain
localhost:9650/ext/bc/X       <- X-Chain
```
 
 
## 3. RPC endpoints - public APIs
 
Ava Labs maintains public nodes for Mainnet and Fuji Testnet, ideal for development and
testing without running a personal node.
 
### Mainnet
 
| Chain | HTTP Endpoint | Websocket Endpoint |
|-------|--------------|-------------------|
| C-Chain | `https://api.avax.network/ext/bc/C/rpc` | `wss://api.avax.network/ext/bc/C/ws` |
| P-Chain | `https://api.avax.network/ext/bc/P` | N/A |
| X-Chain | `https://api.avax.network/ext/bc/X` | N/A |
 
### Fuji Testnet
 
| Chain | HTTP Endpoint |
|-------|--------------|
| C-Chain | `https://api.avax-test.network/ext/bc/C/rpc` |
| P-Chain | `https://api.avax-test.network/ext/bc/P` |
| X-Chain | `https://api.avax-test.network/ext/bc/X` |
 
> Source: [AvalancheGo C-Chain RPC - build.avax.network](https://build.avax.network/docs/rpcs/c-chain)
 
**Important notes:**
 
- Public nodes support HTTP for X-Chain, P-Chain, and C-Chain, but **websocket is only available for C-Chain**.
- For batch requests on the public API node, the maximum is **40 items** per batch.
- For production, running a personal node or using an RPC provider (Infura, Ankr, QuickNode) is recommended.
**URL format for a custom Avalanche L1:**
 
```
http://<node-ip>:9650/ext/bc/<blockchainID>/rpc
```
 
 
## 4. AvalancheGo APIs
 
AvalancheGo exposes multiple APIs via JSON-RPC 2.0. The most relevant for developers:
 
### 4.1 C-Chain API (EVM)
 
The C-Chain exposes exactly the same API as Geth. Supported namespaces:
 
| Namespace | Enabled by default | Description |
|-----------|-------------------|-------------|
| `eth` | Yes | Standard EVM operations (balances, transactions, contracts) |
| `txpool` | No | Mempool state |
| `debug` | No | Debug tooling |
| `personal` | No | Account management (deprecated in modern Geth) |
 
**Example JSON-RPC call to C-Chain:**
 
```bash
curl -X POST https://api.avax.network/ext/bc/C/rpc \
  -H "Content-Type: application/json" \
  --data '{
    "jsonrpc": "2.0",
    "method": "eth_getBalance",
    "params": ["0xYourAddress", "latest"],
    "id": 1
  }'
```
 
### 4.2 P-Chain API (Platform)
 
Manages validators, staking, and L1s. Endpoint: `/ext/bc/P`
 
**Key methods:**
 
| Method | Description |
|--------|-------------|
| `platform.getHeight` | Current P-Chain height |
| `platform.getCurrentValidators` | Active validator list |
| `platform.getBlockchains` | List of all created blockchains |
| `platform.getBalance` | AVAX balance at a P-Chain address |
| `platform.addValidator` | Add a validator to the Primary Network |
| `platform.addDelegator` | Add a delegation to a validator |
 
### 4.3 X-Chain API (AVM)
 
Manages native assets. Endpoint: `/ext/bc/X`
 
**Key methods:**
 
| Method | Description |
|--------|-------------|
| `avm.getBalance` | Balance of an asset at an X-Chain address |
| `avm.createAsset` | Create a new Avalanche Native Token |
| `avm.send` | Transfer assets on the X-Chain |
| `avm.getAssetDescription` | Asset description by ID |
 
### 4.4 Info API
 
General node information. Endpoint: `/ext/info`
 
| Method | Description |
|--------|-------------|
| `info.getNodeID` | Node's NodeID (required for validator registration) |
| `info.getNetworkID` | Network ID (1=Mainnet, 5=Fuji) |
| `info.isBootstrapped` | Whether the node finished bootstrapping a chain |
| `info.uptime` | Node uptime (useful for validator monitoring) |
 
> Source: [Issuing API Calls - build.avax.network](https://build.avax.network/docs/rpcs/other/guides/issuing-api-calls)
 
 
## 5. Avalanche CLI - primary tooling
 
**Avalanche CLI** is the official command-line tool for developers. It centralizes the entire
Avalanche L1 development workflow.
 
> Source: [Avalanche CLI - build.avax.network](https://build.avax.network/docs/tooling/cli-commands)
 
**Installation:**
 
```bash
curl -sSfL https://raw.githubusercontent.com/ava-labs/avalanche-cli/main/scripts/install.sh | sh -s
```
 
**Main commands:**
 
```bash
# Create a new L1
avalanche blockchain create myL1
 
# Deploy locally (downloads AvalancheGo automatically)
avalanche blockchain deploy myL1 --local
 
# Deploy to Fuji Testnet
avalanche blockchain deploy myL1 --fuji
 
# Deploy to Mainnet
avalanche blockchain deploy myL1 --mainnet
 
# View active local networks
avalanche network status
 
# Stop local network
avalanche network stop
```
 
**Deployment flow with Avalanche CLI:**
 
```
avalanche blockchain create  -->  avalanche blockchain deploy  -->  wallet connect
     (configure genesis)           (Local / Fuji / Mainnet)        (MetaMask, Core)
```
 
**Important note:** Avalanche CLI only supports running one local network at a time. For
multiple simultaneous local networks, use the Avalanche Network Runner.
 
 
## 6. Official Ava Labs SDKs
 
### 6.1 Avalanche SDK TypeScript (new - recommended)
 
Ava Labs' modern SDK. Modular: install only the packages you need.
 
> Source: [Avalanche SDK Overview - build.avax.network](https://build.avax.network/docs/tooling/avalanche-sdk)
 
**Available packages:**
 
| Package | Installation | Use |
|---------|-------------|-----|
| `@avalanche-sdk/client` | `npm install @avalanche-sdk/client` | Core RPC, balances, transactions |
| `@avalanche-sdk/chainkit` | `npm install @avalanche-sdk/chainkit` | Indexed Data API (Glacier), metrics, webhooks |
| `@avalanche-sdk/interchain` | `npm install @avalanche-sdk/interchain` | ICM cross-chain, Warp messages |
 
**Basic example - get balance on C-Chain:**
 
```typescript
import { createAvalancheClient } from '@avalanche-sdk/client'
import { avalanche } from '@avalanche-sdk/client/chains'
 
const client = createAvalancheClient({
  chain: avalanche,
  transport: { type: "http" }
})
 
const balance = await client.getBalance({
  address: '0xA0Cf798816D4b9b9866b5330EEa46a18382f251e',
})
```
 
**Note:** This SDK is in developer preview (beta). Use in production at your own risk.
 
### 6.2 AvalancheJS (legacy - stable)
 
The original Ava Labs JavaScript/TypeScript library. More mature but less modular.
 
> Source: [github.com/ava-labs/avalanchejs](https://github.com/ava-labs/avalanchejs)
 
```bash
npm install avalanche
# or
yarn add avalanche
```
 
**Features:**
 
- Supports Node.js (LTS 20.11.1 or higher) and browser.
- Covers X-Chain, P-Chain, and C-Chain.
- Private key management, UTXOs, transaction building and signing.

 
## 7. Ethereum ecosystem tooling on Avalanche
 
Since the C-Chain is 100% EVM-compatible, **all Ethereum tooling works on Avalanche without
modification** - except for configuring the RPC endpoint and Chain ID.
 
### 7.1 Hardhat
 
```javascript
// hardhat.config.js
module.exports = {
  networks: {
    avalanche: {
      url: 'https://api.avax.network/ext/bc/C/rpc',
      chainId: 43114,
      accounts: [process.env.PRIVATE_KEY]
    },
    fuji: {
      url: 'https://api.avax-test.network/ext/bc/C/rpc',
      chainId: 43113,
      accounts: [process.env.PRIVATE_KEY]
    }
  }
}
```
 
### 7.2 Foundry
 
Foundry is a Rust-based smart contract development toolchain. Manages dependencies, compiles,
tests, and deploys Solidity contracts.
 
```toml
# foundry.toml
[profile.default]
evm_version = "cancun"  # Required on Avalanche (Pectra not yet supported)
 
[rpc_endpoints]
fuji-c    = "https://api.avax-test.network/ext/bc/C/rpc"
mainnet-c = "https://api.avax.network/ext/bc/C/rpc"
local-c   = "http://localhost:9650/ext/bc/C/rpc"
```
 
> Source: [Foundry - build.avax.network](https://build.avax.network/docs/dapps/toolchains/foundry)
 
**Important:** The `evm_version = "cancun"` setting is required when deploying to Avalanche.
Pectra is not yet supported.
 
### 7.3 Other compatible tools
 
| Tool | Compatibility |
|------|---------------|
| Remix IDE | Full - connect via MetaMask or Injected Provider |
| ethers.js / viem | Full - use C-Chain RPC URL |
| web3.js | Full |
| OpenZeppelin Contracts | Full |
| The Graph | Available for C-Chain |
 
 
## 8. Wallets
 
| Wallet | Platform | Description |
|--------|----------|-------------|
| **Core** | Web, Extension, Mobile | Avalanche's native wallet. Supports C-Chain, P-Chain, X-Chain, and Avalanche L1s. Developed by Ava Labs |
| **MetaMask** | Extension, Mobile | Compatible with C-Chain via manual network configuration |
| **Rabby** | Extension | Compatible with C-Chain |
| **Ledger** | Hardware | Supported via Core and MetaMask |
 
**Manual Avalanche Mainnet configuration in MetaMask:**
 
| Field | Value |
|-------|-------|
| Network Name | Avalanche C-Chain |
| RPC URL | https://api.avax.network/ext/bc/C/rpc |
| Chain ID | 43114 |
| Symbol | AVAX |
| Explorer | https://snowtrace.io |
 
**Fuji Testnet configuration in MetaMask:**
 
| Field | Value |
|-------|-------|
| Network Name | Avalanche Fuji Testnet |
| RPC URL | https://api.avax-test.network/ext/bc/C/rpc |
| Chain ID | 43113 |
| Symbol | AVAX |
| Explorer | https://testnet.snowtrace.io |
 
 
## 9. Recommended development workflow
 
For any project on Avalanche, the recommended workflow is:
 
```
1. LOCAL  -->  2. FUJI TESTNET  -->  3. MAINNET
```
 
| Stage | Network | Tooling | Funds |
|-------|---------|---------|-------|
| Development and unit testing | Local (AvalancheGo) | Avalanche CLI + Hardhat/Foundry | Pre-funded local accounts |
| Integration testing | Fuji Testnet | Avalanche CLI + MetaMask/Core | Faucet: faucet.avax.network |
| Production | Mainnet | Avalanche CLI + Ledger | Real AVAX |
 
**Fuji Testnet Faucet:**
 
```
https://faucet.avax.network
```
 
Delivers 2 testnet AVAX per request. Requires a minimum balance on Mainnet to prevent abuse.
 
## 10. Avalanche Flutter SDK
 
The **avalanche_flutter_sdk** is the first native Flutter/Dart SDK for interacting with the
Avalanche network. Developed by Nemorix Group under the verified publisher `nemorixpay.com`
on pub.dev.
 
> Repository: https://github.com/nemorixgroup/avalanche-flutter-sdk
> pub.dev: https://pub.dev/packages/avalanche_flutter_sdk
> License: Apache 2.0
 
### Design philosophy
 
- **Pure Dart** - no platform channels. Works identically on Android, iOS, macOS, Linux, and Windows.
- **Minimal dependencies** - only `http`, `meta`, and `pointycastle` (cryptography).
- **Security by design** - private keys and mnemonics are always redacted in logs.
- **LATAM-first** - native BIP-39 Spanish wordlist support from the first release.
### Installation
 
```yaml
# pubspec.yaml
dependencies:
  avalanche_flutter_sdk: ^0.1.0-dev
```
 
```sh
flutter pub get
```
 
### Current status (Phase 1 complete - v0.1.0-dev)
 
| Feature | Status |
|---------|--------|
| AvalancheClient + NetworkConfig (Mainnet / Fuji) | Done |
| secp256k1 PrivateKey / PublicKey | Done |
| CB58 encoding / decoding | Done |
| BIP-39 mnemonics (EN + ES) | Done |
| HD key derivation (BIP-44) | Done |
| EVM address derivation (C-Chain) | Done |
| X/P-Chain address derivation | Done |
| C-Chain JSON-RPC client (eth_getBalance, eth_getTransactionCount) | Milestone 2 |
| AVAX transfers (EIP-1559, signing, broadcast) | Milestone 2 |
| ERC-20 transfers (USDC, USDT, approve, allowance) | Milestone 2 |
| Glacier REST client (balances, transaction history) | Milestone 3 |
| Glacier WebSocket (real-time events, subscriptions) | Milestone 3 |
| ERC-721 / ERC-1155 (NFT metadata, ownership) | Milestone 3 |
| P-Chain staking (addValidator, addDelegator) | Milestone 4 |
| P-Chain validator queries | Milestone 4 |
| X-Chain UTXO transfers (native AVAX) | Milestone 4 |
| Cross-chain (Export/Import C-X-P) | Milestone 4 |
 
### Quick Start
 
#### Network configuration
 
```dart
import 'package:avalanche_flutter_sdk/avalanche_flutter_sdk.dart';
 
// Fuji Testnet
final client = AvalancheClient(network: NetworkConfig.fuji);
print(client.network.cChainRpcUrl);
// -> https://api.avax-test.network/ext/bc/C/rpc
 
// Mainnet
final client = AvalancheClient(network: NetworkConfig.mainnet);
print(client.network.networkId); // -> 1
```
 
#### Key generation (secp256k1)
 
```dart
final privateKey = PrivateKey.generate();
final publicKey = privateKey.publicKey;
 
// Keys are always redacted in logs
print(privateKey); // PrivateKey[REDACTED]
print(publicKey);  // PublicKey(0x02b33c...)
 
// Export / import via hex
final hex      = privateKey.toHex();
final imported = PrivateKey.fromHex(hex);
```
 
#### CB58 encoding / decoding
 
```dart
// CB58 is Avalanche's native encoding format for keys, NodeIDs, and assets
final bytes   = Uint8List.fromList([1, 2, 3, 4, 5]);
final encoded = CB58.encode(bytes);
final decoded = CB58.decode(encoded);
 
// PrivateKey in Avalanche export format: "PrivateKey-<cb58>"
final exportString = 'PrivateKey-${CB58.encode(privateKey.toBytes())}';
```
 
#### BIP-39 mnemonics (English + Spanish)
 
```dart
// Generate a 12-word English mnemonic (default)
final mnemonic = Mnemonic.generate();
print(mnemonic.phrase);
// -> "abandon ability able about above absent absorb..."
 
// Generate a 24-word Spanish mnemonic
final mnemonicEs = Mnemonic.generate(
  strength: MnemonicStrength.words24,
  wordlist: WordlistEs.instance,
);
print(mnemonicEs.phrase);
// -> "ábaco abdomen abeja abierto abogado..."
 
// Import from existing phrase (validates checksum automatically)
final imported = Mnemonic.fromPhrase('abandon abandon ... about');
 
// Mnemonics are always redacted in logs
print(mnemonic); // Mnemonic[REDACTED]
```
 
#### HD Wallet + address derivation
 
```dart
final mnemonic = Mnemonic.generate(wordlist: WordlistEs.instance);
final wallet   = HDWallet.fromMnemonic(mnemonic);
 
// C-Chain address (EVM) - compatible with Core Wallet and MetaMask
// Derivation path: m/44'/60'/0'/0/n
final cPubKey  = wallet.derivePublicKeyForCChain(index: 0);
final cAddress = EvmAddress.fromPublicKey(cPubKey);
print(cAddress.checksumAddress);  // 0x71C7656EC7ab88b098...
print(cAddress.lowercaseAddress); // 0x71c7656ec7ab88b098...
 
// X-Chain address - Avalanche native format
// Derivation path: m/44'/9000'/0'/0/n
final xpPubKey  = wallet.derivePublicKeyForXPChain(index: 0);
final xpAddress = XPAddress.fromPublicKey(xpPubKey);
print(xpAddress.xChainAddress());                               // X-avax1...
print(xpAddress.xChainAddress(network: AvalancheNetwork.fuji)); // X-fuji1...
 
// P-Chain address (same key as X-Chain, different prefix)
print(xpAddress.pChainAddress()); // P-avax1...
 
// Wallets and seeds are always redacted in logs
print(wallet); // HDWallet[REDACTED]
```
 
### Networks
 
| Network | Network ID | C-Chain RPC |
|---------|-----------|-------------|
| Mainnet | 1 | `https://api.avax.network/ext/bc/C/rpc` |
| Fuji Testnet | 5 | `https://api.avax-test.network/ext/bc/C/rpc` |
 
### Dependencies
 
| Package | Version | Purpose |
|---------|---------|---------|
| `http` | ^1.2.1 | HTTP client for RPC calls |
| `meta` | ^1.15.0 | Dart annotations |
| `pointycastle` | ^4.0.0 | Cryptographic primitives (secp256k1) |


## 11. Official sources
 
| Source | URL |
|--------|-----|
| AvalancheGo | https://github.com/ava-labs/avalanchego |
| Avalanche CLI | https://build.avax.network/docs/tooling/cli-commands |
| Avalanche SDK TypeScript | https://build.avax.network/docs/tooling/avalanche-sdk |
| AvalancheJS (legacy) | https://github.com/ava-labs/avalanchejs |
| C-Chain RPC Reference | https://build.avax.network/docs/rpcs/c-chain |
| Issuing API Calls | https://build.avax.network/docs/rpcs/other/guides/issuing-api-calls |
| Foundry Guide | https://build.avax.network/docs/dapps/toolchains/foundry |
| Run a Node | https://build.avax.network/docs/nodes/run-a-node/using-binary |
| Public API Server | https://docs.avax.network/apis/avalanchego/public-api-server |
 
---
 
*Module 05 of 06 - Avalanche Knowledge Base by [Nemorix Group](https://github.com/nemorixgroup)*
