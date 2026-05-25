# 정산 설계: Dondok

기준 문서:

1. 최신 기획안 및 accepted semantic freeze 결과 — L1 intent authority
2. [PRD-dondok.md](./PRD-dondok.md) — canonical synthesis layer
3. `docs/Dondok_요구사항명세서_v0.9.xlsx` — requirement detail reference

이 문서는 위 SoT의 하위 운영/runtime semantics 문서이며, 제품 의미를 새로 정의하거나 PRD synthesis를 override하지 않는다. `API-spec`, 요구사항 명세서, 외부 WBS/GitHub Issues는 downstream 구현/계약 참고 자료로만 교차 확인한다.

## 1. 목적

이 문서는 정산 규칙 엔진, 정산 배치, 골든 데이터 테스트를 구현하기 전에 정산 도메인의 runtime semantics와 구현 전 guardrail을 정리하기 위한 문서다.

이번 개정의 목표는 정산 로직 자체를 갈아엎는 것이 아니라, 아래 운영 요구를 만족하도록 MVP 설계를 보강하는 것이다.

1. 같은 입력이면 언제 다시 계산해도 같은 결과가 나와야 한다.
2. 포인트 환급은 participant 단위 deterministic idempotency key로 지급하며, 배치 재시도나 중복 실행이 있어도 중복 반영되면 안 된다.
3. 운영자는 장애 후 재시도, 실패 분석, 분쟁 대응을 문서와 데이터만으로 수행할 수 있어야 한다.
4. 사용자는 "왜 이 금액을 받았는지"를 나중에도 설명받을 수 있어야 한다.

## 2. 범위와 비범위

### 범위

- 미션 종료 후 authoritative final settlement batch
- 인원 미달 취소 환급
- `SUCCEEDED` 전 MissionLog 기반 최종 계산
- 정산 스냅샷 저장과 포인트 원장 반영
- 배치 재시도, 중복 실행 차단, 장애 복구를 위한 상태 모델
- 관리자 조회와 재시도를 위한 최소 운영 정보 저장

### 비범위

- 현금 인출
- 외부 결제/송금 연동
- 이벤트 소싱
- Kafka 기반 비동기 파이프라인
- 복잡한 분산 트랜잭션
- 성공 후 재정산용 별도 회계 정정 프로세스
- final settlement 이후 payout rewrite, hidden mutation, support/admin override workflow
- MVP 범위를 넘어서는 분산 실행 조정 전략, batch infrastructure topology의 신규 결정
- `point_account` physical balance shape(`available`, `locked`, `pending`, `total` 등) 재설계
- Android-first FCM MVP를 넘어서는 notification transport architecture 결정(SSE/Web realtime reliability, campaign/broadcast 등)
- settlement amount unit 재검토 결정

## 3. 고정할 비즈니스 규칙

### 3.1 시간 기준

- MVP의 정산 기준 시간대는 `Asia/Seoul`로 고정한다.
- 사용자에게 보이는 미션 종료 시점은 `종료일 23:59:59 KST`다.
- `recruitment_deadline`은 신규 참여 마감 시각이며, activation eligibility에 들어갈 수 있는 participant 후보를 freeze하는 기준이다. 단, activation/settlement 실행 기준 시각 자체는 아니다.
- `start_at`은 예정 시작 시각이자 MVP의 자동 activation anchor다.
- `activated_at`은 실제 `RECRUITING -> ACTIVE` 전이 시각이며, MVP에서는 시스템 lifecycle 규칙에 의해 `activated_at = start_at`으로 고정한다.
- Host는 activation authority가 아니다. Host는 방 설정·모집·moderation actor일 뿐, `ACTIVE` 전이를 직접 만들 수 없다.
- `end_at`은 계획된 미션 종료 cutoff로 유지하며, activation 지연으로 자동 연장하지 않는다. MVP에서는 deterministic settlement와 replay consistency를 위해 `activated_at = start_at`, `end_at` 고정을 함께 유지한다.
- Final settlement batch의 제품 기준 시점은 `마지막 인증 주기의 일일 정산 완료 시점 + 24시간`이다.
- 배치 스케줄러의 실제 실행 시각, 지연 버퍼, 운영 창은 구현/runtime metadata이며 정산 금액이나 product authority를 바꾸지 않는다.
- 목표 SLA는 위 product anchor 이후 정해진 운영 창 안에 final settlement batch가 완료되도록 관리한다.
- 일일 인증/정산 cadence는 `mission_rule.daily_settlement_type`이 결정한다. MVP active anchor는 아래 세 가지이며, host 또는 admin이 이 cadence를 사후 변경하지 않는다.
  - `A`: 일일 인증 마감 `09:00 KST`, 일일 정산 batch `12:00 KST`
  - `B`: 일일 인증 마감 `21:00 KST`, 일일 정산 batch `00:00 KST` (익일)
  - `C`: 일일 인증 마감 `23:59 KST`, 일일 정산 batch `익일 12:00 KST`
- A/B/C cutoff은 scheduled semantic anchor이고, 배치 실행 시각 자체는 운영 metadata다. cutoff 이후 도착한 인증은 해당 일자의 settlement 입력에 포함하지 않는다.

### 3.2 정산 대상 참여자

- 정산 대상 participant는 activation 시점 frozen `LOCKED` baseline에 포함된 참여자다. 종료 시점의 locked deposit 조회값은 검증/스냅샷 입력일 뿐, ACTIVE 이후 withdrawal/rejoin으로 baseline을 다시 여는 기준이 아니다.
- 한 `member`는 하나의 `crew`에 대해 하나의 `crew_participant`만 가진다.
- 이 불변식은 `unique(crew_id, member_id)`로 강제하고, `crew_participant`는 정산과 감사 추적을 위해 물리 삭제하지 않는다.
- MVP active `crew_participant.status`: `PENDING`, `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`. 승인 후 lock 대기 상태(`APPROVED_LOCK_PENDING`)는 두지 않으며 방장 승인은 기존 reserve를 `PENDING -> LOCKED`로 확정한다.
- `PENDING`은 신청 제출 + 예치금 reserve 상태다. capacity reservation에는 포함하지만 activation eligibility, minimum participant baseline, frozen participant baseline, settlement eligibility에는 포함하지 않는다. `PENDING` 생성 시 `point_account.balance`(available)가 감소하고 reserve projection이 증가한다.
- 사용자가 승인 전 신청을 취소하면 `PENDING -> CANCELLED`. 기존 reserve는 취소 환급 원장으로 반환한다.
- 방장이 거절하거나 시작 전까지 처리되지 않아 자동 만료된 신청은 `REJECTED` / `EXPIRED`다. 기존 reserve는 반환되며 settlement baseline에 포함하지 않는다.
- `LOCKED`는 방장 승인으로 reserve가 참여 확정된 상태다. MVP에서 activation eligibility, minimum participant baseline, frozen participant baseline, settlement eligibility의 participant anchor는 `LOCKED`만 사용한다.
- `LOCKED` 이후에는 MVP에서 participant-side 변경/취소를 허용하지 않는다. 승인 + 예치 Lock 완료 후 상태 변경은 frozen baseline integrity와 deterministic settlement를 흔들 수 있으므로 별도 후속 설계 없이는 열지 않는다.
- 위 lifecycle 문장은 semantic boundary이며, 구체적인 DB column, API status, enum name, account balance column, lock implementation strategy는 `ERD` / `API-spec`이 소유한다.
- 신규 참여/상태 전이는 `MissionRoom.status = RECRUITING`이고 서버 시간이 `recruitment_deadline` 전일 때만 허용한다.
- `ACTIVE` 이후 신규 참여와 baseline 변경은 허용하지 않는다.
- ACTIVE 이후 탈퇴/재참여 및 중도 탈퇴 정산은 MVP active semantics가 아니라 brownfield/deferred 영역으로 남긴다. 기존 문서/구현 흔적이 있더라도 `LOCKED` frozen baseline을 바꾸는 권한으로 해석하지 않는다.
- frozen participant baseline은 `start_at` 자동 activation 시점에 `LOCKED`인 participant 집합이다. 이 baseline은 final settlement input으로 사용되며 post-freeze에 host/admin이 소급 변경하지 않는다.

### 3.3 `min_participants` 정책

- `min_participants`는 `MissionRoom`별 설정값으로 관리한다.
- 방 생성 시 host가 설정할 수 있고, 기본값은 `2`명이다.
- 제약 조건은 PRD synthesis 기준 `2 <= min_participants <= max_participants <= 15`이다.
- MVP에서 `min_participants` 충족은 host command의 precondition이 아니라 시스템 자동 activation의 eligibility condition이다.
- `start_at` 자동 activation 시점에 `LOCKED` participant 수를 다시 검증하며, 미달이면 `ACTIVE`로 전이하지 않고 시작 전 취소 정산 대상으로 남긴다.

### 3.4 시작 만료 / 인원 미달 취소

- `recruitment_deadline` 이후 신규 참여는 차단한다.
- `start_at`에 시스템은 frozen eligibility condition을 평가한다. `LOCKED` participant 수가 `min_participants` 이상이고 system-recognized terminal cancellation condition이 없으면 자동으로 `ACTIVE` 전이한다.
- `start_at` 이후에도 eligibility condition을 만족하지 못해 `RECRUITING`인 방은 batch가 `CANCELLED` 처리한다.
- 시작 만료 또는 인원 미달 취소는 일반 정산과 별개가 아니라 `취소형 정산`으로 기록한다.
- 취소형 정산에서는 각 참여자에게 `잠긴 보증금 전액`을 환급한다.

### 3.5 정산 계산 입력과 성공 후 운영 원천

