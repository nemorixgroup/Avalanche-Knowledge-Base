# C-Chain WebSocket - Real-Time Subscriptions

**Phase:** 3 - Data API and real-time  
**Status:** ✅ Implemented and verified  
**SDK version:** v0.2.2-dev  
**SDK files:**
- `lib/src/chains/cchain/websocket/cchain_websocket_client.dart`
- `lib/src/chains/cchain/websocket/websocket_connection.dart`
- `lib/src/chains/cchain/websocket/websocket_status.dart`
- `lib/src/chains/cchain/websocket/reconnect_policy.dart`
- `lib/src/chains/cchain/websocket/models/new_head.dart`
- `lib/src/chains/cchain/websocket/models/evm_log.dart`
- `lib/src/chains/cchain/websocket/models/accepted_transaction.dart`
- `lib/src/chains/cchain/websocket/models/erc20_transfer_event.dart`
- `lib/src/utils/hex_quantity.dart`
- `lib/src/client/network_config.dart` (`cChainWsUrl` getter)
- `lib/src/chains/cchain/erc20/erc20_constants.dart` (`transferEventTopic`)

**Tests:**
- `test/src/chains/cchain/websocket/cchain_websocket_client_test.dart`
- `test/src/chains/cchain/websocket/cchain_websocket_client_erc20_test.dart`
- `test/src/chains/cchain/websocket/cchain_websocket_client_reconnect_test.dart`
- `test/src/chains/cchain/websocket/models/` (four model test files)
- `test/src/utils/hex_quantity_test.dart`
- `test/src/client/network_config_ws_test.dart`
- `test/src/chains/cchain/erc20/erc20_constants_test.dart`

---

## Overview

The JSON-RPC client (v0.1.1-dev) and the Glacier REST client (v0.2.1-dev)
answer questions by request. To learn that a payment arrived, they must be
asked again and again. The C-Chain node can instead push the news over a
WebSocket: the client opens one connection, subscribes (`eth_subscribe`)
and the node sends a notification when something happens.

```
Flutter app
    -> CChainWebSocketClient
    -> WebSocketConnection (replaceable transport)
    -> wss://api.avax-test.network/ext/bc/C/ws  (Fuji)
    -> wss://api.avax.network/ext/bc/C/ws       (Mainnet)
    -> eth_subscribe -> notifications -> typed models -> Stream
```

