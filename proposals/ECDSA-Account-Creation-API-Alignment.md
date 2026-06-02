# ECDSA Account Creation API Alignment

**Date Submitted:** 2026-06-01

---

## Summary

Creating an ECDSA Hedera account with an EVM Address derived from the public key is the recommended pattern for EVM-compatible applications, yet the API surface for doing this diverges significantly across all seven Hiero SDKs. The divergences cause AI code generators to produce incorrect cross-SDK translations, block legitimate HSM/key-separation use cases in three SDKs, and require per-SDK workarounds that are not documented anywhere canonical.

This proposal was produced from a direct audit of `hiero-sdk-{js,java,python,rust,cpp,swift,go}` (May 2026). It documents every cross-SDK inconsistency and proposes the minimum set of additive, non-breaking changes needed to bring the SDKs into alignment. Where an alignment requires a breaking change, it is called out explicitly.

---

## Current State

### 1. Key-and-alias method names diverge across all seven SDKs

| SDK | Key-and-alias method |
|-----|-------------------------------|
| JavaScript | `setECDSAKeyWithAlias(key)` |
| Java | `setKeyWithAlias(key)` *(1-arg overload — no `setECDSAKeyWithAlias` exists)* |
| Python | `set_key_with_alias(key)` |
| Rust | `set_ecdsa_key_with_alias(key)` |
| C++ | `setECDSAKeyWithAlias(key)` |
| Swift | `keyWithAlias(privateKeyECDSA:)` *(no `set` prefix — Swift convention)* |
| Go | `SetECDSAKeyWithAlias(key)` |

Five of seven SDKs use a name in the family `setECDSAKeyWithAlias` (camelCase or PascalCase per language convention). Java and Python are outliers.

**Critical semantic trap — Java vs. JavaScript:** `setKeyWithAlias(key)` with a single argument means completely different things in the two SDKs:
- **Java:** the recommended key-and-alias path.
- **JavaScript:** a two-argument method; a one-argument call is a runtime error.

AI agents translating Java code to JavaScript (or vice versa) produce broken code silently.

### 2. Full method matrix on `AccountCreateTransaction`

| Operation | JS | Java | Python | Rust | C++ | Swift | Go |
|--------------------------------------------------|---|---|---|---|---|---|---|
| Sets key + derives EVM address (key-and-alias) | `setECDSAKeyWithAlias(k)` | `setKeyWithAlias(k)` | `set_key_with_alias(k)` | `set_ecdsa_key_with_alias(k)` | `setECDSAKeyWithAlias(k)` | `keyWithAlias(k)` | `SetECDSAKeyWithAlias(k)` |
| Two-key: account key + separate ECDSA alias key | `setKeyWithAlias(k, a)` | `setKeyWithAlias(k, a)` | `set_key_with_alias(k, a)` | `set_key_with_alias(k, a)` | `setKeyWithAlias(k, a)` | `keyWithAlias(k, a)` | `SetKeyWithAlias(k, a)` |
| Key without alias | `setKeyWithoutAlias(k)` | `setKeyWithoutAlias(k)` | `set_key_without_alias(k)` | `set_key_without_alias(k)` | `setKeyWithoutAlias(k)` | `keyWithoutAlias(k)` | `SetKeyWithoutAlias(k)` |
| Explicit alias | `setAlias(addr)` | `setAlias(addr)` | `set_alias(addr)` | `alias(addr)` | `setAlias(addr)` | `alias(addr)` | `SetAlias(hex)` |
| Deprecated key setter | `setKey(k)` ⚠️ | `setKey(k)` ⚠️ | `set_key(k)` ⚠️ | `key(k)` ⚠️ | `setKey(k)` ⚠️ | `key(k)` ⚠️ | `SetKey(k)` ⚠️ |


### 3. Key-and-alias method rejects `PublicKey` in three SDKs

