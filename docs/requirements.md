# PayLink – Requirements Pack (Phase 1)

## 1. Problem Summary

PayLink is a company in Egypt (similar to Fawry or Vodafone Cash). Customers pay utility bills (electricity, water, mobile, internet) from a digital wallet, using the mobile app or an agent's terminal in a small shop. Billers (the utility companies) want to receive payments quickly and correctly.

PayLink earns money from a small **service fee** on each payment and pays a **commission** to agents. It also holds customers' money in wallets, so **correctness with money is the most important goal**. A double charge, a negative balance, or a charge for a payment the biller never received would cause loss of trust, financial loss and regulator problems.

We are building the **backend** of PayLink. The main flow is: login and limits → bill inquiry → fee and total → fraud checks → hold the money → call the biller → confirmation (or failure) → notification → nightly reconciliation with the biller's settlement file.

The system must:

- Keep every money movement in a **ledger**; balances are always calculated from ledger entries.
- Support **different billers** with different protocols (ElectroCo REST/JSON, AquaCo SOAP-style XML, TopUpNet asynchronous REST with callback, and optionally NetCo daily batch CSV).
- Follow a **strict payment life cycle** (Initiated → FundsHeld → SentToBiller → Completed / Failed / Unknown → Refunded / Rejected).
- Handle failures safely: duplicate requests, biller timeouts, biller refusals, simultaneous payments, notification failures, duplicate callbacks and reconciliation differences.
- Be **easy to extend** with new billers (and later absorb the ElectroCo API v2 change by changing only the ElectroCo integration).
- Keep business numbers (limits, fees, commission rates, timeouts) in **configuration**, not in code.

**Out of scope:** a polished UI (a simple web page or CLI is enough), real payment gateways, real SMS providers, real personal data.

---

## 2. Stakeholders

| Stakeholder | Role | What they care about |
|---|---|---|
| **Customer** | Pays bills from the wallet | Fast, correct payments; never charged twice; clear messages |
| **Agent** | Shop owner with a PayLink terminal | Can pay for customers; receives commission |
| **Biller** (ElectroCo, AquaCo, TopUpNet, NetCo) | Issues bills, receives payments | Gets paid correctly; reconciliation matches |
| **Support employee** | Helps customers | Search payments, see history, block/unblock, approve refunds |
| **Admin** | Manages the platform | Manage billers, fees and limits; see reports; audit log |
| **PayLink management** | Business owner | Fee income, low losses, system ready for busy days |
| **Regulators / auditors** | External control | Complete audit trail of money movements |
| **SMS / email / push provider** | External service | Receives messages to send; may be down |
| **Development team** | Builds the system | Clear requirements, testable, maintainable design |

---

## 3. User Stories

**Priority legend:** **Must** = required, **Should** = important, **Could** = nice to have.
Each story maps to a functional requirement (FR) from the brief. Business values come from the brief and are configurable.

### Accounts and Wallet

**US-1 – Register (FR-1) – Priority: Must**
As a *customer*, I want to register an account so that I can use PayLink.
- **Given** I am not registered, **when** I register with valid details, **then** an account is created with the **Basic** verification level and an empty wallet (0.00 EGP available, 0.00 EGP held).
- **Given** the phone number or email is already registered, **when** I try to register, **then** the system rejects it with a clear error.

**US-2 – Upgrade to verified (FR-2) – Priority: Should**
As a *support employee*, I want to upgrade a customer to **Verified** so that the customer gets higher limits.
- **Given** a Basic customer, **when** support upgrades the customer to Verified, **then** the daily limit becomes 30,000 EGP (instead of 5,000 EGP).
- **Given** a customer who is not a support employee/admin, **when** he tries to upgrade an account, **then** the request is refused.

**US-3 – View balance and history (FR-3) – Priority: Must**
As a *customer*, I want to see my available balance, held balance and transaction history so that I know where my money is.
- **Given** I am logged in, **when** I open my wallet, **then** I see my available balance and my held balance.
- **Given** I have past transactions, **when** I open my history, **then** I see them newest first, with date, type, amount and status.

**US-4 – Top up the wallet (FR-4) – Priority: Must**
As a *customer*, I want to top up my wallet (simulated card or agent cash) so that I have money to pay bills.
- **Given** a valid top-up amount, **when** I top up by simulated card, **then** my available balance increases by that amount and a ledger entry is created.
- **Given** I pay cash to an agent, **when** the agent confirms the cash top-up, **then** my available balance increases by that amount.
- **Given** a simulated card that fails, **when** I top up, **then** the balance does not change and I see a clear error.