- 실시간 대시보드는 캐시나 역정규화 테이블을 사용해도 된다.
- 그러나 `Settlement.status = SUCCEEDED` 전 최종 정산 금액 계산은 반드시 `MissionLog`, frozen participant baseline, resolved certification state를 다시 읽어서 수행한다.
- Final settlement batch가 authoritative settlement snapshot을 만들며, dashboard/expected refund 값은 그 전까지 projection일 뿐 확정 환급금이 아니다. Projection은 현재 기준 예상/설명용이며 payout authority가 아니다.
- Settlement input freeze 이후에는 frozen certification outcome과 authoritative daily/final result를 host/admin이 소급 변경하지 않는다.
- Replay는 historical semantic truth reconstruction이다. Replay는 당시 algorithm semantics, cadence/timezone/cutoff interpretation, lifecycle cutoff semantics, effective moderation state, reason-code mapping을 설명 가능하게 복원하는 audit authority이며, current-engine reinterpretation이나 payout rewrite 권한이 아니다.
- Host moderation은 certification input/state를 resolve하는 권한이며, settlement engine, refund amount, point ledger, final settlement snapshot을 직접 조작하는 권한이 아니다.
- 일반 인증 일자는 인증 마감 이후 host에게 최대 `72시간` host moderation correction window가 주어진다. 이 window 안의 결정 변경은 pre-freeze certification input correction이며 settlement input freeze 이전에만 effective하다.
- 미션 종료일 포함 마지막 `3일`의 인증 결과는 72h grace 없이 즉시 terminal/freeze 처리된다. 이 구간의 host moderation은 해당 일자 정산 batch 시각 이전에 완료되어야 하며, 이후 결정 변경은 frozen settlement snapshot에 반영되지 않는다.
- 72h grace는 settlement input freeze 이전 host correction window 설명용이지, settlement input freeze 이후 결과를 변경하는 권한이 아니다. Projection은 이 window를 현재 기준 예상으로 설명할 수 있지만 final settlement input freeze 이후 host/admin이 resolved certification state를 소급 변경하지 않는다.
- Moderation persistence는 authoritative moderation transition ledger와 non-authoritative operational context를 분리한다. Settlement truth에 필요한 것은 effective state, state transition, reason-code, actor, timestamp, append-only chain reference이고, human memo/support note/UX wording/운영 코멘트는 정산 truth가 아니다.
- Pre-freeze moderation resolution은 certification input을 정리하는 행위이고, post-freeze settlement recovery는 누락 row, FK linkage, payout execution failure만 append-only로 복구하는 행위다. 둘은 같은 권한이 아니다.
- `Settlement.status = SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다. 이후 `MissionLog` 기반 replay는 감사/디버깅 검증용이지 지급 결과를 대체하거나 변경하는 기준이 아니다.
- retry는 실패/partial execution을 기존 snapshot과 idempotency key로 이어 붙이는 recovery 동작이고, correction은 별도 support/operations 조정 후보일 뿐 hidden settlement mutation이 아니다.
- `Mission_Room_Stat` 같은 캐시성 테이블은 정산 금액 계산의 근거로 사용하지 않는다.

### 3.6 금액 단위

- 포인트와 환급금은 모두 `원 단위 정수`로 저장하고 노출한다.
- DB 금액 컬럼은 `BIGINT` 또는 `DECIMAL(18,0)` 등 정수 표현을 사용한다.
- Java 계산은 `BigDecimal`과 `MathContext.DECIMAL128`을 사용한다.
- 최종 지급액은 `RoundingMode.FLOOR`로 절사한다.
- settlement amount unit 재검토는 별도 결정 후보로 남긴다. 이 문서는 현행 정수/절사/remainder baseline을 유지하며, 단위 변경이나 rounding 변경을 이번 propagation에서 freeze하지 않는다.

### 3.7 재현 가능성 원칙

- 정산 결과는 `MissionLog`, frozen participant baseline, resolved certification state, 고정된 정산 규칙만으로 재현 가능해야 한다.
- 재현 가능성은 `정산 배치 실행 시각`, `settlement.id`, 런타임 랜덤에 의존하면 안 된다.
- 외부 결제 상태나 후속 이벤트 성공 여부는 정산 계산 결과의 입력값이 아니다.
- Projection boundary는 사용자 안내용 조회/캐시까지이고, settlement snapshot boundary는 final settlement batch가 저장한 `settlement_item`과 연결된 `point_history` 원장까지다. Projection을 재계산해도 succeeded settlement snapshot이나 ledger를 변경하지 않는다. Projection 문구는 `현재 기준 예상`, `진행 상황 업데이트`, `최종 정산 전 변동 가능` 수준으로 제한하고 수익/순위/payout 확정처럼 표현하지 않는다.
- Runtime replay의 최소 authoritative context는 `algorithm_version`, frozen participant baseline, deposit snapshot, recognized success counts, all-fail/remainder policy, cadence interpretation, timezone/cutoff semantics, lifecycle cutoff semantics, effective moderation state, append-only moderation chain reference, reason-code mapping version을 포함한다. 이는 과거 semantic truth를 설명하기 위한 contract이며 과거 지급액을 다시 쓰기 위한 contract가 아니다.
- Versioned semantic replay에서 v2 runtime은 v1 settlement를 수정하지 않고 v1 semantics를 해석 가능해야 한다. Historical replay는 migration-forward reinterpretation이나 current semantics overwrite가 아니다.

## 4. 정산 도메인 책임

| 도메인 객체       | 책임                                                  | 비고                                                          |
| ----------------- | ----------------------------------------------------- | ------------------------------------------------------------- |
| `MissionRoom`     | 미션 모집, 시작, 종료, 취소의 생명주기 관리           | 정산 상태의 원천이 아니다                                     |
| `Settlement`      | 정산의 생성, claim, 실행, 성공/실패, 재시도 상태 관리 | 정산 상태의 소스 오브 트루스                                  |
| `SettlementItem`  | 참여자별 정산 입력 스냅샷, 계산 결과, 계산 근거 저장  | 분쟁 대응과 사후 설명 책임                                    |
| `PointHistory`    | 모든 포인트 증감 원장 기록                            | 사용 가능 잔액 변화와 보증금 잠금/환급의 금액 source of truth |
| `SettlementBatch` | `PENDING` 정산을 조회, claim, 실행, 재시도            | 계산 규칙 자체를 소유하지 않는다                              |

정리:

- `MissionRoom.status`는 방의 상태를 말한다.
- `Settlement.status`는 정산 처리 상태를 말한다.
- `MissionRoom.settlement_status`가 필요하다면 조회 최적화용 비정규화 필드로만 둔다.
- 운영 판단, 재시도 가능 여부, 배치 대상 여부의 원천 상태는 항상 `Settlement.status`다.
- 포인트 금액 판단의 원천은 `point_history`이고, 현재 잔액 테이블의 `balance`는 재계산 가능한 캐시다.
- `member`는 사용자 식별·인증 책임만 가진다. `point_account.balance` 표현은 현재 사용 가능 포인트 잔액 캐시를 설명하는 brownfield/MVP observation이며, physical account shape 결정 권한이 아니다.

## 5. 정산 상태 흐름

### 5.1 MissionRoom 상태

| 값           | 의미         |
| ------------ | ------------ |
| `RECRUITING` | 모집 중      |
| `ACTIVE`     | 진행 중      |
| `CLOSED`     | 정상 종료    |
| `CANCELLED`  | 시작 전 취소 |

기본 흐름:

- `RECRUITING -> ACTIVE`: 시스템 lifecycle 규칙이 `start_at`에 frozen eligibility condition을 만족한다고 판단할 때 발생한다. MVP에서 `activated_at = start_at`이다.
- `RECRUITING -> CANCELLED`: `start_at` 평가 시점에 `LOCKED` participant 수가 `min_participants` 미만이거나 기존 취소 정책이 조건부 전이에 성공할 때 발생한다.
- `ACTIVE -> CLOSED`: 계획된 `end_at` cutoff 이후 정상 종료 처리로 발생한다.

자동 activation과 시작 만료 취소 batch는 모두 `RECRUITING` 상태를 조건으로 하는 시스템 전이다. 동시에 경합하면 하나만 성공하고, loser는 최종 room 상태를 재조회한다. 취소형 settlement 생성은 unique/idempotent해야 한다.

### 5.2 Settlement 상태

| 값           | 의미                                                                              |
| ------------ | --------------------------------------------------------------------------------- |
| `PENDING`    | 정산 row는 생성됐고 아직 워커가 claim하지 않은 실행 전                            |
| `RUNNING`    | 워커가 claim하여 실행 중                                                          |
| `SUCCEEDED`  | 모든 `settlement_item`의 지급 원장 연결과 대응 `point_history` 존재 검증까지 성공 |
| `FAILED`     | 자동 재시도 한도 초과 또는 비재시도 실패                                          |
| `RETRY_WAIT` | 실패했지만 자동/수동 재시도 대기                                                  |

기본 흐름:

```text
MissionRoom 종료/취소 감지
-> Settlement(PENDING) 생성
-> 배치 claim
-> RUNNING
-> SUCCEEDED

MissionRoom 종료/취소 감지
-> Settlement(PENDING) 생성
-> 배치 claim
-> RUNNING
-> 실패
-> RETRY_WAIT
-> 배치 또는 관리자 재시도
-> RUNNING
-> SUCCEEDED 또는 FAILED
```

규칙:

- 하나의 방에 대해 MVP 기준 정산은 `settlement_type`별 1건만 생성한다.
- `Settlement.status`는 생성 후 변경되지만, `settlement_type`과 `room_id`는 변경하지 않는다.
- 이미 `SUCCEEDED`인 정산은 다시 `PENDING`으로 되돌리지 않는다.
- `SUCCEEDED`는 immutable finality boundary다. replay, retry, support/admin recovery가 succeeded settlement snapshot이나 이미 기록된 point ledger를 overwrite하지 않는다.
- 일부 participant 지급만 완료됐거나 원장-FK 연결이 누락된 partial 상태는 복구 가능한 중간 상태이며, `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`로 남긴다.
- `RETRY_WAIT`/`FAILED`에서 retry가 갖는 권한은 unfinished execution completion authority다. 이미 authoritative하게 append된 ledger/item/snapshot은 그대로 두고, 누락된 item completion 또는 FK linkage만 idempotent하게 완료한다.

### 5.3 비정규화된 `MissionRoom.settlement_status`

필요하다면 `MissionRoom`에 아래 값을 둘 수 있다.

| 값           | 설명                     |
| ------------ | ------------------------ |
| `NONE`       | 아직 정산 없음           |
| `PENDING`    | 조회 최적화용 projection |
| `RUNNING`    | 조회 최적화용 projection |
| `SUCCEEDED`  | 조회 최적화용 projection |
| `FAILED`     | 조회 최적화용 projection |
| `RETRY_WAIT` | 조회 최적화용 projection |

단, 이 필드는 아래 원칙을 따른다.

- 조회와 목록 필터 성능을 위한 캐시성 필드다.
- 정산 처리의 조건 판단은 항상 `Settlement.status`를 기준으로 한다.
- `MissionRoom.settlement_status`와 `Settlement.status`가 어긋나면 `Settlement.status`를 신뢰한다.

## 6. 정산 배치 처리 흐름

### 6.1 Settlement 선생성

기존의 `MissionRoom을 직접 스캔해서 바로 정산`하는 구조보다, 정산 대상을 감지하는 시점에 `Settlement(PENDING)`를 먼저 생성하는 구조를 채택한다.

정상 종료 흐름:

```text
MissionRoom 상태가 ACTIVE -> CLOSED
-> Settlement(type=NORMAL, status=PENDING) 생성
-> 배치가 PENDING Settlement를 조회
-> claim 후 실행
```

취소 흐름:

```text
MissionRoom 상태가 RECRUITING -> CANCELLED
-> Settlement(type=CANCELLED_BEFORE_START, status=PENDING) 생성
-> 배치가 PENDING Settlement를 조회
-> claim 후 전액 환급 실행
```

이 구조를 쓰는 이유:

- 정산 대상의 존재를 데이터로 먼저 남길 수 있다.
- 배치가 `방 탐색`보다 `정산 실행`에 집중할 수 있다.
- 장애 후에도 어떤 방이 아직 정산 대기인지 명확하다.
- 운영자가 `왜 이 방이 정산되지 않았는지`를 상태만 보고 추적하기 쉽다.

안전장치:

- 종료 감지 로직이 실패해 `Settlement`가 생성되지 않은 경우를 대비해, 운영용 누락 탐지 잡 또는 관리자 API로 `PENDING` row 생성 복구를 할 수 있다.
- 이 복구는 누락 row 생성 continuation일 뿐, frozen certification outcome이나 settlement input을 수정하는 correction workflow가 아니다.
- 복구 절차는 `room_id` 기준으로 `CLOSED` 또는 `CANCELLED` 상태와 정산 대상 participant 존재 여부를 다시 확인한 뒤, 누락된 `Settlement(PENDING)`를 생성하는 흐름으로 고정한다.
- 이 복구 경로도 동일하게 `unique(room_id, settlement_type)` 제약을 통과해야 하며, 요청자/사유/시각/대상 `room_id`를 감사 로그에 append-only로 남긴다.

### 6.2 배치 대상 선택

배치의 기본 대상은 아래 조건을 만족하는 `Settlement`다.

- `status in (PENDING, RETRY_WAIT)`
- `retry_count < 3`
- `finished_at is null` 또는 재시도 대상 상태

배치는 더 이상 `MissionRoom.status + settlement_status` 조합을 원천 조건으로 사용하지 않는다. 방 상태는 검증용 컨텍스트일 뿐, 실제 실행 대상은 `Settlement` 행이다.

### 6.3 claim과 실행 순서

권장 처리 순서:

1. 대상 `Settlement` 목록 조회
2. 조건부 update로 `PENDING/RETRY_WAIT -> RUNNING` claim
3. 정산 입력 데이터 로드
4. 정산 계산 수행
5. `settlement` 집계값과 `settlement_item` 스냅샷 저장
6. 참여자별 `point_history`를 `idempotency_key`와 함께 생성
7. 생성된 `point_history.id`를 각 `settlement_item.point_history_id`에 연결
8. 모든 `settlement_item`에 대응하는 `point_history`가 존재하고 `point_history_id`가 채워졌는지 검증
9. 참여자/방 projection 갱신
10. 위 검증이 모두 통과한 경우에만 `Settlement.status = SUCCEEDED`로 종료
11. 커밋 후 `SettlementCompleted` 후속 이벤트 발행

claim SQL의 개념 예시:

```sql
update settlement
set status = 'RUNNING',
    batch_run_key = :batchRunKey,
    started_at = now(),
    failure_code = null,
    failure_message = null
where id = :settlementId
  and status in ('PENDING', 'RETRY_WAIT');
