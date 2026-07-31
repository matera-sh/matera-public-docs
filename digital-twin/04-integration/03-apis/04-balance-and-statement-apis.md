# DTW Balance & Statement REST API — Integration Guide

**Audience:** Backend engineers integrating read-only balance queries and account statements into applications that consume DTW ledger data.

---

## Balance and Statement APIs

The DTW Balance & Statement service provides read-only APIs for querying account balances, limits, and statement data.

This service is composed of **two independent HTTP applications** deployed under the same service boundary:

| Application | Responsibility |
|---|---|
| `balance-api` | Current / historical balances and limits per account |
| `entry-api` | Account statement (entry history); entry metadata |

All data is materialized from the event stream processed by DTW Transaction. There is no write path in this service.

### When to use this service vs. others

| Need | Service |
|---|---|
| Post or reverse ledger entries | [Transaction REST API](./02-dtw-transaction-api-integration-guide.md) |
| Create or release monitorings (holds) | [Transaction REST API](./02-dtw-transaction-api-integration-guide.md) |
| Query account balance at a date range | **Balance API** (this guide) |
| List posted/pending entries for an account | **Entry API** (this guide) |

---

## Table of Contents

**Shared Building Blocks**

1. [Authentication & Authorization](#1-authentication--authorization)
2. [Building Blocks](#2-building-blocks)

**API Catalog — Balance API**

3. [API Catalog — Balance API](#3-api-catalog--balance-api)
   - [GET /accounts/{branch}/{account}/balances — Get Account Balances](#32-get-account-balances)
   - [GET /accounts/{branch}/{account}/limits — Get Account Limits](#33-get-account-limits)
   - [GET /balances — Batch Balances (Multiple Accounts)](#34-batch-balances-multiple-accounts)

**API Catalog — Entry API**

4. [API Catalog — Entry API](#4-api-catalog--entry-api)
   - [GET /v2/entries — Get Entries](#42-get-entries)
   - [GET /v1/entries/metadata — Get Entry Metadata](#43-get-entry-metadata)

**Operations**

5. [Pagination](#5-pagination)
6. [Error Reference](#6-error-reference)

**Appendix**

7. [Appendix](#7-appendix)
8. [Advanced Integration Topics](#advanced-integration-topics)
   - [OpenAPI Spec](#openapi-spec)

---

## 1. Authentication & Authorization

All endpoints require a JWT Bearer token issued by Keycloak. Tokens must be sent in the `Authorization: Bearer <token>` header.

Roles are **client-level authorities** on the `DTW` Keycloak client — not realm roles.

### Required roles by endpoint

| Role | Grants access to |
|---|---|
| `view-account-balances` | `GET /accounts/{branch}/{account}/balances` |
| `view-account-limits` | `GET /accounts/{branch}/{account}/limits` |
| `view-balances` | `GET /balances` (batch multi-account) |
| `view-entries` | `GET /v2/entries` |
| `view-entries-metadata` | `GET /v1/entries/metadata` |

A single OAuth2 client may hold multiple roles. All roles are read-only; there are no write roles on this service.

---

## 2. Building Blocks

### 2.1 Account Key

An account is identified by two path segments:

| Segment | Type | Description |
|---|---|---|
| `branch` | string | Branch code (e.g. `0001`) |
| `account` | string | Account number within the branch |

### 2.2 Balance Types

Balance types are domain-defined string codes representing the kind of balance being queried (e.g. `DEBIT_BALANCE`, `CREDIT_BALANCE`, `AVAILABLE_BALANCE`). The valid set of types is configured in the DTW core; consult your DTW administrator for the complete list applicable to your product.

### 2.3 The `Entry` Model

Entries are the atomic records of a statement. Every entry belongs to one account and represents a single ledger movement.

| Field | Type | Description |
|---|---|---|
| `branch` | string | Branch code of the account |
| `account` | string | Account number |
| `entryDate` | date | Accounting date of the entry |
| `eventDate` | date | Business event date |
| `operationType` | enum | `C` (credit) or `D` (debit) |
| `amount` | decimal | Absolute monetary value |
| `currency` | string | ISO 4217 currency code |
| `entryId` | string | System-generated unique entry identifier |
| `entryKind` | enum | See §2.4 |
| `entryDescription` | string | Human-readable description |
| `historyCode` | string | Ledger history / GL code |
| `historyDescription` | string | Description of the history code |
| `metadata` | map\<string, string\> | Arbitrary key-value pairs attached at posting time. Note: the metadata lookup endpoint (`GET /v1/entries/metadata`) returns this map under the key `metaData` (camelCase) — the two spellings reflect different schemas in the same API |
| `correlations` | map\<string, string\> | Cross-references to related systems — maps originator code to the caller-assigned correlation ID for that system |
| `compensatedEntry` | object (recursive) | The original entry this one compensates, if applicable |
| `holdReasonId` | string | Identifier for the hold reason (monitoring entries) |
| `holdReason` | string | Human-readable hold reason description |
| `blockingKind` | enum | `CAUTIONARY` or `BALANCE` (blocking entries only) |

### 2.4 `entryKind` Enum

| Value | Description |
|---|---|
| `LEDGER` | Standard posted ledger entry |
| `BLOCKING` | Hold / monitoring debit or credit |
| `UNBLOCKING` | Release of a previous blocking entry |
| `REVERSAL` | Reversal of a previously posted entry |
| `REMOVAL` | Administrative removal of an entry |
| `EXCLUSION` | Entry marked as excluded from balances |
| `ADJUSTMENT_ENTRY` | Corrective adjustment entry |
| `PENDING_DEBIT` | Pending debit not yet posted |
| `PENDING_CREDIT` | Pending credit not yet posted |

### 2.5 Balance Summary

Returned alongside the entry list in `balanceSummaryV2`:

| Field | Type | Description |
|---|---|---|
| `sumOfPostedEntries` | decimal | Net of all posted entries in the requested date range |
| `sumOfPendingDebitEntries` | decimal | Sum of pending debit amounts in the requested date range |
| `sumOfPendingCreditEntries` | decimal | Sum of pending credit amounts in the requested date range |

### 2.6 Pagination Response Headers

List endpoints include pagination metadata in response headers unless suppressed by the `X-Count-Omit: true` request header.

| Header | Description |
|---|---|
| `X-Total-Elements` | Total number of matching records |
| `X-Total-Pages` | Total number of pages |
| `X-Has-Next` | `true` if more pages follow the current one |
| `X-Count-Limit-Exceeded` | `true` if the result set exceeds the system count limit |

---

## 3. API Catalog — Balance API

Base URL: `http://balance-api`

### 3.1 Endpoint Overview

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/accounts/{branch}/{account}/balances` | `view-account-balances` | Account balances by date range and balance type |
| GET | `/accounts/{branch}/{account}/limits` | `view-account-limits` | Account limits by date range |
| GET | `/balances` | `view-balances` | Batch balances for multiple accounts |

---

### 3.2 Get Account Balances

Queries balances for a single account across a date range, optionally filtered by balance type and currency.

**`GET /accounts/{branch}/{account}/balances`**

#### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `branch` | string | yes | Branch code |
| `account` | string | yes | Account number |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `initialDate` | date (ISO 8601) | yes | Start of the date range (inclusive) |
| `finalDate` | date (ISO 8601) | yes | End of the date range (inclusive) |
| `balanceTypes` | array\<string\> | no | Filter to specific balance type codes; returns all types if omitted |
| `currency` | string | no | ISO 4217 currency code filter |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `initialDate` is required and must be a valid date in ISO 8601 format (`YYYY-MM-DD`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `finalDate` is required and must be a valid date in ISO 8601 format | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `initialDate` must be ≤ `finalDate` (inverted ranges are rejected) | `about:blank` | `Unprocessable Entity` | `422` |
| `currency`, when provided, must be a valid ISO 4217 code | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `balanceTypes`, when provided, must contain only valid balance type codes configured in DTW | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-account-balances` role | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in the DTW Registry | `about:blank` | `Not Found` | `404` |

#### Response `200 OK`

```json
{
  "balances": [
    {
      "date": "2026-07-23",
      "balances": {
        "DEBIT_BALANCE": 5000.00,
        "CREDIT_BALANCE": 3200.00,
        "AVAILABLE_BALANCE": 1800.00
      }
    }
  ]
}
```

`balances` is an array of daily snapshots. Each element has:
- `date` — the calendar date of the snapshot
- `balances` — a map of `balanceType → amount` for that date

#### Example

```http
GET /accounts/0001/123456/balances
    ?initialDate=2026-07-01
    &finalDate=2026-07-23
    &balanceTypes=DEBIT_BALANCE,AVAILABLE_BALANCE
    &currency=USD
Authorization: Bearer <token>
```

---

### 3.3 Get Account Limits

Returns the configured limits for an account within a date range, optionally filtered by limit type and currency.

**`GET /accounts/{branch}/{account}/limits`**

#### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `branch` | string | yes | Branch code |
| `account` | string | yes | Account number |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `limitType` | string | no | Filter by a specific limit type code |
| `initialDate` | date (ISO 8601) | no | Start of validity date range |
| `finalDate` | date (ISO 8601) | no | End of validity date range |
| `currency` | string | no | ISO 4217 currency code filter |


#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `initialDate` and `finalDate`, when provided, must be valid dates in ISO 8601 format | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| When both are provided, `initialDate` must be ≤ `finalDate` | `about:blank` | `Unprocessable Entity` | `422` |
| `currency`, when provided, must be a valid ISO 4217 code | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limitType`, when provided, must be a valid limit type code configured in DTW | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-account-limits` role | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in the DTW Registry | `about:blank` | `Not Found` | `404` |

#### Response `200 OK`

```json
{
  "limits": [
    {
      "currency": "USD",
      "min": 0.00,
      "max": 10000.00,
      "startAt": "2026-01-01",
      "dueAt": "2026-12-31"
    }
  ]
}
```

Each limit record contains:
- `currency` — the currency this limit applies to
- `min` — minimum amount allowed
- `max` — maximum amount allowed
- `startAt` — first date this limit is in effect
- `dueAt` — last date this limit is in effect

---

### 3.4 Batch Balances (Multiple Accounts)

Returns balances for multiple accounts in a single request. Useful for dashboards or reconciliation jobs that need to query many accounts simultaneously.

**`GET /balances`**

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `accountKeys` | array\<string\> | yes | Accounts to query; format: `{branch}/{account}` (e.g. `0001/123456`) |
| `initialDate` | date (ISO 8601) | yes | Start of date range |
| `finalDate` | date (ISO 8601) | yes | End of date range |
| `balanceTypes` | array\<string\> | no | Balance type codes to include |
| `currency` | string | no | ISO 4217 currency code filter |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `accountKeys` is required; at least one value in `{branch}/{account}` format | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each `accountKey` must follow the `{branch}/{account}` format (no spaces, `/` separator) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `initialDate` is required and must be a valid ISO 8601 date | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `finalDate` is required and must be a valid ISO 8601 date | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `initialDate` must be ≤ `finalDate` | `about:blank` | `Unprocessable Entity` | `422` |
| `currency`, when provided, must be a valid ISO 4217 code | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-balances` role | `about:blank` | `Forbidden` | `403` |

> Accounts not found for the supplied keys are silently omitted from the batch result — they do **not** produce `404`.

#### Response `200 OK`

```json
{
  "accountBalances": [
    {
      "branch": "0001",
      "account": "123456",
      "balances": [
        {
          "date": "2026-07-23",
          "balances": {
            "AVAILABLE_BALANCE": 1800.00
          }
        }
      ]
    }
  ]
}
```

#### Example

```http
GET /balances
    ?accountKeys=0001/123456&accountKeys=0001/789012
    &initialDate=2026-07-01
    &finalDate=2026-07-23
    &balanceTypes=AVAILABLE_BALANCE
Authorization: Bearer <token>
```

---

## 4. API Catalog — Entry API

Base URL: `http://entry-api`

### 4.1 Endpoint Overview

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/v2/entries` | `view-entries` | Account statement — posted + pending entries |
| GET | `/v1/entries/metadata` | `view-entries-metadata` | Entry metadata by entry ID |

---

### 4.2 Get Entries

Returns the account statement for a single account. Supports both posted and pending entries.

**`GET /v2/entries`**

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `branch` | string | yes | Branch code |
| `account` | string | yes | Account number |
| `initialDate` | date (ISO 8601) | yes | Start of date range (inclusive) |
| `finalDate` | date (ISO 8601) | yes | End of date range (inclusive) |
| `type` | enum | no | `POSTED` (default), `PENDING`, or omit to include both |
| `dateType` | enum | no | Which date to filter on: `ENTRY` (accounting date) or `EVENT` (business date). Default: `ENTRY` |
| `currency` | string | no | ISO 4217 currency code |
| `historyCodes` | array\<string\> | no | Filter by GL / history codes |
| `operationType` | enum | no | `C` (credits only) or `D` (debits only) |
| `page` | integer | no | Zero-based page number. Default: `0` |
| `size` | integer | no | Page size. Default: system-configured maximum |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `initialDate` is required and must be a valid ISO 8601 date | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `finalDate` is required and must be a valid ISO 8601 date | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `initialDate` must be ≤ `finalDate` | `about:blank` | `Unprocessable Entity` | `422` |
| `type`, when provided, must be `POSTED` or `PENDING` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `dateType`, when provided, must be `ENTRY` or `EVENT` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `operationType`, when provided, must be `C` or `D` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `page` must be ≥ 0 | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `size`, when provided, must be a positive integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-entries` role | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in the DTW Registry | `about:blank` | `Not Found` | `404` |

#### Response `200 OK`

```json
{
  "entries": [
    {
      "branch": "0001",
      "account": "123456",
      "entryDate": "2026-07-22",
      "eventDate": "2026-07-22",
      "operationType": "D",
      "amount": 500.00,
      "currency": "USD",
      "entryId": "ent-pending-001",
      "entryKind": "PENDING_DEBIT",
      "entryDescription": "Pending scheduled payment",
      "historyCode": "AGD",
      "historyDescription": "Scheduled payment",
      "metadata": {},
      "correlations": {
        "CARD": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab"
      },
      "compensatedEntry": null,
      "holdReasonId": "HOLD-42",
      "holdReason": "Scheduled payment hold",
      "blockingKind": "BALANCE"
    }
  ],
  "balanceSummaryV2": {
    "sumOfPostedEntries": -150.00,
    "sumOfPendingDebitEntries": 500.00,
    "sumOfPendingCreditEntries": 0.00
  }
}
```

#### Filtering by entry status

```http
### Posted entries only (default)
GET /v2/entries?branch=0001&account=123456&initialDate=2026-07-01&finalDate=2026-07-23&type=POSTED

### Pending entries (holds / monitorings in flight)
GET /v2/entries?branch=0001&account=123456&initialDate=2026-07-01&finalDate=2026-07-23&type=PENDING

### All entries
GET /v2/entries?branch=0001&account=123456&initialDate=2026-07-01&finalDate=2026-07-23
```

---

### 4.3 Get Entry Metadata

Retrieves the metadata map attached to one or more entries by their entry IDs. Use this endpoint when you need the full metadata payload for entries returned by the statement endpoint but want to avoid fetching full entry records.

**`GET /v1/entries/metadata`**

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `entriesId` | array\<string\> | yes | List of entry IDs to retrieve metadata for |

#### Validations — Entry Metadata

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `entriesId` is required; at least one entry ID must be supplied | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each entry ID must be a non-empty string | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-entries-metadata` role | `about:blank` | `Forbidden` | `403` |

> Entry IDs not found return an empty `metaData` map (`{}`), not `null` — they do **not** produce `404`.

#### Response `200 OK`

```json
[
  {
    "id": "ent-9f3c1a2b",
    "metaData": {
      "invoiceId": "INV-2026-00142",
      "costCenter": "CC-099"
    }
  },
  {
    "id": "ent-7d2e0b1f",
    "metaData": {
      "orderId": "ORD-99871"
    }
  }
]
```

The response is a flat array; each element contains the entry ID and its metadata map. Entries with no metadata return an empty map, not `null`.

#### Example

```http
GET /v1/entries/metadata?entriesId=ent-9f3c1a2b&entriesId=ent-7d2e0b1f
Authorization: Bearer <token>
```

---

## 5. Pagination

The `/v2/entries` endpoint supports offset-based pagination.

### Request Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `0` | Zero-based page index |
| `size` | integer | system max | Number of records per page |

### Response Headers

Unless `X-Count-Omit: true` is sent in the request, the response includes:

| Header | Description |
|---|---|
| `X-Total-Elements` | Total number of entries matching the filter |
| `X-Total-Pages` | Total number of pages |
| `X-Has-Next` | `true` if there is a page after the current one |
| `X-Count-Limit-Exceeded` | `true` if the total count exceeds the system's configured maximum |

### Example — Paginated Statement

```http
GET /v2/entries
    ?branch=0001
    &account=123456
    &initialDate=2026-07-01
    &finalDate=2026-07-23
    &page=0
    &size=50
Authorization: Bearer <token>

HTTP/1.1 200 OK
X-Total-Elements: 137
X-Total-Pages: 3
X-Has-Next: true
```

---

## 6. Error Reference

Both applications return errors in **RFC 7807 Problem format** (`application/problem+json`).

```json
{
  "type": "https://dtw.internal/problems/account-not-found",
  "title": "Account Not Found",
  "status": 404,
  "detail": "Account 0001/999999 does not exist in the registry."
}
```

### Common HTTP Status Codes

| Status | Meaning | Common Causes |
|---|---|---|
| `400 Bad Request` | Invalid request parameters | Missing required query parameter, malformed date, invalid enum value |
| `401 Unauthorized` | Missing or invalid token | Expired JWT, token issued by wrong realm |
| `403 Forbidden` | Insufficient permissions | Client does not hold the required role for the requested endpoint |
| `404 Not Found` | Resource does not exist | Account not found, entry ID not found |
| `422 Unprocessable Entity` | Semantic validation failure | Date range inverted (`initialDate` > `finalDate`), unsupported balance type |
| `500 Internal Server Error` | Unexpected server error | Contact DTW operations team with the `detail` field |

### Field Reference

| Field | Type | Description |
|---|---|---|
| `type` | URI | Machine-readable problem type identifier |
| `title` | string | Short human-readable summary |
| `status` | integer | HTTP status code (mirrors the response status) |
| `detail` | string | Detailed description of the specific problem instance |

---

## 7. Appendix

### 7.1 Service Relationship Summary

```
┌──────────────────────────────────────────────────────┐
│              DTW Balance & Statement                 │
│                                                      │
│  ┌────────────────────┐  ┌────────────────────────┐  │
│  │    balance-api     │  │      entry-api         │  │
│  │                    │  │                        │  │
│  │ /accounts/.../     │  │ /v2/entries            │  │
│  │   balances         │  │ /v1/entries/metadata   │  │
│  │   limits           │  │                        │  │
│  │ /balances (batch)  │  │                        │  │
│  └────────┬───────────┘  └────────────┬───────────┘  │
└───────────┼──────────────────────────┼──────────────┘
            │                          │
            └──────────┬───────────────┘
                       │
              DTW Ledger Event Stream
```

### 7.2 Entry Lifecycle — How Entries Appear in Statements

```
Monitoring created (Transaction API)
  → appears as PENDING_DEBIT or PENDING_CREDIT in /v2/entries (type=PENDING)

Monitoring released (Composite or Transaction API)
  → LEDGER entry appears in /v2/entries (type=POSTED)
  → UNBLOCKING entry also appears
  → original PENDING entry no longer appears

Entry reversed
  → REVERSAL entry appears alongside original LEDGER entry
  → compensatedEntry field links reversal back to original

Entry removed / excluded
  → REMOVAL or EXCLUSION entry appears
  → original entry remains in history with compensatedEntry populated
```

### 7.3 Date Type Filter (`dateType`)

The entries endpoint accepts a `dateType` parameter that controls which date field is used for the range filter:

| Value | Field filtered | Use case |
|---|---|---|
| `ENTRY` (default) | `entryDate` (accounting date) | Reconciliation against ledger |
| `EVENT` | `eventDate` (business date) | Matching against business system records |

### 7.4 `accountKeys` Format for Batch Balances

The `GET /balances` endpoint accepts multiple `accountKeys` query parameters. Each must be formatted as `{branch}/{account}`:

```
GET /balances?accountKeys=0001/123456&accountKeys=0001/789012&accountKeys=0002/000001
```

This is different from the path-based format used by the single-account endpoints (`/accounts/{branch}/{account}/balances`).

### 7.5 Role Summary

| Role | Application | Scope |
|---|---|---|
| `view-account-balances` | balance-api | Single account balance query |
| `view-account-limits` | balance-api | Single account limit query |
| `view-balances` | balance-api | Batch multi-account balance query |
| `view-entries` | entry-api | Entry statement |
| `view-entries-metadata` | entry-api | Entry metadata lookup |

---

## Advanced Integration Topics

### OpenAPI Specification

The OpenAPI specification for this API is available as a YAML file upon request. It can be used for client code generation or tooling integration.
