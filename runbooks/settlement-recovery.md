# Settlement Recovery Runbook

## 원칙

- 복구 기준은 MySQL row, `Settlement.status`, `settlement_item`, `point_history`, DB unique constraint, deterministic idempotency key다.
- Redis/Redisson 상태는 복구 판단의 source of truth가 아니다.
- 운영자는 `point_history`를 직접 수정/삭제하지 않는다.
- 운영자는 `settlement_item` 금액을 직접 변경하지 않는다.
- 운영자는 `Settlement.status`를 수동 DB update로 `SUCCEEDED` 처리하지 않는다.
- Admin UI 없이 관리자 API, CloudWatch, structured log로 복구한다.

## 확인 절차

1. CloudWatch alarm을 확인한다.
2. application log에서 `settlement_id`, `failure_code`, `retry_count`를 확인한다.
3. `GET /api/admin/settlements`로 대상 settlement 상태를 확인한다.
4. `settlement_item.point_history_id` 누락 여부를 확인한다.
5. `point_history.idempotency_key` 기준 기존 원장 존재 여부를 확인한다.
6. 재시도 가능 상태면 `POST /api/admin/settlements/{settlementId}/retry`를 실행한다.

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
- 필요 시 운영자 수동 재발송을 수행한다.

## 금지

- Redis lock 상태만 보고 payout 완료로 판단하지 않는다.
- `point_history`를 직접 수정/삭제하지 않는다.
- `settlement_item` 금액을 직접 변경하지 않는다.
- `Settlement.status`를 수동 DB update로 `SUCCEEDED` 처리하지 않는다.
- direct DB mutation을 정상 복구 경로로 사용하지 않는다.
