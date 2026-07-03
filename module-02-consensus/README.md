# Module 02 - Avalanche Consensus (Snow Family)
 
## Table of Contents
 
- [1. The consensus problem](#1-the-consensus-problem)
- [2. Traditional families and their limitations](#2-traditional-families-and-their-limitations)
- [3. The Snow family - overview](#3-the-snow-family---overview)
- [4. Slush - the base protocol](#4-slush---the-base-protocol)
- [5. Snowflake - adding memory](#5-snowflake---adding-memory)
- [6. Snowball - adding accumulated confidence](#6-snowball---adding-accumulated-confidence)
- [7. Avalanche - DAG over Snowball](#7-avalanche---dag-over-snowball)
- [8. Snowman - linear consensus for chains](#8-snowman---linear-consensus-for-chains)
- [9. Key protocol parameters](#9-key-protocol-parameters)
- [10. Security properties](#10-security-properties)
- [11. Which protocol does each chain use?](#11-which-protocol-does-each-chain-use)
- [12. Official sources](#12-official-sources)

 
## 1. The consensus problem
 
Consensus means getting a group of distributed computers to agree on a decision, even when some
of them may fail or behave maliciously.
 
In a blockchain, that translates to: all validators must agree on the state of the ledger.
Without that agreement, there is no reliable network.
 
 
## 2. Traditional families and their limitations
 
Before Avalanche, consensus protocols fell into two major categories:
 
| Family | Examples | Strength | Weakness |
|--------|----------|----------|---------|
| **Classical (BFT)** | PBFT, Tendermint | Immediate finality, deterministic | Does not scale (O(n²) messages), requires knowing all nodes |
| **Nakamoto** | Bitcoin PoW, Ethereum PoW | Scales well, open to any node | Slow probabilistic finality, high energy consumption |
 
Avalanche was built to combine the best of both: **fast finality + scalability + openness**.
 
 
## 3. The Snow family - overview
 
The Snow family is a set of protocols that evolve in complexity, each built upon the previous:
 
```
Slush  -->  Snowflake  -->  Snowball  -->  Avalanche (DAG)
                                      \-->  Snowman (linear chain)
```
 
All share the same core idea: **repeated random sampling**. Instead of each node talking to
every other node (expensive), each node queries a small random sample of peers. With enough
rounds, the network converges on a decision.
 
> **Source**: [Avalanche Consensus - docs.avax.network](https://docs.avax.network/learn/avalanche-consensus)
 
**Common properties across the Snow family:**
 
- **Leaderless:** Any node can propose and vote. There is no leader that can fail or be attacked.
- **Energy efficient:** No mining. Nodes only work when there are pending transactions - if there is no work, the protocol quiesces (rests).
- **Probabilistic BFT:** Security is probabilistic but tunable. With correct parameters, the probability of failure is astronomically small.

 
## 4. Slush - the base protocol
 
Slush is the simplest in the family and serves as the conceptual foundation. It is a binary
protocol: each node holds a preference (A or B) and repeats sampling rounds until all converge.
 
**How it works:**
 
1. Each node starts with an initial preference (e.g. "pizza" or "burger").
2. In each round, the node queries **k** random nodes for their preference.
3. If at least **α** of those responses are the same, the node switches its preference to that majority.
4. Repeats until the entire network converges.
**Limitation of Slush:** No memory. A node can flip its opinion back, making it vulnerable to
attacks. Snowflake solves this.
 
 
## 5. Snowflake - adding memory
 
Snowflake adds a consecutive confirmation counter. A node does not decide until it has seen the
same majority response **β consecutive times**.
 
**Simplified pseudocode:**
 
```
preference = my_initial_choice
consecutive_count = 0
 
while not_decided:
    query k random nodes
    if >= α respond the same:
        if that response == current_preference:
            consecutive_count++
        else:
            preference = new_response
            consecutive_count = 1
    else:
        consecutive_count = 0
 
    if consecutive_count > β:
        decide(preference)
```
 
**Advance over Slush:** A node now requires **sustained** consistency to decide. A single
isolated round is not enough.
 
**Limitation of Snowflake:** The counter resets on every preference change. It does not
remember how much confidence was accumulated before the switch. Snowball solves this.
 
 
## 6. Snowball - adding accumulated confidence
 
Snowball is the core of Avalanche's consensus mechanism. It adds a **confidence counter per
option** (not just a streak counter).
 
**The key difference:** In addition to the consecutive streak counter (as in Snowflake),
Snowball maintains a count of how many times **in total** each option has been preferred.
When switching preferences, it only does so if the new option has **more accumulated
confidence** than the previous one.
 
**Example from the official documentation:**
 
```
preference = pizza
consecutive_successes = 0
 
while not_decided:
    ask k random people their preference
    if >= α give the same response:
        preference = response_with_majority
        if preference == old_preference:
            consecutive_successes++
        else:
            consecutive_successes = 1
    else:
        consecutive_successes = 0
 
    if consecutive_successes > β:
        decide(preference)
```
 
This makes it much harder for an attacker to flip node opinions, because they would have to
overcome the accumulated confidence already built up.
 
> **Source**: [Avalanche Consensus - docs.avax.network](https://docs.avax.network/learn/avalanche-consensus)
 
 
## 7. Avalanche - DAG over Snowball
 
The Avalanche protocol (the fourth in the family) extends Snowball by applying it over a
**DAG** (Directed Acyclic Graph) structure.
 
### Why a DAG?
 
Instead of processing transactions sequentially one by one, the DAG allows processing many in
parallel. Each transaction names one or more **parent** transactions, forming the graph's edges.
 
**Two key benefits of the DAG:**
 
1. **Efficiency:** A single vote on a DAG vertex implicitly votes for all transactions on the
   path to the genesis vertex. One vote does far more work.
2. **Security:** The DAG intertwines the fate of transactions. Undoing a past decision requires
   the approval of honest nodes, similar to how Bitcoin makes rewriting old blocks costly.
### How are conflicts handled?
 
The concept of "conflict" is application-defined. In a payments system, two transactions
spending the same funds (double-spend) conflict. For each **conflict set**, Avalanche
instantiates a separate Snowball, and only one of the conflicting transactions can be accepted.
 
 
## 8. Snowman - linear consensus for chains
 
Snowman is Ava Labs' implementation of the Avalanche consensus protocol adapted for **linear
chains** (where blocks are chained one after another, as in Ethereum).
 
> **Source**: [Flow of a Single Blockchain - docs.avax.network](https://docs.avax.network/api-reference/standards/guides/blockchain-flow)
 
**Why Snowman instead of Avalanche DAG for chains?**
 
Virtual Machines that assume block linearity (like the EVM) need a protocol that produces an
ordered chain, not a graph. Snowman adapts Snowball logic for that model.
 
**Where Snowman is used:**
 
- **C-Chain** - EVM smart contracts
- **P-Chain** - validators and L1 management
**Where Avalanche DAG is used:**
 
- **X-Chain** - native asset creation and transfers

 
## 9. Key protocol parameters
 
Snow family parameters are configurable. These are the values for the Primary Network:
 
| Parameter | Symbol | Value (Primary Network) | Meaning |
|-----------|--------|------------------------|---------|
| Sample size | **k** | 20 | How many nodes are queried per round |
| Quorum size | **α** | 14 | Minimum matching responses to adopt a preference |
| Decision threshold | **β** | 20 | Consecutive successes required to finalize |
 
> **Source**: [Avalanche Consensus - docs.avax.network](https://docs.avax.network/learn/avalanche-consensus)
 
**How to read these values:**
 
- With **k=20**, each node queries only 20 out of potentially thousands of validators. Communication overhead is minimal regardless of network size.
- With **α=14** (70% of k), a qualified majority is required in each sample.
- With **β=20**, a node needs 20 consecutive rounds of agreement before finalizing. An attacker must sustain their interference consistently across many rounds - statistically very difficult.
**Parameter relationships:**
 
- Increasing **α** raises the safety threshold but lowers liveness (network tolerates more malicious nodes but may progress more slowly).
- Increasing **β** makes the decision more robust but potentially slower.

 
## 10. Security properties
 
| Property | Description |
|----------|-------------|
| **Safety** | All honest nodes eventually decide the same outcome. No permanent forks. |
| **Liveness** | The protocol always makes progress when there are pending transactions. It does not stall. |
| **Leaderless** | No coordinating node exists. Any node can propose and vote. The network is more robust against targeted attacks. |
| **Sybil resistance** | Proof-of-Stake validation prevents attackers from creating thousands of fake identities to dominate the sampling. |
| **Quiescence** | If there are no pending transactions, the protocol rests. No wasted computation (unlike PoW). |
| **Sub-second finality** | Under normal conditions, finality occurs in under 1 second. When conflicts exist, honest validators quickly cluster around one winning option. |
 
 
## 11. Which protocol does each chain use?
 
| Chain | Consensus protocol | Reason |
|-------|--------------------|--------|
| **X-Chain** | Avalanche (DAG) | Optimized for high-speed parallel asset transfers |
| **C-Chain** | Snowman | EVM assumes linear blocks; Snowman provides that ordered chain |
| **P-Chain** | Snowman | Validator coordination requires strict ordering |
 
 
## 12. Official sources
 
| Source | URL |
|--------|-----|
| Avalanche Consensus (official docs) | https://docs.avax.network/learn/avalanche-consensus |
| Flow of a Single Blockchain | https://docs.avax.network/api-reference/standards/guides/blockchain-flow |
| Avalanche L1 Configs (Snow parameters) | https://docs.avax.network/nodes/configure/avalanche-l1-configs |
| Original whitepaper (Ava Labs) | https://files.avalabs.org/papers/consensus.pdf |
 
---
 
*Module 02 of 06 - Avalanche Knowledge Base by [Nemorix Group](https://github.com/nemorixgroup)*  
*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
