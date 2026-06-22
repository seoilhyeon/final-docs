# Dondok Backend Implementation Guardrails

이 문서는 Dondok MVP 백엔드 구현 전에 고정된 lifecycle, balance, ledger, settlement 불변식을 모은 implementation source-of-truth다. PRD를 대체하지 않으며, 구현 중 semantic drift를 막기 위한 guardrail로 사용한다.

## 0. API contract authority and resurrection guard

### Authority hierarchy

- `backend/docs/api/*`는 endpoint inventory, HTTP method/path, path/query/body 규칙, response shape, 공통 API 규칙, active/deferred boundary의 MVP active API source다.
- `docs/API-spec-dondok.md`는 `backend/docs/api/*`에서 동기화된 integrated API contract다. backend API 문서와 충돌하는 오래된 standalone API-spec 문구를 authority로 취급하지 않는다.
- `docs/PRD-dondok.md`와 `docs/Usecase-dondok.md`는 semantic guardrail이다. Backend API 문서는 active API surface를 정의하지만 settlement authority, ledger authority, lifecycle authority, projection wording, notification authority 같은 제품 의미 경계를 override하지 않는다.
- `docs/ERD-dondok.md`, `docs/Schema-migration-spec.md`, `docs/Settlement-design.md`는 API contract stabilization 이후 정렬되는 derived implementation docs다. 이 문서들을 active API source 우회나 신규 endpoint/status semantics 생성 근거로 사용하지 않는다.
- Deferred/Brownfield/Removed/Contract Drift Notes는 historical/reference only다. active MVP API, future delivery commitment, implementation permission이 아니다.

### Resurrection ban

아래 surface는 재활성화 조건을 모두 통과하기 전까지 active contract로 구현, 노출, 테스트하거나 legacy/candidate 문서에서 재사용하지 않는다.

- crew start API (`POST /api/crews/{crewId}/start`) 또는 동등한 명시적 start surface
- crew withdraw API (`POST /api/crews/{crewId}/withdraw`) 또는 active withdrawal/rejoin flow
- admin settlement list/retry API (`GET /api/admin/settlements`, `POST /api/admin/settlements/{settlementId}/retry`) 또는 admin manual settlement mutation surface
- AI habit report endpoint/surface
- `GET /api/notifications/stream` 같은 notification stream / SSE surface
- `notification_delivery_attempt`, notification preference matrix, notification template CMS/table, campaign/broadcast, transport redesign을 MVP active persistence로 승격
- `WITHDRAWN`, active withdrawal, 중도탈퇴, rejoin lifecycle semantics
- `WEEKLY_N`
- correction workflow, public replay engine, recalculation engine, correction/replay mutation surface

재활성화 조건:

1. `backend/docs/api/*`가 먼저 변경되어 해당 surface를 active로 만든다.
2. `docs/API-spec-dondok.md`를 backend API 문서 기준으로 동기화한다.
3. `docs/PRD-dondok.md`와 `docs/Usecase-dondok.md` semantic guardrail을 재검증한다.

이 조건을 통과하기 전까지 구현은 해당 surface를 active routing, service contract, API test, generated client expectation에서 제외한다.

### API convenience fields are non-authoritative

- API display/convenience/projection field는 lifecycle, settlement, ledger authority가 될 수 없다.
- projection != final settlement.
- notification/inbox/read state != canonical domain state.
- retry != correction/replay/recalculation.
- host != lifecycle/settlement/ledger authority.
- all-fail = equal principal refund.
- `settlement_item` + `point_history` linkage가 final settlement와 refund authority다.

### Canonical time authority

- MVP canonical server timezone authority는 `Asia/Seoul` (`KST`)이다.
- lifecycle cutoff, certification window, cadence/day boundary, settlement timing, projection date interpretation, replay/reconstruction date interpretation은 모두 KST 기준이다.
- Client local timezone은 표시용 local rendering context일 뿐 canonical lifecycle/settlement authority가 아니며, timezone ambiguity가 projection/final settlement drift를 만들면 안 된다.

## 0.1 Greenfield schema-to-entity readiness

