# MirrorNodeTokenBalanceQuery

## Summary

`AccountBalanceQuery` originally returned HBAR balance and token balances together. Token balance support was deprecated in Services release 0.50 (HIP-367), and `AccountBalanceQuery` itself is deprecated and will be removed with consensus node release 77 (estimated mainnet September 9, 2026). `MirrorNodeAccountBalanceQuery` replaces the HBAR path but intentionally omits token balances — the mirror node `/api/v1/accounts/{id}/tokens` endpoint paginates, and fetching a full token portfolio implicitly inside a single `execute()` call would make the cost invisible to the caller.

This proposal introduces `MirrorNodeTokenBalanceQuery`: an SDK query for fetching token balances from the mirror node REST API. The query always requires an `accountId`, scoping it to a single account's finite set of token associations. An optional `tokenId` filter narrows the result to a single token and eliminates pagination entirely. The SDK handles all pagination internally; the caller always receives a complete list.

This is a follow-on to `MirrorNodeAccountBalanceQuery` and should be implemented after that class ships in all SDKs.

**Date Submitted:** 2026-08-14

**Related references:**
- [Companion proposal: MirrorNodeAccountBalanceQuery](./account-balance-query-mirror-node-migration.md)
- [Companion proposal: AccountBalanceQuery Deprecation](./account-balance-query-deprecation.md)
- [Mirror node REST docs: GET /api/v1/accounts/{id}/tokens](https://docs.hedera.com/hedera/sdks-and-apis/rest-api#tokens-2)

---

## New APIs

### `MirrorNodeTokenBalance`

A read-only data class representing a single token balance for an account.

```
@@finalType
MirrorNodeTokenBalance {
    @@immutable tokenId: TokenId
    @@immutable balance: Long    // raw balance in the token's smallest denomination
    @@immutable decimals: int    // number of decimal places for the token
}
```

`balance` is the raw on-chain value in the token's smallest unit. To get a human-readable amount, divide by `10^decimals`. This matches the behaviour of the existing `AccountBalance.tokens` map, which also returned raw balances.

**Design note — consolidation of balance and decimals:**
The existing `AccountBalance` type split token data across two separate maps: `tokens: TokenBalanceMap` (balance keyed by `TokenId`) and `tokenDecimals: TokenDecimalMap` (decimals keyed by `TokenId`). `MirrorNodeTokenBalance` consolidates both into a single object per token — balance and decimals are always available together. This is a better design and eliminates the two-map lookup; SDK teams implementing this type do not need to replicate the split-map pattern.

### `MirrorNodeTokenBalanceQuery`

A new standalone query class that fetches token balances from the mirror node REST API. Does not extend the base `Query` class. Follows the same pattern as `MirrorNodeAccountBalanceQuery`.

`setAccountId` accepts `shard.realm.num`, EVM address (`0x...`), or public key alias — all resolved natively by the mirror node.

```
MirrorNodeTokenBalanceQuery {
    @@nullable accountId: AccountId
    @@nullable tokenId: TokenId

    // Required — throws before any network call if not set.
    MirrorNodeTokenBalanceQuery setAccountId(accountId: AccountId)

    // Optional. If set, fetches only this token (single request, no pagination).
    // If not set, fetches all token balances for the account (SDK paginates internally).
    MirrorNodeTokenBalanceQuery setTokenId(tokenId: TokenId)

    @@async
    List<MirrorNodeTokenBalance> execute(client: Client)
}
```

**`execute` return value:**
- When `tokenId` is set: one request, no pagination. Returns a list of 0 or 1 items. SDK does not follow `links.next`.
- When `tokenId` is not set: SDK follows `links.next` cursor (`token.id=gt:{last_id}`) until null. Returns the complete list of token balances for the account.

**Design note — return type (List vs Map):**
`execute` returns `List<MirrorNodeTokenBalance>` rather than a map keyed by `TokenId`. A list preserves ordering from the mirror node response, is natural for iteration (wallet portfolio display), and avoids the ergonomic split of the existing `TokenBalanceMap` / `TokenDecimalMap` pair. For single-token lookups, `setTokenId` eliminates the need to search a list in the first place. SDK teams that have a strong convention around map-keyed balance types may choose to wrap the list in a `MirrorNodeTokenBalanceMap` type that exposes a `get(tokenId)` accessor — that is a valid implementation choice and does not change the wire behaviour described in this proposal.

---

## Internal Changes

### `MirrorNodeTokenBalanceQuery` implementation

The class does not extend `Query`. It uses `fetch` (or the SDK's equivalent HTTP abstraction) and `client.mirrorRestApiBaseUrl`, following the same structure as `MirrorNodeAccountBalanceQuery` and `FeeEstimateQuery`.

**Endpoint (single-token):**
```
GET /api/v1/accounts/{accountId}/tokens?token.id={tokenId}&limit=100
```
Use `limit=100` (not `limit=1`) so that `links.next` is null when the single result is returned. With `limit=1`, the mirror node echoes a non-null `links.next` even for a single-item result (see pagination note below). In either case, the SDK does not follow `links.next` when `tokenId` is set.

**Endpoint (all tokens, first page):**
```
GET /api/v1/accounts/{accountId}/tokens?limit=100&order=asc
```

**Pagination loop (no `tokenId` filter only):**

The SDK follows `links.next` until `links.next` is null. The SDK requests 100 items per page. The cursor is embedded in `links.next` by the mirror node as a `token.id=gt:{last_id}` cursor — the SDK uses the URL verbatim rather than constructing it.

```
GET /api/v1/accounts/{id}/tokens?limit=100&order=asc
→ { tokens: [...100 items...], links: { next: "/api/v1/accounts/{id}/tokens?limit=100&token.id=gt:0.0.500" } }
GET /api/v1/accounts/{id}/tokens?limit=100&token.id=gt:0.0.500&order=asc
→ { tokens: [...remaining...], links: { next: null } }
```

**Important — `links.next` behavior when `tokenId` filter is set:**

When `setTokenId` is used, the SDK must NOT follow `links.next`. The mirror node echoes back a non-null `links.next` with the same `token.id=` filter even when the result fits within the page limit (e.g., `limit=1` returning one result). That echoed link is a filter reference, not a pagination cursor — following it loops indefinitely returning the same token. When `setTokenId` is set, `execute` is always a single request with no pagination regardless of `links.next`.

**Mirror response shape per token:**
```json
{
  "token_id": "0.0.12345",
  "balance": 5000000,
  "decimals": 6,
  "automatic_association": true,
  "created_timestamp": "1234567890.000000000",
  "freeze_status": "UNFROZEN",
  "kyc_status": "GRANTED"
}
```

Parse `token_id` → `TokenId`, `balance` → `Long`, `decimals` → `int`. The remaining fields (`automatic_association`, `created_timestamp`, `freeze_status`, `kyc_status`) are out of scope for a balance query and are not exposed on `MirrorNodeTokenBalance`.

### Free query

`MirrorNodeTokenBalanceQuery` is free. No query payment or operator signing is required.

### Non-existent account or token

The mirror node validates account existence before executing the token query. A non-existent account returns HTTP 404, which the SDK surfaces as an error. A valid account that does not hold the specified `tokenId` returns an empty list (no error).

### Eventual consistency

The mirror node reflects consensus state with a small lag (typically seconds). Token balances read immediately after a transfer may still show pre-transfer values. This should be documented in SDK release notes.

### Response Codes

No consensus node response codes apply. Mirror node HTTP errors:

- `400 Bad Request` — invalid ID format. Surface as an SDK-appropriate error. Do not retry.
- `404 Not Found` — account does not exist. Surface as an SDK-appropriate error. Do not retry.
- `500 / 503` — transient mirror node error. Retry with the same backoff policy used by `MirrorNodeAccountBalanceQuery`.

Retries apply per-page — a transient failure mid-pagination retries the failed page, not the entire query from the start.

---

## Test Plan

1. Given a valid account ID with multiple token associations and no `tokenId` filter, when `execute()` is called, then all token balances for the account are returned.
2. Given a valid account ID and a `tokenId` the account holds, when `execute()` is called with `setTokenId`, then exactly one `MirrorNodeTokenBalance` is returned with the correct balance and decimals.
3. Given a valid account ID and a `tokenId` the account does not hold, when `execute()` is called with `setTokenId`, then an empty list is returned.
4. Given a non-existent account ID, when `execute()` is called, then an error is thrown (mirror node returns 404).
5. Given no `accountId` is set, when `execute()` is called, then an error is thrown before any network call.
6. Given a malformed account ID string, when `execute()` is called, then an SDK error is thrown before any network call.
7. Given a mirror node that returns a transient 503 on one page mid-pagination, when `execute()` is called, then the SDK retries that page and returns the correct complete result.
8. Given an account with token associations spanning multiple pages (more than 100 tokens), when `execute()` is called without `setTokenId`, then all tokens across all pages are returned.
9. Given `setTokenId` is set, when `execute()` is called, then the SDK makes exactly one HTTP request regardless of the value of `links.next` in the response (the mirror node echoes a non-null `links.next` for filtered queries at small page sizes — the SDK must not follow it).

### TCK

Tests 1–9 should have corresponding issues in `hiero-ledger/hiero-sdk-tck`. Test 7 (mid-pagination retry), test 8 (multi-page result), and test 9 (single-request enforcement when `tokenId` is set) are the critical integration checks for the pagination design.

---

## SDK Example

### All token balances — wallet portfolio

```javascript
import { MirrorNodeTokenBalanceQuery, Client } from "@hiero-ledger/sdk";

const client = Client.forMainnet();

const tokens = await new MirrorNodeTokenBalanceQuery()
    .setAccountId("0.0.12345")
    .execute(client);

for (const token of tokens) {
    const humanReadable = token.balance / Math.pow(10, token.decimals);
    console.log(`${token.tokenId}: ${humanReadable}`);
}
```

### Single-token lookup

```javascript
const result = await new MirrorNodeTokenBalanceQuery()
    .setAccountId("0.0.12345")
    .setTokenId("0.0.98765")
    .execute(client);

if (result.length > 0) {
    console.log(`Balance: ${result[0].balance} (${result[0].decimals} decimals)`);
} else {
    console.log("Account does not hold this token");
}
```

### Migration from `AccountBalanceQuery`

```javascript
// Before — deprecated
import { AccountBalanceQuery } from "@hiero-ledger/sdk";
const balance = await new AccountBalanceQuery()
    .setAccountId(accountId)
    .execute(client);
const hbars = balance.hbars;       // Hbar
const tokens = balance.tokens;     // Map<TokenId, Long> — deprecated since HIP-367

// After — HBAR balance
import { MirrorNodeAccountBalanceQuery } from "@hiero-ledger/sdk";
const hbarBalance = await new MirrorNodeAccountBalanceQuery()
    .setAccountId(accountId)
    .execute(client);

// After — token balances
import { MirrorNodeTokenBalanceQuery } from "@hiero-ledger/sdk";
const tokenBalances = await new MirrorNodeTokenBalanceQuery()
    .setAccountId(accountId)
    .execute(client);
// List<MirrorNodeTokenBalance>, each with .tokenId, .balance, .decimals
```
