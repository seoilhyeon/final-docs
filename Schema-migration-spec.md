# Dondok MVP Schema Migration Spec

Status: canonical implementation-ready migration specification for MVP backend schema.

Authoritative references:

- `backend/docs/api/*` — MVP active API source
- `docs/API-spec-dondok.md` — backend API 기준 integrated synchronized contract
- `docs/PRD-dondok.md` / `docs/Usecase-dondok.md` — semantic guardrail lane
- `docs/ERD-dondok.md`
- `docs/Settlement-design.md`
- `docs/Implementation-guardrails.md`

Scope:

- This document is DB/Flyway/JPA oriented.
- Code generation must wait for docs freeze: entity/enums/Flyway migrations are created from the finalized ERD/Schema/guardrail docs.
- It gives detailed Flyway guidance for high-risk core MVP backend entities: `crew_participant`, `point_account`, `point_history`, `settlement`, `settlement_item`.
- Greenfield V1 must create entities/migrations from finalized docs, not patch assumed existing entities. `docs/ERD-dondok.md` owns the active table/relationship set; this spec provides migration ordering and high-risk table guidance plus the minimal notification persistence required by the active notification API.
- It does not redesign product semantics, lifecycle semantics, settlement semantics, or Deferred/Brownfield/Removed domains. It does not create active endpoint/status/feature semantics absent from backend API docs/API-spec.

## 1. Schema conventions

Canonical rules for this migration round:

- PK strategy: every table primary key is `BIGINT` auto increment.
- Money type: every persisted monetary amount uses `BIGINT` only.
- Enum persistence: application enums are stored as `STRING` (`VARCHAR`) values, not ordinal integers.
- External UUID columns (e.g. `member.uuid`, `notification.uuid`) use `BINARY(16)` for DB storage. API serialization renders the canonical `CHAR(36)` form. This is the single project-wide convention; individual table specs do not re-decide UUID storage type.
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
- Terminal `crew_participant` rows are preserved with no physical delete. Eligible pre-approval reapply reuses only the existing `CANCELLED` row in place (`CANCELLED -> PENDING`) and creates no new row; `REJECTED`/`EXPIRED` remain terminal and are not reused.

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
- `REJECTED` and `EXPIRED` are terminal rows and remain preserved. `CANCELLED` is a pre-start exit row that is preserved and is also reopen-eligible.
- MVP allows re-application to the same crew only when the existing row is `CANCELLED`, the crew is `RECRUITING`, server time is before `recruitment_deadline`, capacity is available, and the participant has sufficient balance to re-reserve. `REJECTED`/`EXPIRED` rows block re-application. `unique(crew_id, member_id)` is the DB enforcement point; reopen reuses the existing row in place (no new row).
- Reserve release is allowed once per current reserve cycle of `crew_participant.id`; terminal/`CANCELLED` transition and reserve release happen in the same transaction. On `CANCELLED -> PENDING` reopen, the same transaction resets `released_point_history_id` to `NULL` and inserts a new `CREW_DEPOSIT_RESERVE` `point_history` row, starting the next cycle; the prior `CREW_RESERVE_RELEASE` row is retained as append-only audit.
- `released_point_history_id` is the authoritative evidence that reserve release occurred. Do not rely on timestamp-only release evidence.
- No `WITHDRAWN`, `withdrawn_at`, ACTIVE withdrawal, or rejoin semantics are introduced in this MVP migration. They are Deferred/Brownfield/Removed historical/reference-only, not active MVP implementation surface, future delivery commitment, or implementation permission.

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
- No `point_charge` table is created. Point charge confirmation appends directly to `point_history` with `transaction_type = POINT_CHARGE`, `reference_type = POINT_CHARGE`, `reference_id = created point_history.id`, and `idempotency_key = charge:{paymentKey}`.
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
| `last_retry_at` | `DATETIME(6)` | Y | `NULL` | Deferred/runtime scheduling metadata candidate. Not required active MVP persistence and not API/settlement authority. |
| `next_retry_at` | `DATETIME(6)` | Y | `NULL` | Deferred/runtime scheduling metadata candidate for optional DB polling. Do not treat as active MVP scheduler semantics. |
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
- Deferred/optional: `index(next_retry_at, status)` only if a later runtime scheduling implementation adopts DB polling; not an active MVP migration requirement.

#### Notes

