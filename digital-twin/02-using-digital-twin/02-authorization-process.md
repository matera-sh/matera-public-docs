# Digital Twin Authorization Process

Digital Twin's primary function is to authorize transactions and maintain high-volume account balances in a real-time ledger. It is adept at authorizing transactions on base level components such as balances in an account or account status, as well as evaluating a combination of configuration at the account and system level in real time to provide approval or rejection.

---

## Foundational Concepts for Authorization

Before describing the authorization flow, it is important to understand how Digital Twin structures accounts, balances, and transaction behavior. These elements define the inputs used during authorization.

### Account-Level Data

- **Accounts** — At the core of the authorization process is the account itself. Each account stored in Digital Twin includes identifying information such as account number, routing number, and branch ID, along with an associated account type and a current status. The account status (e.g., active or inactive) is one of the first conditions evaluated during authorization, as inactive accounts are not eligible to transact.

- **Account Types** — Accounts are linked to an account type such as a DDA, savings, HSA, loan, or other. Account types define the baseline behavior of accounts by applying consistent rules, limits, and capabilities across all accounts of that type. These rules, limits, and features configured at the account type level enable the bank to define the product (e.g. check-free banking). This allows banks to configure product-level behavior once and apply it uniformly.

- **Balance Types** — Each account maintains one or more balances such as "available" and "collected." These balances are continuously updated in real time and represent different views of funds based on bank-defined logic.

  The transaction type determines which balance is relevant and how it should be evaluated. For example, an ATM withdrawal transaction type may be configured to evaluate the available balance when authorizing, while other transaction types may consider collected balances or different balance types defined by the institution.

  This separation allows banks to define how funds are treated across different transaction types while maintaining consistent and accurate balance tracking.

### Account Extensions: Features and Limits

Beyond account data and account types, Digital Twin supports additional controls that further refine authorization decisions at the account level.

- **Features** — Account-level features can be used to apply restrictions to an account. For example, features can be used to block an account for debits or credits. These controls are typically applied for operational or risk management purposes.

- **Limits** — More granular control can be applied to transaction activity by applying limits at the account level. These may include dollar-based limits (e.g., maximum withdrawal amount) or count-based limits (e.g., number of transactions per day). Limits can be evaluated against aggregated activity — either dollar value or number of transactions.

Together, features and limits allow banks to tailor authorization behavior at the account level without altering the underlying product configuration. These controls can also be configured to reflect risk considerations, enabling restrictions and limits to be applied based on predefined risk criteria.

### Transaction-Level Configuration

- **Transaction Codes** — Transaction behavior in Digital Twin is defined through transaction codes. Each transaction code represents a specific type of activity (such as an ATM withdrawal or RTP credit) and determines how the system should process that transaction. A transaction code defines whether the transaction is a debit or a credit and specifies which balances are impacted and how they are affected. For example, an ATM withdrawal transaction code may be configured as a debit that reduces both available and collected balances.

  This configuration ensures that authorization decisions are consistent and predictable for each type of transaction while still allowing flexibility in how different transaction types interact with account balances.

### Posting Status

Transactions can affect a balance in one of two ways — pending or posted — depending on the use case.

- **Pending** transactions represent a provisional state in which funds are reserved but not yet fully settled. This is commonly used in two-step processes such as card authorizations where funds are held before final settlement occurs.

- **Posted** transactions represent final settlement where funds are definitively moved and balances are updated.

When a pending transaction transitions to posted, the system performs the update as a single atomic operation, ensuring consistency between the reserved and settled states. This capability allows Digital Twin to support both immediate and multi-step transaction flows without compromising balance accuracy.

---

## How Authorization Works in Practice

When a transaction request is received, Digital Twin evaluates it in real time using the combined inputs described above. The authorization process follows a logical sequence of five gates.

### Gate 1 — Account Exists
Verifies the account exists in its registry. If the account is unknown, the transaction is rejected.

### Gate 2 — Account Active
Verifies that the account status is **ACTIVE**. Transactions against inactive accounts are rejected.

### Gate 3 — Account Not Blocked
Verifies the account is not blocked for the requested transaction type (e.g. `BLOCKED_FOR_DEBIT` or `BLOCKED_FOR_CREDIT`). If the account is blocked, the transaction is rejected.

### Gate 4 — Validation Rules
Evaluates all matching VALIDATION rules. Each rule uses the Validation Framework (fixtures/features) to determine whether the transaction satisfies the bank's configured business requirements. If any validation fails, the transaction is rejected.

### Gate 5 — Balance Check
Evaluates all matching BALANCE rules to determine which balance types are affected and how they should be updated. The resulting balances are validated against their configured minimum and maximum limits. If any balance exceeds its permitted bounds, the transaction is rejected.

If all conditions are met, the transaction is authorized. If any condition fails, the transaction is rejected.

---

## Real-Time Consistency and Control

A key characteristic of Digital Twin's authorization model is that all evaluations occur against a real-time, continuously updated ledger. This ensures that every authorization decision is based on the most accurate and current account state.

Because the system processes transactions in real time and applies controls consistently across all channels, it prevents conflicting transactions and ensures that funds cannot be used more than once. This is particularly important in environments with high transaction concurrency, such as real-time payments or card-based activity.

---

## Extensibility and Future Capabilities

Digital Twin's architecture is designed to support additional capabilities that enhance authorization logic. For example, the Membership ID field identifies accounts that belong to the same customer. Digital Twin is being enhanced to leverage this field for expanded transaction processing scenarios, such as overdraft protection that evaluates balances across multiple accounts.

Additional modules and microservices can also be introduced to extend processing capabilities, such as sweep functionality — end of day or even intraday. These enhancements build on the same foundational authorization model, allowing institutions to evolve their capabilities without redesigning core transaction logic.

---

## Authorization Process Summary

Authorization in Digital Twin is not a single check, but a coordinated evaluation of system-level configuration, account-level data, and transaction-specific rules. By combining these elements in real time, Digital Twin enables financial institutions to make accurate, consistent authorization decisions at scale.

This approach allows banks to move beyond static, batch-oriented processing and operate with the speed, control, and reliability required for modern payment environments.
