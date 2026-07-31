# Digital Twin Integration - Overview

**Digital Twin provides several integration options. These integration interfaces enable core banking systems to keep Digital Twin synchronized, upstream applications to request financial operations, downstream systems to consume completed transactions, and client applications to query balances and account information.

## Table of Contents

- [1. Incoming Events](#1-incoming-events)
- [2. Commands](#2-commands)
- [3. Published Events](#3-published-events)
- [4. REST APIs](#4-rest-apis)

---

# Integration Interfaces


## 1. Incoming Events

Incoming Events communicate that something has already happened. They keep Digital Twin synchronized with account and reference data maintained by the Core Banking system. Whenever that data changes, a corresponding event is published to Digital Twin, which updates its internal Registry.

> See: [Digital Twin Integration Guide — Incoming Events](../_bmad-output/planning-artifacts/integration-guide/02-dtw-account-events-integration-guide.md)


---

## 2. Commands

Commands request that Digital Twin perform financial operations, including debit and credit postings, holds, releases, reversals, and removals. Commands follow an asynchronous request/reply pattern, returning either a success or failure result.

> See: [Digital Twin Integration Guide — Commands](../_bmad-output/planning-artifacts/integration-guide/03-dtw-transaction-entry-commands-integration-guide.md)

---

## 3. Published Events

Digital Twin publishes an event whenever a financial operation is successfully completed. Downstream applications use these events for things like statement generation, AML, fraud monitoring, notifications, reporting, and other post-transaction processing.

> See: [Digital Twin Integration Guide — Published Events](../_bmad-output/planning-artifacts/integration-guide/04-dtw-transaction-entry-events-integration-guide.md)

---

## 4. REST APIs

Digital Twin exposes REST APIs for synchronous integrations, providing access to account information, balances, financial operations, and configuration services. Upstream systems may use these APIs to submit financial operations — such as debits, credits, and holds — as an alternative to Commands. Client applications use these APIs for real-time balance inquiries, account lookups, and other synchronous requests.

All APIs use OAuth2 Client Credentials, return errors in RFC 7807 `application/problem+json` format, and represent monetary amounts as integers in minor units (cents).

> Full API contracts (OpenAPI/Swagger) are maintained in the respective external repositories

---

## Integration Interfaces Diagram

![Digital Twin Integration Interfaces](digital-twin-Integration-Interfaces.png)
