# ERC-20 - Token Transfers (USDC/USDT)

**Phase:** 2 - C-Chain Core (EVM).  
**Status:** ✅ Implemented and verified.  
**SDK version:** v0.2.0-dev.  
**SDK files:**
- `lib/src/chains/cchain/erc20/erc20_constants.dart`
- `lib/src/chains/cchain/erc20/erc20_client.dart`

**Tests:**
- `test/src/chains/cchain/erc20/erc20_client_test.dart`

---

## Overview

ERC-20 is the standard interface for fungible tokens on EVM-compatible
blockchains. On Avalanche C-Chain, USDC and USDT are ERC-20 tokens.
Interacting with them requires:

1. **Read operations** - `eth_call` with ABI-encoded calldata
2. **Write operations** - signed EIP-1559 transaction with ABI-encoded
   calldata in the `data` field

```
Read flow:
ABI encode(selector + params)
    -> eth_call({to: tokenAddress, data: calldata})
    -> ABI decode(result)

Write flow:
ABI encode(selector + params)
    -> AvaxTransferTransaction(to: tokenAddress, value: 0, data: calldata)
    -> sign(PrivateKey)
    -> eth_sendRawTransaction(rawTx)
    -> txHash
```

**Official sources:**
- [EIP-20 - Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [Ethereum ABI Specification](https://docs.soliditylang.org/en/latest/abi-spec.html)
- [USDC contract addresses](https://developers.circle.com/stablecoins/usdc-contract-addresses)

---

## Decision 1: ABI encoding from scratch (no external library)

**Why:**
No Dart library for EVM ABI encoding had sufficient adoption at
implementation time. The encoding for ERC-20 interactions is simple
and well-defined - only two types are needed:

| Solidity type | ABI encoding |
|---|---|
| `address` | 32 bytes, left-padded with zeros |
| `uint256` | 32 bytes, big-endian |
| `string` | offset(32) + length(32) + UTF-8 data padded to 32 bytes |

Implementing these three encodings from scratch (~40 lines) is simpler
and safer than adding an unverified dependency.

**Official source:**
[Ethereum ABI - Basic types](https://docs.soliditylang.org/en/latest/abi-spec.html#types)

**Implementation:**
```dart
static Uint8List _padAddress(String address) {
  final clean = address.startsWith('0x') ? address.substring(2) : address;
  final padded = clean.padLeft(64, '0'); // 32 bytes = 64 hex chars
  return Uint8List.fromList(
    List.generate(32, (i) =>
      int.parse(padded.substring(i * 2, i * 2 + 2), radix: 16)),
  );
}

static Uint8List _padUint256(BigInt value) {
  final hex = value.toRadixString(16).padLeft(64, '0');
  return Uint8List.fromList(
    List.generate(32, (i) =>
      int.parse(hex.substring(i * 2, i * 2 + 2), radix: 16)),
  );
}
```

---

## Decision 2: Function selectors as static const List<int>

**Why:**
ERC-20 function selectors are the first 4 bytes of `keccak256` of
the function signature. They never change - they are part of the
ERC-20 standard. Storing them as `static const List<int>` avoids
recomputing them at runtime and makes them verifiable at a glance.

**Official source:**
[EIP-20 - Methods](https://eips.ethereum.org/EIPS/eip-20)

**Selectors verified:**

| Function | Signature | Selector |
|---|---|---|
| `transfer` | `transfer(address,uint256)` | `0xa9059cbb` |
| `balanceOf` | `balanceOf(address)` | `0x70a08231` |
| `approve` | `approve(address,uint256)` | `0x095ea7b3` |
| `allowance` | `allowance(address,address)` | `0xdd62ed3e` |
| `transferFrom` | `transferFrom(address,address,uint256)` | `0x23b872dd` |
| `decimals` | `decimals()` | `0x313ce567` |
| `symbol` | `symbol()` | `0x95d89b41` |

**Implementation:**
```dart
static const List<int> selectorTransfer   = [0xa9, 0x05, 0x9c, 0xbb];
static const List<int> selectorBalanceOf  = [0x70, 0xa0, 0x82, 0x31];
static const List<int> selectorApprove    = [0x09, 0x5e, 0xa7, 0xb3];
static const List<int> selectorAllowance  = [0xdd, 0x62, 0xed, 0x3e];
```

---

## Decision 3: eth_call for read operations

**Why:**
ERC-20 read functions (`balanceOf`, `decimals`, `symbol`, `allowance`)
do not modify blockchain state. `eth_call` executes them locally on
the node without broadcasting a transaction and without consuming gas.
Using `eth_sendRawTransaction` for reads would waste gas and be
incorrect.

**Official source:**
[AvalancheGo C-Chain RPC - eth_call](https://build.avax.network/docs/rpcs/c-chain)

**Implementation:**
```dart
Future<String> _ethCall(String contractAddress, Uint8List data) async {
  final hex = '0x${data.map((b) =>
    b.toRadixString(16).padLeft(2, '0')).join()}';
  return _client.ethCall(to: contractAddress, data: hex);
}
```

---

## Decision 4: value = BigInt.zero for ERC-20 write transactions

**Why:**
ERC-20 `transfer`, `approve`, and `transferFrom` move tokens tracked
in the contract's internal ledger - not native AVAX. The `value` field
of the EIP-1559 transaction must be zero. Sending AVAX alongside an
ERC-20 call would send native AVAX to the contract address, which
would be locked forever in most ERC-20 contracts.

**Implementation:**
```dart
final tx = AvaxTransferTransaction(
  // ...
  value: BigInt.zero,  // no AVAX sent - token transfer only
  data: data,          // ABI-encoded ERC-20 function call
);
```

---

## Decision 5: gasLimit via eth_estimateGas (not hardcoded)

**Why:**
Unlike AVAX transfers (always 21,000 gas), ERC-20 interactions have
variable gas costs depending on the token contract implementation.
USDC on Avalanche uses approximately 65,000 gas for a transfer, but
this can vary. Using `eth_estimateGas` with the actual calldata and
`from` address ensures the correct gas limit is used.

Passing `from` to `estimateGas` is critical - without it, the node
sees `0x000...000` as the sender and some token contracts (like USDC)
revert with "transfer from the zero address".

**Implementation:**
```dart
final senderAddress = EvmAddress.fromPublicKey(privateKey.publicKey)
    .checksumAddress;

final gasLimit = await _client.estimateGas(
  to:   contractAddress,
  from: senderAddress,   // required - prevents zero-address revert
  data: '0x${data.map(...).join()}',
);
```

---

## Decision 6: Optional gas fees (auto-resolve via eth_suggestPriceOptions)

**Why:**
For simple use cases, the developer should not need to manually
compute `maxFeePerGas` and `maxPriorityFeePerGas`. Both fields are
optional in `ERC20Client` - if not provided, they are automatically
resolved via `eth_suggestPriceOptions` using the `normal` tier.

Advanced users can override by passing explicit values.

**Implementation:**
```dart
Future<(BigInt, BigInt)> _resolveFees(
  BigInt? maxFeePerGas,
  BigInt? maxPriorityFeePerGas,
) async {
  if (maxFeePerGas != null && maxPriorityFeePerGas != null) {
    return (maxFeePerGas, maxPriorityFeePerGas);
  }
  final options = await GasEstimator(_client).getPriceOptions();
  return (
    maxFeePerGas ?? options.normal.maxFeePerGas,
    maxPriorityFeePerGas ?? options.normal.maxPriorityFeePerGas,
  );
}
```

---

## Decision 7: USDC official contract addresses from Circle docs

**Why:**
Using an incorrect USDC contract address would result in interacting
with a different (potentially malicious) token. The addresses in
`ERC20Constants` are sourced directly from Circle's official developer
documentation - the canonical source for USDC addresses.

**Official source:**
[Circle - USDC contract addresses](https://developers.circle.com/stablecoins/usdc-contract-addresses)

| Network | Address |
|---|---|
| Avalanche C-Chain Mainnet | `0xB97EF9Ef8734C71904D8002F8b6Bc66Dd9c48a6E` |
| Avalanche Fuji Testnet | `0x5425890298aed601595a70AB815c96711a31Bc65` |

---

## Decision 8: typedef ERC20Transaction = AvaxTransferTransaction

**Why:**
ERC-20 write operations use the same EIP-1559 transaction format as
AVAX transfers - the only difference is `value = 0` and a non-empty
`data` field. Rather than creating a separate class, `ERC20Transaction`
is a type alias for `AvaxTransferTransaction`. This avoids code
duplication and makes the relationship between the two explicit.

**Implementation:**
```dart
typedef ERC20Transaction = AvaxTransferTransaction;
```

---

## RLP Bug Fix

During ERC-20 implementation, a bug was found in
`AvaxTransferTransaction` where a hardcoded `Uint8List(0)` was
duplicated alongside `dataBytes` in the RLP list. This caused an
extra `0x80` byte before `accessList`, which the node rejected with:

```
rlp: expected input list for types.AccessList
```

Fix: removed the hardcoded `Uint8List(0)` from both `sign()` and
`_buildUnsignedPayload()`. Only `dataBytes` remains, which equals
`Uint8List(0)` for AVAX transfers and the actual calldata for
ERC-20 interactions.

---

## On-Chain Verification

First ERC-20 transfer from `avalanche_flutter_sdk` - Fuji Testnet:

```
TX Hash  : 0x398ac7a99a3407d1b481d7c61ba5aee298b6cc6d500ecdcb614d12a7742aaa71
Block    : 59,193,138
Status   : Success ✅
Token    : USDC (0x5425890298aed601595a70AB815c96711a31Bc65)
From     : 0xe9d70EE1dfcEd40152eA3f030d0a3918F2EE81a8
To       : 0x0fEB475783E004621b9463Aae30F33665485FafA
Amount   : 1 USDC
Time     : ~2 seconds (Snowman consensus)
Explorer : https://testnet.snowtrace.io/tx/0x398ac7a99a3407...aaa71
```

---

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| ABI encoding | From scratch | No reliable Dart library |
| address encoding | 32 bytes left-padded | Ethereum ABI spec |
| uint256 encoding | big-endian 32 bytes | Ethereum ABI spec |
| Function selectors | static const List<int> | Never change, verifiable |
| Read operations | eth_call | No gas, no state change |
| Write value | BigInt.zero | No AVAX sent with ERC-20 tx |
| Gas limit | eth_estimateGas with from | Variable per contract |
| Gas fees | Auto via eth_suggestPriceOptions | Simpler developer API |
| USDC addresses | Circle official docs | Canonical source |
| ERC20Transaction | typedef = AvaxTransferTransaction | No duplication |
