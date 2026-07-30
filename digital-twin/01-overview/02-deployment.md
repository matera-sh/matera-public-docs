# Digital Twin Deployment

Digital Twin can be deployed in two ways, depending on an institution's architecture and business model.

## Alongside an existing Core banking platform

For traditional financial institutions, Digital Twin operates alongside the core banking system. It maintains a replicated copy of accounts (e.g. DDA, savings, investment) and performs real-time transaction authorization and balance updates in those accounts whether the Core is up or down.

The core banking system continues to manage:

- Regulatory reporting
- Risk and compliance
- Accounting and general ledger
- Fee and revenue management

## As a sidecar for digital assets

Digital Twin can also be deployed as a sidecar to extend an existing banking platform with digital asset capabilities. In this model, Digital Twin provides a real-time ledger and authorization layer for stablecoins, tokenized deposits, cryptocurrencies, and other digital assets, while the existing banking platform continues to manage core banking functions.

Digital Twin supports:

- Digital asset ledgering
- Real-time balance management
- Transaction authorization
- Treasury and liquidity management
- Event-driven integration with external systems

## Position in the technology stack

Digital Twin sits between the core banking system and customer-facing channels, payment systems, and digital asset applications, providing real-time transaction authorization and balance management.

- **Inbound requests:** Customer channels, payment systems, and digital asset applications submit transaction requests and balance inquiries to Digital Twin rather than directly to the core banking system. This removes the core from the real-time authorization path.

- **Outbound events:** After processing a transaction, Digital Twin publishes events that can be consumed by the core banking system and other downstream applications, such as fraud, AML, reporting, and analytics. The core banking system consumes these events to maintain the **system of record** and continues to manage account lifecycle functions such as account maintenance, status changes, regulatory reporting, accounting, and general ledger. Digital Twin serves as the **system of reference** for customer channels and payment operations.
