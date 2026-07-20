# AccountBalanceQuery Deprecation

## Summary

The Hedera network is retiring the consensus node `CryptoService/cryptoGetBalance` endpoint used by `AccountBalanceQuery`. This proposal deprecates `AccountBalanceQuery` across all Hiero SDKs: the class emits a runtime warning on construction and throws a hard error on execution. The error message will direct developers to the mirror node REST API as the replacement path; the specific replacement SDK class is still under discussion and may or may not be adopted (see companion proposal).

Removal of the class is out of scope and will be addressed in a future proposal. Keeping the class present but non-functional is a deliberate risk mitigation: if deprecation causes significant user churn, the consensus node path can be restored by reverting a single internal override with no public API changes.

**Date Submitted:** 2026-07-15

**Related references:**
- [Hedera blog: Migrating from AccountBalanceQuery](https://hedera.com/blog/migrating-from-accountbalancequery-what-you-need-to-know/)
- [Companion proposal: Mirror node account balance query](./account-balance-query-mirror-node-migration.md)
- Precedent in JS SDK: `src/account/AccountAllowanceAdjustTransaction.js`

---

## New APIs

None.

---

## Updated APIs

### `AccountBalanceQuery`

No fields or methods added or removed. `@deprecated` added to the class-level JSDoc. `execute()` now throws.

```
@deprecated AccountBalanceQuery is no longer supported. Use the mirror node REST API
            to retrieve account balances. This class will be removed in a future release.
AccountBalanceQuery {
    AccountId | null    accountId
    ContractId | null   contractId

    AccountBalanceQuery setAccountId(accountId: AccountId | string)
    AccountBalanceQuery setContractId(contractId: ContractId | string)

    Promise<AccountBalance> execute(client: Client)  // now throws — see Internal Changes
}
```

---

## Internal Changes

### Constructor — deprecation warning

A `console.warn()` is added to the constructor, consistent with the existing SDK pattern (e.g., `Executable.setMaxRetries`, `ManagedNetwork.setNetworkName`):

```
Deprecated: AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances.
```

### Execute — hard error

`_execute()` (and equivalent internal dispatch methods) are overridden to reject/throw immediately without making any network call, following the `AccountAllowanceAdjustTransaction` precedent:

```
Error: AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances.
```

The `@deprecated` tag on the class causes IDEs and linters (`eslint-plugin-deprecation`, already configured in the JS SDK) to surface a diagnostic at every call site.

### Response Codes / Transaction Retry

Not applicable — no network request is made.

---

## Test Plan

1. Given an `AccountBalanceQuery` is constructed, then `console.warn` is emitted containing `"AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances."`.
2. Given `execute()` is called on an `AccountBalanceQuery`, then the promise rejects with an `Error` containing `"AccountBalanceQuery is no longer supported. Use the mirror node REST API to retrieve account balances."`.
3. Given `execute()` is called on an `AccountBalanceQuery`, then no network call to any consensus node is made.
4. Given code migrated to the mirror node REST API for account balance retrieval, then no deprecation warning or error is emitted.

### TCK

Tests 1–3 should have corresponding issues in `hiero-ledger/hiero-sdk-tck`. Test 3 is the critical integration check confirming the gRPC path is fully bypassed.

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