**US-5 – Ledger-based balances (FR-5) – Priority: Must**
As a *finance owner*, I want every balance change recorded as a ledger entry so that balances can always be proven.
- **Given** any money movement (top-up, hold, charge, release, fee, commission, refund), **when** it happens, **then** a ledger entry is created and the balance is **calculated from the sum of entries**, never overwritten.
- **Given** any wallet, **when** I add up all its ledger entries, **then** the total equals the balance shown (available + held).

### Biller Catalog

**US-6 – Manage billers (FR-6) – Priority: Must**
As an *admin*, I want to add, edit and disable billers and services so that the catalog stays up to date.
- **Given** I am an admin, **when** I add a biller with its services, required input fields and min/max amounts, **then** it appears in the catalog.
- **Given** an active biller, **when** I disable it, **then** customers can no longer pay it, but existing payments are not affected.

**US-7 – View billers and required fields (FR-7) – Priority: Must**
As a *customer*, I want to see the billers and what I must enter for each so that I can start a payment.
- **Given** I am logged in, **when** I open the biller list, **then** I see only **active** billers and their services.
- **Given** I select a service, **when** the details open, **then** I see its required fields (for example meter number or phone number).

### Payment Processing

**US-8 – Bill inquiry (FR-8) – Priority: Must**
As a *customer*, I want to see the amount, fee and total before paying so that I know what I will be charged.
- **Given** a valid meter number for ElectroCo, **when** I run an inquiry and the bill is 350.00 EGP, **then** I see amount 350.00, fee 2.50 and total 352.50 EGP.
- **Given** an unknown meter number, **when** I run an inquiry, **then** I see a clear "bill not found" message.

**US-9 – Pay a bill (FR-9) – Priority: Must**
As a *customer*, I want to pay a bill from my wallet so that I do not need to queue.
- **Given** a wallet with 400.00 EGP and a 352.50 EGP total, **when** I confirm payment and the biller confirms, **then** the payment is **Completed**, my wallet is charged 352.50 EGP and I get a receipt.
- **Given** my balance is lower than the total, **when** I try to pay, **then** the payment is **Rejected** with the reason "insufficient funds".

**US-10 – Idempotent payments (FR-10) – Priority: Must** *(hard case 1)*
As a *customer*, I want a repeated request to have the effect of one request so that a slow network never charges me twice.
- **Given** I sent a payment with idempotency key K, **when** the same request with key K arrives again, **then** no new payment is created, no extra charge is made, and the **first result** is returned.
- **Given** key K was used with a **different** request body, **when** it arrives, **then** the system rejects it with a conflict error.

**US-11 – Payment life cycle (FR-11) – Priority: Must**
As the *business*, I want every payment to follow only the allowed states so that no payment is ever in a wrong state.
- **Given** a payment in any state, **when** a transition from section 2.5 of the brief happens (for example SentToBiller → Completed), **then** it is allowed and recorded.
- **Given** a payment in a final state (Completed, Rejected, Refunded), **when** any change is attempted, **then** it is refused and the state stays the same.
- **Given** an invalid transition (for example Initiated → Completed), **when** it is attempted, **then** it is refused.

**US-12 – Hold and release funds (FR-12) – Priority: Must** *(hard cases 3 and 4)*
As a *customer*, I want my money held before the biller is called and released if the payment fails so that I never lose money on a failed payment.
- **Given** a payment passed the fraud checks, **when** it starts, **then** the total moves from available to held **before** the biller is called.
- **Given** the biller refuses (for example AquaCo returns an error code), **when** the payment becomes Failed, **then** the hold is released, the payment becomes Refunded, and I get an SMS with a clear reason.
- **Given** two payments from the same wallet arrive at the same moment and the balance covers only one, **when** both are processed, **then** one succeeds, the other is Rejected, and the balance **never goes negative**.

**US-13 – Timeout and retry (FR-13) – Priority: Must** *(hard case 2)*
As a *customer*, I want a payment that gets no answer to be handled safely so that I am neither double charged nor wrongly refunded.
- **Given** a live biller does not answer within 10 seconds (for example ElectroCo takes 12 seconds), **when** the timeout happens, **then** the payment becomes **Unknown**, the money stays held, and I am told it is being processed.
- **Given** a payment in Unknown, **when** 1, 5 and 15 minutes pass, **then** the system re-queries the biller (up to 3 times).
- **Given** the biller later confirms, **then** the payment becomes Completed; **given** the biller later refuses (or reconciliation finds no payment), **then** it becomes Failed and the hold is released.
- **Given** a payment in Unknown, **then** it is **never refunded automatically** until the system is sure the biller did not receive it.