**Official sources:**
- [AvalancheGo C-Chain RPC](https://build.avax.network/docs/rpcs/c-chain)
- [C-Chain Configs - Available APIs](https://build.avax.network/docs/nodes/chain-configs/primary-network/c-chain)
- [Public API server](https://docs.avax.network/api-reference/rpc-providers)
- [Webhooks API](https://build.avax.network/docs/api-reference/webhook-api)
- [web_socket_channel (Dart team)](https://dart.googlesource.com/web_socket_channel.git/+/429e7d9/lib/io.dart)
- [dart:io WebSocket](https://api.dart.dev/stable/2.8.0/dart-io/WebSocket-class.html)

---

## Decision 1: The node WebSocket, not webhooks

**Why:**
Avalanche offers two real-time mechanisms, and only one fits a client
library.

| | Node WebSocket | Webhooks API |
|---|---|---|
| Model | The client keeps a connection open and subscribes | Avalanche sends an HTTP POST to a URL you register |
| Runs in a Flutter app | Yes, it is a client | No: needs a backend with a public HTTPS URL |
| Authentication | None on the public server | API key of the Data API |
| Coverage | C-Chain | C-Chain, L1s and P/X chains |

The SDK is a client library, so it uses the node WebSocket. A webhook
management client would only help server-side code and is left for a
separate release.

**Official source:**
[Webhooks API](https://build.avax.network/docs/api-reference/webhook-api)

---

## Decision 2: Endpoints

**Why:**
The C-Chain serves its Ethereum methods over HTTP and WebSocket. The
WebSocket path is `/ext/bc/C/ws`, and the official page shows the Mainnet
public URL. The Fuji URL appears on the public API server page of the older
documentation site, and was confirmed live: the SDK connected to it in
418 ms. The Mainnet public WebSocket accepted subscriptions in a live run.

| Network | URL |
|---|---|
| Mainnet | `wss://api.avax.network/ext/bc/C/ws` |
| Fuji | `wss://api.avax-test.network/ext/bc/C/ws` |

**Official source:**
[AvalancheGo C-Chain RPC - WebSocket Endpoints](https://build.avax.network/docs/rpcs/c-chain)

**Implementation:**
```dart
String get cChainWsUrl => networkId == 1
    ? 'wss://api.avax.network/ext/bc/C/ws'
    : 'wss://api.avax-test.network/ext/bc/C/ws';
```

**Verification:**
```dart
test('WebSocket host matches the C-Chain RPC host of the same network', () {
  for (final network in [NetworkConfig.mainnet, NetworkConfig.fuji]) {
    expect(
      Uri.parse(network.cChainWsUrl).host,
      equals(Uri.parse(network.cChainRpcUrl).host),
    );
  }
});
```

---

## Decision 3: Three subscription kinds

**Why:**
The C-Chain configuration reference lists a `subscription` API, enabled by
default, that serves `eth_subscribe` for `logs`, `newHeads`,
`newPendingTransactions` and the Avalanche-specific
`newAcceptedTransactions`. All of them work only over the WebSocket
endpoint.

| SDK method | Subscription | Delivers |
|---|---|---|
| `newHeads()` | `newHeads` | `NewHead` |
| `logs({address, topics})` | `logs` | `EvmLog` |
| `watchIncomingErc20Transfers(...)` | `logs` with a filter | `Erc20TransferEvent` |
| `newAcceptedTransactions()` | `newAcceptedTransactions`, `fullTx: true` | `AcceptedTransaction` |
| `newAcceptedTransactionHashes()` | `newAcceptedTransactions`, `fullTx: false` | `String` (hash) |

`newPendingTransactions` is not supported. The C-Chain page states that
the node has no accounts and that `eth_pendingTransactions` always returns
an empty list, and the behavior of that subscription on the public node was
not verified.

`newAcceptedTransactions` fires when a transaction is accepted, before the
block is executed (continuous execution, ACP-194). It has two methods rather
than a boolean parameter because the result type changes with `fullTx`: a
full transaction or a bare hash.

**Official source:**
[C-Chain Configs - Available APIs](https://build.avax.network/docs/nodes/chain-configs/primary-network/c-chain)

---

## Decision 4: A replaceable transport

**Why:**
The client never touches a WebSocket package directly. It talks to a small
interface, and only one file connects that interface to the package.

```dart
abstract interface class WebSocketConnection {
  Stream<String> get messages;
  void send(String message);
  int? get closeCode;
  String? get closeReason;
  Future<void> close();
}
```

Two reasons:
- **Tests without network.** The tests play the role of the node with a
  scripted fake connection and use frames copied from live runs.
- **The dependency stays contained.** The default transport uses
  `web_socket_channel`, a package from the Dart team that wraps the
  `dart:io` WebSocket. Since version 3.0.0 `WebSocketChannel` is an abstract
  interface, and `IOWebSocketChannel.connect` accepts `pingInterval` and
  `connectTimeout`. The package was already in the dependency tree as a
  transitive dependency, and the SDK now declares `^3.0.3` directly.

The alternative was using `dart:io` directly, with no new dependency but
with an abstraction written by hand for the tests. The package was chosen
for being maintained by the Dart team and for its testability.

**Official sources:**
[web_socket_channel source](https://dart.googlesource.com/web_socket_channel.git/+/429e7d9/lib/io.dart),
[dart:io WebSocket](https://api.dart.dev/stable/2.8.0/dart-io/WebSocket-class.html)

---

## Decision 5: Each subscription is a lazy Stream

**Why:**
A subscription starts when someone listens to the stream and ends when the
listener cancels, which is the natural contract of a Dart `Stream`.
Nothing is sent to the node until `listen` is called, and cancelling sends
`eth_unsubscribe`.

`cancel()` does not wait for the node. The first version waited for the
`eth_unsubscribe` answer, and a node that never answers would have left
`cancel()` pending forever. Now the subscription is removed from the client
at once and the unsubscribe goes out in the background; a failed unsubscribe
raises nothing.

**Verification:**
```dart
test('cancel() completes without waiting for the node', () async {
  // The node never answers the eth_unsubscribe, and cancel() still ends.
  await heads.subscription.cancel().timeout(const Duration(seconds: 1));
  expect(first.request(1)['method'], equals('eth_unsubscribe'));
});
```

---

## Decision 6: Responses are matched by id

**Why:**
The node can answer requests in any order. In the first live run, the
`logs` request (sent second) was answered before `newHeads`. Matching by
position would have attached the wrong subscription id to a stream. Every
request carries an increasing JSON-RPC `id`, and the response is matched by
it.

The handler of a subscription is registered in the same instant the
response frame is processed. In the live run the first notification arrived
44 ms after the subscription was confirmed, and registering one tick later
would lose it.

**Verification:**
```dart
test('responses that arrive out of order reach the right stream', () async {
  // The node answered the logs request (id 2) before newHeads (id 1).
  fake..respond(2, logsId)..respond(1, headsId);
  // ...each stream receives only its own notifications.
});
```

---

## Decision 7: Models keep what is stable and preserve the rest

**Why:**
The notification shapes were captured from live Fuji frames and used as
test fixtures.

- **`NewHead`** types a stable subset (`number`, `hash`, `parentHash`,
  `timestamp`, `gasUsed`, `gasLimit`, `baseFeePerGas`) and keeps the full
  header in `raw`. The live headers carry Avalanche-specific fields
  (`settledHeight`, `timestampMilliseconds`, `extDataHash`,
  `minDelayExcess`, among others), and the official C-Chain page notes that
  block headers change from the Helicon upgrade. Nothing is lost when a
  field is added or changed.
- **`AcceptedTransaction`** handles the two transaction shapes seen live:
  type `0x0` (legacy: `gasPrice`, `v`, `r`, `s`) and type `0x2` (adds
  `maxFeePerGas`, `maxPriorityFeePerGas`, `accessList`, `yParity`). The fee
  fields are optional. `to` is `null` for contract creation: the first live
  frame was a contract creation with about 4 KB of calldata.
- **`EvmLog`** exposes `removed`, a field of the log object (it was `false`
  in the live frame), so a handler can honor it.

All JSON-RPC numbers are hex strings. `hexToBigInt` is used for amounts and
fees, and `hexToInt` only for block numbers, indexes and nonces, as in
`CChainClient`.

**Verification:**
```dart
test('a null "to" means contract creation', () {
  final tx = AcceptedTransaction.fromJson(legacyCreation());
  expect(tx.to, isNull);
  expect(tx.isContractCreation, isTrue);
});
```

---

## Decision 8: Incoming ERC-20 transfers are filtered by the node

**Why:**
An ERC-20 transfer emits a `Transfer(address,address,uint256)` log: the
sender and the recipient are indexed topics and the amount is the data.
`logs` accepts a filter by contract and by topic position, so the node
sends only the transfers to one recipient.

```
topics[0] = keccak256("Transfer(address,address,uint256)")   (the event)
topics[1] = null                                            (any sender)
topics[2] = 0x + 24 zeros + recipient address, lowercase    (the recipient)
```

The `Transfer` topic is not hardcoded from memory. It was computed with the
SDK's own keccak256 and matched the `topics[0]` of the live USDC log. A
test recomputes it on every run.

The ERC-721 `Transfer` event has the same topic 0 but four topics. The
decoder requires exactly three topics and a 32-byte `data`, so a log from an
NFT contract does not produce invented amounts: it is delivered as a
`FormatException` on the stream, and the stream stays active.

A native AVAX transfer emits no log, so this helper does not see it. For
native payments see Decision 9.

**Implementation:**
```dart
Stream<Erc20TransferEvent> watchIncomingErc20Transfers({
  required String tokenAddress,
  required String to,
}) {
  final token = _checkedAddress(tokenAddress, 'tokenAddress');
  final recipient = _checkedAddress(to, 'to');
  final recipientTopic =
      '0x${'0' * 24}${recipient.substring(2).toLowerCase()}';
  return logs(
    address: token,
    topics: <String?>[ERC20Constants.transferEventTopic, null, recipientTopic],
  ).map(Erc20TransferEvent.fromLog);
}
```

**Verification:**
```dart
test('equals keccak256 of the Transfer event signature', () {
  expect(
    ERC20Constants.transferEventTopic,
    equals(keccakHex('Transfer(address,address,uint256)')),
  );
});
```

---

## Decision 9: newAcceptedTransactions has a real cost

**Why:**
This subscription carries every transaction accepted on the chain, not
only yours. Native AVAX payments have to be detected by filtering it.
One-minute measurements, with both modes subscribed on the same socket so
they saw the same transactions:

| Network | `fullTx` | Frames | KB/s | Average | Maximum |
|---|---|---|---|---|---|
| Mainnet | `true` | 228 (3.80/s) | 5.76 | 1,551 B | 11,753 B |
| Mainnet | `false` | 228 (3.80/s) | 0.69 | 187 B | 187 B |
| Fuji | `true` | 34 (0.57/s) | 1.70 | 3,081 B | 18,791 B |
| Fuji | `false` | 34 (0.57/s) | 0.10 | 187 B | 187 B |

Extrapolating the Mainnet minute, `fullTx: true` is about 486 MB per day
and hashes only about 58 MB per day. It is a single sample and it leaves out
TLS overhead, so it is an order of magnitude and not a firm figure.

Hashes alone cannot tell whether a transaction involves your address, so
detecting a payment needs the full transaction (or a lookup per hash, about
3.8 per second on Mainnet). The SDK exposes both modes, documents the cost
and recommends `watchIncomingErc20Transfers` for token payments.

---

## Decision 10: Heartbeat

**Why:**
A connection can die without the application noticing. The default
transport sends a ping every `pingInterval` (default 20 s); if the answer
does not arrive within that interval, `dart:io` closes the connection and
the client treats it as lost. `connectTimeout` (default 15 s) bounds the
time spent opening the connection.

Verified live on Fuji: a connection with a single `logs` subscription on a
quiet address stayed open for 303 s with a 20 s ping and no frames, and the
final `eth_unsubscribe` returned `true`.

Silence is not death. In a live run, two consecutive blocks on Fuji were 16.7 s
apart (the numbers were consecutive, so no frame was lost). The cause was
not verified, which is one more reason to rely on the heartbeat and not on
the absence of events.

**Official source:**
[dart:io WebSocket - pingInterval](https://api.dart.dev/stable/2.8.0/dart-io/WebSocket-class.html)

---

## Decision 11: Reconnection is opt-in

**Why:**
Without a policy the client behaves simply: when the connection is lost,
every active stream receives an `AvalancheException` and ends. With a
`ReconnectPolicy` the client retries with exponential backoff and keeps the
streams open.

| Option | Default | Meaning |
|---|---|---|
| `initialDelay` | 1 s | Wait before the first attempt |
| `maxDelay` | 30 s | The delay doubles up to this cap |
| `maxAttempts` | null | Failed attempts before giving up (null: keep trying) |

The default delays are 1, 2, 4, 8, 16, 30, 30 s. On reconnection, every
active subscription is sent again with the same parameters; the node assigns
new ids. `status` reports `connected`, `reconnecting`, `reconnected` and
`disconnected`. `reconnected` arrives only after the node confirms the
subscriptions. `close()` stops any reconnection in progress, a stream
cancelled while reconnecting is not subscribed again, and a stream created
while reconnecting is subscribed once the connection is back.

Events emitted while the connection was down are not replayed. The right
source for the gap depends on the application, so the SDK signals it with
`reconnected` and leaves the fill to the app: `eth_getLogs` or the
`GlacierClient`.

**Verification:**
```dart
test('reconnects, re-subscribes with the same filter and keeps the stream open',
    () async {
  // The same eth_subscribe request goes out on the new connection, and the
  // status events are connected, reconnecting, reconnected.
});
```

---

## Decision 12: Errors do not kill healthy subscriptions

**Why:**

| Situation | Result |
|---|---|
| A notification that cannot be parsed | Error event on that stream; the subscription stays active |
| A frame that is not JSON, or not an object | Ignored |
| `eth_subscribe` rejected by the node | Error event, then the stream ends |
| Connection lost, no policy | `AvalancheException`, then the streams end |
| `close()` | The streams end without an error |

---

## Live Verification

| Check | Result |
|---|---|
| Fuji WebSocket | Connected in 418 ms |
| The three subscriptions | Confirmed, ids returned |
| `logs` filter on the node | 1 frame, the expected transfer, `removed: false` |
| Raw-frame latency | The transaction showed up 1,777 ms after it was sent |
| Latency through `watchIncomingErc20Transfers` | 1,954 ms, same transaction hash |
| `eth_unsubscribe` | `true` |
| Idle connection | 303 s open with a 20 s ping |
| Mainnet | Public WebSocket accepted the subscriptions |
| Block interval on Fuji | About 2 s (17 headers in about 35 s) |

---

## Known limitations

- Reconnection is covered by unit tests with a simulated transport; it was
  not exercised against a real network drop.
- A silent drop is detected within one to two heartbeat intervals (20 to
  40 s by default).
- Events emitted while disconnected are not replayed.
- The public API server documents no WebSocket rate limits.
- `newPendingTransactions` is not supported.
- There is no dedicated helper for native AVAX payments; use
  `newAcceptedTransactions` and filter by `to`.

---

## Summary

| Decision | Choice | Rationale |
|---|---|---|
| Real-time source | Node WebSocket | Client connection; webhooks need a backend |
| Endpoints | `NetworkConfig.cChainWsUrl` | Mainnet and Fuji verified live |
| Subscriptions | `newHeads`, `logs`, `newAcceptedTransactions` | Served by default; pending transactions unverified |
| Transport | `WebSocketConnection` over `web_socket_channel` | Network-free tests, contained dependency |
| Streams | Lazy; `cancel()` does not wait | A silent node cannot hang a cancel |
| Responses | Matched by id | Seen out of order in a live run |
| Models | Stable subset plus `raw` | Avalanche header fields change between upgrades |
| ERC-20 helper | Server-side filter, 3 topics + 32-byte data | Rejects ERC-721 transfers |
| Accepted transactions | Two methods, cost documented | Measured on Mainnet and Fuji |
| Heartbeat | 20 s ping, 15 s connect timeout | Verified with a 303 s idle connection |
| Reconnection | Opt-in policy, status events, no replay | Gap fill belongs to the application |
| Errors | One bad frame never ends a subscription | Healthy streams keep running |
