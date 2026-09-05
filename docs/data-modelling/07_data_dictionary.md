# 07 — Data Dictionary

## 1. Purpose

The data dictionary provides the field-level reference for the accepted physical and dimensional model.

---

## 2. Dimension History Convention

The project uses **SCD Type 1** for:

```text
marts.dim_customer
marts.dim_account
```

These dimensions represent the **current known state** of the customer and account.

When descriptive attributes change:

- the existing dimension row is updated
- the surrogate key remains unchanged
- prior attribute values are overwritten
- historical dimension versions are not stored

Therefore these dimensions do not contain:

```text
valid_from
valid_to
is_current
version_number
```

---

# RAW LAYER

## `raw.customers`

**Grain:** one source customer row.

| Column | Type | Description |
|---|---|---|
| `customer_id` | TEXT | Source customer identifier |
| `customer_since_date` | TEXT | Source customer relationship date |
| `customer_status` | TEXT | Source customer status |
| `_ingested_at` | TIMESTAMPTZ | Time the row was ingested |
| `_source_file` | TEXT | File from which the row originated |

## `raw.accounts`

**Grain:** one source account row.

| Column | Type | Description |
|---|---|---|
| `account_id` | TEXT | Source account identifier |
| `account_type` | TEXT | Source account type |
| `account_status` | TEXT | Source account status |
| `opened_date` | TEXT | Source account opening date |
| `closed_date` | TEXT | Source account closure date |
| `_ingested_at` | TIMESTAMPTZ | Time the row was ingested |
| `_source_file` | TEXT | File from which the row originated |

## `raw.customer_accounts`

**Grain:** one source customer-account relationship row.

| Column | Type | Description |
|---|---|---|
| `customer_id` | TEXT | Source customer identifier |
| `account_id` | TEXT | Source account identifier |
| `holder_role` | TEXT | Source ownership role |
| `_ingested_at` | TIMESTAMPTZ | Time the row was ingested |
| `_source_file` | TEXT | File from which the row originated |

## `raw.transactions`

**Grain:** one source transaction row.

| Column | Type | Description |
|---|---|---|
| `transaction_id` | TEXT | Source transaction identifier |
| `account_id` | TEXT | Source account identifier |
| `transaction_timestamp` | TEXT | Source transaction timestamp |
| `transaction_type` | TEXT | Source transaction type |
| `channel_code` | TEXT | Source channel code |
| `amount` | TEXT | Source transaction value |
| `currency_code` | TEXT | Source currency |
| `status` | TEXT | Source transaction status |
| `_ingested_at` | TIMESTAMPTZ | Time the row was ingested |
| `_source_file` | TEXT | File from which the row originated |

---

# STAGING LAYER

## `staging.customers`

**Grain:** one validated customer.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `customer_id` | VARCHAR(20) | PK | No | Customer business identifier |
| `customer_since_date` | DATE |  | No | Date customer relationship began |
| `customer_status` | VARCHAR(10) |  | No | `ACTIVE` or `INACTIVE` |
| `loaded_at` | TIMESTAMPTZ |  | No | Time loaded into staging |

## `staging.accounts`

**Grain:** one validated account.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `account_id` | VARCHAR(20) | PK | No | Account business identifier |
| `account_type` | VARCHAR(20) |  | No | `TRANSACTION` or `SAVINGS` |
| `account_status` | VARCHAR(10) |  | No | `ACTIVE`, `DORMANT`, or `CLOSED` |
| `opened_date` | DATE |  | No | Account opening date |
| `closed_date` | DATE |  | Yes | Account closure date |
| `loaded_at` | TIMESTAMPTZ |  | No | Time loaded into staging |

## `staging.customer_accounts`

**Grain:** one validated customer-account relationship.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `customer_id` | VARCHAR(20) | PK/FK | No | Customer identifier |
| `account_id` | VARCHAR(20) | PK/FK | No | Account identifier |
| `holder_role` | VARCHAR(10) |  | No | `PRIMARY` or `JOINT` |
| `loaded_at` | TIMESTAMPTZ |  | No | Time loaded into staging |

## `staging.channels`

**Grain:** one valid channel.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `channel_code` | VARCHAR(10) | PK | No | Channel code |
| `channel_name` | VARCHAR(50) |  | No | Human-readable channel |

## `staging.transactions`

**Grain:** one validated transaction.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `transaction_id` | VARCHAR(30) | PK | No | Transaction business identifier |
| `account_id` | VARCHAR(20) | FK | No | Linked account |
| `channel_code` | VARCHAR(10) | FK | No | Linked channel |
| `transaction_timestamp` | TIMESTAMP |  | No | Transaction date and time |
| `transaction_type` | VARCHAR(20) |  | No | Transaction type |
| `amount` | NUMERIC(18,2) |  | No | Positive transaction value |
| `currency_code` | CHAR(3) |  | No | Currency code |
| `status` | VARCHAR(15) |  | No | Processing status |
| `loaded_at` | TIMESTAMPTZ |  | No | Time loaded into staging |

