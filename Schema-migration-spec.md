# Dondok MVP Schema Migration Spec

Status: canonical implementation-ready migration specification for MVP backend schema.

Authoritative references:

- `docs/ERD-dondok.md`
- `docs/Settlement-design.md`
- `docs/Implementation-guardrails.md`

Scope:

- This document is DB/Flyway/JPA oriented.
- It covers only core MVP backend entities: `crew_participant`, `point_account`, `point_history`, `settlement`, `settlement_item`.
- It does not redesign product semantics, lifecycle semantics, settlement semantics, or deferred Phase 2 domains.

## 1. Schema conventions

Canonical rules for this migration round:

- PK strategy: every table primary key is `BIGINT` auto increment.
- Money type: every persisted monetary amount uses `BIGINT` only.
- Enum persistence: application enums are stored as `STRING` (`VARCHAR`) values, not ordinal integers.
- Audit columns:
  - `created_at DATETIME(6) NOT NULL`
  - `updated_at DATETIME(6) NOT NULL` for mutable aggregate/cache tables
- Optimistic locking:
  - `version BIGINT NOT NULL` for `point_account`, `crew_participant`, and `settlement`.
- FK delete policy:
  - use `RESTRICT` / `NO ACTION` for money, audit, participant, settlement, and ledger relationships.
- `point_history` is append-only. Do not update or soft-delete ledger rows after insertion.
- Balance reconciliation uses `point_history`, `crew_participant` lifecycle/deposit state, `settlement_item` linkage, and `point_account` cached balances together because approval is a bucket transition without a ledger row.
- Money/audit entities do not use soft delete.
- Terminal `crew_participant` rows are preserved; no physical delete or row reuse for MVP re-application.

## 2. Table specifications

### 2.1 `crew_participant`

#### Table purpose

Participant-level lifecycle row for one member in one crew. It owns the participant deposit snapshot and is the unit for activation eligibility and settlement baseline inclusion.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `crew_id` | `BIGINT` | N | none | FK to `crew.id`. |
| `member_id` | `BIGINT` | N | none | FK to `member.id`. |
| `status` | `VARCHAR(30)` | N | none | STRING enum: `PENDING`, `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`. |
| `deposit_amount` | `BIGINT` | N | none | Participant deposit snapshot copied from `crew.deposit_amount` at `PENDING` creation. |
| `pending_at` | `DATETIME(6)` | N | current timestamp at application creation | Application submission / reserve creation timestamp. |
| `locked_at` | `DATETIME(6)` | Y | `NULL` | Approval timestamp for `PENDING -> LOCKED`. |
| `rejected_at` | `DATETIME(6)` | Y | `NULL` | Terminal timestamp for host rejection. |
| `cancelled_at` | `DATETIME(6)` | Y | `NULL` | Terminal timestamp for applicant cancellation before approval. |
| `expired_at` | `DATETIME(6)` | Y | `NULL` | Terminal timestamp for automatic expiration before start. |
| `released_point_history_id` | `BIGINT` | Y | `NULL` | FK to reserve-release `point_history.id`; authoritative reserve-release ledger evidence. |
| `version` | `BIGINT` | N | `0` | Optimistic locking version. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time. |

#### Constraints

