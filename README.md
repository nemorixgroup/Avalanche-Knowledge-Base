# Avalanche Knowledge Base

A comprehensive, in-depth technical guide to the Avalanche network - covering its architecture,
consensus mechanism, multi-chain design, token economics, development ecosystem, and real-world
use cases.

> All content in this repository is sourced exclusively from official Avalanche and Ava Labs
> documentation. See the [Sources](#official-sources) section for references.


## What is Avalanche?

Avalanche is an ultra-fast, low-latency Layer 1 blockchain platform designed for builders who need
high performance at scale. Its architecture supports sovereign, efficient, and fully interoperable
public and private L1 blockchains, all powered by the Avalanche consensus mechanism - achieving
high throughput and near-instant transaction finality.

<cite index="2-1">The network consists of three integrated blockchains on the Primary Network - the X-Chain for
creating and transferring assets, the P-Chain for coordinating validators and managing Avalanche
L1s, and the C-Chain as a smart contract platform fully compatible with the EVM.</cite>

<cite index="2-1">The total capped supply of AVAX is 720 million tokens, and a portion of transaction fees is
burned, introducing a deflationary mechanism that reduces the circulating supply over time.</cite>


## Repository Structure

```
avalanche-knowledge-base/
- README.md                      <- You are here
- module-01-foundations/         <- History, Ava Labs, origin, governance
- module-02-consensus/           <- Avalanche consensus (Snow family of protocols)
- module-03-avax/                <- Token: tokenomics, fees, staking, burning
- module-04-architecture/        <- Primary Network, P/X/C-Chain, L1s, VMs
- module-05-development/         <- APIs (JSON-RPC), tooling, SDKs, wallets
- module-06-ecosystem/           <- Use cases, L1s in production, roadmap
```


## Modules

| # | Module | Topics | Status |
|---|--------|--------|--------|
| 01 | [Foundations](./module-01-foundations/) | History, Ava Labs, origin, governance model | Pending |
| 02 | [Consensus](./module-02-consensus/) | Snow family protocols, Snowball, Snowman, finality | Pending |
| 03 | [AVAX Token](./module-03-avax/) | Tokenomics, supply cap, fee burning, staking, delegation | Pending |
| 04 | [Architecture](./module-04-architecture/) | Primary Network, X/P/C-Chain, Avalanche L1s, custom VMs | Pending |
| 05 | [Development](./module-05-development/) | JSON-RPC APIs, AvalancheJS, Avalanche CLI, tooling | Pending |
| 06 | [Ecosystem](./module-06-ecosystem/) | Use cases, institutional adoption, grants, roadmap | Pending |

## SDK Technical Decisions

This repository also documents the implementation decisions behind
**[avalanche_flutter_sdk](https://github.com/nemorixgroup/avalanche-flutter-sdk)** -
the first native Flutter/Dart SDK for the Avalanche network. Every implementation decision is grounded in the official sources 
documented in this repository - no third-party references, no unverified code.

Current status: **Phase 1 in progress**

| Feature | Status |
|---------|--------|
| secp256k1 - Key generation and curve parameters | ✅ Done |
| CB58 - Encoding standard and checksum | ✅ Done |
| BIP-39 - Mnemonic generation (EN + ES) | 🔄 Next |
| HD Derivation - BIP-44 key derivation | ⏳ Pending |
| EVM Address - keccak256 + EIP-55 | ⏳ Pending |

See [docs-sdk/](./docs-sdk/) for the full documentation by phase.  

## Key Concepts at a Glance

| Concept | Avalanche |
|---------|-----------|
| Consensus | Snow family (Avalanche + Snowman protocols) |
| Finality | Sub-second (~1s) |
| Throughput | 4,500+ TPS (Primary Network) |
| Token | AVAX (capped at 720M) |
| Fee model | Burned on-chain (deflationary) |
| EVM compatibility | Native on C-Chain |
| Smart contracts | Solidity on C-Chain |
| Custom chains | Avalanche L1s (formerly Subnets) |
| Node consensus participation | Proof-of-Stake (min. 2,000 AVAX) |
| Delegation | Min. 25 AVAX, no slashing |


## Networks

| Network | Purpose | Chain ID (C-Chain) |
|---------|---------|-------------------|
| Mainnet | Production | 43114 |
| Fuji Testnet | Development & testing | 43113 |


## Official Sources

This repository uses only the following official sources:

| Source | URL |
|--------|-----|
| Avalanche Builder Hub (Docs) | https://docs.avax.network |
| Ava Labs GitHub | https://github.com/ava-labs |
| Avalanche Whitepapers | https://avalabs.org/whitepapers |
| Avalanche Official Site | https://avax.network |
| Avalanche Network Status | https://status.avax.network |
| Avalanche Explorer | https://subnets.avax.network |


## How to Use This Repository

Each module is self-contained and can be studied independently, though the recommended order
follows the table above - from foundations to ecosystem. Every module includes:

- Conceptual explanations with simple, concrete examples
- Technical deep-dives grounded in official documentation
- References to the exact official source for every claim


## Contributing

This is a personal knowledge base maintained under the Nemorix Group organization. Contributions,
corrections, and suggestions are welcome via issues or pull requests.


## License

MIT License - see [LICENSE](./LICENSE) for details.


## Related Repositories

| Repository | Description |
|-----------|-------------|
| [Hedera-Knowledge-Base](https://github.com/nemorixgroup/Hedera-Knowledge-Base) | In-depth guide to the Hedera network |
| [XRPL-Knowledge-Base](https://github.com/nemorixgroup/XRPL-Knowledge-Base) | In-depth guide to the XRP Ledger |
| [Stellar-Knowledge-Base](https://github.com/nemorixgroup/Stellar-Knowledge-Base) | In-depth guide to the Stellar network |

## Support This Project

If this project is useful to you or your team, consider supporting its development. Every contribution helps cover infrastructure, documentation, and the time invested in building and maintaining this
open source resource for the Avalanche blockchain community. Thank you!

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Support-FFDD00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/nemorixgroupllc)
[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub-EA4AAA?logo=github-sponsors&logoColor=white)](https://github.com/sponsors/nemorixgroup)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/nemorixgroupllc)

---

*Maintained by [Nemorix Group](https://github.com/nemorixgroup) - built with official sources only.*