```

규칙:

- update 결과 row count가 `1`이면 claim 성공이다.
- row count가 `0`이면 다른 워커가 먼저 claim한 것이므로 즉시 skip한다.
- MVP 실행권은 DB conditional claim의 row count가 결정한다. row count가 `1`이면 해당 실행자가 이번 시도 실행권을 갖고, row count가 `0`이면 이미 다른 실행자가 claim한 것으로 보고 skip한다.
- 중복 지급 방어는 unique 제약, participant 단위 `point_history.idempotency_key`, payload consistency check, `settlement_item.point_history_id` linkage verification before SUCCEEDED로 유지한다.
- 계산 스냅샷인 `settlement_item`이 먼저 저장되고, 실제 잔액 원장인 `point_history`는 그 뒤에 생성되어 `point_history_id`로 연결된다.
- 정산 재시도 시 일부 participant의 `point_history`만 먼저 생성된 partial 상태를 허용한다. 이미 생성된 원장은 같은 `idempotency_key`로 중복 생성되지 않고, 아직 생성되지 않은 participant만 추가 반영한다.
- 재시도는 기존 `Settlement`와 기존/frozen `settlement_item` 스냅샷 기준으로 중단된 authoritative execution을 이어가는 행위다. 재시도는 정산 재계산, moderation re-resolution, frozen input mutation이 아니다.
- 하나라도 `point_history` 생성 또는 연결이 누락되면 `Settlement.status`를 `SUCCEEDED`로 전환할 수 없고, 재시도 가능 여부에 따라 `RETRY_WAIT` 또는 `FAILED`로 남겨야 한다.

### 6.4 트랜잭션 경계

- 정산은 `Settlement 1건 전체가 반드시 하나의 거대 트랜잭션으로 완주해야 한다`는 모델을 쓰지 않는다.
- 계산 스냅샷 저장과 participant별 포인트 지급/연결을 분리하고, 각 지급은 deterministic `idempotency_key`로 멱등하게 처리한다.
- `point_history` insert와 `point_account.balance` 갱신은 같은 participant 지급 트랜잭션에서 처리한다.
- 중간 실패로 일부 participant만 지급된 partial 상태는 허용하되 복구 가능한 상태로만 허용한다. 이때 `Settlement.status`는 `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`로 남긴다.
- 재시도는 이미 생성된 `point_history`를 중복 생성하지 않고 기존 원장을 재사용해 `settlement_item.point_history_id`를 연결하거나, 아직 미지급인 participant만 이어서 처리한다.
- `Settlement(PENDING)` 생성은 방 종료/취소 처리와 같은 트랜잭션 또는 직후 보상 가능한 짧은 트랜잭션으로 생성한다.

### 6.5 기존 직접 스캔 방식과 비교

선택한 안:

- 종료/취소 감지 시 `Settlement(PENDING)` 선생성
- 배치는 `PENDING`만 claim해서 처리

대안:

- 배치가 매번 `MissionRoom`을 직접 스캔해 정산 대상인지 판단

선택 이유:

- 대기열이 명시적으로 남아 장애 복구와 운영 추적이 쉽다.
- 정산 실패와 대상 누락을 구분할 수 있다.
- 동일 방 중복 처리 방어선을 `Settlement` 단위로 세우기 쉽다.

## 7. Settlement / SettlementItem / PointHistory 테이블 책임

### 7.1 `settlement`

책임:

- 정산 단위의 헤더
- 정산 상태의 소스 오브 트루스
- 집계 숫자, 실패 정보, 재시도 정보를 보관

권장 컬럼:

| 컬럼                              | 설명                                                                          |
| --------------------------------- | ----------------------------------------------------------------------------- |
| `id`                              | 정산 PK                                                                       |
| `room_id`                         | 대상 방                                                                       |
| `settlement_type`                 | `NORMAL`, `CANCELLED_BEFORE_START`                                            |
| `status`                          | `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `RETRY_WAIT`                     |
| `batch_run_key`                   | 배치 실행 식별자                                                              |
| `retry_count`                     | 누적 재시도 횟수                                                              |
| `total_participants`              | frozen `LOCKED` participant baseline 기준 정산 대상 participant 수           |
| `total_locked_amount`             | 정산 실행 시점 기준 총 잠긴 보증금 스냅샷                                     |
| `total_recognized_success`        | 전체 인정 성공 횟수                                                           |
| `total_base_refund_amount`        | 절사 전 잔액 배분 전 합계                                                     |
| `total_remainder_amount`          | 잔액 총액                                                                     |
| `remainder_policy`                | `DETERMINISTIC_REMAINDER_ALLOCATION`; brownfield `HOST_REMAINDER`는 legacy alias일 뿐 host reward/authority/discretion이 아님 |
| `remainder_winner_participant_id` | deprecated/brownfield. MVP remainder는 participant draw winner/top contributor를 쓰지 않음 |
| `failure_code`                    | 표준 실패 코드                                                                |
| `failure_message`                 | 최근 실패 원인 요약                                                           |
| `algorithm_version`               | 정산 semantic version (historical replay context)                              |
| `rule_context_snapshot`           | cadence/timezone/cutoff/lifecycle/remainder/reason mapping context JSON         |
| `started_at`                      | 실행 시작 시각                                                                |
| `finished_at`                     | 실행 종료 시각                                                                |
| `created_at`                      | 생성 시각                                                                     |
| `updated_at`                      | 수정 시각                                                                     |

제약:

- `unique(room_id, settlement_type)`
- `status`는 정산 처리의 원천 상태다.
- `total_participants`는 activation 시점 frozen `LOCKED` participant baseline 기준 participant 수를 의미한다.
- `WITHDRAWN`/ACTIVE withdrawal 기반 재계산은 brownfield/deferred semantics이며 MVP frozen baseline을 소급 변경하지 않는다.
- `total_locked_amount`는 정산 실행 시점의 정산 대상 participant `crew_participant.deposit_amount` 합계를 스냅샷으로 고정한 값이다.
- `total_locked_amount`는 `point_history`나 `point_account`를 다시 합산해 계산하지 않는다.
- MVP에서는 별도 `total_active_participants` 컬럼을 두지 않고, 필요 시 조회/분석용 후속 검토 항목으로 남긴다.
- `algorithm_version`과 rule interpretation snapshot은 versioned semantic replay를 위한 설명/감사 context다. 이 값들은 historical semantics를 reconstruct하기 위한 기준이며, succeeded settlement를 현재 엔진 기준으로 다시 쓰는 migration hook이 아니다.
- `remainder_policy`는 all-fail/remainder policy snapshot 역할도 수행한다. Legacy/brownfield alias가 남아 있어도 host/winner/draw payout authority를 부활시키지 않는다.

### 7.2 `settlement_item`

책임:

- 참여자별 정산 입력 스냅샷 저장
- 계산 결과와 계산 근거 저장
- 포인트 원장과 정산 계산 사이의 연결 고리 제공

용어 구분:

- `participant_id`는 특정 `MissionRoom` 참여 단위를 식별하는 값이다. 같은 회원이라도 방이 달라지면 다른 `participant`로 계산되며, 성공 횟수와 지분율, 정산 스냅샷은 이 단위로 확정된다.
- `member_id`는 실제 사용자 계정을 식별하는 값이다. `settlement_item`에 두 값을 함께 두는 이유는 계산 식별자는 `participant`, 실제 환급 대상은 `member`이기 때문이다.
- MVP에서는 한 `member`가 같은 방에 둘 이상의 `participant`를 가질 수 없으므로, 계산 단위와 지급 단위가 분리되어도 동일 방 내부에서는 일관된 1:1 대응이 유지된다.

권장 컬럼:

| 컬럼                          | 설명                                                                  |
| ----------------------------- | --------------------------------------------------------------------- |
| `id`                          | 아이템 PK                                                             |
| `settlement_id`               | 정산 헤더 FK                                                          |
| `participant_id`              | 참여자 FK                                                             |
| `member_id`                   | 회원 FK, 현재 스키마의 `user_id`와 매핑 가능                          |
| `participant_status_snapshot` | MVP에서는 frozen baseline의 `LOCKED`; `WITHDRAWN`은 brownfield/deferred |
| `deposit_amount`              | 잠긴 보증금 스냅샷                                                    |
| `success_count_raw`           | 기간 내 원시 성공 로그 수                                             |
| `recognized_success_count`    | 최종 인정 성공 횟수                                                   |
| `recognized_dates_count`      | 최종 인정된 날짜 수                                                   |
| `excluded_success_count`      | 제외된 성공 로그 수                                                   |
| `period_start_at`             | 계산 기간 시작                                                        |
| `period_end_at`               | 계산 기간 종료                                                        |
| `withdrawn_at_snapshot`       | brownfield/deferred withdrawal reference. MVP active settlement input 아님 |
| `share_ratio`                 | 최종 지분율                                                           |
| `raw_refund_amount`           | 절사 전 계산 금액                                                     |
| `base_refund_amount`          | `FLOOR` 적용 금액                                                     |
| `remainder_bonus_amount`      | deterministic remainder allocation이 특정 participant item에 설명상 귀속될 때의 잔액 가산분. winner/top contributor/host discretion 보너스 아님. all-fail equal-principal refund에서는 `0` |
| `reward_amount`               | 잠긴 보증금 대비 초과 환급분, `max(final_amount - deposit_amount, 0)` |
| `refund_amount`               | 실제 환급 총액, MVP에서는 `final_amount`와 동일                       |
| `final_amount`                | 최종 지급 금액                                                        |
| `draw_key_snapshot`           | non-payout 표시/설명 ordering에 사용한 키. 지급액 결정 권한 아님       |
| `tie_break_rank`              | non-payout 표시/설명 정렬 순위                                        |
| `calculation_reason`          | 포함/제외 근거 JSON 또는 TEXT                                         |
| `effective_moderation_snapshot` | 정산 시점 effective moderation state 설명 context (JSON)              |
| `moderation_chain_ref`        | append-only moderation transition chain reference (JSON)              |
| `point_history_id`            | 환급 원장 FK                                                          |
| `created_at`                  | 생성 시각                                                             |

설계 원칙:

- `settlement_item`은 결과뿐 아니라 계산 근거까지 저장해야 한다.
- `settlement_item`은 참여자별 정산 계산 결과의 source of truth고, `point_history`는 그 결과를 계정 잔액에 반영하는 금액 source of truth다. `Settlement.status = SUCCEEDED` 이후 두 테이블이 운영/분쟁/조회 기준이다.
- `deposit_amount`는 participant 단위로 잠겨 있던 보증금의 입력 스냅샷이며, 실제 잔액 반영은 `point_history`가 담당한다.
- `calculation_reason`은 `DAILY` 중복 제외, `SPECIFIC_DAYS` 비유효 요일 제외, resolved certification state, Phase 2/deferred cadence reference를 설명할 수 있어야 한다.
- `calculation_reason`은 reason-code mapping version과 함께 해석되어야 한다. 과거 settlement의 reason code는 현재 wording/UX 문구가 아니라 당시 vocabulary 기준으로 설명한다.
- Effective moderation state와 append-only moderation chain reference는 settlement-time input truth를 설명하기 위한 replay context다. Human memo, support note, UX wording, 운영 comment는 이 context를 보조할 수 있어도 authoritative settlement truth가 아니다.
- `AFTER_WITHDRAWN_AT` 같은 withdrawal cutoff 값은 brownfield/deferred reference이며 MVP frozen `LOCKED` baseline을 소급 변경하는 active rule이 아니다.
- `reward_amount`는 잠긴 보증금보다 더 많이 환급된 경우를 설명하기 위한 보조 저장값이다.
- 잠긴 보증금보다 적게 환급된 경우는 `deposit_amount`, `final_amount`, `share_ratio`, `recognized_success_count` 비교로 설명한다.
- `settlement_item`을 먼저 생성해 계산 결과를 고정하고, 이후 `point_history`를 생성한 뒤 `point_history_id`를 연결한다.
- 두 단계는 단일 row FK로만 강결합하지 않고, participant별 `idempotency_key`를 통해 느슨하게 이어진다. 따라서 partial 재시도에서도 이미 반영된 환급은 재사용하고 누락된 환급만 안전하게 이어서 처리할 수 있어야 한다.
- `point_history_id`는 정산 실행 중간 상태에서는 nullable일 수 있지만, `Settlement.status = SUCCEEDED`인 결과에서는 모두 채워져 있어야 한다.
- `Settlement.status = SUCCEEDED`가 되려면 모든 `settlement_item`이 유효한 `point_history`를 가리켜야 한다.

