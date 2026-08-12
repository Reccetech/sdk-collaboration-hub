# AccountBalanceQuery Mirror Node Migration

## Summary

`AccountBalanceQuery` currently retrieves HBAR balances by issuing a free gRPC call to a consensus node (`CryptoService/cryptoGetBalance`). The Hedera network is deprecating this endpoint. This proposal introduces a replacement query, `MirrorNodeAccountBalanceQuery`, that fetches HBAR balance from the mirror node REST API (`GET /api/v1/balances?account.id={id}`), and simultaneously deprecates `AccountBalanceQuery`.

The new class follows the naming and structural convention already established in the SDKs for mirror node REST queries (`MirrorNodeContractCallQuery`, `MirrorNodeContractEstimateQuery`). The existing `AccountBalanceQuery` is marked deprecated but not removed, giving developers a migration window. A future `BlockNodeAccountBalanceQuery` is noted as a named placeholder for when block node infrastructure is available across the network.

**Date Submitted:** 2026-07-15

**Related references:**
- [Hedera blog: Migrating from AccountBalanceQuery](https://hedera.com/blog/migrating-from-accountbalancequery-what-you-need-to-know/)
- [Mirror node REST docs: GET /api/v1/balances](https://docs.hedera.com/hedera/sdks-and-apis/rest-api#balances)

---

## New APIs

### `MirrorNodeAccountBalance`

A new read-only data class returned by `MirrorNodeAccountBalanceQuery`. Returns HBAR balance only; token balances are not included (see Internal Changes).

```
@@finalType
MirrorNodeAccountBalance {
    @@immutable hbars: Hbar
}
```

### `MirrorNodeAccountBalanceQuery`

A new standalone query class that fetches HBAR balance from the mirror node REST API. Does not extend the base `Query` class (which carries consensus-node gRPC machinery). Follows the same pattern as `MirrorNodeContractCallQuery`.

`setAccountId` accepts `shard.realm.num`, EVM address (`0x...`), or public key alias — all resolved natively by the mirror node. Contract IDs are also accepted as the balances endpoint supports them; no separate `setContractId` method is needed.

```
MirrorNodeAccountBalanceQuery {
    @@nullable accountId: AccountId

    MirrorNodeAccountBalanceQuery setAccountId(accountId: AccountId)

    @@async
    MirrorNodeAccountBalance execute(client: Client)
}
```

### Future placeholder: `BlockNodeAccountBalanceQuery`

A `BlockNodeAccountBalanceQuery` class is reserved for a future proposal once block node infrastructure is broadly available. Its public API is expected to mirror `MirrorNodeAccountBalanceQuery`. No implementation is defined here.

---

## Updated APIs

### `AccountBalanceQuery` — deprecated

`AccountBalanceQuery` is marked deprecated across all SDKs. Its implementation and behavior are unchanged — it continues to issue gRPC calls to consensus nodes for the duration of the deprecation window. No fields or methods are removed. SDKs should emit a deprecation warning on construction pointing developers to `MirrorNodeAccountBalanceQuery`.

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

### `MirrorNodeAccountBalanceQuery` implementation

The class does not extend `Query`. It uses `fetch` (or the SDK's equivalent HTTP abstraction) and `client.mirrorRestApiBaseUrl`, following the same structure as `MirrorNodeContractCallQuery` and `FeeEstimateQuery`.

**Endpoint:** `GET /api/v1/balances?account.id={accountId}`

The `account.id` parameter accepts `shard.realm.num`, EVM address, public key alias, and contract ID — the mirror node resolves all forms. Parse `balances[0].balance` (tinybars) → `MirrorNodeAccountBalance.hbars`.

**Mirror response shape:**

```json
{
  "timestamp": "1234567890.000000000",
  "balances": [
    {
      "account": "0.0.12345",
      "balance": 123456789
    }
  ],
  "links": { "next": null }
}
```

### Why token balances are not returned

The `/api/v1/balances` endpoint does not paginate token balances. Fetching all token balances would require following unbounded pagination against `/api/v1/accounts/{id}/tokens` — a DDoS risk for accounts with large token portfolios and an unnecessary cost for callers who only need HBAR. Token balances are therefore out of scope for this class.

### Non-existent account

The balances endpoint returns an empty `balances` array for an account that does not exist (no 404). `MirrorNodeAccountBalanceQuery` returns a `MirrorNodeAccountBalance` with `hbars = 0` in this case.

### Eventual consistency

The mirror node reflects network state with a small lag (typically seconds). Applications that require immediate post-transaction balance confirmation should allow for this lag. This should be documented in SDK release notes.

### Free query — no change

Both `AccountBalanceQuery` and `MirrorNodeAccountBalanceQuery` are free. No payment logic is involved in either path.

### Response Codes

No consensus node response codes apply to `MirrorNodeAccountBalanceQuery`. Mirror node HTTP errors:

- `400 Bad Request` — invalid ID format. Surface as an SDK-appropriate error. Do not retry.
- `500 / 503 / 504` — transient mirror node error. Retry with the same backoff policy used by `FeeEstimateQuery`.

#### Transaction Retry

Mirror node retries follow existing mirror REST retry policy (`isRetryableNetworkError`): retry on 500/503/504 and network-level failures; do not retry on 400.

---

## Test Plan

Tests apply to `MirrorNodeAccountBalanceQuery` unless otherwise noted.

1. Given a valid account ID with a non-zero HBAR balance, when `MirrorNodeAccountBalanceQuery` is executed, then `MirrorNodeAccountBalance.hbars` matches the account's current HBAR balance as returned by the mirror node.
2. Given a valid account ID expressed as an EVM address, when `MirrorNodeAccountBalanceQuery` is executed, then the query resolves correctly and returns the HBAR balance.
3. Given a valid account ID expressed as a public key alias, when `MirrorNodeAccountBalanceQuery` is executed, then the query resolves correctly and returns the HBAR balance.
4. Given a valid contract ID passed as `accountId`, when `MirrorNodeAccountBalanceQuery` is executed, then `MirrorNodeAccountBalance.hbars` reflects the contract's current HBAR balance.
5. Given a non-existent account ID, when `MirrorNodeAccountBalanceQuery` is executed, then `MirrorNodeAccountBalance.hbars` is zero (empty balances array from mirror node).
6. Given a malformed account ID string, when `MirrorNodeAccountBalanceQuery` is executed, then the SDK throws an error before making a network call.
7. Given a mirror node that returns a transient 503 error on the first attempt, when `MirrorNodeAccountBalanceQuery` is executed, then the SDK retries and returns the correct result on a subsequent attempt.
8. Given a call to the deprecated `AccountBalanceQuery`, when it is constructed, then the SDK emits a deprecation warning directing the developer to `MirrorNodeAccountBalanceQuery`.
9. Given a call to the deprecated `AccountBalanceQuery`, when it is executed, then it returns a correct result via the consensus node gRPC path (no behavioral regression during the deprecation window).

### TCK

Tests 1–7 should each have a corresponding issue in `hiero-ledger/hiero-sdk-tck`. Tests 2 and 3 exercise identifier formats not covered by the legacy consensus-node path and should be prioritized.

---

## SDK Example

### HBAR balance (new API)

```javascript
import { MirrorNodeAccountBalanceQuery, Client } from "@hiero-ledger/sdk";

const client = Client.forTestnet();

const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0.0.12345")
    .execute(client);

console.log(`HBAR balance: ${balance.hbars.toString()}`);
```

### Lookup by EVM address or alias

```javascript
// EVM address
const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0x00000000000000000000000000000000000bc614e")
    .execute(client);

// Public key alias
const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0.0.302a300506032b6570032100e0c8ec2758a5879ffac226a13c0c516b799e72e35141a905d7822d6526b870d")
    .execute(client);
```

### Contract HBAR balance

```javascript
// Pass a contract ID via setAccountId — the balances endpoint supports contract IDs
const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0.0.98765")
    .execute(client);

console.log(`Contract HBAR balance: ${balance.hbars.toString()}`);
```

### Migration from `AccountBalanceQuery`

```javascript
// Before — deprecated
import { AccountBalanceQuery } from "@hiero-ledger/sdk";
const balance = await new AccountBalanceQuery()  // emits deprecation warning
    .setAccountId(accountId)
    .execute(client);

// After
import { MirrorNodeAccountBalanceQuery } from "@hiero-ledger/sdk";
const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId(accountId)
    .execute(client);

console.log(balance.hbars.toString());
```
