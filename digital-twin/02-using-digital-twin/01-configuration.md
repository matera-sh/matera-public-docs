# Digital Twin Configuration

## Configuring Digital Twin

Digital Twin is designed to make real-time authorization and balance management highly configurable. Rather than embedding transaction logic directly into application code, Digital Twin uses configurable products, transaction types, balances, account attributes, and system-level rules to determine how transactions should be authorized and how balances should be maintained.

The key configuration elements are:

- **Products** (e.g. check-free banking) — rules related to authorization for each product. Example: only 6 transactions allowed on Retail Money Market.
- **Transaction types** and associated codes (e.g. RTP credit = 42) — rules related to authorization for each transaction type. Example: transaction limit for RTP is $1M.
- **Balance types** (e.g. available, collected) — defines which balances are impacted by which transaction types. Example: RTP impacts available balance.
- **Accounts** (account number, status, product) and the associated customer ID — account-level rules for authorization. Example: only accounts with Active status can transact.
- **System-wide rules** for transactions. Example: no transaction of any type can exceed $X.

At the **product level**, institutions define the types of accounts they offer, such as check-free banking or money market accounts, along with the authorization rules associated with each. For example, a retail money market account may be limited to six transactions per month.

**Transaction types** and associated **transaction codes** are also configured within the platform. Each transaction type can have its own authorization rules and limits. For example, RTP credits may be limited to a maximum transaction amount of $1 million.

Digital Twin also supports configurable **balance types**, such as available and collected balance. Institutions define which transaction types impact which balances and when those balances should be updated. For example, an RTP credit may immediately increase the available balance.

At the **account level**, Digital Twin maintains core account reference information including account number, status, associated product, and customer relationship data. Account-specific authorization rules can then be applied. For example, only accounts in Active status may be permitted to transact.

Finally, **system-wide rules** provide centralized controls that apply across all transaction activity. These rules allow institutions to establish enterprise-wide limits and controls, such as maximum transaction thresholds regardless of payment rail or transaction type.

This approach allows Digital Twin to be flexible enough to adapt to different products, payment rails, and institutional policies.

---

## Configuring Account Types (Products)

Digital Twin supports configurable account types, also referred to as products, which define the types of banking services offered. Examples may include check-free banking, premier business checking, or specialized commercial account products. These account types can be replicated from the Core and maintained within Digital Twin through event-driven synchronization.

Each account type can have associated attributes that define operational behavior or descriptive characteristics. Attributes may represent authorization rules or product capabilities, such as whether overdraft protection is enabled. They may also store descriptive information used by downstream systems or channels, such as identifying a premium product tier.

In addition to attributes, Digital Twin supports configurable limits associated with account types. These limits allow institutions to define operational boundaries and authorization thresholds at the product level. For example, a courtesy overdraft limit may allow accounts within a specific product type to transact down to a defined negative balance threshold.

Data required to configure Account Types include:

| Field | Description | Example |
|---|---|---|
| Account Type | Unique identifier of the account type (product) | Check Free Banking |
| Category Type | Category of an account type | Personal Checking |
| Description | Description of an account type | Checking Account |

---

## Configuring Transaction Types

Digital Twin supports configurable transaction types that define how different payment and banking activities should be processed within the platform. Transaction types may be replicated from the Core and maintained within Digital Twin through event-driven synchronization. Each transaction type includes a transaction code, operational behavior such as credit or debit, and a business description such as ACH credit or RTP credit.

Limits can also be associated with transaction types to enforce authorization and operational controls. For example, a transaction type may include a maximum transaction amount or be associated with a specific payment rail such as ACH, RTP, or FedNow. These rules allow institutions to apply different authorization policies depending on the nature of the transaction.

In addition to limits, transaction types may contain configurable attributes used to classify or describe transaction behavior. For example, a transaction may be identified as an online transaction, branch transaction, or another operational category.

Data required to configure Transaction Types include:

| Field | Description | Example |
|---|---|---|
| Transaction Code | Unique identifier of the transaction type | 102 |
| Operation Type | Credit or Debit | Credit |
| Description | Description of transaction type | RTP Credit |

---

## Configuring Balance Types

Digital Twin supports configurable balance types, allowing financial institutions to define how balances should be maintained and used during transaction authorization. Accounts may have multiple, different balance types representing different views of funds based on settlement status, operational restrictions, or regulatory requirements.

For example, the available balance may include the ledger balance plus memo posts or pending transactions, representing funds that are available for real-time digital money movement. In contrast, the collected balance reflects only settled funds and may be used for activities such as branch withdrawals or other transaction types requiring fully cleared funds.

Additional balance types can support operational or compliance-related controls. A hold or blocked balance may represent funds restricted due to sanctions screening, fraud review, or other operational holds. A secured or collateralized balance may represent funds reserved or pledged elsewhere but still available to support specific authorization scenarios.

### Balance type examples

| Balance Type | Description |
|---|---|
| **Available Balance** | Ledger balance + memo posts/pending transactions. Intraday digital money movement may have access to this balance. |
| **Collected Balance** | Ledger balance — what has settled. Branch withdrawals may only have access to this balance. |
| **Hold/Blocked Balance** | Used in hold operations (e.g. OFAC hit). |
| **Secured/Collateralized Balance** | Balance of funds somewhere that can be tapped. |