MVP `calculation_reason` vocabulary:

이 목록은 정산 스냅샷 진단/QA 검색성을 위한 대표 코드다. Public API enum, DB enum, DB constraint로 승격하지 않는다.

| code | 의미 |
| --- | --- |
| `DAILY_DUPLICATE` | `DAILY` 규칙에서 같은 일자 성공 로그 중 대표 1건 외 제외 |
| `INVALID_SCHEDULE_DAY` | `SPECIFIC_DAYS` 규칙에서 허용 요일이 아닌 성공 로그 제외 |
| `WEEKLY_N_OVERFLOW` | Phase 2/deferred `WEEKLY_N` reference. MVP active cadence 아님 |
| `AFTER_WITHDRAWN_AT` | brownfield/deferred withdrawal reference. MVP frozen baseline 변경 권한 아님 |
| `BEFORE_START` | activation/정산 계산 시작 cutoff 이전 로그 제외 |
| `AFTER_END` | 방 종료 cutoff 이후 로그 제외 |

`calculation_reason` 예시:

```json
{
  "includedDates": ["2026-05-01", "2026-05-02"],
  "excludedLogs": [
    {
      "serverTime": "2026-05-02T07:10:11+09:00",
      "code": "DAILY_DUPLICATE"
    },
    {
      "serverTime": "2026-05-05T08:30:00+09:00",
      "code": "AFTER_WITHDRAWN_AT"
    }
  ],
  "weeklyBuckets": [
    {
      "weekIndex": 1,
      "recognized": 3,
      "excluded": 1
    }
  ]
}
```

제약:

- `unique(settlement_id, participant_id)`

### 7.3 `point_history`

책임:

- 모든 포인트 증감의 유일한 원장
- 모든 포인트 이벤트의 멱등성 보장
- 잔액 추적과 감사 로그 제공

권장 컬럼:

| 컬럼               | 설명                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------- |
| `id`               | 원장 PK                                                                                |
| `member_id`        | 회원 FK, 현재 스키마의 `user_id`와 매핑 가능                                           |
| `amount`           | 증감 금액                                                                              |
| `balance_after`    | 반영 후 잔액                                                                           |
| `transaction_type` | `POINT_CHARGE`, `CREW_DEPOSIT_LOCK`, `CREW_SETTLEMENT_REFUND`, `CREW_CANCELLED_REFUND` |
| `reference_type`   | 예: `SETTLEMENT_ITEM`                                                                  |
| `reference_id`     | 참조 엔티티 PK                                                                         |
| `idempotency_key`  | 중복 반영 방지 키, 항상 `NOT NULL`                                                     |
| `created_at`       | 생성 시각                                                                              |

원칙:

- 모든 포인트 변경은 반드시 `point_history`를 통해서만 발생한다.
- `point_history`는 항상 `member_id` 기준으로 기록되며, 정산 계산 결과를 실제 계정 잔액 변화로 반영하는 금액 source of truth다.
- `PointAccount` 또는 `MemberPoint` 같은 현재 잔액 테이블이 있다면, 이 값은 항상 `사용 가능한 포인트 잔액`만 나타내는 재계산 가능한 캐시다.
- 현재 MVP/brownfield 설명은 잔액 캐시에 `pending_balance`, `waiting_balance`, `locked_balance` 같은 대기·잠금 상태 컬럼을 분리하지 않는 형태를 전제로 설명한다. 이는 physical balance shape를 새로 freeze하는 문장이 아니다.
- 현재 잔액 캐시와 `point_history` 원장 재계산값이 다르면 `point_history`를 source of truth로 삼고, 원인 조사 후 잔액 캐시를 보정하거나 재생성한다.
- 현재 MVP/brownfield 설명에서는 보증금이 별도의 계좌로 이동하지 않고, 참여 시점에 `point_account.balance`에서 차감되어 `crew_participant.deposit_amount`로 잠긴 상태로 관리된다. 이 표현의 semantic 핵심은 participant 단위 잠금 금액과 원장 추적이지, physical balance column 확정이 아니다.
- `CREW_DEPOSIT_LOCK`는 자산 이동이 아니라 기존 포인트를 사용 불가 상태로 전환하는 이벤트다.
- 정산 또는 취소 시점에만 해당 잠금 금액이 환급되며, 환급은 `point_history`를 통해 `member` 계정 잔액에 다시 반영된다.
- `point_history` insert와 `point_account.balance` 갱신은 동일 트랜잭션에서 처리한다.
- 정산 지급의 `reference_type + reference_id`는 어느 `settlement_item`에서 발생했는지 추적 가능해야 한다.
- 모든 포인트 변경은 이벤트 타입별 `idempotency_key`를 반드시 가진다.
- 동일 이벤트는 항상 동일한 `idempotency_key`를 생성해야 하며, `settlement.id` 같은 런타임 상태값에 의존하지 않는다.
- 이벤트별 고정 규칙 예시는 아래와 같다.
  - 포인트 충전: `charge:{paymentKey}`
  - 보증금 잠금: `deposit:room:{roomId}:participant:{participantId}`
  - 일반 정산 환급: `settlement:room:{roomId}:type:{settlementType}:participant:{participantId}:refund`
  - 취소형 정산 환급: `settlement:room:{roomId}:type:{settlementType}:participant:{participantId}:cancel_refund`
- `POINT_CHARGE`의 API field `payment_id`는 TossPayments `paymentKey`를 의미한다.
- `orderId`는 confirm 검증과 로그 상관관계 추적용이며 `point_history.idempotency_key`에 사용하지 않는다.
- 동일한 `paymentKey`는 반드시 하나의 충전 이벤트만 의미해야 하며, 재사용되거나 중복 발급되어서는 안 된다.
- `charge:{paymentKey}`는 이 불변성을 전제로 한 설계다. 이 조건이 깨지면 충전 멱등성 보장이 함께 깨진다.
- 동일 `idempotency_key`와 동일 payload의 재시도는 기존 `point_history`를 반환/연결하고, 동일 키에 다른 payload가 확인되면 idempotency conflict로 처리해 새 원장을 만들지 않는다.

제약:

- `unique(idempotency_key)`

### 7.4 `PointAccount`와 보증금 LOCK 모델

원칙:

- 포인트 충전은 현재 brownfield/MVP 설명에서는 `point_account.balance`를 증가시키는 일반 잔액 충전으로 관찰된다. 이 표현은 물리 balance shape를 새로 freeze하지 않는다.
- Layering note: 이 절의 `point_account` 형태는 현재 brownfield/MVP observation을 설명하는 비권위 구현 맥락이다. semantic invariant는 `point_history`가 금액 source of truth이고 balance 계열 값은 재계산 가능한 조회/캐시라는 점이다. `available/locked/pending/total` 같은 physical balance shape 후보는 이 cleanup에서 새로 freeze하지 않는다.
- `point_account`는 `member`와 분리해 사용자 식별·인증 책임과 포인트 잔액 갱신 책임을 나눈다. 현재 문서의 `balance` 표현은 사용 가능 잔액 캐시를 설명하는 brownfield/MVP observation이며, 향후 physical balance shape 결정 권한이 아니다.
- 크루 참여 시 보증금은 별도 자산으로 이동하지 않고, `point_account.balance`에서 차감되어 해당 `crew_participant.deposit_amount`에 participant 단위 잠금 금액으로 기록된다.
- 보증금 잠금 상태는 `point_account.locked_balance`가 아니라 `balance` 차감과 `crew_participant.deposit_amount` 기록으로 표현한다.
- 사용자에게 보여줄 `GET /api/points.locked_balance`는 정산 전 참여 보증금 합계를 API projection으로 제공할 수 있다.
- 이 projection은 UX 표시용이며 정산 계산, 포인트 원장, 출금 가능 여부, 환급 가능 여부, 분쟁 처리, 정산 결과 판단의 source of truth가 아니다.
- MVP projection은 `crew_participant.deposit_amount`와 `mission_room.status IN ('RECRUITING', 'ACTIVE', 'CLOSED')`를 기준으로 시작하며, settlement 조인을 강제하지 않는다.
- `CLOSED` 포함은 정산 완료 전까지 잠겨 있을 것으로 기대되는 보증금 표시를 위한 근사값이다. `Settlement.status = SUCCEEDED` 이후 lock 해제 여부를 더 정확히 제외하는 조건은 Settlement 조회/정산 구현 단계에서 보강할 수 있다.
- 정산 계산의 입력 금액은 여전히 정산 대상 participant의 `crew_participant.deposit_amount` 합계다.
- 보증금 잠금은 `point_account`에 대한 조건부 update로 수행한다. 즉, `WHERE balance >= deposit_amount` 조건을 포함해 잔액이 충분할 때만 차감한다.
- 이 update의 row count가 `1`일 때만 잠금 성공으로 간주하고, `0`이면 동시 요청 또는 잔액 부족으로 보고 참여를 실패 처리한다.
- 보증금 잠금 처리, `crew_participant` 생성, `CREW_DEPOSIT_LOCK` 원장 생성은 반드시 하나의 트랜잭션으로 처리한다.
- 권장 순서는 `point_account.balance` 조건부 차감 -> `crew_participant` 생성 및 `deposit_amount` 반영 -> `CREW_DEPOSIT_LOCK point_history` 생성이다.
- 위 세 단계 중 하나라도 실패하면 전체 롤백한다. 잔액만 차감되고 participant가 생성되지 않거나, participant만 생기고 원장이 누락되는 상태를 허용하지 않는다.
- ACTIVE withdrawal은 MVP active semantics가 아니라 brownfield/deferred다. 향후 재도입하더라도 `deposit_amount` 즉시 환급이나 frozen settlement input 변경으로 해석하지 않는다.
- 최종 정산 또는 취소 환급이 일어날 때만 `point_history`를 통해 `balance`가 증가한다.
- 운영 검증이나 복구 중 `point_account.balance`가 `point_history` 기반 재계산값과 다르면 `point_history`를 기준으로 캐시를 복구한다.

## 8. 인정 성공 횟수 계산 규칙

### 8.1 공통 조건

인정 성공 횟수에 포함되려면 아래를 모두 만족해야 한다.

1. 해당 참여자의 로그여야 한다.
2. `is_success = true`여야 한다.
3. `server_time`이 `room.activated_at` 이상이고 방의 종료 시점 이전이어야 한다.
4. participant가 activation 시점 frozen `LOCKED` baseline에 포함되어야 한다. Withdrawal cutoff는 MVP active settlement input이 아니라 brownfield/deferred reference다.

### 8.2 DAILY

- 기준: 미션 기간 동안 하루 최대 1회 인정
- 구현: 날짜별 성공 로그를 집계하고 같은 날 다중 성공은 1회만 인정
- 스냅샷: `recognized_dates_count`, `excluded_success_count`, `calculation_reason`에 중복 제외 근거 저장

### 8.3 SPECIFIC_DAYS

- 기준: `mission_schedule_day`에 정의된 반복 요일만 유효
- 구현: 성공 로그의 `server_time` 요일이 스케줄 테이블과 일치할 때만 인정
- 같은 유효 날짜에 다중 성공이 있어도 1회만 인정
- 스냅샷: 제외된 로그는 `INVALID_SCHEDULE_DAY`, `DAILY_DUPLICATE` 같은 코드로 남긴다

### 8.4 WEEKLY_N (Phase 2 / deferred)

- `WEEKLY_N`은 PRD synthesis 기준 Phase 2로 이연한다.
- 아래 규칙은 brownfield/reference 후보이며 MVP active settlement semantics로 사용하지 않는다.
- 향후 재도입 시에도 activation anchor는 시스템이 고정한 `room.activated_at = start_at`을 기준으로 해야 하며, host/admin manual activation으로 해석하면 안 된다.
- Reference 후보: `week_index = floor(days_between(kst_date(room.activated_at), log_date) / 7) + 1`, bucket별 상한, `calculation_reason.weeklyBuckets` 저장.

### 8.5 인정 성공 계산 전략

