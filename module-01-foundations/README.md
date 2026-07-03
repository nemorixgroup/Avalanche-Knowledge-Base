# Module 01 - Avalanche Foundations
 
## Table of Contents
 
- [1. What is Avalanche?](#1-what-is-avalanche)
- [2. The problem it solves](#2-the-problem-it-solves)
- [3. History and origin](#3-history-and-origin)
- [4. Ava Labs](#4-ava-labs)
- [5. Key milestones](#5-key-milestones)
- [6. Governance](#6-governance)
- [7. The heterogeneous blockchain model](#7-the-heterogeneous-blockchain-model)
- [8. Official sources](#8-official-sources)

 
## 1. What is Avalanche?
 
Avalanche is a high-performance Layer 1 platform designed for builders who need scale, speed,
and flexibility without compromise. Its architecture supports sovereign, efficient, and fully
interoperable public and private L1 blockchains - all powered by the Avalanche consensus
mechanism, achieving high throughput and near-instant transaction finality.
 
> **Source**: [Avalanche Builder Hub - Primary Network](https://docs.avax.network/docs/primary-network)
 
Unlike homogeneous networks (where all applications share the same chain), Avalanche is a
**heterogeneous network of blockchains**: each application can have its own chain with its own
rules, performance profile, and token economics.
 
**Simple analogy:**
 
Think of a traditional blockchain as a single-lane highway where all cars (transactions) share
the same space. Avalanche, in contrast, is a multi-lane highway with specialized lanes: one
for assets, one for smart contracts, one for validator coordination - and you can even build
your own private lane.
 
 
## 2. The problem it solves
 
First and second-generation blockchains face the classic trilemma: they cannot be simultaneously
secure, scalable, and decentralized. Avalanche addresses all three pillars through a novel
architectural design:
 
| Traditional problem | Avalanche solution |
|--------------------|-------------------|
| Low TPS capacity | 4,500+ TPS on the Primary Network |
| Slow finality (minutes) | Sub-second finality (~1s) |
| One chain for everything | Specialized multi-chain architecture |
| Regulatory compliance challenges | Permissioned Avalanche L1s |
| High energy consumption (PoW) | Efficient Proof-of-Stake |
 
 
## 3. History and origin
 
### 3.1 Team Rocket and the whitepaper (May 2018)
 
It all begins with a paper published anonymously on IPFS in May 2018, titled
**"Snowflake to Avalanche: A Novel Metastable Consensus Protocol for Cryptocurrencies"**,
signed by a group calling themselves **Team Rocket**.
 
The paper introduced a family of leaderless consensus protocols based on repeated random
sampling among nodes, capable of achieving ultra-fast finality without mining or classical
voting. It was a fundamental break from existing models.
 
> **Source**: [Cornell Chronicle - Blockchain startup raises $42M](https://news.cornell.edu/stories/2020/08/blockchain-startup-raises-quick-42m-first-sale)
 
### 3.2 The Cornell University connection
 
While Team Rocket published their paper, professor **Emin Gun Sirer** and two PhD candidates
at Cornell University - **Maofan "Ted" Yin** and **Kevin Sekniqi** - were working on parallel
research on scalable consensus protocols. The coincidence was not accidental: Team Rocket was
connected to Sirer's research team at Cornell.
 
### 3.3 Founding of Ava Labs (2018-2019)
 
In January 2019, Sirer left his tenured associate professor position at Cornell to co-found
**Ava Labs** with Sekniqi and Yin. The project became the first resident of Cornell's Praxis
Center for Venture Development.
 
> **Source**: [Wikipedia - Avalanche blockchain platform](https://en.wikipedia.org/wiki/Avalanche_(blockchain_platform))
 
### 3.4 Open source and launch (2020)
 
- **March 2020:** Ava Labs releases the Avalanche codebase as open-source.
- **July 15, 2020:** First public token sale of AVAX. Goal: raise $42M in 15 days. Result: sold out in under 5 hours.
- **September 2020:** Official launch of Avalanche Mainnet.

 
## 4. Ava Labs
 
Ava Labs is the technology company behind Avalanche's development. Headquartered in New York,
it was founded with the specific purpose of building blockchain infrastructure for the financial
sector.
 
### Co-founders
 
| Name | Role | Background |
|------|------|------------|
| Emin Gun Sirer | CEO | Former CS Professor at Cornell, distributed systems expert, creator of Avalanche consensus |
| Kevin Sekniqi | COO | Cornell PhD (M.S. 2018), Avalanche co-architect |
| Maofan "Ted" Yin | Chief Protocol Architect | Cornell PhD (M.S. 2019), Avalanche co-architect |
 
> **Source**: [Emin Gun Sirer - Wikipedia](https://en.wikipedia.org/wiki/Emin_G%C3%BCn_Sirer)
 
### Main Ava Labs products
 
- **Avalanche** - The primary blockchain platform.
- **AvaCloud** - Service for businesses to launch their own managed blockchains.
- **Core** - Avalanche's native wallet (transactions, trading, bridging).
- **Avalanche Builder Hub** - Official developer documentation and tooling.

 
## 5. Key milestones
 
| Date | Event |
|------|-------|
| May 2018 | Team Rocket publishes whitepaper on IPFS |
| January 2019 | Ava Labs founded by Sirer, Sekniqi and Yin |
| March 2020 | Codebase released as open-source |
| July 2020 | First public AVAX sale: $42M raised in under 5 hours |
| September 2020 | Avalanche Mainnet launch |
| November 2021 | Avalanche Rush: DeFi incentive program with $180M |
| 2021 | ACP framework (Avalanche Community Proposals) introduced |
| Etna Upgrade | Launch of Avalanche L1s (formerly Subnets) with sovereign ValidatorManager |
 
 
## 6. Governance
 
Avalanche uses an open, community-driven governance model articulated through the
**Avalanche Community Proposals (ACPs)** framework.
 
> **Source**: [Avalanche Builder Hub - ACPs](https://docs.avax.network/docs/acps)
 
### 6.1 What is an ACP?
 
An **Avalanche Community Proposal** is a concise document that introduces a change or best
practice for adoption on the Avalanche Network. ACPs must include clear technical specifications
and a compelling rationale for adoption.
 
**Anyone can propose an ACP.** The process is open and transparent.
 
### 6.2 ACP types
 
| Type | Description |
|------|-------------|
| **Standards Track** | Changes to the network's design or function (P2P protocol, P-Chain, L1 architecture) |
| **Best Practices Track** | Design patterns or common interfaces to facilitate integrations |
| **Meta Track** | Changes to the ACP process itself |
| **Subnet Track** | Changes specific to a particular Avalanche L1 |
 
### 6.3 ACP lifecycle
 
```
Idea  -->  Draft (PR)  -->  Proposed (merged)  -->  Implementable  -->  Activated
                                                                    \--> Stale
```
 
1. **Proposed** - Merged into the repository; actively discussed by the community.
2. **Implementable** - Considered "ready for implementation"; will no longer change meaningfully.
3. **Activated** - Activated on the network via a coordinated community upgrade. Once activated, it is locked.
4. **Stale** - Abandoned due to lack of support or replaced by another ACP.
### 6.4 Activation
 
Once an overwhelming majority of the network/community signals support, an ACP may be scheduled
for activation by **Avalanche Network Clients (ANCs)** - such as AvalancheGo. Ultimately,
community members adopt ACPs by running a compatible ANC.
 
### 6.5 Examples of activated ACPs
 
- **ACP-77** - Reinventing Subnets: transformed Subnets into sovereign Avalanche L1s.
- **ACP-103** - Dynamic Fees: introduced a dynamic fee mechanism to the P-Chain.
- **ACP-267** - Increased validator uptime requirement from 80% to 90%.
---
 
## 7. The heterogeneous blockchain model
 
This concept is the heart of Avalanche's architecture and is essential to understand before
diving into the following modules.
 
### 7.1 The Primary Network
 
The **Primary Network** is a special Avalanche L1 that runs three blockchains:
 
| Chain | Full name | Primary function |
|-------|-----------|-----------------|
| **C-Chain** | Contract Chain | Smart contracts (EVM-compatible) |
| **P-Chain** | Platform Chain | Validators, staking and L1 management |
| **X-Chain** | Exchange Chain | Creation and transfer of native assets (AVAX and others) |
 
> **Source**: [Avalanche Builder Hub - Primary Network](https://docs.avax.network/docs/primary-network)
 
**Important note:** Avalanche Mainnet = Primary Network + all deployed Avalanche L1s.
 
### 7.2 Avalanche L1s (formerly: Subnets)
 
An **Avalanche L1** is a sovereign network that defines its own membership rules and token
economics. It is an evolution of the original "Subnet" concept introduced with ACP-77
(Etna upgrade).
 
> **Source**: [Avalanche Builder Hub - Avalanche L1s](https://docs.avax.network/docs/avalanche-l1s)
 
**Key characteristics of an Avalanche L1:**
 
- Has its own validator set (not necessarily the same as the Primary Network).
- Can have its own native gas token.
- Its performance is isolated from the rest of the ecosystem.
- Can be public (permissionless) or private (permissioned).
- Can impose requirements on its validators (KYC, licenses, geographic location).
- Supports custom Virtual Machines (VMs).
**Simple analogy:**
 
The Primary Network is like the federal government, and Avalanche L1s are like states: each
state has its own laws and rules, but all coexist within the same country. A federal validator
(Primary Network) is not required to validate all states (L1s); they can choose only the ones
in which they want to participate.
 
### 7.3 Why does this architecture matter?
 
On homogeneous blockchains (like pure Ethereum L1), all applications compete for the same
block space. A massive NFT mint can drive up fees for a completely unrelated payments app.
 
On Avalanche, each L1's workload is **isolated** from the others. An activity spike on a gaming
L1 has zero impact on an institutional payments L1.
 
 
## 8. Official sources
 
| Source | URL | Used for |
|--------|-----|---------|
| Primary Network | https://docs.avax.network/docs/primary-network | Chain architecture, C/P/X-Chain |
| Avalanche L1s | https://docs.avax.network/docs/avalanche-l1s | Sovereign L1 model |
| ACPs | https://docs.avax.network/docs/acps | Governance process |
| Whitepapers | https://avalabs.org/whitepapers | Original consensus and platform paper |
| Cornell Chronicle | https://news.cornell.edu/stories/2020/08/blockchain-startup-raises-quick-42m-first-sale | Fundraise history |
| Wikipedia - Avalanche | https://en.wikipedia.org/wiki/Avalanche_(blockchain_platform) | Historical timeline |
 
---
 
*Module 01 of 06 - Avalanche Knowledge Base by [Nemorix Group](https://github.com/nemorixgroup)*  
*Last updated: July 2026*
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