- Backend persistence implementation is greenfield for MVP entities/enums/Flyway migrations. Do not assume existing entities are the authority or patch target; create entities, enums, and migrations from finalized docs.
- `docs/ERD-dondok.md` owns the active table/relationship set. `docs/Schema-migration-spec.md` owns high-risk migration guidance, V1 ordering, and minimal notification persistence shape.
- Code generation starts only after docs freeze for the current phase. Generated JPA entities must not introduce endpoint/status/table semantics that are absent from backend API docs and derived implementation docs.
- `backend/build.gradle` must include Flyway support (`org.flywaydb:flyway-core`, `org.flywaydb:flyway-mysql`) before implementation is considered migration-ready.
- V1 migration must be validated with Testcontainers MySQL. H2-only validation is not enough for MySQL FK, nullable unique, CHECK, index, and boolean behavior.

## 0.2 Minimal notification persistence allowlist

Allowed MVP notification persistence is limited to:

- `notification_device`: authenticated member FCM WEB/ANDROID device/token registration lifecycle for active device register/update/delete API.
- `notification`: minimal inbox/read/unread row for active notification list, unread count, mark-read, and read-all API.
- `notification.read_at`: nullable timestamp; `NULL` means unread.

Notification implementation guardrails:

- notification persistence is UX/refetch hint only and not canonical domain state.
- notification payload, inbox row, unread/read state, and FCM delivery state must not be modeled as mission certification, crew lifecycle, settlement, moderation, point ledger, audit history, or unresolved task authority.
- Do not add notification status workflow/status machine. `read_at` is the only read/unread persistence state.
- Do not add `notification_delivery_attempt` table, notification preference matrix, template CMS/table, campaign/broadcast, SSE/stream, realtime transport redesign, or notification transport retry topology in V1.

## 0.3 Schema implementation conventions

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
- 승인 시 해당 상태 전환은 `CREW_DEPOSIT_LOCK` 거래 유형으로 기록한다.

### Terminal state preservation

- `REJECTED`, `EXPIRED`는 terminal state다. `CANCELLED`는 reopen 가능한 pre-start exit state다.
- terminal/`CANCELLED` `crew_participant` row는 삭제하지 않는다.
- 모든 row는 audit, duplicate-apply prevention, reapply policy enforcement의 근거다.

### Reapply policy in MVP

- `REJECTED`/`EXPIRED` row가 있는 동일 `(crew_id, member_id)` 재신청은 `APPLICATION_NOT_ALLOWED`로 차단한다. terminal row를 삭제하거나 다른 status로 되돌리지 않는다.
- `CANCELLED` row가 있는 동일 `(crew_id, member_id)` 재신청은 `crew.status = RECRUITING` + 서버 시간이 `recruitment_deadline` 전 + capacity 가능 + reserve 가능일 때 허용한다. 신규 row를 만들지 않고 기존 row를 `CANCELLED -> PENDING`으로 in-place 전이(reopen)하며 새 `CREW_DEPOSIT_RESERVE point_history`를 append-only로 추가한다. `released_point_history_id`는 `null`로 reset되고, `pending_at`이 갱신된다.
- DB는 `unique(crew_id, member_id)`로 row duplication을 강제로 차단한다. reopen 경로도 같은 row를 재사용하므로 unique 제약을 유지한다.
- host auto-created `LOCKED` row는 reopen 경로에 포함되지 않는다.

## 2. Balance model

### Persisted account balances

`point_account`의 MVP balance model은 아래 세 값을 구현 대상으로 삼는다.

- `available_balance`: 즉시 사용 가능한 잔액 (`BIGINT`)
- `reserved_balance`: `PENDING` 신청 reserve 총액 (`BIGINT`)
- `locked_balance`: `LOCKED` 크루 보증금 총액의 persisted cache/source (`BIGINT`)

### Projection-only fields

- `settlement_pending_amount`는 미션 종료 후 사용자에게 환급 예정액을 보여주는 wallet/projection 응답 필드다.
- `settlement_pending_amount`는 DB/account column이 아니다.
- `settlement_pending_balance`라는 persisted column을 만들지 않는다.

### Reconciliation invariant

- `active_locked_amount`는 `locked_balance` 설명용 projection-only field이고, `settlement_pending_amount`는 정산/스냅샷 기반 환급 예정 projection-only field다.
- `locked_balance == active_locked_amount + settlement_pending_amount` 정합성 불변식을 두지 않는다. pending 환급액은 손익 반영으로 인해 locked principal과 다를 수 있다.

## 3. Ledger invariants

### Append-only ledger

