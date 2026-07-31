# Digital Twin Integration Guide — Transaction REST API

**Audience:** Developers integrating with the Digital Twin Transaction service to submit ledger entry batches, delete entries, and manage monitorings (pre-authorized holds with deferred release) via HTTP.

---

## Transaction APIs

The Transaction APIs allow applications to submit financial operations to Digital Twin for a single account or an account group. Supported operations include debit and credit postings, holds, releases, reversals, removals, and other ledger operations. The APIs also support creating and managing monitorings (pre-authorized amounts that are held and released later).

Amounts are expressed in minor currency units (for example, USD 50.00 is represented as `5000`). Unless otherwise specified, all operation amounts (including entries, settlements, releases, and monitoring operations) must be greater than or equal to one minor unit.

The request and response field types described below are shared across the Transaction APIs and the Composite Transaction APIs.

This guide covers the synchronous REST APIs only. For asynchronous transaction processing using Kafka commands, see the Digital Twin Integration Guide — Commands.

### When to use this service vs. others

| Need | Service |
|---|---|
| Post ledger entries, holds, or reversals for a single account | **Transaction API** (this guide) |
| Manage monitorings — create, update, cancel, or release — for a single account or account group | **Transaction API** (this guide) |
| Submit entries spanning multiple accounts in a single atomic request | [Composite API](./05-dtw-composite-api-integration-guide.md) |
| Release a monitoring with side effects across multiple accounts | [Composite API](./05-dtw-composite-api-integration-guide.md) |
| Query account balances or statement history | [Balance & Statement API](./04-dtw-balance-statement-api-integration-guide.md) |
| Look up account status, type, features, or limits | [Registry API](./03-dtw-registry-api-integration-guide.md) |
| Submit transactions asynchronously via Kafka | Commands Integration Guide |

---

## Table of Contents