- `frequency_type`별 인정 성공 계산은 `SettlementBatch`나 `PointHistory`가 아니라 별도 recognition strategy가 책임진다.
- 각 전략은 `MissionLog` 원본과 참여자 스냅샷을 입력으로 받아 아래 출력만 반환한다.
  - `recognized_success_count`
  - `recognized_dates_count`
  - `excluded_success_count`
  - `calculation_reason`
- `Settlement`의 생성, claim, 상태 전이, 포인트 반영 흐름은 유지하고, 빈도별 인정 규칙 차이는 전략 내부에서만 흡수한다.
- MVP active cadence는 `DAILY`, `SPECIFIC_DAYS`만 고정한다. `WEEKLY_N`은 Phase 2/deferred reference로만 남긴다.
- 향후 연속 성공 보너스, 요일별 가중치 같은 정책이 필요해지더라도 `SettlementBatch`나 `PointHistory`를 수정하지 않고 전략 구현 추가로 확장한다.

### 8.6 왜 실시간 수치와 최종 수치가 달라질 수 있는가

- 실시간 대시보드는 빠른 projection이며, `SUCCEEDED` 전 정산 계산은 `MissionLog`, frozen `LOCKED` participant baseline, resolved certification state를 다시 읽어 final batch에서 확정한다.
- 실시간 대시보드는 캐시 반영 지연이 있을 수 있다.
- `SUCCEEDED` 전 정산 계산은 로그 원본, frozen baseline, resolved certification state, cadence 대표 선택을 계산한다. `SUCCEEDED` 이후 같은 입력으로 다시 검증하는 행위는 replay/audit이며 final payout 변경 권한이 아니다.
- `mission_log.failure_reason`은 인증 요청 시점의 1차 실패 사유만 저장하고, `DAILY` 중복 제외나 `SPECIFIC_DAYS` 비유효 요일 제외 같은 정산 단계 설명은 `settlement_item.calculation_reason`이 맡는다. `WEEKLY_N` 상한은 Phase 2/deferred reference다.
- 정산 화면은 파생 표시일 뿐 source of truth가 아니다. `Settlement.status = SUCCEEDED` 이후 금액의 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다.

### 8.7 `failure_reason`과 `calculation_reason` 책임 분리

- `mission_log.failure_reason`은 인증 시점 실패 사유를 저장한다. 예: `BEFORE_START`, `AFTER_END`. `EXIF_MISSING` 같은 EXIF 부재/이상은 단독 authoritative failure가 아니라 risk/review signal로 보존하고 moderation/review flow에서 해석한다.
- `settlement_item.calculation_reason`은 정산 시점 포함/제외 근거를 저장한다.
- 인증 자체가 실패한 경우에는 `mission_log.failure_reason`을 채우고, 필요하면 정산 스냅샷에도 같은 제외 근거를 남길 수 있다.
- 인증은 성공했지만 정산 규칙상 제외된 경우에는 `mission_log.failure_reason` 없이 `calculation_reason`에만 남긴다.
- 예를 들어 `DAILY` 중복은 `calculation_reason`만 기록하고, `BEFORE_START`/`AFTER_END`처럼 서버 시간 기준으로 인증 시점에 이미 실패한 건은 `failure_reason`과 `calculation_reason` 양쪽에 함께 남길 수 있다. EXIF/hash 부재·이상은 fraud/risk signal이며 단독 authoritative failure가 아니다.

### 8.8 `failure_reason`(system axis)과 `reject_reason_code`(moderation axis) 분리

- `mission_log.failure_reason`은 system/timing/upload validation axis다. 서버가 인증 시점에 결정하는 system 실패 사유(예: `BEFORE_START`, `AFTER_END`, EXIF risk signal)를 저장한다.
- `mission_log.reject_reason_code`는 host moderation rejection axis다. 호스트 검수자가 결정하는 거절 사유 코드이며, 6종 enum + `OTHER` 존재만 freeze되고 정확한 값 이름은 deferred decision이다.
- 두 axis는 서로 다른 의미 vocabulary다. 한쪽 enum 값을 다른 쪽에 재사용하거나 한쪽 컬럼에 다른 axis의 값을 저장하지 않는다.
- host 거절은 `mission_log.failure_reason`에 기록하지 않고 `reject_reason_code` + `decision_type` + `moderator_*` 컬럼과 `moderation_history` append로 표현한다.

## 9. 정산 알고리즘

### 9.1 기본 공식

```text
지분율 = 참여자 인정 성공 횟수 / 전체 참여자 인정 성공 횟수 합계
raw_refund_amount = total_locked_amount × 지분율
base_refund_amount = FLOOR(raw_refund_amount)
remainder = total_locked_amount - SUM(base_refund_amount)
final_amount = base_refund_amount + remainder_bonus_amount
reward_amount = max(final_amount - deposit_amount, 0)
refund_amount = final_amount
```

### 9.2 일반 정산

일반 정산은 `전체 인정 성공 횟수 > 0`인 경우다.

1. 전체 인정 성공 횟수를 계산한다.
2. 각 참여자의 지분율을 계산한다.
3. 각 참여자의 `raw_refund_amount`를 `DECIMAL128`로 계산한다.
4. 각 참여자의 `base_refund_amount`에 `FLOOR`를 적용한다.
5. 남은 `remainder`를 계산한다.
6. 일반 정산에서 절사 후 남은 잔액은 deterministic remainder allocation rule로 처리한다. MVP brownfield alias가 `HOST_REMAINDER`로 남아 있더라도 의미는 replayable floor-remainder calculation metadata이며 host reward/authority/privilege가 아니다.
7. 이 remainder 처리는 host가 settlement authority를 가진다는 뜻이 아니며, host가 금액·원장·최종 정산을 선택하거나 override할 수 없다.

참고:

- 잠긴 보증금이 참여자마다 달라도 공식은 동일하다.
- `total_locked_amount`는 정산 실행 시점의 정산 대상 participant `deposit_amount` 합계 스냅샷이다.
- `total_locked_amount`는 `point_history`나 `point_account` 잔액을 다시 합산해 계산하지 않는다.
- 누가 본인 보증금보다 많이 또는 적게 돌려받았는지는 `deposit_amount`와 `final_amount` 비교로 설명한다.

### 9.3 전원 성공 0회 정산

`전체 인정 성공 횟수 = 0`이면 전원 equal-principal refund를 적용한다. 이는 같은 금액을 나눠 갖는 규칙이 아니라 각 참여자가 잠겨 있던 자기 원금을 그대로 돌려받는 규칙이다.

```text
for each participant:
  base_refund_amount = deposit_amount
  final_amount = deposit_amount
  refund_amount = deposit_amount
  remainder_bonus_amount = 0
total_remainder_amount = 0
```

규칙:

- 모든 참여자의 `base_refund_amount = deposit_amount`
- 각 참여자의 `final_amount = deposit_amount`로 고정한다. 즉, 잠겨 있던 자기 원금을 그대로 환급하는 equal-principal refund다.
- 이 분기에서는 `remainder_bonus_amount = 0`, `reward_amount = 0`, `remainder = 0`으로 수렴해야 한다.
- 이 분기에서는 추가 차감 규칙을 두지 않는다.
- 이 분기에서는 remainder allocation, legacy top-contributor/draw/winner 지급 규칙을 사용하지 않는다. 누군가의 전원 실패가 다른 참여자의 추가 환급으로 이어지지 않는다.

### 9.4 잔액 상한

- 일반/비례 정산에서 참여자 수가 `n`명일 때 절사 후 잔액은 항상 `0 <= remainder < n`이다. all-fail equal-principal refund에서는 각자 원금 전액 환급이므로 `total_remainder_amount = 0`이어야 한다.
- PRD synthesis의 MVP 최대 인원 `15명` 기준으로 남는 잔액은 `최대 14원`이다.
- 원문 기획안/legacy 기획서의 `1~10원`은 설명용 legacy 표현으로 보고, 구현 기준은 위 수학적 상한으로 고정한다.

### 9.5 tie / representative ordering 규칙

MVP payout remainder는 draw/winner가 아니라 deterministic remainder allocation rule을 사용한다. Brownfield `HOST_REMAINDER` 명칭이 남아 있더라도 host reward, privilege, or discretionary payout을 뜻하지 않는다. 따라서 draw/tie ordering은 최종 지급액을 바꾸는 권한이 아니다.

필요한 경우 대표 성공 로그, 동일 점수 표시 순서, 설명용 tie-break 같은 비금액성 UX에는 아래처럼 재현 가능한 ordering을 사용할 수 있다.

```text
ordering_key = stable domain input(room_id, participant_id/member_id, event_id 등)
정렬 기준: ordering_key 오름차순 또는 created_at ASC, id ASC
```

원칙:

- `settlement.id` 같은 실행 시점 생성 PK나 런타임 랜덤에 의존하지 않는다.
- 같은 입력이면 재시도, replay, 테스트, 데이터 이관 이후에도 같은 표시/설명 순서가 나와야 한다.
- 이 ordering은 settlement amount, remainder recipient, point_history amount를 변경하지 않는다.

## 10. 멱등성 및 동시성 제어

동시성 제어는 `인증 시점`과 `최종 정산 시점`을 분리해 다룬다. 인증은 빠른 기록 보존이 우선이고, 정산은 중복 실행 방지가 우선이다.

### 10.1 인증 시점 동시성 원칙

- 인증 업로드와 `MissionLog` 저장 시점에는 추가 실행 조정 계층을 기본값으로 두지 않는다.
- 인증은 `MissionLog` append-only 저장을 우선해 원본 로그 유실 가능성을 낮춘다.
- 실시간 대시보드, 예상 지분율, 임시 진행/참여도 표시는 Redis `INCR`, `SETNX`, Lua script 같은 원자적 캐시 연산으로 갱신할 수 있다.
- 위 실시간 지표는 사용자 안내용 current-basis 추정값일 뿐이며, 최종 정산 금액의 source of truth가 아니다.
- 실시간 캐시가 일부 지연되거나 누락돼도 최종 정산은 반드시 `MissionLog` 원본을 다시 읽어 확정한다.
- 따라서 인증 시점에는 선행 직렬화보다 `로그 보존 + 원자적 캐시 갱신 + 사후 replay/audit 가능성`을 우선한다.

### 10.2 정산 시점 방어선

- 최종 정산 시점에는 중복 정산/중복 지급 방지를 위해 여러 방어선을 함께 사용한다.
- 1차 실행권은 DB 조건부 claim(DB conditional claim)이 결정한다. DB unique, `point_history.idempotency_key`, payload consistency check, `settlement_item.point_history_id` linkage verification before SUCCEEDED가 MVP 최종 방어선이다.
- 이 절에서 고정하는 semantic은 duplicate payout 방지, idempotent recovery, conditional claim 우선순위, point_history 멱등성이다. 분산 실행 조정 전략은 MVP correctness proof가 아니다.
- 아래 10.3~10.6은 같은 방에 대해 배치 워커, 관리자 재시도, 복구 작업이 겹쳐도 결과가 participant 단위로 한 번만 확정되도록 하는 장치다.

### 10.3 상태 기반 claim

- 워커는 `PENDING` 또는 `RETRY_WAIT` 상태의 `Settlement`만 대상으로 한다.
- claim은 조건부 update로 수행한다.
- 같은 `Settlement`를 두 워커가 동시에 집어도 row count `1`을 받은 워커만 실행한다.

### 10.4 Phase 2 scale hardening candidate (non-authoritative)

- Redisson / Redis distributed lock은 MVP requirement나 optional runtime path가 아니라 Phase 2 scale hardening candidate only다.
- MVP가 고정하는 것은 배치 워커와 관리자 재시도, 복구 작업이 같은 방을 동시에 처리하려 해도 DB conditional claim, unique constraint, participant 단위 idempotency, point_history 방어선으로 중복 지급이 발생하지 않아야 한다는 invariant다.
- 다중 인스턴스 정산 워커, 관리자 재시도 동시성, 복구 스크립트 동시성이 MVP 범위를 넘어 실제 운영 요구가 될 때만 별도 architecture decision에서 분산 실행 조정 전략을 검토한다.
- Redis availability, Redis HA, lease timeout, watchdog, split brain 같은 운영 복잡도와 구체 topology는 이 문서의 MVP runtime path와 correctness proof에 포함하지 않는다.

