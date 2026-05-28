# TTL-Legacy API Reference

## Smart Contract API

### Vault Cloning

#### `clone_vault`

```rust
clone_vault(source_vault_id: u64, new_owner: Address, new_beneficiary: Address) -> u64
```

Clones all settings from a source vault into a new vault. All fields (interval, beneficiaries, metadata, token, release condition, spending limits) are copied verbatim.

---

#### `clone_vault_with_overrides`

```rust
clone_vault_with_overrides(
    source_vault_id: u64,
    new_owner: Address,
    new_beneficiary: Address,
    override_interval: Option<u64>,
    override_beneficiaries: Option<Vec<BeneficiaryEntry>>,
    override_metadata: Option<String>,
) -> u64
```

Clones a vault configuration into a new vault with selective parameter overrides. Fields passed as `None` are copied from the source vault unchanged.

**Parameters**

| Parameter                | Type                           | Description                                                                  |
|--------------------------|--------------------------------|------------------------------------------------------------------------------|
| `source_vault_id`        | `u64`                          | Template vault (must be Locked and owned by `new_owner`)                     |
| `new_owner`              | `Address`                      | Owner of the new vault (must authorize; must be source vault owner)          |
| `new_beneficiary`        | `Address`                      | Primary beneficiary for the new vault                                        |
| `override_interval`      | `Option<u64>`                  | Override check-in interval in seconds, or `None` to copy from source         |
| `override_beneficiaries` | `Option<Vec<BeneficiaryEntry>>`| Override multi-beneficiary split (BPS must sum to 10 000), or `None` to copy |
| `override_metadata`      | `Option<String>`               | Override metadata string (max 256 chars), or `None` to copy                  |

**Returns** the new vault ID.

**Errors**

| Error                | Code | Condition                                               |
|----------------------|------|---------------------------------------------------------|
| `NotOwner`           | 6    | Caller is not the source vault owner                    |
| `AlreadyReleased`    | 7    | Source vault is not in Locked status                    |
| `InvalidBeneficiary` | 17   | `new_owner == new_beneficiary`, or owner in BPS list    |
| `InvalidInterval`    | 2    | `override_interval` is `Some(0)`                        |
| `IntervalTooLow`     | 14   | `override_interval` below configured minimum            |
| `IntervalTooHigh`    | 15   | `override_interval` above configured maximum            |
| `InvalidBps`         | 12   | `override_beneficiaries` BPS sum ≠ 10 000               |
| `InvalidAmount`      | 5    | `override_metadata` exceeds 256 characters              |

**Event emitted:** `v_clo_ov` — `(source_vault_id, new_vault_id, new_beneficiary, check_in_interval)`

**Comparison with `clone_vault`**

| Feature                   | `clone_vault` | `clone_vault_with_overrides` |
|---------------------------|:---:|:---:|
| Copies interval           | ✅  | ✅ (or override) |
| Copies beneficiaries      | ✅  | ✅ (or override) |
| Copies metadata           | ✅  | ✅ (or override) |
| Copies token / conditions | ✅  | ✅               |
| Selective field override  | ❌  | ✅               |

---

## Backend API

Base URL: `http://localhost:3000`

### Reminder Preferences

#### POST `/api/vaults/{vault_id}/reminder-preferences`

Create or update reminder preferences for a vault.

**Request Body** (`application/json`)

| Field                 | Type            | Required | Description                                           |
|-----------------------|-----------------|----------|-------------------------------------------------------|
| `channels`            | array of string | Yes      | One or more of: `"email"`, `"sms"`, `"push"`         |
| `hours_before_expiry` | integer (> 0)   | Yes      | Hours before TTL expiry to send the first reminder    |
| `frequency`           | string          | Yes      | `"once"`, `"daily"`, or `"hourly"`                   |

**Responses:** `200` saved object · `422` validation error · `500` server error

---

#### GET `/api/vaults/{vault_id}/reminder-preferences`

Retrieve current reminder preferences for a vault.

**Responses:** `200` preferences object · `404` not found · `500` server error

---

### Scheduler Behaviour

The background scheduler polls every 60 seconds. For each vault with stored preferences it:

1. Fetches the vault's TTL remaining (hours) from the Stellar RPC.
2. Compares against `hours_before_expiry`.
3. Fires reminders on the configured channels according to `frequency`:
   - `once` — fires exactly once when TTL enters the window.
   - `daily` — fires every 24 hours while inside the window.
   - `hourly` — fires every hour while inside the window.

Preferences are stored off-chain in the backend SQLite database and are never written to the Soroban contract.