- `point_history`는 authoritative append-only ledger다.
- 포인트 금액 변경은 `point_account` cache만으로 완료된 것으로 보지 않는다.
- `point_account`와 reconciliation 결과가 불일치하면 `point_history`, `crew_participant` lifecycle/deposit state, `settlement_item` linkage를 함께 기준으로 조사하고 cache를 보정한다.
- `point_history` row를 수정/삭제해 결과를 맞추지 않는다.
- `point_history.idempotency_key`는 `UNIQUE`이며 권장 길이는 `VARCHAR(160)` 또는 동등한 안전 canonical size다.
- `payload_hash` 저장과 payload consistency framework는 MVP에서 명시적으로 deferred이며 필수 구현 요건이 아니다.
- 잔액/버킷 reconciliation은 `point_history`, `crew_participant` lifecycle/deposit state, `settlement_item` linkage, `point_account` cached balances를 함께 사용한다.

### Balance-after snapshots

- `point_history`는 `available_after`, `reserved_after`, `locked_after` snapshot을 저장한다.
- 이 값들은 reconciliation/debugging snapshot이다.
- append-only ledger event ordering과 idempotency가 authoritative source-of-truth이며, balance-after snapshot만으로 원장 truth를 대체하지 않는다.

### Canonical transaction types

MVP 구현에서 이 guardrail 범위의 canonical transaction type은 아래와 같다.

| Flow | Transaction type | Meaning |
| --- | --- | --- |
| Point charge | `POINT_CHARGE` | 결제 승인 후 포인트 충전 반영 |
| Apply reserve / host lock | `CREW_DEPOSIT_RESERVE` | 일반 참여자 `PENDING` 신청 reserve 생성 또는 host auto-created `LOCKED` 보증금 lock event |
| Approval / lock confirm | `CREW_DEPOSIT_LOCK` | `PENDING` 승인 시 reserve를 locked로 확정 |
| Reserve release | `CREW_RESERVE_RELEASE` | `PENDING` reserve를 terminal 전이와 함께 반환 |
| Settlement refund | `CREW_SETTLEMENT_REFUND` | final settlement item 환급 반영 |

### Approval is an explicit ledger event

- 승인(`PENDING -> LOCKED`)은 reserve-to-lock 확정 이벤트다.
- `CREW_DEPOSIT_LOCK` `point_history` row를 append/reuse하며 승인 시 보증금 이동을 불변 로그로 남긴다.
- 승인 이력은 `crew_participant.status=LOCKED`, `locked_at`, account bucket delta, 승인 ledger 연계로 설명한다.

## 4. Idempotency rules

### Canonical idempotency key formats

- Apply reserve: `crew:{crewId}:participant:{participantId}:reserve:{cycle}`
- Reserve lock / approval: `crew:{crewId}:participant:{participantId}:reserve-lock:{cycle}`
- Reserve release: `crew:{crewId}:participant:{participantId}:reserve-release:{cycle}`
- Settlement refund: `crew:{crewId}:participant:{participantId}:settlement-refund:final`

### Reserve release once per current reserve cycle

- reserve release는 같은 `crew_participant.id`의 현재 reserve cycle 기준으로 한 번만 허용한다.
- `{cycle}`은 최초 `1`이며, 새 reserve cycle은 해당 participant의 누적 `CREW_RESERVE_RELEASE` 원장 수 + 1로 계산한다. reserve 원장 수를 기준으로 cycle을 증가시키면 duplicate reserve retry가 cycle을 밀 수 있으므로 사용하지 않는다.
- terminal transition과 reserve release ledger creation/reuse는 같은 transaction 안에서 처리한다.
- 구현은 `released_point_history_id`를 둔다. 이 값은 현재 cycle의 authoritative reserve-release ledger evidence이며, `reserve_released_at`만으로 release 완료를 증명하지 않는다.
- `released_point_history_id`는 nullable unique로 강제한다. 여러 row가 `NULL`일 수는 있지만, 하나의 reserve-release `point_history` row를 여러 `crew_participant`가 공유할 수 없다.
- `CANCELLED -> PENDING` reopen 시 같은 transaction에서 `released_point_history_id`를 `NULL`로 reset하고 새 `CREW_DEPOSIT_RESERVE`를 append해 다음 cycle을 시작한다.

### Settlement refund once per settlement item