### 10.5 DB unique 제약

- `unique(room_id, settlement_type)`
- `unique(settlement_id, participant_id)`
- `unique(point_history.idempotency_key)`

역할:

- 중복 정산 헤더 생성 방지
- 동일 참여자 아이템 중복 생성 방지
- 포인트 중복 지급 방지

### 10.6 PointHistory idempotency

이벤트별 권장 키:

```text
charge:{paymentKey}
deposit:room:{roomId}:participant:{participantId}
settlement:room:{roomId}:type:{settlementType}:participant:{participantId}:refund
settlement:room:{roomId}:type:{settlementType}:participant:{participantId}:cancel_refund
```

원칙:

- 모든 포인트 이벤트는 `idempotency_key`가 필수고, 이벤트 타입별 생성 규칙을 고정한다.
- `settlement.id`는 실행 시 생성되는 PK이므로 `idempotency_key` 구성값으로 쓰지 않는다.
- `room_id`, `settlement_type`, `participant_id`처럼 입력 기반 식별자를 사용해 재시도, replay/audit 테스트, 데이터 이관 상황에서도 같은 키가 재현되도록 한다.
- `participant`는 물리 삭제하지 않고 같은 방 재참여도 지원하지 않으므로, 같은 정산 대상에 대한 `participant_id`는 생명주기 동안 안정적으로 유지된다.
- 이 규칙은 non-payout stable ordering과 같은 철학을 따른다. 즉, 런타임 생성값이 아니라 동일 입력이면 동일 결과가 나와야 한다.
- 동일 이벤트는 항상 동일한 `idempotency_key`를 사용한다.
- `POINT_CHARGE`의 API field `payment_id`는 TossPayments `paymentKey`를 의미하며, 하나의 충전 이벤트에만 1:1로 매핑되어야 한다.
- `orderId`는 confirm 검증과 로그 상관관계 추적용이며 `point_history.idempotency_key`에 사용하지 않는다.
- 동일한 `paymentKey`가 재사용되거나 중복 발급되면 `charge:{paymentKey}` 기반 멱등성이 깨지므로, 결제 연동 계층에서 이를 허용하지 않아야 한다.
- provider success 이후 client timeout이 발생해도 같은 `paymentKey` 재시도는 중복 충전이 아니라 기존 원장 재사용으로 수렴해야 한다.
- 재시도 중 중복 insert가 발생하면 unique 제약으로 차단된다.
- 애플리케이션은 동일 키 충돌을 먼저 기존 `point_history`와 요청 payload가 같은 semantic event인지 검증한다. 동일 payload면 기존 원장을 재사용/연결하고, 다른 payload면 idempotency conflict로 실패 처리한다.
- 애플리케이션은 이 충돌을 `이미 지급됨`으로 해석하고 무조건 성공 처리하지 말고, 현재 `Settlement.status`, `settlement_item.point_history_id`, `point_history` payload 일치 여부를 함께 검증해야 한다.
- 이 원칙은 정산 환급뿐 아니라 충전, 보증금 잠금 같은 모든 포인트 이벤트에도 동일하게 적용한다. 따라서 `point_history.idempotency_key`는 항상 `NOT NULL`이어야 한다.

### 10.7 Implementation guardrails (non-authoritative topology)

이 절은 stack topology를 freeze하지 않고, 구현 편의나 batch metadata로 대체할 수 없는 semantic guardrail만 나열한다.

- `Settlement.status` 조건부 claim(DB conditional claim)이 실행권의 최종 기준이다. 분산 실행 조정 계층은 MVP 실행권 authority나 finality proof가 아니다.
- `settlement_item`은 먼저 생성되어 participant별 계산 snapshot을 고정하고, 이후 `point_history` 생성/재사용 결과를 `point_history_id`로 연결한다.
- retry는 새 계산 결과를 만드는 재정산이나 correction이 아니라, 기존 `Settlement`와 `settlement_item` 기준으로 미완료 participant만 복구하는 작업이다.
- replay는 settlement-time input/rule/snapshot으로 결과 재현성을 검증하는 audit 동작이며, payout mutation이나 succeeded settlement 변경이 아니다.
- 이미 `point_history_id`가 연결된 item은 재지급하지 않는다.
- `point_history`는 있으나 `settlement_item.point_history_id`만 누락된 경우 새 원장을 만들지 않고 기존 원장 payload를 확인한 뒤 FK 연결 보정 대상으로 다룬다.
- `settlement_item.point_history_id`가 non-null인데 대응 `point_history`가 없으면 `INVALID_INCONSISTENT`로 취급하고 `SUCCEEDED`로 보지 않는다.
- retry는 item-level idempotent recovery만 수행한다. 기존 snapshot을 폐기하거나 current engine 산출값으로 payout을 교체하는 동작은 retry가 아니라 금지된 hidden mutation이다.
- 모든 item의 `point_history_id`와 대응 `point_history` 존재가 검증되기 전까지 parent `Settlement.status`를 `SUCCEEDED`로 바꾸지 않는다.
- Email/AI/FCM/SSE 같은 후속 이벤트 실패는 settlement, settlement_item, point_history를 rollback하지 않는다.
- Notification은 best-effort re-entry hint이자 reconciled UX signal이다. FCM/SSE/알림 payload, inbox/read state, delivery attempt가 누락·중복·역순·실패 상태여도 authoritative REST state와 `Settlement`/`settlement_item`/`point_history`가 우선한다.
- Notification retry는 FCM delivery transport retry이며 settlement retry/replay/correction이 아니다. notification delivery/read/failure 상태는 `Settlement.status`, settlement item, point ledger/history를 변경하지 않는다.
- AI report/explanation은 non-authoritative 후행 artifact다. AI 산출물은 settlement authority, replay authority, payout truth가 아니며, version/stale/invalidation-aware artifact lifecycle이나 regeneration append semantics는 Phase 2 hardening guardrail로 남긴다.

## 11. 실패/재시도 정책

### 11.1 실패 코드 표준

| failure_code           | 의미                                         | 자동 재시도 | 운영 액션                             |
| ---------------------- | -------------------------------------------- | ----------- | ------------------------------------- |
| `INPUT_LOAD_FAILED`    | MissionLog, Participant, Room 입력 로드 실패 | 가능        | `RETRY_WAIT` 후 재시도                |
| `CALCULATION_FAILED`   | 계산 중 예외 또는 불일치 감지                | 불가        | `FAILED`, 감사 로그 기반 원인 분석 후 재시도/복구 판단 |
| `POINT_CREDIT_FAILED`  | 포인트 원장 반영 실패                        | 가능        | `RETRY_WAIT` 후 재시도                |
| `DUPLICATE_SETTLEMENT` | 중복 정산 생성 또는 이미 존재하는 정산 감지  | 불가        | 데이터 점검 후 기존 settlement 조회/복구 판단          |
| `UNKNOWN`              | 분류되지 않은 예외                           | 제한적 가능 | 기본은 `RETRY_WAIT`, 반복 시 `FAILED` |

### 11.2 상태 전이 규칙

- 실행 실패 시 `retry_count + 1`
- `retry_count < 3`이고 재시도 가능한 코드면 `RETRY_WAIT`
- `retry_count >= 3`이거나 비재시도 코드면 `FAILED`
- 성공 시 `SUCCEEDED`

### 11.3 장애 복구 원칙

- `RUNNING` 상태에서 프로세스가 비정상 종료된 건은 운영 배치가 `started_at` 기준 timeout을 보고 `RETRY_WAIT`로 재전환할 수 있어야 한다.
- 장애 복구는 항상 기존 `Settlement`를 재사용한다. 같은 방에 새 `Settlement`를 추가 생성하지 않는다.
- `PointHistory`가 이미 생성된 경우에도 `idempotency_key`가 중복 지급을 막는다.
- 복구 실행은 이미 `point_history_id`가 연결된 `settlement_item`은 그대로 두고, 아직 반영되지 않은 participant만 이어서 처리할 수 있어야 한다.
- 정산 재시도 중에는 일부 participant에 대해 `point_history`는 이미 생성되었지만 `settlement_item.point_history_id` 연결만 누락된 partial 상태가 발생할 수 있다.
- 이 경우 해당 participant는 이미 포인트 지급이 완료된 상태로 간주하고, 재시도는 새 `point_history`를 만들지 않아야 한다.
- 운영 복구는 관리자 API 또는 배치를 통해 기존 `point_history`를 조회한 뒤 `settlement_item.point_history_id`만 연결하는 방식으로 수행할 수 있어야 한다.
- 이 보정 작업은 재정산이나 correction이 아니라 FK 연결 보정이며, frozen certification outcome, succeeded settlement snapshot, authoritative daily/final result, 기존 포인트 지급 이벤트를 변경하지 않는다.
- 운영자는 `point_history` 자체가 없는 지급 실패와, `point_history`는 있으나 `settlement_item` 연결만 누락된 상태를 구분해서 대응해야 한다.
- 전자는 미반영 participant에 대한 재시도 대상이고, 후자는 FK 연결 보정 대상이다.
- `Settlement.status`는 `point_history` 생성 여부만이 아니라 `settlement_item`과의 연결 완료 여부까지 포함해 판단한다.
- 모든 `settlement_item`의 `point_history_id`가 채워지고 대응 `point_history` 존재가 검증되기 전까지는 `Settlement.status`를 `SUCCEEDED`로 바꾸지 않는다.
- Scheduler downtime이나 delayed batch execution은 audit/recovery fact로 기록하되 lifecycle authority를 바꾸지 않는다. `start_at`, room timezone, daily cutoff, mission period end 같은 scheduled semantic anchor가 계약 기준이며, 실제 job 실행 시각이 cutoff를 밀거나 participant/log eligibility를 확장하지 않는다.

## 12. 예외 케이스

| 시나리오                     | 결정                                                                                                           |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 인원 미달 방 취소            | 방별 `min_participants`를 기준으로 `CANCELLED_BEFORE_START` 정산 생성 후 전액 환급                             |
| 중도 탈퇴                    | ACTIVE withdrawal은 brownfield/deferred. MVP 정산은 frozen `LOCKED` baseline을 소급 변경하지 않음              |
| DAILY 하루 다중 인증         | 같은 날짜 성공 로그는 1회만 인정, 나머지는 제외 근거 저장                                                      |
| SPECIFIC_DAYS 비유효 요일    | `mission_schedule_day`에 없는 요일의 성공 로그는 제외                                                          |
| WEEKLY_N 상한 초과           | Phase 2/deferred reference. MVP active cadence 아님                                                           |
| 전체 인정 성공 0회           | 각 참여자의 잠겨 있던 자기 보증금을 equal-principal refund로 전액 환급한다. host/winner/draw remainder 지급 없음 |
| 참여자별 보증금 상이         | 총 풀은 합산하되, 결과 설명은 `deposit_amount`, `final_amount`, `share_ratio`로 제공                           |
| `ACTIVE` 이후 신규 참여 요청 | 거절한다. MVP에서는 모집 완료 후 참여자 구성을 고정한다.                                                       |
| 탈퇴 후 동일 방 재참여 요청  | MVP active flow에서는 지원하지 않고 거절한다. WITHDRAWN/rejoin은 brownfield/deferred이며 frozen baseline을 변경하지 않는다. |
| 같은 방 중복 정산 시도       | 상태 claim + unique 제약 + `point_history.idempotency_key` + payload consistency check + `point_history_id` 연결 검증으로 차단 |
| `Settlement` 누락            | 운영 복구 경로로 `PENDING` 생성, 단 `unique(room_id, settlement_type)` 준수                                    |
| 이미 `SUCCEEDED`인 방 재요청 | 새 정산 생성 금지, 기존 결과 조회만 허용                                                                       |

## 13. 내부 계약

### 13.1 서비스 인터페이스

```text
Settlement createPendingSettlementIfAbsent(roomId, settlementType)
SettlementInput loadSettlementInput(settlementId)
SettlementResult calculateSettlement(input)
SettlementExecutionResult executeSettlement(settlementId, batchRunKey)
void creditRefundIfAbsent(memberId, amount, transactionType, idempotencyKey, referenceType, referenceId)
```

### 13.2 MissionRecognitionStrategy

