# Module 04 - Avalanche Architecture
 
## Table of Contents
 
- [1. Overview - network of networks](#1-overview---network-of-networks)
- [2. The Primary Network](#2-the-primary-network)
- [3. C-Chain - Contract Chain](#3-c-chain---contract-chain)
- [4. P-Chain - Platform Chain](#4-p-chain---platform-chain)
- [5. X-Chain - Exchange Chain](#5-x-chain---exchange-chain)
- [6. Virtual Machines (VMs)](#6-virtual-machines-vms)
- [7. Avalanche L1s - sovereign networks](#7-avalanche-l1s---sovereign-networks)
- [8. Subnets vs Avalanche L1s - the evolution](#8-subnets-vs-avalanche-l1s---the-evolution)
- [9. The Etna / Avalanche9000 upgrade](#9-the-etna--avalanche9000-upgrade)
- [10. Interoperability - ICM and Avalanche Warp Messaging](#10-interoperability---icm-and-avalanche-warp-messaging)
- [11. Official sources](#11-official-sources)

 
## 1. Overview - network of networks
 
Avalanche is not a single blockchain. It is a **network of networks**: an ecosystem where
multiple blockchains coexist, each with its own purpose, performance profile, and validator set.
 
```
Avalanche Mainnet
├── Primary Network (special L1)
│   ├── C-Chain  (EVM smart contracts)
│   ├── P-Chain  (validators and L1 management)
│   └── X-Chain  (native assets)
├── Avalanche L1 A  (e.g. DFK Chain - gaming)
├── Avalanche L1 B  (e.g. Beam - gaming)
├── Avalanche L1 C  (e.g. Dexalot - DEX)
└── ... (100+ L1s in production)
```
 
> Source: [Avalanche L1s - docs.avax.network](https://docs.avax.network/docs/avalanche-l1s)
 
**Key distinction:** Avalanche Mainnet = Primary Network + all deployed Avalanche L1s in production.
 
 
## 2. The Primary Network
 
The **Primary Network** is a special Avalanche L1 that runs the three foundational chains of
the ecosystem. Any node that wants to become a Primary Network validator must stake at least
2,000 AVAX.
 
All Primary Network validators are required to run three VMs:
 
| VM | Chain it defines |
|----|-----------------|
| **Coreth** | C-Chain |
| **Platform VM** | P-Chain |
| **Avalanche VM (AVM)** | X-Chain |
 
> Source: [Virtual Machines - build.avax.network](https://build.avax.network/docs/primary-network/virtual-machines)
 
**Bootstrap order:** When a node starts, the Manager boots the P-Chain first, and once ready,
kickstarts the C-Chain and X-Chain. The P-Chain is the coordinator of everything.
 
 
## 3. C-Chain - Contract Chain
 
The **C-Chain** is Avalanche's smart contract chain. It is an EVM implementation powered by
the **Snowman** consensus protocol.
 
| Aspect | Detail |
|--------|--------|
| VM | Coreth (fork of go-ethereum) |
| Consensus | Snowman |
| Compatibility | 100% EVM - supports Solidity and Geth API |
| Chain ID (Mainnet) | 43114 |
| Chain ID (Fuji Testnet) | 43113 |
| Fees | Paid in AVAX, burned |
 
**What you can do on the C-Chain:**
 
- Deploy Solidity smart contracts exactly as on Ethereum.
- Use any Ethereum ecosystem tooling: Hardhat, Foundry, Remix, Wagmi, ethers.js.
- Connect EVM-compatible wallets: MetaMask, Core, Rabby.
- Interact with DeFi, NFTs, DAOs, and any EVM-compatible dApp.
**Why Coreth and not the standard EVM?**
 
Coreth is a fork of go-ethereum (Geth) modified to operate under Snowman consensus instead
of Ethereum's PoW/PoS. This maintains full compatibility with the Ethereum ecosystem while
benefiting from Avalanche's sub-second finality.
 
> Source: [Primary Network - docs.avax.network](https://docs.avax.network/docs/primary-network)

 
## 4. P-Chain - Platform Chain
 
The **P-Chain** is the coordination chain for the entire Avalanche architecture. It manages
everything related to validators, staking, and the creation of new blockchains.
 
| Aspect | Detail |
|--------|--------|
| VM | Platform VM (PlatformVM) |
| Consensus | Snowman |
| Data model | UTXO (not account-based like EVM) |
 
**P-Chain responsibilities:**
 
- Registration and management of Primary Network validators.
- Creation of new blockchains and Avalanche L1s.
- Addition and removal of validators for each L1.
- Staking and delegation operations.
- Index of all validator sets in the ecosystem (including BLS public keys).
- Cross-chain coordination via ICM / Warp Messaging.
**Why is the P-Chain so critical?**
 
Every L1 in the ecosystem, no matter how sovereign, must sync the P-Chain to maintain an
up-to-date view of the validator sets across the entire network. The P-Chain is Avalanche's
"civil registry": it knows who all the validators are across every chain.
 
> Source: [Primary Network - docs.avax.network](https://docs.avax.network/docs/primary-network)
 
 
## 5. X-Chain - Exchange Chain
 
The **X-Chain** is Avalanche's native asset chain. It is designed for the creation and
transfer of **Avalanche Native Tokens** - digital assets that can have programmed behavioral
rules.
 
| Aspect | Detail |
|--------|--------|
| VM | Avalanche VM (AVM) |
| Consensus | Snowman (since Cortina upgrade, April 2023) |
| Data model | UTXO |
 
**What are Avalanche Native Tokens?**
 
Digital representations of real-world assets (stocks, bonds, commodities) with sets of rules
governing their behavior. Example: a token that "cannot be transferred until tomorrow" or
"can only be sold to KYC-verified persons".
 
**Important historical note:** Before the Cortina upgrade (April 2023), the X-Chain used the
Avalanche DAG consensus protocol. With Cortina, it was **linearized** and migrated to Snowman,
like the P-Chain and C-Chain. All three Primary Network chains now use Snowman.
 
**AVAX on X-Chain vs C-Chain:**
 
- AVAX on X-Chain: UTXO model, optimized for asset transfers.
- AVAX on C-Chain: account-based model (like Ethereum), optimized for smart contract interaction.
Moving AVAX between chains requires an atomic cross-chain transfer.
 
> Source: [Primary Network - build.avax.network](https://build.avax.network/docs/primary-network/virtual-machines)
 
 
## 6. Virtual Machines (VMs)
 
A **Virtual Machine** in Avalanche is the blueprint for a blockchain. It fully defines:
 
- The blockchain's state.
- Valid state transitions.
- Transaction rules.
- The API exposed to clients.
> Source: [Virtual Machines - build.avax.network](https://build.avax.network/docs/nodes/architecture/virtual-machines)
 
**Native VMs of the Primary Network:**
 
| VM | Chain | Description |
|----|-------|-------------|
| **Coreth** | C-Chain | Fork of go-ethereum, EVM-compatible, supports Solidity |
| **Platform VM** | P-Chain | Manages validators, staking, and L1s |
| **Avalanche VM (AVM)** | X-Chain | Manages native assets with UTXO model |
 
**VMs available for Avalanche L1s:**
 
| VM | Description |
|----|-------------|
| **Subnet-EVM** | Simplified version of Coreth for custom EVM-compatible L1s. Enables Solidity deployment with custom fees and gas limits |
| **Custom VM** | Developer implements their own VM from scratch via RPC. Can be written in any language (Go, Rust, etc.) |
 
**How does a validator install a new VM?**
 
Validators install the VM binary in AvalancheGo's plugins directory (default:
`~/.avalanchego/plugins/`). AvalancheGo discovers and loads the plugin automatically. VMs
communicate with AvalancheGo via a language-agnostic RPC protocol.
 
 
## 7. Avalanche L1s - sovereign networks
 
An **Avalanche L1** is a sovereign network with its own validator set, token economics, and
rules. It is the central concept of Avalanche's heterogeneous architecture.
 
> Source: [Avalanche L1s - build.avax.network](https://build.avax.network/docs/avalanche-l1s)
 
**Characteristics of an Avalanche L1:**
 
| Characteristic | Detail |
|---------------|---------|
| Own validator set | Sovereign, independent from the Primary Network |
| Own gas token | Can use AVAX or another native token |
| Own VM | Subnet-EVM, Custom VM, or any VM the deployer defines |
| Permissioned/Permissionless | Can require KYC, geographic licenses, minimum hardware |
| Performance isolation | Throughput does not compete with other chains |
| Multiple blockchains | One L1 can validate more than one blockchain |
| Interoperability | All L1s can communicate via ICM / Warp Messaging |
 
**Cost to be an L1 validator (post-Etna):**
 
- No longer required to stake 2,000 AVAX on the Primary Network.
- Validator pays a continuous fee of approximately 1 to 10 AVAX per month to maintain registration on the P-Chain.
- Validator only needs to meet the staking requirements defined by the L1 it validates.
**Validator Manager Contract:**
 
Each L1 manages its own validator set via a **ValidatorManager smart contract** deployed on
its own chain. This contract defines the logic for validator addition and removal, and
communicates with the P-Chain via ICM to keep the registry up to date.
 
 
## 8. Subnets vs Avalanche L1s - the evolution
 
This is a critical point for understanding the current architecture. "Subnet" is the previous
model; "Avalanche L1" is the current model since the Etna upgrade (Avalanche9000).
 
| Aspect | Subnet (previous model) | Avalanche L1 (current model) |
|--------|------------------------|------------------------------|
| Validators | Subset of Primary Network validators | Own sovereign set |
| AVAX requirement | 2,000 AVAX on Primary Network per validator | Continuous fee ~1-10 AVAX/month |
| Independence | Depended on Primary Network | Sovereign - does not need to validate the Primary Network |
| Economic barrier | High (min. ~$40,000 USD per validator at peak prices) | Low (few AVAX/month) |
| P-Chain sync | Mandatory (validators validated the Primary Network) | Sync only (do not participate in Primary Network consensus) |
 
> Source: [Avalanche9000 Upgrade - build.avax.network](https://build.avax.network/academy/avalanche-l1/permissioned-l1s/01-introduction/02-multi-chain-review#subnets)
 
**Compatibility note:** Existing Subnets are not required to convert to L1s. They can
continue operating as before. All can still use ICM/Warp Messaging.
 
 
## 9. The Etna / Avalanche9000 upgrade
 
The **Etna upgrade** (implemented as part of Avalanche9000) was the largest network upgrade
in Avalanche's history, enabled primarily by **ACP-77: Reinventing Subnets**.
 
**Main changes:**
 
1. **Sovereign validators:** L1s no longer require their validators to stake 2,000 AVAX on the Primary Network.
2. **Continuous fee:** Instead of a large upfront stake, L1 validators pay a dynamic continuous fee (~1.3 AVAX/month initially).
3. **Validator Manager via smart contract:** Each L1 can define its own validation logic via a smart contract (PoA or PoS).
4. **Lower entry barrier:** Dramatic reduction in the cost of launching a sovereign blockchain.
5. **Full isolation:** L1 validators only sync the P-Chain but do not participate in its consensus.
> Source: [Etna Upgrade - build.avax.network](https://build.avax.network/blog/etna-enhancing-sovereignty-avalanche-l1s)
 
 
## 10. Interoperability - ICM and Avalanche Warp Messaging
 
All chains in the Avalanche ecosystem - both the Primary Network and L1s - can communicate
with each other via two complementary layers:
 
### 10.1 Avalanche Warp Messaging (AWM)
 
The base cross-chain communication protocol. Allows any VM on Avalanche to send and receive
messages to/from any other VM.
 
**How it works technically:**
 
1. The source chain calls the `sendWarpMessage` precompile with the message.
2. Source chain validators sign the message with their BLS private keys.
3. An off-chain relayer aggregates the signatures into a BLS multi-signature.
4. The destination chain verifies the aggregated signatures against the source chain's validator set (stored on the P-Chain).
5. If the signing stake threshold is sufficient, the message is accepted.
> Source: [ICM Deep Dive - docs.avax.network](https://docs.avax.network/academy/avalanche-l1/interchain-messaging/08-avalanche-warp-messaging/01-avalanche-warp-messaging)
 
### 10.2 ICM - Interchain Messaging (Teleporter)
 
**ICM** (Avalanche Interchain Messaging), also known as **Teleporter**, is a smart contract
layer built on top of AWM that simplifies cross-chain application development.
 
| Layer | Description |
|-------|-------------|
| **AWM (Warp)** | Base protocol for signing and verifying cross-chain messages |
| **ICM / Teleporter** | EVM contract layer that uses AWM for high-level use cases (token transfers, arbitrary messages) |
 
 
## 11. Official sources
 
| Source | URL |
|--------|-----|
| Primary Network | https://docs.avax.network/learn/primary-network |
| Avalanche Mainnet | https://docs.avax.network/protocol/networks/mainnet |
| Virtual Machines | https://build.avax.network/docs/primary-network/virtual-machines |
| Avalanche L1s | https://build.avax.network/docs/avalanche-l1s |
| Network Architecture | https://build.avax.network/academy/avalanche-l1/avalanche-fundamentals/04-creating-an-l1/03-network-architecture |
| Avalanche9000 Upgrade | https://build.avax.network/academy/avalanche-l1/avalanche-fundamentals/03-multi-chain-architecture-intro/03a-etna-upgrade |
| Etna Upgrade Blog | https://build.avax.network/blog/etna-enhancing-sovereignty-avalanche-l1s |
| ICM Deep Dive | https://docs.avax.network/cross-chain/avalanche-warp-messaging/deep-dive |
| Warp EVM Integration | https://build.avax.network/docs/cross-chain/avalanche-warp-messaging/evm-integration |
 
---
 
*Module 04 of 06 - Avalanche Knowledge Base by [Nemorix Group](https://github.com/nemorixgroup)*  
*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
