# Moneytrail - Product Requirements

## 1. What are we building?

Moneytrail is a multi-merchant payment platform that implements end-to-end payment lifecycle. This includes
authorization, capture, ledger positioning, settlement, reconciliation, payouts, and recovery. It uses synthetic data
and simulated external providers, while its internal data, workflow, and failure-handling logic are implemented for
real.

This project supports multiple simulated merchants. A merchant can:

- accept simulated customer payments;
- view pending, available, reserved, and paid-out balances;
- compare payment records with settlement reports; and
- receive simulated payouts through a mock bank.

Operators of this system can run repeatable failure scenarios, inspect the resulting records and traces, and follow safe
recovery paths.

Put in simple words:
> Moneytrail is an engineering system for demoing payment and distributed-systems correctness. It does not process real
> money or credentials, and makes no production, compliance, Stripe-equivalent claims.

## 2. How does the money flow?

Customer pays merchant → payment is authorized → merchant captures payment → provider settles the money → merchant
balance becomes available → platform pays out the merchant.

### 2.1. Definitions

1. **Authorization:** the provider approves the payment.
2. **Capture:** the merchant confirms that the approved payment should be collected.
3. **Settlement:** the Payment Service Provider (PSP) sends the collected money to the platform.
4. **Payout:** the platform sends available money to the merchant's bank account.
5. **Ledger:** this is system's accounting record or money moving in and out.
6. **Idempotency key:** this is the id that prevents a retry creating duplicate action (such as payments).
7. **Settlement report:** the PSP's list of payments that are settled.

> Remember, a payout is not the customer's payment. It is the payment paid to the merchants by the platform.

This lab focuses more on the merchant and payment-processor aspects. The merchant calls the payment API oon behalf of
the customer. We will not build a customer checkout or an issuing bank.

## 3. Who is it for?

| User                | What they need                                                       |
|---------------------|----------------------------------------------------------------------|
| Merchant developer  | Create a payment, retry safely, check its status, and receive events |
| Merchant operator   | See payments, balances, and payouts for their own shop               |
| Platform operator   | Find missing or conflicting records and recover safely               |
| Engineer or learner | Trigger a failure and understand how the system handles it           |

## 4. First release

### 4.1. What is included?

- Multiple simulated merchants and fake transactions. Few merchants will be seeded.
- Payment creation, authorization, capture, and status lookup.
- One simulated payment provider with success, refusal, delay, timeout, duplicate callback, and lost-response modes.
- A ledger that records capture, settlement, and payout.
- Settlement reports.
- Checks that find missing, duplicate, extra, or incorrect settlement lines.
- Merchant balances: `PENDING`, `AVAILABLE`, `RESERVED`, `PAID`.
- Payouts by a mock bank.
- Merchant webhooks with retries and delivery history.
- Dashboard for merchants and platform operators.
- Failure lab with repeatable scenarios.
- Automated tests for the main flow and failures.

The next release may include: a delayed payment method, partial refunds, load tests, and schema-migration exercises.

### 4.2. What are we not building?

- Real payments, cards, banks accounts, or personal data.
- A customer checkout.
- A real card network, issuing bank, or acquiring bank.
- Fraud, risk, KYC, AML, tax, disputes, subscriptions, or marketplace splits.
- Fees, foreign exchange, or international settlement
- Production-scale availability, security, or compliance.
- Kubernetes, a service mesh, or a custom database, or message broker.

## 5. User Journey

1. A customer chooses to pay a merchant. The checkout is simulated.
2. The merchant creates a payment with an order ID, amount, currency, and idempotency key.
3. The simulated provider authorizes the payment.
4. The merchant captures it.
5. The ledger records money owed to the merchant as `PENDING`.
6. A provider report confirms settlement. The money becomes available.
7. An operator requests a payout.
8. The system reserves the balance and asks the mock bank to pay the merchant.
9. The ledger marks the payout complete after the bank completes it.

> The UI must show payment, settlement, and payout as separate states. It must not use one vague `PAID` status.

## 6. How should failures behave?