---

# AUDIT LAYER

## `audit.rejected_records`

**Grain:** one rejected source record.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `rejection_id` | BIGINT IDENTITY | PK | No | Generated rejection identifier |
| `source_name` | VARCHAR(50) |  | No | Source entity/file |
| `record_key` | TEXT |  | Yes | Best available business key |
| `rejection_reason` | TEXT |  | No | Reason the row failed validation |
| `raw_payload` | JSONB |  | No | Original source row |
| `rejected_at` | TIMESTAMPTZ |  | No | Rejection timestamp |

---

# MART LAYER

## `marts.dim_customer`

**Grain:** one current customer.

**History strategy:** SCD Type 1.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `customer_key` | BIGINT IDENTITY | PK | No | Surrogate customer key retained across Type 1 updates |
| `customer_id` | VARCHAR(20) | UK | No | Customer business identifier used to match incoming rows |
| `customer_since_date` | DATE |  | No | Current recorded date customer relationship began |
| `customer_status` | VARCHAR(10) |  | No | Current customer status |

**Change behaviour:** if an incoming row has the same `customer_id`, changed descriptive attributes overwrite the current values.

## `marts.dim_account`

**Grain:** one current account.

**History strategy:** SCD Type 1.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `account_key` | BIGINT IDENTITY | PK | No | Surrogate account key retained across Type 1 updates |
| `account_id` | VARCHAR(20) | UK | No | Account business identifier used to match incoming rows |
| `account_type` | VARCHAR(20) |  | No | Current account type |
| `account_status` | VARCHAR(10) |  | No | Current account status |
| `opened_date` | DATE |  | No | Account opening date |
| `closed_date` | DATE |  | Yes | Current recorded account closure date |

**Change behaviour:** if an incoming row has the same `account_id`, changed descriptive attributes overwrite the current values.

## `marts.dim_channel`

**Grain:** one transaction channel.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `channel_key` | SMALLINT IDENTITY | PK | No | Surrogate channel key |
| `channel_code` | VARCHAR(10) | UK | No | Business channel code |
| `channel_name` | VARCHAR(50) |  | No | Channel display name |

## `marts.dim_date`

**Grain:** one calendar day.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `date_key` | INTEGER | PK | No | `YYYYMMDD` date key |
| `full_date` | DATE | UK | No | Calendar date |
| `day_name` | VARCHAR(10) |  | No | Day of week |
| `month_number` | SMALLINT |  | No | Month number 1–12 |
| `month_name` | VARCHAR(10) |  | No | Month name |
| `quarter_number` | SMALLINT |  | No | Quarter 1–4 |
| `year` | SMALLINT |  | No | Calendar year |

## `marts.bridge_account_customer`

**Grain:** one account-customer relationship.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `account_key` | BIGINT | PK/FK | No | Linked account surrogate key |
| `customer_key` | BIGINT | PK/FK | No | Linked customer surrogate key |
| `holder_role` | VARCHAR(10) |  | No | `PRIMARY` or `JOINT` |
| `allocation_weight` | NUMERIC(8,6) |  | No | Equal allocation across account holders |

## `marts.fact_transactions`

**Grain:** one validated financial transaction.

| Column | Type | Key | Nullable | Description |
|---|---|---|---:|---|
| `transaction_key` | BIGINT IDENTITY | PK | No | Surrogate transaction key |
| `transaction_id` | VARCHAR(30) | UK | No | Source transaction identifier |
| `account_key` | BIGINT | FK | No | Account dimension key |
| `channel_key` | SMALLINT | FK | No | Channel dimension key |
| `date_key` | INTEGER | FK | No | Date dimension key |
| `transaction_timestamp` | TIMESTAMP |  | No | Exact transaction timestamp |
| `transaction_type` | VARCHAR(20) |  | No | Transaction type |
| `amount` | NUMERIC(18,2) |  | No | Positive transaction value |
| `currency_code` | CHAR(3) |  | No | Transaction currency |
| `status` | VARCHAR(15) |  | No | `SUCCESSFUL`, `FAILED`, or `REVERSED` |

---

# DOMAIN VALUES

## Customer Status

```text
ACTIVE
INACTIVE
```

## Account Type

```text
TRANSACTION
SAVINGS
```

## Account Status

```text
ACTIVE
DORMANT
CLOSED
```

## Holder Role

```text
PRIMARY
JOINT
```

## Transaction Type

```text
PURCHASE
WITHDRAWAL
DEPOSIT
TRANSFER
```

## Transaction Status

```text
SUCCESSFUL
FAILED
REVERSED
```

## Channel

| Code | Name |
|---|---|
| `APP` | Mobile App |
| `ATM` | ATM |
| `CARD` | Card |

## Currency

The synthetic project dataset uses:

```text
ZAR
```
