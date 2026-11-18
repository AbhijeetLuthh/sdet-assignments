# Subscription & Billing Service — SDET Submission (Abhijeet Luthra)

> **Status: initial design submission.** This PR contains documentation only. **Implementation has not started yet**: there is no service code, test code, `package.json` or database. Implementation continues on this branch straight after this PR is raised.

## 1. Understanding of the assignment

The task is to design and build an **automated validation suite** for a stateful subscription-billing service, not a CRUD smoke suite. A subscription is always in one of four states (`trialing`, `active`, `past_due`, `canceled`). It moves between them through a small state machine, not a straight line:

```text
trialing ──first charge succeeds──▶ active ◀──retry succeeds── past_due
trialing ──first charge fails─────────────────────────────────▶ past_due
active   ──recurring charge fails─────────────────────────────▶ past_due
past_due ──retries exhausted──▶ canceled
trialing / active ──customer cancels──▶ canceled   (terminal: never leaves)
```

`past_due` can return to `active` after a successful retry. Any move not shown is rejected or ignored. API calls and **asynchronous, possibly duplicated or out-of-order, signed webhooks** from an external payment provider drive these moves. The suite must prove the following at the API, state-machine, database and provider-integration levels:

- Only the allowed lifecycle transitions ever happen.
- Duplicate webhooks have exactly one effect (mandatory).
- Persisted state always matches the API and the business rules.
- The payment provider is mocked and its calls are verified precisely.

The recruiter's guidance adds a two-working-day deadline and asks for an approach-first PR. It puts the emphasis on retries, duplicate requests, partial failures, API/database consistency and reasoning about **transactional systems** rather than individual endpoints.

## 2. Proposed approach

- **Hybrid system under test** (assignment Option 3). A minimal TypeScript/Express service fixture with SQLite persistence and a **mocked** payment provider exists only to be tested.
- **Design first, then Red → Blue → Green.** Each business rule starts as a failing test.
- **Mandatory scope first.** Optional items, such as concurrent delivery, only start once everything mandatory passes. Duplicate subscription-creation requests are **must-have**, handled through an `Idempotency-Key` header.
- **Honest consistency model.** The provider charge and the local DB update are *not* treated as atomic. Charges use per-attempt idempotency keys, and webhooks reconcile uncertain outcomes (timeouts, local failures).

## 3. Test architecture (planned)

| Layer | Planned component |
|---|---|
| Environment fixture | Fresh in-memory SQLite + seed data, Express app, injected mock provider and fake clock per test |
| API client | Typed wrapper over Supertest |
| Mock provider / webhook simulator | Scripted success, decline and timeout; recorded calls; signed, unsigned and tampered webhooks |
| Verification layer | API, DB (via repositories) and provider assertions; an **invariant checker** that runs after every test |
| Builders | Customers, subscription requests, webhook payloads |
| Scenarios | `unit`, `api`, `lifecycle`, `webhooks`, `persistence`, `e2e` |

**Planned design patterns:** a transition table (State) for the lifecycle, Builder for test data, Adapter/dependency injection for the payment provider, and Repository for persistence. Each is described with the problem it solves in `docs/PRE_IMPLEMENTATION_DESIGN.md` §8. These are plans, not implemented code.

## 4. Technology choices (planned)

TypeScript · Node.js · Express · Jest · Supertest · SQLite via `better-sqlite3`. All of these are suggested or allowed by the assignment and were chosen for low setup friction and real relational constraints.

## 5. Key business scenarios

- Trial signup → trial ends → charge succeeds → `active`.
- Immediate signup → decline → `past_due` → retry succeeds → `active`; or retries exhausted → `canceled`.
- Cancel from `trialing` or `active`. Canceling an already-canceled subscription is rejected, and later webhooks never reactivate it.
- The same `POST /subscriptions` sent twice with one `Idempotency-Key` creates one subscription and one charge. Reusing the key with a different payload is rejected (422).
- The same webhook `event_id` delivered many times has one effect, and duplicates are recorded.
- A webhook is matched to a payment attempt by `provider_reference`, or by the invoice's single unresolved attempt. If the match is ambiguous, the event is quarantined and nothing financial changes.
- `payment.failed` arriving after `payment.succeeded` for the same invoice does not regress the subscription.
- A provider timeout is an *unknown* outcome. There is no automatic double charge, and a later webhook reconciles it.
- Invalid, missing or tampered signatures and malformed payloads change nothing.
- A local failure mid-webhook rolls back completely, and redelivery then applies once.

## 6. Documentation map

| Document | Contents |
|---|---|
| [`docs/PRE_IMPLEMENTATION_DESIGN.md`](docs/PRE_IMPLEMENTATION_DESIGN.md) | Understanding, scope, architecture, lifecycle, patterns, consistency boundaries, **assumptions register**, two-day plan |
| [`docs/API_CONTRACTS.md`](docs/API_CONTRACTS.md) | The four endpoints: requirement vs. proposed status codes, errors and webhook outcomes |
| [`docs/DATABASE_DESIGN.md`](docs/DATABASE_DESIGN.md) | Eight tables, constraints, transaction boundaries, invariants, seed/cleanup, failure injection |
| [`docs/TEST_STRATEGY.md`](docs/TEST_STRATEGY.md) | Real vs. mocked, **must-have vs. optional** scenarios with IDs, traceability to the assignment |
| [`CLAUDE.md`](CLAUDE.md) | Guidance for AI-assisted implementation sessions |

## 7. Implementation status

| Item | Status |
|---|---|
| Design documentation | ✅ Drafted for review (this PR) |
| Project setup (`package.json`, TS config) | ⏳ Not started |
| Service fixture (API, state machine, repositories, provider seam) | ⏳ Not started |
| Automated tests | ⏳ Not started; **no tests have been run** |
| Run instructions | ⏳ Will be added with the implementation |

## 8. AI usage

Claude Code was used to review these design documents against the assignment, find gaps and contradictions, and reorganise the files. I review and decide the business rules, assumptions and design. The final PR will include a full, candid AI-usage section.