```text
MissionRecognitionStrategy
- supports(frequencyType)
- recognize(input, participant)
```

원칙:

- 전략 구현은 `MissionLog` 원본과 참여자 스냅샷을 읽어 인정 성공 결과만 반환한다.
- `SettlementBatch`는 어떤 전략을 호출할지 선택만 하고, 빈도별 인정 규칙 자체를 소유하지 않는다.
- 새 빈도 정책이 추가될 때 `PointHistory`, 락 정책, 배치 claim 로직을 함께 수정하지 않도록 경계를 고정한다.

### 13.3 SettlementInput

```text
settlementId
roomId
settlementType
roomStatus
frequencyType
frequencyDays
frequencyCount
timezone
participants[]
missionLogs[]
totalLockedAmount
minParticipants
startAt
endAt
algorithmVersion
ruleInterpretationSnapshot
reasonCodeMappingVersion
effectiveModerationStateSnapshot
moderationChainReference
```

`ruleInterpretationSnapshot`은 cadence interpretation, timezone/cutoff semantics, lifecycle cutoff semantics, all-fail/remainder policy를 포함하는 semantic context다. 이 내부 계약은 replay explanation/audit를 위한 것이며 public API field 확정이나 event-sourcing redesign을 뜻하지 않는다.

### 13.4 SettlementResult

```text
settlementHeader
participantResults[]
totalLockedAmount
totalRecognizedSuccess
totalBaseRefundAmount
totalRemainderAmount
remainderPolicy
```

원칙:

- `remainderPolicy`는 deterministic allocation metadata이며 host/winner authority가 아니다.
- `SettlementResult`는 historical semantic truth reconstruction에 필요한 snapshot/version context를 보존해야 하지만, replay 결과가 final payout을 변경하는 authority가 되면 안 된다.

## 14. 외부 API 계약

> 외부 API의 최종 요청/응답 계약은 `API-spec-dondok.md`를 따른다. 이 섹션은 정산 운영 흐름 설명을 위한 보조 설명이며, API 계약과 충돌할 경우 API-spec이 우선한다.

### 사용자 조회 API

#### `GET /api/rooms/{roomId}/settlement`

용도:

- 방 기준으로 현재 정산 상태와 정산 식별자를 조회한다.
- 취소형 정산도 같은 응답 구조 사용

응답 예시:

```json
{
  "room_id": 42,
  "settlement_id": 501,
  "settlement_type": "NORMAL",
  "status": "RUNNING",
  "retry_count": 1,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-02T13:12:10+09:00",
  "finished_at": null
}
```

참여자별 정산 상세 결과는 API-spec의 `GET /api/settlements/{settlementId}` 계약을 따른다.

### 관리자 API

#### `GET /api/admin/settlements?status=FAILED`

응답 예시:

```json
{
  "items": [
    {
      "settlement_id": 501,
      "room_id": 42,
      "settlement_type": "NORMAL",
      "status": "FAILED",
      "retry_count": 3,
      "failure_code": "POINT_CREDIT_FAILED",
      "failure_message": "point_history insert timeout",
      "started_at": "2026-06-02T13:12:10+09:00",
      "finished_at": "2026-06-02T13:12:20+09:00"
    }
  ]
}
```

#### `POST /api/admin/settlements/{settlementId}/retry`

응답 예시:

```json
{
  "settlement_id": 501,
  "room_id": 42,
  "status": "RUNNING",
  "retry_count": 2
}
```

규칙:

- `FAILED` 또는 `RETRY_WAIT` 상태에서만 허용
- 이미 `SUCCEEDED`면 재시도 불가
- retry 대상은 특정 `Settlement` row다
- 같은 `room_id`라도 `settlement_type`에 따라 별도 `Settlement`가 존재할 수 있으므로 기준 식별자는 `settlement_id`다
- 내부적으로는 지정된 기존 `Settlement`를 다시 claim한다
- partial 상태에서는 기존 `point_history`와 payload가 일치하면 재사용해 FK만 보정하고, 미지급 participant만 새로 지급한다
- retry는 current-engine recalculation이나 payout rewrite가 아니라 기존 authoritative snapshot의 unfinished execution completion이다

## 15. 후속 이벤트

정산 커밋 이후에만 아래 후속 작업을 수행한다.

1. Android FCM/in-app notification hint 발송
2. 정산 완료 이메일
3. AI 습관 리포트 생성
4. 운영 모니터링 지표 적재

원칙:

- 후속 작업 실패는 정산 성공을 되돌리지 않는다.
- 따라서 정산 트랜잭션 밖에서 `SettlementCompleted` 이벤트를 소비하게 한다.
- 정산 완료 이메일/FCM notification 실패는 `Settlement.status`, `settlement_item`, `point_history`, 결제 충전 원장을 수정하거나 롤백하지 않는다.
- 이메일 발송은 SMTP 기반 best-effort 후속 작업이고, FCM 발송은 Android push transport 후속 작업이다.
- notification event/log, inbox/read, delivery attempt log는 UX/transport observability 후보로 둘 수 있지만 settlement evidence, audit-grade 정산 이력, outbox authority가 아니다.
- notification retry는 FCM delivery attempt recovery로만 제한하고 settlement retry/replay/correction 또는 payout mutation으로 연결하지 않는다.
- 이메일 실패는 structured log, bounded retry, 운영자 수동 재발송 대상으로만 다룬다. notification delivery attempt 실패는 token/device invalidation 또는 transport retry 판단에만 사용한다.
- structured email log는 최소 `settlement_id`, `member_id`, `email_type`, `recipient_hash`, `attempt`, `result`, `smtp_error_code`, `created_at`을 포함한다. notification delivery attempt log 후보는 settlement/ledger truth를 복제하지 않고 refetch metadata와 transport result만 남긴다.

## 16. 골든 데이터 예시

### 예시 A. 일반 정산

조건:

- 참여자 5명
- 참여자 A가 이 방의 host
- 각 예치금 100,000원
- 총 예치금 500,000원
- 성공 횟수: `A=90`, `B=75`, `C=75`, `D=75`, `E=75`

계산:

```text
전체 성공 합계 = 390
A raw = 500000 × 90 / 390 = 115384.615...
A base = 115384
나머지 4명 raw = 500000 × 75 / 390 = 96153.846...
나머지 4명 base = 96153
base 합계 = 115384 + 96153×4 = 499996
remainder = 4
deterministic remainder allocation rule에 따라 replayable calculation context에 remainder 4원이 설명상 배정됨. 이는 host reward/authority/privilege나 winner payout이 아님
```

최종:

| 참여자 | 최종 환급금 |
| ------ | ----------- |
| A      | 115,388     |
| B      | 96,153      |
| C      | 96,153      |
| D      | 96,153      |
| E      | 96,153      |

### 예시 B. 전체 인정 성공 0회

조건:

- 참여자 3명
- 각 예치금 100,000원
- 총 예치금 300,000원
- 성공 횟수: `0, 0, 0`

계산:

```text
for each participant:
  base_refund_amount = deposit_amount = 100000
  final_amount = deposit_amount = 100000
  remainder_bonus_amount = 0
total_remainder_amount = 0
```

최종:

- 전원 `100,000원`
- 이 all-fail 분기에서는 각자 자기 보증금을 그대로 돌려받으므로 잔액/보너스/추가 환급이 생기지 않는다. remainder allocation, winner/draw payout을 적용하지 않는다.

### 예시 C. 동점 표시 / 설명 ordering

조건:

- 참여자 4명
- 총 예치금 40,000원
- 성공 횟수: `A=5`, `B=5`, `C=3`, `D=1`

결정:

- A와 B가 동일 성공 수라도 payout remainder recipient는 동점자 중 draw/winner/top contributor로 결정하지 않는다.
- 원단위 절사 잔액은 deterministic remainder allocation rule로 처리한다.
- 동점자 표시나 대표 로그 설명 순서가 필요하면 payout과 분리된 stable ordering을 사용한다.

## 17. 대안 비교

### 선택한 안

`정산 스냅샷 테이블 + Settlement 선생성 + Settlement 단위 claim + PointHistory 원장`

장점:

- 어떤 방이 아직 정산되지 않았는지 명시적으로 남는다.
- 재시도와 중복 지급 방지가 쉽다.
- 왜 이 금액이 나왔는지 설명 가능하다.
- 골든 데이터 테스트와 장애 복구 절차를 만들기 쉽다.

단점:

- 테이블 수와 상태 관리 포인트가 늘어난다.
- 종료 감지 시 `PENDING` 생성 책임이 추가된다.

### 대안

`MissionLog를 직접 집계하고 PointHistory만 쓰는 안`

장점:

- 구현이 빠르다.
- 테이블이 적다.

단점:

- 사후 설명, 운영 디버깅, 재현이 어렵다.
- 어떤 방이 정산 대기인지 데이터만으로 구분하기 어렵다.
- 동일 방 재시도 시 중복 반영 리스크가 커진다.

결론:

- MVP라도 돈이 오가는 규칙은 `설명 가능성`, `재현 가능성`, `재시도 가능성`이 우선이므로 스냅샷 테이블 + 선생성된 `Settlement` 구조를 채택한다.

## 18. 테스트 시나리오

### 18.0 Lifecycle / activation 테스트

- `TS-LC-01` 시스템 자동 activation 성공
  기대 결과: `start_at`에 `LOCKED` participant 수가 `min_participants` 이상이고 terminal action이 없으면 시스템이 `ACTIVE`로 전이하고 `activated_at = start_at`이 기록된다.
- `TS-LC-02` `min_participants` 미달 자동 activation 실패
  기대 결과: `start_at` 평가 시점의 `LOCKED` participant 수가 `min_participants` 미만이면 `ACTIVE`로 전이하지 않고 취소형 정산 대상으로 남는다.
- `TS-LC-03` host manual start 비권한
  기대 결과: host는 `ACTIVE` 전이를 직접 만들 수 없고, StartRoom brownfield endpoint/command는 MVP active contract가 아니다.
- `TS-LC-04` 시작 만료 취소 settlement 멱등성
  기대 결과: `RECRUITING -> CANCELLED` batch가 성공하면 `CANCELLED_BEFORE_START` settlement가 1회만 생성되고 재시도해도 중복 환급되지 않는다.
- `TS-LC-05` 자동 activation과 취소 batch 경합
  기대 결과: 하나의 시스템 조건부 전이만 성공하고 loser는 최종 상태를 재조회한다.
- `TS-LC-06` 인증/log eligibility anchor
  기대 결과: `MissionLog.server_time < room.activated_at`이면 `BEFORE_START` 또는 동등 사유로 정산 인정에서 제외한다.
- `TS-LC-07` delayed scheduler reconciliation
  기대 결과: activation/settlement/cancel job이 늦게 실행되어도 판정은 scheduled semantic anchor(`start_at`, room timezone, daily cutoff, mission period end) 기준으로 수행되고, 실제 실행 지연은 audit/recovery log로만 남는다.
- `TS-LC-08` replay semantic context preservation
  기대 결과: algorithm version, cadence/timezone/cutoff interpretation, lifecycle cutoff, effective moderation state, moderation chain reference, reason-code mapping version으로 당시 semantic truth를 설명할 수 있으며 replay가 payout rewrite로 이어지지 않는다.

### 18.1 단위 테스트

- `TS-01` `min_participants`가 방별로 적용되는지
  기대 결과: 방 생성 시 host가 값을 설정할 수 있고, 설정하지 않으면 기본값 `2`가 저장된다.
- `TS-02` `min_participants > max_participants`이면 생성 실패
  기대 결과: `2 <= min_participants <= max_participants <= 15` 검증을 통과하지 못하면 요청이 reject된다.
- `TS-03` participant lifecycle baseline
  기대 결과: `PENDING`, `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`가 구분되고, capacity reservation에는 `PENDING + LOCKED`, activation/minimum/frozen baseline 및 settlement eligibility에는 `LOCKED`만 포함된다. 별도 중간 상태는 두지 않는다.
- `TS-03A` 신청 시 예치금 reserve 처리
  기대 결과: 신청 생성 transaction 내에서 `point_account.balance`가 보증금만큼 감소하고, 같은 금액이 `crew_participant.deposit_amount`에 snapshot되며 `CREW_DEPOSIT_LOCK` 원장이 append되고 `crew_participant.status`가 `PENDING`으로 기록된다.
