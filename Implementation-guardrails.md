# Dondok Backend Implementation Guardrails

이 문서는 Dondok MVP 백엔드 구현 전에 고정된 lifecycle, balance, ledger, settlement 불변식을 모은 implementation source-of-truth다. PRD를 대체하지 않으며, 구현 중 semantic drift를 막기 위한 guardrail로 사용한다.


## 0. Schema implementation conventions

- Primary keys use `BIGINT` auto increment.
- Monetary amounts use `BIGINT` only. Do not use floating-point money types.
- FK delete policy is `RESTRICT` / `NO ACTION` for domain, money, and audit references.
- Enums are stored as `STRING` values in persistence.
- Standard audit columns are `created_at` and `updated_at`.
- Version-based optimistic locking is required for `point_account`, `crew_participant`, and `settlement`.
- Money/audit entities must not use soft delete.

## 1. Canonical lifecycle semantics

### `crew_participant` lifecycle

MVP active status는 아래 다섯 개만 사용한다.

- `PENDING`
- `LOCKED`
- `REJECTED`
- `CANCELLED`
- `EXPIRED`

`APPLIED`, `JOINED`, `APPROVED_LOCK_PENDING` 같은 중간/legacy 상태를 구현 상태로 재도입하지 않는다.

### `PENDING` reserve semantics

- `PENDING`은 신청 제출과 보증금 reserve가 함께 완료된 상태다.
- `PENDING` 생성 시 `available_balance`는 감소하고 `reserved_balance`는 증가한다.
- `PENDING`은 capacity reservation에는 포함한다.
- `PENDING`은 activation eligibility, minimum participant baseline, frozen settlement baseline에는 포함하지 않는다.

### `LOCKED` baseline semantics

- `LOCKED`는 승인 완료 + reserved deposit이 locked deposit으로 확정된 상태다.
- activation eligibility, frozen participant baseline, settlement baseline에는 `LOCKED`만 포함한다.
- 승인은 `reserved_balance -> locked_balance` bucket/state transition이다.
- 승인 시 새로운 `point_history` transaction type을 만들지 않는다.

### Terminal state preservation

- `REJECTED`, `CANCELLED`, `EXPIRED`는 terminal state다.
- terminal `crew_participant` row는 삭제하지 않는다.
- terminal row는 audit, duplicate-apply prevention, no-reapplication enforcement의 근거다.

### No re-application in MVP

- MVP에서는 같은 `crew`에 같은 `member`가 재신청할 수 없다.
- DB는 `unique(crew_id, member_id)`로 이를 강제한다.
- terminal row를 삭제하거나 재사용해서 재신청처럼 보이게 만들지 않는다.

## 2. Balance model

### Persisted account balances

`point_account`의 MVP balance model은 아래 세 값을 구현 대상으로 삼는다.

- `available_balance`: 즉시 사용 가능한 잔액 (`BIGINT`)
- `reserved_balance`: `PENDING` 신청 reserve 총액 (`BIGINT`)
- `locked_balance`: `LOCKED` 크루 보증금 총액의 persisted cache/source (`BIGINT`)

### Projection-only fields

- `settlement_pending_amount`는 종료 후 최종 정산 전 `LOCKED` 금액을 보여주는 wallet/projection 응답 필드다.
- `settlement_pending_amount`는 DB/account column이 아니다.
- `settlement_pending_balance`라는 persisted column을 만들지 않는다.

### Reconciliation invariant

- `active_locked_amount`와 `settlement_pending_amount`는 `locked_balance`의 projection-only split field다.
- 구현 검증/정합성 점검에서는 아래 관계를 만족해야 한다.

```text
locked_balance == active_locked_amount + settlement_pending_amount
```

## 3. Ledger invariants

### Append-only ledger

