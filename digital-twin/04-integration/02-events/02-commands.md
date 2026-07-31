# Digital Twin Integration Guide — Commands

**Audience:** Developers integrating core banking systems and other upstream systems with Digital Twin to request transaction (ledger) operations

---

## Getting Started

This guide describes the commands used to request entry (ledger) operations on a Digital Twin account — postings, blocks, releases, removals, reversals, and unblocks — and the asynchronous results Digital Twin sends back for each request. It explains when to send each command, the data required, how Digital Twin processes it, and how you receive success or failure results. It does not describe the events used to populate and maintain account and reference data (account, account type, history code, generic domain/hold reason) — those are covered in the companion **Digital Twin Integration Guide — Events**, which explicitly excludes transaction authorization requests in favor of this guide.

Each command carries one entry-kind operation per message. **Coming soon:** multiple entry-kind operations submitted for one account atomically in a single message (combo commands) — see [Combo Entry Commands](#combo-entry-commands).


---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)

**Shared Building Blocks**

3. [When to Send Commands](#when-to-send-commands)
4. [In What Order to Send Commands](#in-what-order-to-send-commands)
5. [Kafka Topic Naming Convention](#kafka-topic-naming-convention)
6. [What's Inside Every Command](#whats-inside-every-command)
7. [How Results Come Back](#how-results-come-back)

**Command Catalog**

8. [Entry Commands](#entry-commands)
   - [EntryRequestCommand](#entryrequestcommand)
   - [Ledger Entry](#ledger-entry)
   - [Blocking Entry](#blocking-entry)
   - [Unblocking Entry](#unblocking-entry)
   - [Removal Entry](#removal-entry)
   - [Reversal Entry](#reversal-entry)
   - [Exclusion Entry](#exclusion-entry)
   - [Release Entry](#release-entry)

**Command Catalog — Coming Soon**

9. [Combo Entry Commands](#combo-entry-commands)
   - [ComboEntryRequestCommand](#comboentryrequestcommand)
   - [ComboEntryExclusionRequestCommand](#comboentryexclusionrequestcommand)

**Operations**

10. [Command Validation and Error Handling](#command-validation-and-error-handling)
    - [What Happens When a Command Fails?](#what-happens-when-a-command-fails)
    - [Idempotency Behavior](#idempotency-behavior)
    - [Ledger Entry — Validations and Error Responses](#ledger-entry--validations-and-error-responses)
    - [Blocking Entry — Validations and Error Responses](#blocking-entry--validations-and-error-responses)
    - [Unblocking Entry — Validations and Error Responses](#unblocking-entry--validations-and-error-responses)
    - [Removal Entry — Validations and Error Responses](#removal-entry--validations-and-error-responses)
    - [Reversal Entry — Validations and Error Responses](#reversal-entry--validations-and-error-responses)
    - [Exclusion Entry — Validations and Error Responses](#exclusion-entry--validations-and-error-responses)
    - [Release Entry — Validations and Error Responses](#release-entry--validations-and-error-responses)
    - [Combo Entry Command — Validations and Error Responses](#combo-entry-command--validations-and-error-responses)
    - [ComboEntryExclusionRequestCommand — Validations and Error Responses](#comboentryexclusionrequestcommand--validations-and-error-responses)
    - [Complete Error Reference](#complete-error-reference)

**Appendix**

11. [Appendix](#appendix)
    - [Full Schema Definitions](#full-schema-definitions)
    - [Advanced Integration Topics](#advanced-integration-topics)

---

## Overview

Digital Twin (DTW) is Matera's real-time authorization ledger. Once an account exists in DTW (via the Events flow), core banking systems and other upstream systems send **commands** to request ledger operations against it — post an entry, block or release a credit, remove or reverse a previous entry.

Unlike events, commands are **request/reply**: every command carries a `replyTo` (and optional `replyKey`) telling Digital Twin where to publish the outcome. Digital Twin does not respond synchronously over HTTP — the result arrives as a separate, asynchronous Kafka message on the topic you specified.

```
Core Banking System / Upstream System ──► Kafka (operations.<subject>.command) ──► DTW Transaction
                                                                                          │
                        Core Banking System / Upstream System ◄──── Kafka (your replyTo) ─┘
```

The publishing system is responsible for detecting the need for a ledger operation, translating it into the command format described in this guide, and publishing it. Digital Twin validates the command, executes it against the account, and publishes either a success result (the created/affected entry data) or a failure result (structured error information) back to `replyTo`.

---

## Prerequisites

Before sending commands, make sure the following are in place:

| Requirement | Details | Why it's needed |
|---|---|---|
| Everything from the Events guide's prerequisites | Kafka/Schema Registry access, Avro support | Commands use the same transport and serialization as events |
| The target account already exists in DTW | Published via `AccountChangeEvent` — see the Events guide | Commands act on an existing account; DTW does not create accounts on demand from a command (see [In What Order to Send Commands](#in-what-order-to-send-commands)) |
| Required Digital Twin command libraries | Maven dependencies `dtw-entry-events`, `dtw-entry-common-events`, `dtw-common-events` | These define the exact command/result structures DTW expects |
| A reply topic your system owns and consumes | You provide this per-command via `replyTo` (and optionally `replyKey`) | This is how you receive the success or failure result — there is no shared "results" topic |
| Reply-topic and consumer-lag monitoring | Alerting on your own `replyTo` topic and on your command producer's ability to publish | **Digital Twin has no retry topic or DLQ for commands (see [Command Validation and Error Handling](#command-validation-and-error-handling)) — if your `replyTo` value is misconfigured, the result is silently dropped, not queued.** |

---

## When to Send Commands

Send a command whenever your system needs Digital Twin to perform a ledger operation. Digital Twin validates the command, executes it, and asynchronously publishes the result to the topic named in `replyTo`.

| Operation | Command | What it does |
|---|---|---|
| Post a ledger entry | `EntryRequestCommand` (entry = `LedgerEntryRequestData`) | Creates a debit or credit entry |
| Block (hold) a credit | `EntryRequestCommand` (entry = `BlockingEntryRequestData`) | Creates an entry whose credit is cautionary-blocked |
| Release part of a held amount | `EntryRequestCommand` (entry = `ReleaseEntryRequestData`) | Releases part of a previously blocked amount without fully unblocking it |
| Unblock a held credit | `EntryRequestCommand` (entry = `UnblockingEntryRequestData`) | Releases a cautionary block, optionally posting a new entry at the same time |
| Remove an entry | `EntryRequestCommand` (entry = `RemovalEntryRequestData`) | Deletes a previously created entry, identified by its correlation id |
| Reverse an entry | `EntryRequestCommand` (entry = `ReversalEntryRequestData`) | Posts a compensating entry that undoes a previous one |
| Exclude a pending entry | `EntryRequestCommand` (entry = `ExclusionEntryRequestData`) | Excludes (rolls back) a previously created **pending** entry — immediately or deferred |

**Coming soon:** combo commands let you submit several of these operations for one account **atomically, in a single message** — see [Combo Entry Commands](#combo-entry-commands).

---

## In What Order to Send Commands

| Constraint | Detail |
|---|---|
| **Account must already exist** | DTW does not create accounts from a command. If the account is not found, the command fails with an `AccountNotFoundException` and a failure result is returned via `replyTo` — see [Command Validation and Error Handling](#command-validation-and-error-handling). Publish `AccountChangeEvent` first (Events guide). |
| **Reversal/Removal targets must already exist** | `ReversalEntryRequestData.entryToRevert` and `RemovalEntryRequestData.entryToRemoval` reference another entry's `correlationId`. If that entry cannot be found, the command fails with an `EntryNotFoundException`. |
| **Unblocking `releasing.correlationId` must reference the original block** | Must match the `correlationId` supplied in that entry's own `uponCreationHeldAs` at creation time. |
| **History code registration** | *Inconclusive from source analysis* — no dedicated "history code not found" exception was found in the domain layer. Confirm with the DTW team whether an unregistered `historyCode` fails cleanly (e.g. a constraint violation) or is accepted; do not assume it is validated the same way `AccountNotFoundException`/`EntryNotFoundException` are. |
| **Duplicate `correlationId` is not an ordering hazard — it's handled as idempotent replay** | Re-sending a command with a `correlationId` that was already processed does not create a duplicate entry or error; see [Command Validation and Error Handling](#command-validation-and-error-handling). |

---

## Kafka Topic Naming Convention

Commands are published to a topic matched by this **pattern** (DTW's consumer subscribes via a Kafka `topicPattern` regex, not one fixed topic name):

```
[ENV].dtw.transaction.operations.*.command
```

| Component | Description |
|---|---|
| `[ENV]` | Environment prefix (e.g., `dev`, `uat`, `prod`) — provided by the DTW team |
| `operations.*.command` | Wildcard segment — DTW's single command listener subscribes to every topic matching this pattern, regardless of the specific subject in place of `*` (e.g. entry-related commands) |

> **Confirm the exact subject segment with the DTW team when provisioning your topic.** The pattern was confirmed from `dtw-transaction`'s Kafka listener configuration, but the literal subject string used for entry commands (analogous to `account`/`account-type`/`history-code` in the Events guide) was not independently verified against a running environment as part of this guide's research.

> **Current and combo commands share the same topic.** Diffing the unmerged combo branches against `develop` shows no new topic configuration for Combo commands — combo messages are expected to arrive on the same pattern as current commands.

**Reply topics are not fixed by DTW.** You choose the reply topic per command via `BaseCommand.replyTo` (and optionally `replyKey` for the message key). Digital Twin republishes the result — success or failure — to exactly that topic/key. If `replyTo` names a topic DTW cannot resolve or publish to, DTW logs an error internally and **drops the result** rather than retrying it — there is no mechanism to recover a lost reply, so treat `replyTo` correctness as your responsibility to validate before going live.

---

## What's Inside Every Command

Every command sent to Digital Twin wraps a `BaseCommand`, and every result Digital Twin sends back wraps a `BaseCommandSuccessResult` or `BaseCommandFailureResult`. These three types are common to all commands — understand them once, up front.

### `BaseCommand`

```json
{
  "baseEvent": {
    "content": {
      "source": "checking-account-system",
      "eventId": "550e8400-e29b-41d4-a716-446655441000",
      "createdAtUtc": 1719360000000,
      "kind": "entry-ledger-request-command"
    },
    "parents": []
  },
  "replyTo": "checking-account-system.dtw.entry.reply",
  "replyKey": null
}
```

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | — | Same header used by events — see the Events guide's "What's Inside Every Event" section. `content.kind` should describe the command, not an event (e.g. `"entry-ledger-request-command"`). |
| `replyTo` | `string` | Yes | Not found — not persisted in `dtw-transaction` | The Kafka topic DTW publishes the result to. **Not validated ahead of time** — an invalid value causes the result to be silently dropped (see [Kafka Topic Naming Convention](#kafka-topic-naming-convention)). |
| `replyKey` | `string`, nullable | No (default `null`) | Not found | The Kafka message key to use when publishing the result. If left `null`/blank, DTW falls back to the original command's routing key. |

### `AccountKey`

Identical to the Events guide's `AccountKey` — `branch` (int) + `account` (long), both required and must be positive.

### How Results Come Back

Every command produces exactly one of two outcomes, published to `replyTo`:

#### `BaseCommandSuccessResult`

```json
{
  "baseEvent": {
    "content": {
      "source": "dtw-transaction",
      "eventId": "550e8400-e29b-41d4-a716-446655442000",
      "createdAtUtc": 1719360000500,
      "kind": "entry-ledger-command-success-result"
    },
    "parents": [
      { "source": "checking-account-system", "eventId": "550e8400-e29b-41d4-a716-446655441000", "createdAtUtc": 1719360000000, "kind": "entry-ledger-request-command" }
    ]
  }
}
```

| Field | Type | Description |
|---|---|---|
| `baseEvent` | `BaseEvent` | Result's own header. `parents` links back to the originating command's `BaseEventContent` — this is how you correlate a result to the command that produced it if you don't already track it by `correlationId`/`replyKey`. |

Every command-specific success result (see the catalogs below) wraps this plus the created/affected entry's data.

#### `BaseCommandFailureResult` / `FailureInfo`

```json
{
  "baseEvent": { "content": { "...": "..." }, "parents": [ "..." ] },
  "failureInfo": {
    "type": "http://dtw.matera.com/transaction/account-not-found",
    "title": "Account Not Found",
    "details": "Could not find account [AccountKey(branch=1, account=987654321)]",
    "parameters": null
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `baseEvent` | `BaseEvent` | Yes | Result header, same lineage mechanism as the success result |
| `failureInfo` | `FailureInfo` | Yes | Structured error data, loosely modeled on **RFC 7807** |
| `failureInfo.type` | `string` (URI) | Yes | Identifies the problem type. Defaults to `http://dtw.matera.com/transaction/generic-error` for exceptions Digital Twin doesn't recognize as a specific business rule |
| `failureInfo.title` | `string` | Yes | Short, human-readable summary (e.g. `"Account Not Found"`) |
| `failureInfo.details` | `string`, nullable | No (default `null`) | Human-readable explanation specific to this occurrence |
| `failureInfo.parameters` | `string` (json), nullable | No (default `null`) | Additional structured context about the failure |

See [Command Validation and Error Handling](#command-validation-and-error-handling) for the concrete `type`/`title` values Digital Twin produces today and what triggers each.

---

## Entry Commands

**Topic:** `[ENV].dtw.transaction.operations.*.command` (see [Kafka Topic Naming Convention](#kafka-topic-naming-convention))

**Direction:** Core Banking / Upstream System → DTW Transaction, with an asynchronous reply to `replyTo`

Every command uses the same envelope, `EntryRequestCommand`, wrapping **exactly one** entry-kind operation. To request multiple operations, publish multiple `EntryRequestCommand` messages.

> **Kafka poll batching is not the same as atomic submission.** Internally, `dtw-transaction` groups commands that land in the same Kafka consumer poll for the same account into one database transaction as a throughput optimization. This can incidentally make two separately-sent commands commit together, but it is not something a producer can request or rely on — if you need guaranteed atomicity across multiple entries, see the coming-soon [Combo Entry Commands](#combo-entry-commands).

### EntryRequestCommand

#### Conceptual Overview

The single command envelope for all entry operations. `entry` is a union of seven possible payload types — DTW dispatches based on which one you send.

#### Field Reference

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `baseCommand` | `BaseCommand` | Yes | — | See [What's Inside Every Command](#whats-inside-every-command) |
| `account` | `AccountKey` | Yes | — | The account the entry applies to |
| `entry` | union of `LedgerEntryRequestData` \| `BlockingEntryRequestData` \| `ExclusionEntryRequestData` \| `ReleaseEntryRequestData` \| `RemovalEntryRequestData` \| `ReversalEntryRequestData` \| `UnblockingEntryRequestData` | Yes | each per its section below | The one operation this command requests |

#### Avro Record Name

`com.matera.dtw.command.transaction.entry.EntryRequestCommand`

---

### Ledger Entry

#### Conceptual Overview

Posts a debit, credit, or balance entry to the account. This is the most common command — a payment received, a fee charged, an interest credit.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "checking-account-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441001",
        "createdAtUtc": 1719360000000,
        "kind": "entry-ledger-request-command"
      },
      "parents": []
    },
    "replyTo": "checking-account-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "historyCode": 101,
    "status": "POSTED",
    "balanceUpdateStrategy": "CONDITIONAL",
    "asOfDate": 20609,
    "source": "channel",
    "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
    "settlement": {
      "fulfillment": "TOTAL",
      "amount": { "currency": "USD", "amount": 1485 }
    },
    "description": "FedNow received",
    "metaData": null,
    "uponCreationHeldAs": null
  }
}
```

#### Field Reference — `LedgerEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `historyCode` | `int` | Yes | N/A (int in Avro) — ⚠ resolves to a `String` business key (`HistoryCodeKey`, VARCHAR 64, `@NotNull`) downstream; confirm the code→key mapping with the DTW team | Classifies the entry as debit/credit/balance |
| `status` | enum `EntryStatus` | No (default `POSTED`) | N/A | `POSTED` or `PENDING` |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | `CONDITIONAL` or `UNCONDITIONAL` |
| `asOfDate` | `int` (date) | Yes | N/A | Date the entry happened |
| `source` | `string` | Yes | **64** — `DTW_TR_CORRELATION_ID.ORIGINATOR` / `DTW_TR_ENTRY_BATCH.ORIGINATOR` | Which system requested the entry |
| `correlationId` | `string` | Yes | **128** — `DTW_TR_CORRELATION_ID.CORRELATION_ID`, `@NotNull` | Uniquely identifies this entry; re-sending the same value is treated as idempotent replay, not a duplicate |
| `settlement` | union `TotalSettlementInfo` \| `PartialSettlementInfo` | Yes | — | Total or partial amount |
| `description` | `string`, nullable | No (default `null`) | **40** — `DTW_TR_ENTRY.DESCRIPTION` | Appears on the customer statement. Must not contain personal data |
| `metaData` | `string` (json), nullable | No (default `null`) | Not found — column type is JSON/JSONB/CLOB depending on RDBMS | Free-form metadata |
| `uponCreationHeldAs` | `CautionaryBlockInformation`, nullable | No (default `null`) | — | Only used when `status = ON_HOLD` on a credit |

#### Success Result — `EntryLedgerCommandSuccessResult`

| Field | Type | Description |
|---|---|---|
| `baseResult` | `BaseCommandSuccessResult` | — |
| `entry` | `LedgerEntryResultData` | The newly-created entry: `id` (uuid), `account`, `kind` (default `"LEDGER"`), `historyCode`, `status`, `balanceUpdateStrategy`, `asOfDate`, `correlations` (map), `amount`, `description`, `metaData`, `balanceChanges` (map), `initiator`, `uponCreationHeldAs` |
| `correlationId` | `string`, nullable | The correlation id of the created entry |

#### Failure Result — `EntryLedgerCommandFailureResult`

| Field | Type | Description |
|---|---|---|
| `baseResult` | `BaseCommandFailureResult` | Carries `FailureInfo` — see [Command Validation and Error Handling](#command-validation-and-error-handling) |
| `account` | `AccountKey` | — |
| `entry` | `LedgerEntryRequestData` | The rejected request, echoed back in full |
| `correlationId` | `string`, nullable | — |

#### Avro Record Names

`com.matera.dtw.command.transaction.command.LedgerEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryLedgerCommandSuccessResult` · `...EntryLedgerCommandFailureResult`

---

### Blocking Entry

#### Conceptual Overview

Creates an entry whose credit is placed under a **cautionary block** (a hold) instead of being immediately available — for example, a pre-authorization on a card.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "card-authorization-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441002",
        "createdAtUtc": 1719360000000,
        "kind": "entry-blocking-request-command"
      },
      "parents": []
    },
    "replyTo": "card-authorization-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "source": "CARD",
    "correlationId": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
    "metaData": null,
    "description": "Card Pre-Authorization hold",
    "status": "ON_HOLD",
    "asOfDate": 20609,
    "settlement": { "fulfillment": "TOTAL", "amount": { "currency": "USD", "amount": 5000 } },
    "holdReasonId": "PRE_AUTH",
    "balanceUpdateStrategy": "CONDITIONAL"
  }
}
```

#### Field Reference — `BlockingEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | **64** | Which system requested the entry |
| `correlationId` | `string` | Yes | **128** | Uniquely identifies this entry |
| `metaData` | `string` (json), nullable | No (default `null`) | Not found | Free-form metadata |
| `description` | `string`, nullable | No (default `null`) | **40** | Statement description |
| `status` | enum | No (default `POSTED`) | N/A | Typically `ON_HOLD` for a block |
| `asOfDate` | `int` (date) | Yes | N/A | — |
| `settlement` | union | Yes | — | Total or partial amount to block |
| `holdReasonId` | `string`, nullable | No | **64** — `DTW_TR_BLOCKING_RELATED_ENTRY.HOLD_REASON_ID` | Reason code for the hold — should reference a Hold Reason configured via the Events guide's Generic Domain events |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |

#### Success Result — `EntryBlockingCommandSuccessResult`

`baseResult` + `id` + `account` + `balanceChanges` (map) + `correlationId` + `metaData` + `description` + `status` + `amount` + `holdReasonId` + `source` + `balanceUpdateStrategy` + `initiator`.

#### Failure Result — `EntryBlockingCommandFailureResult`

`baseResult` + `account` + `correlationId` + `metaData` + `description` + `status` + `balanceUpdateStrategy` + `settlement` (echoed) + `holdReasonId` + `source`.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.BlockingEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryBlockingCommandSuccessResult` · `...EntryBlockingCommandFailureResult`

---

### Unblocking Entry

#### Conceptual Overview

Releases a cautionary block. `releasing.correlationId` must reference the `correlationId` originally supplied in the blocked entry's `uponCreationHeldAs`. This record can also post a fresh cautionary-blocked entry in the same call (`uponCreationHeldAs`) — allowing "unblock A, immediately re-block as B" in one command.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "card-authorization-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441003",
        "createdAtUtc": 1719360100000,
        "kind": "entry-unblocking-request-command"
      },
      "parents": []
    },
    "replyTo": "card-authorization-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "source": "CARD",
    "correlationId": "b2c3d4e5-f6a7-4b1c-8d2e-1234567890ac",
    "metaData": null,
    "description": "Card Pre-Authorization Release",
    "status": "POSTED",
    "asOfDate": 20609,
    "settlement": { "fulfillment": "TOTAL", "amount": { "currency": "USD", "amount": 5000 } },
    "holdReasonId": null,
    "balanceUpdateStrategy": "CONDITIONAL",
    "uponCreationHeldAs": null,
    "releasing": { "correlationId": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab" }
  }
}
```

#### Field Reference — `UnblockingEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | **64** | — |
| `correlationId` | `string` | Yes | **128** | Correlation id of this unblocking entry itself |
| `metaData` | `string` (json), nullable | No | Not found | — |
| `description` | `string`, nullable | No | **40** | — |
| `status` | enum | No (default `POSTED`) | N/A | — |
| `asOfDate` | `int` (date) | Yes | N/A | — |
| `settlement` | union | Yes | — | — |
| `holdReasonId` | `string`, nullable | No | **64** | — |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |
| `uponCreationHeldAs` | `CautionaryBlockInformation`, nullable | No | — | Only when re-blocking in the same call |
| `releasing` | `CautionaryUnblockInformation`, nullable | No | — | `releasing.correlationId` must match the target block's own `correlationId` |

#### Success / Failure Results

`EntryUnblockingCommandSuccessResult` / `EntryUnblockingCommandFailureResult` — same field shape pattern as Blocking, plus `uponCreationHeldAs` and `releasing` echoed back.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.UnblockingEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryUnblockingCommandSuccessResult` · `...EntryUnblockingCommandFailureResult`

---

### Removal Entry

#### Conceptual Overview

Deletes a previously created entry, identified by `entryToRemoval` (its `correlationId`). The target entry must already exist — see [In What Order to Send Commands](#in-what-order-to-send-commands).

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "checking-account-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441004",
        "createdAtUtc": 1719360200000,
        "kind": "entry-removal-request-command"
      },
      "parents": []
    },
    "replyTo": "checking-account-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "source": "channel",
    "correlationId": "c3d4e5f6-a7b8-4c1d-9e2f-1234567890ad",
    "description": "Reversal of an Incorrect Entry",
    "entryToRemoval": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
    "metaData": null,
    "asOfDate": 20609,
    "balanceUpdateStrategy": "CONDITIONAL",
    "status": "POSTED"
  }
}
```

#### Field Reference — `RemovalEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | **64** | — |
| `correlationId` | `string` | Yes | **128** | Correlation id of this removal entry |
| `description` | `string`, nullable | No | **40** | — |
| `entryToRemoval` | `string` | Yes | **128** — references the target entry's `correlationId` column | The entry being removed |
| `metaData` | `string` (json), nullable | No | Not found | — |
| `asOfDate` | `int` (date) | Yes | N/A | — |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |
| `status` | enum | No (default `POSTED`) | N/A | — |

#### Success / Failure Results

`EntryRemovalCommandSuccessResult` (adds `id`, `balanceChanges`, `initiator`) / `EntryRemovalCommandFailureResult`.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.RemovalEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryRemovalCommandSuccessResult` · `...EntryRemovalCommandFailureResult`

---

### Reversal Entry

#### Conceptual Overview

Posts a compensating entry that undoes a previous one, identified by `entryToRevert`. Unlike Removal (which deletes), Reversal adds a new offsetting entry — the original remains on the statement.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "checking-account-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441005",
        "createdAtUtc": 1719360300000,
        "kind": "entry-reversal-request-command"
      },
      "parents": []
    },
    "replyTo": "checking-account-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "source": "channel",
    "correlationId": "d4e5f6a7-b8c9-4d1e-9f2a-1234567890ae",
    "entryToRevert": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
    "asOfDate": 20609,
    "description": "Refund at the Customer's Requeste",
    "metaData": null,
    "balanceUpdateStrategy": "CONDITIONAL",
    "status": "POSTED"
  }
}
```

#### Field Reference — `ReversalEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | **64** | — |
| `correlationId` | `string` | Yes | **128** | Correlation id of the reversal entry itself |
| `entryToRevert` | `string` | Yes | **128** — references the target entry's `correlationId` column | The entry being reversed |
| `asOfDate` | `int` (date) | Yes | N/A | — |
| `description` | `string`, nullable | No | **40** | — |
| `metaData` | `string` (json), nullable | No | Not found | — |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |
| `status` | enum | No (default `POSTED`) | N/A | — |

> **History code compatibility check:** reversing an entry validates the reversal's history code against the original entry's configured reversal code (`IllegalReversalHistoryCodeException` if mismatched) — this is a business rule, not a raw existence check.

#### Success / Failure Results

`EntryReversalCommandSuccessResult` (adds `id`, `balanceChanges`, `initiator`) / `EntryReversalCommandFailureResult`.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.ReversalEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryReversalCommandSuccessResult` · `...EntryReversalCommandFailureResult`

---

### Exclusion Entry

#### Conceptual Overview

Excludes (rolls back) a **pending** entry identified by `correlationId`. Exclusion applies specifically to entries in `PENDING` status — it is not used for entries that have already been posted. Unlike Removal, exclusion can be **deferred** rather than immediate — the success result's `status` field (`ResultStatus`: `IMMEDIATE` | `DEFERRED`) tells you which happened. Bulk exclusion for combo scenarios is handled by the coming-soon `ComboEntryExclusionRequestCommand` (see [Combo Entry Commands](#combo-entry-commands)).

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "checking-account-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441006",
        "createdAtUtc": 1719360400000,
        "kind": "entry-exclusion-request-command"
      },
      "parents": []
    },
    "replyTo": "checking-account-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "source": "channel",
    "balanceUpdateStrategy": "CONDITIONAL",
    "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10"
  }
}
```

#### Field Reference — `ExclusionEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `source` | `string` | Yes | **64** | Which system created the entry being excluded |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |
| `correlationId` | `string` | Yes | **128** | Correlation id of the entry to exclude |

#### Success Result — `EntryExclusionCommandSuccessResult`

`baseResult` + `account` + `source` + `correlationId` + `status` (`ResultStatus`: `IMMEDIATE`/`DEFERRED`) + `balanceUpdateStrategy` + `entry` (nullable `ExclusionEntry`, populated only if `status = IMMEDIATE`).

#### Failure Result — `EntryExclusionCommandFailureResult`

`baseResult` + `account` + `source` + `correlationId`.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.ExclusionEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryExclusionCommandSuccessResult` · `...EntryExclusionCommandFailureResult`

---

### Release Entry

#### Conceptual Overview

Releases part of a previously blocked amount without fully unblocking the entry — a partial capture. No combo equivalent is currently planned; flag with the DTW team if you need batch release support.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "card-authorization-system",
        "eventId": "550e8400-e29b-41d4-a716-446655441007",
        "createdAtUtc": 1719360500000,
        "kind": "entry-release-request-command"
      },
      "parents": []
    },
    "replyTo": "card-authorization-system.dtw.entry.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "correlationId": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
    "balanceChanges": {},
    "balanceUpdateStrategy": "CONDITIONAL",
    "amount": { "currency": "USD", "amount": 2000 }
  }
}
```

#### Field Reference — `ReleaseEntryRequestData`

| Field | Type | Required | Size | Description |
|---|---|---|---|---|
| `correlationId` | `string` | Yes | **128** (assumed same correlation-id column as other kinds — confirm before relying on it) | Correlation id of the blocked entry being partially released |
| `balanceChanges` | `map<string, FinancialAmount>` | No (default `{}`) | — | Delta values changed in the account balances; may be empty |
| `balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A | — |
| `amount` | `FinancialAmount` | Yes | N/A (numeric) | Amount being released |

#### Success Result — `EntryReleaseCommandSuccessResult`

`baseResult` + `id` (nullable uuid) + `account` + `source` + `correlationId` + `balanceUpdateStrategy` + `initiator`.

#### Failure Result — `EntryReleaseCommandFailureResult`

`baseResult` + `account` + `source` + `correlationId`.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.ReleaseEntryRequestData` · `com.matera.dtw.command.transaction.entry.EntryReleaseCommandSuccessResult` · `...EntryReleaseCommandFailureResult`

---

## Combo Entry Commands

> **Coming soon.** Everything in this section describes the schema as designed and the behavior expected by analogy with the shared validation/error-handling layer (see [Command Validation and Error Handling](#command-validation-and-error-handling)).

**Topic:** expected to share the same pattern as current commands — `[ENV].dtw.transaction.operations.*.command` 

**Direction:** Core Banking / Upstream System → DTW Transaction, with an asynchronous reply to `replyTo`

The key difference from `EntryRequestCommand`: `ComboEntryRequestCommand` carries an **array** of entry-kind operations (`entries[]`) for one account, processed **atomically** as a single "combo" — either all entries in the array are committed together, or none are. This is the producer-guaranteed atomicity that the current command's opportunistic Kafka-poll batching cannot offer.

### ComboEntryRequestCommand

#### Conceptual Overview

Requests one or more entry operations — Ledger, Blocking, Removal, Reversal, Unblocking — against a single account, atomically. Note the `entries[]` union covers **five** kinds; `Exclusion` and `Release` are not part of it. Bulk exclusion uses the separate `ComboEntryExclusionRequestCommand` below instead.

Each combo entry-kind payload record (`LedgerEntryRequestData`, `BlockingEntryRequestData`, `RemovalEntryRequestData`, `ReversalEntryRequestData`, `UnblockingEntryRequestData`, all in the `com.matera.dtw.command.transaction.command.v2` namespace) matches its current counterpart field-for-field, **plus one addition**: an optional `deferredUntil` field that lets an entry's execution be deferred until a related entry reaches a given phase (see `DeferredUntil` in the Appendix). Field sizes are identical to the tables above — they map to the same persisted columns.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "digital-account-core",
        "eventId": "8f14e45f-ceea-467e-bd6c-2b0f5c1e2a1a",
        "createdAtUtc": 1783440000000,
        "kind": "combo-entry-request-command"
      },
      "parents": []
    },
    "replyTo": "dtw.transaction.combo-entry-request.reply",
    "replyKey": null
  },
  "account": { "branch": 1, "account": 123456789 },
  "entries": [
    {
      "historyCode": 101,
      "status": "POSTED",
      "balanceUpdateStrategy": "CONDITIONAL",
      "asOfDate": 20609,
      "source": "channel",
      "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
      "settlement": { "fulfillment": "TOTAL", "amount": { "currency": "USD", "amount": 1485 } },
      "description": "FedNow received",
      "metaData": null,
      "uponCreationHeldAs": null,
      "deferredUntil": null
    },
    {
      "source": "CARD",
      "correlationId": "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab",
      "metaData": null,
      "description": "Pre-authorization card hold",
      "status": "ON_HOLD",
      "asOfDate": 20609,
      "settlement": { "fulfillment": "TOTAL", "amount": { "currency": "USD", "amount": 5000 } },
      "holdReasonId": "PRE_AUTH",
      "balanceUpdateStrategy": "CONDITIONAL",
      "deferredUntil": null
    }
  ]
}
```

#### Field Reference

| Field | Type | Required | Size |
|---|---|---|---|
| `baseCommand` | `BaseCommand` | Yes | see [What's Inside Every Command](#whats-inside-every-command) |
| `account` | `AccountKey` | Yes | see above |
| `entries` | array of union (`LedgerEntryRequestData` \| `BlockingEntryRequestData` \| `RemovalEntryRequestData` \| `ReversalEntryRequestData` \| `UnblockingEntryRequestData`, all in the `v2` namespace) | Yes (no default) | each item per its equivalent table above, plus optional `deferredUntil` |

#### Success Result — `ComboEntryRequestCommandSuccessResult`

| Field | Type | Description |
|---|---|---|
| `baseResult` | `BaseCommandSuccessResult` | — |
| `account` | `AccountKey` | — |
| `entries` | array of `ComboEntryResponseData` | One result per submitted entry — see below |
| `balances` | `map<string, FinancialAmount>` | Account balances after the combo was applied |

**`ComboEntryResponseData`** — per-entry result: `id` (nullable uuid), `entryKind` (string, e.g. `"LEDGER"`, `"BLOCKING"`), `historyCode` (nullable int), `holdReasonId`, `status` (`ResultEntryStatus`: `POSTED`/`PENDING`/`DEFERRED`/`ON_HOLD`), `description`, `asOfDate`, `correlations` (map), `initiator`, `balanceChanges` (map), `amount`, `metaData`.

#### Failure Result — `ComboEntryRequestCommandFailureResult`

`baseResult` (carries the `FailureInfo` for whichever entry/entries caused the combo to fail) + `account` + `entries[]` (the original rejected array, echoed back in full).

#### Avro Record Names

`com.matera.dtw.command.transaction.command.v2.ComboEntryRequestCommand` · `...ComboEntryRequestCommandSuccessResult` · `...ComboEntryRequestCommandFailureResult` · `...ComboEntryResponseData`

---

### ComboEntryExclusionRequestCommand

#### Conceptual Overview

Requests the exclusion (rollback) of a previously-created combo of entries, identified by their correlation ids — the combo counterpart to the per-entry `ExclusionEntryRequestData`, generalized to a batch of correlation ids under one account.

#### Payload

```json
{
  "baseCommand": {
    "baseEvent": {
      "content": {
        "source": "digital-account-core",
        "eventId": "0c9d8b7a-1234-4a5b-9c0d-abcdef123456",
        "createdAtUtc": 1783440300000,
        "kind": "combo-entry-exclusion-request-command"
      },
      "parents": []
    },
    "replyTo": "dtw.transaction.combo-entry-exclusion-request.reply",
    "replyKey": null
  },
  "exclusionBaseCommand": {
    "account": { "branch": 1, "account": 123456789 },
    "balanceUpdateStrategy": "CONDITIONAL",
    "correlations": [
      "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
      "a1b2c3d4-e5f6-4a1b-8c2d-1234567890ab"
    ]
  }
}
```

#### Field Reference

| Field | Type | Required | Size |
|---|---|---|---|
| `baseCommand` | `BaseCommand` | Yes | see above |
| `exclusionBaseCommand.account` | `AccountKey` | Yes | see above |
| `exclusionBaseCommand.balanceUpdateStrategy` | enum | No (default `CONDITIONAL`) | N/A |
| `exclusionBaseCommand.correlations` | array\<string\> | Yes (no default) | each item **128** — `DTW_TR_CORRELATION_ID.CORRELATION_ID` |

#### Success / Failure Results

`ComboEntryExclusionRequestCommandSuccessResult` / `...FailureResult` — both simply wrap `baseResult` + the `exclusionBaseCommand` echoed back.

#### Avro Record Names

`com.matera.dtw.command.transaction.command.v2.ComboEntryExclusionRequestCommand` · `...ComboEntryExclusionBaseCommand` · `...ComboEntryExclusionRequestCommandSuccessResult` · `...ComboEntryExclusionRequestCommandFailureResult`

---

## Command Validation and Error Handling

This section describes how Digital Twin responds when a command contains invalid data, references something that doesn't exist, or fails a business rule. Each command kind has its own set of validations — see the per-command subsections below.

**Unlike events, commands have no retry topic or Dead Letter Queue.** Outcomes are communicated synchronously (in Kafka terms — i.e., within the same processing cycle) via the result published to `replyTo`. Only a genuinely unrecognized/malformed message falls back to Kafka's own consumer redelivery, since there is no dedicated infrastructure to catch it.

### Command Processing Outcomes

| Symbol | Meaning |
|---|---|
| 🟢 **SUCCESS** | The command is accepted, executed, and a success result is published to `replyTo` |
| 🟡 **WARNING** | The command was already processed (idempotent replay); a success result is published to `replyTo`, but no new entry is created — the `entry` field in the result is `null` |
| 🔴 **FAILURE** | A business-rule failure occurs; a `BaseCommandFailureResult` with `FailureInfo` is published to `replyTo` |
| ⚫ **INVALID OR UNRECOGNIZED MESSAGE** | The message could not be parsed as a valid Digital Twin command. No command result is returned. The message is handled by the underlying Kafka consumer rather than Digital Twin's command processing logic.|

---

### What Happens When a Command Fails?

**Central mechanism.** Digital Twin processes commands that arrive in the same Kafka poll as one batch. If executing the batch throws an exception, Digital Twin does not fail the whole batch — it recursively **bisects** the batch to isolate exactly which command(s) caused the exception, down to a single command if necessary, and produces a targeted failure result only for the offending command(s). Business-rule failures are logged as warnings; anything Digital Twin doesn't recognize as a specific business rule is logged as an error internally (still producing a failure result with the generic `http://dtw.matera.com/transaction/generic-error` type, rather than crashing).

**`FailureInfo` structure.** Every failure result published to `replyTo` includes a `failureInfo` object:

| Field | Type | Description |
|---|---|---|
| `type` | `string` (URI) | Identifies the problem type (e.g. `http://dtw.matera.com/transaction/account-not-found`) |
| `title` | `string` | Short, human-readable summary (e.g. `"Account Not Found"`) |
| `details` | `string`, nullable | Human-readable explanation specific to this occurrence |
| `parameters` | `string` (JSON), nullable | Structured context — account key, balance amounts, conflicting correlation IDs, etc. |

#### Example Failure Results Published to `replyTo`

The examples below show what your consumer receives on the `replyTo` topic for common failure scenarios. The Avro record type on the message identifies which command kind failed (e.g. `EntryLedgerCommandFailureResult`) — your consumer uses `TopicRecordNameStrategy` to distinguish it from a success result on the same topic.

**Account not found:**

```json
{
  "baseResult": {
    "baseEvent": {
      "content": {
        "source": "dtw-transaction",
        "eventId": "7a3f1c2e-9b4d-4e0a-8c6f-112233445566",
        "createdAtUtc": 1719360001000,
        "kind": "entry-ledger-command-failure-result"
      },
      "parents": [
        {
          "source": "checking-account-system",
          "eventId": "550e8400-e29b-41d4-a716-446655441001",
          "createdAtUtc": 1719360000000,
          "kind": "entry-ledger-request-command"
        }
      ]
    },
    "failureInfo": {
      "type": "http://dtw.matera.com/transaction/account-not-found",
      "title": "Account Not Found",
      "details": "Could not find account [AccountKey(branch=1, account=987654321)]",
      "parameters": null
    }
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": {
    "historyCode": 101,
    "status": "POSTED",
    "balanceUpdateStrategy": "CONDITIONAL",
    "asOfDate": 20609,
    "source": "channel",
    "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10",
    "settlement": { "fulfillment": "TOTAL", "amount": { "currency": "USD", "amount": 1485 } },
    "description": "FedNow received",
    "metaData": null,
    "uponCreationHeldAs": null
  },
  "correlationId": "3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10"
}
```

**Insufficient funds:**

```json
{
  "baseResult": {
    "baseEvent": {
      "content": {
        "source": "dtw-transaction",
        "eventId": "9c1e3a5b-2d4f-4b8c-a7e0-aabbccddeeff",
        "createdAtUtc": 1719360001500,
        "kind": "entry-ledger-command-failure-result"
      },
      "parents": [
        {
          "source": "checking-account-system",
          "eventId": "550e8400-e29b-41d4-a716-446655441002",
          "createdAtUtc": 1719360001000,
          "kind": "entry-ledger-request-command"
        }
      ]
    },
    "failureInfo": {
      "type": "http://dtw.matera.com/transaction/insufficient-funds",
      "title": "Insufficient funds",
      "details": "Account [AccountKey(branch=1, account=987654321)] does not have sufficient funds for this operation.",
      "parameters": "{\"availableBalance\": \"50.00\", \"requiredAmount\": \"200.00\", \"currency\": \"USD\"}"
    }
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": { "...": "echoed request fields" },
  "correlationId": "550e8400-e29b-41d4-a716-446655441002"
}
```

**Entry not found (Reversal):**

```json
{
  "baseResult": {
    "baseEvent": {
      "content": {
        "source": "dtw-transaction",
        "eventId": "b4d2f6a8-3e5c-4a7b-9f1d-001122334455",
        "createdAtUtc": 1719360002000,
        "kind": "entry-reversal-command-failure-result"
      },
      "parents": [
        {
          "source": "checking-account-system",
          "eventId": "550e8400-e29b-41d4-a716-446655441005",
          "createdAtUtc": 1719360300000,
          "kind": "entry-reversal-request-command"
        }
      ]
    },
    "failureInfo": {
      "type": "http://dtw.matera.com/transaction/entry-not-found",
      "title": "Entry Not Found",
      "details": "Could not find entry with correlationId [3f9c6a10-6e2b-4b0a-9b2a-6e6f2f6b6a10] for account [AccountKey(branch=1, account=987654321)]",
      "parameters": null
    }
  },
  "account": { "branch": 1, "account": 987654321 },
  "entry": { "...": "echoed request fields" },
  "correlationId": "d4e5f6a7-b8c9-4d1e-9f2a-1234567890ae"
}
```

> Key points for consuming failure results:
> - `baseResult.baseEvent.parents[0].eventId` equals the original command's `baseEvent.content.eventId` — use this to correlate the failure back to the request that caused it.
> - The original request payload is echoed back in the `entry` (or `entries[]`) field — you don't need to maintain a local copy to know which request failed.
> - `failureInfo.parameters` is a JSON string (not a nested object) containing structured context that varies by error type — parse it separately if you need the values programmatically.

> **`replyTo` failures are invisible to you.** If the topic you specified is invalid, Digital Twin logs an error on its side and drops the result — you will not receive a failure result telling you your own `replyTo` was wrong. Validate your reply-topic configuration in a lower environment before relying on it in production.

---

### Idempotency Behavior

Digital Twin treats a re-sent command that carries an already-processed `correlationId` as an **idempotent replay** — not an error. This makes it safe to retry a command after a timeout when you don't know whether the first attempt succeeded.

| Scenario | `correlationId` state | Outcome |
|---|---|---|
| **Idempotent replay** — same command resent with the same `correlationId` for the same account | Already persisted for that account/type | 🟡 WARNING — a success result is published; no new entry is created; the `entry` field in the result is `null` |
| **Correlation ID conflict** — same `correlationId` used by a different entity (different account or operation type) | Already persisted by a different holder | 🔴 FAILURE — `http://dtw.matera.com/transaction/correlation-conflict` / "Correlation conflict" |
| **Intra-combo duplicate** (combo only) — two entries in the same `entries[]` share the same `correlationId` | — | The later duplicate is silently removed before execution; no failure result is generated for it |

---

### Ledger Entry — Validations and Error Responses

Applies to `LedgerEntryRequestData` when submitted as an individual command or as part of a Combo Entry Command (coming soon).


| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist in DTW | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account exists but is inactive (`active = false`) | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Account is temporarily unavailable (lock conflict) | `…/account-temporarily-unavailable` | Account temporarily unavailable | 🔴 FAILURE |
| `historyCode` does not resolve to a configured entry type | `…/entry-type-not-found` | Entry type not found | 🔴 FAILURE |
| Entry currency does not match account currency | `…/currency-mismatch` | Currency not allowed | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and account balance is insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| Account feature flags block the operation (debit/credit blocked) | `…/operation-not-allowed` | Operation not allowed | 🔴 FAILURE |
| No balance configuration exists for this `historyCode` on this account type | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Settlement type incompatible with entry kind or strategy | `…/invalid-settlement` | Invalid settlement | 🔴 FAILURE |
| `correlationId` already used by a different entity | `…/correlation-conflict` | Correlation conflict | 🔴 FAILURE |
| `status = ON_HOLD` but `uponCreationHeldAs.holdReasonId` is missing | `…/missing-hold-reason` | Missing hold reason | 🔴 FAILURE |
| `uponCreationHeldAs` references a debit entry (cautionary blocks require credits) | `…/invalid-cautionary-block-target/debit-based-entry` | Invalid cautionary block target | 🔴 FAILURE |
| `uponCreationHeldAs` target entry is already cautionary-blocked | `…/invalid-cautionary-block-target/target-already-blocked` | Invalid cautionary block target | 🔴 FAILURE |
| `uponCreationHeldAs` target entry is not in POSTED status | `…/invalid-cautionary-block-target/invalid-status` | Invalid cautionary block target | 🔴 FAILURE |
| `uponCreationHeldAs.correlationId` conflicts with this entry's own `correlationId` | `…/invalid-cautionary-block` | Invalid cautionary block | 🔴 FAILURE |
| Pattern-based business rule validation failure | `…/generic-validation-failure` *(or rule-specific URI)* | Generic Validation Failure *(or rule-defined)* | 🔴 FAILURE |
| Same `correlationId` already processed for this account | — | — | 🟡 WARNING |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

> All `failureInfo.type` URIs use the base `http://dtw.matera.com/transaction/` prefix, abbreviated above as `…/`.

---

### Blocking Entry — Validations and Error Responses

Applies to all Blocking Entry requests, whether submitted individually or as part of a Combo Entry Command (coming soon).

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account is inactive | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Account is temporarily unavailable | `…/account-temporarily-unavailable` | Account temporarily unavailable | 🔴 FAILURE |
| `historyCode` does not resolve to a configured entry type | `…/entry-type-not-found` | Entry type not found | 🔴 FAILURE |
| Entry currency does not match account currency | `…/currency-mismatch` | Currency not allowed | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance is insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| Account feature flags block the operation | `…/operation-not-allowed` | Operation not allowed | 🔴 FAILURE |
| No balance configuration for this `historyCode` | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Settlement type incompatible with entry kind | `…/invalid-settlement` | Invalid settlement | 🔴 FAILURE |
| `correlationId` already used by a different entity | `…/correlation-conflict` | Correlation conflict | 🔴 FAILURE |
| `holdReasonId` provided but does not exist in DTW registry | `…/hold-reason-not-found` | Hold reason not found | 🔴 FAILURE |
| `holdReasonId` is not allowed for the given entry status | `…/hold-reason-not-allowed` | Hold reason not allowed | 🔴 FAILURE |
| Pattern-based business rule failure | `…/generic-validation-failure` | Generic Validation Failure | 🔴 FAILURE |
| Same `correlationId` already processed | — | — | 🟡 WARNING |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Unblocking Entry — Validations and Error Responses

Applies to all Unblocking Entry requests, whether submitted individually or as part of a Combo Entry Command (coming soon).

> `releasing.correlationId` must reference the `correlationId` of the cautionary block being released. Digital Twin validates the entire cautionary block chain before processing the request (see the conditions below).


| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account is inactive | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Account is temporarily unavailable | `…/account-temporarily-unavailable` | Account temporarily unavailable | 🔴 FAILURE |
| The entry referenced by `releasing.correlationId` does not exist | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| Target entry is a debit (cautionary blocks only apply to credits) | `…/invalid-cautionary-block-target/debit-based-entry` | Invalid cautionary block target | 🔴 FAILURE |
| Target entry is already unblocked | `…/invalid-cautionary-block-target/target-already-unblocked` | Invalid cautionary block target | 🔴 FAILURE |
| Target entry is already cautionary-blocked by another block | `…/invalid-cautionary-block-target/target-already-blocked` | Invalid cautionary block target | 🔴 FAILURE |
| Target entry is already compensated (removed/reversed) | `…/invalid-cautionary-block-target/target-already-compensated` | Invalid cautionary block target | 🔴 FAILURE |
| Target entry is not in POSTED status | `…/invalid-cautionary-block-target/invalid-status` | Invalid cautionary block target | 🔴 FAILURE |
| Target entry kind cannot be unblocked | `…/invalid-cautionary-block-target/invalid-entry-kind` | Invalid cautionary block target | 🔴 FAILURE |
| Target's `correlationId` matches the cautionary block's own `correlationId` (self-reference) | `…/invalid-cautionary-block-target/invalid-correlation-id` | Invalid cautionary block | 🔴 FAILURE |
| Blocking entry has no `correlationId` stored (required for the releasing reference) | `…/invalid-cautionary-block-target/missing-cautionary-block-correlation-id` | Missing correlation ID of cautionary block entry | 🔴 FAILURE |
| Amount or `holdReasonId` mismatch between the blocking and unblocking entries | `…/cautionary-block-parameter-mismatch` | Cautionary block parameter mismatch | 🔴 FAILURE |
| `holdReasonId` does not exist in DTW registry | `…/hold-reason-not-found` | Hold reason not found | 🔴 FAILURE |
| `holdReasonId` is not allowed for the given status | `…/hold-reason-not-allowed` | Hold reason not allowed | 🔴 FAILURE |
| `status = ON_HOLD` but `uponCreationHeldAs.holdReasonId` is missing | `…/missing-hold-reason` | Missing hold reason | 🔴 FAILURE |
| `historyCode` does not resolve to a configured entry type | `…/entry-type-not-found` | Entry type not found | 🔴 FAILURE |
| Entry currency does not match account currency | `…/currency-mismatch` | Currency not allowed | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| Account feature flags block the operation | `…/operation-not-allowed` | Operation not allowed | 🔴 FAILURE |
| No balance configuration for this `historyCode` | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Settlement type incompatible | `…/invalid-settlement` | Invalid settlement | 🔴 FAILURE |
| `correlationId` already used by a different entity | `…/correlation-conflict` | Correlation conflict | 🔴 FAILURE |
| Pattern-based business rule failure | `…/generic-validation-failure` | Generic Validation Failure | 🔴 FAILURE |
| Same `correlationId` already processed | — | — | 🟡 WARNING |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Removal Entry — Validations and Error Responses

Applies to all Removal Entry requests, whether submitted individually or as part of a Combo Entry Command (coming soon).

> `entryToRemoval` must reference the `correlationId` of a previously created entry. A Removal permanently deletes the referenced entry. If you need to preserve the original entry while offsetting its financial effect, use a Reversal Entry instead.

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account is inactive | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Entry referenced by `entryToRemoval` does not exist | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| Target entry is in PENDING status (PENDING entries cannot be compensated) | `…/pending-entry-compensation` | Compensation of entries with status PENDING is not allowed | 🔴 FAILURE |
| Target entry is already compensated or has an active compensation pending | `…/active-compensation` | Entry that is already compensated, or that has an active compensation, is not allowed | 🔴 FAILURE |
| Target entry status does not permit compensation | `…/invalid-compensation-target` | Invalid compensation target | 🔴 FAILURE |
| Compensation period for this entry kind has been exceeded | `…/entry-compensation-period-exceeded` | Transaction compensation period for kind `{kind}` exceeded | 🔴 FAILURE |
| Two entries in the same batch target the same removal | `…/duplicated-compensation-entry` | Duplicated compensating entry | 🔴 FAILURE |
| Settlement type mismatch between removal and target entry | `…/settlement-mismatch` | Settlement mismatch | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| No balance configuration for this entry | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| `correlationId` already used by a different entity | `…/correlation-conflict` | Correlation conflict | 🔴 FAILURE |
| Pattern-based business rule failure | `…/generic-validation-failure` | Generic Validation Failure | 🔴 FAILURE |
| Same `correlationId` already processed | — | — | 🟡 WARNING |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Reversal Entry — Validations and Error Responses

Applies to `ReversalEntryRequestData` when submitted as an individual command or as part of a Combo Entry Command (coming soon).

> Unlike a Removal Entry, a Reversal Entry preserves the original entry and posts a new offsetting entry. The `historyCode` specified for the reversal must be configured as the reversal history code for the original entry's `historyCode`.

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account is inactive | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Entry referenced by `entryToRevert` does not exist | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| Target entry is in PENDING status | `…/pending-entry-compensation` | Compensation of entries with status PENDING is not allowed | 🔴 FAILURE |
| Target entry is already compensated or has an active compensation | `…/active-compensation` | Entry that is already compensated, or that has an active compensation, is not allowed | 🔴 FAILURE |
| Target entry was already removed/deleted | `…/deleted-entry-reversal` | Reversal on previously deleted entry is not allowed | 🔴 FAILURE |
| Reversal's `historyCode` does not match the configured reversal code of the original entry | `…/illegal-reversal-history-code` | Illegal reversal history code | 🔴 FAILURE |
| Reversal amount does not match the original entry's amount | `…/illegal-reversal-amount` | Illegal reversal amount | 🔴 FAILURE |
| Target entry status does not permit compensation | `…/invalid-compensation-target` | Invalid compensation target | 🔴 FAILURE |
| Compensation period for this entry kind has been exceeded | `…/entry-compensation-period-exceeded` | Transaction compensation period for kind `{kind}` exceeded | 🔴 FAILURE |
| Two entries in the same batch target the same reversal | `…/duplicated-compensation-entry` | Duplicated compensating entry | 🔴 FAILURE |
| Settlement type mismatch | `…/settlement-mismatch` | Settlement mismatch | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| No balance configuration for this `historyCode` | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| `correlationId` already used by a different entity | `…/correlation-conflict` | Correlation conflict | 🔴 FAILURE |
| Pattern-based business rule failure | `…/generic-validation-failure` | Generic Validation Failure | 🔴 FAILURE |
| Same `correlationId` already processed | — | — | 🟡 WARNING |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Exclusion Entry — Validations and Error Responses

Applies to `ExclusionEntryRequestData`.

> An Exclusion Entry rolls back a previously created pending entry. The operation may be processed immediately or deferred. The command result indicates the outcome through the `status` field (`IMMEDIATE` or `DEFERRED`). Bulk exclusion of multiple entries will be supported through `ComboEntryExclusionRequestCommand` (coming soon).

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Account is inactive | `…/inactive-account` | Inactive account | 🔴 FAILURE |
| Entry identified by `correlationId` does not exist | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| No balance configuration for this entry | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Release Entry — Validations and Error Responses

Applies to `ReleaseEntryRequestData`.

> A Release Entry releases a portion of a previously blocked amount while leaving the remaining amount blocked. This is commonly used to support partial captures.

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Entry identified by `correlationId` does not exist | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| No balance configuration for this entry | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Combo Entry Command — Validations and Error Responses

All validations for the individual entry types in `entries[]` apply as described in the corresponding sections above. In addition, Combo Entry Commands enforce atomic processing and validate the request as a whole.

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Any entry in `entries[]` fails validation for any reason | *(per-entry type from tables above)* | *(per-entry title)* | 🔴 FAILURE — the **entire combo is rejected**; no entries are committed |
| Two entries in `entries[]` target the same compensation | `…/duplicated-compensation-entry` | Duplicated compensating entry | 🔴 FAILURE |
| Entries belong to different accounts | `…/account-mismatch` | Account mismatch | 🔴 FAILURE |
| Internal batch processing error (unclassified) | `…/invalid-batch-entries` | Problems occurred in process entry batch, contact the system support | 🔴 FAILURE |
| Two entries in `entries[]` share the same `correlationId` | — | — | Later duplicate is silently removed before execution; no failure result |

---

### ComboEntryExclusionRequestCommand — Validations and Error Responses

The entire command is atomic — if any correlation fails, no exclusions are applied.

| Condition | `failureInfo.type` | `failureInfo.title` | Outcome |
|---|---|---|---|
| Account does not exist | `…/account-not-found` | Account Not Found | 🔴 FAILURE |
| Any `correlationId` in `correlations[]` does not map to an existing entry | `…/entry-not-found` | Entry Not Found | 🔴 FAILURE |
| Any entry belongs to a different account than specified | `…/account-mismatch` | Account mismatch | 🔴 FAILURE |
| `balanceUpdateStrategy = CONDITIONAL` and balance insufficient | `…/insufficient-funds` | Insufficient funds | 🔴 FAILURE |
| No balance configuration for one of the entries | `…/balances-not-configured` | No balances were affected by the given entry's profile | 🔴 FAILURE |
| Unrecognized/malformed command payload | — | — | ⚫ INVALID OR UNRECOGNIZED MESSAGE |

---

### Complete Error Reference

All `failureInfo.type` values use the base `http://dtw.matera.com/transaction/` prefix.

| `failureInfo.type` (suffix after `…/`) | `failureInfo.title` | Applies to |
|---|---|---|
| `account-not-found` | Account Not Found | All commands |
| `account-temporarily-unavailable` | Account temporarily unavailable | All commands |
| `inactive-account` | Inactive account | All commands |
| `account-mismatch` | Account mismatch | ComboEntry, ComboEntryExclusion |
| `entry-not-found` | Entry Not Found | Removal, Reversal, Unblocking, Release, Exclusion, ComboEntryExclusion |
| `entry-type-not-found` | Entry type not found | Ledger, Blocking, Unblocking, Removal, Reversal |
| `entry-compensation-period-exceeded` | Transaction compensation period for kind `{kind}` exceeded | Removal, Reversal |
| `insufficient-funds` | Insufficient funds | All (CONDITIONAL strategy, debit-type entries) |
| `correlation-conflict` | Correlation conflict | All creation commands |
| `currency-mismatch` | Currency not allowed | Ledger, Blocking, Unblocking, Removal, Reversal |
| `invalid-settlement` | Invalid settlement | Ledger, Blocking, Unblocking |
| `settlement-mismatch` | Settlement mismatch | Removal, Reversal |
| `operation-not-allowed` | Operation not allowed | All (debit/credit blocked by account feature flag) |
| `balances-not-configured` | No balances were affected by the given entry's profile | All |
| `hold-reason-not-found` | Hold reason not found | Blocking, Unblocking |
| `hold-reason-not-allowed` | Hold reason not allowed | Blocking, Unblocking |
| `missing-hold-reason` | Missing hold reason | Ledger (`status=ON_HOLD`), Unblocking (`uponCreationHeldAs`) |
| `invalid-cautionary-block` | Invalid cautionary block | Ledger, Unblocking |
| `invalid-cautionary-block-target/debit-based-entry` | Invalid cautionary block target | Ledger, Unblocking |
| `invalid-cautionary-block-target/invalid-correlation-id` | Invalid cautionary block | Unblocking |
| `invalid-cautionary-block-target/invalid-entry-kind` | Invalid cautionary block target | Unblocking |
| `invalid-cautionary-block-target/invalid-status` | Invalid cautionary block target | Ledger, Unblocking |
| `invalid-cautionary-block-target/target-already-blocked` | Invalid cautionary block target | Ledger, Unblocking |
| `invalid-cautionary-block-target/target-already-compensated` | Invalid cautionary block target | Unblocking |
| `invalid-cautionary-block-target/target-already-unblocked` | Invalid cautionary block target | Unblocking |
| `invalid-cautionary-block-target/missing-cautionary-block-correlation-id` | Missing correlation ID of cautionary block entry | Unblocking |
| `cautionary-block-parameter-mismatch` | Cautionary block parameter mismatch | Unblocking |
| `pending-entry-compensation` | Compensation of entries with status PENDING is not allowed | Removal, Reversal |
| `active-compensation` | Entry that is already compensated, or that has an active compensation, is not allowed | Removal, Reversal |
| `deleted-entry-reversal` | Reversal on previously deleted entry is not allowed | Reversal |
| `illegal-reversal-history-code` | Illegal reversal history code | Reversal |
| `illegal-reversal-amount` | Illegal reversal amount | Reversal |
| `invalid-compensation-target` | Invalid compensation target | Removal, Reversal |
| `duplicated-compensation-entry` | Duplicated compensating entry | Removal, Reversal (batch), ComboEntry |
| `invalid-batch-entries` | Problems occurred in process entry batch, contact the system support | ComboEntry |
| `generic-validation-failure` *(or rule-defined URI)* | Generic Validation Failure *(or rule-defined)* | All (pattern-based rule engine) |
| `generic-error` | *(not published — logged internally only)* | Any unrecognized exception |

---

## Appendix

The following sections provide detailed reference information for the Digital Twin command schemas. Refer to these sections as needed while implementing your Digital Twin integration.

---

## Full Schema Definitions

### Common Types

#### `FailureInfo`, `BaseCommand`, `BaseCommandSuccessResult`, `BaseCommandFailureResult`, `ResultStatus`

`dtw-common-events/src/main/avro/common-commands.avsc`:

```avro
{
  "namespace": "com.matera.dtw.command.base.common",
  "type": "record",
  "name": "FailureInfo",
  "doc": "Represent a command execution failure metadata. Loosely based on RFC7807.",
  "fields": [
    { "name": "type", "type": "string" },
    { "name": "title", "type": "string" },
    { "name": "details", "type": ["null", "string"], "default": null },
    { "name": "parameters", "type": ["null", { "type": "string", "logicalType": "json" }], "default": null }
  ]
}

{
  "namespace": "com.matera.dtw.command.base.common",
  "type": "record",
  "name": "BaseCommand",
  "fields": [
    { "name": "baseEvent", "type": "com.matera.dtw.event.base.common.BaseEvent" },
    { "name": "replyTo", "type": "string" },
    { "name": "replyKey", "type": ["null", "string"], "default": null }
  ]
}

{
  "namespace": "com.matera.dtw.command.base.common",
  "type": "record",
  "name": "BaseCommandSuccessResult",
  "fields": [
    { "name": "baseEvent", "type": "com.matera.dtw.event.base.common.BaseEvent" }
  ]
}

{
  "namespace": "com.matera.dtw.command.base.common",
  "type": "record",
  "name": "BaseCommandFailureResult",
  "fields": [
    { "name": "baseEvent", "type": "com.matera.dtw.event.base.common.BaseEvent" },
    { "name": "failureInfo", "type": "com.matera.dtw.command.base.common.FailureInfo" }
  ]
}

{
  "namespace": "com.matera.dtw.command.base.common",
  "type": "enum",
  "name": "ResultStatus",
  "symbols": ["IMMEDIATE", "DEFERRED"]
}
```

#### `AccountKey`

Same as the Events guide's Appendix — `com.matera.dtw.event.account.common.AccountKey` (`branch: int`, `account: long`).

#### `FinancialAmount` / Settlement types

`dtw-common-events/src/main/avro/common-financial.avsc` + `dtw-entry-common-events/src/main/avro/entry-financial.avsc`:

```avro
{
  "namespace": "com.matera.dtw.event.common.financial",
  "type": "record",
  "name": "FinancialAmount",
  "fields": [
    { "name": "currency", "type": "com.matera.dtw.event.common.financial.Currency", "default": "USD" },
    { "name": "amount", "type": { "type": "bytes", "logicalType": "decimal", "precision": 19 } }
  ]
}

{
  "namespace": "com.matera.dtw.event.transaction.common",
  "type": "enum",
  "name": "Fulfillment",
  "default": "TOTAL",
  "symbols": ["TOTAL", "PARTIAL"]
}

{
  "namespace": "com.matera.dtw.command.transaction.command",
  "type": "record",
  "name": "TotalSettlementInfo",
  "fields": [
    { "name": "fulfillment", "type": "com.matera.dtw.event.transaction.common.Fulfillment", "default": "TOTAL" },
    { "name": "amount", "type": "com.matera.dtw.event.common.financial.FinancialAmount" }
  ]
}

{
  "namespace": "com.matera.dtw.command.transaction.command",
  "type": "record",
  "name": "PartialSettlementInfo",
  "fields": [
    { "name": "fulfillment", "type": "com.matera.dtw.event.transaction.common.Fulfillment", "default": "PARTIAL" },
    { "name": "amountRange", "type": "com.matera.dtw.event.common.financial.FinancialAmountRange" }
  ]
}
```

#### `CautionaryBlockInformation` / `CautionaryUnblockInformation`

`dtw-entry-events/src/main/avro/entry-commands.avsc` (top of file, shared across entry command payloads):

```avro
{
  "namespace": "com.matera.dtw.command.transaction.command",
  "type": "record",
  "name": "CautionaryUnblockInformation",
  "fields": [
    { "name": "correlationId", "type": "string" }
  ]
}
```

> `CautionaryBlockInformation` is defined analogously in `dtw-entry-common-events/src/main/avro/entry-common-commands.avsc` with `holdReasonId` (string) and `correlationId` (string) fields.

#### `DeferredUntil` (combo commands only)

`dtw-entry-events/src/main/avro/v2/common-deferred.avsc`:

```avro
{
  "namespace": "com.matera.dtw.event.command.intent.commons.v2",
  "type": "enum",
  "name": "TriggeredByKind",
  "symbols": ["MAIN", "COMPENSATION"]
}

{
  "namespace": "com.matera.dtw.event.command.intent.commons.v2",
  "type": "enum",
  "name": "ExecuteOnKind",
  "symbols": ["REMOVAL", "REVERSAL"]
}

{
  "namespace": "com.matera.dtw.event.command.intent.commons.v2",
  "type": "enum",
  "name": "PhaseKind",
  "symbols": ["REGULAR", "POSTED_ONLY"]
}

{
  "namespace": "com.matera.dtw.event.command.intent.commons.v2",
  "type": "record",
  "name": "DeferredUntil",
  "fields": [
    { "name": "triggeredBy", "type": "com.matera.dtw.event.command.intent.commons.v2.TriggeredByKind", "default": "MAIN" },
    { "name": "initiatorCorrelationID", "type": "string" },
    { "name": "executeOn", "type": ["null", "com.matera.dtw.event.command.intent.commons.v2.ExecuteOnKind"] },
    { "name": "onPhase", "type": "com.matera.dtw.event.command.intent.commons.v2.PhaseKind", "default": "REGULAR" }
  ]
}
```

### Schema Modules

| Maven Dependency | Contains |
|---|---|
| `dtw-common-events` | `BaseCommand`, `BaseCommandSuccessResult`, `BaseCommandFailureResult`, `FailureInfo`, `ResultStatus`, `AccountKey`, `FinancialAmount` |
| `dtw-entry-common-events` | `LedgerEntryRequestData`, `TotalSettlementInfo`/`PartialSettlementInfo`, `CautionaryBlockInformation` |
| `dtw-entry-events` | `EntryRequestCommand` and all per-kind payload/result records; `dtw-entry-events/.../v2/*` for `ComboEntryRequestCommand`, `ComboEntryExclusionRequestCommand`, and all combo payload/result records |

groupId for all of the above: `com.matera.dtw.event` (parent `dtw-events-parent`).

---

## Advanced Integration Topics

### Avro Serialization & Schema Registry

Same as the Events guide — commands are serialized as **Apache Avro** and registered in the **Confluent Schema Registry** using `TopicRecordNameStrategy` (subject = topic name + Avro record full name), which is what allows the same command topic to carry multiple distinct command/result types.

```properties
key.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
value.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
key.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
schema.registry.url=<provided-by-dtw-team>
```

### Maven Dependencies

```xml
<dependency>
    <groupId>com.matera.dtw.event</groupId>
    <artifactId>dtw-entry-events</artifactId>
</dependency>
<dependency>
    <groupId>com.matera.dtw.event</groupId>
    <artifactId>dtw-entry-common-events</artifactId>
</dependency>
<dependency>
    <groupId>com.matera.dtw.event</groupId>
    <artifactId>dtw-common-events</artifactId>
</dependency>
```

### Consuming Your Own Reply Topic

Because `replyTo` is producer-supplied rather than fixed by Digital Twin, your consumer needs to:

1. Subscribe to the topic you put in `replyTo` (and use `replyKey`, if set, for partitioning/lookup).
2. Distinguish success from failure by the Avro record type on each message — e.g. `EntryLedgerCommandSuccessResult` vs. `EntryLedgerCommandFailureResult` for a Ledger command (`TopicRecordNameStrategy` means both can coexist on the same reply topic).
3. Correlate a result back to the command that produced it via `baseEvent.parents[0].eventId` (which equals the original command's `baseEvent.content.eventId`) — or via `correlationId`/`replyKey`, if you set those to a value meaningful to your own system.