| SDK | Accepts `PublicKey`? | Failure mode |
|-----|----------------------|--------------|
| JavaScript | ✅ | — |
| Java | ✅ | — |
| Python | ✅ | — |
| Go | ✅ | — |
| Rust | ❌ | Panics |
| C++ | ❌ | `invalid_argument` |
| Swift | ❌ | Throws `HError` |

**Impact:** Hardware wallets, HSMs, and any key-separation architecture that holds only the public key cannot use the key-and-alias method in Rust, C++, or Swift. The manual `toEvmAddress()` + `setAlias()` two-step is required, but this path is not documented in any of the three SDKs.

### 4. `PublicKey.toEvmAddress()` — class location and return type diverge

| SDK | Class | Return type | Non-ECDSA behavior |
|-----|-------|-------------|---------------------|
| JavaScript | `PublicKey` | `string` (hex, no `0x`) | Returns `undefined` |
| Java | `PublicKey` (abstract) | `EvmAddress` | Throws `IllegalStateException` |
| Python | `PublicKey` | `EvmAddress` | Raises `ValueError` |
| Rust | `PublicKey` | `Option<EvmAddress>` | Returns `None` |
| C++ | `ECDSAsecp256k1PublicKey` **only** | `EvmAddress` | N/A — typed out of existence |
| Swift | `PublicKey` | `EvmAddress?` | Returns `nil` |
| Go | `PublicKey` | `string` (hex, no `0x`) | Returns `""` |

**C++ outlier:** `toEvmAddress()` lives on the typed subclass `ECDSAsecp256k1PublicKey`, not on the abstract `PublicKey`. Code holding a generic `PublicKey*` pointer must downcast before calling the helper, which is a common source of compilation errors.

**Return type split:** Five SDKs return a structured `EvmAddress` type; JS and Go return a raw hex string without a `0x` prefix.

### 5. `AccountId.toEvmAddress()` — missing or renamed in two SDKs

| SDK | Status |
|-----|--------|
| JavaScript | ✅ `accountId.toEvmAddress()` |
| Java | ✅ `accountId.toEvmAddress()` (`toSolidityAddress()` deprecated) |
| Python | ✅ `account_id.to_evm_address()` |
| Swift | ✅ `accountId.toEvmAddress()` |
| Go | ✅ `accountId.ToEvmAddress()` |
| **Rust** | ❌ **Not implemented** (`from_evm_address()` exists as a constructor, but no inverse) |
| **C++** | ⚠️ `accountId.toSolidityAddress()` (different name; `toEvmAddress()` does not exist) |

### 6. `PrivateKey.generateECDSA()` — four different patterns

| SDK | Call | Issue |
|-----|------|-------|
| JavaScript | `PrivateKey.generateECDSA()` | — |
| Java | `PrivateKey.generateECDSA()` | — |
| Python | `PrivateKey.generate_ecdsa()` | Snake_case, expected |
| Rust | `PrivateKey::generate_ecdsa()` | Snake_case, expected |
| **C++** | `ECDSAsecp256k1PrivateKey::generatePrivateKey()` | Different class + method name; no `PrivateKey::generateECDSA()` |
| **Swift** | `PrivateKey.generateEcdsa()` | `Ecdsa` casing (not `ECDSA`) |
| **Go** | `PrivateKeyGenerateEcdsa()` | Package-level function, not a method |

### 7. Broken Python official example

`examples/account/account_create_transaction_evm_alias.py` uses the deprecated `set_key()` method instead of `set_key_with_alias()`. This is the first result for many developers looking for reference code.

### 8. Example and deprecation status

| SDK | Example file | Status |
|-----|-------------|--------|
| JavaScript | `examples/create-account-with-alias.js` | ✅ Correct |
| Java | Integration tests only | ✅ Correct |
| **Python** | `examples/account/account_create_transaction_evm_alias.py` | ❌ Uses deprecated `set_key()` |
| Rust | Unit tests only | ✅ Correct |
| C++ | `CreateAccountExample.cpp` | ✅ Correct |
| **Swift** | None found | ❌ Missing |
| Go | `examples/create_account_with_alias/main.go` | ✅ Correct |

