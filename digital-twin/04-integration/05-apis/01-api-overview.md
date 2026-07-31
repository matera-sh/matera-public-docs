# Digital Twin REST API endpoints

Digital Twin provides four REST API service groups for submitting financial operations and retrieving account information, balances, and statements. Equivalent financial operations are also available asynchronously through Kafka commands.

All REST endpoints require an OAuth2 bearer token. Depending on the deployment, the API gateway may also require mutual TLS (mTLS) to authenticate the connecting application.

For asynchronous transaction processing, see the [Commands Integration Guide](../03-dtw-commands-integration-guide.md).

| API | Purpose |
| ----- | ----- |
| **[Registry API](./03-dtw-registry-api-integration-guide.md)** | Create and maintain reference data such as accounts, account holders, account types, memberships, and limits. |
| **[Transaction API](./02-dtw-transaction-api-integration-guide.md)** | Process transactions and manage their lifecycle, including holds, releases, reversals, and removals. |
| **[Balance & Statement API](./04-dtw-balance-statement-api-integration-guide.md)** | Retrieve account balances and transaction history. |
| **[Composite API](./05-dtw-composite-api-integration-guide.md)** | Execute coordinated operations involving multiple accounts in a single atomic request. |
