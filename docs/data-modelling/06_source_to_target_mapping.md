# 06 — Source-to-Target Mapping

## 1. Purpose

This document traces each source field from the synthetic CSV extracts through the raw and staging layers into the reporting marts.

Pipeline:

```text
CSV
 ↓
raw
 ↓
staging
 ↓
marts
```

Invalid records are written to `audit.rejected_records`.

---

## 2. Dimension Load Strategy

The project uses **SCD Type 1** for:

```text
marts.dim_customer
marts.dim_account
```

### Type 1 Load Behaviour

For a new business key:

```text
business key not found
→ INSERT new dimension row
```

For an existing business key with changed descriptive attributes:

```text
business key found
→ UPDATE existing dimension row
```

No new historical version is created.

No `valid_from`, `valid_to`, `is_current`, or version columns are maintained.

---

## 3. Source Files

```text
customers.csv
accounts.csv
customer_accounts.csv
transactions.csv
```

---

## 4. Customers Mapping

| Source Field | Raw Target | Staging Target | Mart Target | Transformation / Rule |
|---|---|---|---|---|
| `customer_id` | `raw.customers.customer_id` | `staging.customers.customer_id` | `marts.dim_customer.customer_id` | Trim; reject null/blank; must be unique |
| `customer_since_date` | `raw.customers.customer_since_date` | `staging.customers.customer_since_date` | `marts.dim_customer.customer_since_date` | Parse to DATE; reject invalid date |
| `customer_status` | `raw.customers.customer_status` | `staging.customers.customer_status` | `marts.dim_customer.customer_status` | Trim + uppercase; allow `ACTIVE`, `INACTIVE` |

Mart-only field:

| Mart Field | Derivation |
|---|---|
| `customer_key` | Surrogate key generated in `marts.dim_customer` |

### `dim_customer` Type 1 Rule

If `customer_id` does not exist:

```text
INSERT new row
```

If `customer_id` already exists:

```text
UPDATE customer_since_date
UPDATE customer_status
```

The existing `customer_key` is retained.

---

## 5. Accounts Mapping

| Source Field | Raw Target | Staging Target | Mart Target | Transformation / Rule |
|---|---|---|---|---|
| `account_id` | `raw.accounts.account_id` | `staging.accounts.account_id` | `marts.dim_account.account_id` | Trim; reject null/blank; must be unique |
| `account_type` | `raw.accounts.account_type` | `staging.accounts.account_type` | `marts.dim_account.account_type` | Uppercase; allow `TRANSACTION`, `SAVINGS` |
| `account_status` | `raw.accounts.account_status` | `staging.accounts.account_status` | `marts.dim_account.account_status` | Uppercase; allow `ACTIVE`, `DORMANT`, `CLOSED` |
| `opened_date` | `raw.accounts.opened_date` | `staging.accounts.opened_date` | `marts.dim_account.opened_date` | Parse to DATE |
| `closed_date` | `raw.accounts.closed_date` | `staging.accounts.closed_date` | `marts.dim_account.closed_date` | Blank → NULL; otherwise parse DATE; must be >= opened_date |

Mart-only field:

| Mart Field | Derivation |
|---|---|
| `account_key` | Surrogate key generated in `marts.dim_account` |

### `dim_account` Type 1 Rule

If `account_id` does not exist:

```text
INSERT new row
```

If `account_id` already exists:

```text
UPDATE account_type
UPDATE account_status
UPDATE opened_date
UPDATE closed_date
```

The existing `account_key` is retained.

---

## 6. Customer Account Mapping

| Source Field | Raw Target | Staging Target | Mart Target | Transformation / Rule |
|---|---|---|---|---|
| `customer_id` | `raw.customer_accounts.customer_id` | `staging.customer_accounts.customer_id` | `marts.bridge_account_customer.customer_key` | Validate customer exists; lookup `customer_key` |
| `account_id` | `raw.customer_accounts.account_id` | `staging.customer_accounts.account_id` | `marts.bridge_account_customer.account_key` | Validate account exists; lookup `account_key` |
| `holder_role` | `raw.customer_accounts.holder_role` | `staging.customer_accounts.holder_role` | `marts.bridge_account_customer.holder_role` | Uppercase; allow `PRIMARY`, `JOINT` |