- `TS-03B` `PENDING` 단계 경계
  기대 결과: `PENDING`은 capacity reservation과 reserved balance projection에는 포함되지만 activation eligibility/minimum baseline/frozen participant baseline/settlement eligibility에는 포함되지 않는다.
- `TS-03C` reserve 시 사용 가능 잔액 음수 방지
  기대 결과: 신청 생성 transaction 내 `point_account.balance >= deposit_amount` 조건부 update의 row count가 `1`일 때만 reserve가 성공하고, 동시 신청 또는 잔액 부족으로 row count가 `0`이면 신청 transaction 전체가 rollback된다.
- `TS-03D` 승인 전 신청 취소/거절/만료 환급
  기대 결과: `PENDING` 신청자는 `DELETE /api/crews/{crewId}/participants/me`로 직접 취소할 수 있고 상태가 `CANCELLED`로 전이되며 reserve 금액이 취소 환급 원장으로 반환된다. `REJECTED`/`EXPIRED`도 동일하게 reserve를 반환하고, `LOCKED` 이후에는 취소가 reject된다.
- `TS-04` non-LOCKED 인증 요청 차단
  기대 결과: `participant.status != LOCKED`이면 인증 API가 reject된다.
- `TS-04A` `ACTIVE` 이후 신규 참여 불가
  기대 결과: `MissionRoom.status != RECRUITING`이면 신규 참여 요청이 reject된다.
- `TS-04B` ACTIVE 이후 baseline 변경 불가
  기대 결과: `ACTIVE` 이후 신규 참여/재참여/상태 소급 변경은 frozen participant baseline을 바꾸지 않는다.
- `TS-05` post-freeze certification mutation 차단
  기대 결과: freeze 이후 host/admin은 resolved certification state를 소급 변경해 `recognized_success_count`를 바꾸지 못한다.
- `TS-06` DAILY 하루 다중 인증은 1회만 인정
  기대 결과: 같은 날짜 성공 로그가 3건이어도 인정은 1건이고 제외 사유가 저장된다.
- `TS-07` SPECIFIC_DAYS 유효 요일 외 성공 로그 제외
  기대 결과: 스케줄에 없는 요일의 성공 로그는 제외되고 `INVALID_SCHEDULE_DAY` 근거가 저장된다.
- `TS-07A` 인증 시점 실패 사유와 정산 시점 제외 사유가 분리되는지
  기대 결과: `BEFORE_START`, `AFTER_END` 같은 1차 실패는 `mission_log.failure_reason`에 남고, `DAILY` 중복 제외나 `SPECIFIC_DAYS` 비유효 요일 제외는 `settlement_item.calculation_reason`에만 남는다. `AFTER_WITHDRAWN`/`WEEKLY_N`은 brownfield/deferred reference다.
- `TS-08` WEEKLY_N deferred guard
  기대 결과: `WEEKLY_N`은 MVP active strategy로 선택되지 않고 Phase 2/deferred로 표시된다.
- `TS-08A` `frequency_type`별 recognition strategy가 올바르게 선택되는지
  기대 결과: MVP에서는 `DAILY`, `SPECIFIC_DAYS` 각각에서 대응 전략 1개만 선택되고, `recognized_success_count`, `recognized_dates_count`, `excluded_success_count`, `calculation_reason` 출력 계약이 동일하게 유지된다.
- `TS-09` 전체 성공 0회 시 equal-principal refund 적용
  기대 결과: 모든 참여자의 `base_refund_amount`, `final_amount`, `refund_amount`가 각자의 `deposit_amount`와 같고 별도 차감 규칙이 적용되지 않는다.
- `TS-10` 전체 성공 0회에서 remainder/winner/draw 추가 지급이 없는지
  기대 결과: 모든 참여자의 `final_amount`가 각자 `deposit_amount`와 같고, `remainder_bonus_amount = 0`, `reward_amount = 0`, `total_remainder_amount = 0`으로 고정된다.
- `TS-11` 참여자별 보증금이 다른 경우
  기대 결과: `total_locked_amount`는 합산되고 각 참여자의 `deposit_amount` 대비 `final_amount`가 일관되게 계산된다.
- `TS-11A` `total_participants`가 frozen `LOCKED` baseline 기준으로 계산되는지
  기대 결과: activation 시점 frozen `LOCKED` participant baseline이 `total_participants`와 `settlement_item` 생성 대상의 기준이다.
- `TS-11B` 동일 `member`가 여러 방에 참여한 경우 보증금 잠금이 방별로 분리되는지
  기대 결과: 각 방의 `crew_participant.deposit_amount`와 관련 `point_history`가 독립적으로 기록되고, 한 방의 정산이 다른 방 잠금 금액을 변경하지 않는다.
- `TS-11C` `total_locked_amount`가 participant 잠금 스냅샷 기준으로 계산되는지
  기대 결과: 정산 실행 시점의 정산 대상 participant `deposit_amount` 합계가 그대로 `total_locked_amount`에 저장되고, `point_history`나 `point_account` 재합산값은 사용되지 않는다.
- `TS-12` 일반 정산 로직이 영향받지 않았는지
  기대 결과: `전체 인정 성공 횟수 > 0`인 경우 지분율 기반 정산은 유지되고 원단위 절사 잔액은 deterministic remainder allocation rule로 처리된다. 이는 host discretion이나 winner payout이 아니다.
- `TS-13` non-payout ordering 재현성
  기대 결과: 같은 stable input 집합이면 대표/동점 표시 순서는 재실행해도 동일하지만, 이 ordering은 지급액을 변경하지 않는다.

### 18.2 통합 테스트

- `TS-14` 방 1개 정산 시 participant 단위 지급 트랜잭션에서 `point_history`와 `point_account.balance`가 함께 반영되고, 전체 정산은 partial 복구 가능 상태로 관리되는지
- `TS-14A` `settlement_item`이 먼저 생성되고 이후 `point_history`가 연결되는지
  기대 결과: 계산 스냅샷 없이 원장만 단독 생성되지 않고, `SUCCEEDED` 상태에서는 모든 `settlement_item.point_history_id`가 채워진다.
- `TS-14AA` `SUCCEEDED` 전환 전에 모든 환급 원장 연결이 검증되는지
  기대 결과: `point_history` 생성 또는 `point_history_id` 연결이 하나라도 누락되면 `Settlement.status`는 `SUCCEEDED`로 바뀌지 않고 `RETRY_WAIT` 또는 `FAILED`로 남는다.
- `TS-14B` 정산 환급 후 사용 가능 잔액이 증가하는지
  기대 결과: `CREW_SETTLEMENT_REFUND` 또는 `CREW_CANCELLED_REFUND` 기록과 함께 `point_account.balance`가 `final_amount`만큼 증가한다.
- `TS-14C` 포인트 원장 기록이 충전, 잠금, 환급 흐름과 일치하는지
  기대 결과: 같은 `member` 기준으로 `POINT_CHARGE -> CREW_DEPOSIT_LOCK -> CREW_SETTLEMENT_REFUND` 또는 `CREW_CANCELLED_REFUND` 순서의 잔액 변화가 `balance_after`와 함께 일관되게 남는다.
- `TS-14D` `point_account.balance`와 `point_history` 재계산값이 불일치할 때 원장 기준으로 복구되는지
  기대 결과: 운영 검증은 `point_history`를 source of truth로 삼아 불일치 원인을 기록하고, `point_account.balance` 캐시를 원장 재계산값으로 보정한다.
- `TS-15` 배치 재시도 시 `point_history.idempotency_key`로 중복 지급이 차단되는지
  기대 결과: 같은 `room_id`, `settlement_type`, `participant_id` 입력이면 같은 `idempotency_key`가 재사용되고, 동일 payload duplicate는 기존 원장 재사용/연결로 수렴하며, 다른 payload duplicate는 idempotency conflict로 실패한다.
- `TS-15A` `point_history`는 존재하지만 `settlement_item.point_history_id`만 누락된 partial 상태를 안전하게 복구하는지
  기대 결과: 해당 participant는 이미 지급 완료로 간주되고 새 환급 원장은 생성되지 않으며, 관리자 API 또는 배치가 기존 `point_history`를 조회해 FK만 연결한 뒤 전체 연결이 완료되면 `Settlement.status`가 `SUCCEEDED`로 전이된다.
- `TS-16` 취소형 정산과 일반 정산이 같은 조회 API 구조로 반환되는지
- `TS-17` 종료/취소 감지 시 `Settlement(PENDING)`가 먼저 생성되는지
- `TS-17A` `unique(crew_id, member_id)` 제약이 같은 크루 중복 participant row 생성을 막는지
  기대 결과: 동일 `member`가 같은 `crew`에 두 번째 `crew_participant` row 생성을 시도하면 DB 제약 또는 동일 수준의 저장 전 검증으로 차단된다.
- `TS-17A1` terminal 상태 재신청 차단
  기대 결과: 동일 `member`가 같은 `crew`에 `REJECTED` / `CANCELLED` / `EXPIRED` 상태 row를 보유한 상태에서 `POST /api/crews/{crewId}/participants`를 재호출하면 `APPLICATION_NOT_ALLOWED`로 reject되고 기존 row의 status는 변경되지 않으며 신규 row도 생성되지 않는다.
- `TS-17B` 실시간 대시보드 캐시와 `SUCCEEDED` 전 정산 계산 결과가 일시적으로 달라도 authoritative settlement input으로 정산값이 확정되는지
  기대 결과: 캐시 누락 또는 지연이 있어도 `settlement_item` 계산값은 `MissionLog` 원본, frozen `LOCKED` baseline, resolved certification state 기준으로 일관되게 생성된다.

### 18.3 동시성 테스트

- `TS-17C` 동시 인증 업로드 시 `MissionLog` append-only 저장이 보존되는지
  기대 결과: 같은 시점 인증 요청이 겹쳐도 로그는 유실 없이 저장되고, 실시간 지표는 일부 지연될 수 있지만 최종 정산 결과는 변하지 않는다.
- `TS-18` 같은 방에 대해 동시 정산 요청이 들어온 경우 조건부 claim에서 1개 워커만 성공하는지
  기대 결과: 같은 `Settlement`에 대해 동시 실행 요청이 들어와도 조건부 claim에서 하나의 실행자만 성공하고 나머지는 skip된다.
- `TS-20` `unique(room_id, settlement_type)` 제약이 중복 생성 시도를 막는지

### 18.4 운영 검증

- `TS-21` 실패 코드별 `RETRY_WAIT` / `FAILED` 전이가 기대대로 기록되는지
- `TS-22` `FAILED` 건이 관리자 API에서 조회되는지
- `TS-23` `RUNNING` timeout 건을 운영 배치가 `RETRY_WAIT`로 복구할 수 있는지
- `TS-23A` 종료 감지 누락 시 운영 복구로 `Settlement(PENDING)`를 생성할 수 있는지
  기대 결과: `room_id` 기준 정산 대상 여부 재검사 후 누락된 정산만 복구 생성되고, `unique(room_id, settlement_type)` 제약 위반 시 중복 생성되지 않는다.

## 19. 구현 전 비권위 체크포인트

이 섹션은 implementation sequence나 stack topology를 freeze하지 않는다. 구현 계획은 별도 architecture/implementation decision에서 다루며, 이 문서는 아래 semantic guardrail만 유지한다.

1. `settlement`, `settlement_item`, `point_history`가 정산 snapshot, item linkage, ledger idempotency를 설명할 수 있어야 한다.
2. `SettlementPendingCreator`, `SettlementInputAssembler`, `MissionRecognitionStrategy`, `SettlementCalculator` 책임은 계산 규칙·입력 조립·실행 상태가 뒤섞이지 않도록 분리되어야 한다.
3. `TS-01`~`TS-09`와 `TS-08A` 골든/단위 테스트는 semantic invariant를 먼저 검증해야 한다.
4. Batch infrastructure와 trigger topology는 이 문서가 확정하지 않는다. MVP semantic guardrail은 조건부 claim(DB conditional claim), duplicate payout 방지, participant 단위 idempotency, point_history 방어선이다. 분산 실행 조정 전략은 MVP requirement나 optional runtime path가 아니며 별도 Phase 2 scale hardening 후보로만 다룬다.
