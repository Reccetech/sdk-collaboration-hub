# AccountBalanceQuery Mirror Node Migration

## Summary

`AccountBalanceQuery` currently retrieves HBAR and token balances by issuing a free gRPC call to a consensus node (`CryptoService/cryptoGetBalance`). The Hedera network is deprecating this endpoint. This proposal introduces a replacement query, `MirrorNodeAccountBalanceQuery`, that fetches the same data from the mirror node REST API (`GET /api/v1/accounts/{idOrAliasOrEvmAddress}`), and simultaneously deprecates `AccountBalanceQuery`.

The new class follows the naming and structural convention already established in the SDKs for mirror node REST queries (`MirrorNodeContractCallQuery`, `MirrorNodeContractEstimateQuery`). The existing `AccountBalanceQuery` is marked deprecated but not removed, giving developers a migration window. A future `BlockNodeAccountBalanceQuery` is noted as a named placeholder for when block node infrastructure is available across the network.

**Relationship to the companion proposal:** The companion [AccountBalanceQuery Deprecation](./account-balance-query-deprecation.md) proposal makes `execute()` fail immediately with a hard error, while this proposal keeps `AccountBalanceQuery` fully functional during a migration window. The two are mutually exclusive strategies for retiring the consensus-node balance path; only one of them can be adopted.

**Date Submitted:** 2026-07-15