Derived mart field:

| Mart Field | Derivation |
|---|---|
| `allocation_weight` | `1.0 / number of valid holders for the account` |

Validation rules:

- customer-account pair must be unique
- referenced customer must exist
- referenced account must exist
- every account must have exactly one `PRIMARY` holder
- every account must have at least one valid holder

---

## 7. Transactions Mapping

| Source Field | Raw Target | Staging Target | Mart Target | Transformation / Rule |
|---|---|---|---|---|
| `transaction_id` | `raw.transactions.transaction_id` | `staging.transactions.transaction_id` | `marts.fact_transactions.transaction_id` | Trim; reject null/blank; must be unique |
| `account_id` | `raw.transactions.account_id` | `staging.transactions.account_id` | `marts.fact_transactions.account_key` | Validate account exists; lookup `account_key` |
| `transaction_timestamp` | `raw.transactions.transaction_timestamp` | `staging.transactions.transaction_timestamp` | `marts.fact_transactions.transaction_timestamp` | Parse timestamp; reject invalid |
| `transaction_type` | `raw.transactions.transaction_type` | `staging.transactions.transaction_type` | `marts.fact_transactions.transaction_type` | Uppercase; allow `PURCHASE`, `WITHDRAWAL`, `DEPOSIT`, `TRANSFER` |
| `channel_code` | `raw.transactions.channel_code` | `staging.transactions.channel_code` | `marts.fact_transactions.channel_key` | Uppercase; validate channel; lookup `channel_key` |
| `amount` | `raw.transactions.amount` | `staging.transactions.amount` | `marts.fact_transactions.amount` | Cast NUMERIC(18,2); must be > 0 |
| `currency_code` | `raw.transactions.currency_code` | `staging.transactions.currency_code` | `marts.fact_transactions.currency_code` | Uppercase; three characters; synthetic data uses `ZAR` |
| `status` | `raw.transactions.status` | `staging.transactions.status` | `marts.fact_transactions.status` | Uppercase; allow `SUCCESSFUL`, `FAILED`, `REVERSED` |

Derived mart fields:

| Mart Field | Derivation |
|---|---|
| `transaction_key` | Generated surrogate key |
| `date_key` | Integer `YYYYMMDD` derived from `transaction_timestamp::date` |

Additional validation:

- transaction account must exist
- timestamp cannot be after the account `closed_date`
- duplicates are rejected
- malformed numeric values are rejected

---

## 8. Channel Mapping

Channel data is derived from valid `channel_code` values in `transactions.csv`.

| Source Value | Staging Target | Mart Target |
|---|---|---|
| `APP` | `staging.channels` → Mobile App | `marts.dim_channel` |
| `ATM` | `staging.channels` → ATM | `marts.dim_channel` |
| `CARD` | `staging.channels` → Card | `marts.dim_channel` |

Mart-only field:

| Mart Field | Derivation |
|---|---|
| `channel_key` | Generated surrogate key |

---

## 9. Date Dimension Mapping

`marts.dim_date` has no source file.

It is generated from the transaction date range.

| Mart Field | Derivation |
|---|---|
| `date_key` | `YYYYMMDD` integer |
| `full_date` | Calendar date |
| `day_name` | Day name from full date |
| `month_number` | Month number |
| `month_name` | Month name |
| `quarter_number` | Calendar quarter |
| `year` | Calendar year |

---

## 10. Rejected Record Mapping

Any invalid source record is written to:

```text
audit.rejected_records
```

| Audit Field | Source / Derivation |
|---|---|
| `rejection_id` | Generated identity |
| `source_name` | Source file/entity name |
| `record_key` | Best available business key |
| `rejection_reason` | Validation message |
| `raw_payload` | Original row serialized to JSON |
| `rejected_at` | Current timestamp |

A rejected record does not proceed into staging or marts.