- PK: `primary key (id)`.
- FK: `crew_id -> crew.id` with `RESTRICT` / `NO ACTION`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- FK: `released_point_history_id -> point_history.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE: `unique(crew_id, member_id)`.
- UNIQUE: `unique(released_point_history_id)` with normal nullable-unique semantics; multiple `NULL` values are allowed, but a non-null release ledger row cannot be shared by multiple participants.
- CHECK:
  - `deposit_amount >= 0`.
  - `status in ('PENDING', 'LOCKED', 'REJECTED', 'CANCELLED', 'EXPIRED')` if DB-level enum checks are used.
  - timestamp consistency checks may be enforced in application/service layer when DB dialect support is limited.

#### Indexes

- `index(crew_id, status)` for capacity reservation, activation eligibility, and settlement baseline queries.
- `index(member_id, status)` for member participation lookup.
- `unique(released_point_history_id)` also serves release evidence lookup and prevents multiple participants from sharing one reserve-release ledger row.

#### Notes

- `PENDING` means application submitted plus reserve state.
- `PENDING` counts toward capacity reservation only.
- `LOCKED` is the only participant status included in activation eligibility, frozen participant baseline, and settlement baseline.
- `REJECTED`, `CANCELLED`, and `EXPIRED` are terminal rows and remain preserved.
- MVP disallows re-application to the same crew; `unique(crew_id, member_id)` is the DB enforcement point.
- Reserve release is allowed once per `crew_participant.id`; terminal transition and reserve release happen in the same transaction.
- `released_point_history_id` is the authoritative evidence that reserve release occurred. Do not rely on timestamp-only release evidence.
- No `WITHDRAWN` / rejoin semantics are introduced in this MVP migration.

### 2.2 `point_account`

#### Table purpose

Current balance bucket cache/source for a member. It is updated transactionally with `point_history` but remains reconcilable from the append-only ledger.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `member_id` | `BIGINT` | N | none | FK to `member.id`. |
| `available_balance` | `BIGINT` | N | `0` | Immediately usable point balance. |
| `reserved_balance` | `BIGINT` | N | `0` | Sum of `PENDING` application reserve amounts. |
| `locked_balance` | `BIGINT` | N | `0` | Sum of `LOCKED` crew deposit amounts still locked before final release/refund. |
| `version` | `BIGINT` | N | `0` | Optimistic locking version. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time. |

#### Constraints

- PK: `primary key (id)`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE: `unique(member_id)`.
- CHECK:
  - `available_balance >= 0`.
  - `reserved_balance >= 0`.
  - `locked_balance >= 0`.

#### Indexes

- `unique(member_id)` is also the primary lookup path for wallet and ledger updates.
- No settlement-specific lookup index is required on `point_account`; settlement baseline must use `crew_participant`, not `point_account` aggregation.

#### Notes

- Persist only `available_balance`, `reserved_balance`, and `locked_balance`.
- `settlement_pending_amount` is projection-only.
- `settlement_pending_balance` must not exist as a persisted DB/account column.
- Apply/reserve: `available_balance` decreases and `reserved_balance` increases.
- Approval: `reserved_balance` decreases and `locked_balance` increases. Approval is a bucket/state transition only and does not create a new ledger transaction type.
- Reserve release: `reserved_balance` decreases and `available_balance` increases.
- Settlement refund: `locked_balance` decreases and `available_balance` increases by the final refund amount.
- Use optimistic locking or conditional update predicates for balance mutation concurrency.
- No soft delete.

### 2.3 `point_history`

#### Table purpose

Append-only authoritative point ledger. It records every reserve, reserve release, and settlement refund effect with deterministic idempotency.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `member_id` | `BIGINT` | N | none | FK to `member.id`; all point events are member-scoped. |
| `transaction_type` | `VARCHAR(40)` | N | none | STRING enum ledger event type. |
| `amount` | `BIGINT` | N | none | Monetary event amount. |
| `available_after` | `BIGINT` | N | none | Reconciliation/debugging snapshot after event. |
| `reserved_after` | `BIGINT` | N | none | Reconciliation/debugging snapshot after event. |
| `locked_after` | `BIGINT` | N | none | Reconciliation/debugging snapshot after event. |
| `reference_type` | `VARCHAR(40)` | N | none | STRING enum reference type. |
| `reference_id` | `BIGINT` | N | none | Referenced aggregate PK. |
| `idempotency_key` | `VARCHAR(160)` | N | none | Deterministic duplicate-prevention key. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Ledger append time. |

#### Constraints

- PK: `primary key (id)`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE: `unique(idempotency_key)` / canonical name `unique(point_history.idempotency_key)`.
- CHECK:
  - `amount >= 0` unless the implementation explicitly standardizes signed amounts before migration.
  - `available_after >= 0`.
  - `reserved_after >= 0`.
  - `locked_after >= 0`.
  - `transaction_type in ('POINT_CHARGE', 'CREW_DEPOSIT_RESERVE', 'CREW_RESERVE_RELEASE', 'CREW_SETTLEMENT_REFUND')` if DB-level enum checks are used.
  - `reference_type in ('POINT_CHARGE', 'CREW_PARTICIPANT', 'SETTLEMENT_ITEM')` if DB-level enum checks are used.

#### Indexes

- `unique(idempotency_key)` for all retry/idempotency paths.
- `index(member_id, created_at)` for wallet history.
- `index(reference_type, reference_id)` for domain-to-ledger lookup.
- Optional operational index: `index(transaction_type, created_at)` for ledger audit/reporting if needed.

#### Notes

- `point_history` is append-only authoritative ledger.
- `available_after`, `reserved_after`, and `locked_after` are reconciliation/debugging snapshots, not authority over append-only ledger ordering or idempotency.
- `payload_hash` is deferred for MVP and is not required in this migration.
- Payload consistency verification framework is deferred for MVP.
- Same-key retry with the same canonical input should reuse/link the existing ledger row.
- Same-key retry with a different canonical input is an idempotency conflict and must not create a new ledger row.

### 2.4 `settlement`

#### Table purpose

Settlement header and execution claim row for one crew. It stores frozen aggregate totals, retry metadata, and completion state.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `crew_id` | `BIGINT` | N | none | FK to `crew.id`. |
| `status` | `VARCHAR(20)` | N | `PENDING` | STRING enum: `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRY_WAIT`. |
| `baseline_frozen_at` | `DATETIME(6)` | N | settlement creation/freeze time | Timestamp when LOCKED-only settlement baseline is frozen. |
| `batch_run_key` | `VARCHAR(100)` | Y | `NULL` | Worker/batch execution identifier. |
| `retry_count` | `INT` | N | `0` | Accumulated retry count. |
| `last_retry_at` | `DATETIME(6)` | Y | `NULL` | Last retry attempt timestamp. |
| `next_retry_at` | `DATETIME(6)` | Y | `NULL` | Optional retry scheduling timestamp. |
| `failure_code` | `VARCHAR(50)` | Y | `NULL` | Latest failure code. |
| `failure_message` | `VARCHAR(500)` | Y | `NULL` | Latest failure summary. |
| `total_participants` | `INT` | N | `0` | Count of `LOCKED` participants in frozen baseline. |
| `total_locked_amount` | `BIGINT` | N | `0` | Sum of baseline participant `deposit_amount`. |
| `total_recognized_success` | `INT` | N | `0` | Aggregate recognized success count. |
| `total_base_refund_amount` | `BIGINT` | N | `0` | Aggregate base refund amount. |
| `total_remainder_amount` | `BIGINT` | N | `0` | Deterministic remainder total. |
| `remainder_policy` | `VARCHAR(30)` | N | canonical MVP policy | STRING enum; MVP uses deterministic remainder allocation semantics. |
| `algorithm_version` | `VARCHAR(50)` | N | current settlement algorithm version | Historical semantic replay context. |
| `rule_context_snapshot` | `JSON` | N | none | Minimal opaque cadence/timezone/cutoff/lifecycle/remainder/reason mapping context required for historical settlement explanation. |
| `started_at` | `DATETIME(6)` | Y | `NULL` | Execution start timestamp. |
| `finished_at` | `DATETIME(6)` | Y | `NULL` | Execution finish timestamp. |
| `version` | `BIGINT` | N | `0` | Optimistic locking version. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time. |

#### Constraints

- PK: `primary key (id)`.
- FK: `crew_id -> crew.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE settlement strategy: `unique(crew_id)`.
- CHECK:
  - `status in ('PENDING', 'RUNNING', 'SUCCEEDED', 'FAILED', 'RETRY_WAIT')` if DB-level enum checks are used.
  - `retry_count >= 0`.
  - money aggregate columns are `>= 0`.
  - count aggregate columns are `>= 0`.

