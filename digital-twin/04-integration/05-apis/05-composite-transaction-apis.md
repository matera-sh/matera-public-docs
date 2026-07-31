# Digital Twin Integration Guide — Composite Transaction REST API

**Audience:** Developers integrating with the Digital Twin Composite Transaction service to submit multi-account ledger bundles and release monitorings that span multiple accounts via a single atomic HTTP call.

---

The Composite Transactions service exposes REST APIs for performing operations that span multiple accounts in a single request. It provides two capabilities:

- **Transaction bundles** — create or delete bundles containing operations for multiple accounts.
- **Monitoring releases** — release a monitoring for an account or account group in a single request.

Monetary amounts are represented as unscaled integers in the currency's minor unit (for example, cents for USD).

The service orchestrates requests across the underlying Transaction service, enabling operations that involve multiple accounts. For operations on a single account, use the Transaction REST API instead.

### When to use this service vs. others

| Need | Service |
|---|---|
| Submit entries spanning multiple accounts in a single atomic request | **Composite API** (this guide) |
| Release a monitoring with entry side effects across multiple accounts | **Composite API** (this guide) |
| Post entries, holds, or reversals targeting a single account | [Transaction REST API](./02-dtw-transaction-api-integration-guide.md) |
| Manage monitorings — create, update, cancel, or release — for a single account or account group | [Transaction REST API](./02-dtw-transaction-api-integration-guide.md) |
| Query account balances or statement history | [Balance & Statement API](./04-dtw-balance-statement-api-integration-guide.md) |
| Look up account status, type, features, or limits | [Registry API](./03-dtw-registry-api-integration-guide.md) |

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)

**Shared Building Blocks**

