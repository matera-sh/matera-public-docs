# Digital Twin Integration Guide — Registry REST API

**Audience:** Developers integrating with the Digital Twin Registry to query account information, manage account limits, and manage system-wide limits via HTTP.

---

## Registry APIs

Digital Twin Registry APIs are used for managing accounts, account memberships, account-level and system-wide limits, and account configuration. Monetary amounts are represented as unscaled integers in minor units. Accounts also support typed configuration values called *features* (BOOLEAN, STRING, DECIMAL, DATE, and DATE_RANGE), which are structured account settings that control account behavior. Unlike a transaction's free-form `metaData`, features are defined for an account and can be updated over time.

This guide covers querying accounts and their features, retrieving accounts by membership key, and managing limit ranges. It does **not** cover ledger operations (postings, blocks, or reversals), which are described in the *Digital Twin Integration Guide — Commands*, or data population (accounts, history codes, and hold reasons), which is described in the *Digital Twin Integration Guide — Events*.

### When to use this service vs. others

| Need | Service |
|---|---|
| Look up an account's current status, type, features, or memberships | **Registry API** (this guide) |
| Find all accounts associated with a holder by tax ID or membership key | **Registry API** (this guide) |
| Read or manage per-account financial limits | **Registry API** (this guide) |
| Create accounts, account types, history codes, or hold reasons | Events Integration Guide |
| Post ledger entries, holds, reversals, or manage monitorings | [Transaction REST API](./02-dtw-transaction-api-integration-guide.md) |
| Submit entries spanning multiple accounts atomically | [Composite API](./05-dtw-composite-api-integration-guide.md) |
| Query account balances or statement history | [Balance & Statement API](./04-dtw-balance-statement-api-integration-guide.md) |

---

## Table of Contents