---

## Mapping Transaction Codes to Balances

Transaction types drive which balances get updated (e.g. RTP transactions impact available balance). Posting logic can be defined to determine how different transaction types impact account balances. Each transaction code is mapped to one or more balance types, allowing institutions to define how funds should be reflected during transaction processing and authorization.

For example, an incoming wire credit may update only the collected balance, while an ATM deposit may immediately increase the available balance even before funds are fully settled. Debit card transactions typically reduce the available balance at the time of authorization, while real-time payments such as RTP may impact both available and collected balances simultaneously due to immediate settlement.

This mapping allows institutions to align authorization behavior with the operational characteristics of different payment rails and transaction types. Rather than applying the same balance logic to every transaction, Digital Twin allows balance updates to be configured based on settlement timing, payment certainty, and institutional policy.

### Mapping examples

| Code | Transaction | Operation | Balance Impacted |
|---|---|---|---|
| 22 | Incoming Wire | Credit | Collected |
| 42 | ATM Deposit | Credit | Available |
| 47 | Debit Card | Debit | Available |
| 27 | Check Deposit | Credit | Collected |
| 102 | RTP | Credit | Available & Collected |
| 107 | RTP | Debit | Available & Collected |

---

## Configuring Accounts

Digital Twin maintains account and customer reference information required to support real-time authorization and balance management. Account information may or may not be replicated from the Core. To minimize operational complexity and synchronization overhead, Digital Twin stores only the account-level information necessary to support authorization and transaction processing, such as account and routing number, account status, account type, and timezone.

Multiple accounts belonging to the same customer can be linked through a MembershipId, which acts as the customer-level relationship identifier within the platform. This allows Digital Twin to support customer-level authorization rules, relationship management, and consolidated balance or limit evaluations across multiple accounts.

Each account may also have multiple associated balances and limits, enabling the platform to support different authorization models and operational controls depending on the transaction type or payment rail.

In addition, configurable attributes may be associated with either the account itself or the MembershipId. These attributes can represent operational rules, such as whether an account is blocked for credits, or descriptive metadata, such as the channel through which the account was opened. This flexible configuration model allows institutions to apply customer-level and account-level policies without requiring custom application logic.

Data required to configure Accounts and Customer (Membership) ID include:

| Field | Applies To | Description |
|---|---|---|
| Account Number | Accounts | Copied from core, or hash representation |
| Routing Number | Accounts | May not be needed — must be 0 if none used |
| Account Status | Accounts | Active or Inactive |
| Account Type | Accounts | System-level account type configured. Example: Check-free banking |
| Time Zone | Accounts | — |
| ID | MembershipId | Unique identifier representing the customer |
| Kind | MembershipId | Type of identifier represented in the ID (e.g. SSN or hash representation) |
| Account & Routing Number | MembershipId | Copied from core |

---

## Attributes

Attributes can be established on Digital Twin that can represent either descriptive information or operational rules used during authorization and transaction processing. Attributes provide a flexible way to extend system behavior and account configuration without requiring custom application development.

Attributes may be applied at the system level to account types and transaction types. For example, an account type may contain a descriptive attribute identifying the product tier as Premium, while a transaction type may include an attribute identifying the associated payment rail as ACH.

Attributes may also be applied at the account or MembershipId level to support customer-specific operational controls. For example, an account may contain an operational attribute indicating whether credits or debits are permitted on the account.

This approach allows institutions to centrally manage product characteristics, transaction classifications, operational controls, and customer-specific rules while maintaining flexibility across different payment scenarios and account relationships.

### Attribute examples

| Attribute Type | Name | Type | Value |
|---|---|---|---|
| Account Type Attributes | PRODUCT_TIER | string | PREMIUM |
| Transaction Type Attributes | PAYMENT_RAIL | string | ACH |
| Account Attributes | BLOCKED_FOR_DEBIT | string | #1234 |

---

## Limits

Financial institutions can control transaction activity at both the account type and individual account level by configuring limits. Limits may be used to manage transaction amounts, transaction counts, overdraft exposure, or other operational thresholds associated with authorization decisions.

Each limit is associated with a balance that tracks how much of the limit has been consumed. Limits can define both lower and upper thresholds, allowing institutions to control acceptable transaction ranges based on dollar amounts or transaction volumes. Limits may also include configurable expiration periods, such as daily, weekly, or custom time windows.

Limits defined at the account type level can be inherited across groups of accounts while still allowing institution-specific overrides at the individual account level when needed.

This flexible model allows institutions to implement a wide range of authorization and risk controls. Examples may include restricting new customers to a maximum RTP transaction amount per day, limiting the number of withdrawals permitted on money market accounts, or allowing intraday balances to temporarily move into a controlled negative position.

### Limit use case examples

- New customers can only spend $500 per day via RTP
- Only 5 withdrawals can be made on money market accounts
- Intraday balance can only go negative by $1,000