**US-14 – Asynchronous callbacks (FR-14) – Priority: Must** *(hard case 6)*
As the *system*, I want to accept TopUpNet callbacks and ignore duplicates so that a payment completes only once.
- **Given** TopUpNet first replied "accepted", **when** it calls the callback with "success", **then** the payment becomes Completed.
- **Given** the callback with "success" arrives a second time, **when** it is processed, **then** the payment does not change and the duplicate is ignored and **logged**.
- **Given** the callback says "failed", **when** it is processed, **then** the payment becomes Failed and the hold is released.

**US-15 – Multiple billers (FR-15) – Priority: Must** *(hard case 8)*
As a *customer*, I want to pay electricity (ElectroCo), water (AquaCo) and mobile top-up (TopUpNet) bills so that I can pay all my bills in one place.
- **Given** each of the three billers, **when** I pay through PayLink, **then** the payment works even though each biller uses a different format.
- **Given** a new biller must be added, **when** the team adds it, **then** only a new integration and configuration are added; existing modules are unchanged.
- *(NetCo daily batch is optional and not required for the minimum.)*

**US-16 – Fee calculation (FR-16) – Priority: Must**
As the *business*, I want the correct service fee calculated on every payment so that PayLink earns its fee.
- **Given** electricity or water, **when** the fee is calculated, **then** it is a fixed 2.50 EGP.
- **Given** a mobile top-up, **when** the fee is calculated, **then** it is 1.5% of the amount with a minimum of 1 EGP (for example 100 EGP → 1.50; 50 EGP → 1.00).
- **Given** internet, **when** the amount is 200 EGP or less, **then** the fee is 2 EGP; **when** above 200 EGP, **then** the fee is 4 EGP.
- **Given** a promotion period for one chosen biller, **when** a payment is made within the start and end dates, **then** the fee is 0 for that biller.
- **Given** any calculation, **then** exact decimal types are used (no floating point).

**US-17 – Agent payment and commission (FR-17) – Priority: Should**
As an *agent*, I want to pay a bill for a customer at my terminal and earn commission so that I earn money from my shop.
- **Given** an agent, **when** a payment made at the agent's terminal is Completed, **then** the agent's wallet is credited 40% of the service fee (for example 1.00 EGP of a 2.50 EGP fee).
- **Given** the payment fails, **when** it becomes Failed, **then** no commission is paid.

**US-18 – Payment limits (FR-18) – Priority: Must**
As the *business*, I want payments above the limits rejected with a clear reason so that risk is controlled.
- **Given** an amount below 5 EGP or above 10,000 EGP, **when** I pay, **then** it is Rejected with the reason "amount outside allowed range".
- **Given** a Basic account that already paid 4,800 EGP today, **when** it tries to pay 300 EGP, **then** it is Rejected with the reason "daily limit exceeded" (limit 5,000 EGP; Verified: 30,000 EGP).

**US-19 – Velocity rule (FR-19) – Priority: Should**
As the *business*, I want rapid repeated payments blocked so that fraud is reduced.
- **Given** an account makes more than 5 payments within 60 seconds, **when** the 6th arrives, **then** it is Rejected and the account is blocked for 10 minutes.
- **Given** the 10 minutes have passed, **when** the account tries again, **then** it can pay normally.

### Support and Administration

**US-20 – Support payment search (FR-20) – Priority: Must**
As a *support employee*, I want to search a payment by reference and see its full history so that I can help customers.
- **Given** a valid payment reference, **when** I search, **then** I see the payment details and **every state change** with time and reason.
- **Given** an unknown reference, **when** I search, **then** I see "not found".

**US-21 – Block and unblock accounts (FR-21) – Priority: Should**
As a *support employee*, I want to block and unblock an account so that I can stop suspicious activity.
- **Given** an active account, **when** support blocks it, **then** every new payment from it is Rejected with the reason "account blocked".
- **Given** a blocked account, **when** support unblocks it, **then** the customer can pay again.

**US-22 – Payment notifications (FR-22) – Priority: Must** *(hard case 5)*
As a *customer*, I want an SMS and a push message when my payment completes or fails so that I know the result.
- **Given** a payment becomes Completed or Failed, **when** the result is final, **then** I receive an SMS and a push notification with a clear message.
- **Given** the SMS provider is down, **when** a payment completes, **then** the payment **still completes**.

