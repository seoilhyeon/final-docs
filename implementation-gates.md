# Implementation Gates: 갓세이빙 MVP Invariants

이 문서는 구현 단계에서 반복해서 확인할 blocker-level invariant와 PR 리뷰 gate를 모은다. 새로운 설계 문서가 아니라, 아래 source of truth를 구현자가 놓치지 않도록 압축한 governance 문서다.

Source-of-truth overlay:

이 문서는 전체 제품/기술 source-of-truth hierarchy를 새로 정의하지 않는다. 문서 간 충돌 시 기존 기준인 `PRD -> API-spec -> ERD -> Settlement-design -> ADR -> Tech-stack summary -> Ticket breakdown` 순서를 따른다.

이 문서가 압축해 참조하는 구현 gate 근거 문서는 다음이다.

- `docs/API-spec-god-saving.md` — API 계약, 요청/응답, 상태 노출 규칙
- `docs/ERD-god-saving.md` — 테이블, FK, unique 제약, source-of-truth 관계
- `docs/Settlement-design.md` — 정산 계산, batch/retry/recovery 동작
- `docs/adr/ADR-mvp-tech-architecture.md` — 기술 선택, cut line, 운영 철학
- `docs/runbooks/settlement-recovery.md` — 장애 시 운영 절차
- `docs/MVP-ticket-breakdown.md` — 구현 티켓과 phase gate

구현자는 원 source 문서를 우선하고, 이 문서는 PR 리뷰 체크리스트와 drift 방지 gate로 사용한다.

## 1. Gate severity

| 구분 | 의미 | 처리 |
| --- | --- | --- |
| Blocker invariant | 깨지면 source-of-truth drift 또는 금전성/복구 불가능 상태를 만든다 | 구현 중단 후 설계 문서와 구현을 정합화한다 |
| Recovery gate | 장애 후 운영자가 API/batch/runbook으로 복구할 수 있어야 한다 | 복구 경로가 없으면 scope reduction 또는 redesign 대상이다 |
| Implementation refinement | 수치, 주기, threshold, 세부 필드처럼 구현 중 조정 가능한 사항 | 테스트/운영 경험으로 조정 가능하되 blocker invariant를 깨지 않는다 |

Refinement 예시: CloudWatch alarm threshold, orphan cleanup 주기, retry backoff, structured log 추가 필드, batch step 세분화, presigned URL TTL, content-type whitelist 상세값.

## 2. Payment implementation gates

Blocker invariant:

- `payment_id`는 Toss `paymentKey`다.
- `orderId`는 confirm validation과 log correlation 전용이며 `point_history.idempotency_key`에 들어가면 안 된다.
- `POINT_CHARGE` idempotency key는 `charge:{paymentKey}`다.
- 동일 key + 동일 semantic payload는 기존 `point_history`를 재사용한다.
- 동일 key + 다른 payload는 idempotency conflict로 실패한다.
- provider success 후 client timeout이 발생해도 같은 `paymentKey` 재시도는 기존 원장 재사용으로 수렴해야 한다.
- `point_history` 없는 `point_account.balance` 증가는 금지한다.
- Redis idempotency cache는 빠른 판별용 보조 계층이며 correctness source가 아니다.

Forbidden patterns:

- `orderId` 기반 idempotency.
- provider timeout만으로 성공 처리.
- Redis cache hit/miss만 믿고 DB unique 또는 payload 비교 생략.
- `point_account.balance`를 금액 source of truth로 취급.
- point_history insert 없이 balance만 update.

Minimum PR evidence:

- duplicate confirm integration test.
- same key/same payload reuse test.
- same key/different payload conflict test.
- provider success/client timeout recovery test 또는 이에 준하는 service-level fixture.
- DB unique constraint 확인: `point_history.idempotency_key`.

## 3. Storage / EXIF implementation gates

Blocker invariant:

- presigned URL은 upload delegation only다. validation delegation이 아니다.
- S3 object key는 서버가 생성한다.
- bucket/object는 private이다.
- client arbitrary key/path 지정은 금지한다.
- mission-log 생성 시 서버가 S3 object를 직접 조회해 existence, size, content-type, ownership, EXIF를 검증한다.
- EXIF는 서버가 object에서 추출한 값만 authoritative하다.
- client가 보낸 `exif_taken_at`을 authoritative input으로 저장하면 안 된다.
- `MissionLog.exif_taken_at`은 server-side validation result다.
- orphan object는 lifecycle rule 또는 최소 cleanup job으로 관리 가능해야 한다.

Forbidden patterns:

- presigned 발급 시점 검증만으로 mission-log 생성 검증을 생략.
- public bucket 전제.
- client-provided EXIF trust.
- S3 object validation 없이 `MissionLog` 생성.
- `MissionLog.is_success = true`를 최종 정산 포함 확정으로 해석.

Minimum PR evidence:

