# Settlement Recovery Runbook

## Authority boundary

이 runbook은 `backend/docs/api/*`와 `docs/API-spec-dondok.md` 아래의 internal/incomplete settlement recovery 절차다. 여기의 운영 절차는 public active API contract, admin mutation authority, correction/recalculation authority가 아니다.

Legacy/Brownfield 문구로 남아 있던 `GET /api/admin/settlements`와 `POST /api/admin/settlements/{settlementId}/retry` 형태는 historical/internal recovery surface reference only다. backend active API contract가 별도로 승격하기 전까지 active MVP API, future delivery commitment, implementation permission이 아니다.
Deferred/Brownfield/Removed recovery wording is historical/reference-only and does not grant implementation permission.

## 원칙

- 복구 기준은 MySQL row, `Settlement.status`, `settlement_item`, `point_history`, DB unique constraint, deterministic idempotency key다.
- `settlement_item + point_history`가 성공 정산 이후 final authority다.
- retry는 unfinished existing settlement completion이며 correction, recalculation, replay engine, payout rewrite가 아니다.
- Redis/Redisson 상태는 복구 판단의 source of truth가 아니다.
- Notification/FCM/SSE/inbox/read/AI report 상태는 settlement state evidence가 아니다.
- 운영자는 `point_history`를 직접 수정/삭제하지 않는다.
- 운영자는 `settlement_item` 금액을 직접 변경하지 않는다.
- 운영자는 `Settlement.status`를 수동 DB update로 `SUCCEEDED` 처리하지 않는다.
- Admin UI/API가 없더라도 CloudWatch, structured log, 승인된 internal batch/operator trigger로 복구한다. 이 문서는 새 public admin endpoint를 설계하지 않는다.

## 확인 절차

1. CloudWatch alarm을 확인한다.
2. application log에서 `settlement_id`, `failure_code`, `retry_count`를 확인한다.
3. 승인된 internal operator surface 또는 read-only DB/log inspection으로 대상 settlement 상태를 확인한다. Legacy `GET /api/admin/settlements` 문구는 active public API가 아니다.
4. `settlement_item.point_history_id` 누락 여부를 확인한다.
5. `point_history.idempotency_key` 기준 기존 원장 존재 여부를 확인한다.
6. 재시도 가능 상태(`FAILED` 또는 `RETRY_WAIT`)면 승인된 internal recovery trigger로 기존 `Settlement` row를 다시 claim한다. Legacy `POST /api/admin/settlements/{settlementId}/retry` 문구는 active public API가 아니며, retry 의미는 unfinished settlement completion으로 제한한다.

## 이메일 실패 확인

- 이메일 실패는 정산 실패가 아니다.
- SMTP structured log에서 아래 필드를 확인한다.
  - `settlement_id`
  - `member_id`
  - `email_type`
  - `recipient_hash`
  - `attempt`
  - `result`
  - `smtp_error_code`
  - `created_at`
- 필요 시 운영자 수동 재발송을 수행한다. 이 재발송은 settlement retry/replay/correction이 아니다.

## Redis unavailable 복구 절차

Redis 장애는 정산 correctness source 상실이 아니라 보조 계층 장애다. 단, Redis lock 상태를 확인할 수 없다는 이유로 일반 worker가 금전성 실행을 강행하면 안 된다.

1. CloudWatch alarm 또는 application log에서 Redis unavailable 상태를 확인한다.
2. 일반 settlement worker가 새로운 금전성 지급/정산 실행을 fail-closed 또는 controlled retry로 전환했는지 확인한다.
3. 복구가 필요한 settlement는 승인된 internal operator surface 또는 read-only DB/log inspection으로 `PENDING`, `RETRY_WAIT`, stale `RUNNING` timeout 후보를 확인한다.
4. stale `RUNNING` timeout 후보는 DB-claim-only fallback을 직접 시도하지 않는다. 먼저 timeout recovery path로 `RETRY_WAIT` 전환을 수행한다.
5. 운영자는 승인된 internal batch/operator trigger로 DB-claim-only fallback을 실행한다. 직접 DB update로 claim하지 않는다.
6. fallback 실행은 `Settlement.status in (PENDING, RETRY_WAIT)` 조건부 update로 `RUNNING` claim을 시도한다.
7. update row count가 `1`이면 해당 실행자만 실행권을 가진다. row count가 `0`이면 다른 worker/operator가 먼저 claim한 것이므로 skip하고 상태를 재조회한다.
8. participant별 중복 지급 여부는 `point_history.idempotency_key` unique와 기존 `settlement_item.point_history_id` 연결 상태로 확인한다.
9. 복구 완료 전 모든 `settlement_item.point_history_id`와 대응 `point_history` 존재를 검증한다.

이 fallback은 Redis를 우회하지만 MySQL ownership semantics를 사용하는 정상 복구 경로다. `point_history`, `settlement_item` 금액, `Settlement.status = SUCCEEDED`를 직접 수정하는 절차가 아니다.

## 운영 관측 최소 신호

CloudWatch/log에서 최소한 아래 신호를 확인할 수 있어야 한다. threshold와 세부 metric 이름은 구현 중 조정할 수 있지만, 신호 자체를 제거하지 않는다.

- settlement batch failure
- `RUNNING` timeout
- `RETRY_WAIT` 또는 `FAILED` 증가
- Redis unavailable
- DB connection failure
- payment confirm failure
- idempotency conflict spike
- reconciliation mismatch
- disk usage 또는 instance health

정산/복구 structured log는 최소 아래 식별자를 포함한다.

- `settlement_id`
- `room_id`
- `settlement_type`
- `status`
- `failure_code`
- `retry_count`
- `batch_run_key`
- `started_at`
- `finished_at`
- worker/operator identifier
- `participant_id`
- `member_id`
- `idempotency_key`
- `point_history_id`
- recovery action/result

## Forbidden Recovery Paths

아래는 break-glass emergency를 제외하고 정상 복구 경로가 아니다. break-glass가 필요하면 incident record, audit, post-repair invariant validation을 남긴다.

- Redis lock 상태만 보고 payout 완료로 판단하지 않는다.
- `point_history`를 직접 수정/삭제하지 않는다.
- `settlement_item` 금액을 직접 변경하지 않는다.
- `Settlement.status`를 수동 DB update로 `SUCCEEDED` 처리하지 않는다.
- `point_account.balance`만 직접 고쳐 정합성을 맞춘 것으로 간주하지 않는다. balance 보정은 `point_history` 재계산 결과를 기준으로 한다.
- `point_history`는 존재하지만 `settlement_item.point_history_id`가 누락된 상태에서 새 환급 원장을 생성하지 않는다. 기존 원장 payload 확인 후 FK 연결 보정 대상으로 다룬다.
- parent `SUCCEEDED` + null `settlement_item.point_history_id` 상태를 정상 완료로 안내하지 않는다.
- direct DB mutation을 정상 복구 경로로 사용하지 않는다.
- legacy admin settlement list/retry endpoint wording을 active MVP API나 admin mutation authority로 해석하지 않는다.
- retry를 correction, recalculation, replay, payout rewrite로 사용하지 않는다.
