# Glacier Data API - REST Client

**Phase:** 3 - Data API (Glacier)  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.2.1-dev  
**SDK files:**  
- `lib/src/chains/cchain/glacier/glacier_client.dart`
- `lib/src/chains/cchain/glacier/models/glacier_native_balance.dart`
- `lib/src/chains/cchain/glacier/models/glacier_erc20_balance.dart`
- `lib/src/chains/cchain/glacier/models/glacier_transaction.dart`
- `lib/src/chains/cchain/glacier/models/glacier_erc20_transfer.dart`
- `lib/src/chains/cchain/glacier/models/glacier_page.dart`
- `lib/src/client/network_config.dart` (`chainId` getter)

**Tests:**
- `test/src/chains/cchain/glacier/glacier_client_test.dart`

---

## Overview

The Glacier Data API is the indexed data service run by Ava Labs for
Avalanche. The C-Chain JSON-RPC methods used so far (v0.1.1-dev) answer
questions about current state or look up a single object by hash or block
number. None of the RPC methods in the rate-limits reference lists the
history of an address. The Data API adds address-centric queries: native
and ERC-20 balances, transaction history, and ERC-20 transfer history.

```
Flutter app
    -> GlacierClient (HTTPS GET, JSON)
    -> https://glacier-api.avax.network/v1/chains/{chainId}/...
    -> indexed Avalanche data
    -> typed models (GlacierNativeBalance, GlacierTransaction, ...)
```

