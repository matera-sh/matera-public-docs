# Digital Twin Integration Guide — Incoming Events

**Audience:** Developers integrating core banking systems and other upstream systems with Digital Twin

---

## Getting Started

This guide describes the events used to populate and maintain the account and reference data required by Digital Twin. It explains when each event should be sent, the data required for each event, and how Digital Twin processes those events to keep its data synchronized with upstream systems. It does not describe the APIs or messages used to submit transaction authorization requests, nor the events Digital Twin publishes for other systems to consume — those are covered in separate guides.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)

**Shared Building Blocks**

3. [When to Send Events](#when-to-send-events)
4. [In What Order to Send Events](#in-what-order-to-send-events)
5. [Kafka Topic Naming Convention](#kafka-topic-naming-convention)
6. [What's Inside Every Event](#whats-inside-every-event)

**Event Catalog**

7. [Account Type Events](#account-type-events)
   - [AccountTypeChangeEvent](#accounttypechangeevent)
   - [AccountTypeDeleteEvent](#accounttypedeleteevent)
   - [AccountTypeFeatureChangeEvent](#accounttypefeaturechangeevent)
   - [AccountTypeFeatureDeleteEvent](#accounttypefeaturedeletevent)
8. [Account Events](#account-events)
   - [AccountChangeEvent](#accountchangeevent)
   - [AccountDeleteEvent](#accountdeleteevent)
   - [AccountStatusChangeEvent](#accountstatuschangeevent)
   - [AccountFeatureChangeEvent](#accountfeaturechangeevent)
   - [AccountFeatureDeleteEvent](#accountfeaturedeleteevent)
9. [Account Membership Events](#account-membership-events)
   - [AccountHolderChangeEvent](#accountholderchangeevent)
   - [AccountHolderDeleteEvent](#accountholderdeleteevent)
   - [AccountMembershipFeatureChangeEvent](#accountmembershipfeaturechangeevent)
   - [AccountMembershipFeatureDeleteEvent](#accountmembershipfeaturedeleteevent)
10. [History Code Events](#history-code-events)
    - [HistoryCodeChangeEvent](#historycodechangeevent)
    - [HistoryCodeDeleteEvent](#historycodedeleteevent)
    - [HistoryCodeFeatureChangeEvent](#historycodefeaturechangeevent)
    - [HistoryCodeFeatureDeleteEvent](#historycodefeaturedeletevent)
11. [Generic Domain Events](#generic-domain-events)
    - [GenericDomainChangeEvent](#genericdomainchangeevent)
    - [GenericDomainDeleteEvent](#genericdomaindeleteevent)
    - [GenericDomainFeatureChangeEvent](#genericdomainfeaturechangeevent)
    - [GenericDomainFeatureDeleteEvent](#genericdomainfeaturedeleteevent)

**Operations**

12. [Event Validation and Error Handling](#event-validation-and-error-handling)
    - [What Happens When An Event Fails?](#what-happens-when-an-event-fails)

**Appendix**

13. [Appendix](#appendix)
    - [Full Schema Definitions](#full-schema-definitions)
    - [Advanced Integration Topics](#advanced-integration-topics)

---

## Overview

Digital Twin (DTW) is Matera's real-time authorization ledger. It maintains a synchronized copy of selected account and configuration data from your core banking system or other upstream systems, enabling real-time transaction authorization without replacing those source platforms.

Digital Twin receives account and reference data through event messages published by upstream systems. Depending on your architecture, these events may originate directly from your core banking system or from an integration component, messaging layer, or other intermediary that prepares and publishes events to Digital Twin.

The publishing system is responsible for detecting changes to account and reference data, transforming those changes into the event formats described in this guide, and publishing them to the appropriate Digital Twin Kafka topics.

Digital Twin validates each event and updates its internal data, making that information available to support real-time transaction authorization.

```
Core Banking System / Upstream System  ──►  Kafka (legacy-adapter.event.*)  ──►  DTW Registry
```

Events do not generate a response. Events are processed asynchronously — after sending an event, the publishing system does not wait for an immediate response from Digital Twin.

---

## Prerequisites

Before sending events, make sure the following are in place:

| Requirement | Details | Why it's needed |
|---|---|---|
| Kafka cluster access | Connection credentials and broker addresses provided by the DTW team | Your system needs a network path to send events to Digital Twin's topics |
| Confluent Schema Registry access | URL and credentials for schema resolution | Digital Twin validates every event against a registered Avro schema before accepting it |
| Apache Avro message format | Your integration must support Apache Avro message serialization | Digital Twin only accepts events encoded in Avro — events in other formats are rejected |
| Required Digital Twin event libraries | Maven dependencies `dtw-account-events`, `dtw-configuration-events`, `dtw-common-events` | These libraries define the exact event structures Digital Twin expects, so you don't have to build the schemas by hand |
| Environment identifier | The `[ENV]` value used in topic names (for example, `dev`, `uat`, or `prod`) | Digital Twin uses this to route your events to the correct environment's topics |
| Retry topic monitoring | Alerting configured on the depth of your event retry topics (see [What Happens When An Event Fails?](#what-happens-when-an-event-fails)) | **A stuck event blocks every event queued behind it on the same partition — proactive monitoring is the only way to detect this before it affects your integration**|

---

## When to Send Events

Whenever account or reference data changes in an upstream system, an event is published to the appropriate Digital Twin topic. Digital Twin validates the event and updates its internal data, making the information available to support real-time transaction authorization.

| Kafka Topic | Events Included | Digital Twin Action |
|---|---|---|
| `legacy-adapter.event.account` | Account lifecycle, feature, and membership events | Validates the event and updates its data, then serves it downstream |
| `legacy-adapter.event.account-type` | Account type configuration events | Validates the event and updates its data, then serves it downstream |
| `legacy-adapter.event.history-code` | History code configuration events | Validates the event and updates its data, then serves it downstream |

---

## In What Order to Send Events

Some events depend on others being sent first. Sending events out of order causes the dependent event to fail or be silently discarded — see [Event Validation and Error Handling](#event-validation-and-error-handling) for the specific behavior of each event.

```
AccountTypeChangeEvent          ──must precede──► AccountChangeEvent
                                                  (AccountChangeEvent references accountType.key)

AccountTypeChangeEvent          ──must precede──► AccountTypeFeatureChangeEvent

HistoryCodeChangeEvent (target) ──must precede──► HistoryCodeChangeEvent (with reversalHistoryCode pointing to it)

AccountChangeEvent              ──must precede──► AccountHolderChangeEvent

AccountChangeEvent              ──must precede──► AccountFeatureChangeEvent
                                                  (account must exist before features can be attached)
```

---

## Kafka Topic Naming Convention

Now that you know *when* and in what order events go out, here's *where* they go. All topics follow this naming convention:

```
[ENV].dtw.<source-module>.event.<subject>
```

| Component | Description |
|---|---|
| `[ENV]` | Environment prefix (e.g., `dev`, `uat`, `prod`) — provided by the DTW team |
| `<source-module>` | `legacy-adapter` for the account/account-type/history-code topics (these are consumed by the DTW Registry service) |
| `<subject>` | Event category: `account`, `account-type`, `history-code`, `generic-domain`, `system-configuration` |

**Topics you publish to:**

| Topic | Content |
|---|---|
| `[ENV].dtw.legacy-adapter.event.account` | Account lifecycle events, Account feature events, and Account Membership events |
| `[ENV].dtw.legacy-adapter.event.account-type` | Account type configuration events |
| `[ENV].dtw.legacy-adapter.event.history-code` | History code configuration events |
| `[ENV].dtw.legacy-adapter.event.generic-domain` | Generic Domain configuration events, including **Hold Reason** — see [Generic Domain Events](#generic-domain-events) |
| `[ENV].dtw.legacy-adapter.event.system-configuration` | System-wide configuration events (feature flags and limits) — see [Event Validation and Error Handling](#event-validation-and-error-handling) |

**Topic reference table:**

| Event category | Events | Publish to |
|---|---|---|
| Account & Membership | `AccountChangeEvent`, `AccountDeleteEvent`, `AccountStatusChangeEvent`, `AccountFeatureChangeEvent`, `AccountFeatureDeleteEvent`, `AccountHolderChangeEvent`, `AccountHolderDeleteEvent`, `AccountMembershipFeatureChangeEvent`, `AccountMembershipFeatureDeleteEvent` | `[ENV].dtw.legacy-adapter.event.account` |
| Account Type | `AccountTypeChangeEvent`, `AccountTypeDeleteEvent`, `AccountTypeFeatureChangeEvent`, `AccountTypeFeatureDeleteEvent` | `[ENV].dtw.legacy-adapter.event.account-type` |
| History Code | `HistoryCodeChangeEvent`, `HistoryCodeDeleteEvent`, `HistoryCodeFeatureChangeEvent`, `HistoryCodeFeatureDeleteEvent` | `[ENV].dtw.legacy-adapter.event.history-code` |
| Generic Domain (incl. Hold Reason) | `GenericDomainChangeEvent`, `GenericDomainDeleteEvent`, `GenericDomainFeatureChangeEvent`, `GenericDomainFeatureDeleteEvent` | `[ENV].dtw.legacy-adapter.event.generic-domain` |
| System Configuration | `SystemConfigurationFeatureChangeEvent`, `SystemConfigurationFeatureDeleteEvent`, `SystemConfigurationLimitChangeEvent`, `SystemConfigurationLimitDeleteEvent` | `[ENV].dtw.legacy-adapter.event.system-configuration` |

---

## What's Inside Every Event

Every event sent to Digital Twin includes a standard header called `BaseEvent`. This header identifies where the event came from, when it was created, and what type of event it contains. Every event in the catalog below starts with it, so it's worth understanding once, up front.

### Conceptual Overview

Think of `BaseEvent` as the shipping label on a package — it tells Digital Twin what the message is, where it came from, and when it was created. It also records *what earlier event triggered it* (lineage), if any. The actual payload — the package's contents — is made up of the specific event fields that travel alongside it.

### `BaseEvent`

```json
{
  "content": {
    "source": "checking-account-system",
    "eventId": "550e8400-e29b-41d4-a716-446655440000",
    "createdAtUtc": 1719360000000,
    "kind": "account-changed-event"
  },
  "parents": []
}
```

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `content` | `BaseEventContent` | Yes | — | Core event metadata |
| `parents` | `BaseEventContent[]` | No | — | Lineage — the chain of events that originated this one. Oldest first. Defaults to empty. |

### `BaseEventContent`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | no limit | Identifies the originating system (e.g., `"checking-account-system"`). Use a stable, lowercase, hyphenated identifier. |
| `eventId` | `string (uuid)` | Yes | 36 chars | Unique event ID in the originating system. Used for idempotency. Must be a valid UUID v4. |
| `createdAtUtc` | `long (local-timestamp-millis)` | Yes | — | Timestamp of when the event was generated, in **UTC milliseconds** since epoch. |
| `kind` | `string` | Yes | no limit | Descriptor of the event type. Convention: `{aggregate}-{past-tense-action}-event`, e.g., `"account-changed-event"`, `"account-type-excluded-event"`. |

> **Note:** `source` and `kind` are not persisted to Digital Twin's database — they exist only in the event itself, used for identification and logging. That's why no size limit applies to them, unlike the business fields covered elsewhere in this guide, which are constrained by the database column they're eventually stored in.

### `AccountKey`

Identifies an account uniquely within DTW. Every event below that acts on an account carries one of these.

```json
{
  "branch": 1,
  "account": 123456789
}
```

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `branch` | `int` | Yes | — | Rounting number |
| `account` | `long` | Yes | — | Account number |

> **Important:** Account keys must be positive. DTW explicitly filters out non-positive keys (they are reserved for system/sentinel accounts).

> note: FI Branch numbers can be included elsewhere.

---

## Account Type Events

**Topic:** `[ENV].dtw.legacy-adapter.event.account-type`

**Direction:** Core Banking / Upstream System → DTW Registry

Account types are configuration records that classify accounts (e.g., checking, savings, credit). They must exist in DTW before any account can reference them via `AccountChangeEvent.accountType`.

---

### AccountTypeChangeEvent

#### Conceptual Overview

Creates or updates an account type definition. Like `AccountChangeEvent`, this event handles both creation and updates — DTW uses upsert semantics. Publish this event before publishing `AccountChangeEvent` events that reference this account type.

> **Order matters:** DTW must know about an account type before processing `AccountChangeEvent` events that reference it. Publish account type events first, or ensure your publishing order guarantees this.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440010",
      "createdAtUtc": 1719360000000,
      "kind": "account-type-changed-event"
    },
    "parents": []
  },
  "accountType": {
    "key": "CHECKING"
  },
  "description": "Standard demand deposit checking account",
  "categoryType": "DEMAND_DEPOSIT"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `accountType` | `AccountTypeKey` | Yes | — | Unique identifier for this account type |
| `accountType.key` | `string` | Yes | max 15 chars | The type identifier referenced by `AccountChangeEvent.accountType` |
| `description` | `string` | Yes | max 100 chars | Human-readable description. Free-text field. **Must not contain personal data** (not subject to LGPD/GDPR protections) |
| `categoryType` | `string` | Yes | max 64 chars | Category classification of the account type (e.g., `"DEMAND_DEPOSIT"`, `"SAVINGS"`) |

#### Avro Record Name

`com.matera.dtw.event.configuration.AccountTypeChangeEvent`

---

### AccountTypeDeleteEvent

#### Conceptual Overview

Removes an account type from DTW's registry. After this event, any `AccountChangeEvent` referencing the deleted type's key will fail validation.

> Ensure no active accounts reference this account type before publishing a delete event.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440011",
      "createdAtUtc": 1719360000000,
      "kind": "account-type-excluded-event"
    },
    "parents": []
  },
  "accountType": {
    "key": "CHECKING"
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `accountType` | `AccountTypeKey` | Yes | — | The account type to be deleted |

#### Avro Record Name

`com.matera.dtw.event.configuration.AccountTypeDeleteEvent`

---

### AccountTypeFeatureChangeEvent

#### Conceptual Overview

Features let you attach optional configuration values to an account type without changing its core structure.

Adds or updates a **feature flag** on an account type. Features are key-value pairs that configure optional behavior for all accounts of this type. DTW uses features to enable platform-specific capabilities.


#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440012",
      "createdAtUtc": 1719360000000,
      "kind": "account-type-feature-changed-event"
    },
    "parents": []
  },
  "accountType": {
    "key": "CHECKING"
  },
  "feature": {
    "name": "CATEGORY",
    "value": {
      "type": "STRING",
      "booleanValue": null,
      "stringValue": "TRANSACTIONAL",
      "decimalValue": null,
      "dateValue": null,
      "dateRangeValue": null
    }
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `accountType` | `AccountTypeKey` | Yes | — | The account type receiving the feature |
| `feature.name` | `string` | Yes | max 64 chars | Feature identifier (e.g., `"CATEGORY"`) |
| `feature.value.type` | `enum` | Yes | — | Declares which typed field carries the value: `BOOLEAN`, `STRING`, `DECIMAL`, `DATE`, `DATE_RANGE` |
| `feature.value.booleanValue` | `boolean \| null` | Conditional | — | Populated when `type = BOOLEAN`. All other value fields must be `null`. |
| `feature.value.stringValue` | `string \| null` | Conditional | max 1024 chars | Populated when `type = STRING`. |
| `feature.value.decimalValue` | `bytes (decimal, precision 19) \| null` | Conditional | precision 19 | Populated when `type = DECIMAL`. |
| `feature.value.dateValue` | `int (date) \| null` | Conditional | — | Populated when `type = DATE`. Days since epoch. |
| `feature.value.dateRangeValue` | `DateRange \| null` | Conditional | — | Populated when `type = DATE_RANGE`. Contains `startDate` and `endDate` as millisecond timestamps. |

> **Rule:** Set `type` to the matching enum value and populate **only** the corresponding typed field. Leave all other value fields as `null`.

#### Avro Record Name

`com.matera.dtw.event.configuration.AccountTypeFeatureChangeEvent`

---

### AccountTypeFeatureDeleteEvent

#### Conceptual Overview

Removes a feature from an account type. After this event, the feature is no longer active for that account type and will no longer be applied to accounts of that type.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440013",
      "createdAtUtc": 1719360000000,
      "kind": "account-type-feature-excluded-event"
    },
    "parents": []
  },
  "accountType": {
    "key": "CHECKING"
  },
  "featureName": "CATEGORY"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `accountType` | `AccountTypeKey` | Yes | — | The account type losing the feature |
| `featureName` | `string` | Yes | max 64 chars | Name of the feature to remove (e.g., `"CATEGORY"`) |

#### Avro Record Name

`com.matera.dtw.event.configuration.AccountTypeFeatureDeleteEvent`

---

## Account Events

**Topic:** `[ENV].dtw.legacy-adapter.event.account`

**Direction:** Core Banking / Upstream System → DTW Registry

These events represent changes to the lifecycle and features of individual accounts. Publish one of these events whenever an account is created, updated, deleted, has its status changed, or has account-level features added, updated, or removed.

---

### AccountChangeEvent

#### Conceptual Overview

`AccountChangeEvent` is the workhorse of account integration. It handles both **account creation and account updates** with a single event type — DTW applies upsert semantics. If the account does not exist, it is created. If it does, it is updated.

> Publish this event whenever: an account is opened, an account's type changes, an account's timezone changes, or the global active flag changes (other than a status-only change, which uses `AccountStatusChangeEvent`).

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440001",
      "createdAtUtc": 1719360000000,
      "kind": "account-changed-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "accountType": "CHECKING",
  "timeZone": "America/Chicago",
  "active": true
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header — see [What's Inside Every Event](#whats-inside-every-event) |
| `account` | `AccountKey` | Yes | — | The account being created or updated |
| `accountType` | `string` | Yes | max 15 chars | The unique identifier of the account type. Must match a previously published `AccountTypeChangeEvent.accountType.key` |
| `timeZone` | `string` | Yes | max 30 chars | Account timezone using IANA canonical name (e.g., `"America/Chicago"`, `"America/New_York"`, `"UTC"`) |
| `active` | `boolean` | Yes | — | Global account status. `true` = active, `false` = inactive |

#### Avro Record Name

`com.matera.dtw.event.account.AccountChangeEvent`

---

### AccountDeleteEvent

#### Conceptual Overview

Signals that an account has been **permanently and irreversibly deleted** (a "hard delete") from the core banking system or other upstream system, and should be removed from the DTW registry entirely — as opposed to closing or deactivating an account, which keeps the account record but marks it inactive. Use this event only for permanent deletion; if you are closing or deactivating an account, use `AccountStatusChangeEvent` instead.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440002",
      "createdAtUtc": 1719360000000,
      "kind": "account-excluded-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account to be deleted |

#### Avro Record Name

`com.matera.dtw.event.account.AccountDeleteEvent`

---

### AccountStatusChangeEvent

#### Conceptual Overview

Changes the **active/inactive status** of an account without modifying any other account data. This is a targeted event — use it instead of re-sending a full `AccountChangeEvent` when only the status changes.

DTW models account status as a **binary value**: `true` (active) or `false` (inactive). There is no intermediate or richer lifecycle state at the event level.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440004",
      "createdAtUtc": 1719360000000,
      "kind": "account-status-changed-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "active": false
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account whose status is changing |
| `active` | `boolean` | Yes | — | New status: `true` = active, `false` = inactive |

#### Avro Record Name

`com.matera.dtw.event.account.status.AccountStatusChangeEvent`

---

### AccountFeatureChangeEvent

#### Conceptual Overview

Features allow banks to attach optional configuration values to accounts without changing the core account structure.

Adds or updates a single feature for an account. Each event contains exactly one `FeatureEntry`. If the feature already exists, Digital Twin updates its value. If the feature does not exist, Digital Twin creates it.

> Publish this event whenever an account-level flag or attribute changes (e.g., blocking/unblocking credits or debits).

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440040",
      "createdAtUtc": 1719360000000,
      "kind": "account-feature-changed-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "feature": {
    "name": "BLOCKED_FOR_CREDIT",
    "value": {
      "type": "BOOLEAN",
      "booleanValue": true,
      "stringValue": null,
      "decimalValue": null,
      "dateValue": null,
      "dateRangeValue": null
    }
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account receiving the feature |
| `feature.name` | `string` | Yes | max 64 chars | Feature identifier — see Known Account Features below |
| `feature.value.type` | `enum` | Yes | — | Declares which typed field carries the value: `BOOLEAN`, `STRING`, `DECIMAL`, `DATE`, `DATE_RANGE` |
| `feature.value.booleanValue` | `boolean \| null` | Conditional | — | Populated when `type = BOOLEAN`. All other value fields must be `null`. |
| `feature.value.stringValue` | `string \| null` | Conditional | max 1024 chars | Populated when `type = STRING`. |
| `feature.value.decimalValue` | `bytes (decimal, precision 19) \| null` | Conditional | precision 19 | Populated when `type = DECIMAL`. |
| `feature.value.dateValue` | `int (date) \| null` | Conditional | — | Populated when `type = DATE`. Days since epoch. |
| `feature.value.dateRangeValue` | `DateRange \| null` | Conditional | — | Populated when `type = DATE_RANGE`. Contains `startDate` and `endDate` as millisecond timestamps. |

> **Rule:** Set `type` to the matching enum value and populate **only** the corresponding typed field. Leave all other value fields as `null`.

#### Avro Record Name

`com.matera.dtw.event.account.feature.AccountFeatureChangeEvent`

#### Known Account Features

| Feature name | Type | Description |
|---|---|---|
| `BLOCKED_FOR_CREDIT` | `BOOLEAN` | Blocks the account from receiving credits |
| `BLOCKED_FOR_DEBIT` | `BOOLEAN` | Blocks the account from generating debits |
| `EMPLOYER_TAX_ID` | `STRING` | Employer tax ID, used for salary account validation |
| `UNCONDITIONAL_INTEREST_CHARGE` | `BOOLEAN` | Indicates unconditional interest charge is active on the account |
| `ONGOING_MIGRATION` | `STRING` | Migration mode marker. Value must be one of `REGISTRATION_DATA` or `FINANCIAL_OPEN` — this constraint is enforced in code, not by the database |

> To unblock an account, send `AccountFeatureDeleteEvent` with `featureName: "BLOCKED_FOR_CREDIT"` (or `BLOCKED_FOR_DEBIT`).

---

### AccountFeatureDeleteEvent

#### Conceptual Overview

Removes a single account-level feature. The account itself remains unchanged — only the specified feature is deleted.


#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440041",
      "createdAtUtc": 1719360000000,
      "kind": "account-feature-excluded-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "featureName": "BLOCKED_FOR_CREDIT"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account losing the feature |
| `featureName` | `string` | Yes | max 64 chars | Name of the feature to remove (e.g., `"BLOCKED_FOR_CREDIT"`) |

#### Avro Record Name

`com.matera.dtw.event.account.feature.AccountFeatureDeleteEvent`

---

## Account Membership Events

**Topic:** `[ENV].dtw.legacy-adapter.event.account`

**Direction:** Core Banking / Upstream System → DTW Registry

Account membership events associate parties (persons or organizations) with accounts and manage metadata attached to those associations. They share the same Kafka topic as account lifecycle events.

There are two operation levels:

| Level | What it affects | Events |
|---|---|---|
| **Holder** | The membership link between a party and an account | `AccountHolderChangeEvent` / `AccountHolderDeleteEvent` |
| **Membership Feature** | Key-value metadata attached to an existing membership | `AccountMembershipFeatureChangeEvent` / `AccountMembershipFeatureDeleteEvent` |

> **Prerequisite:** A holder must exist on an account (via `AccountHolderChangeEvent`) before you can attach or remove features for that membership.

---

### AccountHolderChangeEvent

#### Conceptual Overview

Creates or updates the relationship between an account and an account holder. If the relationship already exists, Digital Twin updates it. If it does not exist, Digital Twin creates it.

The kind field is a user-defined label that identifies the type of account holder relationship (for example, PERSON_HOLDER_TAX_ID or COMPANY_HOLDER). Digital Twin stores this value but does not interpret it. Your organization defines the values and their meaning.


> Publish this event when a party is linked to an account for the first time, or when the holder record needs to be refreshed.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440030",
      "createdAtUtc": 1719360000000,
      "kind": "account-holder-changed-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "accountHolder": {
    "kind": "PERSON_HOLDER_TAX_ID",
    "clientId": "a1b2c3d4e5f6"
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header — see [What's Inside Every Event](#whats-inside-every-event) |
| `account` | `AccountKey` | Yes | — | The account receiving the holder |
| `accountHolder.kind` | `string` | Yes | max 255 chars | Membership category label. Convention is free-form; use stable, uppercase, hyphen-free identifiers (e.g., `"PERSON_HOLDER_TAX_ID"`) |
| `accountHolder.clientId` | `string` | Yes | max 64 chars | Cryptographic hash that identifies the client — replaces the raw tax ID. Must be stable across events for the same party. **Hash algorithms producing more than 64 characters (e.g. hex-encoded SHA-512) will not fit — use a hash truncated/encoded to 64 chars or fewer.** |

#### Avro Record Name

`com.matera.dtw.event.account.holder.AccountHolderChangeEvent`

---

### AccountHolderDeleteEvent

#### Conceptual Overview

Removes a **membership link** entirely from an account. After this event, the party is no longer associated with the account and any features attached to that membership are also removed.

> Use this event only when the party-to-account relationship must be severed. To remove individual features while keeping the membership active, use `AccountMembershipFeatureDeleteEvent`.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440031",
      "createdAtUtc": 1719360000000,
      "kind": "account-holder-excluded-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "accountHolder": {
    "kind": "PERSON_HOLDER_TAX_ID",
    "clientId": "a1b2c3d4e5f6"
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account losing the holder |
| `accountHolder.kind` | `string` | Yes | max 255 chars | Must exactly match the `kind` used when the membership was created |
| `accountHolder.clientId` | `string` | Yes | max 64 chars | Must exactly match the `clientId` used when the membership was created |

#### Avro Record Name

`com.matera.dtw.event.account.holder.AccountHolderDeleteEvent`

---

### AccountMembershipFeatureChangeEvent

#### Conceptual Overview

Features let you attach optional metadata to a membership without changing its core structure.

Adds or updates **features** (key-value metadata) on an **existing** membership. Unlike the holder events that take a single holder reference, this event carries an **array** of `FeatureEntry` objects — you can update multiple features in one event.

DTW applies upsert semantics per feature name on the identified membership. If a feature with the same `name` already exists, its value is replaced; otherwise it is added.

> The membership identified by `(membership.kind, membership.clientId)` must already exist on the account before features can be attached. Publish `AccountHolderChangeEvent` first.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440032",
      "createdAtUtc": 1719360000000,
      "kind": "account-membership-feature-changed-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "membership": {
    "kind": "PERSON_HOLDER_TAX_ID",
    "clientId": "a1b2c3d4e5f6"
  },
  "features": [
    {
      "name": "RISK_SCORE",
      "value": {
        "type": "STRING",
        "booleanValue": null,
        "stringValue": "A",
        "decimalValue": null,
        "dateValue": null,
        "dateRangeValue": null
      }
    }
  ]
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account that owns the membership |
| `membership.kind` | `string` | Yes | max 255 chars | Must match the `kind` of the target membership |
| `membership.clientId` | `string` | Yes | max 64 chars | Must match the `clientId` of the target membership |
| `features` | `FeatureEntry[]` | Yes | — | One or more features to add or update. Must not be empty |
| `features[].name` | `string` | Yes | max 64 chars | Feature identifier (e.g., `"RISK_SCORE"`) |
| `features[].value.type` | `enum` | Yes | — | Declares which typed field carries the value: `BOOLEAN`, `STRING`, `DECIMAL`, `DATE`, `DATE_RANGE` |
| `features[].value.booleanValue` | `boolean \| null` | Conditional | — | Populated when `type = BOOLEAN`. All other value fields must be `null`. |
| `features[].value.stringValue` | `string \| null` | Conditional | max 1024 chars | Populated when `type = STRING`. |
| `features[].value.decimalValue` | `bytes (decimal, precision 19) \| null` | Conditional | precision 19 | Populated when `type = DECIMAL`. |
| `features[].value.dateValue` | `int (date) \| null` | Conditional | — | Populated when `type = DATE`. Days since epoch. |
| `features[].value.dateRangeValue` | `DateRange \| null` | Conditional | — | Populated when `type = DATE_RANGE`. Contains `startDate` and `endDate` as millisecond timestamps. |

> **Field naming difference:** The holder-level events use `accountHolder` to identify the party. This event uses `membership` — both fields have the same structure (`kind` + `clientId`), but the name differs.

> **Rule:** Set `type` to the matching enum value and populate **only** the corresponding typed field. Leave all other value fields as `null`.

#### Avro Record Name

`com.matera.dtw.event.account.membership.AccountMembershipFeatureChangeEvent`

---

### AccountMembershipFeatureDeleteEvent

#### Conceptual Overview

Removes one or more **named features** from an existing membership. The membership itself remains — only the specified features are deleted. To remove the entire membership, use `AccountHolderDeleteEvent`.

The `featureNames` field accepts an array, so multiple features can be removed atomically.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440033",
      "createdAtUtc": 1719360000000,
      "kind": "account-membership-feature-excluded-event"
    },
    "parents": []
  },
  "account": {
    "branch": 1,
    "account": 987654321
  },
  "membership": {
    "kind": "PERSON_HOLDER_TAX_ID",
    "clientId": "a1b2c3d4e5f6"
  },
  "featureNames": [
    "RISK_SCORE"
  ]
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `account` | `AccountKey` | Yes | — | The account that owns the membership |
| `membership.kind` | `string` | Yes | max 255 chars | Must match the `kind` of the target membership |
| `membership.clientId` | `string` | Yes | max 64 chars | Must match the `clientId` of the target membership |
| `featureNames` | `string[]` | Yes | — | Names of the features to remove. Must not be empty |

> **Note:** Unlike `AccountMembershipFeatureChangeEvent` which takes full `FeatureEntry` objects, this event takes only **feature names** (strings) — there is no value in a delete operation.

#### Avro Record Name

`com.matera.dtw.event.account.membership.AccountMembershipFeatureDeleteEvent`

---

### Known Membership Features

The following feature names are defined in `AccountMembershipFeature` (`dtw-common-events`):

| Feature name | Type | Description |
|---|---|---|
| `NAME` | `STRING` | Display name of the membership holder |

---

## History Code Events

**Topic:** `[ENV].dtw.legacy-adapter.event.history-code`

**Direction:** Core Banking / Upstream System → DTW Registry

History codes (also known as transaction codes) classify ledger entries by type. They define whether an operation is a debit, credit, or balance adjustment, and optionally link to a reversal code.

History codes must exist in DTW before any transaction using them is authorized.

---

### HistoryCodeChangeEvent

#### Conceptual Overview

Creates or updates a history code. A history code defines the nature of a ledger entry: whether it moves money out (`DEBIT`), in (`CREDIT`), or adjusts the balance directly (`BALANCE`).

Each code can optionally reference a **reversal code** — another history code that compensates it. For example, if `ACH_DEBIT` is a debit code, its reversal might be `ACH_DEBIT_REVERSAL` (a credit code that undoes it).

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440020",
      "createdAtUtc": 1719360000000,
      "kind": "history-code-changed-event"
    },
    "parents": []
  },
  "historyCodeKey": {
    "historyCodeId": "ACH_DEBIT"
  },
  "historyCodeType": "DEBIT",
  "description": "ACH outbound transfer debit",
  "reversalHistoryCode": {
    "historyCodeId": "ACH_DEBIT_REVERSAL"
  }
}
```

#### Payload — without reversal

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440021",
      "createdAtUtc": 1719360000000,
      "kind": "history-code-changed-event"
    },
    "parents": []
  },
  "historyCodeKey": {
    "historyCodeId": "INTEREST_CREDIT"
  },
  "historyCodeType": "CREDIT",
  "description": "Interest income credit",
  "reversalHistoryCode": null
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `historyCodeKey` | `HistoryCodeKey` | Yes | — | Unique identifier for this history code |
| `historyCodeKey.historyCodeId` | `string` | Yes | max 64 chars | The history code identifier (e.g., `"ACH_DEBIT"`) |
| `historyCodeType` | `enum` | Yes | — | Entry type: `DEBIT`, `CREDIT`, or `BALANCE` |
| `description` | `string` | Yes | max 50 chars | Human-readable description. Free-text. **Must not contain personal data** |
| `reversalHistoryCode` | `HistoryCodeKey \| null` | No | — | The history code that reverses this one. `null` if this code has no reversal |

#### `historyCodeType` Values

| Value | Meaning |
|---|---|
| `DEBIT` | Entry moves value out of the account (reduces balance) |
| `CREDIT` | Entry moves value into the account (increases balance) |
| `BALANCE` | Entry directly adjusts balance without a directional debit/credit |

#### Avro Record Name

`com.matera.dtw.event.configuration.HistoryCodeChangeEvent`

---

### HistoryCodeDeleteEvent

#### Conceptual Overview

Removes a history code from DTW's registry. After this event, any transaction referencing the deleted code will fail authorization.

> Ensure no active transaction flows use this history code before deleting it.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440022",
      "createdAtUtc": 1719360000000,
      "kind": "history-code-excluded-event"
    },
    "parents": []
  },
  "historyCodeKey": {
    "historyCodeId": "ACH_DEBIT"
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `historyCodeKey` | `HistoryCodeKey` | Yes | — | The history code to be deleted |

#### Avro Record Name

`com.matera.dtw.event.configuration.HistoryCodeDeleteEvent`

---

### HistoryCodeFeatureChangeEvent

#### Conceptual Overview

Features let you attach optional configuration values to a history code without changing its core structure.

Adds or updates a feature on a history code. Features on history codes configure optional behavior for entries processed under that code.

**Known features for history codes:**

| Feature Name | Type | Description |
|---|---|---|
| `DOCUMENT_TYPE` | `STRING` | Document type classification associated with this history code |

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440023",
      "createdAtUtc": 1719360000000,
      "kind": "history-code-feature-changed-event"
    },
    "parents": []
  },
  "historyCodeKey": {
    "historyCodeId": "ACH_DEBIT"
  },
  "feature": {
    "name": "DOCUMENT_TYPE",
    "value": {
      "type": "STRING",
      "booleanValue": null,
      "stringValue": "TED",
      "decimalValue": null,
      "dateValue": null,
      "dateRangeValue": null
    }
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `historyCodeKey` | `HistoryCodeKey` | Yes | — | The history code receiving the feature |
| `feature.name` | `string` | Yes | max 64 chars | Feature identifier (e.g., `"DOCUMENT_TYPE"`) |
| `feature.value.type` | `enum` | Yes | — | Declares which typed field carries the value: `BOOLEAN`, `STRING`, `DECIMAL`, `DATE`, `DATE_RANGE` |
| `feature.value.booleanValue` | `boolean \| null` | Conditional | — | Populated when `type = BOOLEAN` |
| `feature.value.stringValue` | `string \| null` | Conditional | max 1024 chars | Populated when `type = STRING` |
| `feature.value.decimalValue` | `bytes (decimal, precision 19) \| null` | Conditional | precision 19 | Populated when `type = DECIMAL` |
| `feature.value.dateValue` | `int (date) \| null` | Conditional | — | Populated when `type = DATE`. Days since epoch. |
| `feature.value.dateRangeValue` | `DateRange \| null` | Conditional | — | Populated when `type = DATE_RANGE`. Contains `startDate` and `endDate` as millisecond timestamps. |

> **Rule:** Set `type` to the matching enum value and populate **only** the corresponding typed field. Leave all other value fields as `null`.

#### Avro Record Name

`com.matera.dtw.event.configuration.HistoryCodeFeatureChangeEvent`

---

### HistoryCodeFeatureDeleteEvent

#### Conceptual Overview

Removes a feature from a history code. Use this to retract a feature previously set via `HistoryCodeFeatureChangeEvent`.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440024",
      "createdAtUtc": 1719360000000,
      "kind": "history-code-feature-excluded-event"
    },
    "parents": []
  },
  "historyCodeKey": {
    "historyCodeId": "ACH_DEBIT"
  },
  "featureName": "DOCUMENT_TYPE"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `historyCodeKey` | `HistoryCodeKey` | Yes | — | The history code losing the feature |
| `featureName` | `string` | Yes | max 64 chars | Name of the feature to remove (e.g., `"DOCUMENT_TYPE"`) |

#### Avro Record Name

`com.matera.dtw.event.configuration.HistoryCodeFeatureDeleteEvent`

---

## Generic Domain Events

**Topic:** `[ENV].dtw.legacy-adapter.event.generic-domain`

**Direction:** Core Banking / Upstream System → DTW Registry

Generic Domain events configure a generic `(domainType, itemId)` record in DTW's registry — a mechanism shared by several configuration domains that don't warrant their own dedicated topic. **Hold Reason** is published through this mechanism: to configure a hold reason, publish a Generic Domain event with `domainTypeId.key` set to the literal value `"BALANCE_HOLD_REASON"`.

> **Where this comes from:** Unlike the rest of this guide's illustrative examples, the identifiers below are taken directly from source — the `GenericDomainChangeEvent`/`GenericDomainDeleteEvent`/`GenericDomainFeatureChangeEvent`/`GenericDomainFeatureDeleteEvent` Avro records (`dtw-configuration-events`), the `BALANCE_HOLD_REASON` constant and its `ALLOW_OVERDRAFT_LIMIT_USAGE`/`ALLOW_PARTIAL_UNHOLD` feature keys (`GenericDomainConstants`, `dtw-common-events`), and the automated consumer tests and seed fixtures for Hold Reason in `dtw-registry`'s and `dtw-transaction`'s `config-event-consumer` modules. Item IDs and descriptions in the examples are illustrative; the field names, record names, and the `"BALANCE_HOLD_REASON"` value are not.

---

### GenericDomainChangeEvent

#### Conceptual Overview

Creates or updates a generic domain record — DTW uses upsert semantics, same as `AccountTypeChangeEvent`. For a Hold Reason, `itemId` is the hold reason's natural key and `itemDescription` is its human-readable description.

#### Payload — Hold Reason example

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440050",
      "createdAtUtc": 1719360000000,
      "kind": "generic-domain-changed-event"
    },
    "parents": []
  },
  "domainTypeId": {
    "key": "BALANCE_HOLD_REASON"
  },
  "itemId": "56",
  "itemDescription": "Judicial hold"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `domainTypeId` | `GenericDomainKey` | Yes | — | The generic domain type. For Hold Reason, always `{"key": "BALANCE_HOLD_REASON"}` |
| `domainTypeId.key` | `string` | Yes | max 255 chars | The domain type identifier |
| `itemId` | `string` | Yes | max 255 chars | Natural key of the item within that domain type (for Hold Reason, the hold reason's key) |
| `itemDescription` | `string` | Yes | max 255 chars | Human-readable description of the item |

#### Avro Record Name

`com.matera.dtw.event.configuration.GenericDomainChangeEvent`

---

### GenericDomainDeleteEvent

#### Conceptual Overview

Removes a generic domain record. For Hold Reason, this deletes the hold reason identified by `itemId`.

#### Payload — Hold Reason example

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440052",
      "createdAtUtc": 1719360000000,
      "kind": "generic-domain-excluded-event"
    },
    "parents": []
  },
  "domainTypeId": {
    "key": "BALANCE_HOLD_REASON"
  },
  "itemId": "56"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `domainTypeId` | `GenericDomainKey` | Yes | — | The generic domain type being deleted from |
| `itemId` | `string` | Yes | max 255 chars | Natural key of the item to delete |

#### Avro Record Name

`com.matera.dtw.event.configuration.GenericDomainDeleteEvent`

---

### GenericDomainFeatureChangeEvent

#### Conceptual Overview

Adds or updates a single feature on a generic domain item. For Hold Reason, this is how the hold's overdraft/partial-unhold behavior is configured.

#### Payload — Hold Reason example

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440053",
      "createdAtUtc": 1719360000000,
      "kind": "generic-domain-feature-changed-event"
    },
    "parents": []
  },
  "domainTypeId": {
    "key": "BALANCE_HOLD_REASON"
  },
  "itemId": "56",
  "feature": {
    "name": "ALLOW_OVERDRAFT_LIMIT_USAGE",
    "value": {
      "type": "BOOLEAN",
      "booleanValue": false,
      "stringValue": null,
      "decimalValue": null,
      "dateValue": null,
      "dateRangeValue": null
    }
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `domainTypeId` | `GenericDomainKey` | Yes | — | The generic domain type that owns the item |
| `itemId` | `string` | Yes | max 255 chars | Natural key of the item receiving the feature |
| `feature.name` | `string` | Yes | max 64 chars | Feature identifier — see Known Hold Reason Features below |
| `feature.value.type` | `enum` | Yes | — | Declares which typed field carries the value: `BOOLEAN`, `STRING`, `DECIMAL`, `DATE`, `DATE_RANGE` |
| `feature.value.booleanValue` | `boolean \| null` | Conditional | — | Populated when `type = BOOLEAN`. All other value fields must be `null`. |

> **Rule:** Set `type` to the matching enum value and populate **only** the corresponding typed field, same as every other feature event in this guide.

#### Avro Record Name

`com.matera.dtw.event.configuration.GenericDomainFeatureChangeEvent`

#### Known Hold Reason Features

| Feature name | Type | Description |
|---|---|---|
| `ALLOW_OVERDRAFT_LIMIT_USAGE` | `BOOLEAN` | Whether a hold placed under this reason is allowed to use the account's overdraft limit |
| `ALLOW_PARTIAL_UNHOLD` | `BOOLEAN` | Whether a hold placed under this reason can be partially released |

---

### GenericDomainFeatureDeleteEvent

#### Conceptual Overview

Removes a single feature from a generic domain item. For Hold Reason, this retracts a previously set `ALLOW_OVERDRAFT_LIMIT_USAGE` or `ALLOW_PARTIAL_UNHOLD` flag.

#### Payload — Hold Reason example

```json
{
  "baseEvent": {
    "content": {
      "source": "core-banking-system",
      "eventId": "550e8400-e29b-41d4-a716-446655440054",
      "createdAtUtc": 1719360000000,
      "kind": "generic-domain-feature-excluded-event"
    },
    "parents": []
  },
  "domainTypeId": {
    "key": "BALANCE_HOLD_REASON"
  },
  "itemId": "56",
  "featureName": "ALLOW_OVERDRAFT_LIMIT_USAGE"
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Event header |
| `domainTypeId` | `GenericDomainKey` | Yes | — | The generic domain type that owns the item |
| `itemId` | `string` | Yes | max 255 chars | Natural key of the item losing the feature |
| `featureName` | `string` | Yes | max 64 chars | Name of the feature to remove (e.g., `"ALLOW_OVERDRAFT_LIMIT_USAGE"`) |

#### Avro Record Name

`com.matera.dtw.event.configuration.GenericDomainFeatureDeleteEvent`

---

## Event Validation and Error Handling

This section describes how Digital Twin responds when events contain invalid data or arrive in the wrong sequence.

If Digital Twin encounters an unexpected processing error, the event is automatically routed to a **retry topic** for later processing. The information below describes how Digital Twin validates events and handles common data and sequencing issues before an event is retried.

> **Important:** Digital Twin does not use a Dead Letter Queue (DLQ). All processing failures are routed to a retry topic, regardless of whether the failure is temporary or permanent. See **What Happens When An Event Fails?** for additional details.

### Event Processing Outcomes

| Symbol | Meaning |
|---|---|
| 🔴 **RETRY** | An unexpected error occurs and the event is routed to the retry topic |
| 🟡 **WARNING** | Digital Twin detects the condition, logs a warning and processes the event |
| 🟢 **OK** | The event is accepted and processed successfully, with no warnings logged |

---

### What Happens When An Event Fails?

Digital Twin uses retry topics rather than a Dead Letter Queue to manage failed event processing. Internally, DTW routes a failed event by its **aggregate** (the data type it belongs to): a router reads the failure and republishes it to that aggregate's own dedicated retry topic, looked up from a routing table. An aggregate with no entry in that table is simply never retried — which is also why there isn't one shared retry topic for everything, but one per aggregate.

There are five retry topics — four for configuration and reference data (Account Types, History Codes, System Configuration, and Generic Domain) and one for Account events:

| Aggregate | Retry Topic | Carries failures from |
|---|---|---|
| Account | `[ENV].dtw.registry.retry.event.account` | `[ENV].dtw.legacy-adapter.event.account` (Account, Account Feature, and Account Membership events) |
| Account Type | `[ENV].dtw.registry.retry.event.config.account-type` | `[ENV].dtw.legacy-adapter.event.account-type` |
| History Code | `[ENV].dtw.registry.retry.event.config.history-code` | `[ENV].dtw.legacy-adapter.event.history-code` |
| System Configuration | `[ENV].dtw.registry.retry.event.config.system-configuration` | `[ENV].dtw.legacy-adapter.event.system-configuration` |
| Generic Domain (incl. Hold Reason) | `[ENV].dtw.registry.retry.event.config.generic-domain` | `[ENV].dtw.legacy-adapter.event.generic-domain` |

Hold Reason events do not have a dedicated retry topic. They are published to the **Generic Domain** topic and therefore follow the same retry behavior.

> **Naming pattern:** retry topics mirror the regular topic's subject, just swapping `legacy-adapter` for `registry` and inserting `retry` before `event` — e.g. `legacy-adapter.event.history-code` → `registry.retry.event.config.history-code`. Use this to set up monitoring/alerting per aggregate (see [Prerequisites](#prerequisites)).

Each retry topic contains two types of events:

- **Transient failures** – Events that are expected to succeed once the underlying issue is resolved (for example, a temporary service outage, or a prerequisite event — see [In What Order to Send Events](#in-what-order-to-send-events) — that hasn't landed yet). No action needed on your side.
- **Permanent failures** – Events that will continue to fail until the event data or system configuration is corrected (for example, a malformed `timeZone` — see the `AccountChangeEvent` example further down in this section). DTW has no mechanism to detect this and discard the event automatically; someone from the operations team has to manually **remove** the message from the retry topic.

Determining whether a failure is transient or permanent is essential because it determines the appropriate remediation.

> **The retry topic also preserves order: events are consumed sequentially per partition, so a stuck message — transient or permanent — blocks every event queued behind it on that same partition until it is either successfully retried or manually removed.**

Permanent errors should be rare in a well-implemented integration. Each event represents a change that has already occurred in your core banking system or another upstream system. The publishing component's responsibility is to translate that information into the event format expected by Digital Twin. By adhering to the documented event contracts—including field types, lengths, required fields, and formatting rules—most permanent errors can be prevented before an event is published.

In practice, these errors occur most often during integration and testing, while event mappings and publishing logic are being validated against the event contracts described in this guide. This is expected and is a normal part of the implementation process. Once the integration has been validated and deployed to production, permanent errors should be the exception rather than routine operation.

**How to avoid it:** Events published out of sequence are typically a transient condition and usually resolve once the missing prerequisite event is received, as described in [In What Order to Send Events](#in-what-order-to-send-events). Invalid event data—such as an incorrect format, missing required field, or value outside the permitted range—is a permanent condition and requires correction before the event can be successfully processed. Following the field definitions and validation rules in each event's **Field Reference** helps prevent these errors. Monitoring retry topics provides an early indication that events are failing and require investigation. See [Prerequisites](#prerequisites) for additional guidance.

---

### Account Events

#### `AccountChangeEvent`

If the account does not exist, Digital Twin creates it. If the account already exists, Digital Twin updates it. Although there are no additional business validation rules for this event, all required fields must contain valid values and be correctly formatted:

| Condition | Log | Outcome |
|---|---|---|
| Database constraint violation | None | 🔴 RETRY |
| **`timeZone` is not a valid IANA time zone ID** (e.g. `"Americas/Chicago"` instead of `"America/Chicago"`) | None | 🔴 RETRY |

---

#### `AccountDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| Account does not exist | WARNING: `"Received an event to delete account [{}]. However, it does not seem to exist."` | 🟢 OK |

---

#### `AccountStatusChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
| `active` already matches current status | WARNING: `"The event requested a status change for account [{}], but the account is already in status [{}]. Skipping event."` | 🟡 WARNING |

---

#### `AccountHolderChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
| **Account does not exist in the registry** | *(none — `NoSuchElementException` propagates)* | 🔴 RETRY |

> **Order constraint:** The account must already exist (via `AccountChangeEvent`) before this event is published.

---

#### `AccountHolderDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| Holder (`clientId`) not found | WARNING: `"Account holder [{}] not found. Nothing to delete."` | 🟡 WARNING |
| Membership link (account ↔ holder) not found | WARNING: `"Trying to delete the account holder {} from account {}, but it does not seems to exists. Skipping it"` | 🟡 WARNING |

---

#### `AccountMembershipFeatureChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
| Membership link not found | WARNING: `"No relationship found between account [{}] and account holder [{}]. Creating a new relationship."` | 🟡 WARNING |
| Account does not exist (during auto-creation) | None | 🔴 RETRY |

---

#### `AccountMembershipFeatureDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| No explicit validation. If the membership or feature does not exist, the operation succeeds silently  | None| 🟢 OK|


---

### Account Type Events

#### `AccountTypeChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
|Creates a new account type if it does not already exist; otherwise, updates the existing account type. No additional business validation rules apply | None| 🟢 OK|

---

#### `AccountTypeDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| Account type does not exist | WARNING: `"AccountType id [{}] not found to be deleted"` | 🟡 WARNING |
| **Active accounts still reference this type** | WARNING: `"The account type of [{}] cannot be deleted since it is related to at least an account."` | 🟡 WARNING|

> **Important:** The registry does **not** notify the publisher when a delete is blocked. The event is silently discarded — the publisher has no feedback.

---

#### `AccountTypeFeatureChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
| **Account type does not exist** | WARNING: `"Account type {} not found, so skipping event processing"` | 🟡 WARNING |

> **Order constraint:** `AccountTypeChangeEvent` must be published before any feature event for that type.

---

#### `AccountTypeFeatureDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| Account type does not exist | WARNING: `"Account type {} not found, so skipping event processing"` | 🟡 WARNING |

---

### History Code Events

#### `HistoryCodeChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
| **`reversalHistoryCode` references a non-existent code** | None | 🔴 RETRY |

> **Order constraint:** The reversal history code must already exist in the registry before publishing the event that references it.

---

#### `HistoryCodeDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
| History code does not exist | WARNING: `"History Code id [{}] not found to be deleted"` | 🟡 WARNING |

---

#### `HistoryCodeFeatureChangeEvent`

| Condition | Log | Outcome |
|---|---|---|
|No explicit validation. If the history code does not exist, a `NullPointerException` propagates | None | 🔴 RETRY |

---

#### `HistoryCodeFeatureDeleteEvent`

| Condition | Log | Outcome |
|---|---|---|
|No explicit validation. Silent success if the feature does not exist| None| 🟢 OK|


---

### Common Integration Errors

| Event | Failure Condition | Outcome |
|---|---|---|
| `AccountChangeEvent` | `timeZone` is not a valid IANA time zone ID (e.g. `"Americas/Chicago"`) | 🔴 RETRY |
| `AccountHolderChangeEvent` | Account does not exist | 🔴 RETRY |
| `AccountFeatureChangeEvent` | Account does not exist | 🔴 RETRY |
| `AccountMembershipFeatureChangeEvent` | Account not found during auto-create | 🔴 RETRY |
| `HistoryCodeChangeEvent` | `reversalHistoryCode` not found | 🔴 RETRY |
| `AccountTypeDeleteEvent` | Active accounts reference the type | 🟡 WARNING |
| `AccountTypeFeatureChangeEvent` | Account type not found | 🟡 WARNING |
| `AccountTypeFeatureDeleteEvent` | Account type not found | 🟡 WARNING |
| `AccountStatusChangeEvent` | Status already matches | 🟡 WARNING |
| `AccountDeleteEvent` | Account not found | 🟡 WARNING|
| `AccountHolderDeleteEvent` | Holder not found | 🟢 OK |
| `AccountHolderDeleteEvent` | Membership link not found | 🟢 OK |
| `AccountFeatureDeleteEvent` | Feature not found  | 🟢 OK |
| `HistoryCodeDeleteEvent` | History code not found | 🟢 OK |

---

## Appendix

The following sections provide detailed reference information for the Digital Twin event schemas and Schema Registry configuration. Refer to these sections as needed while implementing your Digital Twin integration.

---

## Full Schema Definitions

### Common Types

#### `BaseEvent`
```avro
{
  "namespace": "com.matera.dtw.event.base.common",
  "type": "record",
  "name": "BaseEvent",
  "fields": [
    { "name": "content",  "type": "BaseEventContent" },
    { "name": "parents", "default": [], "type": { "type": "array", "items": "BaseEventContent" } }
  ]
}
```

#### `BaseEventContent`
```avro
{
  "namespace": "com.matera.dtw.event.base.common",
  "type": "record",
  "name": "BaseEventContent",
  "fields": [
    { "name": "source",       "type": "string" },
    { "name": "eventId",      "type": { "type": "string", "logicalType": "uuid" } },
    { "name": "createdAtUtc", "type": { "type": "long",   "logicalType": "local-timestamp-millis" } },
    { "name": "kind",         "type": "string" }
  ]
}
```

#### `AccountKey`
```avro
{
  "namespace": "com.matera.dtw.event.account.common",
  "type": "record",
  "name": "AccountKey",
  "fields": [
    { "name": "branch",  "type": "int" },
    { "name": "account", "type": "long" }
  ]
}
```

#### `AccountHolder`

Defined in `dtw-account-events`. Used as `accountHolder` in holder events and as `membership` in membership feature events — same structure, different field name.

```avro
{
  "namespace": "com.matera.dtw.event.account",
  "type": "record",
  "name": "AccountHolder",
  "fields": [
    { "name": "kind",     "type": "string" },
    { "name": "clientId", "type": "string" }
  ]
}
```

#### `AccountTypeKey`
```avro
{
  "namespace": "com.matera.dtw.event.configuration",
  "type": "record",
  "name": "AccountTypeKey",
  "fields": [
    { "name": "key", "type": "string" }
  ]
}
```

#### `HistoryCodeKey`
```avro
{
  "namespace": "com.matera.dtw.event.configuration",
  "type": "record",
  "name": "HistoryCodeKey",
  "fields": [
    { "name": "historyCodeId", "type": "string" }
  ]
}
```

#### `GenericDomainKey`

Sourced from `dtw-configuration-events`'s `generic-domain-event-key.avsc`. For Hold Reason, `key` is always the literal `"BALANCE_HOLD_REASON"`.

```avro
{
  "namespace": "com.matera.dtw.event.configuration",
  "type": "record",
  "name": "GenericDomainKey",
  "fields": [
    { "name": "key", "type": "string", "doc": "The unique identifier to generic domain record" }
  ]
}
```

#### `FeatureEntry` and `FeatureValue`
```avro
{
  "name": "FeatureEntry",
  "fields": [
    { "name": "name",  "type": "string" },
    { "name": "value", "type": "FeatureValue" }
  ]
}

{
  "name": "FeatureValue",
  "fields": [
    { "name": "type",           "type": "FeatureType",  "default": "STRING" },
    { "name": "booleanValue",   "type": ["null", "boolean"],                                              "default": null },
    { "name": "stringValue",    "type": ["null", "string"],                                               "default": null },
    { "name": "decimalValue",   "type": ["null", { "type": "bytes", "logicalType": "decimal", "precision": 19 }], "default": null },
    { "name": "dateValue",      "type": ["null", { "type": "int",   "logicalType": "date" }],             "default": null },
    { "name": "dateRangeValue", "type": ["null", "DateRange"],                                            "default": null }
  ]
}
```

### Schema Modules

| Maven Dependency | Contains |
|---|---|
| `dtw-common-events` | `BaseEvent`, `BaseEventContent`, `AccountKey`, `FeatureEntry`, `FeatureValue` |
| `dtw-account-events` | `AccountHolder`, `AccountChangeEvent`, `AccountDeleteEvent`, `AccountStatusChangeEvent`, `AccountFeatureChangeEvent`, `AccountFeatureDeleteEvent`, `AccountHolderChangeEvent`, `AccountHolderDeleteEvent`, `AccountMembershipFeatureChangeEvent`, `AccountMembershipFeatureDeleteEvent` |
| `dtw-configuration-events` | `AccountTypeChangeEvent`, `AccountTypeDeleteEvent`, `AccountTypeFeatureChangeEvent`, `AccountTypeFeatureDeleteEvent`, `HistoryCodeChangeEvent`, `HistoryCodeDeleteEvent`, `HistoryCodeFeatureChangeEvent`, `HistoryCodeFeatureDeleteEvent`, `GenericDomainKey`, `GenericDomainChangeEvent`, `GenericDomainDeleteEvent`, `GenericDomainFeatureChangeEvent`, `GenericDomainFeatureDeleteEvent`, `SystemConfigurationKey`, `SystemConfigurationFeatureChangeEvent`, `SystemConfigurationFeatureDeleteEvent`, `SystemConfigurationLimitChangeEvent`, `SystemConfigurationLimitDeleteEvent` |

---

## Advanced Integration Topics

### Avro Serialization & Schema Registry

All events are serialized as **Apache Avro** and registered in the **Confluent Schema Registry**.

#### Subject Name Strategy

DTW uses `TopicRecordNameStrategy` — the schema subject is derived from the **topic name + Avro record full name**, not just the topic name. This allows multiple event types to coexist on the same topic.

```properties
# Producer configuration
key.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
value.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
key.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
schema.registry.url=<provided-by-dtw-team>
```

#### Maven Dependencies

```xml
<dependency>
    <groupId>com.matera.dtw</groupId>
    <artifactId>dtw-account-events</artifactId>
</dependency>
<dependency>
    <groupId>com.matera.dtw</groupId>
    <artifactId>dtw-configuration-events</artifactId>
</dependency>
<dependency>
    <groupId>com.matera.dtw</groupId>
    <artifactId>dtw-common-events</artifactId>
</dependency>
```
