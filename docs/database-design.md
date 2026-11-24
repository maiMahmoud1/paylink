# PayLink Database Design (Phase 1)

## 1. Design Principles
- **Relational Integrity**: All core entities and financial records are stored in a relational database with explicit Primary Keys (`PK`), Foreign Keys (`FK`), `UNIQUE` constraints, and non-null checks.
- **Ledger-Driven Balances**: Wallet balances (`available` and `held`) are computed from append-only `ledger_entries` rather than overwritten directly on the wallet record.
- **Exact Monetary Precision**: All monetary columns use `DECIMAL(14,2)` — never floating-point numbers.
- **Idempotency & Concurrency Safety**: Unique constraints on `idempotency_key` and `payment_ref`, paired with hold ledger entries and transactional checks, prevent double charges and negative balances.

## 2. Entities & Tables (11 Tables)
1. `users`: Customer, agent, admin, and support accounts; tracks `verification_level` (`basic` / `verified`), `is_blocked`, and `blocked_until`.
2. `agents`: Links an agent profile (`terminal_code`) to a `user_id`.
3. `wallets`: Stores a customer's wallet (`currency` in EGP) linked uniquely (`UNIQUE`) to `user_id`.
4. `ledger_entries`: Append-only financial movements (`entry_type`, `bucket` for `AVAILABLE` or `HELD`, `amount` as `DECIMAL(14,2)`, `reference_key`, and `description`).
5. `billers`: Utility billers (`ElectroCo`, `AquaCo`, `TopUpNet`, `NetCo`) with protocol metadata and `is_active` status.
6. `biller_services`: Bill services under each biller, including `required_field_label`, `min_amount`, `max_amount`, and `is_active`.
7. `payments`: Core payment entity storing `payment_ref`, `idempotency_key`, monetary breakdown (`bill_amount`, `fee_amount`, `total_amount`, `commission_amount`), lifecycle `state`, and `failure_reason`.
8. `payment_attempts`: Records biller calls, timeouts, scheduled retries (`1, 5, 15` minutes), and asynchronous callbacks.
9. `notifications`: Outbound SMS, email, and push notifications with `status` and `retry_count` (up to 3 retries before `undelivered`).
10. `reconciliation_items`: Nightly settlement comparisons per biller and batch, storing both `biller_amount` and `paylink_amount`, `paylink_status`, `mismatch_type`, and review `status`.
11. `audit_logs`: Audit trail recording `actor_user_id`, `action_type`, `entity_type`, `entity_id`, `reason`, `metadata_json`, and `created_at`.

## 3. Relationships, Keys & Constraints
- `users (1) -- (0..1) agents` (`agents.user_id` FK & `UNIQUE`).
- `users (1) -- (0..1) wallets` (`wallets.user_id` FK & `UNIQUE`; wallet provisioned for customer accounts).
- `wallets (1) -- (0..*) ledger_entries` (`ledger_entries.wallet_id` FK).
- `payments (1) -- (0..*) ledger_entries` (`ledger_entries.payment_id` FK, nullable for top-ups).
- `billers (1) -- (0..*) biller_services` (`biller_services.biller_id` FK).
- `biller_services (1) -- (0..*) payments` (`payments.service_id` FK).
- `agents (0..1) -- (0..*) payments` (`payments.agent_id` FK, nullable when customer pays directly).
- `payments (1) -- (0..*) payment_attempts` (`payment_attempts.payment_id` FK).
- `payments (0..1) -- (0..*) notifications` (`notifications.payment_id` FK, nullable).
- `users (1) -- (0..*) notifications` (`notifications.user_id` FK).
- `billers (1) -- (0..*) reconciliation_items` (`reconciliation_items.biller_id` FK).
- `users (0..1) -- (0..*) audit_logs` (`audit_logs.actor_user_id` FK, nullable for automated system actions).
- **Key Constraints**:
  - `payments.idempotency_key` (`UNIQUE`) and `payments.payment_ref` (`UNIQUE`).
  - `users.email` (`UNIQUE`) and `users.phone` (`UNIQUE`).
  - All money amounts stored as `DECIMAL(14,2)`.
  - Payment `state` constrained to the approved lifecycle states (`Initiated`, `FundsHeld`, `Rejected`, `SentToBiller`, `Completed`, `Failed`, `Unknown`, `Refunded`).