**Official sources:**
- [Data API - Usage Guide](https://build.avax.network/docs/api-reference/data-api/usage)
- [Data API - Getting Started](https://build.avax.network/docs/api-reference/data-api/getting-started)
- [Data API - Rate Limits](https://build.avax.network/docs/api-reference/data-api/rate-limits)
- [Get native token balance](https://build.avax.network/docs/api-reference/data-api/evm-balances/getNativeBalance)
- [List ERC-20 balances](https://build.avax.network/docs/api-reference/data-api/evm-balances/listErc20Balances)
- [List native transactions](https://build.avax.network/docs/api-reference/data-api/evm-transactions/listNativeTransactions)
- [List ERC-20 transfers](https://build.avax.network/docs/api-reference/data-api/evm-transactions/listErc20Transactions)
- [Get transaction](https://build.avax.network/docs/api-reference/data-api/evm-transactions/getTransaction)

---

## Decision 1: Reuse the http package (no new dependency)

**Why:**
The Data API is plain HTTPS GET with JSON responses. The `http` package
is already a dependency (used by `CChainClient`), so Glacier support adds
no dependency. As in `CChainClient`, the `http.Client` is injectable, which
lets the unit tests run without network access.

**Implementation:**
```dart
GlacierClient({
  required this.network,
  this.apiKey,
  http.Client? httpClient,
}) : _http = httpClient ?? http.Client();
```

---

## Decision 2: The API key is optional

**Why:**
The official rate-limits page defines a tier for requests without a key,
so the API works unauthenticated. Making the key optional lets a developer
start with zero setup and add a key later when the limits matter.

**Official source:**
[Data API - Rate Limits](https://build.avax.network/docs/api-reference/data-api/rate-limits)

| Tier | Per minute (CUs) | Per day (CUs) |
|---|---|---|
| Unauthenticated | 6,000 | 1,200,000 |
| Free (key) | 8,000 | 2,000,000 |
| Base | 10,000 | 3,750,000 |
| Growth | 14,000 | 11,200,000 |
| Pro | 20,000 | 25,000,000 |

Every endpoint the SDK uses is weighted Medium (20 CUs per request). That
is 300 requests per minute unauthenticated and 400 per minute with a free
key. Limits apply collectively across the Data and Webhooks APIs, and the
documentation states that all limits, weights, and CU values are subject
to change.

The key is sent in the `x-glacier-api-key` header, as described in the
Usage Guide. A free key is created in the Builder Hub console
(Data API Keys). The SDK never stores or logs the key; the application
owns it.

**Implementation:**
```dart
final headers = <String, String>{
  'accept': 'application/json',
  if (apiKey != null) _apiKeyHeader: apiKey!,
};
```

**Verification:**
```dart
test('omits x-glacier-api-key header when no key is provided', () async {
  mockResponse(nativeBalanceJson());
  await glacier.getAvaxBalance(sender);
  final headers = verify(
    mockHttp.get(any, headers: captureAnyNamed('headers')),
  ).captured.single as Map<String, String>;
  expect(headers.containsKey('x-glacier-api-key'), isFalse);
});
```

---

## Decision 3: Base URL and chain ID

**Why:**
The Usage Guide shows requests to `https://glacier-api.avax.network/v1`,
while the endpoint reference pages show `https://data-api.avax.network/v1`.
Both host names appear in the official documentation. The SDK uses
`glacier-api.avax.network`, which is the host verified live on Fuji
(see the verification section). The base URL lives in a single constant,
so switching host is a one-line change if Ava Labs retires one of them.

Glacier identifies EVM chains by chain ID in the path. `NetworkConfig`
exposes it as a string so `GlacierClient` does not duplicate the mapping:

| Network | Chain ID |
|---|---|
| Mainnet | `43114` |
| Fuji Testnet | `43113` |

**Implementation:**
```dart
static const String _baseUrl = 'https://glacier-api.avax.network/v1';

final response = await _get(
  '/chains/${network.chainId}/addresses/$address/balances:getNative',
);
```

---

## Decision 4: Five endpoints for the first release

**Why:**
These five cover the first Glacier use case for a wallet or remittance
app: show what an address holds and what happened to it.

| SDK method | Endpoint | Response key | Cost |
|---|---|---|---|
| `getAvaxBalance` | `GET /v1/chains/{chainId}/addresses/{address}/balances:getNative` | `nativeTokenBalance` | 20 CUs |
| `getErc20Balances` | `GET /v1/chains/{chainId}/addresses/{address}/balances:listErc20` | `erc20TokenBalances`, `nextPageToken` | 20 CUs |
| `getTransactions` | `GET /v1/chains/{chainId}/addresses/{address}/transactions:listNative` | `transactions`, `nextPageToken` | 20 CUs |
| `getErc20Transfers` | `GET /v1/chains/{chainId}/addresses/{address}/transactions:listErc20` | `transactions`, `nextPageToken` | 20 CUs |
| `getTransaction` | `GET /v1/chains/{chainId}/transactions/{txHash}` | `nativeTransaction` | 20 CUs |

ERC-721, ERC-1155 and the WebSocket or webhook features are separate
releases.

---

## Decision 5: Models follow the published response schemas

**Why:**
Unit tests with mocks only prove that the parser agrees with the fixture.
If the fixture reflects a wrong assumption, the tests pass and the parser
fails against the real API. Before release, each response model was
compared with the response example on the official reference page, and the
test fixtures were rewritten from those schemas.

That comparison found five differences between the first draft and the
published schemas:

| Area | First draft | Official schema | Resolution |
|---|---|---|---|
| Native transaction status | `status` | `txStatus` | Field is `txStatus`; `isSuccess` reads it |
| Native transaction gas price | `effectiveGasPrice` | `gasPrice` | Field is `gasPrice` |
| Single transaction envelope | `transaction` | `nativeTransaction`, next to `erc20Transfers`, `erc721Transfers`, `erc1155Transfers`, `internalTransactions` | `getTransaction` reads `nativeTransaction` |
| ERC-20 transfer items | Same shape as a native transaction | Transfer events: `logIndex`, `value`, `erc20Token {...}`; no gas fields, no status | Separate `GlacierErc20Transfer` model (Decision 6) |
| ERC-20 balance price | Numeric `price` and `priceUsd` | Object `{currencyCode, value}` with `value` as a string; no `priceUsd` | Parsed as an object (Decision 8) |

**Implementation:**
```dart
factory GlacierTransaction.fromJson(Map<String, dynamic> json) {
  return GlacierTransaction(
    txHash: json['txHash'] as String,
    blockNumber: json['blockNumber'] as String,
    blockTimestamp: json['blockTimestamp'] as int,
    from: _addressOf(json['from']),
    to: _addressOf(json['to']),
    value: json['value'] as String? ?? '0',
    gasUsed: json['gasUsed'] as String? ?? '0',
    gasPrice: json['gasPrice'] as String? ?? '0',
    txStatus: json['txStatus'] as String? ?? '',
    chainId: json['chainId'] as String? ?? '',
  );
}
```

---

## Decision 6: ERC-20 transfers are a separate model

**Why:**
`transactions:listErc20` returns token transfer events, not transactions.
Each item has `logIndex`, `value`, and an `erc20Token` object (address,
name, symbol, decimals), and carries no gas or status fields. Reusing
`GlacierTransaction` would parse without throwing but produce wrong
semantics (`isSuccess` always false, gas always zero). `GlacierErc20Transfer`
exposes what the endpoint actually returns, plus `valueInTokens` for display.

A related behavior to expect: an ERC-20 transfer sent from a wallet is a
contract call, so `transactions:listNative` shows it as a native transaction
with value 0 addressed to the token contract. The token movement itself
appears in `transactions:listErc20`.

**Implementation:**
```dart
final token =
    json['erc20Token'] as Map<String, dynamic>? ?? <String, dynamic>{};
return GlacierErc20Transfer(
  txHash: json['txHash'] as String,
  logIndex: json['logIndex'] as int? ?? 0,
  value: BigInt.parse(json['value'] as String? ?? '0'),
  tokenSymbol: token['symbol'] as String? ?? '',
  tokenDecimals: token['decimals'] as int? ?? 18,
  // ...
);
```

---

## Decision 7: Amounts stay exact (BigInt and strings)

**Why:**
The API returns amounts as decimal strings (`"balance": "2000000000000000000"`).
Token amounts and Wei values exceed the 2^53 limit where a Dart `double`
stops being exact, so balances are parsed to `BigInt` and transaction
values are kept as the original strings. The `double` getters
(`balanceInAvax`, `balanceInTokens`, `valueInAvax`, `valueInTokens`) exist
for display only and should not be used for arithmetic on money.

**Implementation:**
```dart
balance: BigInt.parse(json['balance'] as String),

double get balanceInTokens =>
    balance.toDouble() / BigInt.from(10).pow(decimals).toDouble();
```

---

## Decision 8: Optional price object parsed defensively

**Why:**
The ERC-20 balance schema returns `price` as an object
`{ "currencyCode": "usd", "value": "42.42" }` with `value` as a string,
and the field can be absent (for example, tokens without a market price).
The parser checks the type before reading, so a missing or unexpected
`price` yields `null` instead of an exception. The currency defaults to USD
on the API side, so the code is kept in `priceCurrency`.

**Implementation:**
```dart
final priceNode = json['price'];
if (priceNode is Map<String, dynamic>) {
  final rawValue = priceNode['value'];
  price = rawValue is String ? double.tryParse(rawValue) : null;
  priceCurrency = priceNode['currencyCode'] as String?;
}
```

---

## Decision 9: Token-based pagination exposed as GlacierPage

**Why:**
List endpoints return a page of items and, when more data exists, a
`nextPageToken`. Per the Usage Guide, tokens are UUIDs, are only returned
when a next page exists, and expire after 24 hours. `GlacierPage<T>` keeps
the items and the token together, with `hasNextPage` for the loop
condition, and each list method accepts `pageSize` and `pageToken`.

Page size limits from the endpoint references (default 10 on all of them):

| Endpoint | Maximum `pageSize` |
|---|---|
| `transactions:listNative` | 100 |
| `transactions:listErc20` | 100 |
| `balances:listErc20` | 500 |

**Implementation:**
```dart
static Map<String, String> _pageParams(int? pageSize, String? pageToken) {
  return <String, String>{
    if (pageSize != null) 'pageSize': pageSize.toString(),
    if (pageToken != null) 'pageToken': pageToken,
  };
}
```

**Verification:**
```dart
test('sends pageSize and pageToken as query parameters', () async {
  mockResponse({'transactions': <Map<String, dynamic>>[]});
  await glacier.getTransactions(sender, pageSize: 25, pageToken: 'tok-1');
  final uri = verify(
    mockHttp.get(captureAny, headers: anyNamed('headers')),
  ).captured.single as Uri;
  expect(uri.queryParameters['pageSize'], equals('25'));
  expect(uri.queryParameters['pageToken'], equals('tok-1'));
});
```

---

## Decision 10: Error handling

**Why:**
The Usage Guide documents the error body as
`{ "message": ..., "error": ..., "statusCode": ... }` and the 429 behavior:
the server answers 429 with `ratelimit-*` and `retry-after` headers and
recommends waiting or using exponential backoff.

- **429** raises an `AvalancheException` whose message points to the API
  key signup, since a key raises the limit.
- **Other non-200 codes** raise an `AvalancheException` with the status
  code and the `message` from the body.
- **Non-JSON bodies** (for example an HTML page from a gateway on a 502)
  must not surface as a `FormatException`, so the body is decoded inside a
  guarded helper.
- **Missing list keys** (`erc20TokenBalances`, `transactions`) are read as
  empty lists.
- **`to` is null** for contract creation and is parsed as an empty string.

**Implementation:**
```dart
if (response.statusCode == 429) {
  throw AvalancheException(
    'Glacier API rate limit exceeded. '
    'Add an API key for higher limits: '
    'build.avax.network/console/utilities/data-api-keys',
  );
}

static String _errorMessage(String body) {
  try {
    final decoded = jsonDecode(body);
    if (decoded is Map<String, dynamic>) {
      return '${decoded['message'] ?? decoded['error'] ?? 'Unknown error'}';
    }
  } on FormatException {
    // Non-JSON error body; fall through.
  }
  return 'Unknown error';
}
```

**Verification:**
```dart
test('throws AvalancheException (not FormatException) on non-JSON 502',
    () async {
  mockRawResponse('<html>Bad Gateway</html>', 502);
  expect(
    () => glacier.getAvaxBalance(sender),
    throwsA(isA<AvalancheException>()),
  );
});
```

---

## Live Verification (Fuji Testnet)

Verified by running `example/phase3/glacier_read_example.dart` against the
public API on Fuji (chain ID `43113`), without an API key, using the SDK
test wallet `0xe9d70EE1dfcEd40152eA3f030d0a3918F2EE81a8`:

| Check | Result |
|---|---|
| Native balance | 0.999 AVAX (`998999999986694560` Wei) |
| ERC-20 balance | USDC 19.0 (`0x5425890298aed601595a70AB815c96711a31Bc65`) |
| Native history, `pageSize: 3` | USDC transfer tx (value 0, addressed to the token contract), 0.001 AVAX transfer, incoming 0.5 AVAX; `hasNextPage: true` |
| Second page | 1 transaction returned using `nextPageToken` |
| ERC-20 transfers | 1 USDC sent (`0x398ac7...aa71`, 2026-10-08T18:04:24Z) and 20 USDC received (`0x1f3581...df17`, 2026-10-08T17:38:22Z) |
| `getTransaction` on `0x43f9d7...1ff2` | block 58,211,707, `isSuccess: true` |

The USDC balance matches the history (20 received minus 1 sent), and the
transaction hashes and timestamps for the AVAX and USDC transfers match
Snowtrace.

---

## Known limitations

- `getTransaction` returns only the `nativeTransaction` object. The
  `erc20Transfers`, `erc721Transfers`, `erc1155Transfers` and
  `internalTransactions` arrays in the same response are not modeled yet.
- There is no automatic retry or backoff on 429; the application decides
  how to wait, using the `retry-after` guidance from the Usage Guide.
- Optional query parameters (`blockNumber`, `startBlock`, `endBlock`,
  `currency`, `filterSpamTokens`, `contractAddresses`) are not exposed yet.
  `balances:listErc20` therefore uses the API defaults, including the spam
  token filter.
- The API key header is covered by unit tests only; the live run used the
  unauthenticated tier.

---

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| HTTP client | `http` package, injectable | Already a dependency, testable |
| API key | Optional, `x-glacier-api-key` header | Unauthenticated tier exists (6,000 CUs/min) |
| Base URL | `glacier-api.avax.network/v1` in one constant | Verified live; both hosts appear in the docs |
| Chain ID | `NetworkConfig.chainId` string | `43113` Fuji, `43114` Mainnet |
| Endpoints | 5, all 20 CUs | Balances, history, transfers, lookup |
| Models | Follow published schemas | Fixtures alone cannot catch wrong assumptions |
| ERC-20 transfers | Separate model | Transfer events differ from transactions |
| Amounts | `BigInt` and strings | Exact beyond 2^53; `double` for display only |
| Price | Defensive object parsing | Optional field, string value |
| Pagination | `GlacierPage<T>` with page token | UUID tokens, 24-hour validity |
| Errors | 429 message, guarded body decode | Gateway errors may not be JSON |