**US-23 – Notification retry (FR-23) – Priority: Should** *(hard case 5)*
As the *business*, I want failed notifications retried so that customers are informed whenever possible.
- **Given** an SMS or email fails, **when** it is retried, **then** the system tries **3 times**.
- **Given** all 3 retries fail, **when** the last retry ends, **then** the notification is marked **undelivered**.

**US-24 – Edit fees and limits (FR-24) – Priority: Should**
As an *admin*, I want to edit fee and limit values without redeploying so that the business can react quickly.
- **Given** I am an admin, **when** I change a fee or limit value, **then** new payments use the new value without restarting the application.
- **Given** a payment already in progress, **then** it is not affected by the change.

**US-25 – Audit log (FR-25) – Priority: Must**
As an *auditor*, I want every money movement and admin action written to an audit log so that everything can be traced.
- **Given** any money movement or admin action, **when** it happens, **then** an audit record is saved with **who, when, what and why**.
- **Given** an audit record, **then** it cannot be edited or deleted through the API.

**US-26 – Daily reconciliation (FR-26) – Priority: Should** *(hard case 7)*
As a *support employee*, I want a daily reconciliation report so that differences with billers are found.
- **Given** a biller's settlement file, **when** reconciliation runs, **then** PayLink payments are compared with the file and a difference report is produced.
- **Given** the file lists a payment that PayLink has as Failed, **when** the report is made, **then** the difference is **flagged** for support to review.
- **Given** an Unknown payment that is in the file, **then** it becomes Completed; **given** one that is not in the file, **then** it becomes Failed.

**US-27 – Refund of a completed payment (FR-27) – Priority: Could**
As a *customer*, I want to request a refund of a completed payment so that I can get my money back in case of a problem.
- **Given** a Completed payment, **when** I request a refund, **then** a refund request is created with status "pending approval".
- **Given** a pending refund, **when** support approves it, **then** the money is returned to my wallet through ledger entries and the action is audited.
- **Given** support rejects the request, **then** no money moves.

**US-28 – Admin daily totals (FR-28) – Priority: Could**
As an *admin*, I want to see daily totals so that I can follow the business.
- **Given** a selected day, **when** I open the totals, **then** I see number and amount of payments, fees earned, commissions paid, and failures per biller.

### Priority summary

| Priority | Count | Stories |
|---|---|---|
| **Must** | 19 | US-1, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 20, 22, 25 |
| **Should** | 7 | US-2, 17, 19, 21, 23, 24, 26 |
| **Could** | 2 | US-27, 28 |
| **Total** | 28 | |

### Hard cases → user stories

| # | Hard case (brief section 2.7) | Covered by |
|---|---|---|
| 1 | Same payment sent twice | US-10 |
| 2 | ElectroCo takes 12 seconds | US-13 |
| 3 | AquaCo refuses with an error code | US-12 |
| 4 | Two payments at the same moment | US-12 |
| 5 | SMS provider down | US-22, US-23 |
| 6 | TopUpNet calls callback twice | US-14 |
| 7 | Settlement file lists a Failed payment | US-26 |
| 8 | A new biller must be added | US-15 |

---

## 4. Non-Functional Requirements (NFRs)

| ID | Quality | Measurable requirement | How we measure it |
|---|---|---|---|
| **NFR-1** | Correctness | **Zero** double charges and **zero** negative balances in the duplicate and concurrency tests | Automated tests (hard cases 1 and 4) run with many parallel requests; check ledger totals after the run |
| **NFR-2** | Performance | At least **100 payments per second** sustained for **1 minute** against the mock billers | Load test (for example k6 or Locust) with the result report |
| **NFR-3** | Resilience | When one biller is down, payments to **other billers still succeed** (success rate for other billers ≥ 99% during the outage) | Test that stops one mock biller (or makes it fail on purpose) while paying the others |
| **NFR-4** | Modifiability | Adding a new biller changes **only** the Biller Integration module and configuration (**0 changes** in other modules) | Compare the changed files in the commit/pull request for a new mock biller |
| **NFR-5** | Auditability | **100%** of money movements have an audit record with who, when and why | For every money-movement ledger entry, verify that a corresponding audit record exists with who, when, what and why. Admin-only audit records are excluded from the count comparison |
| **NFR-6** | Testability | At least **70% line coverage** on domain logic, measured in CI on every push | Coverage report in the GitHub Actions pipeline |

---

## 5. Ambiguities and Decisions

