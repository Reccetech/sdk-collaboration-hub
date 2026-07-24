# Network Node Health Report

## Summary

The Hiero SDKs track per-node health state internally — backoff windows, readmit timestamps, bad gRPC status counts, and use counters — but expose none of it to developers. This proposal surfaces that state through a `NodeHealth` data class and `client.getNetworkHealth()`. It also standardises `ping()` and `pingAll()` across all SDKs; Go and Java already have these, but JS has neither.

### The external signing problem

Developers who sign transactions outside the SDK — using HSMs, offline signers, or custom signing services — must set `nodeAccountID` in `TransactionBody` before signing. Because the signature covers the full serialised `TransactionBody` bytes, the node choice is irrevocably locked in at signing time. If that node is unhealthy, the transaction fails on submission and cannot be re-routed; a new signing operation is required for each new `nodeAccountID`, which may mean a round-trip to an HSM or air-gapped environment.

The SDK's internal submission path sets `nodeAccountID` transparently at submission time, hiding this constraint. Developers building external signing pipelines have no SDK-supported way to check node health before committing a `nodeAccountID` to the transaction body. `getNetworkHealth()` closes this gap by exposing the same health state the SDK uses for its own node selection.

**Date Submitted:** 2026-07-22

**Related references:**
- Go precedent: `Client.Ping(AccountID)` / `Client.PingAll()` — `sdk/client.go:755,764`
- Java precedent: `Client.ping(AccountId)` / `Client.pingAll()` / async variants — `sdk/src/main/java/com/hedera/hashgraph/sdk/Client.java:654–851`
- Mirror node `/api/v1/network/nodes` — address book snapshot only; no live health signals

---

## New APIs

### `NodeHealth`

A read-only snapshot of one node's health state at the time `getNetworkHealth()` is called.

```
@@finalType
NodeHealth {
    @@immutable nodeId: int32
    @@immutable accountId: AccountId
    @@immutable @@nullable description: string
    @@immutable isHealthy: bool
    @@immutable currentBackoff: Duration
    @@immutable remainingBackoff: Duration
    @@immutable useCount: int64
    @@immutable badGrpcStatusCount: int64
}
```

- `nodeId` — the numeric node identifier (e.g. `0`, `1`, `2`); the primary key used by explorers such as HashScan and the mirror node REST API (`node_id` in `GET /api/v1/network/nodes`)
- `accountId` — the consensus node's account ID in `shard.realm.num` form (e.g. `0.0.3`); used for gRPC routing and matching entries in `GET /api/v1/network/nodes` (`node_account_id`)
- `description` — the human-readable node label from the address book (e.g. `"Hedera | 0.0.3 | East Coast, USA"`); `null` if the address book entry did not include a description
- `isHealthy` — `true` if the node is in the healthy pool (i.e. `readmitTime` has passed)
- `currentBackoff` — current exponential backoff step (8s minimum, 1 hour maximum)
- `remainingBackoff` — time until the node is re-admitted; `0` if healthy
- `useCount` — total times this node was selected for a request in this client's lifetime
- `badGrpcStatusCount` — cumulative gRPC errors received from this node

### `Client`

```
Client {
    @@async
    list<NodeHealth> getNetworkHealth()

    @@async
    @@throws(ping-timeout-error, not-found-error)
    void ping(nodeId: int32)

    @@async
    @@throws(ping-timeout-error, not-found-error)
    void ping(nodeId: int32, timeout: Duration)

    @@async
    @@throws(ping-timeout-error, not-found-error)
    void ping(accountId: AccountId)

    @@async
    @@throws(ping-timeout-error, not-found-error)
    void ping(accountId: AccountId, timeout: Duration)

    @@async
    void pingAll()

    @@async
    void pingAll(timeout: Duration)
}
```

- `getNetworkHealth()` — returns one `NodeHealth` per known node (healthy and unhealthy). No network calls; reads internal state only. Result is ordered by `nodeId` ascending.
- `ping(nodeId)` / `ping(accountId)` — sends a lightweight request to the specified node (identified by either its numeric `nodeId` or its `accountId`). Throws `ping-timeout-error` on failure or timeout; throws `not-found-error` if no node matches the given identifier. Updates the node's backoff state. Proceeds even if the node is currently in backoff.
- `pingAll()` — pings all known nodes in parallel. Does not throw on individual failures; each result updates that node's backoff state.

---

## Updated APIs

None.

---

## Internal Changes

### `getNetworkHealth()` implementation

Iterates the client's internal node list and constructs one `NodeHealth` snapshot per node. Result is ordered by `nodeId` ascending for deterministic output.

| Internal field | `NodeHealth` field | Source |
|---|---|---|
| `node.nodeId` | `nodeId` | Address book |
| `node.accountId` | `accountId` | Address book |
| `node.description` | `description` | Address book (`null` if absent) |
| `node.isHealthy()` | `isHealthy` | SDK health state |
| `node.currentBackoff` | `currentBackoff` | SDK health state |
| `max(0, node.readmitTime - now)` | `remainingBackoff` | SDK health state |
| `node.useCount` | `useCount` | SDK health state |
| `node.badGrpcStatusCount` | `badGrpcStatusCount` | SDK health state |

`nodeId`, `accountId`, and `description` are populated when the SDK initialises its node list from the address book. No additional network call is made by `getNetworkHealth()` itself.