- settlement refund는 crew participant의 최종 정산 환급 기준으로 한 번만 반영한다.
- `settlement_item.point_history_id`는 최종 환급 ledger row와 연결되어야 한다.
- `settlement.status = SUCCEEDED` 전 모든 `settlement_item`이 point history link를 가져야 한다.
- MVP 정산 환급 idempotency identity는 `crew:{crewId}:participant:{participantId}:settlement-refund:final`이다. Runtime-generated `settlement.id`는 idempotency key 구성값으로 사용하지 않는다.
- `settlement_item.id`는 지급 근거 스냅샷 추적용 linkage metadata이며 `point_history.reference_id`와 `settlement_item.point_history_id`로 연결한다.
- 동일 `settlement-refund:final` key에 대해 `settlement_item.id`, 환급 금액, 정산 algorithm version, 인정 성공 수 등 canonical payout input이 다르면 idempotency conflict로 실패해야 한다.
- 재정산/보정 지급은 `final` key를 재사용하지 않고 별도 transaction type/key로 분리한다.

### One final settlement per crew

- MVP에는 crew당 authoritative final settlement row가 정확히 하나만 존재할 수 있다.
- DB는 `unique(crew_id)`로 중복 settlement header 생성을 막는다.
- retry/replay는 기존 `settlement` row와 기존 `settlement_item` snapshot/linkage를 대상으로 수행하며, 새 settlement type이나 새 settlement row를 만들어 의미를 갈라타지 않는다.

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
- `settlement_pending_amount`: 정산 row 우선, 없으면 `CLOSED` 크루의 최신 `FINALIZED`/`SUCCEEDED` 일일 정산 스냅샷으로 계산하는 환급 예정 projection-only field

구현은 projection 값을 출금 가능 여부, 최종 환급 확정, settlement authority 판단에 사용하지 않는다.

## 7. Forbidden implementation patterns

아래 패턴은 구현 중 재도입하지 않는다.

- `backend/docs/api/*` active contract에 없는 endpoint/method/path/status/field를 구현 또는 API test 대상으로 승격
- Deferred/Brownfield/Removed/Contract Drift Notes surface를 active MVP API, roadmap commitment, implementation permission처럼 사용
- crew start, crew withdraw, admin settlement list/retry, AI habit report, notification stream/SSE, `WITHDRAWN`, active withdrawal/rejoin, `WEEKLY_N`, correction/replay engine, admin manual settlement surface 재도입
- notification payload, inbox row, unread/read state를 mission certification, crew lifecycle, settlement, point ledger canonical state로 모델링
- notification task workflow/status machine을 생성하거나 `read_at` 외 별도 notification status enum을 도입
- `notification_delivery_attempt` MVP table, notification preference/template persistence, SSE/stream, transport redesign 재도입
- API display/convenience/projection field를 final settlement, payout guarantee, lifecycle authority, ledger truth로 모델링
- host 또는 admin/manual surface를 lifecycle, settlement, ledger authority로 모델링
- `settlement_pending_balance` persisted column 추가
- `settlement_pending_amount`를 account DB column으로 저장
- approval 시 새 ledger event 생성
- `CREW_DEPOSIT_LOCK`을 apply/approval ledger로 재도입
- terminal `crew_participant` row 삭제
- `REJECTED`/`EXPIRED` row를 삭제하거나 다른 status로 되돌려 재신청을 허용 (`CANCELLED` reopen은 신규 row 생성 없이 기존 row를 in-place 전이하는 경로이며 unique 우회가 아니다)
- mutable `point_history`
- projection을 frozen/final/guaranteed payout처럼 모델링
- private crew MVP semantics 구현
- `APPROVED_LOCK_PENDING` 재도입
- `approved_at` 컬럼 추가
- `payload_hash` / persisted payload consistency framework 구현
- AI habit report entity/API 구현
- replay/correction engine 구현
- active `WITHDRAWN` 또는 active `WEEKLY_N` 구현 (no active WITHDRAWN / no active WEEKLY_N)
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
- Expected behavior: release는 현재 reserve cycle당 한 번만 성공한다. 이미 `released_point_history_id`가 연결된 동일 cycle 중복 요청은 기존 release 원장 재사용 또는 invalid transition으로 수렴한다.

### Settlement retry duplication

- Failure mode: retry worker가 같은 settlement item refund를 중복 반영함.
- Guardrail: `settlement.version` optimistic lock/conditional claim, `unique(settlement_id, crew_participant_id)`, unique settlement-refund idempotency key, `settlement_item.point_history_id` link.
- Expected behavior: retry는 기존 ledger/link를 재사용하거나 누락된 link만 보강한다.