| # | Ambiguity / question found | Decision | To confirm? |
|---|---|---|---|
| 1 | Does the **daily limit** use a calendar day or a rolling 24 hours? | **Calendar day** in Egypt time (Africa/Cairo), resetting at 00:00. Simple to explain and test. | No |
| 2 | What counts toward the daily limit: the bill amount only, or amount + fee? Which statuses count? | **Bill amount only**. Counts payments that are Completed or in progress (FundsHeld, SentToBiller, Unknown). Failed, Rejected and Refunded payments do **not** count. | No |
| 3 | Do the min 5 / max 10,000 EGP limits apply to the bill amount or the total with fee? | To the **bill amount** (without the fee). | No |
| 4 | A Basic account has a 5,000 EGP daily limit but a single payment can be up to 10,000 EGP. | The **stricter** rule applies: a Basic account cannot pay more than 5,000 EGP in one day, so a 6,000 EGP payment is rejected with "daily limit exceeded". | No |
| 5 | How is the **1.5% mobile top-up fee** rounded? | Round **half up to 2 decimals**, then apply the minimum of 1 EGP. Exact decimal type only. | No |
| 6 | Internet fee "2 EGP up to 200 EGP, 4 EGP above" – what about exactly 200? | **200 EGP → 2 EGP fee**; **200.01 EGP and above → 4 EGP**. | No |
| 7 | Can several billers have a promotion at once? What is the commission when the fee is 0? | **One biller** at a time (as in the brief). Commission is 40% of the fee, so it is **0** during a promotion. | No |
| 8 | When an **agent pays on behalf of a customer**, whose wallet is charged? | The **agent's own wallet** is charged (the agent takes cash from the customer); the commission is credited to the same agent wallet on completion. The customer reference is stored on the payment. | **Yes – confirm with instructor/TA** |
| 9 | **Refund of a completed payment (FR-27)** conflicts with "Completed is final" in the life cycle. | The payment **stays Completed**. A refund is a **separate refund record** with its own ledger entries. It returns the total paid (amount + fee) to the wallet and reverses the agent commission if any. Only a **Could** story. | **Yes – confirm** |
| 10 | How long to wait if an **Unknown** payment gets no answer after 3 retries? | It waits for **reconciliation** (settlement file). If the file has the payment → Completed; if not → Failed and the hold is released. | No |
| 11 | Can the customer pay a **different amount** than the bill? | For **ElectroCo and AquaCo**: pay the **full bill amount** from the inquiry (AquaCo rejects more than the bill). For **TopUpNet**: the customer chooses the amount within 5–10,000 EGP. | No |
| 12 | Is the **velocity rule** counted on attempts or on successful payments? "More than 5" means which one? | Counts **payment attempts** from one account in 60 seconds. The **6th attempt** is rejected and blocks the account for 10 minutes. | No |
| 13 | Difference between a **velocity block** and a **manual block** by support. | Both set "blocked". The velocity block **expires after 10 minutes**; the manual block stays until support unblocks. | No |
| 14 | How does an account become **Verified**? (not described) | A **support employee or admin** upgrades it manually (simulated verification, no real documents). | No |
| 15 | How do users **log in**? Is there a password, OTP or token? | Simple **token-based login** (for example JWT) with roles: Customer, Agent, Support, Admin. No real OTP or real personal data. | No |
| 16 | What does a **simulated card top-up** do? | No real gateway. Any test card succeeds, one fixed test value fails (to test failures). **No real card numbers** are stored. | No |
| 17 | **Idempotency key** scope and reuse with a different body. | The key is unique **per user** and stored with the payment. Same key + same request → first result. Same key + different request → **409 Conflict**. | No |
| 18 | Which notifications are mandatory, and what about the **Unknown** state? | SMS + push on **Completed** and **Failed** (FR-22). Email is optional. For Unknown, the customer sees an "in process" message in the API/app. | No |
| 19 | Is **NetCo** (daily batch CSV) required? | **Optional** per the brief. Not in our minimum scope; the Biller Integration design will allow adding it later. | No |
| 20 | Currency and decimals. | **EGP only**, 2 decimal places, exact decimal type. All amounts in the API are in pounds (ElectroCo v2 later uses piasters, handled only inside its adapter). | No |
| 21 | Programming language and framework. | **Python with FastAPI** was selected by the team for the PayLink backend. The framework is fixed for Phase 1 and Phase 2. | No |
| 22 | Who edits **fees and limits** (FR-24)? | Only the **Admin**. Changes apply to **new** payments only. Values are stored in configuration/database, not in code. | No |