- `point_history`는 authoritative append-only ledger다.
- 포인트 금액 변경은 `point_account` cache만으로 완료된 것으로 보지 않는다.
- `point_account`와 `point_history`가 불일치하면 `point_history` 기준으로 조사하고 cache를 보정한다.
- `point_history` row를 수정/삭제해 결과를 맞추지 않는다.
- `point_history.idempotency_key`는 `UNIQUE`이며 권장 길이는 `VARCHAR(160)` 또는 동등한 안전 canonical size다.
- `payload_hash` 저장과 payload consistency framework는 MVP에서 명시적으로 deferred이며 필수 구현 요건이 아니다.

### Balance-after snapshots

- `point_history`는 `available_after`, `reserved_after`, `locked_after` snapshot을 저장한다.
- 이 값들은 reconciliation/debugging snapshot이다.
- append-only ledger event ordering과 idempotency가 authoritative source-of-truth이며, balance-after snapshot만으로 원장 truth를 대체하지 않는다.

### Canonical transaction types

MVP 구현에서 이 guardrail 범위의 canonical transaction type은 아래와 같다.

| Flow | Transaction type | Meaning |
| --- | --- | --- |
| Apply reserve | `CREW_DEPOSIT_RESERVE` | `PENDING` 신청 reserve 생성 |
| Reserve release | `CREW_RESERVE_RELEASE` | `PENDING` reserve를 terminal 전이와 함께 반환 |
| Settlement refund | `CREW_SETTLEMENT_REFUND` | final settlement item 환급 반영 |

### Approval is not a new ledger event

- 승인(`PENDING -> LOCKED`)은 reserve를 locked deposit으로 확정하는 bucket/state transition이다.
- 승인 시 `CREW_DEPOSIT_LOCK` 같은 새 원장 이벤트를 만들지 않는다.
- 승인 이력은 `crew_participant.status=LOCKED`, `locked_at`, account bucket delta, 기존 reserve ledger link로 설명한다.

## 4. Idempotency rules

### Canonical idempotency key formats

- Apply reserve: `crew:{crewId}:participant:{participantId}:reserve`
- Reserve release: `crew:{crewId}:participant:{participantId}:reserve-release`
- Settlement refund: `crew:{crewId}:participant:{participantId}:settlement-refund:{settlementId}`

### Reserve release once per participant

- reserve release는 `crew_participant.id` 기준으로 한 번만 허용한다.
- terminal transition과 reserve release ledger creation은 같은 transaction 안에서 처리한다.
- 구현은 `released_point_history_id`를 둔다. 이 값은 authoritative reserve-release ledger evidence이며, `reserve_released_at`만으로 release 완료를 증명하지 않는다.

### Settlement refund once per settlement item

- settlement refund는 `settlement_item` 기준으로 한 번만 반영한다.
- `settlement_item.point_history_id`는 최종 환급 ledger row와 연결되어야 한다.
- `settlement.status = SUCCEEDED` 전 모든 `settlement_item`이 point history link를 가져야 한다.

### Duplicate payload conflict guidance

- 같은 idempotency key와 같은 canonical input은 기존 결과 재사용으로 수렴한다.
- 같은 idempotency key와 다른 canonical input이 확인되면 idempotency conflict로 실패시킨다.
- MVP는 `payload_hash` 저장을 요구하지 않으므로, 별도 payload consistency framework를 구현하지 않는다.
- 중복 요청을 새로운 ledger row 생성으로 해결하지 않는다.

## 5. Settlement invariants

### LOCKED-only baseline

- final settlement baseline에는 `LOCKED` participant만 포함한다.
- `PENDING`, `REJECTED`, `CANCELLED`, `EXPIRED`는 settlement baseline에 포함하지 않는다.

### `settlement_item` snapshot authority

- `settlement_item`은 participant별 정산 snapshot과 결과 설명의 기준이다.
- `crew_participant.deposit_amount`, participant status snapshot, 성공/실패 계산 결과, refund amount를 설명 가능하게 남긴다.

### Settlement success gate

`settlement.status = SUCCEEDED`는 아래 조건을 모두 만족한 뒤에만 가능하다.

