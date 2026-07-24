# AccountBalanceQuery Deprecation

## Summary

The Hedera network is retiring the consensus node `CryptoService/cryptoGetBalance` endpoint used by `AccountBalanceQuery`. This proposal deprecates `AccountBalanceQuery` across all Hiero SDKs: the class emits a runtime warning on construction and throws a hard error on execution. The error message will direct developers to the mirror node REST API as the replacement path; the specific replacement SDK class is still under discussion and may or may not be adopted (see companion proposal).

Removal of the class is out of scope and will be addressed in a future proposal. Keeping the class present but non-functional is a deliberate risk mitigation: if deprecation causes significant user churn, the consensus node path can be restored by reverting a single internal override with no public API changes.

**Date Submitted:** 2026-07-15

**Target deprecation date:** September 2025

**Related references:**
- [Hedera blog: Migrating from AccountBalanceQuery](https://hedera.com/blog/migrating-from-accountbalancequery-what-you-need-to-know/)
- [Companion proposal: Mirror node account balance query](./account-balance-query-mirror-node-migration.md)
- [Network node health report proposal](./network-node-health-report.md) — introduces the replacement `ping()` / `pingAll()` liveness mechanism
- [hiero-ledger/hiero-consensus-node #26457](https://github.com/hiero-ledger/hiero-consensus-node/issues/26457) — CN-side removal of `GetAccountBalance` from throttle configuration
- Precedent in JS SDK: `src/account/AccountAllowanceAdjustTransaction.js`

---

## New APIs

None.

---

## Updated APIs

### `AccountBalanceQuery`

No fields or methods added or removed. The class is marked deprecated in each SDK's idiomatic way (the
meta-language defines no deprecation annotation), and `execute()` now throws.

```
@@oneOrNoneOf(accountId, contractId)
AccountBalanceQuery {
    @@nullable accountId: AccountId
    @@nullable contractId: ContractId

    AccountBalanceQuery setAccountId(accountId: AccountId)
    AccountBalanceQuery setContractId(contractId: ContractId)

    @@async
    @@throws(deprecated-query-error)
    AccountBalance execute(client: Client)
}
```

---

## Internal Changes

### Constructor — deprecation warning

Each SDK emits its idiomatic deprecation warning when the query is constructed (precedent in the JS SDK:
`console.warn`, as used by `Executable.setMaxRetries` and `ManagedNetwork.setNetworkName`):

```
Deprecated: AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances.
```

### Execute — hard error

The internal execution path is overridden to fail immediately without making any network call (precedent in the
JS SDK: `AccountAllowanceAdjustTransaction` overriding `_execute()`):

```
Error: AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances.
```

Each SDK marks the class deprecated in its idiomatic way (e.g., JSDoc `@deprecated`, Java `@Deprecated`,
Rust `#[deprecated]`) so IDEs and linters surface a diagnostic at every call site — in the JS SDK,
`eslint-plugin-deprecation` is already configured for this.

### `Client.ping()` / `Client.pingAll()` — replacement required

`Client.ping()` and `Client.pingAll()` currently use `AccountBalanceQuery` internally as a liveness probe across all SDK implementations. Once `AccountBalanceQuery` is deprecated and the network throttle is removed, these methods will fail. This must be addressed in all SDK languages as part of this work — not deferred.

Each SDK must replace the `AccountBalanceQuery` probe with `NetworkService/getVersionInfo` before the September deprecation date. This RPC is free, requires no entity ID, is available on every consensus node, and tests the full gRPC path end-to-end — a faithful substitute for what the ping probe needs to do. Go and Java SDKs already have `NetworkVersionQuery` wrapping this RPC. The [network node health report proposal](./network-node-health-report.md) standardises the public `ping()` / `pingAll()` API surface as part of this replacement.

An XTS dry run (July 2025) confirmed that failing to replace this probe breaks Solo's consensus-node readiness checks and connectivity tests. SDK teams should audit all internal uses of `AccountBalanceQuery` — not only the public `ping()` path — before the deprecation date.

### Response Codes / Transaction Retry

Not applicable — no network request is made by `execute()`.

---

## Test Plan

1. Given an `AccountBalanceQuery` is constructed, then `console.warn` is emitted containing `"AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances."`.
2. Given `execute()` is called on an `AccountBalanceQuery`, then the promise rejects with an `Error` containing `"AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances."`.
3. Given `execute()` is called on an `AccountBalanceQuery`, then no network call to any consensus node is made.
4. Given code migrated to the mirror node REST API for account balance retrieval, then no deprecation warning or error is emitted.
5. Given `AccountBalanceQuery` is deprecated, when `client.ping(nodeId)` or `client.pingAll()` is called, then the call succeeds using the replacement liveness probe and no `deprecated-query-error` is thrown.

### TCK

Tests 1–3 should have corresponding issues in `hiero-ledger/hiero-sdk-tck`. Test 3 is the critical integration check confirming the gRPC path is fully bypassed. Test 5 should be validated against a live network to confirm the replacement probe is accepted.

---

## SDK Example

```javascript
// Before — emits warning on construction, throws on execute
import { AccountBalanceQuery } from "@hiero-ledger/sdk";
const query = new AccountBalanceQuery().setAccountId("0.0.12345"); // warns
await query.execute(client); // throws: "AccountBalanceQuery is no longer supported..."

// After — query the mirror node REST API directly or via a replacement SDK class
// GET /api/v1/accounts/0.0.12345
```