- Settlement baseline uses `LOCKED` participants only.
- `PENDING`, `REJECTED`, `CANCELLED`, and `EXPIRED` participants are excluded from settlement baseline.
- MVP has exactly one authoritative final settlement row per crew. Normal end and before-start cancellation are lifecycle/reason inputs to that row, not separate settlement types.
- `baseline_frozen_at` is persisted freeze evidence for the LOCKED-only baseline; retry must complete the same settlement row rather than replace baseline semantics.
- `Settlement.status = SUCCEEDED` is allowed only after every `settlement_item` has a valid `point_history_id` and corresponding `point_history` row.
- Retry/replay/correction remain separated. Retry completes unfinished existing settlement execution; replay is audit/reconstruction; correction workflow is Deferred/Brownfield historical/reference-only and not implemented by this migration.
- `last_retry_at` / `next_retry_at` may be omitted from V1 unless the runtime intentionally implements DB polling/scheduling. If retained, they remain Deferred/runtime metadata candidates and never active scheduler/API authority.

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
| `base_refund_amount` | `BIGINT` | N | `0` | Pre-remainder base refund snapshot. Explanation column, not payout authority. |
| `remainder_bonus_amount` | `BIGINT` | N | `0` | Deterministic remainder allocation share snapshot. Explanation column, not payout authority. |
| `reward_amount` | `BIGINT` | N | `0` | `base_refund_amount + remainder_bonus_amount` snapshot. Explanation column, not payout authority. |
| `refund_amount` | `BIGINT` | N | `0` | Final credited/refunded amount. Persisted per-item payout source of truth. API response `final_amount` is a read-only alias for this column. Invariant: `refund_amount = reward_amount = base_refund_amount + remainder_bonus_amount`. |
| `withdrawn_at_snapshot` | `DATETIME(6)` | Y | `NULL` | Settlement-time `crew_participant.withdrawn_at` snapshot. Deferred/Brownfield historical/reference-only; always `NULL`/ignored in MVP active settlement. |
| `effective_moderation_snapshot` | `JSON` | Y | `NULL` | Settlement-time latest-effective moderation state snapshot. Read-only audit/replay context. |
| `moderation_chain_ref` | `JSON` | Y | `NULL` | Settlement-time `moderation_history` chain reference (e.g. `{"latest_id":..., "count":...}`). Audit linkage, not payout authority. |
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

## 5. Implementation enum / column notes

- `SettlementStatus.NONE` is API projection only and must not be stored as a DB settlement status.
- `ParticipantStatus` active persistence values are only `PENDING`, `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`.
- `FrequencyType` active values are only `DAILY`, `SPECIFIC_DAYS`; `WEEKLY_N` is not active V1 cadence.
- `MissionLogFailureReason` excludes `AFTER_WITHDRAWN` in MVP active persistence.
- `MissionLogReactionType` should remain free `VARCHAR(20)` for MVP unless a later source-of-truth freezes a bounded Java enum.
- `participant_status_snapshot` is `LOCKED` only for MVP active settlement.
- `member.password_hash` may remain DB nullable; email/password signup enforces password presence at service level.
- `crew.image_s3_key` is nullable and UI may use category/default fallback.
- `withdrawn_at_snapshot` is always `NULL`/ignored in MVP active settlement.

## 6. Projection rules

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

## 7. Minimal notification persistence

The active notification API requires minimal server-side persistence for FCM device lifecycle and inbox/read/unread. These tables are UX/refetch support only; they are not certification, settlement, lifecycle, ledger, or audit authority.

### 7.1 `notification_device`

#### Table purpose

Authenticated member Android FCM device/token registration for active device register/update/delete API.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `member_id` | `BIGINT` | N | none | FK to `member.id`. |
| `device_id` | `VARCHAR(100)` | N | none | Client device/installation identifier. |
| `platform` | `VARCHAR(20)` | N | `ANDROID` | MVP active platform value is `ANDROID`. |
| `fcm_token` | `VARCHAR(512)` | N | none | Current FCM token for the member/device. |
| `app_version` | `VARCHAR(50)` | Y | `NULL` | Optional app version metadata. |
| `enabled` | `BOOLEAN` | N | `TRUE` | Whether this registration is active for sending. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time. |

#### Constraints / indexes

- PK: `primary key (id)`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- UNIQUE: `unique(member_id, device_id)`.
- INDEX: `index(member_id, enabled)`.
- Optional INDEX: `index(fcm_token)` only if token lookup is needed; do not require global unique token semantics in V1.

#### Notes

- This table does not freeze token refresh, invalid-token, provider retry, or delivery attempt lifecycle semantics.
- Delete API may disable or delete a row according to implementation policy, but neither choice changes domain state authority.
- Same `(member_id, device_id)` re-register updates the existing row in place: refresh `fcm_token`, set `enabled = TRUE`, bump `updated_at`. It does not insert a duplicate row.
- A soft-disabled row (`enabled = FALSE`) is re-enabled by the same `(member_id, device_id)` re-register through the same in-place update path. `unique(member_id, device_id)` guarantees no duplicate device rows.