1. [Prerequisites](#prerequisites)

**Shared Building Blocks**

2. [Authentication and Authorization](#authentication-and-authorization)
3. [Common Request and Response Types](#common-request-and-response-types)
4. [Error Handling](#error-handling)
5. [Pagination](#pagination)
6. [Optimistic Locking (ETag / If-Match)](#optimistic-locking-etag--if-match)

**API Catalog — Account Endpoints**

7. [Entry Batch API](#entry-batch-api)
   - [POST /v1/accounts/{branch}/{account}/batches — Submit Entry Batch](#post-v1accountsbranchaccountbatches--submit-entry-batch)
   - [DELETE /v1/accounts/{branch}/{account}/entries — Delete Entries](#delete-v1accountsbranchaccountentries--delete-entries)
8. [Account Monitorings API](#account-monitorings-api)
    - [POST /v1/accounts/{branch}/{account}/monitorings — Create Monitoring](#post-v1accountsbranchaccountmonitorings--create-monitoring)
    - [GET /v1/accounts/{branch}/{account}/monitorings — List Monitorings](#get-v1accountsbranchaccountmonitorings--list-monitorings)
    - [GET /v1/accounts/{branch}/{account}/monitorings/{id} — Get Monitoring](#get-v1accountsbranchaccountmonitoringsid--get-monitoring)
    - [PATCH /v1/accounts/{branch}/{account}/monitorings — Update Monitoring by CorrelationId](#patch-v1accountsbranchaccountmonitorings--update-monitoring-by-correlationid)
    - [PATCH /v1/accounts/{branch}/{account}/monitorings/{id} — Update Monitoring by ID](#patch-v1accountsbranchaccountmonitoringsid--update-monitoring-by-id)
    - [GET /v1/accounts/{branch}/{account}/monitorings/{id}/entries — List Monitoring Entries](#get-v1accountsbranchaccountmonitoringsidentries--list-monitoring-entries)
    - [POST /v1/accounts/{branch}/{account}/monitorings/cancel — Cancel Monitoring](#post-v1accountsbranchaccountmonitoringscancel--cancel-monitoring)
    - [POST /v2/accounts/{branch}/{account}/monitorings/{id}/release — Release Monitoring by ID](#post-v2accountsbranchaccountmonitoringsidrelease--release-monitoring-by-id)
    - [POST /v2/accounts/{branch}/{account}/monitorings/release — Release Monitoring by CorrelationId](#post-v2accountsbranchaccountmonitoringsrelease--release-monitoring-by-correlationid)

**API Catalog — Account Group Endpoints**

9. [Account Group Monitorings API](#account-group-monitorings-api)
    - [POST /v1/account-groups/{kind}/{membershipId}/monitorings — Create Group Monitoring](#post-v1account-groupskindmembershipidmonitorings--create-group-monitoring)
    - [GET /v1/account-groups/{kind}/{membershipId}/monitorings — List Group Monitorings](#get-v1account-groupskindmembershipidmonitorings--list-group-monitorings)
    - [GET /v1/account-groups/{kind}/{membershipId}/monitorings/{id} — Get Group Monitoring](#get-v1account-groupskindmembershipidmonitoringsid--get-group-monitoring)
    - [PATCH /v1/account-groups/{kind}/{membershipId}/monitorings — Update Group Monitoring by CorrelationId](#patch-v1account-groupskindmembershipidmonitorings--update-group-monitoring-by-correlationid)
    - [PATCH /v1/account-groups/{kind}/{membershipId}/monitorings/{id} — Update Group Monitoring by ID](#patch-v1account-groupskindmembershipidmonitoringsid--update-group-monitoring-by-id)
    - [GET /v1/account-groups/{kind}/{membershipId}/monitorings/{id}/entries — List Group Monitoring Entries](#get-v1account-groupskindmembershipidmonitoringsidentries--list-group-monitoring-entries)
    - [POST /v1/account-groups/{kind}/{membershipId}/monitorings/cancel — Cancel Group Monitoring](#post-v1account-groupskindmembershipidmonitoringscancel--cancel-group-monitoring)
    - [POST /v2/account-groups/{kind}/{membershipId}/monitorings/{id}/release — Release Group Monitoring by ID](#post-v2account-groupskindmembershipidmonitoringsidrelease--release-group-monitoring-by-id)
    - [POST /v2/account-groups/{kind}/{membershipId}/monitorings/release — Release Group Monitoring by CorrelationId](#post-v2account-groupskindmembershipidmonitoringsrelease--release-group-monitoring-by-correlationid)

**Operations**

10. [Complete Error Reference](#complete-error-reference)

**Appendix**

11. [Appendix](#appendix)
    - [Full Schema Definitions](#full-schema-definitions)
    - [Advanced Integration Topics](#advanced-integration-topics)

---

## Prerequisites

| Requirement | Details | Why it's needed |
|---|---|---|
| OAuth2 client credentials | A `clientId` and `clientSecret` issued by the DTW Keycloak realm | Every API call requires a Bearer token |
| Required client roles | `transact`, `view-monitorings`, `modify-monitorings`, `release-monitorings` — only the roles needed for your use case | Authorization is enforced per endpoint |
| Target accounts must exist | Populated via the Events flow (AccountChangeEvent) | Entry batches and monitorings reference existing accounts |
| History codes and hold reasons must be configured | Registered via the Events guide's history code and hold reason events | Entry operations reference these codes; unrecognized codes produce `422` errors |
| Maximum batch size | 1–100 entries per batch request | Enforced by the API |
| ETag / If-Match header for PATCH and release | Read the current ETag from a prior GET response | Optimistic locking — missing or mismatched If-Match produces `412 Precondition Failed` |

---

## Authentication and Authorization

### OAuth2 Credentials

All endpoints require a JWT bearer token obtained using the OAuth2 Client Credentials grant from the DTW Keycloak realm. Include the token in the `Authorization` header of every request:

```http
Authorization: Bearer <access_token>
```

### Required Roles

| Endpoint group | Required Role |
|---|---|
| `POST /*/batches`, `DELETE /*/entries` | `transact` |
| `GET /*/monitorings*` | `view-monitorings` |
| `POST /*/monitorings` (create), `PATCH /*/monitorings*`, `POST /*/monitorings/cancel` | `modify-monitorings` |
| `POST /*/monitorings/*/release`, `POST /*/monitorings/release` | `release-monitorings` |

---

## Common Request and Response Types

Many request and response fields use the common data types described below.

### Amount

```json
{ "value": 10000, "currency": "USD" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | `long` (≥ 1) | Yes | Monetary value in the minor currency unit (for example, USD 100.00 is represented as 10000). |
| `currency` | `string` (ISO 4217) | Yes | Currency code. Supports all ISO 4217 fiat codes plus `BTC`, `USC`, `UST`, `SOL`, `XRP` |

### AmountRange

| Field | Type | Required | Description |
|---|---|---|---|
| `min` | `long` (≥ 1) | Yes | Minimum amount |
| `max` | `long` (≥ 1) | Yes | Maximum amount |
| `currency` | `string` (ISO 4217) | Yes | Currency code |

### SettlementInfoRequest

SettlementInfoRequest specifies whether the operation settles a fixed amount or a range of amounts.

| `fulfillment` | Required field |
|---|---|
| `TOTAL` | `amount` |
| `PARTIAL` | `amountRange` |

### DeferredUntil

Used when `status = DEFERRED` on an entry.

| Field | Type | Description |
|---|---|---|
| `triggeredBy` | `MAIN` \| `COMPENSATION` | Whether this entry is triggered by the main entry or a compensating one |
| `initiatorCorrelationID` | `string` | Correlation ID of the triggering entry |
| `executeOn` | `REMOVAL` \| `REVERSAL` \| null | Which compensation event triggers execution |
| `onPhase` | `REGULAR` \| `POSTED_ONLY` | Specifies when the deferred entry is executed during processing. |

### EntryResponse

Returned inside batch responses and monitoring entry lists.

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Correlation ID of the entry |
| `entryKind` | `ledger` \| `release` \| `reversal` \| `removal` \| `blocking` \| `unblocking` \| `exclusion` | Kind of entry |
| `historyCode` | `long` | History code that classifies the entry |
| `holdReasonId` | `string` | Hold reason (blocking/unblocking only) |
| `status` | `POSTED` \| `PENDING` \| `ON_HOLD` \| `DEFERRED` | Entry status |
| `description` | `string` | Statement description (max 40 chars) |
| `asOfDate` | `date` | Effective date |
| `correlations` | `map<string, string>` | Map of originator system to correlation ID |
| `initiator` | `Initiator` | Who triggered the entry |
| `balanceChanges` | `map<string, Amount>` | Balance changes applied by this entry |
| `amount` | `Amount` | Entry amount |
| `metaData` | `object` | Free-form metadata |

### MonitoringResponse

| Field | Type | Description |
|---|---|---|
| `id` | `UUID` | Unique monitoring identifier |
| `correlations` | `map<string, string>` | Map of originator to correlation ID |
| `metaData` | `object` | Free-form metadata |
| `status` | `MonitoringStatus` | See [Monitoring Lifecycle](#monitoring-lifecycle) |
| `entries` | `MonitoringEntryResponse[]` | Ledger entries associated with this monitoring |
| `blueprint` | `EntryResponse` | The blueprint entry that will be created on release |
| `amount` | `Amount` | Reserved amount |
| `amountDue` | `AmountDue` | Amount still to be released |
| `releasedAmount` | `ReleasedAmount` | Amount already released |

### MonitoringEntryResponse

All fields of `EntryResponse` plus:

| Field | Type | Description |
|---|---|---|
| `branch` | `integer` | Account branch |
| `account` | `long` | Account number |
| `createdAt` | `datetime` | When this entry was created |

### ProblemResponse

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `URI` | No | Problem type URI (default: `about:blank`) |
| `title` | `string` | Yes | Short summary |
| `status` | `integer` | Yes | HTTP status code |
| `detail` | `string` | No | Human-readable detail |

### Monitoring Lifecycle

| Status | Meaning |
|---|---|
| `ACTIVE` | Monitoring is active; amount is held |
| `INACTIVE` | Monitoring is temporarily inactive |
| `EXCLUDED` | Monitoring was excluded before release |
| `REALIZED` | Monitoring has been fully released |
| `CANCELLED` | Monitoring was cancelled |

---

## Error Handling

| Status | When it occurs |
|---|---|
| `400 Bad Request` | Invalid or missing request parameters/body fields |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `402 Payment Required` | Insufficient balance to perform the operation |
| `403 Forbidden` | Valid token but missing required role |
| `404 Not Found` | Account, monitoring, or entry not found |
| `409 Conflict` | Release conflict (V2 release endpoints) |
| `410 Gone` | The target resource is no longer available (e.g., account deactivated) |
| `412 Precondition Failed` | `If-Match` header missing, absent, or does not match the current ETag |
| `422 Unprocessable Entity` | Business rule violation (e.g., history code not found, invalid entry state) |
| `503 Service Unavailable` | Downstream service or account temporarily unavailable |
| `500 Internal Server Error` | Unexpected server-side failure |

---

## Pagination

List endpoints support offset-based pagination:

| Parameter | Type | Default | Constraints | Description |
|---|---|---|---|---|
| `page` | `integer` | `0` | ≥ 0 | Zero-based page index |
| `size` | `integer` | `20` | 1–100 | Items per page |

Paginated responses include response headers:

| Header | Type | Description |
|---|---|---|
| `X-Total-Items` | `integer` | Total number of items across all pages |
| `X-Total-Pages` | `integer` | Total number of pages |



---

## Entry Batch API

### POST /v1/accounts/{branch}/{account}/batches — Submit Entry Batch

The batch endpoint is an all-or-nothing operation. Every entry in the request must be individually valid and pass all business rules (balance checks, history code validation, status constraints, etc.). If any single entry in the list is rejected, the entire batch is rolled back — no entries are created, and no balance changes are applied. There is no partial success: either all entries are accepted and persisted, or none are.

This is the **synchronous HTTP alternative to Kafka commands** — use it when you need a synchronous response with the created entries and updated balances, rather than the asynchronous Kafka reply flow.

#### Request

```http
POST /v1/accounts/4919/223166/batches
Authorization: Bearer <token>
Content-Type: application/json
X-Generate-Balance-Snapshot: false

{
  "entries": [
    {
      "entryKind": "ledger",
      "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
      "status": "POSTED",
      "historyCode": 101,
      "settlementInfo": {
        "fulfillment": "TOTAL",
        "amount": { "value": 1485, "currency": "USD" }
      },
      "asOfDate": "2024-06-15",
      "description": "FedNow received"
    },
    {
      "entryKind": "blocking",
      "correlationId": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
      "status": "ON_HOLD",
      "holdReasonId": "PRE_AUTH",
      "settlementInfo": {
        "fulfillment": "TOTAL",
        "amount": { "value": 5000, "currency": "USD" }
      },
      "asOfDate": "2024-06-15",
      "description": "Card pre-authorization"
    }
  ]
}
```

#### Path Parameters

| Parameter | Type | Description |
|---|---|---|
| `branch` | `integer` | Branch number |
| `account` | `long` | Account number |

#### Request Headers

| Header | Required | Description |
|---|---|---|
| `X-Generate-Balance-Snapshot` | No (default `false`) | If `true`, the response includes a full snapshot of all account balances. If `false`, only the deltas from this batch are included |

#### Query Parameters

| Parameter | Type | Description |
|---|---|---|
| `balances` | `string[]` | When `X-Generate-Balance-Snapshot: true`, filter the snapshot to only these balance types |

#### EntryRequest Field Reference

All entry kinds share the following base fields:

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `entryKind` | string enum | Yes | `ledger`, `release`, `reversal`, `removal`, `blocking`, `unblocking` | Determines which entry subtype fields apply |
| `correlationId` | `string` | Yes | 1–128 chars | Unique identifier for this entry |
| `status` | `EntryStatus` | Yes | `POSTED`, `PENDING`, `ON_HOLD`, `DEFERRED` | Entry status |
| `description` | `string` | No | 1–40 chars | Statement description |
| `metaData` | `object` | No | Free key-value | Free-form metadata |
| `deferredUntil` | `DeferredUntil` | Only when `status=DEFERRED` | — | Deferred execution configuration |
| `extraCorrelations` | `Correlation[]` | No | — | Additional correlation references |

#### Validations — Entry Batch

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `entries` is required and must contain between 1 and 100 items | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `entryKind` is required on each entry and must be one of the valid enum values | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `correlationId` is required on each entry; length must be between 1 and 128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `status` is required on each entry | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `description`, when provided, must not exceed 40 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `settlementInfo.amount.value` must be ≥ 1 (minor currency unit, e.g. centavos) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `deferredUntil` must be present when `status = DEFERRED` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `historyCode` must be registered in DTW for `ledger` entries | `http://dtw.matera.com/transaction/entry-type-not-found` | `Entry type not found` | `400` |
| `holdReasonId` must be registered in DTW for `blocking`/`unblocking` entries | `http://dtw.matera.com/transaction/hold-reason-not-found` | `Hold reason not found` | `400` |
| `entryToReverse` must reference an existing, reversible entry (`correlationId`) | `http://dtw.matera.com/transaction/entry-not-found` | `Entry Not Found` | `400` |
| `entryToRemove` must reference an existing, removable entry | `http://dtw.matera.com/transaction/entry-not-found` | `Entry Not Found` | `400` |
| `entryToRelease` must reference an existing blocked entry | `http://dtw.matera.com/transaction/entry-not-found` | `Entry Not Found` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `transact` role on the `dtw-transaction` client | `about:blank` | `Forbidden` | `403` |
| The account (`branch`/`account`) must exist in DTW | `http://dtw.matera.com/transaction/account-not-found` | `Account Not Found` | `404` |
| The account (`branch`/`account`) must be active (not deactivated) | `http://dtw.matera.com/transaction/inactive-account` | `Inactive account` | `400` |
| Available balance must be sufficient for debit operations | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

> **Idempotency**: If **all** `correlationId`s in the batch have already been processed, the response is `204 No Content` — no new entries are created. If only some are duplicates, the operation fails with `422`.

Per-kind additional fields:

| `entryKind` | Extra Required Fields | Extra Optional Fields |
|---|---|---|
| `ledger` | `historyCode`, `settlementInfo` | `asOfDate`, `uponCreationHeldAs` (when `status=ON_HOLD`) |
| `blocking` | `holdReasonId`, `settlementInfo` | `asOfDate` |
| `unblocking` | `holdReasonId`, `settlementInfo` | `asOfDate`, `uponCreationHeldAs`, `releasing` |
| `reversal` | `entryToReverse` (correlationId of target) | `asOfDate` |
| `removal` | `entryToRemove` (correlationId of target) | `asOfDate` |
| `release` | `amount`, `entryToRelease` (correlationId of blocked entry) | — |

#### Response — 201 Created

```json
{
  "createdAt": "2024-06-15T14:30:00Z",
  "entries": [
    {
      "id": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
      "entryKind": "ledger",
      "historyCode": 101,
      "status": "POSTED",
      "description": "FedNow received",
      "asOfDate": "2024-06-15",
      "correlations": { "checking-account-system": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10" },
      "initiator": { "source": "checking-account-system" },
      "balanceChanges": { "AVAILABLE": { "value": 1485, "currency": "USD" } },
      "amount": { "value": 1485, "currency": "USD" }
    }
  ],
  "balances": {
    "AVAILABLE": { "value": 10485, "currency": "USD" }
  }
}
```

| Status | Response | Description |
|---|---|---|
| `201 Created` | `EntryBatchGroupResponse` | All entries created. Body includes entries + balances |
| `204 No Content` | — | Idempotent replay — all `correlationId`s already processed; no new entries created |
| `400 Bad Request` | `ProblemResponse` | Invalid request body |
| `401 Unauthorized` | `ProblemResponse` | — |
| `402 Payment Required` | `ProblemResponse` | Insufficient funds |
| `403 Forbidden` | `ProblemResponse` | Missing `transact` role |
| `410 Gone` | `ProblemResponse` | Account is deactivated |
| `422 Unprocessable Entity` | `ProblemResponse` | Business rule violation (e.g. history code not found) |
| `503 Service Unavailable` | `ProblemResponse` | Account temporarily unavailable |
| `500 Internal Server Error` | `ProblemResponse` | — |

---

### DELETE /v1/accounts/{branch}/{account}/entries — Delete Entries

Deletes up to 100 entries identified by their `correlationId`. All specified entries must belong to the same account.

#### Request

```http
DELETE /v1/accounts/4919/223166/entries?correlationId=3f9c6a10-...&correlationId=a1b2c3d4-...
Authorization: Bearer <token>
```

#### Path and Query Parameters

| Parameter | In | Type | Required | Constraints | Description |
|---|---|---|---|---|---|
| `branch` | path | `integer` | Yes | — | Branch number |
| `account` | path | `long` | Yes | — | Account number |
| `correlationId` | query | `string[]` | Yes | 1–100 items | Correlation IDs of entries to delete |

#### Validations — Delete Entries

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `correlationId` is required; must contain between 1 and 100 values | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| All referenced entries must belong to the same account (`branch`/`account`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `transact` role on the `dtw-transaction` client | `about:blank` | `Forbidden` | `403` |
| Account balance must remain valid after the entries are removed | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

#### Response — 204 No Content

| Status | Response | Description |
|---|---|---|
| `204 No Content` | — | Entries deleted |
| `400 Bad Request` | `ProblemResponse` | Invalid parameters |
| `401 Unauthorized` | `ProblemResponse` | — |
| `402 Payment Required` | `ProblemResponse` | Insufficient balance after deletion |
| `403 Forbidden` | `ProblemResponse` | Missing `transact` role |
| `503 Service Unavailable` | `ProblemResponse` | — |
| `500 Internal Server Error` | `ProblemResponse` | — |

---

## Account Monitorings API

Monitorings are pre-authorized holds scoped to a single `branch/account`. For group-scoped monitorings (by membership), see [Account Group Monitorings API](#account-group-monitorings-api). The request/response schemas are identical; only the resource path differs.

### POST /v1/accounts/{branch}/{account}/monitorings — Create Monitoring

Creates a monitoring that reserves an amount for a future ledger entry.

#### Request

```http
POST /v1/accounts/4919/223166/monitorings
Authorization: Bearer <token>
Content-Type: application/json

{
  "monitoringKind": "ledger",
  "correlationId": "mon-corr-001",
  "amount": { "value": 50000, "currency": "USD" },
  "status": "ACTIVE",
  "blueprint": {
    "entryKind": "ledger",
    "historyCode": 200,
    "description": "Installment debit"
  },
  "metaData": { "contractId": "C-9999" }
}
```

#### Field Reference — MonitoringRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `monitoringKind` | `Kind` enum | Yes | Entry kind the monitoring will create on release: `ledger`, `blocking`, `unblocking`, `reversal`, `removal`, `release` |
| `correlationId` | `string` (1–128) | Yes | Unique identifier for this monitoring |
| `amount` | `Amount` | Yes | Amount to reserve |
| `status` | `MonitoringStatus` | Yes | Initial status (typically `ACTIVE`) |
| `blueprint` | `MonitoringCreateEntryRequest` | Yes | Template for the entry to create on release |
| `metaData` | `object` | No | Free-form metadata |

#### Validations — Create Monitoring

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `correlationId` is required; length must be between 1 and 128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `amount` is required; `amount.value` must be ≥ 1 | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `amount.currency` must be a valid ISO 4217 code or supported crypto ticker | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `monitoringKind` is required and must be a valid enum value | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `status` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `blueprint` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-monitorings` role on the `dtw-transaction` client | `about:blank` | `Forbidden` | `403` |
| The account (`branch`/`account`) must exist in DTW | `http://dtw.matera.com/transaction/account-not-found` | `Account Not Found` | `404` |
| The account (`branch`/`account`) must be active (not deactivated) | `http://dtw.matera.com/transaction/inactive-account` | `Inactive account` | `400` |
| Available balance must be sufficient to reserve the amount | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

> **Idempotency**: Re-submitting the same `correlationId` with identical data returns `204 No Content`.

#### Response — 201 Created

Returns `MonitoringResponse`. Includes a `Location` header and `ETag` for optimistic locking.

| Status | Response | Description |
|---|---|---|
| `201 Created` | `MonitoringResponse` | Monitoring created |
| `204 No Content` | — | Idempotent replay — `correlationId` already exists |
| `400 Bad Request` | `ProblemResponse` | — |
| `401 Unauthorized` | `ProblemResponse` | — |
| `402 Payment Required` | `ProblemResponse` | Insufficient balance |
| `403 Forbidden` | `ProblemResponse` | Missing `modify-monitorings` role |
| `410 Gone` | `ProblemResponse` | Account deactivated |
| `503 Service Unavailable` | `ProblemResponse` | — |
| `500 Internal Server Error` | `ProblemResponse` | — |

---

### GET /v1/accounts/{branch}/{account}/monitorings — List Monitorings

Returns a paginated list of monitorings for an account, optionally filtered by correlation IDs.

#### Request

```http
GET /v1/accounts/4919/223166/monitorings?page=0&size=20&correlationIds=mon-corr-001
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | — |
| `account` | path | `long` | Yes | — |
| `correlationIds` | query | `string[]` | No | 1–100 items. Filter by specific correlation IDs |
| `page` | query | `integer` | No | Default `0` |
| `size` | query | `integer` | No | Default `20`, max `100` |

#### Validations — List Monitorings

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` must be parseable as a non-negative integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` must be parseable as a non-negative long | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `correlationIds`, when provided, must contain between 1 and 100 items | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `page` must be ≥ 0; `size` must be between 1 and 100 | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-monitorings` role | `about:blank` | `Forbidden` | `403` |

#### Response — 200 OK

Returns `MonitoringsAccountResponse` with `monitorings[]` plus `X-Total-Items` / `X-Total-Pages` headers.

---

### GET /v1/accounts/{branch}/{account}/monitorings/{id} — Get Monitoring

Retrieves a single monitoring by UUID. The response includes an `ETag` header for use in subsequent PATCH and release calls.

#### Request

```http
GET /v1/accounts/4919/223166/monitorings/550e8400-e29b-41d4-a716-446655441000
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | — |
| `account` | path | `long` | Yes | — |
| `id` | path | `UUID` | Yes | Monitoring UUID |

#### Validations — Get Monitoring

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `id` must be a valid UUID | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-monitorings` role | `about:blank` | `Forbidden` | `403` |
| The monitoring with the supplied `id` must exist for this account | `http://dtw.matera.com/transaction/intent-not-found` | `Intent Not Found` | `404` |

#### Response — 200 OK

Returns `MonitoringResponse` with an `ETag` header.

| Status | Response | Description |
|---|---|---|
| `200 OK` | `MonitoringResponse` | Monitoring found |
| `401 Unauthorized` | `ProblemResponse` | — |
| `403 Forbidden` | `ProblemResponse` | Missing `view-monitorings` role |
| `404 Not Found` | `ProblemResponse` | Monitoring not found |
| `500 Internal Server Error` | `ProblemResponse` | — |

---

### PATCH /v1/accounts/{branch}/{account}/monitorings — Update Monitoring by CorrelationId

Updates a monitoring identified by correlation ID. Requires the `If-Match` header. Uses `application/merge-patch+json` semantics — omitted fields are left unchanged.

#### Request

```http
PATCH /v1/accounts/4919/223166/monitorings?correlationId=mon-corr-001
Authorization: Bearer <token>
Content-Type: application/merge-patch+json
If-Match: "42"

{
  "status": "INACTIVE",
  "amount": { "value": 30000, "currency": "USD" }
}
```

| Parameter | In | Required | Description |
|---|---|---|---|
| `correlationId` | query | Yes (exactly 1) | Correlation ID of the monitoring to update |
| `If-Match` | header | Yes | ETag from a prior GET |

#### Validations — Update Monitoring (PATCH)

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `correlationId` is required; exactly 1 value (cannot be empty or multiple) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `Content-Type` must be `application/merge-patch+json` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `If-Match` header is required | `http://dtw.matera.com/transaction/monitoring-version-mismatch` | `Monitoring service version mismatch` | `412` |
| The `If-Match` value must match the current `ETag` of the monitoring | `http://dtw.matera.com/transaction/monitoring-version-mismatch` | `Monitoring service version mismatch` | `412` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-monitorings` role | `about:blank` | `Forbidden` | `403` |
| The referenced monitoring must be active and not in a terminal state | `http://dtw.matera.com/transaction/illegal-monitoring-status` | `Illegal monitoring service status` | `412` |
| Available balance must be sufficient for the updated reserved amount | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

#### Field Reference — MonitoringPatchRequest

| Field | Type | Description |
|---|---|---|
| `status` | `MonitoringStatus` | New status |
| `amount` | `Amount` | Updated reserved amount |

#### Response

| Status | Response | Description |
|---|---|---|
| `200 OK` | `MonitoringResponse` | Monitoring updated |
| `204 No Content` | — | No change needed |
| `400 Bad Request` | `ProblemResponse` | — |
| `401 Unauthorized` | `ProblemResponse` | — |
| `402 Payment Required` | `ProblemResponse` | — |
| `403 Forbidden` | `ProblemResponse` | Missing `modify-monitorings` role |
| `410 Gone` | `ProblemResponse` | — |
| `412 Precondition Failed` | `ProblemResponse` | `If-Match` mismatch |
| `503 Service Unavailable` | `ProblemResponse` | — |

---

### PATCH /v1/accounts/{branch}/{account}/monitorings/{id} — Update Monitoring by ID

Same as the correlation-based PATCH but identified by UUID path parameter.

```http
PATCH /v1/accounts/4919/223166/monitorings/550e8400-e29b-41d4-a716-446655441000
Authorization: Bearer <token>
Content-Type: application/merge-patch+json
If-Match: "42"

{ "status": "INACTIVE" }
```

---

### GET /v1/accounts/{branch}/{account}/monitorings/{id}/entries — List Monitoring Entries

Returns the ledger entries associated with a monitoring, paginated.

#### Request

```http
GET /v1/accounts/4919/223166/monitorings/550e8400-e29b-41d4-a716-446655441000/entries?page=0&size=20
Authorization: Bearer <token>
```

Returns `MonitoringAccountEntriesResponse` with `monitoringEntries[]` and pagination headers.

---

### POST /v1/accounts/{branch}/{account}/monitorings/cancel — Cancel Monitoring

Cancels a monitoring identified by either UUID or correlation ID (exactly one must be supplied).

#### Request

```http
POST /v1/accounts/4919/223166/monitorings/cancel?id=550e8400-e29b-41d4-a716-446655441000
Authorization: Bearer <token>
```

| Parameter | In | Required | Constraints | Description |
|---|---|---|---|---|
| `id` | query | One of `id` or `correlationId` | max 1 | Monitoring UUID |
| `correlationId` | query | One of `id` or `correlationId` | max 1 | Monitoring correlation ID |

#### Validations — Cancel Monitoring

| Rule | `type` | `title` | Status |
|---|---|---|---|
| Exactly one of `id` or `correlationId` must be supplied (not both, not neither) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `id`, when provided, must be a valid UUID | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-monitorings` role | `about:blank` | `Forbidden` | `403` |
| The monitoring must exist for this account | `http://dtw.matera.com/transaction/intent-not-found` | `Intent Not Found` | `404` |
| Only monitorings in `ACTIVE` or `INACTIVE` state can be cancelled | `http://dtw.matera.com/transaction/monitoring-cancellation-forbidden` | `Cancellation of Monitoring Intent not allowed` | `422` |

Returns `MonitoringResponse` with `status = CANCELLED`.

---

### POST /v2/accounts/{branch}/{account}/monitorings/{id}/release — Release Monitoring by ID

Releases a monitoring by UUID. Creates ledger entries according to the specified blueprints and marks the monitoring as `REALIZED`.

#### Request

```http
POST /v2/accounts/4919/223166/monitorings/550e8400-e29b-41d4-a716-446655441000/release
Authorization: Bearer <token>
Content-Type: application/json

{
  "correlationId": "release-corr-001",
  "settlementInfo": {
    "fulfillment": "TOTAL",
    "amount": { "value": 50000, "currency": "USD" }
  },
  "blueprints": [],
  "metaData": { "releaseReason": "contract-fulfilled" }
}
```

#### Field Reference — MonitoringReleaseRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `correlationId` | `string` | Yes | Unique ID for this release operation |
| `settlementInfo` | `SettlementInfoRequest` | Yes | Amount being released (`TOTAL` or `PARTIAL`) |
| `blueprints` | `MonitoringEntryRequest[]` | No (0–100) | Additional entries to create alongside the release |
| `metaData` | `object` | No | — |

#### Validations — Release Monitoring (V2)

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `correlationId` is required in the request body | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `settlementInfo` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `fulfillment = "TOTAL"`: `amount` field is required with `value ≥ 1` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `fulfillment = "PARTIAL"`: `amountRange` is required with `min` and `max` (both ≥ 1) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `blueprints`, when provided, must contain between 0 and 100 items | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `release-monitorings` role | `about:blank` | `Forbidden` | `403` |
| The monitoring must exist for this account | `http://dtw.matera.com/transaction/intent-not-found` | `Intent Not Found` | `404` |
| The monitoring must be in a state that allows release | `http://dtw.matera.com/transaction/illegal-monitoring-status` | `Illegal monitoring service status` | `412` |
| Release conflict (duplicate or concurrent release correlationId) | `http://dtw.matera.com/transaction/correlation-conflict` | `Correlation conflict` | `409` |
| Balance must be sufficient to cover the release amount | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

> **Idempotency**: Re-submitting the same release `correlationId` returns `204 No Content`.

Returns `MonitoringReleaseResponse`.

| Status | Response | Description |
|---|---|---|
| `200 OK` | `MonitoringReleaseResponse` | Released |
| `204 No Content` | — | Idempotent replay |
| `400 Bad Request` | `ProblemResponse` | — |
| `401 Unauthorized` | `ProblemResponse` | — |
| `402 Payment Required` | `ProblemResponse` | — |
| `403 Forbidden` | `ProblemResponse` | Missing `release-monitorings` role |
| `409 Conflict` | `ProblemResponse` | Release conflict |
| `410 Gone` | `ProblemResponse` | — |
| `412 Precondition Failed` | `ProblemResponse` | — |
| `503 Service Unavailable` | `ProblemResponse` | — |

---

### POST /v2/accounts/{branch}/{account}/monitorings/release — Release Monitoring by CorrelationId

Same as the ID-based release, but identifies the monitoring via the `correlationId` query parameter.

```http
POST /v2/accounts/4919/223166/monitorings/release?correlationId=mon-corr-001
Authorization: Bearer <token>
Content-Type: application/json
```

| Parameter | In | Required | Constraints | Description |
|---|---|---|---|---|
| `correlationId` | query | Yes | exactly 1 | Correlation ID of the monitoring to release |

#### Validations — Release by CorrelationId

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `correlationId` query parameter is required; exactly 1 value | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| All body validations are identical to the ID-based release (see above) | — | — | — |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `release-monitorings` role | `about:blank` | `Forbidden` | `403` |
| The monitoring referenced by `correlationId` must exist | `http://dtw.matera.com/transaction/intent-not-found` | `Intent Not Found` | `404` |

Body and response are identical to the ID-based release.

---

## Account Group Monitorings API

Account group monitorings function identically to account monitorings, but are scoped to a membership group rather than a single `branch/account`. The resource path prefix changes from `/v1/accounts/{branch}/{account}` to `/v1/account-groups/{kind}/{membershipId}`.

### Path Parameters

| Parameter | Type | Description |
|---|---|---|
| `kind` | `string` | Membership group kind (e.g. `HOLDER`, `CORPORATE_GROUP_HOLDER_TAX_ID`) |
| `membershipId` | `string` | Membership identifier (e.g. a customerId) |

### Available Endpoints

All endpoints mirror the account monitoring endpoints in behavior, parameters, and response schemas:

| Method | Path | Role Required | Account Equivalent |
|---|---|---|---|
| `POST` | `/v1/account-groups/{kind}/{membershipId}/monitorings` | `modify-monitorings` | Create monitoring |
| `GET` | `/v1/account-groups/{kind}/{membershipId}/monitorings` | `view-monitorings` | List monitorings |
| `GET` | `/v1/account-groups/{kind}/{membershipId}/monitorings/{id}` | `view-monitorings` | Get monitoring |
| `PATCH` | `/v1/account-groups/{kind}/{membershipId}/monitorings` | `modify-monitorings` | Update by correlationId |
| `PATCH` | `/v1/account-groups/{kind}/{membershipId}/monitorings/{id}` | `modify-monitorings` | Update by ID |
| `GET` | `/v1/account-groups/{kind}/{membershipId}/monitorings/{id}/entries` | `view-monitorings` | List entries |
| `POST` | `/v1/account-groups/{kind}/{membershipId}/monitorings/cancel` | `modify-monitorings` | Cancel |
| `POST` | `/v2/account-groups/{kind}/{membershipId}/monitorings/{id}/release` | `release-monitorings` | Release by ID |
| `POST` | `/v2/account-groups/{kind}/{membershipId}/monitorings/release` | `release-monitorings` | Release by correlationId |

---

## Complete Error Reference

| HTTP Status | When it occurs |
|---|---|
| `400 Bad Request` | Malformed or missing request fields; constraint violation on parameters |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `402 Payment Required` | Insufficient account balance for the requested operation |
| `403 Forbidden` | Token valid but lacks the required role |
| `404 Not Found` | Account, monitoring, or entry not found |
| `409 Conflict` | Release conflict (V2 release endpoints) |
| `410 Gone` | Account is permanently deactivated |
| `412 Precondition Failed` | `If-Match` header absent or stale; concurrent modification detected |
| `422 Unprocessable Entity` | Business rule violation (invalid entry state, unknown history code, etc.) |
| `503 Service Unavailable` | Account temporarily locked or downstream unavailable |
| `500 Internal Server Error` | Unexpected server-side failure |

---

## Appendix

---

## Full Schema Definitions

### EntryStatus enum
`POSTED` | `PENDING` | `ON_HOLD` | `DEFERRED`

### MonitoringStatus enum
`ACTIVE` | `INACTIVE` | `EXCLUDED` | `REALIZED` | `CANCELLED`

### Entry kind enums

**Request (`entryKind`):** `ledger` | `release` | `reversal` | `removal` | `blocking` | `unblocking`

**Response (`entryKind`):** above + `exclusion`

### EntryBatchGroupResponse

```json
{
  "createdAt": "2024-06-15T14:30:00Z",
  "entries": [ "...EntryResponse objects..." ],
  "balances": {
    "AVAILABLE": { "value": 10485, "currency": "USD" }
  }
}
```

### MonitoringReleaseResponse

| Field | Type | Description |
|---|---|---|
| `id` | `UUID` | Monitoring UUID |
| `correlations` | `map<string, string>` | — |
| `metaData` | `object` | — |
| `status` | `MonitoringStatus` | — |
| `entries` | `MonitoringEntryResponse[]` | Entries created by the release |
| `releasedAmount` | `ReleasedAmount` | Total released so far |
| `releaseCorrelations` | `map<string, string>` | Correlation IDs for the release operation |

---

## Advanced Integration Topics

### HTTP vs. Kafka Command API

The entry batch HTTP API and the Kafka command API are two integration paths for the same underlying ledger operation. Choose based on your needs:

| Concern | HTTP Batch API | Kafka Command API |
|---|---|---|
| Response latency | Synchronous — immediate | Asynchronous — result on `replyTo` topic |
| Batch size | 1–100 entries per request | 1 entry per command (combo: multiple) |
| Balance snapshot | Available in the response | Not in the command result |
| Retry behavior | Standard HTTP retry | No DLQ — producer must track `replyTo` |

### Monitoring vs. Entry Direct

Use a monitoring when the final amount is unknown at the time of authorization, or when the release should be deferred to a future event. Use a direct ledger entry (via batch or Kafka command) when the amount and timing are known at the time of the request.

### Deprecated V1 Release Endpoints

The `/v1/.../monitorings/{id}/release` endpoints (`releaseAccountMonitoringV1`, `releaseAccountGroupMonitoringV1`) are deprecated. They accept `MonitoringReleaseRequestV1`, which differs from the V2 request in that it does not carry a `correlationId`. Use the `/v2/` equivalents for all new integrations.

### OpenAPI Specification

The OpenAPI specification for this API is available as a YAML file upon request. It can be used for client code generation or tooling integration.