- baseline `LOCKED` participant마다 정확히 하나의 `settlement_item`이 존재한다.
- 모든 `settlement_item`의 refund amount가 계산되어 있다.
- 모든 `settlement_item`이 최종 `point_history` refund row와 연결되어 있다.
- account delta와 ledger creation이 transactionally committed 되어 있다.

### All-fail rule

- all-fail scenario는 equal principal refund다.
- all-fail을 host reward, platform bonus, 다른 참여자 실패로 인한 이익으로 모델링하지 않는다.

### Retry / replay / correction separation

- retry는 실패/중단된 동일 작업을 idempotency key로 재시도하는 운영 행위다.
- replay는 immutable snapshot/history로 결과를 재구성해 검증하는 감사/복구 행위다.
- correction/dispute workflow는 MVP 구현 범위가 아니며, `SUCCEEDED` settlement나 `point_history`를 덮어쓰는 방식으로 암묵 구현하지 않는다.

## 6. Projection invariants

- projection은 current-basis estimate다.
- final settlement success 전 projection은 확정 환급금이 아니다.
- projection wording과 API 응답은 non-final / 변동 가능 의미를 유지한다.

### Locked balance split

- `locked_balance`: persisted `point_account` cache/source
- `active_locked_amount`: 진행/모집 중 `LOCKED` 금액을 보여주는 projection-only split
- `settlement_pending_amount`: 종료 후 final settlement success 전 `LOCKED` 금액을 보여주는 projection-only split

구현은 projection 값을 출금 가능 여부, 최종 환급 확정, settlement authority 판단에 사용하지 않는다.

## 7. Forbidden implementation patterns

아래 패턴은 구현 중 재도입하지 않는다.

- `settlement_pending_balance` persisted column 추가
- `settlement_pending_amount`를 account DB column으로 저장
- approval 시 새 ledger event 생성
- `CREW_DEPOSIT_LOCK`을 apply/approval ledger로 재도입
- terminal `crew_participant` row 삭제
- 같은 crew/member 재신청을 위해 terminal row를 삭제하거나 우회
- mutable `point_history`
- host를 activation, settlement, ledger authority로 모델링
- projection을 frozen/final/guaranteed payout처럼 모델링
- private crew MVP semantics 구현
- `APPROVED_LOCK_PENDING` 재도입
- all-fail을 profit/reward/increase로 모델링
- retry/replay를 correction/dispute mutation으로 섞기

## 8. Concurrency guardrails

### Duplicate apply

- Failure mode: 같은 crew/member에 두 개의 participant row 또는 double reserve가 생김.
- Guardrail: `unique(crew_id, member_id)`, `point_account.version` optimistic lock 또는 conditional update, capacity check inside transaction.
- Expected behavior: duplicate는 기존 row/상태를 기준으로 거절하거나 idempotent response로 수렴한다.

### Approve/reject race

- Failure mode: 같은 `PENDING` row가 동시에 `LOCKED`와 terminal release로 전이됨.
- Guardrail: conditional update `WHERE status = 'PENDING'`, `crew_participant.version`, `point_account.version`.
- Expected behavior: first committer wins; loser는 최신 상태를 reload하고 invalid transition 또는 idempotent no-op으로 처리한다.

### Reserve release race

- Failure mode: reject/cancel/expire가 중복 실행되어 reserve가 두 번 반환됨.
- Guardrail: `released_point_history_id IS NULL` guard, reserve-release idempotency key unique.
- Expected behavior: release는 participant당 한 번만 성공한다.

### Settlement retry duplication

- Failure mode: retry worker가 같은 settlement item refund를 중복 반영함.
- Guardrail: `settlement.version` optimistic lock/conditional claim, `unique(settlement_id, crew_participant_id)`, unique settlement-refund idempotency key, `settlement_item.point_history_id` link.
- Expected behavior: retry는 기존 ledger/link를 재사용하거나 누락된 link만 보강한다.
