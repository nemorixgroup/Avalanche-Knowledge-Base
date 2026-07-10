# Module 03 - AVAX Token
 
## Table of Contents
 
- [1. What is AVAX?](#1-what-is-avax)
- [2. Token utilities](#2-token-utilities)
- [3. Tokenomics - supply and distribution](#3-tokenomics---supply-and-distribution)
- [4. The burning mechanism](#4-the-burning-mechanism)
- [5. Staking - validators](#5-staking---validators)
- [6. Staking - delegators](#6-staking---delegators)
- [7. Rewards formula](#7-rewards-formula)
- [8. No slashing](#8-no-slashing)
- [9. AVAX denominations](#9-avax-denominations)
- [10. Primary Network parameters](#10-primary-network-parameters)
- [11. Official sources](#11-official-sources)

 
## 1. What is AVAX?
 
AVAX is the native utility token of Avalanche. It is a hard-capped, scarce asset that serves
three core functions in the ecosystem:
 
1. Paying transaction fees on the network.
2. Securing the platform through staking.
3. Serving as a basic unit of account between multiple Avalanche L1s.
> Source: [AVAX Token - build.avax.network](https://build.avax.network/docs/primary-network/avax-token)
 
 
## 2. Token utilities
 
| Utility | Description |
|---------|-------------|
| **Fee payment** | Every transaction on the Primary Network pays a fee in AVAX |
| **Staking (validators)** | Validators lock AVAX to participate in consensus and earn rewards |
| **Delegation** | Users without their own node can delegate AVAX to a validator and receive rewards |
| **Consensus weight** | The probability of being sampled in Snow is proportional to a validator's stake |
| **Gas on L1s** | Avalanche L1s that use AVAX as their gas token also burn AVAX on every transaction |
 
 
## 3. Tokenomics - supply and distribution
 
| Parameter | Value |
|-----------|-------|
| Maximum supply (cap) | 720,000,000 AVAX |
| Minted at genesis | 360,000,000 AVAX |
| Remaining to emit | 360,000,000 AVAX (via staking rewards) |
 
Initial genesis distribution:
 
| Recipient | Percentage |
|-----------|-----------|
| Public sale | 10% |
| Ava Labs (team) | 9.26% |
| Foundation | 9.26% |
| Community airdrops | 7% |
| Strategic partners | 5% |
| Other allocations | ~59.48% |
 
> Source: [AVAX Token - avax.network](https://www.avax.network/about/tokens)
 
**Key dynamic:** Staking rewards **mint** new AVAX, which partially offsets the tokens burned
by fees. While supply is far from the 720M cap, AVAX will almost always behave as an
**inflationary** asset. However, the burn mechanism acts as a permanent deflationary
counterweight.
 
 
## 4. The burning mechanism
 
Every transaction fee paid on the Primary Network is **burned** - permanently removed from
circulating supply. This applies to:
 
- Transactions on the C-Chain (EVM smart contracts).
- Transactions on the X-Chain (native asset transfers).
- Transactions on the P-Chain (validator and L1 operations).
- Any Avalanche L1 that uses AVAX as its gas token.
> Source: [AVAX Token - avax.network](https://www.avax.network/about/tokens)
 
**Important effect:** Staking reward rates are adjusted as supply approaches the 720M cap.
The closer to the cap, the lower the rate becomes. At the same time, burned fees act
independently, preventing the total supply from ever reaching the cap exactly.
 
**Why this matters for Avalanche L1s:**
 
Each L1 using AVAX as its gas token reduces circulating supply. As the ecosystem grows with
more active L1s, deflationary pressure on AVAX increases. Additionally, L1 validators must
keep AVAX staked, further reducing the liquid supply in circulation.
 
 
## 5. Staking - validators
 
A **validator** is a node that actively participates in Avalanche consensus. To become a
Primary Network validator:
 
| Requirement | Value |
|-------------|-------|
| Minimum stake | 2,000 AVAX (Mainnet) |
| Minimum stake (Fuji Testnet) | 1 AVAX |
| Minimum staking duration | 2 weeks |
| Maximum staking duration | 1 year |
| Uptime required for reward | 80% (ACP-267 will raise this to 90%) |
| Maximum validator weight | min(5x own stake, 3,000,000 AVAX) |
 
> Source: [How to Stake - build.avax.network](https://build.avax.network/docs/primary-network/validate/how-to-stake)
 
**Key process points:**
 
- Once the transaction to add a validator is issued, **parameters cannot be changed**. The stake amount, node ID, and reward address are fixed.
- Staked tokens are returned to their original addresses at the end of the validation period.
- The probability of being sampled during consensus is proportional to the validator's stake.

 
## 6. Staking - delegators
 
A **delegator** is someone who holds AVAX but does not want to (or cannot) run their own node.
They can delegate their tokens to an existing validator and receive proportional rewards.
 
| Requirement | Value |
|-------------|-------|
| Minimum delegation | 25 AVAX (Mainnet) |
| Minimum delegation (Fuji Testnet) | 1 AVAX |
| Validator fee | Defined by the validator (delegation fee rate) |
 
> Source: [Validate vs. Delegate - build.avax.network](https://build.avax.network/docs/primary-network/validate/validate-vs-delegate)
 
**How the delegation fee works:**
 
The validator keeps a percentage of the rewards generated by the delegator. Example: if the
delegator earns 10 AVAX and the fee is 2%, the validator retains 0.2 AVAX and the delegator
receives 9.8 AVAX.
 
**Important constraint:** The maximum weight of a validator (own stake + all delegated stake)
is `min(5x own stake, 3,000,000 AVAX)`. If a validator staked 2,000 AVAX, they can only
receive up to 8,000 additional AVAX in delegations total.
 
 
## 7. Rewards formula
 
The potential reward for a staker is calculated with the following formula:
 
```
Potential Reward = (MaximumSupply - Supply)
                 x (Stake / Supply)
                 x (StakingPeriod / MintingPeriod)
                 x EffectiveConsumptionRate
```
 
Where:
- `MaximumSupply - Supply` = AVAX tokens left to emit in the network.
- `Stake / Supply` = the individual's stake as a percentage of all available AVAX tokens.
- `StakingPeriod / MintingPeriod` = time locked divided by the minting period (1 year).
- `EffectiveConsumptionRate` = effective rate between MinConsumptionRate and MaxConsumptionRate based on stake duration.
> Source: [Rewards Formula - build.avax.network](https://build.avax.network/docs/quick-start/rewards-formula)
 
**Long-term incentive:** Staking for the maximum duration (1 year) generates 11.11% more
minted tokens than staking for the minimum (2 weeks). The protocol explicitly incentivizes
longer staking commitments.
 
**When rewards are paid:** Rewards are automatically sent to the reward address specified at
setup, at the end of the staking period - not during it.
 
 
## 8. No slashing
 
Avalanche **does not implement slashing**. If a validator behaves negligently or maliciously,
they do not lose their staked AVAX.
 
However, incorrect behavior does have consequences:
 
- A validator that fails to meet the required uptime (80%) **receives no reward** at the end
  of the period - neither the validator nor its delegators.
- A validator that attempts to harm the network spends its node's computing resources with no
  reward in return.
**Why no slashing?**
 
Avalanche's design considers that the opportunity cost (losing rewards) and the operational
cost (hardware, energy) are sufficient disincentives without directly penalizing capital. This
simplifies the validator experience and reduces the perceived risk of participating in the network.
 
 
## 9. AVAX denominations
 
AVAX can be expressed in different denominations, similar to how ETH uses Wei or BTC uses Satoshi:
 
| Denomination | Equivalence |
|-------------|-------------|
| 1 AVAX | 1,000,000,000 nAVAX |
| 1 nAVAX | 0.000000001 AVAX |
 
`nAVAX` is the smallest unit of AVAX. In low-level operations (APIs, transactions) all values
are expressed in nAVAX.
 
> Source: [AVAX Token - build.avax.network](https://build.avax.network/docs/primary-network/avax-token)
 
 
## 10. Primary Network parameters
 
For technical reference, these are the staking parameters configured on the Primary Network:
 
| Parameter | Value |
|-----------|-------|
| MinValidatorStake | 2,000 AVAX |
| MaxValidatorStake | 3,000,000 AVAX |
| MinDelegatorStake | 25 AVAX |
| MinStakeDuration | 2 weeks |
| MaxStakeDuration | 1 year |
| MinDelegationFee | 2% |
| MaxValidatorWeightFactor | 5x own stake |
| UptimeRequirement | 80% |
| MintingPeriod | 1 year |
 
> Source: [Rewards Formula - build.avax.network](https://build.avax.network/docs/quick-start/rewards-formula)
 
 
## 11. Official sources
 
| Source | URL |
|--------|-----|
| AVAX Token (Builder Hub) | https://build.avax.network/docs/primary-network/avax-token |
| AVAX Token (avax.network) | https://www.avax.network/about/tokens |
| Rewards Formula | https://build.avax.network/docs/quick-start/rewards-formula |
| How to Stake | https://build.avax.network/docs/primary-network/validate/how-to-stake |
| Validate vs. Delegate | https://build.avax.network/docs/primary-network/validate/validate-vs-delegate |
| Staking FAQ | https://support.avax.network/en/articles/6235660-staking-faq |
| Tokenomics FAQ | https://support.avax.network/en/articles/6912428-tokenomics-faq |
 
---
 
*Module 03 of 06 - Avalanche Knowledge Base by [Nemorix Group](https://github.com/nemorixgroup)*  
*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
