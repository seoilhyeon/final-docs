# Settlement Test Matrix / Contract Alignment Runbook

## BATCH-000 scope

This runbook is the docs-only handoff for settlement batch work. It records the active settlement contract, known drift, and future green-slice tests without changing frontend/backend source, schema, or test code.

Hard boundary: if a mismatch requires implementation, keep it as a follow-up item here. Do not add failing tests, disabled tests, schema changes, or speculative API fields in BATCH-000.

## Canonical contract snapshot

### Settlement statuses

| Status | Meaning | Persistence |
| --- | --- | --- |
| `NONE` | No `Settlement` row exists yet. | API projection only. `NONE` MUST NOT be persisted in `Settlement.status`. |
| `PENDING` | Settlement row exists and is waiting to be claimed. | `Settlement.status` |
| `RUNNING` | A worker/operator has claimed settlement execution. | `Settlement.status` |
| `SUCCEEDED` | All settlement items are linked to verified `point_history` rows. | `Settlement.status` |
| `FAILED` | Retry is exhausted or the failure is terminal. | `Settlement.status` |
| `RETRY_WAIT` | Recoverable failure waiting for retry. | `Settlement.status` |

### Settlement failure codes

`INPUT_LOAD_FAILED`, `CALCULATION_FAILED`, `POINT_CREDIT_FAILED`, `DUPLICATE_SETTLEMENT`, `LOCK_ACQUIRE_FAILED`, `UNKNOWN`.

### Public response contract

- `GET /api/crews/{crewId}/settlement` returns crew-level settlement status: `crew_id`, `settlement_id`, `status`, `retry_count`, `failure_code`, `failure_message`, `started_at`, `finished_at`.
- `GET /api/settlements/{settlementId}` returns final/partial settlement detail with header totals and `items[]`.
- Active detail response uses header-level `remainder_policy` and item-level `remainder_bonus_amount`.
- Active detail response must not expose `remainder_winner_*` unless the product/API contract is explicitly changed later.
- `HOST_REMAINDER` is a deterministic fixed policy, not host discretion, random winner selection, payout mutation authority, or ledger authority.
- Final payout authority is `settlement_item.refund_amount` plus linked `point_history`.

## Drift findings

| ID | Finding | Evidence | BATCH-000 action | Follow-up owner |
| --- | --- | --- | --- | --- |
| DRIFT-001 | FE type contract includes `settlement_type` on `CrewSettlementSummary` and `SettlementDetail`, but active settlement API examples/field tables do not expose it. | `frontend/src/types/domain.ts`; `docs/API-spec-dondok.md`; `backend/docs/api/settlement.md` | Record only; no FE/source edit in BATCH-000. | FE/API contract follow-up before FE settlement view implementation. |
| DRIFT-002 | FE type contract includes `remainder_winner_crew_participant_id`, but active docs state remainder is represented by `remainder_policy` + item-level `remainder_bonus_amount`. | `frontend/src/types/domain.ts`; BATCH-000 PRD acceptance criterion; API spec drift note | Record only; no FE/source edit in BATCH-000. | FE type alignment follow-up; remove/ignore field unless API spec changes. |
| DRIFT-003 | Active backend settlement API doc has readable field names/examples, but much prose appears mojibake in current checkout. | `backend/docs/api/settlement.md` | Avoid semantic rewrite in BATCH-000; add this runbook as clear handoff. | Docs hygiene follow-up if the team wants prose repair. |
| DRIFT-004 | Status list and failure-code list are aligned between integrated API spec and FE enum values. | `docs/API-spec-dondok.md`; `frontend/src/types/domain.ts` | No change needed. | None. |
| DRIFT-005 | API spec already documents that `NONE` is projection-only and `SUCCEEDED` requires settlement item + point history linkage. | `docs/API-spec-dondok.md` | No change needed. | None. |

## Future green-slice test matrix

### BATCH-005 calculator unit tests

- [ ] Equal success split returns equal share/refund.
- [ ] Unequal success counts produce scale-6 `share_ratio` using `RoundingMode.FLOOR`.
- [ ] Floor remainder is assigned only by `HOST_REMAINDER` policy.
- [ ] All-fail total recognized success 0 returns principal refund for all participants and no remainder bonus.
- [ ] Duplicate or excluded logs increase raw/excluded counts but not recognized count.
- [ ] Last 3 mission days are frozen without 72h grace.

### BATCH-006 settlement API contract tests

- [ ] Crew settlement summary returns `NONE` with `settlement_id: null` when no row exists.
- [ ] Summary returns persisted status/failure fields when a row exists.
- [ ] Detail returns settlement items with snake_case fields and no internal member IDs.
- [ ] Detail does not expose `remainder_winner_*` fields.
- [ ] Detail omits or deliberately documents `settlement_type`; do not let FE assumptions define the API implicitly.

### BATCH-001 / BATCH-002 batch execution tests

- [ ] Eligible ended/cancelled crew creates exactly one `Settlement(PENDING)` under `unique(crew_id)`.
- [ ] `PENDING`/`RETRY_WAIT` row is claimable by conditional update.
- [ ] Concurrent loser claim skips safely without payout.
- [ ] Successful run writes settlement item snapshots, participant-level idempotent point histories, and only then `SUCCEEDED`.

### BATCH-003 all-fail integration case

- [ ] Total recognized success 0 credits each participant original deposit amount.
- [ ] `share_ratio`, `refund_amount`, and reason fields remain deterministic and explain all-fail policy.

### BATCH-004 retry / observability tests

- [ ] Point credit failure leaves settlement in `RETRY_WAIT` before retry limit.
- [ ] Retry resumes from existing settlement items and idempotency keys.
- [ ] Retry exhaustion marks `FAILED` with failure code.
- [ ] Metrics/logs expose job/step status and settlement failure-code counts.

### FE-SETTLE-001 fixture/view cases

- [ ] `NONE`: settlement not started / waiting copy.
- [ ] `PENDING`: waiting for settlement processing.
- [ ] `RUNNING`: processing copy.
- [ ] `RETRY_WAIT`: automatic recovery/retry copy.
- [ ] `FAILED`: user-safe error copy and contact/support path.
- [ ] `SUCCEEDED`: final result detail.
- [ ] All-fail succeeded result: principal refund copy.
- [ ] Remainder case: render `remainder_bonus_amount` without implying winner lottery or host discretion.

## WBS handoff notes

| Row | Cell-size description |
| --- | --- |
| BATCH-000 | Align settlement status/failure/response contract and future green-slice test matrix; docs-only, no failing tests. |
| BATCH-005 | Implement calculator with one green unit-test case at a time from the BATCH-000 matrix. |
| BATCH-006 | Implement settlement summary/detail API against the aligned contract and add contract tests. |
| FE-SETTLE-001 | Build settlement status/exception/result views from aligned fixtures; do not invent API fields. |

## Completion checks for BATCH-000

- [x] Mismatches identified.
- [x] Docs/runbook updated only.
- [x] Implementation-required mismatches recorded as follow-ups.
- [x] No source code, schema, or test code changes required for this batch.
- [x] No disabled/failing tests added as placeholders.