#### Indexes

- `unique(crew_id)` for duplicate settlement prevention.
- `index(status, retry_count, created_at)` for batch claim/retry scans.
- Optional: `index(next_retry_at, status)` if retry scheduling uses DB polling.

#### Notes

- Settlement baseline uses `LOCKED` participants only.
- `PENDING`, `REJECTED`, `CANCELLED`, and `EXPIRED` participants are excluded from settlement baseline.
- MVP has exactly one authoritative final settlement row per crew. Normal end and before-start cancellation are lifecycle/reason inputs to that row, not separate settlement types.
- `baseline_frozen_at` records when baseline selection is frozen; retry must complete the same settlement row rather than replace baseline semantics.
- `Settlement.status = SUCCEEDED` is allowed only after every `settlement_item` has a valid `point_history_id` and corresponding `point_history` row.
- Retry/replay/correction remain separated. Retry completes unfinished execution; replay is audit/reconstruction; correction workflow is deferred.

### 2.5 `settlement_item`

#### Table purpose

Participant-level settlement calculation snapshot and refund ledger linkage row.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `settlement_id` | `BIGINT` | N | none | FK to `settlement.id`. |
| `crew_participant_id` | `BIGINT` | N | none | FK to `crew_participant.id`. |
| `member_id` | `BIGINT` | N | none | FK to refund recipient `member.id`. |
| `participant_status_snapshot` | `VARCHAR(20)` | N | `LOCKED` | Frozen participant status snapshot. |
| `deposit_amount` | `BIGINT` | N | none | Locked deposit snapshot used for calculation. |
| `success_count_raw` | `INT` | N | `0` | Raw success log count. |
| `recognized_success_count` | `INT` | N | `0` | Settlement-recognized success count. |
| `recognized_dates_count` | `INT` | N | `0` | Recognized date count. |
| `excluded_success_count` | `INT` | N | `0` | Excluded success count. |
| `period_start_at` | `DATETIME(6)` | N | none | Calculation period start. |
| `period_end_at` | `DATETIME(6)` | N | none | Calculation period end. |
| `share_ratio` | `DECIMAL(18,8)` | N | none | Calculation share ratio. |
| `refund_amount` | `BIGINT` | N | `0` | Final credited/refunded amount. Other per-item amount values are calculation intermediates and are not persisted in MVP. |
| `draw_key_snapshot` | `CHAR(64)` | Y | `NULL` | Non-payout display/explanation ordering key. |
| `tie_break_rank` | `INT` | Y | `NULL` | Non-payout display/explanation rank. |
| `calculation_reason` | `JSON` | N | none | Minimal opaque inclusion/exclusion reason context required for MVP explanation/replay. Do not model it as query-heavy JPA subgraphs. |
| `point_history_id` | `BIGINT` | Y | `NULL` | FK to refund ledger row; nullable until payout link completes. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time / ledger link completion time. |