### 7.2 `notification`

#### Table purpose

Per-member notification inbox/read row for active inbox, unread count, mark-read, and read-all API.

#### Columns

| Column | DB type | Nullable | Default | Meaning / purpose |
| --- | --- | --- | --- | --- |
| `id` | `BIGINT` | N | auto increment | Primary key. |
| `uuid` | `BINARY(16)` | N | generated | External `notification_id`. DB stores `BINARY(16)`; API serializes canonical `CHAR(36)` per §1 convention. |
| `member_id` | `BIGINT` | N | none | FK to receiving `member.id`. |
| `event_type` | `VARCHAR(80)` | N | none | App routing/UI vocabulary, not DB enum/audit catalog. |
| `resource_type` | `VARCHAR(50)` | N | none | Linked resource type string. |
| `resource_id` | `VARCHAR(100)` | N | none | Linked resource identifier string. |
| `deep_link` | `VARCHAR(255)` | N | none | App URL scheme for navigation. |
| `display_text` | `VARCHAR(500)` | N | none | Server-generated display text; not final state. |
| `requires_refetch` | `BOOLEAN` | N | `TRUE` | MVP treats every notification as requiring canonical refetch. |
| `occurred_at` | `DATETIME(6)` | N | none | Event occurrence time for ordering. |
| `read_at` | `DATETIME(6)` | Y | `NULL` | Read timestamp. `NULL` means unread. |
| `created_at` | `DATETIME(6)` | N | current timestamp | Audit create time. |
| `updated_at` | `DATETIME(6)` | N | current timestamp | Audit update time. |

#### Constraints / indexes

- PK: `primary key (id)`.
- UNIQUE: `unique(uuid)`.
- FK: `member_id -> member.id` with `RESTRICT` / `NO ACTION`.
- INDEX: `index(member_id, occurred_at, id)` or an equivalent cursor-supporting order index.
- INDEX: `index(member_id, read_at)` for unread count and read-all updates.

#### Notes

- `read_at IS NULL` is the only unread rule. Do not add notification status enum/workflow/status machine.
- Inbox/read rows are non-authoritative UX/refetch hints. They do not own crew lifecycle, certification, moderation, settlement, point ledger, or unresolved task truth.
- Do not create `notification_delivery_attempt`, notification preference matrix, template CMS, campaign/broadcast, SSE/stream, or notification transport redesign tables in V1.

## 8. Migration sequencing

Flyway readiness:

- `backend/build.gradle` must include `org.flywaydb:flyway-core` and `org.flywaydb:flyway-mysql` before implementation is considered migration-ready.
- Use `src/main/resources/db/migration/V1__init.sql` as the initial greenfield schema migration.
- Validate V1 with Testcontainers MySQL; H2-only validation is not sufficient for MySQL FK/check/index behavior.

Recommended Flyway-style order:

1. Create prerequisite identity/domain tables from ERD, including `member` and `crew`.
2. Create `crew_participant` with lifecycle columns and `unique(crew_id, member_id)`.
3. Create `point_account` with `available_balance`, `reserved_balance`, `locked_balance`, and `version`.
4. Create `point_history` with append-only ledger columns and `unique(idempotency_key)`.
5. Add/verify `crew_participant.released_point_history_id -> point_history.id` FK if circular ordering requires a post-create `ALTER TABLE`.
6. Create `settlement` with status, baseline freeze, retry metadata, aggregate snapshots, and `version`.
7. Create `settlement_item` with calculation snapshot fields and nullable `point_history_id` linkage.
8. Create minimal notification tables after `member` exists: `notification_device`, then `notification`.
9. Add indexes and constraint hardening after base tables exist.
10. Add DB-level CHECK constraints only where supported consistently by the chosen RDBMS and migration policy; otherwise enforce the same rules at application/JPA validation layer.

## 9. Explicitly deferred MVP hardening

Do not implement these in this migration round. These are historical/reference-only or future hardening candidates, not active MVP implementation permission or future delivery commitment:

- `payload_hash`
- Payload consistency verification framework
- Withdrawal / rejoin lifecycle (`WITHDRAWN`, `withdrawn_at`, ACTIVE withdrawal)
- `WEEKLY_N` active cadence
- Admin settlement mutation/list/retry public API surface
- Correction workflow
- Private crew semantics
- Automatic replay recovery engine
- Notification delivery topology redesign, delivery attempt persistence expansion, SSE/stream
- AI habit report/report lifecycle