All 7 SDKs have deprecated the bare key setter (`setKey()` / `key()`) in favour of the `*WithAlias` / `*WithoutAlias` variants using their language's idiomatic deprecation annotation.

---

## Proposed Changes

Changes are grouped by priority. All changes in **P0** and **P1** are additive (non-breaking). Changes that would be breaking are listed separately under **P2** with explicit warnings.

### P0 — Fix broken state (non-breaking)

#### P0-1: Fix the Python official example [hiero-sdk-python]

`examples/account/account_create_transaction_evm_alias.py` calls the deprecated `set_key()` method. Replace with `set_key_with_alias()`.

```python
# Before (broken — uses deprecated API)
tx = AccountCreateTransaction().set_key(account_public_key)

# After (correct)
tx = AccountCreateTransaction().set_key_with_alias(account_private_key)
```

This is a pure example fix. No API changes required.

#### P0-2: Add a standalone Swift example [hiero-sdk-swift]

Add `examples/CreateAccountWithAlias.swift` covering the standard ECDSA + alias creation flow. No API changes required. See the [SDK Example](#sdk-example) section for the proposed code.

---

### P1 — Add missing APIs (non-breaking, additive)

#### P1-1: Add `setECDSAKeyWithAlias(Key)` to Java [hiero-sdk-java]

Java is the only SDK with a key-and-alias method not in the `setECDSAKeyWithAlias` family. Add it as an overload that delegates to the existing 1-arg `setKeyWithAlias(Key)`. The existing method is unchanged; no migration is required.

**Proposed API addition:**

```java
/**
 * Sets an ECDSA key and derives the EVM Address from the public key,
 * setting it as the account alias. Equivalent to {@code setKeyWithAlias(key)}.
 *
 * @param key an ECDSA key (PublicKey or PrivateKey)
 * @return this
 */
public AccountCreateTransaction setECDSAKeyWithAlias(Key key) {
    return setKeyWithAlias(key);
}
```

**Why:** Resolves the Java/JS semantic trap at the call site. Developers reading multi-SDK docs or AI-generated code can use the same name across JS, Java, C++, Go, and Rust without triggering a runtime error.

**Breaking:** No — purely additive.

---

#### P1-2: Add `AccountId.to_evm_address()` to Rust [hiero-sdk-rust]

Rust is the only SDK that implements `from_evm_address()` but not its inverse. Without this, a Rust developer who retrieves an `AccountId` from a receipt cannot programmatically verify what EVM address the account resolves to.

**Proposed behavior:** Mirror the specification from the [`remove-shard-and-realm-encoding-from-evm-address-generation` proposal](https://github.com/hiero-ledger/sdk-collaboration-hub/blob/main/proposals/remove-shard-and-realm-encoding-from-evm-address-generation.md):
- If the `AccountId` has an alias-style EVM address, return it directly.
- Otherwise, return 12 zero bytes + the 8-byte big-endian account number.

```rust
impl AccountId {
    /// Returns the EVM address for this account ID.
    ///
    /// If the account was created with an EVM Address from Public Key alias,
    /// returns that alias. Otherwise returns the long-zero form
    /// (12 zero bytes + 8-byte big-endian account number).
    pub fn to_evm_address(&self) -> EvmAddress {
        // implementation
    }
}
```

**Breaking:** No — purely additive.

---

#### P1-3: Add `AccountId.toEvmAddress()` to C++ [hiero-sdk-cpp]

C++ exposes `toSolidityAddress()` for the same concept. Add `toEvmAddress()` as a preferred alias and deprecate `toSolidityAddress()`. This aligns with the direction already taken in Java.

```cpp
/// Returns the EVM address for this AccountId.
/// Preferred replacement for toSolidityAddress().
std::string toEvmAddress() const;

/// @deprecated Use toEvmAddress() instead.
[[deprecated("Use toEvmAddress() instead")]]
std::string toSolidityAddress() const;
```

**Breaking:** No — `toSolidityAddress()` is retained (deprecated, not removed).

---

#### P1-4: Expose `toEvmAddress()` on the abstract `PublicKey` class in C++ [hiero-sdk-cpp]

Currently `toEvmAddress()` is only on `ECDSAsecp256k1PublicKey`. Code holding a generic `PublicKey*` must downcast. Add a virtual method to the base class that returns an empty/null result for non-ECDSA keys, consistent with how JS returns `undefined` and Go returns `""`.

```cpp
/// Returns the EVM Address derived from this public key,
/// or std::nullopt if this key is not an ECDSA secp256k1 key.
virtual std::optional<EvmAddress> toEvmAddress() const { return std::nullopt; }
```

Override in `ECDSAsecp256k1PublicKey` with the actual Keccak-256 derivation.

**Breaking:** No — virtual method addition with a default implementation is backward compatible. Callers already on the typed subclass are unaffected.

---

#### P1-5: Accept `PublicKey` in the key-and-alias method for Rust, C++, and Swift [hiero-sdk-rust, hiero-sdk-cpp, hiero-sdk-swift]

Add an overload (or generalize the signature) of the key-and-alias method to accept a `PublicKey` in addition to the existing `PrivateKey` path. This unblocks HSM and key-separation flows in these SDKs.

**Rust:**
```rust
// Existing — retained unchanged
pub fn set_ecdsa_key_with_alias(&mut self, key: PrivateKey) -> &mut Self;

// Proposed addition
pub fn set_ecdsa_key_with_alias_from_public_key(&mut self, key: PublicKey) -> &mut Self;
```

> **Note:** If the Rust builder pattern does not easily support overloading, a separate method name (`set_ecdsa_key_with_alias_from_public_key`) is preferred over changing the existing signature. Name is open for SDK team input.

**C++:**
```cpp
// Existing — retained unchanged
AccountCreateTransaction& setECDSAKeyWithAlias(const PrivateKey& key);

// Proposed addition
AccountCreateTransaction& setECDSAKeyWithAlias(
    std::shared_ptr<ECDSAsecp256k1PublicKey> key);
```

**Swift:**
```swift
// Existing — retained unchanged
func keyWithAlias(privateKeyECDSA: PrivateKey) -> AccountCreateTransaction

// Proposed addition
func keyWithAlias(publicKeyECDSA: PublicKey) -> AccountCreateTransaction
```

**Breaking:** No — all existing call sites continue to work unchanged; the new overloads are purely additive.

---

## New / Updated APIs (summary)

### New APIs (non-breaking)

| SDK | API | Description |
|-----|-----|-------------|
| Java | `AccountCreateTransaction.setECDSAKeyWithAlias(Key)` | Alias for `setKeyWithAlias(Key)` (1-arg) |
| Rust | `AccountId.to_evm_address() -> EvmAddress` | Missing inverse of `from_evm_address()` |
| C++ | `AccountId.toEvmAddress() -> std::string` | Preferred alias for `toSolidityAddress()` |
| C++ | `PublicKey.toEvmAddress() -> std::optional<EvmAddress>` | Virtual base-class method; ECDSA subclass overrides |
| Rust | `AccountCreateTransaction.set_ecdsa_key_with_alias_from_public_key(PublicKey)` | PublicKey overload for HSM flows |
| C++ | `AccountCreateTransaction.setECDSAKeyWithAlias(shared_ptr<ECDSAsecp256k1PublicKey>)` | PublicKey overload for HSM flows |
| Swift | `AccountCreateTransaction.keyWithAlias(publicKeyECDSA: PublicKey)` | PublicKey overload for HSM flows |

### Deprecated APIs (retain, mark deprecated)

| SDK | API | Replacement |
|-----|-----|-------------|
| C++ | `AccountId.toSolidityAddress()` | `AccountId.toEvmAddress()` |

---

## SDK Example

### Creating an ECDSA account with EVM Address from Public Key — all seven SDKs

```javascript
// JavaScript
const privateKey = PrivateKey.generateECDSA();
const response = await new AccountCreateTransaction()
  .setECDSAKeyWithAlias(privateKey)       // sets key + derives EVM address
  .setInitialBalance(new Hbar(1))
  .execute(client);
const { accountId } = await response.getReceipt(client);

// Verify
const info = await new AccountInfoQuery().setAccountId(accountId).execute(client);
console.log("EVM address:", "0x" + info.contractAccountId);
```

```java
// Java
PrivateKey privateKey = PrivateKey.generateECDSA();
TransactionResponse response = new AccountCreateTransaction()
  .setECDSAKeyWithAlias(privateKey)       // NEW: was setKeyWithAlias(privateKey)
  .setInitialBalance(new Hbar(1))
  .execute(client);
AccountId accountId = response.getReceipt(client).accountId;

// Verify
AccountInfo info = new AccountInfoQuery().setAccountId(accountId).execute(client);
System.out.println("EVM address: 0x" + info.contractAccountId);
```

```python
# Python
private_key = PrivateKey.generate_ecdsa()
response = (
    AccountCreateTransaction()
    .set_key_with_alias(private_key)
    .set_initial_balance(Hbar(1))
    .execute(client)
)
account_id = response.get_receipt(client).account_id

# Verify
info = AccountInfoQuery().set_account_id(account_id).execute(client)
print(f"EVM address: 0x{info.contract_account_id}")
```

```rust
// Rust
let private_key = PrivateKey::generate_ecdsa();
let response = AccountCreateTransaction::new()
    .set_ecdsa_key_with_alias(private_key.clone())
    .initial_balance(Hbar::new(1))
    .execute(&client)
    .await?;
let account_id = response.get_receipt(&client).await?.account_id.unwrap();

// Verify
let info = AccountInfoQuery::new()
    .account_id(account_id)
    .execute(&client)
    .await?;
println!("EVM address: 0x{}", info.contract_account_id);
```

```cpp
// C++
auto privateKey = ECDSAsecp256k1PrivateKey::generatePrivateKey();
auto response = AccountCreateTransaction()
    .setECDSAKeyWithAlias(privateKey)
    .setInitialBalance(Hbar(1, HbarUnit::HBAR()))
    .execute(client);
AccountId accountId = response.getReceipt(client).mAccountId.value();

// Verify
AccountInfo info = AccountInfoQuery().setAccountId(accountId).execute(client);
std::cout << "EVM address: 0x" << info.mContractAccountId << "\n";
```

```swift
// Swift
let privateKey = PrivateKey.generateEcdsa()
let response = try await AccountCreateTransaction()
    .keyWithAlias(privateKeyECDSA: privateKey)
    .initialBalance(Hbar(1))
    .execute(client)
let accountId = try await response.getReceipt(client).accountId!

// Verify
let info = try await AccountInfoQuery().accountId(accountId).execute(client)
print("EVM address: 0x\(info.contractAccountId)")
```

```go
// Go
privateKey, _ := hedera.PrivateKeyGenerateEcdsa()
response, _ := hedera.NewAccountCreateTransaction().
    SetECDSAKeyWithAlias(privateKey).
    SetInitialBalance(hedera.NewHbar(1)).
    Execute(client)
accountId, _ := response.GetReceipt(client)

// Verify
info, _ := hedera.NewAccountInfoQuery().SetAccountID(accountId.AccountID).Execute(client)
fmt.Printf("EVM address: 0x%s\n", info.ContractAccountID)
```

### HSM flow — using the new `PublicKey` overload (Rust example)

```rust
// Rust — HSM flow (only public key available)
let public_key: PublicKey = hsm_get_public_key();      // typed ECDSA public key from HSM
let response = AccountCreateTransaction::new()
    .set_ecdsa_key_with_alias_from_public_key(public_key.clone())  // NEW P1-5 overload
    .initial_balance(Hbar::new(1))
    .execute(&client)
    .await?;
```

---
