# 01 — Data Modelling Foundation

## Banking Reporting Platform

This document records the agreed modelling foundation for the portfolio project. The scope is intentionally small, but the model includes realistic one-to-many and many-to-many relationships.

## 1. Business Process

The business process in scope is:

> **Processing retail banking transactions.**

A banking customer holds one or more accounts. An account can also be shared by more than one customer. Transactions are processed against accounts through defined channels.

The primary measurable business event is a **financial transaction**.

## 2. Model Scope

The model contains five core business entities plus one associative entity:

- Customer
- Account
- Customer Account
- Transaction
- Channel
- Customer Account is the associative entity used to resolve the many-to-many Customer–Account relationship.

The project intentionally excludes loans, fraud, AML, KYC, branches, employees, general ledger, insurance, investments, credit scoring, and real-time streaming.

## 3. Core Entities

### Customer

Represents a retail banking customer.

### Account

Represents a bank account. An account can be individually held or shared by multiple customers.

### Customer Account

Represents the ownership relationship between a customer and an account.

### Transaction

Represents one financial event processed against an account.

### Channel

Represents the mechanism used to initiate or process a transaction.

Initial channels:

- Mobile App
- ATM
- Card

## 4. Relationships and Cardinality

### Customer to Account — Many-to-Many

A customer may hold zero or many accounts.

An account must have one or more customers.

```text
CUSTOMER M:N ACCOUNT
```

The many-to-many relationship is resolved through `CUSTOMER_ACCOUNT`.

```text
CUSTOMER 1:M CUSTOMER_ACCOUNT
ACCOUNT  1:M CUSTOMER_ACCOUNT
```

Example:

```text
Customer A
├── Account 001
└── Account 002

Customer B
└── Account 002
```

`Account 002` is a joint account.

### Account to Transaction — One-to-Many

An account may have zero or many transactions.

Every transaction must belong to exactly one account.

```text
ACCOUNT 1:M TRANSACTION
```

### Channel to Transaction — One-to-Many

A channel may be used by zero or many transactions.

Every transaction must use exactly one valid channel.

```text
CHANNEL 1:M TRANSACTION
```

## 5. Grain

| Entity | Grain |
|---|---|
| Customer | One row per customer |
| Account | One row per account |
| Customer Account | One row per customer-account relationship |
| Transaction | One row per financial transaction |
| Channel | One row per transaction channel |

## 6. Business Keys

| Entity | Business Key |
|---|---|
| Customer | `customer_id` |
| Account | `account_id` |
| Customer Account | (`customer_id`, `account_id`) |
| Transaction | `transaction_id` |
| Channel | `channel_code` |

Surrogate keys are introduced later in the dimensional model.

## 7. Business Rules

1. `customer_id` must be unique and non-null.
2. `account_id` must be unique and non-null.
3. `transaction_id` must be unique and non-null.
4. `channel_code` must identify a valid channel.
5. A customer may be linked to zero or many accounts.
6. An account must be linked to at least one customer.
7. A customer-account pair must be unique.
8. Every account must have exactly one `PRIMARY` holder.
9. Additional holders on the same account use the `JOINT` holder role.
10. Every transaction must reference an existing account.
11. Every transaction must reference a valid channel.
12. Every transaction must contain a timestamp.
13. Transaction amount must be greater than zero.
14. Transaction status must be one of `SUCCESSFUL`, `FAILED`, or `REVERSED`.
15. Transaction type must be one of `PURCHASE`, `WITHDRAWAL`, `DEPOSIT`, or `TRANSFER`.
16. Currency code must use a three-character ISO-style code; the synthetic dataset uses `ZAR`.
17. An account may exist without transactions.
18. Historical transactions remain associated with the account on which they occurred.
19. A closed account may retain historical transactions, but a new transaction timestamp may not be later than its `closed_date`.
20. Invalid source records are rejected to the audit layer rather than silently discarded.

## 8. Source Files

The minimum synthetic source set is:

```text
customers.csv
accounts.csv
customer_accounts.csv
transactions.csv
```

`customer_accounts.csv` is required to represent joint accounts correctly.

Channel reference data is derived from `transactions.csv` using the distinct valid `channel_code` values.

## 9. Modelling Sequence

The modelling artifacts are organised as:

```text
01 Data Modelling Foundation
02 Conceptual Data Model
03 Logical Data Model
04 Physical Data Model
05 Dimensional Data Model
06 Source-to-Target Mapping
07 Data Dictionary
```

These seven documents define the accepted model for the current project scope.