- arbitrary key rejection test.
- ownership mismatch rejection test.
- object missing test.
- EXIF missing / invalid test.
- content-type spoofing 또는 invalid object fixture test.
- orphan cleanup policy 문서/설정 확인.

## 4. Settlement / Batch / Refund implementation gates

Blocker invariant:

- settlement 실행권은 `Settlement.status in (PENDING, RETRY_WAIT)` 조건부 claim으로 얻는다.
- Redisson lock은 duplicate suppression 보조 계층이며 correctness source가 아니다.
- `settlement_item`은 participant-level calculation snapshot이고 성공 정산 이후 운영/분쟁/조회 기준이다.
- `point_history`는 금액 source of truth다.
- participant별 환급 idempotency key는 입력 기반 deterministic key다.
- retry는 재정산이 아니라 기존 settlement/snapshot 기준 미완료 participant recovery다.
- partial success는 `RETRY_WAIT` 또는 `FAILED`로 남아야 한다.
- `Settlement.status = SUCCEEDED`는 모든 `settlement_item.point_history_id`가 non-null이고 대응 `point_history`가 존재할 때만 가능하다.
- parent `SUCCEEDED` + null `settlement_item.point_history_id`는 invalid state다.
- `settlement_item.point_history_id`가 non-null인데 대응 `point_history`가 없으면 `INVALID_INCONSISTENT`로 취급하고 succeeded로 보지 않는다.
- `point_history` 직접 수정/삭제는 정상 복구 경로가 아니다.

Forbidden patterns:

- parent `Settlement.status`만 보고 participant payout 완료 판단.
- partial payout 후 parent `SUCCEEDED` 처리.
- retry마다 새 `Settlement` 또는 새 계산 snapshot 생성.
- 이미 지급된 participant에 새 refund `point_history` 생성.
- Batch metadata나 Redis lock state를 source of truth로 승격.
- 후속 Email/AI/SSE 실패로 settlement rollback.

Minimum PR evidence:

- concurrent claim test: row count `1` worker만 실행.
- duplicate settlement creation 방지 test: `unique(room_id, settlement_type)`.
- participant duplicate payout 방지 test: deterministic idempotency + unique.
- point_history exists + FK missing recovery test.
- null FK 상태에서 SUCCEEDED 전환 거부 test.
- FK non-null + missing point_history inconsistent fixture test.
- RUNNING timeout recovery test.
- Redis unavailable normal worker fail-closed/controlled-retry test.
- DB-claim-only fallback recovery test.

## 5. Recovery / Ops implementation gates

Blocker invariant:

- Admin UI는 MVP 제외다.
- 정상 복구는 관리자 API, batch trigger, CloudWatch/log, runbook으로 수행한다.
- Redis unavailable 중 일반 worker는 unsafe monetary execution을 강행하지 않는다.
- Redis unavailable 복구가 필요하면 DB-claim-only fallback path를 사용한다.
- stale `RUNNING` timeout 후보는 먼저 timeout recovery로 `RETRY_WAIT` 전환한 뒤 DB-claim-only fallback 대상이 된다.
- RUNNING timeout recovery가 있어야 한다.
- direct DB mutation은 정상 복구 경로가 아니다.
- break-glass DB repair는 incident record, audit, post-repair invariant validation이 있을 때만 예외적으로 다룬다.

Structured log minimum set for settlement/recovery:

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
- recovery action and result

CloudWatch minimum alarms:

- settlement batch failure
- `RUNNING` timeout
- `RETRY_WAIT` / `FAILED` increase
- Redis unavailable
- DB connection failure
- payment confirm failure
- idempotency conflict spike
- reconciliation mismatch
- disk usage / instance health

Forbidden patterns:

- runbook이 `point_history`, `settlement_item` 금액, `Settlement.status = SUCCEEDED` 직접 update를 정상 절차로 안내.
- Admin UI가 없다는 이유로 recovery API 생략.
- operator가 Redis lock 상태만 보고 payout 완료 판단.
- 후속 이벤트 실패를 금전성 transaction 실패로 전파.

Minimum PR evidence:

- `GET /api/admin/settlements`로 실패/대기/timeout 후보 식별 가능.
- `POST /api/admin/settlements/{settlementId}/retry`는 기존 settlement만 claim.
- SUCCEEDED retry 거부.
- retry/recovery command idempotent/resumable 검증.
- runbook 절차와 structured log 필드가 같은 식별자를 사용.

## 6. Identity / Notification implementation gates

Blocker invariant:

- `member.uuid`는 회원 생성 시 발급되는 immutable external canonical identifier다.
- JWT access token subject(`sub`)는 `member.uuid`다.
- Spring Security principal은 내부 처리를 위해 `memberId(Long)`와 `memberUuid(UUID)`를 함께 보유할 수 있지만, external boundary에는 UUID를 사용한다.
- `member.id` / `member_id`는 DB 내부 FK, join, persistence identity로 유지한다.
- SSE emitter registry key와 notification/event routing key는 `member.uuid`다.
- `email`은 로그인 식별자/연락처이며 routing key, stream identifier, notification recipient key, JWT subject가 아니다.
- SSE는 state source가 아니라 invalidate/refetch signal이다. FE는 이벤트 수신 후 필요한 REST API를 refetch/invalidate한다.
- notification persistence/read-state/unread count source of truth는 MVP 필수 범위가 아니다.