3. [Authentication and Authorization](#authentication-and-authorization)
4. [Common Request and Response Types](#common-request-and-response-types)
5. [Error Handling](#error-handling)
6. [Optimistic Locking (ETag / If-Match)](#optimistic-locking-etag--if-match)

**API Catalog**

7. [Transaction Bundles API](#transaction-bundles-api)
   - [POST /v1/transaction-bundles — Submit Bundle](#post-v1transaction-bundles--submit-bundle)
   - [DELETE /v1/transaction-bundles/{bundle} — Delete Bundle](#delete-v1transaction-bundlesbundle--delete-bundle)
8. [Monitoring Release API](#monitoring-release-api)
   - [POST /v1/accounts/{branch}/{account}/monitorings/{monitoringId}/release — Release Account Monitoring](#post-v1accountsbranchaccountmonitoringsmonitoringidrelease--release-account-monitoring)
   - [POST /v1/account-groups/{kind}/{membershipId}/monitorings/{monitoringId}/release — Release Group Monitoring](#post-v1account-groupskindmembershipidmonitoringsmonitoringidrelease--release-group-monitoring)

**Operations**

9. [Complete Error Reference](#complete-error-reference)

**Appendix**

10. [Appendix](#appendix)
    - [Full Schema Definitions](#full-schema-definitions)
    - [Advanced Integration Topics](#advanced-integration-topics)

---

## Prerequisites

| Requirement | Details | Why it's needed |
|---|---|---|
| OAuth2 client credentials | `clientId` + `clientSecret` for the DTW Keycloak realm `DTW` | Every API call requires a Bearer token |
| Both composite and transaction authorities | Each endpoint requires two `hasAuthority` checks — see [Authentication and Authorization](#authentication-and-authorization) | The composite service validates both the composite-level and the downstream transaction-level roles |
| Target accounts must exist in DTW | Populated via the Events flow | Bundle entries and monitoring releases reference existing accounts |
| History codes must be configured | Registered via the Events guide | `ledgerEntryRequest` fields reference history codes; unrecognized codes produce errors |
| ETag / If-Match for releases | Read ETag from a prior monitoring GET via `dtw-transaction` | Required by all release endpoints |
| Maximum bundle size | 1–100 entries per `regularBundleRequest` | Enforced by the API |

---

## Authentication and Authorization

### OAuth2 — Client Credentials Flow

All endpoints require a JWT Bearer token from the Keycloak realm `DTW`:

```http
Authorization: Bearer <access_token>
```

### Required Roles

Authorization is enforced via **two simultaneous authority checks** — one scoped to the composite service client (`dtw-composite-transaction`) and one to the downstream transaction service client (`dtw-transaction`).

| Endpoint | Required Authorities |
|---|---|
| `POST /v1/transaction-bundles` | `dtw-composite-transaction::transact` AND `dtw-transaction::transact` |
| `DELETE /v1/transaction-bundles/{bundle}` | `dtw-composite-transaction::transact` AND `dtw-transaction::transact` |
| `POST /v1/accounts/.../monitorings/{id}/release` | `dtw-composite-transaction::release-monitorings` AND `dtw-transaction::view-monitorings` AND `dtw-transaction::modify-monitorings` AND `dtw-transaction::release-monitorings` |
| `POST /v1/account-groups/.../monitorings/{id}/release` | `dtw-composite-transaction::release-monitorings` AND `dtw-transaction::view-monitorings` AND `dtw-transaction::modify-monitorings` AND `dtw-transaction::release-monitorings` |

All authorities must be present in the JWT. A missing authority on either client returns `403 Forbidden`.

---

## Common Request and Response Types

### amount

```json
{ "value": 10000, "currency": "USD" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | `long` (≥ 1) | Yes | Monetary value in the minor unit (e.g. cents for USD) |
| `currency` | `string` (ISO 4217) | Yes | Currency code. Supports fiat codes and `BTC`, `USC`, `UST`, `SOL`, `XRP` |

> **Difference from the Transaction API:** The fields in this API are very similar to those in the [Transaction REST API](./02-dtw-transaction-api-integration-guide.md), with one important difference in how the monetary value is declared on entries. In the Transaction API, the amount is wrapped inside a `settlementInfo` object, which also carries the `fulfillment` discriminator (`TOTAL` or `PARTIAL`):
> ```json
> "settlementInfo": { "fulfillment": "TOTAL", "amount": { "value": 5000, "currency": "USD" } }
> ```
> In the Composite API, the `amount` object is declared **directly** on the entry — there is no `settlementInfo` wrapper:
> ```json
> "amount": { "value": 5000, "currency": "USD" }
> ```
> The `settlementInfo` object still exists in the Composite API, but is used exclusively in monitoring release requests (`monitoringReleaseRequest` and `monitoringReleaseEntryRequest`).

### cautionaryBlockInformation

| Field | Type | Required | Description |
|---|---|---|---|
| `holdReasonId` | `string` | Yes | Hold reason code (must be registered in DTW) |
| `correlationId` | `string` | Yes | Correlation ID of the entry being held |

### cautionaryUnblockInformation

| Field | Type | Required | Description |
|---|---|---|---|
| `correlationId` | `string` | Yes | Must match the `correlationId` supplied in the target entry's `uponCreationHeldAs` at creation time |

### settlementInfoRequest

Discriminated union on `fulfillment`:

| Variant | `fulfillment` | Extra fields |
|---|---|---|
| `amountSettlementInfoRequest` | `"TOTAL"` | `amount` (required) |
| `amountRangeSettlementInfoRequest` | `"PARTIAL"` | `amountRange` (min + max + currency, all required) |

### entryResponse

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Correlation ID of the entry |
| `entryKind` | enum | `ledger`, `release`, `reversal`, `removal`, `blocking`, `unblocking`, `exclusion` |
| `historyCode` | `long` | History code |
| `holdReasonId` | `string` | Hold reason (blocking/unblocking only) |
| `status` | enum | `POSTED`, `PENDING`, `ON_HOLD`, `DEFERRED` |
| `description` | `string` | Statement description (max 40 chars) |
| `asOfDate` | `date` | Effective date |
| `correlations` | `map<string, string>` | Originator → correlation ID |
| `initiator` | `object` | The entry/transaction that initiated the deferred transaction |
| `initiator.id` | `string` | Unique identifier of the initiator (1–128 chars) |
| `initiator.correlations` | `map<string, string>` | Originator → correlation ID map. Example: `{"dummy-originator": "839bb00f-ed02-4d8c-b5c8-77c762ea3832"}` |
| `balanceChanges` | `map<string, amount>` | Balance type → amount change |
| `amount` | `amount` | Entry amount |
| `metaData` | `object` | Free-form metadata |

### ProblemResponse

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `URI` | No | Problem type (default: `about:blank`) |
| `title` | `string` | Yes | Short summary |
| `status` | `integer` | Yes | HTTP status code |
| `detail` | `string` | No | Detail message |

---

## Error Handling

| Status | When it occurs |
|---|---|
| `400 Bad Request` | Invalid or missing request fields; constraint violations |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `402 Payment Required` | Insufficient balance on one or more accounts in the bundle |
| `403 Forbidden` | Token valid but one or more required authorities missing |
| `409 Conflict` | Idempotency conflict (bundle already submitted with different content) or bundle mismatch |
| `410 Gone` | An entry targeted by the operation has been removed |
| `500 Internal Server Error` | Unexpected server-side failure |

---

## Optimistic Locking (ETag / If-Match)

All monitoring release endpoints require the `If-Match` header. The ETag is obtained from a `GET` on the target monitoring via `dtw-transaction`:

1. Call `GET /v1/accounts/{branch}/{account}/monitorings/{id}` on `dtw-transaction`
2. Read the `ETag` header from the response (e.g. `"42"`)
3. Include it verbatim in the `If-Match` header on the release request to `dtw-composite-transaction`

A missing or stale `If-Match` returns `412 Precondition Failed` from `dtw-transaction`.

---

## Transaction Bundles API

### POST /v1/transaction-bundles — Submit Bundle

Submits a bundle of 1–100 ledger operations spread across **multiple accounts** as a single atomic request. All entries in the bundle succeed or fail together.

Three bundle kinds are supported, distinguished by the `kind` field:

| `kind` | Description |
|---|---|
| `regular` | Creates new entries across one or more accounts |
| `removal` | Removes (rolls back) a previously submitted bundle by its `bundle` ID |
| `exclusion` | Excludes pending entries from a previously submitted bundle |

#### Regular Bundle Request

```http
POST /v1/transaction-bundles
Authorization: Bearer <token>
Content-Type: application/json

{
  "bundle": "bundle-corr-001",
  "kind": "regular",
  "status": "POSTED",
  "entries": [
    {
      "branch": 4919,
      "account": 223166,
      "entryKind": "ledger",
      "correlationId": "entry-corr-001",
      "historyCode": 101,
      "amount": { "value": 5000, "currency": "USD" },
      "asOfDate": "2024-06-15",
      "description": "Debit transfer out"
    },
    {
      "branch": 4919,
      "account": 999888,
      "entryKind": "ledger",
      "correlationId": "entry-corr-002",
      "historyCode": 102,
      "amount": { "value": 5000, "currency": "USD" },
      "asOfDate": "2024-06-15",
      "description": "Credit transfer in"
    }
  ]
}
```

#### Field Reference — bundleRequest (base fields)

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `bundle` | `string` | Yes | 1–128 chars | Unique identifier (correlationId) for this bundle |
| `kind` | `string` enum | Yes | `regular`, `removal`, `exclusion` | Bundle operation type |
| `status` | `string` enum | No (default `POSTED`) | `POSTED`, `PENDING` | Entry status to apply to all entries in the bundle |

#### Field Reference — regularBundleRequest additional field

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `entries` | `entryRequest[]` | Yes | 1–100 items | The entry operations to submit. Each entry specifies its own `branch` and `account` |

#### Field Reference — removalBundleRequest additional field

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `bundleToRemove` | `string` | Yes | 1–128 chars | The `bundle` ID (correlationId) of the bundle to roll back |

#### Field Reference — entryRequest (base fields, all kinds)

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `branch` | `integer` | Yes | ≥ 0 | Account branch |
| `account` | `long` | Yes | ≥ 0 | Account number |
| `entryKind` | `string` enum | Yes | `ledger`, `release`, `reversal`, `removal`, `blocking`, `unblocking` | Entry type |
| `correlationId` | `string` | Yes | 1–128 chars | Unique entry identifier |
| `description` | `string` | No | 1–40 chars | Statement description |
| `metaData` | `object` | No | — | Free-form metadata |
| `dependsOn` | `string` | No | 1–128 chars | Correlation ID of an entry this one depends on |
| `asOfDate` | `date` | No | — | Accounting date |

Per-kind additional fields:

**`ledger`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `historyCode` | `long` | Yes | ≥ 0 | History code that categorizes the transaction as credit or debit |
| `amount` | `amount` | Yes | `value` ≥ 1; valid ISO 4217 `currency` | Monetary value in the currency's minor unit |
| `uponCreationHeldAs` | `cautionaryBlockInformation` | No | — | Hold configuration when creating the entry in `ON_HOLD` status |

**`release`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `amount` | `amount` | Yes | `value` ≥ 1; valid ISO 4217 `currency` | Monetary value to release |
| `entryToRelease` | `string` | Yes | 1–128 chars | Correlation ID of the entry to release |

**`reversal`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `entryToReverse` | `string` | Yes | 1–128 chars | Correlation ID of the entry to reverse |
| `holdReasonId` | `string` | No | — | Hold reason ID, when reversing a blocking entry |

**`removal`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `entryToRemove` | `string` | Yes | 1–128 chars | Correlation ID of the entry to remove |

**`blocking`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `amount` | `amount` | Yes | `value` ≥ 1; valid ISO 4217 `currency` | Monetary value to block |
| `holdReasonId` | `string` | Yes | — | Reason code for the hold (must be registered in DTW) |

**`unblocking`**

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `amount` | `amount` | Yes | `value` ≥ 1; valid ISO 4217 `currency` | Monetary value to unblock |
| `holdReasonId` | `string` | No | — | Hold reason ID of the blocking entry being released |
| `uponCreationHeldAs` | `cautionaryBlockInformation` | No | — | Cautionary hold information |
| `releasing` | `cautionaryUnblockInformation` | No | — | Correlation ID of the cautionary-blocked entry to unblock |

#### Response — 201 Created

```json
{
  "bundleId": "bundle-corr-001",
  "status": "POSTED",
  "bundleEntries": [
    {
      "branch": 4919,
      "account": 223166,
      "createdAt": "2024-06-15T14:30:00Z",
      "entries": [
        {
          "id": "entry-corr-001",
          "entryKind": "ledger",
          "historyCode": 101,
          "status": "POSTED",
          "description": "Debit transfer out",
          "asOfDate": "2024-06-15",
          "correlations": { "transfer-system": "entry-corr-001" },
          "balanceChanges": { "AVAILABLE": { "value": -5000, "currency": "USD" } },
          "amount": { "value": 5000, "currency": "USD" }
        }
      ],
      "balances": {
        "AVAILABLE": { "value": 45000, "currency": "USD" }
      }
    },
    {
      "branch": 4919,
      "account": 999888,
      "createdAt": "2024-06-15T14:30:00Z",
      "entries": [ "..." ],
      "balances": { "AVAILABLE": { "value": 25000, "currency": "USD" } }
    }
  ]
}
```

#### entryBundleGroupResponse Fields

| Field | Type | Description |
|---|---|---|
| `bundleId` | `string` | The bundle's correlation ID |
| `status` | `bundleStatusResponse` | See [Bundle Status Values](#bundle-status-values) |
| `bundleEntries` | `bundleEntries[]` | Per-account results |

**bundleEntries** fields:

| Field | Type | Description |
|---|---|---|
| `branch` | `integer` | Account branch |
| `account` | `long` | Account number |
| `createdAt` | `datetime` | Timestamp of entry creation |
| `entries` | `entryResponse[]` | Entries created for this account |
| `balances` | `map<string, amount>` | Updated account balances by balance type |

#### Bundle Status Values

| Status | Description |
|---|---|
| `POSTED` | Bundle was committed synchronously |
| `PENDING` | Bundle was accepted but not yet posted |
| `POSTING` | Bundle is in the process of being posted |
| `NEW` | Bundle is newly created |
| `PENDING_ROLLBACK` | Rollback is pending |
| `ROLLED_BACK` | Bundle has been rolled back |
| `EXCLUDING` | Exclusion is in progress |
| `ROLLING_BACK` | Rollback is in progress |

#### Validations — Submit Bundle

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `bundle` (correlation ID) is required; length 1–128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `kind` is required and must be one of `regular`, `removal`, `exclusion` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `regular` bundles: `entries` is required and must contain 1–100 items | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `removal` bundles: `bundleToRemove` is required (length 1–128 chars) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each entry: `branch`, `account`, `entryKind`, `correlationId` are required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `correlationId` per entry: length 1–128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `description`, when provided, must not exceed 40 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `amount.value` must be ≥ 1 for `ledger`, `blocking`, `unblocking`, `release` entries | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `historyCode` is required for `ledger` entries and must exist in DTW | `http://dtw.matera.com/transaction/entry-type-not-found` | `Entry type not found` | `400` |
| `holdReasonId` is required for `blocking`/`unblocking` entries and must exist in DTW | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry **both** `dtw-composite-transaction::transact` AND `dtw-transaction::transact` | `about:blank` | `Forbidden` | `403` |
| All referenced accounts must exist in DTW | `http://dtw.matera.com/transaction/account-not-found` | `Account Not Found` | `422` |
| Available balance must be sufficient for all debit operations in the bundle | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |
| Re-submitting the same `bundle` ID with different content is rejected | `http://dtw.matera.com/transaction/correlation-conflict` | `Correlation conflict` | `409` |

> **Idempotency**: Re-submitting the same `bundle` ID with **identical content** returns the original `201` response without creating duplicate entries.

#### HTTP Status Codes

| Status | Response | Description |
|---|---|---|
| `201 Created` | `entryBundleGroupResponse` | Bundle submitted and entries created |
| `204 No Content` | — | Idempotent replay — same `bundle` ID and identical content |
| `400 Bad Request` | `ProblemResponse` | Invalid request body or fields |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `402 Payment Required` | `ProblemResponse` | Insufficient balance on one or more accounts |
| `403 Forbidden` | `ProblemResponse` | Missing one or more required authorities |
| `409 Conflict` | `ProblemResponse` | Bundle `correlationId` already submitted with different content |
| `410 Gone` | `ProblemResponse` | A target entry has been removed |
| `422 Unprocessable Entity` | `ProblemResponse` | Business rule violation (e.g. unknown history code, invalid account) |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### DELETE /v1/transaction-bundles/{bundle} — Delete Bundle

Removes all entries from a previously submitted bundle, identified by the bundle's correlation ID. The operation is atomic — all entries in the bundle are removed or none are.

#### Request

```http
DELETE /v1/transaction-bundles/bundle-corr-001
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Constraints | Description |
|---|---|---|---|---|---|
| `bundle` | path | `string` | Yes | 1–128 chars | The `bundle` correlation ID of the bundle to delete |

#### Validations — Delete Bundle

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `bundle` path parameter is required; length 1–128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry **both** `dtw-composite-transaction::transact` AND `dtw-transaction::transact` | `about:blank` | `Forbidden` | `403` |
| Account balance must remain valid after removing all entries in the bundle | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

#### Response — 204 No Content

| Status | Response | Description |
|---|---|---|
| `204 No Content` | — | Bundle deleted |
| `400 Bad Request` | `ProblemResponse` | Invalid path parameter (empty or exceeds 128 chars) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `402 Payment Required` | `ProblemResponse` | Insufficient balance after removal |
| `403 Forbidden` | `ProblemResponse` | Missing one or more required authorities |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

## Monitoring Release API

The composite service provides monitoring release endpoints that coordinate with `dtw-transaction`. These differ from the direct `dtw-transaction` release endpoints in that:

- They use a V1-style `If-Match` header (obtained from a prior GET on `dtw-transaction`)
- They are designed for scenarios where the release involves multi-account side effects that the composite layer needs to orchestrate
- They return an updated `ETag` header in the response for further concurrency control

### POST /v1/accounts/{branch}/{account}/monitorings/{monitoringId}/release — Release Account Monitoring

Releases a monitoring scoped to a single account.

#### Request

```http
POST /v1/accounts/4919/223166/monitorings/550e8400-e29b-41d4-a716-446655441000/release
Authorization: Bearer <token>
Content-Type: application/json
If-Match: "42"

{
  "settlementInfo": {
    "fulfillment": "TOTAL",
    "amount": { "value": 50000, "currency": "USD" }
  },
  "status": "INACTIVE",
  "blueprints": [
    {
      "entryKind": "ledger",
      "correlationId": "blueprint-corr-001",
      "historyCode": 300,
      "settlementInfo": {
        "fulfillment": "TOTAL",
        "amount": { "value": 50000, "currency": "USD" }
      },
      "description": "Installment debit"
    }
  ],
  "metaData": { "contractId": "C-9999" }
}
```

#### Path and Header Parameters

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | Account branch |
| `account` | path | `long` | Yes | Account number |
| `monitoringId` | path | `string` | Yes | Monitoring UUID (from `dtw-transaction`) |
| `If-Match` | header | `string` | Yes | ETag from a prior `GET` on `dtw-transaction`. See [Optimistic Locking](#optimistic-locking-etag--if-match) |

#### Field Reference — monitoringReleaseRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `settlementInfo` | `settlementInfoRequest` | Yes | Amount to release: `TOTAL` (with `amount`) or `PARTIAL` (with `amountRange`) |
| `status` | `string` enum | No (default `INACTIVE`) | Target monitoring status after release: `ACTIVE`, `INACTIVE`, `EXCLUDED`, `REALIZED` |
| `blueprints` | `monitoringReleaseEntryRequest[]` | No (0–100) | Entry instructions to execute on release. Only `ledgerEntryDebitRequest` is currently supported |
| `metaData` | `object` | No | Free-form metadata |

#### Validations — Release Account Monitoring

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `monitoringId` path parameter must be a valid UUID | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `If-Match` header is required (ETag obtained from a prior `GET` on `dtw-transaction`) | `http://dtw.matera.com/transaction/monitoring-version-mismatch` | `Monitoring service version mismatch` | `412` |
| `settlementInfo` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `fulfillment = "TOTAL"`: `amount` is required with `value ≥ 1` and a valid currency | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| For `fulfillment = "PARTIAL"`: `amountRange` is required with `min`, `max` (both ≥ 1), and `currency` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `blueprints`, when provided, must contain 0–100 items | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each blueprint `correlationId` is required; length 1–128 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each blueprint `historyCode` is required and must be registered in DTW | `http://dtw.matera.com/transaction/entry-type-not-found` | `Entry type not found` | `400` |
| Each blueprint `settlementInfo` is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `description` in blueprints, when provided, must not exceed 40 characters | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry **all four** authorities: `dtw-composite-transaction::release-monitorings`, `dtw-transaction::view-monitorings`, `dtw-transaction::modify-monitorings`, `dtw-transaction::release-monitorings` | `about:blank` | `Forbidden` | `403` |
| The monitoring must exist and be in a state that allows release | `http://dtw.matera.com/transaction/intent-not-found` | `Intent Not Found` | `404` |
| Account balance must be sufficient to fulfil the release | `http://dtw.matera.com/transaction/insufficient-funds` | `Insufficient funds` | `402` |

**monitoringReleaseEntryRequest** (concrete subtype: `ledgerEntryDebitRequest`):

| Field | Type | Required | Description |
|---|---|---|---|
| `entryKind` | `string` | Yes | Must be `ledger` |
| `correlationId` | `string` (1–128) | Yes | Unique ID for this blueprint entry |
| `historyCode` | `long` | Yes | History code for the created entry |
| `settlementInfo` | `settlementInfoRequest` | Yes | Amount for this specific blueprint entry |
| `description` | `string` (1–40) | No | Statement description |
| `metaData` | `object` | No | — |

#### Response — 200 OK

```json
{
  "monitoringId": "550e8400-e29b-41d4-a716-446655441000",
  "correlations": { "installment-system": "blueprint-corr-001" },
  "status": "REALIZED",
  "releasedAmount": { "value": 50000, "currency": "USD" },
  "totalAmount": { "value": 50000, "currency": "USD" },
  "amountDue": { "value": 0, "currency": "USD" },
  "totalReleased": { "value": 50000, "currency": "USD" },
  "metaData": { "contractId": "C-9999" },
  "entries": [
    {
      "branch": 4919,
      "account": 223166,
      "createdAt": "2024-06-15T14:30:00Z",
      "id": "blueprint-corr-001",
      "entryKind": "ledger",
      "historyCode": 300,
      "status": "POSTED",
      "amount": { "value": 50000, "currency": "USD" }
    }
  ]
}
```

Response includes an `ETag` header with the updated monitoring version.

#### monitoringReleaseResponse Fields

| Field | Type | Description |
|---|---|---|
| `monitoringId` | `string` | Monitoring UUID |
| `correlations` | `map<string, string>` | Originator → correlation ID |
| `status` | `string` | Final monitoring status |
| `releasedAmount` | `amount` | Amount released in this call |
| `totalAmount` | `amount` | Total amount originally under monitoring |
| `amountDue` | `amount` | Outstanding amount (0 if fully released) |
| `totalReleased` | `amount` | Cumulative released amount including this call |
| `metaData` | `object` | — |
| `entries` | `monitoringReleaseEntryResponse[]` | Entries created by this release |

**monitoringReleaseEntryResponse** contains all `entryResponse` fields plus `branch`, `account`, and `createdAt`.

#### HTTP Status Codes

| Status | Response | Description |
|---|---|---|
| `200 OK` | `monitoringReleaseResponse` | Released. Response includes updated `ETag` header |
| `204 No Content` | — | Idempotent replay — release already processed |
| `400 Bad Request` | `ProblemResponse` | Invalid or missing request fields |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `402 Payment Required` | `ProblemResponse` | Insufficient balance to fulfil the release |
| `403 Forbidden` | `ProblemResponse` | Missing one or more required authorities |
| `404 Not Found` | `ProblemResponse` | Monitoring not found |
| `412 Precondition Failed` | `ProblemResponse` | `If-Match` header absent or stale |
| `422 Unprocessable Entity` | `ProblemResponse` | Business rule violation (e.g. unknown history code) |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### POST /v1/account-groups/{kind}/{membershipId}/monitorings/{monitoringId}/release — Release Group Monitoring

Identical to the account monitoring release, but scoped to a membership group.

#### Request

```http
POST /v1/account-groups/HOLDER/12345678/monitorings/550e8400-e29b-41d4-a716-446655441000/release
Authorization: Bearer <token>
Content-Type: application/json
If-Match: "42"

{
  "settlementInfo": {
    "fulfillment": "PARTIAL",
    "amountRange": { "min": 1000, "max": 50000, "currency": "USD" }
  },
  "blueprints": [],
  "metaData": {}
}
```

#### Path Parameters

| Parameter | Type | Description |
|---|---|---|
| `kind` | `string` | Membership kind (e.g. `HOLDER`, `CORPORATE_GROUP_HOLDER_TAX_ID`) |
| `membershipId` | `string` | Membership identifier (e.g. a CNPJ) |
| `monitoringId` | `string` | Monitoring UUID |

All other parameters (`If-Match`, request body, and response body) are identical to the account monitoring release.

#### Validations — Release Group Monitoring

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `kind` path parameter must be a valid membership group kind (e.g. `HOLDER`, `CORPORATE_GROUP_HOLDER_TAX_ID`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `membershipId` path parameter is required and must not be empty | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `monitoringId` must be a valid UUID | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| All body and `If-Match` validations are identical to the account monitoring release (see above) | `—` | `—` | `—` |
| Token must carry all four required authorities (same as account release) | `about:blank` | `Forbidden` | `403` |

#### HTTP Status Codes

Same as the account monitoring release endpoint.

---

## Complete Error Reference

| HTTP Status | When it occurs |
|---|---|
| `400 Bad Request` | Invalid/missing request fields; constraint violations on parameters or body |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `402 Payment Required` | Insufficient balance on one or more accounts involved in the bundle |
| `403 Forbidden` | Missing one or more required authorities (`dtw-composite-transaction::*` or `dtw-transaction::*`) |
| `409 Conflict` | Bundle `correlationId` was already submitted with different content (non-idempotent re-submission) |
| `410 Gone` | A target entry referenced by the operation has been removed |
| `412 Precondition Failed` | `If-Match` header absent or stale (returned by `dtw-transaction`, surfaced by composite) |
| `500 Internal Server Error` | Unexpected server-side or downstream failure |

---

## Appendix

---

## Full Schema Definitions

### bundleRequest discriminator

```yaml
discriminator:
  propertyName: kind
  mapping:
    regular: regularBundleRequest
    removal: removalBundleRequest
    exclusion: exclusionBundleRequest
```

### bundleStatus (request-time)
`POSTED` | `PENDING`

### bundleStatusResponse (response)
`POSTED` | `PENDING` | `POSTING` | `NEW` | `PENDING_ROLLBACK` | `ROLLED_BACK` | `EXCLUDING` | `ROLLING_BACK`

### monitoringStatus (release request/response)
`ACTIVE` | `INACTIVE` | `EXCLUDED` | `REALIZED`

### entryStatus (per entry)
`POSTED` | `PENDING` | `ON_HOLD` | `DEFERRED`

### entryKind (request)
`ledger` | `release` | `reversal` | `removal` | `blocking` | `unblocking`

### entryKind (response)
`ledger` | `release` | `reversal` | `removal` | `blocking` | `unblocking` | `exclusion`

---

## Advanced Integration Topics

### Composite vs. Direct Transaction API

Use the composite service when:
- A single business operation must debit one account and credit another **atomically** (e.g. internal transfers)
- A monitoring release must trigger entries on multiple accounts simultaneously
- You need bundle-level idempotency across accounts (the `bundle` ID is the idempotency key)

Use `dtw-transaction` directly when:
- All entries target the **same account**
- You need full monitoring CRUD (create, update, list, cancel)
- You need the V2 release endpoints (by correlation ID)

### Bundle Idempotency

The `bundle` field in the request is the idempotency key for the entire bundle. Re-submitting a `regularBundleRequest` with the same `bundle` ID and **identical content** returns the same `201` response without creating duplicate entries. Re-submitting with the **same ID but different content** returns `409 Conflict`.

### Authority Scoping

The composite service enforces two independent authority namespaces:

- `dtw-composite-transaction::<role>` — grants permission at the composite layer
- `dtw-transaction::<role>` — required for the downstream forwarded call

Both must be present in the same JWT. Confirm with the DTW team that your client's JWT carries both when setting up a new integration.

### OpenAPI Specification

The OpenAPI specification for this API is available as a YAML file upon request. It can be used for client code generation or tooling integration.