**Implementation note:** Current SDK node runtime objects (`ManagedNode`/`Node`) track `accountId` for gRPC routing but may not retain `nodeId` or `description` after address book parsing. Each SDK must verify whether these fields are already stored in its node representation; if not, they must be added when the address book is loaded.

### `ping()` / `pingAll()` implementation

`ping()` sends a lightweight gRPC request to the target node and updates its backoff on success or failure. `pingAll()` calls `ping()` for each node in parallel and swallows individual failures.

The existing Go and Java implementations use `AccountBalanceQuery` as the probe request (precedent: `sdk/client.go` and `Client.java`). As part of the companion [AccountBalanceQuery Deprecation](./account-balance-query-deprecation.md) proposal, all SDKs must replace this probe with `NetworkService/getVersionInfo` before the September 2025 deprecation date. This RPC is free, requires no entity ID, and is available on every consensus node. Go and Java SDKs already have `NetworkVersionQuery` wrapping this RPC; JS will need to call it directly or through an equivalent wrapper.

### JS SDK — prerequisite bug fix

One pre-existing bug in `ManagedNode.js` affects `NodeHealth` field accuracy and must be fixed as part of this work:

- **`_badGrpcStatusCount` never incremented**: the field is declared and read but never written after construction, so `badGrpcStatusCount` is always 0. Fix: increment on each call to `increaseBackoff()`, matching Go and Java behaviour.

### Response Codes

No new consensus node response codes are introduced. `ping()` may surface any gRPC status that `NetworkService/getVersionInfo` returns. `getNetworkHealth()` makes no network calls and has no error codes.

#### Transaction Retry

`ping()` does not retry on failure — a single attempt is made, and the result updates the node's backoff state.

---

## Test Plan

1. Given a client connected to a multi-node network, when `getNetworkHealth()` is called, then a `NodeHealth` entry is returned for every known node, including nodes currently in backoff.
2. Given all nodes are healthy, when `getNetworkHealth()` is called, then every entry has `isHealthy: true`, `remainingBackoff: 0`, and `badGrpcStatusCount: 0`.
3. Given a node has received 3 consecutive gRPC errors, when `getNetworkHealth()` is called, then that node's entry has `isHealthy: false`, `badGrpcStatusCount: 3`, and `currentBackoff` reflecting 3 doublings from the minimum backoff value.
4. Given an unhealthy node whose backoff window has elapsed, when `getNetworkHealth()` is called, then that node's entry has `isHealthy: true` and `remainingBackoff: 0`.
5. Given a reachable node, when `ping(accountId)` is called, then the call completes without error and the node's `currentBackoff` is decreased.
6. Given an unreachable node, when `ping(accountId)` is called, then a `ping-timeout-error` is thrown and the node's `currentBackoff` is increased.
7. Given a node currently in backoff, when `ping(accountId)` is called, then the ping proceeds regardless of backoff state.
8. Given a multi-node network with some unreachable nodes, when `pingAll()` is called, then all nodes are contacted in parallel and no exception is thrown.
9. Given a multi-node network where some nodes fail during `pingAll()`, when `getNetworkHealth()` is called immediately after, then the failing nodes show `isHealthy: false` and incremented `badGrpcStatusCount`.
10. Given the JS SDK, when a node has received gRPC errors, then `badGrpcStatusCount` reflects the accurate count (verifying the `_badGrpcStatusCount` increment fix).

### TCK

Tests 1–9 should have corresponding issues in `hiero-ledger/hiero-sdk-tck`. Test 10 is a JS-specific unit test.

---

## SDK Example

### External signing — select a healthy node before signing

When signing outside the SDK, `nodeAccountID` must be embedded in the transaction body before the signature is produced. It cannot be changed after signing without re-signing.

```javascript
import { Client } from "@hiero-ledger/sdk";

const client = Client.forTestnet();

// Refresh health state with a live ping before selecting
await client.pingAll();

const health = await client.getNetworkHealth();
const healthyNodes = health.filter(n => n.isHealthy);

if (healthyNodes.length === 0) {
    throw new Error("No healthy nodes available");
}

// Pick the node with the fewest accumulated errors
healthyNodes.sort((a, b) => a.badGrpcStatusCount - b.badGrpcStatusCount);
const selectedNode = healthyNodes[0];

console.log(
    `Selected node ${selectedNode.nodeId} ` +        // e.g. 3
    `(${selectedNode.accountId}) ` +                  // e.g. 0.0.3
    `"${selectedNode.description ?? "unknown"}"`      // e.g. "Hedera | 0.0.3 | East Coast, USA"
);

// nodeAccountID is now fixed in the transaction body — the signature covers
// these bytes and the node cannot be changed without re-signing
const txBytes = buildTransactionBody({
    nodeAccountID: selectedNode.accountId.toString(),
    // ... other fields
});

const signature = await hsm.sign(txBytes);
// submit txBytes + signature to the network
```

### Ping a specific node to test recovery

Nodes can be targeted by numeric `nodeId` (matching the ID shown in HashScan and `GET /api/v1/network/nodes`) or by `accountId`.

```javascript
// By numeric node ID (matches HashScan / mirror node explorer)
try {
    await client.ping(3);
    console.log("Node 3 is reachable");
} catch (e) {
    console.error("Node 3 is unreachable:", e.message);
}

// By account ID
try {
    await client.ping(AccountId.fromString("0.0.3"));
    console.log("Node 0.0.3 is reachable");
} catch (e) {
    console.error("Node 0.0.3 is unreachable:", e.message);
}
```