Forbidden patterns:

- JWT `sub = email` 또는 `sub = member.id`.
- SSE emitter registry key가 email.
- notification recipient routing key가 email.
- 외부 API 응답에서 사용자 canonical identifier로 Long `member_id`를 노출.
- SSE payload에 email, Long `member.id`, 불필요한 PII를 포함.
- SSE/badge/count를 unread/read source of truth로 취급.
- notification inbox/read-state/read-all/mark-as-read 구현을 UUID migration의 MVP 필수 범위로 확장.

Minimum PR evidence:

- ERD에 `member.uuid unique not null`이 존재한다.
- JWT 생성/검증 테스트가 `sub = member.uuid`를 확인한다.
- 인증 principal에서 UUID로 외부 boundary를 구성하고 내부 DB 조회에는 Long PK를 사용한다.
- SSE subscribe/publish 테스트가 `member.uuid` routing을 검증한다.
- email 변경 또는 email 값 불일치가 SSE/notification routing identity를 깨지 않음을 service-level fixture로 확인한다.

## 7. PR review checklist

### Payment PR Gate

- [ ] `payment_id`가 Toss `paymentKey`로만 해석된다.
- [ ] `orderId`가 idempotency key에 사용되지 않는다.
- [ ] `charge:{paymentKey}` unique idempotency가 적용된다.
- [ ] same key/same payload는 reuse된다.
- [ ] same key/different payload는 conflict다.
- [ ] provider success/client timeout recovery가 중복 충전을 만들지 않는다.
- [ ] `point_history` 없이 balance update가 발생하지 않는다.
- [ ] Redis cache는 보조 계층으로만 사용된다.

### Storage/EXIF PR Gate

- [ ] presigned URL을 validation delegation으로 취급하지 않는다.
- [ ] object key는 서버 생성이며 client arbitrary key/path가 차단된다.
- [ ] bucket/object private 정책이 유지된다.
- [ ] mission-log 생성 시 S3 object를 서버가 직접 검증한다.
- [ ] EXIF는 server-side extracted value만 authoritative하다.
- [ ] client `exif_taken_at`이 저장 source가 아니다.
- [ ] orphan cleanup 최소 정책이 있다.

### Settlement PR Gate

- [ ] `Settlement.status` 조건부 claim이 실행권을 결정한다.
- [ ] Redisson lock은 correctness source가 아니다.
- [ ] `settlement_item` snapshot이 생성되고 이후 point_history가 연결된다.
- [ ] participant-level deterministic idempotency가 적용된다.
- [ ] retry가 재정산이 아니라 미완료 participant recovery로 구현된다.
- [ ] partial success는 `RETRY_WAIT`/`FAILED`로 남는다.
- [ ] parent `SUCCEEDED` + null FK가 불가능하다.
- [ ] FK non-null + missing point_history는 succeeded로 취급하지 않는다.
- [ ] 이미 지급된 participant에 중복 refund가 발생하지 않는다.

### Recovery/Ops PR Gate

- [ ] Admin UI 없이 관리자 API/runbook으로 복구할 수 있다.
- [ ] Redis unavailable 일반 worker는 fail-closed 또는 controlled retry다.
- [ ] DB-claim-only fallback path가 조건부 claim을 사용한다.
- [ ] stale `RUNNING` timeout 후보는 먼저 `RETRY_WAIT`로 normalize된다.
- [ ] RUNNING timeout recovery가 있다.
- [ ] structured log minimum set이 남는다.
- [ ] CloudWatch minimum alarms가 정의되어 있다.
- [ ] direct DB mutation이 정상 복구 경로로 문서화되지 않는다.
- [ ] Email/AI/SSE 실패가 settlement rollback으로 전파되지 않는다.

### Identity / Notification PR Gate

- [ ] `member.uuid`가 생성되고 unique/not-null/immutable external identifier로 취급된다.
- [ ] JWT `sub`가 `member.uuid`이며 email/Long PK가 아니다.
- [ ] principal은 내부 `memberId`와 외부 `memberUuid` 경계를 분리한다.
- [ ] SSE emitter registry와 notification routing이 UUID 기준이다.
- [ ] email routing 구현 또는 문서 표현이 남아 있지 않다.
- [ ] SSE payload가 최소 invalidate/refetch signal이며 PII/state snapshot을 포함하지 않는다.
- [ ] FE는 SSE 수신 후 REST API를 refetch/invalidate한다.
- [ ] notification persistence/read-state/unread sync가 MVP 필수 구현으로 scope creep 되지 않는다.
