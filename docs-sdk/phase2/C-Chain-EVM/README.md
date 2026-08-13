# C-Chain EVM - JSON-RPC Client + Gas Estimation

**Phase:** 2 - C-Chain Core (EVM)  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.1.1-dev  
**Covers:** CChainClient (read operations) + GasEstimator  

**SDK files:**
- [lib/src/chains/cchain/cchain_client.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/lib/src/chains/cchain/cchain_client.dart)
- [lib/src/chains/cchain/gas_estimator.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/lib/src/chains/cchain/gas_estimator.dart)
- [lib/src/chains/cchain/models/gas_price_option.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/lib/src/chains/cchain/models/gas_price_option.dart)
- [lib/src/chains/cchain/models/gas_price_options.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/lib/src/chains/cchain/models/gas_price_options.dart)

**Tests:**
- [test/src/chains/cchain/cchain_client_test.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/test/src/chains/cchain/cchain_client_test.dart)
- [test/src/chains/cchain/gas_estimator_test.dart](https://github.com/nemorixgroup/avalanche-flutter-sdk/blob/main/test/src/chains/cchain/gas_estimator_test.dart)


## Overview

The Avalanche C-Chain is fully EVM-compatible and exposes a JSON-RPC
API identical to Geth, plus Avalanche-specific methods. This document
covers the read layer implemented in v0.1.1-dev. Write operations
(EIP-1559 transfers) are covered in the next release.

```
Flutter app
    -> CChainClient (JSON-RPC over HTTPS)
    -> https://api.avax-test.network/ext/bc/C/rpc (Fuji)
    -> https://api.avax.network/ext/bc/C/rpc      (Mainnet)
    -> Avalanche C-Chain node
```

**Official sources:**
- [AvalancheGo C-Chain RPC](https://build.avax.network/docs/rpcs/c-chain)
- [Transaction Fees](https://build.avax.network/docs/rpcs/other/guides/txn-fees)


## Decision 1: http package for JSON-RPC (not web3dart)

**Why:**
The `http` package was already a dependency of `avalanche_flutter_sdk`
for future Glacier API calls. Reusing it for C-Chain JSON-RPC avoids
adding `web3dart` as a dependency - which is Ethereum-focused and would
add unnecessary overhead for Avalanche-specific features.

JSON-RPC over HTTPS is a simple POST request with a JSON body - no
specialized library is needed.

**Alternatives considered:**

| Library | Reason rejected |
|---|---|
| `web3dart` | Ethereum-focused, includes wallet management we already handle, unnecessary dependency |
| `dio` | Overkill for simple JSON-RPC calls, adds a heavy dependency |
| `http` ✅ | Already in dependencies, minimal, sufficient for JSON-RPC |

**Implementation:**
```dart
Future<dynamic> _call({
  required String method,
  required List<dynamic> params,
}) async {
  final body = jsonEncode({
    'jsonrpc': '2.0',
    'id': _idCounter++,
    'method': method,
    'params': params,
  });

  final response = await _http.post(
    Uri.parse(network.cChainRpcUrl),
    headers: {'Content-Type': 'application/json'},
    body: body,
  );

  if (response.statusCode != 200) {
    throw AvalancheException(
      'C-Chain RPC HTTP error: ${response.statusCode}',
    );
  }

  final json = jsonDecode(response.body) as Map<String, dynamic>;
  if (json.containsKey('error')) {
    final error = json['error'] as Map<String, dynamic>;
    throw AvalancheException(
      'C-Chain RPC error: ${error['message']} (code: ${error['code']})',
    );
  }

  return json['result'];
}
```


## Decision 2: Avalanche-specific methods alongside standard Ethereum

**Why:**
Avalanche adds several JSON-RPC methods not present in standard Geth.
These are implemented alongside the standard `eth_*` methods in the
same `CChainClient` class since they share the same endpoint
(`/ext/bc/C/rpc`) and the same HTTP client.

**Official source:**
[AvalancheGo C-Chain RPC - Avalanche Ethereum APIs](https://build.avax.network/docs/rpcs/c-chain#avalanche---ethereum-apis)

> "In addition to the standard Ethereum APIs, Avalanche offers
> eth_baseFee, eth_getChainConfig, eth_callDetailed, eth_getBadBlocks,
> eth_suggestPriceOptions, and the eth_newAcceptedTransactions
> subscription."

| Method | Type | Purpose |
|---|---|---|
| `eth_getBalance` | Standard | AVAX balance in Wei |
| `eth_getTransactionCount` | Standard | Nonce for signing |
| `eth_blockNumber` | Standard | Latest accepted block |
| `eth_getTransactionReceipt` | Standard | Tx confirmation |
| `eth_estimateGas` | Standard | Gas units for a tx |
| `eth_maxPriorityFeePerGas` | Standard | Tip in Wei |
| `eth_baseFee` | Avalanche-specific | Next block base fee |
| `eth_suggestPriceOptions` | Avalanche-specific | slow/normal/fast options |


## Decision 3: eth_suggestPriceOptions over manual calculation

**Why:**
Avalanche provides `eth_suggestPriceOptions` - an Avalanche-specific
method that returns ready-to-use `maxPriorityFeePerGas` and
`maxFeePerGas` for slow, normal, and fast speed tiers. This is
simpler and more accurate than manually computing fees from
`eth_baseFee` + `eth_maxPriorityFeePerGas`.

Manual calculation would require choosing a buffer strategy (e.g.
baseFee * 2 + tip), which is error-prone and less reliable than
using the node's own suggestions.

**Official source:**
[eth_suggestPriceOptions](https://build.avax.network/docs/rpcs/c-chain#eth_suggestpriceoptions)

**Response format (verified from official docs):**
```json
{
  "slow":   { "maxPriorityFeePerGas": "0x5d21dba00", "maxFeePerGas": "0x6fc23ac00" },
  "normal": { "maxPriorityFeePerGas": "0x2540be400", "maxFeePerGas": "0x4a817c800" },
  "fast":   { "maxPriorityFeePerGas": "0x12a05f200", "maxFeePerGas": "0x37e11d600" }
}
```

**Implementation:**
```dart
Future<GasPriceOptions> suggestPriceOptions() async {
  final result = await _call(method: 'eth_suggestPriceOptions', params: []);
  final map = Map<String, dynamic>.from(result as Map);
  return GasPriceOptions(
    slow:   _parseGasPriceOption(Map<String, dynamic>.from(map['slow'] as Map)),
    normal: _parseGasPriceOption(Map<String, dynamic>.from(map['normal'] as Map)),
    fast:   _parseGasPriceOption(Map<String, dynamic>.from(map['fast'] as Map)),
  );
}
```


## Decision 4: EIP-1559 fee formula - effectiveFee

**Why:**
The EIP-1559 effective fee formula is a critical correctness requirement.
Both `maxFeePerGas` and `baseFee + tip` must be computed correctly to
avoid overpaying or underestimating fees.

**Official source:**
[Transaction Fees - Dynamic Fee Transactions](https://build.avax.network/docs/rpcs/other/guides/txn-fees#dynamic-fee-transactions)

> "The effective gas price paid by a transaction will be
> min(gasFeeCap, baseFee + gasTipCap)."

**Key Avalanche difference vs Ethereum:**
In Ethereum, the priority fee goes to the validator. In Avalanche,
**both** the base fee and the priority fee are burned.

**Implementation:**
```dart
BigInt effectiveFee(BigInt baseFee) {
  return bigIntMin(maxFeePerGas, baseFee + maxPriorityFeePerGas);
}
```

**Verification:**
```dart
test('effectiveFee = min(maxFeePerGas, baseFee + tip)', () {
  final option = GasPriceOption(
    maxPriorityFeePerGas: BigInt.from(10000000000), // 10 Gwei
    maxFeePerGas: BigInt.from(20000000000),          // 20 Gwei
  );
  // baseFee(8) + tip(10) = 18 < maxFee(20) -> effective = 18
  expect(option.effectiveFee(BigInt.from(8000000000)),
      equals(BigInt.from(18000000000)));

  // baseFee(15) + tip(10) = 25 > maxFee(20) -> effective = 20 (capped)
  expect(option.effectiveFee(BigInt.from(15000000000)),
      equals(BigInt.from(20000000000)));
});
```


## Decision 5: avaxTransferGasLimit = 21,000

**Why:**
A simple AVAX transfer (no contract interaction) always costs exactly
21,000 gas units. This is the same value as a standard ETH transfer
in Ethereum - Avalanche C-Chain uses the same EVM intrinsic gas cost.

**Source:** Ethereum Yellow Paper - intrinsic gas cost for a transaction
without data and without contract creation.

**Implementation:**
```dart
/// Standard gas limit for a simple AVAX transfer (EIP-1559).
static const int avaxTransferGasLimit = 21000;

Future<BigInt> estimateTransferFee({GasSpeed speed = GasSpeed.normal}) async {
  final option = await getOption(speed);
  return option.maxFeePerGas * BigInt.from(avaxTransferGasLimit);
}
```


## Decision 6: Hex parsing always via BigInt

**Why:**
All JSON-RPC numeric responses are hex-encoded (e.g. `"0x5208"`).
Converting to `int` directly via `int.parse(hex, radix: 16)` would
overflow for large values like balances (AVAX balances can exceed
`2^53`, the JavaScript/Dart integer precision limit). Using
`BigInt.parse` is always safe regardless of magnitude.

**Implementation:**
```dart
static BigInt _hexToBigInt(String hex) {
  final clean = hex.startsWith('0x') ? hex.substring(2) : hex;
  return BigInt.parse(clean.isEmpty ? '0' : clean, radix: 16);
}

static int _hexToInt(String hex) => _hexToBigInt(hex).toInt();
```

> `_hexToInt` is only used for values guaranteed to fit in an `int`
> (block number, nonce). Balance and fee values always use
> `_hexToBigInt`.


## Live Verification (Fuji Testnet)

Verified by running `example/phase2/cchain_read_example.dart`
against the public Fuji Testnet endpoint:

```
Endpoint: https://api.avax-test.network/ext/bc/C/rpc
Block:    57,729,617
Balance:  659,634,504,216,688,480 Wei = 0.659635 AVAX
Nonce:    15,935
Base fee: 10 Wei (Fuji minimum - low testnet activity)
Transfer fee (normal): 3,570,000 Wei = 0.003570 nAVAX
```


## Summary

| Decision | Choice | Rationale |
|---|---|---|
| HTTP client | `http` package | Already a dependency, sufficient |
| Avalanche methods | Same endpoint as standard | `/ext/bc/C/rpc` for all |
| Gas estimation | `eth_suggestPriceOptions` | More accurate than manual calc |
| Fee formula | `min(maxFeePerGas, baseFee + tip)` | Official Avalanche spec |
| Avalanche vs Ethereum | Both fees burned | Key difference from Ethereum |
| Gas limit (transfer) | 21,000 gas units | Standard EVM intrinsic cost |
| Hex parsing | Always via `BigInt` | Safe for large balance values |
