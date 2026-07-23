---
title: 'Payment System'
weight: 1
type: docs
---

Designing a payment system like Stripe, Adyen, or a bank's internal payments API. A buyer submits a payment request — amount, currency, and a payment method such as a card or bank account. The system authorises the charge with an external processor, debits the buyer, credits the merchant, and records every money movement in an immutable double-entry ledger, all while guaranteeing the buyer is **never charged twice** and the merchant is **never underpaid**.

The interesting challenges are correctness and reliability at scale: 10,000 transactions per second, each spanning multiple networked services that can fail independently, an external processor that may time out or return ambiguous results, and strict regulatory requirements that financial records be retained for years and be auditable to the penny.

## Functional Requirements

1. **Initiate payment:** A client creates a payment intent specifying the amount, currency, buyer, merchant, and payment method. Returns a unique `payment_intent_id`.
2. **Authorise with processor:** The system contacts an external payment processor (e.g., Stripe, Adyen, Visa/Mastercard network) to authorise and capture the charge against the buyer's payment method.
3. **Debit buyer / credit merchant:** Upon successful authorisation, debit the buyer's account and credit the merchant's settlement account in the internal ledger.
4. **Record in ledger:** Every money movement is appended to an immutable double-entry ledger. Every debit is paired with a matching credit; accounts always balance.
5. **Idempotent payments:** Each payment intent has a unique `payment_intent_id`. Retrying with the same ID returns the same result — no double charges, ever.
6. **Payment status:** Clients can query the current status of any payment (created, processing, succeeded, failed).
7. **Nightly reconciliation:** Automatically compare internal ledger entries against the processor's daily settlement report and surface discrepancies.

## Out of Scope

- Refunds, chargebacks, and disputes (separate state machine and workflow).
- User authentication and KYC/AML identity verification (upstream identity service).
- Currency conversion / FX rates (amounts pre-converted by callers).
- Subscription billing and recurring payment scheduling.
- Card tokenisation (assume a PCI-compliant vault is a separate service; raw card data never enters our system).
- Real-time merchant dashboards and BI reporting (separate read model / data warehouse layer).

## Non-Functional Requirements

- **Scale:** 10,000 TPS peak payments; ~500 M transactions per day average; ~40,000 ledger writes/s (4 ledger entries per payment).
- **Latency:** Payment initiation p99 < 300 ms; authorisation round-trip (including processor) p99 < 2 s; ledger write p99 < 100 ms.
- **Availability:** 99.99% (four nines) for the payment flow — ~52 minutes downtime budget per year. 99.9% for reconciliation (batch, less critical).
- **Consistency:** Exactly-once money movement. Strong consistency within the ledger service. Idempotency at every API and service boundary. External processors may deliver at-least-once callbacks; our idempotency layer converts these to exactly-once.
- **Durability:** Zero tolerance for data loss. Financial records retained for **7 years** minimum (regulatory requirement). Synchronous replication with WAL and tiered archival to cold storage.
- **Security & compliance:** All payment data encrypted at rest (AES-256) and in transit (TLS 1.3). PCI-DSS scope minimised by delegating card data to the vault service. Every ledger entry is tamper-evident and immutable.

## Terminology

| Term | Meaning |
|---|---|
| **Payment intent** | A record of the buyer's intent to pay, created before execution begins. Tracks the lifecycle from creation through authorisation to settlement. |
| **Payment processor** | External network (Stripe, Adyen, Visa/MC) that moves funds between the buyer's bank and the merchant's bank. |
| **Ledger** | Internal append-only record of every debit and credit. The single source of truth for account balances. |
| **Double-entry** | Every transaction debits one account and credits another by the same amount; the net always sums to zero. |
| **Idempotency key** | A client-supplied or system-generated ID ensuring retrying an operation is safe — the server returns the same result without re-executing. |
| **Outbox** | A DB table written atomically with the business data; a separate process publishes its rows to a message queue. Guarantees no event is lost even if the queue is temporarily unavailable. |
| **Saga** | A distributed transaction pattern where a multi-step workflow uses compensating transactions on failure rather than holding a global lock. |