1. [Prerequisites](#prerequisites)

**Shared Building Blocks**

2. [Authentication and Authorization](#authentication-and-authorization)
3. [Common Request and Response Types](#common-request-and-response-types)
4. [Error Handling](#error-handling)
5. [Pagination](#pagination)

**API Catalog**

6. [Accounts API](#accounts-api)
   - [GET /v1/accounts — Query Accounts by Key](#get-v1accounts--query-accounts-by-key)
   - [GET /v1/accounts/{branch}/{account}/limits — List Account Limits](#get-v1accountsbranchaccountlimits--list-account-limits)
   - [GET /v1/accounts/{branch}/{account}/limits/{limitType} — Get Account Limit by Type](#get-v1accountsbranchaccountlimitslimittype--get-account-limit-by-type)
   - [PUT /v1/accounts/{branch}/{account}/limits/{limitType} — Create or Update Account Limit](#put-v1accountsbranchaccountlimitslimittype--create-or-update-account-limit)
   - [DELETE /v1/accounts/{branch}/{account}/limits/{limitType} — Delete Account Limit](#delete-v1accountsbranchaccountlimitslimittype--delete-account-limit)
7. [Memberships API](#memberships-api)
   - [GET /v1/memberships/accounts — Query Accounts by Membership](#get-v1membershipsaccounts--query-accounts-by-membership)
8. [System Limits API](#system-limits-api)
    - [GET /v1/system/limits — List System Limits](#get-v1systemlimits--list-system-limits)
    - [GET /v1/system/limits/{limitType} — Get System Limit by Type](#get-v1systemlimitslimittype--get-system-limit-by-type)
    - [PUT /v1/system/limits/{limitType} — Create or Update System Limit](#put-v1systemlimitslimittype--create-or-update-system-limit)
    - [DELETE /v1/system/limits/{limitType} — Delete System Limit](#delete-v1systemlimitslimittype--delete-system-limit)

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
| OAuth2 client credentials | A `clientId` and `clientSecret` issued by the DTW Keycloak realm | Every API call requires a Bearer token. Obtain credentials from the DTW team |
| Required client roles | The specific roles for the operations you need (e.g. `view-accounts`, `modify-account-limits`) | Authorization is enforced per endpoint — requests with missing roles receive `403 Forbidden` |
| The target accounts must already exist in DTW | Accounts are populated via the Events flow — see the Events guide | Limit endpoints reference an existing `branch`/`account` pair. Querying a non-existent account returns `404 Not Found` |
| Allowed limit types | Configured server-side via the `DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE` environment variable | The `limitType` path segment must be one of the configured values; confirm the list with the DTW team before going live |

---

## Authentication and Authorization

### OAuth2 Credentials

All endpoints require a JWT bearer token obtained using the OAuth2 Client Credentials grant from the DTW Keycloak realm. Include the token in the `Authorization` header of every request:

```
POST {DTW_REGISTRY_OIDC_AUTHORIZATION_HOST}/auth/realms/DTW/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=<your-client-id>
&client_secret=<your-client-secret>
```

The token issuer URL follows the pattern:

```
{DTW_REGISTRY_OIDC_AUTHORIZATION_HOST}/auth/realms/DTW
```

Include the token in every API request:

```http
Authorization: Bearer <access_token>
```

### Required Roles

Roles are extracted from the JWT as client-level roles on the `dtw-registry` client. Requesting a role that is not granted to your client returns `403 Forbidden`.

| Endpoint | Required Role |
|---|---|
| `GET /v1/accounts` | `view-accounts` |
| `GET /v1/accounts/{branch}/{account}/limits` | `view-account-limits` |
| `GET /v1/accounts/{branch}/{account}/limits/{limitType}` | `view-account-limits` |
| `PUT /v1/accounts/{branch}/{account}/limits/{limitType}` | `modify-account-limits` |
| `DELETE /v1/accounts/{branch}/{account}/limits/{limitType}` | `modify-account-limits` |
| `GET /v1/memberships/accounts` | `view-account-memberships` |
| `GET /v1/system/limits` | `view-system-limits` |
| `GET /v1/system/limits/{limitType}` | `view-system-limits` |
| `PUT /v1/system/limits/{limitType}` | `modify-system-limits` |
| `DELETE /v1/system/limits/{limitType}` | `modify-system-limits` |

---

## Common Request and Response Types

### Account Object

Returned as elements inside `AccountResponse.accounts[]` by the account query endpoints.

```json
{
  "branch": 4919,
  "account": 223166,
  "status": "ACTIVE",
  "timeZone": "America/Chicago",
  "type": {
    "id": "CHECKING",
    "description": "Checking Account",
    "category": "individual",
    "features": [
      { "name": "OVERDRAFT_ENABLED", "value": "true", "type": "BOOLEAN" }
    ]
  },
  "features": [
    { "name": "BLOCKED_FOR_DEBIT", "value": "false", "type": "BOOLEAN" },
    { "name": "DAILY_LIMIT", "value": "5000.00", "type": "DECIMAL" }
  ],
  "memberships": [
    {
      "id": "60672403000109",
      "kind": "COMPANY_HOLDER_TAX_ID",
      "features": []
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `branch` | `integer` (≥ 0) | Branch number (Routing Transit Number / RTN) |
| `account` | `long` (≥ 0) | Account number |
| `status` | `ACTIVE` \| `INACTIVE` | Current account status |
| `timeZone` | `string` | IANA time zone identifier (e.g. `"America/Chicago"`) |
| `type` | `AccountType` | Account type metadata — see below |
| `features` | `Feature[]` | Account-level feature flags and parameters |
| `memberships` | `Membership[]` | Membership associations for this account |

### AccountType Object

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Account type identifier |
| `description` | `string` | Human-readable description |
| `category` | `string` | Category label (e.g. `"checking account"`) |
| `features` | `Feature[]` | Type-level feature flags and parameters |

### Feature Object

| Field | Type | Values | Description |
|---|---|---|---|
| `name` | `string` | — | Feature flag name (e.g. `BLOCKED_FOR_CREDIT`, `DAILY_LIMIT`) |
| `value` | `string` | — | Serialized value — a string regardless of type (e.g. `"true"`, `"12345"`, `"2024-01-01"`) |
| `type` | `string` | `BOOLEAN`, `DECIMAL`, `DATE`, `STRING` | Logical type of the value — use this to parse `value` correctly |

### Membership Object

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Membership identifier (e.g. a natural person or legal entity identification number) |
| `kind` | `string` | Membership kind — see [Membership Kinds](#membership-kinds) below |
| `features` | `Feature[]` | Membership-level feature flags |

#### Membership Kinds

| Kind | Description |
|---|---|
| `PERSON_HOLDER_TAX_ID` | Natural person identification of an individual account holder |
| `COMPANY_HOLDER_TAX_ID` | Legal entity identification of a company account holder |
| `CORPORATE_GROUP_HOLDER_TAX_ID` | Legal entity identification of a corporate group |
| `NON_SALARY_PERSON_HOLDER_TAX_ID` | Natural person identification for non-salary individuals |

### LimitResult Object

Returned by all limit read and write endpoints.

```json
{
  "type": "OVERDRAFT",
  "limit": {
    "currency": "USD",
    "min": 0,
    "max": 10000
  },
  "lastUpdatedAt": "2024-06-01T12:00:00Z"
}
```

| Field | Type | Description |
|---|---|---|
| `type` | `string` | The limit type identifier (e.g. `"OVERDRAFT"`, `"FEDNOW_MONTHLY_LIMIT"`). Default: `"ALL"` |
| `limit` | `Range` | The min/max range for the limit — see below |
| `lastUpdatedAt` | `string` (ISO 8601) | Timestamp of the last update |

### Range Object

Used in both `LimitResult` and `LimitRequest`.

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | `string` (ISO 4217) | Yes | Currency code (e.g. `"USD"`) |
| `min` | `long` | Yes | Minimum value of the limit range |
| `max` | `long` | Yes | Maximum value of the limit range |

### LimitRequest Object

Request body for PUT limit endpoints.

```json
{
  "limit": {
    "currency": "USD",
    "min": 0,
    "max": 50000
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `limit` | `Range` | Yes | The new limit range to apply |

### AccountResponse Object

Top-level envelope returned by account query endpoints.

| Field | Type | Description |
|---|---|---|
| `accounts` | `Account[]` | The accounts matching the query. May be empty |

---

## Error Handling

All error responses use the `application/problem+json` media type (RFC 7807). The response body is a `ProblemResponse`:

```json
{
  "type": "https://zalando.github.io/problem/constraint-violation",
  "title": "Constraint Violation",
  "status": 400,
  "detail": "getAccountNotMigratingByKey.key: must match '^\\d+/\\d+$'"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `URI` | No (default: `about:blank`) | URI identifying the problem type |
| `title` | `string` | Yes | Short, human-readable summary |
| `status` | `integer` (100–600) | Yes | HTTP status code |
| `detail` | `string` | No | Human-readable explanation specific to this occurrence |
| *(additional fields)* | varies | No | RFC 7807 extension fields (e.g. `violations[]` for constraint errors) |

### HTTP Status Codes

| Status | Meaning |
|---|---|
| `200 OK` | Request succeeded. Response body contains the requested data |
| `204 No Content` | Request succeeded. No response body (DELETE operations) |
| `400 Bad Request` | Request validation failed (malformed parameters, constraint violation) |
| `401 Unauthorized` | Missing or invalid Bearer token |
| `403 Forbidden` | Valid token but insufficient roles for this endpoint |
| `404 Not Found` | The specified resource (account, limit type) does not exist |
| `500 Internal Server Error` | Unexpected server-side failure |

---

## Pagination

Endpoints that return lists support offset-based pagination via query parameters:

| Parameter | Type | Default | Constraints | Description |
|---|---|---|---|---|
| `page` | `integer` | `0` | `>= 0` | Zero-based page index |
| `size` | `integer` | `20` | `1–100` | Number of items per page |

---

## Accounts API

### GET /v1/accounts — Query Accounts by Key

Retrieves one or more accounts by their `branch/account` key. Use this to look up current account state, including status, type, features, and membership associations.

#### Request

```http
GET /v1/accounts?key=4919/223166&key=4919/987654&page=0&size=20
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Pattern | Description |
|---|---|---|---|---|---|
| `key` | query | `string[]` | Yes | `^\d+/\d+$` | One or more `BranchNumber/AccountNumber` pairs. Can be repeated or comma-separated |
| `page` | query | `integer` | No | — | Page index (default `0`) |
| `size` | query | `integer` | No | 1–100 | Page size (default `20`) |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `key` is required; at least one value must be supplied | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each `key` value must match the pattern `^\d+/\d+$` (e.g. `4919/223166`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `page` must be ≥ 0 | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `size` must be between 1 and 100 (inclusive) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid (not expired, issued by the correct realm) | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-accounts` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |

> Accounts not found for the supplied `key`s are silently omitted from the result — they do **not** produce `404`.

#### Response — 200 OK

```json
{
  "accounts": [
    {
      "branch": 4919,
      "account": 223166,
      "status": "ACTIVE",
      "timeZone": "America/Chicago",
      "type": {
        "id": "CHECKING",
        "description": "Checking Account",
        "category": "individual",
        "features": []
      },
      "features": [
        { "name": "BLOCKED_FOR_DEBIT", "value": "false", "type": "BOOLEAN" }
      ],
      "memberships": [
        { "id": "12345678901", "kind": "PERSON_HOLDER_TAX_ID", "features": [] }
      ]
    }
  ]
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `AccountResponse` | Accounts matching the supplied keys. Not-found accounts are silently omitted from the list |
| `400 Bad Request` | `ProblemResponse` | Invalid `key` format or pagination parameters |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-accounts` role |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server error |

---

### GET /v1/accounts/{branch}/{account}/limits — List Account Limits

Returns all financial limits configured for the specified account.

#### Request

```http
GET /v1/accounts/4919/223166/limits
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | Branch number (Routing Transit Number / RTN) |
| `account` | path | `long` | Yes | Account number |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` must be parseable as a non-negative integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` must be parseable as a non-negative long | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-account-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in DTW Registry | `http://dtw.matera.com/registry/account-not-found` | `Not Found` | `404` |

#### Response — 200 OK

```json
[
  {
    "type": "OVERDRAFT",
    "limit": { "currency": "USD", "min": 0, "max": 10000 },
    "lastUpdatedAt": "2024-06-01T12:00:00Z"
  },
  {
    "type": "FEDNOW_MONTHLY_LIMIT",
    "limit": { "currency": "USD", "min": 0, "max": 50000 },
    "lastUpdatedAt": "2024-05-15T08:30:00Z"
  }
]
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult[]` | All limits for this account. Empty array if none configured |
| `400 Bad Request` | `ProblemResponse` | Invalid path parameters (non-numeric `branch` or `account`) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-account-limits` role |
| `404 Not Found` | `ProblemResponse` | Account not found in DTW |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### GET /v1/accounts/{branch}/{account}/limits/{limitType} — Get Account Limit by Type

Returns a single financial limit for the specified account and limit type.

#### Request

```http
GET /v1/accounts/4919/223166/limits/OVERDRAFT
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | Branch number (Routing Transit Number / RTN) |
| `account` | path | `long` | Yes | Account number |
| `limitType` | path | `string` | Yes | Limit type identifier — must be one of the allowed types configured on the server |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` must be parseable as a non-negative integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` must be parseable as a non-negative long | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-account-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in DTW Registry | `http://dtw.matera.com/registry/account-not-found` | `Not Found` | `404` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |
| A limit of this type must be configured for this account | `http://dtw.matera.com/registry/account-limit-not-found` | `Not Found` | `404` |

#### Response — 200 OK

```json
{
  "type": "OVERDRAFT",
  "limit": { "currency": "USD", "min": 0, "max": 10000 },
  "lastUpdatedAt": "2024-06-01T12:00:00Z"
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult` | The limit for this account and type |
| `400 Bad Request` | `ProblemResponse` | Invalid path parameters (non-numeric `branch` or `account`) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-account-limits` role |
| `404 Not Found` | `ProblemResponse` | Account not found in DTW; or `limitType` not in allowed list; or no limit of this type configured for this account |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### PUT /v1/accounts/{branch}/{account}/limits/{limitType} — Create or Update Account Limit

Creates or replaces the financial limit of the specified type for a given account. This operation is idempotent — calling it multiple times with the same values produces the same result.

#### Request

```http
PUT /v1/accounts/4919/223166/limits/OVERDRAFT
Authorization: Bearer <token>
Content-Type: application/json

{
  "limit": {
    "currency": "USD",
    "min": 0,
    "max": 10000
  }
}
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | Branch number (Routing Transit Number / RTN) |
| `account` | path | `long` | Yes | Account number |
| `limitType` | path | `string` | Yes | Limit type identifier |
| *(body)* | body | `LimitRequest` | Yes | The new limit range to apply |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` must be parseable as a non-negative integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` must be parseable as a non-negative long | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Request body (`Content-Type: application/json`) is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit` field is required in the request body | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.currency` is required and must be a valid ISO 4217 code or supported crypto ticker (`BTC`, `USC`, `UST`, `SOL`, `XRP`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.min` is required (cannot be null) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.max` is required (cannot be null) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-account-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in DTW Registry | `http://dtw.matera.com/registry/account-not-found` | `Not Found` | `404` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |

#### Response — 200 OK

```json
{
  "type": "OVERDRAFT",
  "limit": { "currency": "USD", "min": 0, "max": 10000 },
  "lastUpdatedAt": "2024-06-23T09:15:00Z"
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult` | The updated limit, with the new `lastUpdatedAt` timestamp |
| `400 Bad Request` | `ProblemResponse` | Missing or invalid fields in the request body (e.g. `limit` absent, `currency` invalid) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `modify-account-limits` role |
| `404 Not Found` | `ProblemResponse` | Account not found in DTW; or `limitType` not in allowed list |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### DELETE /v1/accounts/{branch}/{account}/limits/{limitType} — Delete Account Limit

Removes the financial limit of the specified type from the given account.

#### Request

```http
DELETE /v1/accounts/4919/223166/limits/OVERDRAFT
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `branch` | path | `integer` | Yes | Branch number (Routing Transit Number / RTN) |
| `account` | path | `long` | Yes | Account number |
| `limitType` | path | `string` | Yes | Limit type identifier |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `branch` must be parseable as a non-negative integer | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `account` must be parseable as a non-negative long | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-account-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| The `branch`/`account` pair must exist in DTW Registry | `http://dtw.matera.com/registry/account-not-found` | `Not Found` | `404` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |
| A limit of this type must be configured for this account | `http://dtw.matera.com/registry/account-limit-not-found` | `Not Found` | `404` |

#### Response — 204 No Content

No response body on success.

| Status | Response Body | Description |
|---|---|---|
| `204 No Content` | — | Limit successfully deleted |
| `400 Bad Request` | `ProblemResponse` | Invalid path parameters (non-numeric `branch` or `account`) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `modify-account-limits` role |
| `404 Not Found` | `ProblemResponse` | Account not found in DTW; or `limitType` not in allowed list; or no limit of this type configured for this account |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

## Memberships API

### GET /v1/memberships/accounts — Query Accounts by Membership

Returns accounts that are associated with the given membership key(s) — for example, all accounts held by a given tax ID. This is the complement to the key-based account query for use cases where only a natural person or legal entity identification number is known.

#### Request

```http
GET /v1/memberships/accounts?membershipKey=PERSON_HOLDER_TAX_ID/12345678901&page=0&size=20
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Pattern | Description |
|---|---|---|---|---|---|
| `membershipKey` | query | `string[]` | Yes | `^\w+/\w+$` | One or more `MembershipKind/MembershipId` pairs. Can be repeated or comma-separated. See [Membership Kinds](#membership-kinds) for valid kind values |
| `page` | query | `integer` | No | — | Page index (default `0`) |
| `size` | query | `integer` | No | 1–100 | Page size (default `20`) |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| `membershipKey` is required; at least one value must be supplied | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Each `membershipKey` must match the pattern `^\w+/\w+$` (e.g. `PERSON_HOLDER_TAX_ID/12345678901`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| The `MembershipKind` segment must be one of: `PERSON_HOLDER_TAX_ID`, `COMPANY_HOLDER_TAX_ID`, `CORPORATE_GROUP_HOLDER_TAX_ID`, `NON_SALARY_PERSON_HOLDER_TAX_ID` | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `page` must be ≥ 0 | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `size` must be between 1 and 100 (inclusive) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-account-memberships` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |

> Membership keys with no associated accounts return an empty list — they do **not** produce `404`.

#### Response — 200 OK

```json
{
  "accounts": [
    {
      "branch": 4919,
      "account": 223166,
      "status": "ACTIVE",
      "timeZone": "America/Chicago",
      "type": {
        "id": "CHECKING",
        "description": "Checking Account",
        "category": "individual",
        "features": []
      },
      "features": [],
      "memberships": [
        { "id": "12345678901", "kind": "PERSON_HOLDER_TAX_ID", "features": [] }
      ]
    }
  ]
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `AccountResponse` | Accounts associated with the given membership keys. Empty list if none found |
| `400 Bad Request` | `ProblemResponse` | Invalid `membershipKey` format or pagination parameters |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-account-memberships` role |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

## System Limits API

System limits are global defaults that apply across all accounts, independently of per-account limits. They follow the same `LimitResult` / `LimitRequest` / `Range` schema as account limits.

### GET /v1/system/limits — List System Limits

Returns all system-wide financial limits currently configured.

#### Request

```http
GET /v1/system/limits
Authorization: Bearer <token>
```

No parameters.

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-system-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |

#### Response — 200 OK

```json
[
  {
    "type": "OVERDRAFT",
    "limit": { "currency": "USD", "min": 0, "max": 100000 },
    "lastUpdatedAt": "2024-01-01T00:00:00Z"
  }
]
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult[]` | All system limits. Empty array if none configured |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-system-limits` role |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### GET /v1/system/limits/{limitType} — Get System Limit by Type

Returns the system-wide limit for a specific limit type.

#### Request

```http
GET /v1/system/limits/OVERDRAFT
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `limitType` | path | `string` | Yes | Limit type identifier |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `view-system-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |
| A system-wide limit of this type must be configured | `http://dtw.matera.com/registry/system-limit-not-found` | `Not Found` | `404` |

#### Response — 200 OK

```json
{
  "type": "OVERDRAFT",
  "limit": { "currency": "USD", "min": 0, "max": 100000 },
  "lastUpdatedAt": "2024-01-01T00:00:00Z"
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult` | The system limit for this type |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `view-system-limits` role |
| `404 Not Found` | `ProblemResponse` | `limitType` not in allowed list, or no system limit of this type configured |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### PUT /v1/system/limits/{limitType} — Create or Update System Limit

Creates or replaces the system-wide limit for the given type.

#### Request

```http
PUT /v1/system/limits/OVERDRAFT
Authorization: Bearer <token>
Content-Type: application/json

{
  "limit": {
    "currency": "USD",
    "min": 0,
    "max": 100000
  }
}
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `limitType` | path | `string` | Yes | Limit type identifier |
| *(body)* | body | `LimitRequest` | Yes | The new limit range to apply |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| Request body (`Content-Type: application/json`) is required | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit` field is required in the request body | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.currency` is required and must be a valid ISO 4217 code or supported crypto ticker (`BTC`, `USC`, `UST`, `SOL`, `XRP`) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.min` is required (cannot be null) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| `limit.max` is required (cannot be null) | `https://zalando.github.io/problem/constraint-violation` | `Constraint Violation` | `400` |
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-system-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |

#### Response — 200 OK

```json
{
  "type": "OVERDRAFT",
  "limit": { "currency": "USD", "min": 0, "max": 100000 },
  "lastUpdatedAt": "2024-06-23T09:15:00Z"
}
```

| Status | Response Body | Description |
|---|---|---|
| `200 OK` | `LimitResult` | The updated system limit |
| `400 Bad Request` | `ProblemResponse` | Missing or invalid request body fields (e.g. `limit` absent, `currency` invalid) |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `modify-system-limits` role |
| `404 Not Found` | `ProblemResponse` | `limitType` not in allowed list |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

### DELETE /v1/system/limits/{limitType} — Delete System Limit

Removes the system-wide limit for the given type.

#### Request

```http
DELETE /v1/system/limits/OVERDRAFT
Authorization: Bearer <token>
```

| Parameter | In | Type | Required | Description |
|---|---|---|---|---|
| `limitType` | path | `string` | Yes | Limit type identifier |

#### Validations

| Rule | `type` | `title` | Status |
|---|---|---|---|
| Bearer token must be present and valid | `about:blank` | `Unauthorized` | `401` |
| Token must carry the `modify-system-limits` role on the `dtw-registry` client | `about:blank` | `Forbidden` | `403` |
| `limitType` must be in the server-configured allowed list (`DTW_REGISTRY_CONSUMER_ALLOWED_LIMIT_TYPE`) | `http://dtw.matera.com/registry/limit-type-not-allowed` | `Bad Request` | `400` |
| `limitType` must be a type known to the system | `http://dtw.matera.com/registry/unknown-limit-type` | `Not Found` | `404` |

#### Response — 204 No Content

No response body on success.

| Status | Response Body | Description |
|---|---|---|
| `204 No Content` | — | System limit successfully deleted |
| `401 Unauthorized` | `ProblemResponse` | Missing or invalid Bearer token |
| `403 Forbidden` | `ProblemResponse` | Valid token but missing `modify-system-limits` role |
| `404 Not Found` | `ProblemResponse` | `limitType` not in allowed list |
| `500 Internal Server Error` | `ProblemResponse` | Unexpected server-side failure |

---

## Complete Error Reference

All `ProblemResponse` bodies include at minimum `title` and `status`. The `type` URI describes the problem category.

| HTTP Status | Common `title` | When it occurs |
|---|---|---|
| `400 Bad Request` | `Constraint Violation` | Invalid parameter format (e.g. `key` does not match `^\d+/\d+$`), out-of-range pagination values, missing required request body fields |
| `401 Unauthorized` | `Unauthorized` | Bearer token absent, expired, or issued by an unrecognized issuer |
| `403 Forbidden` | `Forbidden` | Token valid but client does not hold the required role for this endpoint |
| `404 Not Found` | `Not Found` | Account not found in DTW Registry (account limit endpoints), or no limit of the requested type exists |
| `500 Internal Server Error` | `Service Unavailable` (or similar) | Unexpected server-side failure — contact the DTW team if recurring |

> **Constraint violation responses** include a `violations` array extension field with per-field details:
>
> ```json
> {
>   "type": "https://zalando.github.io/problem/constraint-violation",
>   "title": "Constraint Violation",
>   "status": 400,
>   "violations": [
>     { "field": "key", "message": "must match '^\\d+/\\d+$'" }
>   ]
> }
> ```

---

## Appendix

---

## Full Schema Definitions

### AccountResponse

```json
{
  "type": "object",
  "properties": {
    "accounts": {
      "type": "array",
      "items": { "$ref": "#/components/schemas/Account" }
    }
  }
}
```

### Account

```json
{
  "type": "object",
  "properties": {
    "branch":      { "type": "integer", "minimum": 0 },
    "account":     { "type": "integer", "format": "int64", "minimum": 0 },
    "status":      { "type": "string", "enum": ["ACTIVE", "INACTIVE"] },
    "timeZone":    { "type": "string" },
    "type":        { "$ref": "#/components/schemas/AccountType" },
    "features":    { "type": "array", "items": { "$ref": "#/components/schemas/Feature" } },
    "memberships": { "type": "array", "items": { "$ref": "#/components/schemas/Membership" } }
  }
}
```

### AccountType

```json
{
  "type": "object",
  "properties": {
    "id":          { "type": "string" },
    "description": { "type": "string" },
    "category":    { "type": "string" },
    "features":    { "type": "array", "items": { "$ref": "#/components/schemas/Feature" } }
  }
}
```

### Feature

```json
{
  "type": "object",
  "properties": {
    "name":  { "type": "string" },
    "value": { "type": "string" },
    "type":  { "type": "string", "enum": ["STRING", "BOOLEAN", "DECIMAL", "DATE"] }
  }
}
```

### Membership

```json
{
  "type": "object",
  "properties": {
    "id":       { "type": "string" },
    "kind":     { "type": "string" },
    "features": { "type": "array", "items": { "$ref": "#/components/schemas/Feature" } }
  }
}
```

### LimitRequest

```json
{
  "type": "object",
  "required": ["limit"],
  "properties": {
    "limit": { "$ref": "#/components/schemas/Range" }
  }
}
```

### LimitResult

```json
{
  "type": "object",
  "properties": {
    "type":          { "type": "string", "default": "ALL" },
    "limit":         { "$ref": "#/components/schemas/Range" },
    "lastUpdatedAt": { "type": "string", "format": "date-time" }
  }
}
```

### Range

```json
{
  "type": "object",
  "required": ["currency", "min", "max"],
  "properties": {
    "currency": { "type": "string", "description": "ISO 4217 currency code" },
    "min":      { "type": "integer", "format": "int64" },
    "max":      { "type": "integer", "format": "int64" }
  }
}
```

### ProblemResponse

```json
{
  "type": "object",
  "required": ["title", "status"],
  "properties": {
    "type":   { "type": "string", "format": "uri", "default": "about:blank" },
    "title":  { "type": "string" },
    "status": { "type": "integer", "minimum": 100, "maximum": 600 },
    "detail": { "type": "string" }
  },
  "additionalProperties": true
}
```

---

## Advanced Integration Topics

### OpenAPI Specification

The OpenAPI specification for this API is available as a YAML file upon request. It can be used for client code generation or tooling integration.

### Pagination Behavior

Pagination is offset-based. The response envelope does not currently include total-count or next-page metadata — iterate by incrementing `page` until you receive fewer items than `size`.