#### Constraints

- PK: `primary key (id)`.
- FK: `settlement_id -> settlement.id` with `RESTRICT` / `NO ACTION`.
- FK: `crew_participant_id -> crew_participant.id` with `RESTRICT` / `NO ACTION`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- FK: `point_history_id -> point_history.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE: `unique(settlement_id, crew_participant_id)`.
- UNIQUE: `unique(point_history_id)` where DB dialect handles nullable unique semantics as expected; otherwise enforce non-null uniqueness in application/partial index where supported.
- CHECK:
  - `participant_status_snapshot = 'LOCKED'` for canonical MVP settlement rows.
  - money amount columns are `>= 0`.
  - count columns are `>= 0`.

#### Indexes

- `index(member_id)` for member refund history.
- `index(crew_participant_id)` for participant settlement lookup.

#### Notes

- `settlement_item` is the participant-level calculation snapshot authority after settlement success.
- `point_history_id` may be nullable during partial retry, but every item must have a valid `point_history_id` before parent `settlement.status = SUCCEEDED`.
- Same participant must not have duplicate items for one settlement.
- `point_history` is the actual balance mutation authority; `settlement_item` explains the calculation and links to that ledger row.

## 3. Canonical transaction types

Active canonical MVP ledger transaction types:

- `POINT_CHARGE`: created for successful point charge confirmation.
- `CREW_DEPOSIT_RESERVE`: created when application reserve succeeds and participant enters `PENDING`.
- `CREW_RESERVE_RELEASE`: created once when `PENDING` transitions to `REJECTED`, `CANCELLED`, or `EXPIRED` and reserve is returned.
- `CREW_SETTLEMENT_REFUND`: created for final settlement refund linked to `settlement_item`.

Forbidden for MVP active implementation:

- `CREW_DEPOSIT_LOCK` must not be introduced as an active ledger type.
- Approval-time ledger creation is prohibited. Approval is only `crew_participant.status` plus balance bucket transition from `reserved_balance` to `locked_balance`.


## 4. Idempotency strategy

Canonical idempotency key formats:

- Reserve: `crew:{crewId}:participant:{participantId}:reserve`
- Reserve release: `crew:{crewId}:participant:{participantId}:reserve-release`
- Settlement refund: `crew:{crewId}:participant:{participantId}:settlement-refund`

Rules:

- `point_history.idempotency_key` is `NOT NULL` and `UNIQUE`.
- Reserve release is allowed once per `crew_participant.id`.
- Reserve release terminal transition and `CREW_RESERVE_RELEASE` ledger insertion happen in the same transaction.
- `crew_participant.released_point_history_id` links the terminal participant row to the authoritative release ledger row.
- Settlement refund is allowed once per `settlement_item.id` / participant settlement item.
- `settlement_item.point_history_id` links the settlement calculation snapshot to the authoritative refund ledger row.
- Runtime-generated `settlement.id` is linkage metadata, not authoritative idempotency identity. Settlement linkage is tracked through `settlement_item` and `point_history` linkage.
- Same-key retry with same canonical input returns/reuses the existing `point_history` and linkage.
- Same-key retry with different canonical input is an idempotency conflict; do not create a second ledger row.
- `payload_hash` is deferred for MVP, so same/different canonical input checks are application-level rules, not a persisted payload consistency framework.

## 5. Projection rules

Wallet/projection fields:

- `active_locked_amount`: projection-only split of `locked_balance` for active/recruiting/ongoing crews.
- `settlement_pending_amount`: projection-only split of `locked_balance` for ended crews before final settlement success.
- `locked_balance`: persisted `point_account` cache/source.

Invariant:

```text
locked_balance == active_locked_amount + settlement_pending_amount
```

Rules:

- Projection is current-basis and non-final before `Settlement.status = SUCCEEDED`.
- Projection does not determine settlement baseline, refund authority, or final payout.
- Settlement baseline and final refund use frozen `LOCKED` participant snapshots and `settlement_item` / `point_history` linkage.
- `settlement_pending_balance` must not exist as a persisted DB/account column.

## 6. Migration sequencing

Recommended Flyway-style order:

1. Ensure prerequisite core `member` and `crew` tables exist.
2. Create `crew_participant` with lifecycle columns and `unique(crew_id, member_id)`.
3. Create `point_account` with `available_balance`, `reserved_balance`, `locked_balance`, and `version`.
4. Create `point_history` with append-only ledger columns and `unique(idempotency_key)`.
5. Add/verify `crew_participant.released_point_history_id -> point_history.id` FK if circular ordering requires a post-create `ALTER TABLE`.
6. Create `settlement` with status, baseline freeze, retry metadata, aggregate snapshots, and `version`.
7. Create `settlement_item` with calculation snapshot fields and nullable `point_history_id` linkage.
8. Add indexes and constraint hardening after base tables exist.
9. Add DB-level CHECK constraints only where supported consistently by the chosen RDBMS and migration policy; otherwise enforce the same rules at application/JPA validation layer.

## 7. Explicitly deferred MVP hardening

Do not implement these in this migration round:

- `payload_hash`
- Payload consistency verification framework
- Withdrawal / rejoin lifecycle
- Correction workflow
- Private crew semantics
- Automatic replay recovery engine