**Related references:**
- [Hedera blog: Migrating from AccountBalanceQuery](https://hedera.com/blog/migrating-from-accountbalancequery-what-you-need-to-know/)
- [Mirror node REST docs: GET /api/v1/accounts/{id}](https://docs.hedera.com/hedera/sdks-and-apis/rest-api#accounts)
- [Mirror node REST docs: GET /api/v1/contracts/{id}](https://docs.hedera.com/hedera/sdks-and-apis/rest-api#contracts)
- [Docs: Get Account Token Balance](https://docs.hedera.com/native/tokens/get-balance#get-account-token-balance)

---

## New APIs

### `MirrorNodeAccountBalanceQuery`

A new standalone query class that fetches account or contract balance from the mirror node REST API. Does not extend the base `Query` class (which carries consensus-node gRPC machinery). Follows the same pattern as `MirrorNodeContractCallQuery`.

```
MirrorNodeAccountBalanceQuery {
    AccountId | null    accountId
    ContractId | null   contractId

    MirrorNodeAccountBalanceQuery setAccountId(accountId: AccountId | string)
    MirrorNodeAccountBalanceQuery setContractId(contractId: ContractId | string)

    Promise<AccountBalance> execute(client: Client)
}
```

`setAccountId` accepts any form `AccountId.fromString(...)` accepts: `shard.realm.num`, EVM address (`0x...`), or public key alias. The mirror node natively resolves all three, which is an improvement over the consensus node path.

`setContractId` routes to `GET /api/v1/contracts/{contractId}` instead; see Internal Changes.

### Future placeholder: `BlockNodeAccountBalanceQuery`

A `BlockNodeAccountBalanceQuery` class is reserved for a future proposal once block node infrastructure is broadly available. Its public API is expected to mirror `MirrorNodeAccountBalanceQuery`. No implementation is defined here.

---

## Updated APIs

### `AccountBalanceQuery` — deprecated

`AccountBalanceQuery` is marked `@deprecated` across all SDKs. Its implementation and behavior are **unchanged** — it continues to issue gRPC calls to consensus nodes for the duration of the deprecation window. No fields or methods are removed. SDKs should emit a deprecation warning on construction or first `execute()` call pointing developers to `MirrorNodeAccountBalanceQuery`.

```
@deprecated Use MirrorNodeAccountBalanceQuery instead.
AccountBalanceQuery {
    AccountId | null    accountId     // unchanged
    ContractId | null   contractId    // unchanged

    AccountBalanceQuery setAccountId(accountId: AccountId | string)
    AccountBalanceQuery setContractId(contractId: ContractId | string)

    Promise<AccountBalance> execute(client: Client)
}
```

### `AccountBalance` — `tokenDecimals` deprecated

`tokenDecimals` is retained on `AccountBalance` so that existing code using `AccountBalanceQuery` continues to compile and run unchanged during the deprecation window. When using the new `MirrorNodeAccountBalanceQuery`, `tokenDecimals` will always be empty (the mirror node account balance endpoint does not return decimals). SDKs should mark it `@deprecated`.

```
AccountBalance {
    Hbar            hbars           // unchanged
    TokenBalanceMap tokens          // unchanged
    @deprecated
    TokenDecimalMap tokenDecimals   // empty when returned by MirrorNodeAccountBalanceQuery
}
```

---

## Internal Changes

### `MirrorNodeAccountBalanceQuery` implementation

The class does not extend `Query`. It directly uses `fetch` (or the SDK's equivalent HTTP abstraction) and `client.mirrorRestApiBaseUrl` from the client's mirror network, following the same structure as `MirrorNodeContractCallQuery` and `FeeEstimateQuery`.

**Request routing:**

1. If `accountId` is set:
   - Issue `GET /api/v1/accounts/{accountId}` where `accountId` is the string form of the `AccountId` (shard.realm.num, EVM address, or alias — all accepted by the mirror node).
   - Parse `balance.balance` (tinybars) → `AccountBalance.hbars`.
   - Parse `balance.tokens[]` → `AccountBalance.tokens` (TokenBalanceMap), following `next` pagination links until exhausted.
   - Leave `AccountBalance.tokenDecimals` empty.

2. If `contractId` is set:
   - Issue `GET /api/v1/contracts/{contractId}`.
   - Parse `balance` (tinybars) → `AccountBalance.hbars`.
   - Return an empty `TokenBalanceMap` (the mirror node contracts endpoint does not expose token balances).

**Mirror response shape (account path):**

```json
{
  "balance": {
    "balance": 123456789,
    "tokens": [
      { "token_id": "0.0.12345", "balance": 500 },
      { "token_id": "0.0.67890", "balance": 100 }
    ]
  },
  "links": { "next": "/api/v1/accounts/0.0.12345/tokens?limit=25&..." }
}
```

### Why `tokenDecimals` cannot be populated

The mirror node `GET /api/v1/accounts/{id}` response does not include token decimals. Retrieving them would require one `GET /api/v1/tokens/{tokenId}` call per token held — an unbounded number of extra calls for accounts with large token portfolios. `tokenDecimals` is therefore empty when using `MirrorNodeAccountBalanceQuery`. Developers who need decimals should use a `TokenInfoQuery` (or equivalent) per token.

### Pagination

The `balance.tokens` array in the mirror node account response is paginated. `MirrorNodeAccountBalanceQuery` must follow `links.next` until it is absent, accumulating all token entries before constructing the `TokenBalanceMap`. This matches the pagination behavior of `AddressBookQueryWeb`.

### Eventual consistency

The mirror node reflects network state with a small lag (typically seconds). `AccountBalanceQuery` (consensus node) returns data current as of the latest handled block. Applications that require immediate post-transaction balance confirmation should allow for this lag when using `MirrorNodeAccountBalanceQuery`. This should be documented in SDK release notes.

### Free query — no change

Both `AccountBalanceQuery` and `MirrorNodeAccountBalanceQuery` are free. No payment logic is involved in either path.

### Response Codes

No consensus node response codes apply to `MirrorNodeAccountBalanceQuery`. Mirror node HTTP errors:

- `400 Bad Request` — invalid ID format. Surface as an SDK-appropriate error with a clear message. Do not retry.
- `404 Not Found` — account or contract does not exist. Surface as an SDK-appropriate error. Do not retry.
- `500 / 503 / 504` — transient mirror node error. Retry with the same backoff policy used by `FeeEstimateQuery`.

#### Transaction Retry

Mirror node retries follow existing mirror REST retry policy (`isRetryableNetworkError`): retry on 500/503/504 and network-level failures; do not retry on 400/404.

---

## Test Plan

Tests apply to `MirrorNodeAccountBalanceQuery` unless otherwise noted.

1. Given a valid account ID with a non-zero HBAR balance, when `MirrorNodeAccountBalanceQuery` is executed, then `AccountBalance.hbars` matches the account's current HBAR balance as returned by the mirror node.

2. Given a valid account ID that holds fungible tokens, when `MirrorNodeAccountBalanceQuery` is executed, then `AccountBalance.tokens` contains each token ID with the correct raw balance amount.

3. Given a valid account ID expressed as an EVM address string, when `MirrorNodeAccountBalanceQuery` is executed with `setAccountId(evmAddress)`, then the query resolves correctly and returns the HBAR and token balances for that account.

4. Given a valid account ID expressed as a public key alias string, when `MirrorNodeAccountBalanceQuery` is executed with `setAccountId(alias)`, then the query resolves correctly and returns the HBAR balance.

5. Given a valid contract ID, when `MirrorNodeAccountBalanceQuery` is executed with `setContractId(contractId)`, then `AccountBalance.hbars` reflects the contract's current HBAR balance and `AccountBalance.tokens` is empty.

6. Given any account ID, when `MirrorNodeAccountBalanceQuery` is executed, then `AccountBalance.tokenDecimals` is always empty, confirming decimals are not populated.

7. Given a non-existent account ID, when `MirrorNodeAccountBalanceQuery` is executed, then the SDK throws a clear error indicating the account was not found (mirror node 404).

8. Given a malformed account ID string, when `MirrorNodeAccountBalanceQuery` is executed, then the SDK throws an error indicating invalid input before making a network call.

9. Given a mirror node that returns a transient 503 error on the first attempt, when `MirrorNodeAccountBalanceQuery` is executed, then the SDK retries and returns the correct result on a subsequent attempt.

10. Given an account that holds more than the mirror node's per-page token limit, when `MirrorNodeAccountBalanceQuery` is executed, then all token balances across all pages are returned in `AccountBalance.tokens`.

11. Given a call to the deprecated `AccountBalanceQuery`, when it is constructed or executed, then the SDK emits a deprecation warning directing the developer to `MirrorNodeAccountBalanceQuery`.

12. Given a call to the deprecated `AccountBalanceQuery`, when it is executed, then it still returns a correct result via the consensus node gRPC path (no behavioral regression during the deprecation window).

### TCK

The above tests should each have a corresponding issue created in `hiero-ledger/hiero-sdk-tck` and linked back to this design doc. Tests 3, 4, and 10 exercise capabilities not covered by the legacy consensus-node path and should be prioritized.

---

## SDK Example

### HBAR and token balances (new API)

```javascript
import { MirrorNodeAccountBalanceQuery, Client } from "@hiero-ledger/sdk";

const client = Client.forTestnet();

const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0.0.12345")
    .execute(client);

console.log(`HBAR balance: ${balance.hbars.toString()}`);

for (const [tokenId, amount] of balance.tokens) {
    console.log(`Token ${tokenId.toString()}: ${amount.toString()}`);
}
```

### Lookup by EVM address (new capability)

```javascript
const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId("0x00000000000000000000000000000000000bc614e")
    .execute(client);

console.log(`HBAR balance: ${balance.hbars.toString()}`);
```

### Contract balance

```javascript
import { MirrorNodeAccountBalanceQuery, ContractId, Client } from "@hiero-ledger/sdk";

const balance = await new MirrorNodeAccountBalanceQuery()
    .setContractId("0.0.98765")
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

// After — use MirrorNodeAccountBalanceQuery
import { MirrorNodeAccountBalanceQuery } from "@hiero-ledger/sdk";

const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId(accountId)
    .execute(client);
```

### Fetching token decimals separately (if needed)

`MirrorNodeAccountBalanceQuery` does not populate `tokenDecimals`. Use `TokenInfoQuery` per token if decimals are required.

```javascript
import { MirrorNodeAccountBalanceQuery, TokenInfoQuery } from "@hiero-ledger/sdk";

const balance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId(accountId)
    .execute(client);

// balance.tokenDecimals is empty — fetch decimals separately
const tokenInfo = await new TokenInfoQuery()
    .setTokenId(myTokenId)
    .execute(client);

console.log(`Decimals: ${tokenInfo.decimals}`);
```