| Failure                                        | Required behavior                                                           |
|------------------------------------------------|-----------------------------------------------------------------------------|
| The client repeats a request                   | Return the same payment; do not charge twice                                |
| The provider captures but its response is lost | Mark the result as `UNKNOWN`, check later, and do not capture again blindly |
| An event is delivered twice                    | Process it once                                                             |
| Events arrive out of order                     | Do not replace a final success with an older failure                        |
| The ledger is temporarily unavailable          | Keep the payment captured, show the delay, and catch up later               |
| A settlement line is missing or wrong          | Create a visible exception and keep that money unavailable                  |
| Two payouts use the same balance               | Allow only the amount that was safely reserved                              |
| The bank pays but its response is lost         | Check the existing payout; do not create a new transfer                     |
| A webhook fails                                | Retry it without changing the payment state                                 |

Every failure scenario above must be repeatable and covered by a test.

## 7. Invariants

- Store money as positive integer minor units such as paise for INR, cents for USD, etc. NEVER use floating point
  numbers.
- Store the currency on every money record. Don't mix currencies.
- The same merchant request and idempotency key must return the same result.
- Reusing an idempotency key with request data must fail.
- A timeout is not a proof that an operation failed.
- A captured payment must have one matching ledger entry or a visible exception.
- Every ledger transaction must balance.
- Ledger history is append-only. Don't edit old entries to hide a mistake.
- A settlement report line is imported once.
- A payout can't be larger than the merchant's reserved available balance.
- An uncertain payment or payout remains unresolved until it can be checked safely.
- One merchant can't view or change another merchant's data.
- Services cannot read each other's databases directly.

### 7.1. Simple accounting example

When ₹100 is captured, the platform records ₹100 to receive from the provider and ₹100 owed to the merchant. After
settlement, that ₹100 becomes available. After payout, the available balance and platform cash both fall by ₹100.

Authorization alone doesn't move money. Fees and reserves are not included in the first release.

## 8. Main system parts

1. **Payments:** creates payments, talks to the simulated PSP, stops duplicate requests, and owns payment state.
2. **Ledger:** stores balanced entries and merchant balances. It also reserves payout money.
3. **Settlement:** imports reports, finds differences, and manages payouts.
4. **Notifications:** sends merchant webhooks, and retries failures.
5. **Mock provider and mock bank:** behave like external systems and keep their own state.
6. **Dashboard:** shows merchants and operator views. It doesn't own payment data.

### 8.1. Tech stack

- Programming Languages: Java for backend, TypeScript for frontend.
- Frameworks: Spring Boot for backend, Next.JS/Vite for frontend.
- Tooling: Maven
- Database: Postgres
- Messaging: Kafka
- Containerization: Docker
- Tests: Testcontainers for integration tests
- Observability: OpenTelemetry

Each service will own its data. A state change and its outgoing event must be saved together. Message consumers must
handle duplicate delivery. The system doesn't assume that one transaction can cover the database, Kafka, and an external
provider.

## 9. APIs and Screens

### 9.1. APIs

- Create and view a payment.
- Capture a payment.
- View merchant balances and payouts.
- Import a settlement report.
- View reconciliation exceptions.
- Request a payout.
- View the full trace for one payment.
- Run a local failure scenario.

Publish and OpenAPI contract and example commands. Errors must include a stable error code and tracking ID.

### 9.2. Screens

- **Payments:** create, capture, retry, and inspect a payment.
- **Funds:** show pending, available, reserved, and paid-out money.
- **Operator workbench:** show exceptions and safe recovery actions.
- **Failure lab:** run a failure and inspect its events, ledger entries, and recovery.
- **System health:** show unresolved work, message delay, and webhook retries.

## 10. Success criteria for the first release

- A new contributor can start the first platform with one docker compose command and finish the walkthrough quickly.
- Repeated requests cannot create duplicate payments, captures, ledger entries, or payouts.
- Every capture, settlement, and payout is either accounted for or shown as a clear exception.
- Ledger entries balance and can rebuild every merchant balance.
- The main failure scenarios run in CLI with real Postgres and Kafka containers.
- One experienced engineer outside the project can run a scenario and understand the system's limits.

## 11. Build order

1. **Payment:** create, authorize, capture, and safely resolve a lost response.
2. **Ledger:** publish payment events and record each capture once.
3. **Settlement:** import reports and show differences.
4. **Payout:** reserve a balance and pay through the mock bank once.
5. **Product:** add the dashboard and webhooks.
6. **Later work:** delayed payments, refunds, load tests, and migration tests.

Each step must end with a working demo, one injected failure, and an automated test.