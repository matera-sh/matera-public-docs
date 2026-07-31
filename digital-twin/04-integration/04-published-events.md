# Digital Twin Integration Guide — Published Events

> **Audience:** Developers building systems that need to consume Digital Twin's transaction stream (statement services, notification systems, data warehouses, fraud/AML pipelines, and other downstream/interested systems).

> **Scope:** The entry-related events Digital Twin publishes once a transaction (ledger, blocking, unblocking, removal, reversal, or exclusion) has been authorized and processed.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [When These Events Are Published](#when-these-events-are-published)
- [Delivery & Ordering Guarantees](#delivery--ordering-guarantees)
- [Where These Events Are Published (Topic)](#where-these-events-are-published-topic)
- [What's Inside Every Event](#whats-inside-every-event)

**Event Catalog**

- [Ledger Entry Events](#ledger-entry-events)
  - [EntryCreationEvent](#entrycreationevent)
  - [PendingEntryCreationEvent](#pendingentrycreationevent)
- [Blocking Entry Events](#blocking-entry-events)
  - [BlockingEntryEvent](#blockingentryevent)
  - [PendingBlockingEntryEvent](#pendingblockingentryevent)
- [Unblocking Entry Events](#unblocking-entry-events)
  - [UnblockingEntryEvent](#unblockingentryevent)
  - [PendingUnblockingEntryEvent](#pendingunblockingentryevent)
- [Removal Entry Events](#removal-entry-events)
  - [EntryRemovalEvent](#entryremovalevent)
  - [PendingEntryRemovalEvent](#pendingentryremovalevent)
- [Reversal Entry Events](#reversal-entry-events)
  - [EntryReversalEvent](#entryreversalevent)
  - [PendingReversalEntryEvent](#pendingreversalentryevent)
- [Exclusion Event](#exclusion-event)
  - [EntryExclusionEvent](#entryexclusionevent)

**Operations**

- [Consumer Setup Guidance](#consumer-setup-guidance)
- [Appendix — Full Schema Definitions](#appendix--full-schema-definitions)

---

## Overview

Once a channel submits a transaction authorization request to Digital Twin and DTW authorizes and processes it, the resulting change to the ledger is captured and published as an **event** to a Kafka topic. Interested systems — statement services, notification systems, data warehouses, reconciliation and AML pipelines — subscribe to this topic to build their own view of what happened in DTW, without querying DTW's APIs directly for every transaction.

```
Channel → DTW (command) → DTW authorizes & persists the entry → CDC polling picks up the change
    → transaction-event-producer → Kafka (dtw.transaction.event.transaction) → Interested Systems
```

The component responsible for this is **`transaction-event-producer`**, a CDC (Change Data Capture) service that polls Digital Twin's internal transaction data for changes and publishes one event per resulting entry. 

---

## Prerequisites

Before consuming these events, make sure the following are in place:

| Requirement | Details | Why it's needed |
|---|---|---|
| Kafka cluster access | Connection credentials and broker addresses provided by the DTW team | Your system needs a network path to read from Digital Twin's topic |
| Confluent Schema Registry access | URL and credentials for schema resolution | Every event is registered with an Avro schema; your consumer needs to resolve it to deserialize messages |
| Apache Avro message format | Your integration must support Apache Avro deserialization | All events on this topic are Avro-encoded |
| Required Digital Twin event library | Maven dependency `dtw-entry-events` (brings in `dtw-common-events`, `dtw-entry-common-events` transitively) | Defines the exact event structures so you don't have to build the schemas by hand |
| Environment identifier | The `[ENV]` value used in the topic name (e.g., `dev`, `uat`, `prod`) | Used to subscribe to the correct environment's topic |
| Idempotent consumption | Dedupe incoming events by `baseEvent.content.eventId` | See [Delivery & Ordering Guarantees](#delivery--ordering-guarantees) — this guide could not confirm an exactly-once delivery guarantee from the producer side, so treat delivery as at-least-once |

---

## When These Events Are Published

An event is published whenever an entry's state changes as a result of an authorized command — one event per resulting entry, not one event per command. Since a single `ComboEntryRequestCommand` can carry multiple entries (see the commands guide), processing it can produce **multiple** events, one per entry in the combo.

| Trigger (command) | Resulting event(s) |
|---|---|
| `LedgerEntryRequestData` with `status: POSTED` | `EntryCreationEvent` |
| `LedgerEntryRequestData` with `status: PENDING` | `PendingEntryCreationEvent` |
| `BlockingEntryRequestData` with `status: POSTED`/`ON_HOLD` | `BlockingEntryEvent` |
| `BlockingEntryRequestData` with `status: PENDING` | `PendingBlockingEntryEvent` |
| `UnblockingEntryRequestData` with `status: POSTED` | `UnblockingEntryEvent` |
| `UnblockingEntryRequestData` with `status: PENDING` | `PendingUnblockingEntryEvent` |
| `RemovalEntryRequestData` with `status: POSTED` | `EntryRemovalEvent` |
| `RemovalEntryRequestData` with `status: PENDING` | `PendingEntryRemovalEvent` |
| `ReversalEntryRequestData` with `status: POSTED` | `EntryReversalEvent` |
| `ReversalEntryRequestData` with `status: PENDING` | `PendingReversalEntryEvent` |
| `ComboEntryExclusionRequestCommand` | `EntryExclusionEvent` (one per excluded correlation id) |

> **Note:** This mapping follows directly from the command's `status` field and entry kind (see the commands guide); it was not independently re-verified against the `transaction-event-producer` mapper code field-by-field. The event *shapes* below (fields, types, sizes) were verified against source.

---

## Delivery & Ordering Guarantees

- **Single topic, multiple record types.** Every event in this guide is published to the **same** Kafka topic. Digital Twin uses Confluent's **`TopicRecordNameStrategy`** for the schema subject on the value, so multiple Avro record types can coexist on one topic — your consumer distinguishes the event type by its Avro record, not by the topic.
- **Partition key = account.** Every message is keyed by `AccountKey` (`branch` + `account`). This guarantees relative ordering of events for the **same account** within a partition, but gives **no ordering guarantee across different accounts**.
- **No dedicated retry/DLQ topic on this outbound flow.** Unlike inbound event flows (where a failed event goes to a retry topic), dispatch failures on this producer are handled internally via a database routing/error table and re-attempted through the CDC polling loop itself — there is no consumer-visible retry topic to monitor for this direction.
- **Treat delivery as at-least-once.** This guide did not find an explicit exactly-once guarantee for this producer. Design your consumer to be idempotent, using `baseEvent.content.eventId` (or the entry's own `id`, where present) to detect and discard duplicates.

---

## Where These Events Are Published (Topic)

```
[ENV].dtw.transaction.event.transaction
```

| Segment | Value |
|---|---|
| `[ENV]` | Environment prefix (e.g., `dev`, `uat`, `prod`) — provided by the DTW team |
| Fixed suffix | `.dtw.transaction.event.transaction` — carries every event in this guide's catalog |

> **Internal-only topic, not for consumers:** Digital Twin also uses a `[ENV].dtw.transaction.private.event.transaction` topic internally, as an intermediate stage of its own CDC pipeline before enrichment. Do not subscribe to it — it is not a supported consumer-facing contract.

---

## What's Inside Every Event

Every event in this catalog wraps the same `BaseEvent` header used throughout the Digital Twin platform, plus one `entry` payload specific to what happened.

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440100",
      "createdAtUtc": 1719360500000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": { "...": "one of the entry payloads catalogued below" }
}
```

### `BaseEvent`

Think of `BaseEvent` as the shipping label on a package — it tells Digital Twin what the message is, where it came from, and when it was created. It also records *what earlier event triggered it* (lineage), if any.

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `content` | `BaseEventContent` | Yes | — | Core event metadata |
| `parents` | `BaseEventContent[]` | No | — | Lineage — the chain of events that originated this one. Oldest first. Defaults to empty. |

### `BaseEventContent`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | no limit | Identifies the originating system. For DTW-published events this is always `"dtw-transaction"`. |
| `eventId` | `string (uuid)` | Yes | 36 chars | Unique event ID. Used for idempotent consumption — deduplicate on this field. Must be a valid UUID. |
| `createdAtUtc` | `long (local-timestamp-millis)` | Yes | — | Timestamp of when the event was generated, in **UTC milliseconds** since epoch. |
| `kind` | `string` | Yes | no limit | Descriptor of the event type. See the ⚠️ callout below — for transaction entry events, this field is **not** a reliable discriminator. |

### ⚠️ Important — the `kind` field does **not** identify which event this is

Unlike the account events topic (where `kind` follows a predictable `{aggregate}-{action}-event` pattern unique per event), all creation-family events here (`EntryCreationEvent`, `PendingEntryCreationEvent`, `BlockingEntryEvent`, `PendingBlockingEntryEvent`, `UnblockingEntryEvent`, `PendingUnblockingEntryEvent`, `EntryRemovalEvent`, `PendingEntryRemovalEvent`, `EntryReversalEvent`, `PendingReversalEntryEvent`) are published with the **exact same literal value**:

```
"kind": "entry-creation-event"
```

`EntryExclusionEvent` is the one exception — its `kind` is built dynamically as `"{domainKind}-exclusion-event"`, where `domainKind` is the lowercase hyphenated name derived from the compensating entry's domain class (not the Avro `entry.kind` discriminator). The confirmed values from test assertions are:

| Compensating entry type | `BaseEventContent.kind` |
|---|---|
| Posted ledger entry | `"posted-ledger-entry-exclusion-event"` |
| Pending ledger entry | `"pending-ledger-entry-exclusion-event"` |

> **Note:** do not confuse this with the Avro payload's `entry.kind` field (which uses uppercase discriminators like `"LEDGER"`, `"BLOCKING"` — see below). The `BaseEventContent.kind` for exclusion events follows the same lowercase-kebab convention as the rest of DTW events.

**Practical consequence:** do not branch your consumer logic on `baseEvent.content.kind`. Use the Avro record's own type (resolved via the Schema Registry/`TopicRecordNameStrategy`) to know which event you received, and use the entry payload's own `kind` field (documented per event below) to know which specific entry variant it carries.

### `AccountKey`

Identifies an account uniquely within DTW. Every event in this catalog that acts on an account carries one of these.

```json
{
  "branch": 1,
  "account": 230621
}
```

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `branch` | `int` | Yes | — | Routing number |
| `account` | `long` | Yes | — | Account number |

> **Important:** Account keys must be positive. DTW explicitly filters out non-positive keys (they are reserved for system/sentinel accounts).

### `FinancialAmount`

Every monetary amount in this guide's events uses this structure.

```json
{ "currency": "USD", "amount": 200000 }
```

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `currency` | `enum` | Yes | — | ISO 4217 currency code plus supported stablecoin/crypto tickers. |
| `amount` | `bytes (decimal, precision 19)` | Yes | precision 19 | Unscaled value — multiply by the currency's scale to get the real amount (e.g. $14.85 is encoded as `1485`) |

---

## Ledger Entry Events

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of the commands guide's `LedgerEntryRequestData` — a simple debit/credit movement, once realized.

---

### `EntryCreationEvent`

#### Conceptual Overview

Published when a ledger (debit/credit) entry is created and immediately effective (`POSTED`).

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440101",
      "createdAtUtc": 1719360500000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "id": "e02bb9c2-2771-4f03-9299-16db85485ee1",
    "account": { "branch": 1, "account": 230621 },
    "kind": "LEDGER",
    "historyCode": 998,
    "holdReasonId": null,
    "asOfDate": 20609,
    "correlations": { "FedNow": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10" },
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": 200000 } },
    "amount": { "currency": "USD", "amount": 200000 },
    "description": "Credit Card",
    "pendingEntry": null,
    "metaData": null,
    "initiator": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.id` | `string (uuid)`, nullable | No | — | The newly-created entry's id. Optional, default `null` |
| `entry.account` | `AccountKey` | Yes | — | The account the entry belongs to |
| `entry.kind` | `string` | No | — | Discriminates the entry variant. Optional, default `"LEDGER"` |
| `entry.historyCode` | `int` | Yes | N/A (int) | Determines credit/debit classification. Resolves to a `String` business key (`HistoryCodeKey`) internally — see the commands guide's caveat on this type mismatch |
| `entry.holdReasonId` | `string`, nullable | No | max 64 chars (by analogy to the equivalent command field — not independently re-verified for this event class) | Only meaningful for credit entries created `ON_HOLD`. Optional, default `null` |
| `entry.asOfDate` | `date` | Yes | N/A | The accounting date of the entry |
| `entry.correlations` | `map<string,string>` | No | — | Correlation map to identify this entry in other systems. Optional, default `{}` |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Which balances changed and by how much. Optional, default `{}`, may be empty |
| `entry.amount` | `FinancialAmount` | Yes | — | The entry's amount |
| `entry.description` | `string`, nullable | No | max 40 chars (`DTW_TR_ENTRY.DESCRIPTION`) | Free text shown on the customer statement. Optional, default `null` |
| `entry.pendingEntry` | `PendingEntryInfo`, nullable | No | — | Set if this entry originated from a prior pending/blocked entry. Optional, default `null` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded by a discovered column limit | Free-form metadata; must not contain personal data (LGPD/GDPR). Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | The entry's initiator, if not originated from an API/Command. Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.EntryCreationEvent` (payload: `com.matera.dtw.event.transaction.entry.LedgerEntry`)

---

### `PendingEntryCreationEvent`

#### Conceptual Overview

Published when a credit entry is created in the `PENDING` state — the credit is held rather than immediately available.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440102",
      "createdAtUtc": 1719360600000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "id": "f1a2b3c4-d5e6-4f70-8a9b-1c2d3e4f5062",
    "account": { "branch": 1, "account": 230621 },
    "kind": "PENDING_LEDGER",
    "historyCode": 998,
    "asOfDate": 20609,
    "correlations": {},
    "balanceChanges": {},
    "amount": { "currency": "USD", "amount": 200000 },
    "description": "Credit pending confirmation",
    "holdReasonId": null,
    "metaData": null,
    "initiator": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `entry.account` | `AccountKey` | Yes | — | The account the entry belongs to |
| `entry.kind` | `string` | No | — | Optional, default `"PENDING_LEDGER"` |
| `entry.historyCode` | `int` | Yes | N/A | Same caveat as `EntryCreationEvent.entry.historyCode` |
| `entry.asOfDate` | `date` | Yes | N/A | Accounting date |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.amount` | `FinancialAmount` | Yes | — | The entry's amount |
| `entry.description` | `string`, nullable | No | max 40 chars | Optional, default `null` |
| `entry.holdReasonId` | `string`, nullable | No | max 64 chars (by analogy) | Reason the credit is held. No declared default in the schema |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.PendingEntryCreationEvent` (payload: `com.matera.dtw.event.transaction.entry.PendingLedgerEntry`)

---

## Blocking Entry Events

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of `BlockingEntryRequestData` — funds reserved (held) on an account.

---

### `BlockingEntryEvent`

#### Conceptual Overview

Published when a cautionary or balance-level hold is placed and immediately effective.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440103",
      "createdAtUtc": 1719360700000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": -5000 } },
    "correlations": { "CARD": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab" },
    "description": "Pre-auth hold",
    "id": "f1a2b3c4-d5e6-4f70-8a9b-1c2d3e4f5061",
    "kind": "BLOCKING",
    "metaData": null,
    "historyCode": 998,
    "holdReasonId": "PRE_AUTH",
    "asOfDate": 20609,
    "amount": { "currency": "USD", "amount": 5000 },
    "pendingEntry": null,
    "initiator": null,
    "blockingKind": "BALANCE",
    "target": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.account` | `AccountKey` | Yes | — | The blocked account |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.description` | `string`, nullable | No | max 40 chars | Optional, default `null` |
| `entry.id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `entry.kind` | `string` | No | — | Optional, default `"BLOCKING"` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.historyCode` | `int` | Yes | N/A | Determines blocked credit type |
| `entry.holdReasonId` | `string`, nullable | No | max 64 chars (by analogy) | Optional, default `null` |
| `entry.asOfDate` | `date` | Yes | N/A | — |
| `entry.amount` | `FinancialAmount` | Yes | — | The amount held |
| `entry.pendingEntry` | `PendingEntryInfo`, nullable | No | — | Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |
| `entry.blockingKind` | `enum` (`CAUTIONARY`\|`BALANCE`) | No | N/A | Optional, default `"BALANCE"` |
| `entry.target` | `CautionaryBlockTargetInfo`, nullable | No | — | Required (functionally) when `blockingKind` is `CAUTIONARY`. Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.BlockingEntryEvent` (payload: `com.matera.dtw.event.transaction.entry.BlockingEntry`)

---

### `PendingBlockingEntryEvent`

#### Conceptual Overview

Published when a hold is created in the `PENDING` state.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440104",
      "createdAtUtc": 1719360800000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": {},
    "correlations": {},
    "description": "Pre-auth hold pending",
    "id": "f1a2b3c4-d5e6-4f70-8a9b-1c2d3e4f5063",
    "kind": "PENDING_BLOCKING",
    "metaData": null,
    "historyCode": 998,
    "holdReasonId": "PRE_AUTH",
    "asOfDate": 20609,
    "amount": { "currency": "USD", "amount": 5000 },
    "initiator": null,
    "blockingKind": "BALANCE",
    "target": null
  }
}
```

#### Field Reference

Same fields as `BlockingEntryEvent` above, minus `pendingEntry` (not applicable — this event *is* the pending state, so it has no `pendingEntry` of its own), and with `entry.kind` defaulting to `"PENDING_BLOCKING"`.

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.PendingBlockingEntryEvent` (payload: `com.matera.dtw.event.transaction.entry.PendingBlockingEntry`)

---

## Unblocking Entry Events

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of `UnblockingEntryRequestData` — releasing (or confirming) a previously held amount.

---

### `UnblockingEntryEvent`

#### Conceptual Overview

Published when a hold is released and the release is immediately effective.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440105",
      "createdAtUtc": 1719360900000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": 5000 } },
    "correlations": { "CARD": "a7b8c9d0-e1f2-4456-0123-567890123456" },
    "description": "Pre-auth released",
    "id": "a7b8c9d0-e1f2-4456-0123-567890123456",
    "kind": "UNBLOCKING",
    "metaData": null,
    "historyCode": 998,
    "holdReasonId": "PRE_AUTH",
    "asOfDate": 20611,
    "amount": { "currency": "USD", "amount": 5000 },
    "pendingEntry": null,
    "initiator": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.account` | `AccountKey` | Yes | — | — |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.description` | `string`, nullable | No | max 40 chars | Optional, default `null` |
| `entry.id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `entry.kind` | `string` | No | — | Optional, default `"UNBLOCKING"` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.historyCode` | `int` | Yes | N/A | Determines unblocked credit type |
| `entry.holdReasonId` | `string`, nullable | No | max 64 chars (by analogy) | Optional, default `null` |
| `entry.asOfDate` | `date` | Yes | N/A | — |
| `entry.amount` | `FinancialAmount` | Yes | — | The amount released |
| `entry.pendingEntry` | `PendingEntryInfo`, nullable | No | — | Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.UnblockingEntryEvent` (payload: `com.matera.dtw.event.transaction.entry.UnblockingEntry`)

---

### `PendingUnblockingEntryEvent`

#### Conceptual Overview

Published when a release is created in the `PENDING` state.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440106",
      "createdAtUtc": 1719361000000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": {},
    "correlations": {},
    "description": "Pre-auth release pending",
    "id": "a7b8c9d0-e1f2-4456-0123-567890123457",
    "kind": "PENDING_UNBLOCKING",
    "metaData": null,
    "historyCode": 998,
    "holdReasonId": "PRE_AUTH",
    "asOfDate": 20611,
    "amount": { "currency": "USD", "amount": 5000 },
    "initiator": null
  }
}
```

#### Field Reference

Same fields as `UnblockingEntryEvent`, minus `pendingEntry` (not applicable), with `entry.kind` defaulting to `"PENDING_UNBLOCKING"`.

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.PendingUnblockingEntryEvent` (payload: `com.matera.dtw.event.transaction.entry.PendingUnblockingEntry`)

---

## Removal Entry Events

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of `RemovalEntryRequestData` — cancelling an entry that had not yet taken full effect.

---

### `EntryRemovalEvent`

#### Conceptual Overview

Published when an entry is removed (cancelled) and the removal is immediately effective.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440107",
      "createdAtUtc": 1719361100000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": 5000 } },
    "correlations": { "CARD": "c3d4e5f6-a7b8-4012-cdef-123456789012" },
    "description": "Cancel pre-auth",
    "id": "c3d4e5f6-a7b8-4012-cdef-123456789012",
    "kind": "REMOVAL",
    "metaData": null,
    "compensatingEntry": {
      "id": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
      "kind": "BLOCKING",
      "amount": { "currency": "USD", "amount": 5000 },
      "correlations": {},
      "balanceChanges": {},
      "pendingEntry": null
    },
    "pendingEntry": null,
    "initiator": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.account` | `AccountKey` | Yes | — | The account associated with the removal |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.description` | `string`, nullable | No | max 40 chars | Optional, default `null` |
| `entry.id` | `string (uuid)`, nullable | No | — | The id of the newly-removed entry. Optional, default `null` |
| `entry.kind` | `string` | No | — | Optional, default `"REMOVAL"` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.compensatingEntry` | `CompensatingEntryInfo` | Yes | — | Information about the entry that was removed — see [Common Supporting Types](#common-supporting-types) |
| `entry.pendingEntry` | `PendingEntryInfo`, nullable | No | — | Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.EntryRemovalEvent` (payload: `com.matera.dtw.event.transaction.entry.RemovalEntry`)

---

### `PendingEntryRemovalEvent`

#### Conceptual Overview

Published when a removal is created in the `PENDING` state.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440108",
      "createdAtUtc": 1719361200000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": {},
    "correlations": {},
    "description": "Cancel pre-auth pending",
    "id": "c3d4e5f6-a7b8-4012-cdef-123456789013",
    "kind": "PENDING_REMOVAL",
    "metaData": null,
    "compensatingEntry": {
      "id": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
      "kind": "BLOCKING",
      "amount": { "currency": "USD", "amount": 5000 },
      "correlations": {},
      "balanceChanges": {},
      "pendingEntry": null
    },
    "initiator": null
  }
}
```

#### Field Reference

Same fields as `EntryRemovalEvent`, minus the entry's own `pendingEntry` (not applicable), with `entry.kind` defaulting to `"PENDING_REMOVAL"`.

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.PendingEntryRemovalEvent` (payload: `com.matera.dtw.event.transaction.entry.PendingRemovalEntry`)

---

## Reversal Entry Events

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of `ReversalEntryRequestData` — undoing the financial effect of an already-posted entry.

---

### `EntryReversalEvent`

#### Conceptual Overview

Published when a posted entry is reversed and the reversal is immediately effective.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440109",
      "createdAtUtc": 1719361300000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": -200000 } },
    "id": "e5f6a7b8-c9d0-4234-ef01-345678901234",
    "kind": "REVERSAL",
    "correlations": { "FedNow": "e5f6a7b8-c9d0-4234-ef01-345678901234" },
    "metaData": null,
    "reversedEntry": {
      "id": "e02bb9c2-2771-4f03-9299-16db85485ee1",
      "kind": "LEDGER",
      "amount": { "currency": "USD", "amount": 200000 },
      "correlations": {},
      "balanceChanges": {},
      "pendingEntry": null
    },
    "pendingEntry": null,
    "initiator": null,
    "asOfDate": 20610
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.account` | `AccountKey` | Yes | — | — |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `entry.kind` | `string` | No | — | Optional, default `"REVERSAL"` |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.reversedEntry` | `CompensatingEntryInfo` | Yes | — | Information about the entry being reversed |
| `entry.pendingEntry` | `PendingEntryInfo`, nullable | No | — | Optional, default `null` |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |
| `entry.asOfDate` | `date` | Yes | N/A | The accounting date of the reversal |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.EntryReversalEvent` (payload: `com.matera.dtw.event.transaction.entry.ReversalEntry`)

---

### `PendingReversalEntryEvent`

#### Conceptual Overview

Published when a reversal is created in the `PENDING` state.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440110",
      "createdAtUtc": 1719361400000,
      "kind": "entry-creation-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "balanceChanges": {},
    "id": "e5f6a7b8-c9d0-4234-ef01-345678901235",
    "kind": "PENDING_REVERSAL",
    "correlations": {},
    "description": "Refund (Pending)",
    "metaData": null,
    "reversedEntry": {
      "id": "e02bb9c2-2771-4f03-9299-16db85485ee1",
      "kind": "LEDGER",
      "amount": { "currency": "USD", "amount": 200000 },
      "correlations": {},
      "balanceChanges": {},
      "pendingEntry": null
    },
    "initiator": null,
    "asOfDate": 20610
  }
}
```

#### Field Reference

Same fields as `EntryReversalEvent`, minus the entry's own `pendingEntry` (not applicable), with `entry.kind` defaulting to `"PENDING_REVERSAL"` — **plus one field `EntryReversalEvent` does not have:** `entry.description` (`string`, nullable, optional, default `null`, max 40 chars). Unlike the other posted/pending pairs in this guide, `PendingReversalEntry` is not simply `ReversalEntry` minus `pendingEntry` — it also adds this `description` field, which the posted `ReversalEntry` record does not carry at all.

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.PendingReversalEntryEvent` (payload: `com.matera.dtw.event.transaction.entry.PendingReversalEntry`)

---

## Exclusion Event

**Topic:** `[ENV].dtw.transaction.event.transaction`

**Direction:** DTW → Interested Systems

The counterpart of `ComboEntryExclusionRequestCommand` — one `EntryExclusionEvent` is published per correlation id excluded.

---

### `EntryExclusionEvent`

#### Conceptual Overview

Published when a previously-created entry is excluded (rolled back) as part of a combo exclusion.

> **Reminder:** this is the one event whose `kind` is **not** the shared `"entry-creation-event"` literal — see [What's Inside Every Event](#whats-inside-every-event). It is `{compensatingEntry.kind}-exclusion-event`, e.g. `"LEDGER-exclusion-event"`.

#### Payload

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655440111",
      "createdAtUtc": 1719361500000,
      "kind": "LEDGER-exclusion-event"
    },
    "parents": []
  },
  "entry": {
    "account": { "branch": 1, "account": 230621 },
    "kind": "EXCLUSION",
    "id": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
    "correlations": {},
    "balanceChanges": { "AVAILABLE": { "currency": "USD", "amount": -200000 } },
    "metaData": null,
    "compensatingEntry": {
      "id": "e02bb9c2-2771-4f03-9299-16db85485ee1",
      "kind": "LEDGER",
      "amount": { "currency": "USD", "amount": 200000 },
      "correlations": {},
      "balanceChanges": {},
      "pendingEntry": null
    },
    "initiator": null
  }
}
```

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `entry.account` | `AccountKey` | Yes | — | — |
| `entry.kind` | `string` | No | — | Optional, default `"EXCLUSION"` |
| `entry.id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `entry.correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `entry.balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `entry.metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `entry.compensatingEntry` | `CompensatingEntryInfo` | Yes | — | Information about the entry being excluded |
| `entry.initiator` | `InitiatorInfo`, nullable | No | — | Optional, default `null` |

#### Avro Record Name

`com.matera.dtw.event.transaction.entry.EntryExclusionEvent` (payload: `com.matera.dtw.event.transaction.entry.ExclusionEntry`)

---

## Common Supporting Types

These records appear nested inside multiple events above.

### `InitiatorInfo`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `kind` | `string` | Yes | — | The initiator kind. No declared default |
| `correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |

### `PendingEntryInfo`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `kind` | `string` | No | — | Optional, default `"LEDGER"` |
| `correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |

### `CompensatingEntryInfo`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `id` | `string (uuid)`, nullable | No | — | The id of the entry being compensated. Optional, default `null` |
| `kind` | `string` | Yes | — | The kind of the entry being compensated. No declared default |
| `amount` | `FinancialAmount` | Yes | — | The amount of the compensating entry |
| `correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |
| `pendingEntry` | `PendingEntryInfo`, nullable | No | — | Optional, default `null` |

### `CautionaryBlockTargetInfo`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `id` | `string (uuid)`, nullable | No | — | Optional, default `null` |
| `kind` | `string` | No | — | Optional, default `"LEDGER"` |
| `correlations` | `map<string,string>` | No | — | Optional, default `{}` |
| `metaData` | `string (json)`, nullable | No | Not bounded | Optional, default `null` |
| `balanceChanges` | `map<string,FinancialAmount>` | No | — | Optional, default `{}` |

### `BlockingKind` (enum)

| Value | Meaning |
|---|---|
| `CAUTIONARY` | The hold targets a specific, not-yet-created entry (see `target`) |
| `BALANCE` (default) | The hold applies against the account's balance directly, with no specific target entry |

---

## Consumer Setup Guidance

#### Maven Dependency

```xml
<dependency>
    <groupId>com.matera.dtw.event</groupId>
    <artifactId>dtw-entry-events</artifactId>
</dependency>
```

This transitively brings in `dtw-common-events` (for `BaseEvent`/`BaseEventContent`/`AccountKey`/`FinancialAmount`) and `dtw-entry-common-events` (for `InitiatorInfo`, `PendingEntryInfo`, `CompensatingEntryInfo`, `ExclusionEntry`).

#### Subject Name Strategy

```properties
key.deserializer=io.confluent.kafka.serializers.KafkaAvroDeserializer
value.deserializer=io.confluent.kafka.serializers.KafkaAvroDeserializer
value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
schema.registry.url=<provided-by-dtw-team>
```

> This guide confirmed the **value** subject uses `TopicRecordNameStrategy` in the producer's configuration. A specific key subject strategy was not found configured in the producer module — if your consumer needs to deserialize the key as Avro, confirm the expected strategy with the DTW team before assuming it matches the value.

#### Distinguishing Event Types

Because `baseEvent.content.kind` does not reliably distinguish these events from one another (see [the callout above](#whats-inside-every-event)), branch your consumer logic on:
1. The **Avro record type** resolved by your Avro deserializer (e.g. the generated Java class, or the schema's `name`/`namespace`), and/or
2. The `entry.kind` field inside the payload (`LEDGER`, `PENDING_LEDGER`, `BLOCKING`, `PENDING_BLOCKING`, `UNBLOCKING`, `PENDING_UNBLOCKING`, `REMOVAL`, `PENDING_REMOVAL`, `REVERSAL`, `PENDING_REVERSAL`, `EXCLUSION`).

#### Idempotency

Treat delivery as at-least-once (see [Delivery & Ordering Guarantees](#delivery--ordering-guarantees)) and deduplicate using `baseEvent.content.eventId`.

---

## Appendix — Full Schema Definitions

### Schema Modules

| Maven Dependency | Contains |
|---|---|
| `dtw-common-events` | `BaseEvent`, `BaseEventContent`, `AccountKey`, `FinancialAmount` |
| `dtw-entry-common-events` | `InitiatorInfo`, `PendingEntryInfo`, `CompensatingEntryInfo`, `ExclusionEntry` |
| `dtw-entry-events` | `LedgerEntry`, `PendingLedgerEntry`, `ReversalEntry`, `RemovalEntry`, `BlockingEntry`, `UnblockingEntry`, `PendingBlockingEntry`, `PendingUnblockingEntry`, `PendingRemovalEntry`, `PendingReversalEntry`, `CautionaryBlockTargetInfo`, `BlockingKind`, and all 11 wrapper events catalogued above |

### Source Files (for reference)

| Type(s) | File |
|---|---|
| The 11 wrapper events | `dtw-events/dtw-entry-events/src/main/avro/entry-events.avsc` |
| `LedgerEntry`, `PendingLedgerEntry`, `ReversalEntry`, `RemovalEntry`, `BlockingEntry`, `UnblockingEntry`, `PendingBlockingEntry`, `PendingUnblockingEntry`, `PendingRemovalEntry`, `PendingReversalEntry`, `BlockingKind`, `CautionaryBlockTargetInfo` | `dtw-events/dtw-entry-events/src/main/avro/entry.avsc` |
| `ExclusionEntry` | `dtw-events/dtw-entry-common-events/src/main/avro/entry-common-commands.avsc` |
| `InitiatorInfo`, `PendingEntryInfo`, `CompensatingEntryInfo` | `dtw-events/dtw-entry-common-events/src/main/avro/common-entry-info.avsc` |
| `FinancialAmount` | `dtw-events/dtw-common-events/src/main/avro/common-financial.avsc` |
| `BaseEvent`, `BaseEventContent` | `dtw-events/dtw-common-events/src/main/avro/common-events.avsc` |
| `AccountKey` | `dtw-events/dtw-common-events/src/main/avro/common-events-key.avsc` |

### Producer Internals (for context, not part of the consumer contract)

| Detail | Value | Source |
|---|---|---|
| Producer module | `transaction-event-producer` (CDC polling-based) | `dtw-transaction/transaction-event-producer` |
| Public topic property | `application.kafka.topics.transaction-events` | `transaction-event-producer/src/main/resources/application-kafka.yml` |
| Internal-only topic property | `application.kafka.topics.private-transaction-events` (`[ENV].dtw.transaction.private.event.transaction`) — not for consumers | same file |
| Error handling | Routed to a database table (`DTW_TR_EVENT_ROUTING`), reprocessed via the CDC polling loop — no consumer-visible Kafka retry topic | `transaction-event-producer/src/main/resources/application.yml` |
