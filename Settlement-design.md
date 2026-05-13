# 정산 설계: 갓세이빙

기준 문서:

- [PRD-god-saving.md](./PRD-god-saving.md)
- [MVP-backlog-user-stories.md](./MVP-backlog-user-stories.md)
- [MVP-ticket-breakdown.md](./MVP-ticket-breakdown.md)
- `docs/갓세이빙_프로젝트기획안.docx` (활성 제품/UX 참고 자료; 기존 `갓세이빙_프로젝트기획서.docx` 명칭이나 사본은 legacy/reference 입력)

## 1. 목적

이 문서는 `T-17 정산 규칙 엔진`, `T-18 정산 배치`, `T-22 골든 데이터 테스트`를 구현하기 전에 정산 도메인의 기술 설계를 고정하기 위한 문서다.

이번 개정의 목표는 정산 로직 자체를 갈아엎는 것이 아니라, 아래 운영 요구를 만족하도록 MVP 설계를 보강하는 것이다.

1. 같은 입력이면 언제 다시 계산해도 같은 결과가 나와야 한다.
2. 포인트 환급은 participant 단위 deterministic idempotency key로 지급하며, 배치 재시도나 중복 실행이 있어도 중복 반영되면 안 된다.
3. 운영자는 장애 후 재시도, 실패 분석, 분쟁 대응을 문서와 데이터만으로 수행할 수 있어야 한다.
4. 사용자는 "왜 이 금액을 받았는지"를 나중에도 설명받을 수 있어야 한다.

## 2. 범위와 비범위

### 범위

- 미션 종료 후 배치 기반 최종 정산
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

## 3. 고정할 비즈니스 규칙

### 3.1 시간 기준

- MVP의 정산 기준 시간대는 `Asia/Seoul`로 고정한다.
- 사용자에게 보이는 미션 종료 시점은 `종료일 23:59:59 KST`다.
- `recruitment_deadline`은 신규 참여 마감 시각이고, activation/settlement 기준 시각이 아니다.
- `start_at`은 예정 시작 시각이자 MVP에서 host 수동 시작 가능 latest deadline이다. `start_at` 이후에도 `RECRUITING`이면 시작 만료 취소 대상이다.
- `activated_at`은 실제 `RECRUITING -> ACTIVE` 전이 시각이며, 인증 가능 여부, 정산 인정, projection/log eligibility의 actual anchor다.
- `end_at`은 계획된 미션 종료 cutoff로 유지하며, activation 지연으로 자동 연장하지 않는다. 실제 수행 가능 기간이 짧아질 수 있지만, MVP에서는 단순한 운영 정책, deterministic settlement, replay consistency를 우선한 의도적 trade-off다.
- 따라서 성공한 activation은 운영상 `activated_at <= start_at`을 만족한다. `StartRoom`은 `start_at` 이후 실패하고, `start_at` 이후에도 `RECRUITING`인 방은 취소 batch 대상이기 때문이다. 이는 구현/운영 판단용 MVP invariant이며, 별도 DB constraint 추가를 의미하지 않는다.
- 배치 스케줄러는 `익일 00:05 KST`부터 정산을 시작한다.
- 이 `5분` 버퍼는 마지막 인증 저장, 로그 커밋, 시계 오차 흡수를 위한 구현 결정이다.
- 목표 SLA는 `00:30 KST` 이내 정산 완료다.

### 3.2 정산 대상 참여자

- 정산 대상 참여자는 `종료 시점까지 locked deposit이 존재하는 모든 참여자`다.
- 한 `member`는 하나의 `MissionRoom`에 대해 하나의 `participant`만 가진다.
- 이 불변식은 `unique(room_id, member_id)`로 강제하고, `participant`는 정산과 감사 추적을 위해 물리 삭제하지 않는다.
- `participant` 생명주기는 `JOINED`, `WITHDRAWN` 같은 status 기반으로만 관리한다.
- 사용자는 `MissionRoom.status = ACTIVE` 상태에서도 탈퇴할 수 있다.
- 중도 탈퇴자도 정산 대상에서 제외되지 않는다.
- 탈퇴자는 `withdrawn_at` 이전에 성공한 인증만 인정된다.
- `participant.status == WITHDRAWN`이면 이후 인증 요청은 서버에서 reject한다.
- 정산 시에는 `server_time < withdrawn_at` 조건으로 다시 필터링한다.
- 탈퇴 후에는 추가 인증이 불가하고, 탈퇴 직후 보증금을 즉시 환급하지 않는다.
- 탈퇴자의 보증금은 최종 정산 시점까지 풀에 남는다.
- 탈퇴는 `미래 기여를 중단하는 이벤트`이며, 과거 기여는 그대로 인정한다.
- 신규 참여는 `MissionRoom.status = RECRUITING`이고 서버 시간이 `recruitment_deadline` 전일 때만 허용한다.
- `ACTIVE` 이후 신규 참여는 허용하지 않고, 탈퇴 후 동일 방 재참여도 MVP에서는 지원하지 않는다.
- `StartRoom` 성공 시점의 eligible participant 집합을 MVP 정산 기준 ACTIVE participant baseline으로 본다. 별도 snapshot/versioning table은 MVP에서 추가하지 않고, `activated_at`과 participant 상태 이력으로 replay한다.

### 3.3 `min_participants` 정책

- `min_participants`는 `MissionRoom`별 설정값으로 관리한다.
- 방 생성 시 host가 설정할 수 있고, 기본값은 `2`명이다.
- 제약 조건은 `2 <= min_participants <= max_participants <= 10`이다.
- MVP에서 `min_participants` 충족은 자동 시작 트리거가 아니라 `StartRoom` command의 precondition이다.
- `StartRoom` 실행 시점에 eligible participant 수를 다시 검증하며, 미달이면 `ACTIVE`로 전이하지 않는다.

### 3.4 시작 만료 / 인원 미달 취소

- `recruitment_deadline` 이후 신규 참여는 차단한다.
- `min_participants`를 충족한 방도 host가 `start_at`까지 `StartRoom`을 성공시키지 않으면 시작되지 않는다.
- `start_at` 이후에도 `RECRUITING`인 방은 batch가 `CANCELLED` 처리한다.
- 시작 만료 또는 인원 미달 취소는 일반 정산과 별개가 아니라 `취소형 정산`으로 기록한다.
- 취소형 정산에서는 각 참여자에게 `잠긴 보증금 전액`을 환급한다.

### 3.5 정산 계산 입력과 성공 후 운영 원천

- 실시간 대시보드는 캐시나 역정규화 테이블을 사용해도 된다.
- 그러나 `Settlement.status = SUCCEEDED` 전 최종 정산 금액 계산은 반드시 `MissionLog`와 참여자 상태를 다시 읽어서 수행한다.
- `Settlement.status = SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다. 이후 `MissionLog` 재계산은 감사/디버깅 검증용이지 지급 결과를 대체하는 기준이 아니다.
- `Mission_Room_Stat` 같은 캐시성 테이블은 정산 금액 계산의 근거로 사용하지 않는다.

### 3.6 금액 단위

- 포인트와 환급금은 모두 `원 단위 정수`로 저장하고 노출한다.
- DB 금액 컬럼은 `BIGINT` 또는 `DECIMAL(18,0)` 등 정수 표현을 사용한다.
- Java 계산은 `BigDecimal`과 `MathContext.DECIMAL128`을 사용한다.
- 최종 지급액은 `RoundingMode.FLOOR`로 절사한다.

### 3.7 재현 가능성 원칙

- 정산 결과는 `MissionLog`, 참여자 스냅샷, 고정된 정산 규칙만으로 재현 가능해야 한다.
- 재현 가능성은 `정산 배치 실행 시각`, `settlement.id`, 런타임 랜덤에 의존하면 안 된다.
- 외부 결제 상태나 후속 이벤트 성공 여부는 정산 계산 결과의 입력값이 아니다.

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
- `member`는 사용자 식별·인증 책임만 가지며, `point_account.balance`는 현재 사용 가능 포인트 잔액 캐시만 담당한다.

## 5. 정산 상태 흐름

### 5.1 MissionRoom 상태

| 값           | 의미         |
| ------------ | ------------ |
| `RECRUITING` | 모집 중      |
| `ACTIVE`     | 진행 중      |
| `CLOSED`     | 정상 종료    |
| `CANCELLED`  | 시작 전 취소 |

기본 흐름:

- `RECRUITING -> ACTIVE`: host의 `StartRoom` command가 조건부 전이에 성공할 때만 발생한다.
- `RECRUITING -> CANCELLED`: `start_at` 만료 후에도 미시작 상태이거나 기존 취소 정책이 조건부 전이에 성공할 때 발생한다.
- `ACTIVE -> CLOSED`: 계획된 `end_at` cutoff 이후 정상 종료 처리로 발생한다.

`StartRoom`과 시작 만료 취소 batch는 모두 `RECRUITING` 상태를 조건으로 하는 전이다. 동시에 경합하면 하나만 성공하고, loser는 최종 room 상태를 재조회한다. 취소형 settlement 생성은 unique/idempotent해야 한다.

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
- 일부 participant 지급만 완료됐거나 원장-FK 연결이 누락된 partial 상태는 복구 가능한 중간 상태이며, `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`로 남긴다.

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

- 종료 감지 로직이 실패해 `Settlement`가 생성되지 않은 경우를 대비해, 운영용 재조정 잡 또는 관리자 API로 `PENDING` 생성 보정을 할 수 있다.
- 보정 절차는 `room_id` 기준으로 `CLOSED` 또는 `CANCELLED` 상태와 정산 대상 participant 존재 여부를 다시 확인한 뒤, 누락된 `Settlement(PENDING)`를 생성하는 흐름으로 고정한다.
- 이 보정 경로도 동일하게 `unique(room_id, settlement_type)` 제약을 통과해야 한다.

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
3. `lock:settlement:room:{roomId}` Redis/Redisson 락 시도
4. 정산 입력 데이터 로드
5. 정산 계산 수행
6. `settlement` 집계값과 `settlement_item` 스냅샷 저장
7. 참여자별 `point_history`를 `idempotency_key`와 함께 생성
8. 생성된 `point_history.id`를 각 `settlement_item.point_history_id`에 연결
9. 모든 `settlement_item`에 대응하는 `point_history`가 존재하고 `point_history_id`가 채워졌는지 검증
10. 참여자/방 projection 갱신
11. 위 검증이 모두 통과한 경우에만 `Settlement.status = SUCCEEDED`로 종료
12. 커밋 후 `SettlementCompleted` 후속 이벤트 발행

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
- claim 후 Redis 락 획득에 실패하면 `LOCK_ACQUIRE_FAILED`로 기록하고 `RETRY_WAIT`로 되돌린다.
- Redis 락은 보조 안전장치고, 최종 claim 성공 여부는 DB 조건부 update가 결정한다.
- 계산 스냅샷인 `settlement_item`이 먼저 저장되고, 실제 잔액 원장인 `point_history`는 그 뒤에 생성되어 `point_history_id`로 연결된다.
- 정산 재시도 시 일부 participant의 `point_history`만 먼저 생성된 partial 상태를 허용한다. 이미 생성된 원장은 같은 `idempotency_key`로 중복 생성되지 않고, 아직 생성되지 않은 participant만 추가 반영한다.
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
| `total_participants`              | 정산 대상 participant 수, `WITHDRAWN`이어도 locked deposit이 남아 있으면 포함 |
| `total_locked_amount`             | 정산 실행 시점 기준 총 잠긴 보증금 스냅샷                                     |
| `total_recognized_success`        | 전체 인정 성공 횟수                                                           |
| `total_base_refund_amount`        | 절사 전 잔액 배분 전 합계                                                     |
| `total_remainder_amount`          | 잔액 총액                                                                     |
| `remainder_policy`                | `TOP_1_ALL`, `DRAW_SPLIT_ONE_WON`                                             |
| `remainder_winner_participant_id` | 일반 정산 잔액 귀속 대상                                                      |
| `failure_code`                    | 표준 실패 코드                                                                |
| `failure_message`                 | 최근 실패 원인 요약                                                           |
| `started_at`                      | 실행 시작 시각                                                                |
| `finished_at`                     | 실행 종료 시각                                                                |
| `created_at`                      | 생성 시각                                                                     |
| `updated_at`                      | 수정 시각                                                                     |

제약:

- `unique(room_id, settlement_type)`
- `status`는 정산 처리의 원천 상태다.
- `total_participants`는 `종료 시점까지 locked deposit이 존재하는 participant 수`를 의미한다.
- `WITHDRAWN` 참여자도 locked deposit이 남아 있으면 포함한다.
- `total_locked_amount`는 정산 실행 시점의 정산 대상 participant `room_participant.deposit_amount` 합계를 스냅샷으로 고정한 값이다.
- `total_locked_amount`는 `point_history`나 `point_account`를 다시 합산해 계산하지 않는다.
- MVP에서는 별도 `total_active_participants` 컬럼을 두지 않고, 필요 시 조회/분석용 후속 검토 항목으로 남긴다.

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
| `participant_status_snapshot` | `JOINED`, `WITHDRAWN` 등                                              |
| `deposit_amount`              | 잠긴 보증금 스냅샷                                                    |
| `success_count_raw`           | 기간 내 원시 성공 로그 수                                             |
| `recognized_success_count`    | 최종 인정 성공 횟수                                                   |
| `recognized_dates_count`      | 최종 인정된 날짜 수                                                   |
| `excluded_success_count`      | 제외된 성공 로그 수                                                   |
| `period_start_at`             | 계산 기간 시작                                                        |
| `period_end_at`               | 계산 기간 종료                                                        |
| `withdrawn_at_snapshot`       | 탈퇴 시각 스냅샷                                                      |
| `share_ratio`                 | 최종 지분율                                                           |
| `raw_refund_amount`           | 절사 전 계산 금액                                                     |
| `base_refund_amount`          | `FLOOR` 적용 금액                                                     |
| `remainder_bonus_amount`      | 잔액 가산분                                                           |
| `reward_amount`               | 잠긴 보증금 대비 초과 환급분, `max(final_amount - deposit_amount, 0)` |
| `refund_amount`               | 실제 환급 총액, MVP에서는 `final_amount`와 동일                       |
| `final_amount`                | 최종 지급 금액                                                        |
| `draw_key_snapshot`           | tie-break 계산에 사용한 키                                            |
| `tie_break_rank`              | draw 정렬 순위                                                        |
| `calculation_reason`          | 포함/제외 근거 JSON 또는 TEXT                                         |
| `point_history_id`            | 환급 원장 FK                                                          |
| `created_at`                  | 생성 시각                                                             |

설계 원칙:

- `settlement_item`은 결과뿐 아니라 계산 근거까지 저장해야 한다.
- `settlement_item`은 참여자별 정산 계산 결과의 source of truth고, `point_history`는 그 결과를 계정 잔액에 반영하는 금액 source of truth다. `Settlement.status = SUCCEEDED` 이후 두 테이블이 운영/분쟁/조회 기준이다.
- `deposit_amount`는 participant 단위로 잠겨 있던 보증금의 입력 스냅샷이며, 실제 잔액 반영은 `point_history`가 담당한다.
- `calculation_reason`은 `DAILY` 중복 제외, `SPECIFIC_DAYS` 비유효 요일 제외, `WEEKLY_N` 상한 제외, `withdrawn_at` cutoff 적용을 설명할 수 있어야 한다.
- `calculation_reason`의 `AFTER_WITHDRAWN_AT` 같은 값은 정산 계산 스냅샷 key이며, public `MissionLogFailureReason` enum(`AFTER_WITHDRAWN` 등)과 동일 책임이 아니다.
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
| `WEEKLY_N_OVERFLOW` | `WEEKLY_N` 규칙에서 주간 허용 횟수 초과분 제외 |
| `AFTER_WITHDRAWN_AT` | `withdrawn_at` 이후 성공 로그 제외 |
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
| `transaction_type` | `POINT_CHARGE`, `ROOM_DEPOSIT_LOCK`, `ROOM_SETTLEMENT_REFUND`, `ROOM_CANCELLED_REFUND` |
| `reference_type`   | 예: `SETTLEMENT_ITEM`                                                                  |
| `reference_id`     | 참조 엔티티 PK                                                                         |
| `idempotency_key`  | 중복 반영 방지 키, 항상 `NOT NULL`                                                     |
| `created_at`       | 생성 시각                                                                              |

원칙:

- 모든 포인트 변경은 반드시 `point_history`를 통해서만 발생한다.
- `point_history`는 항상 `member_id` 기준으로 기록되며, 정산 계산 결과를 실제 계정 잔액 변화로 반영하는 금액 source of truth다.
- `PointAccount` 또는 `MemberPoint` 같은 현재 잔액 테이블이 있다면, 이 값은 항상 `사용 가능한 포인트 잔액`만 나타내는 재계산 가능한 캐시다.
- MVP에서는 현재 잔액 캐시에 `pending_balance`, `waiting_balance`, `locked_balance` 같은 대기·잠금 상태 컬럼을 분리하지 않는다.
- 현재 잔액 캐시와 `point_history` 원장 재계산값이 다르면 `point_history`를 source of truth로 삼고, 원인 조사 후 잔액 캐시를 보정하거나 재생성한다.
- 보증금은 별도의 계좌로 이동하지 않고, 참여 시점에 `point_account.balance`에서 차감되어 `room_participant.deposit_amount`로 잠긴 상태로 관리된다.
- `ROOM_DEPOSIT_LOCK`는 자산 이동이 아니라 기존 포인트를 사용 불가 상태로 전환하는 이벤트다.
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

- 포인트 충전은 `point_account.balance`를 증가시키는 일반 잔액 충전이다.
- `point_account`는 `member`와 분리해 사용자 식별·인증 책임과 포인트 잔액 갱신 책임을 나누며, MVP에서는 `balance` 하나만 현재 사용 가능 잔액 캐시로 둔다.
- 크루 참여 시 보증금은 별도 자산으로 이동하지 않고, `point_account.balance`에서 차감되어 해당 `room_participant.deposit_amount`에 participant 단위 잠금 금액으로 기록된다.
- 보증금 잠금 상태는 `point_account.locked_balance`가 아니라 `balance` 차감과 `room_participant.deposit_amount` 기록으로 표현한다.
- 사용자에게 보여줄 `GET /api/points.locked_balance`는 정산 전 참여 보증금 합계를 API projection으로 제공할 수 있다.
- 이 projection은 UX 표시용이며 정산 계산, 포인트 원장, 출금 가능 여부, 환급 가능 여부, 분쟁 처리, 정산 결과 판단의 source of truth가 아니다.
- MVP projection은 `room_participant.deposit_amount`와 `mission_room.status IN ('RECRUITING', 'ACTIVE', 'CLOSED')`를 기준으로 시작하며, settlement 조인을 강제하지 않는다.
- `CLOSED` 포함은 정산 완료 전까지 잠겨 있을 것으로 기대되는 보증금 표시를 위한 근사값이다. `Settlement.status = SUCCEEDED` 이후 lock 해제 여부를 더 정확히 제외하는 조건은 Settlement 조회/정산 구현 단계에서 보강할 수 있다.
- 정산 계산의 입력 금액은 여전히 정산 대상 participant의 `room_participant.deposit_amount` 합계다.
- 보증금 잠금은 `point_account`에 대한 조건부 update로 수행한다. 즉, `WHERE balance >= deposit_amount` 조건을 포함해 잔액이 충분할 때만 차감한다.
- 이 update의 row count가 `1`일 때만 잠금 성공으로 간주하고, `0`이면 동시 요청 또는 잔액 부족으로 보고 참여를 실패 처리한다.
- 보증금 잠금 처리, `room_participant` 생성, `ROOM_DEPOSIT_LOCK` 원장 생성은 반드시 하나의 트랜잭션으로 처리한다.
- 권장 순서는 `point_account.balance` 조건부 차감 -> `room_participant` 생성 및 `deposit_amount` 반영 -> `ROOM_DEPOSIT_LOCK point_history` 생성이다.
- 위 세 단계 중 하나라도 실패하면 전체 롤백한다. 잔액만 차감되고 participant가 생성되지 않거나, participant만 생기고 원장이 누락되는 상태를 허용하지 않는다.
- 중도 탈퇴가 발생해도 `deposit_amount`는 정산 시점까지 유지되며, 즉시 환급되지 않는다.
- 최종 정산 또는 취소 환급이 일어날 때만 `point_history`를 통해 `balance`가 증가한다.
- 운영 검증이나 복구 중 `point_account.balance`가 `point_history` 기반 재계산값과 다르면 `point_history`를 기준으로 캐시를 복구한다.

## 8. 인정 성공 횟수 계산 규칙

### 8.1 공통 조건

인정 성공 횟수에 포함되려면 아래를 모두 만족해야 한다.

1. 해당 참여자의 로그여야 한다.
2. `is_success = true`여야 한다.
3. `server_time`이 방의 종료 시점 이전이어야 한다.
4. 참여자가 탈퇴했다면 `server_time < withdrawn_at`이어야 한다.

### 8.2 DAILY

- 기준: 미션 기간 동안 하루 최대 1회 인정
- 구현: 날짜별 성공 로그를 집계하고 같은 날 다중 성공은 1회만 인정
- 스냅샷: `recognized_dates_count`, `excluded_success_count`, `calculation_reason`에 중복 제외 근거 저장

### 8.3 SPECIFIC_DAYS

- 기준: `mission_schedule_day`에 정의된 반복 요일만 유효
- 구현: 성공 로그의 `server_time` 요일이 스케줄 테이블과 일치할 때만 인정
- 같은 유효 날짜에 다중 성공이 있어도 1회만 인정
- 스냅샷: 제외된 로그는 `INVALID_SCHEDULE_DAY`, `DAILY_DUPLICATE` 같은 코드로 남긴다

### 8.4 WEEKLY_N

- 기준: 실제 activation 이후에는 `room.activated_at`의 KST date를 기준으로 7일씩 끊은 주간 버킷마다 최대 `N회` 인정
- 구현: `week_index = floor(days_between(kst_date(room.activated_at), log_date) / 7) + 1`
- 각 `week_index`별 성공 개수 중 `min(success_count, frequency_count)`만 인정
- 스냅샷: 주차별 인정/제외 수를 `calculation_reason.weeklyBuckets`에 남긴다

### 8.5 인정 성공 계산 전략

- `frequency_type`별 인정 성공 계산은 `SettlementBatch`나 `PointHistory`가 아니라 별도 recognition strategy가 책임진다.
- 각 전략은 `MissionLog` 원본과 참여자 스냅샷을 입력으로 받아 아래 출력만 반환한다.
  - `recognized_success_count`
  - `recognized_dates_count`
  - `excluded_success_count`
  - `calculation_reason`
- `Settlement`의 생성, claim, 상태 전이, 포인트 반영 흐름은 유지하고, 빈도별 인정 규칙 차이는 전략 내부에서만 흡수한다.
- MVP에서는 `DAILY`, `SPECIFIC_DAYS`, `WEEKLY_N` 세 전략만 지원한다.
- 향후 연속 성공 보너스, 요일별 가중치 같은 정책이 필요해지더라도 `SettlementBatch`나 `PointHistory`를 수정하지 않고 전략 구현 추가로 확장한다.

### 8.6 왜 실시간 수치와 최종 수치가 달라질 수 있는가

- 실시간 대시보드는 빠른 추정값이며, `SUCCEEDED` 전 정산 계산은 `MissionLog`와 참여자 상태를 다시 읽어 확정한다.
- 실시간 대시보드는 캐시 반영 지연이 있을 수 있다.
- `SUCCEEDED` 전 정산 계산은 로그 원본, 탈퇴 시각, 주차 버킷 상한을 다시 계산한다.
- `mission_log.failure_reason`은 인증 요청 시점의 1차 실패 사유만 저장하고, `DAILY` 중복 제외나 `WEEKLY_N` 상한 제외 같은 정산 단계 설명은 `settlement_item.calculation_reason`이 맡는다.
- 정산 화면은 파생 표시일 뿐 source of truth가 아니다. `Settlement.status = SUCCEEDED` 이후 금액의 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다.

### 8.7 `failure_reason`과 `calculation_reason` 책임 분리

- `mission_log.failure_reason`은 인증 시점 실패 사유를 저장한다. 예: `EXIF_MISSING`, `BEFORE_START`, `AFTER_END`
- `settlement_item.calculation_reason`은 정산 시점 포함/제외 근거를 저장한다.
- 인증 자체가 실패한 경우에는 `mission_log.failure_reason`을 채우고, 필요하면 정산 스냅샷에도 같은 제외 근거를 남길 수 있다.
- 인증은 성공했지만 정산 규칙상 제외된 경우에는 `mission_log.failure_reason` 없이 `calculation_reason`에만 남긴다.
- 예를 들어 `DAILY` 중복은 `calculation_reason`만 기록하고, `EXIF` 오류처럼 인증 시점에 이미 실패한 건은 `failure_reason`과 `calculation_reason` 양쪽에 함께 남길 수 있다.

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
6. 일반 정산에서 절사 후 남은 잔액은 기여도 1위 참여자에게 지급한다.
7. 기여도 1위가 동점인 경우 먼저 성공 횟수를 비교하고, 그래도 동일하면 재현 가능한 draw 규칙으로 1명을 결정한다.

참고:

- 잠긴 보증금이 참여자마다 달라도 공식은 동일하다.
- `total_locked_amount`는 정산 실행 시점의 정산 대상 participant `deposit_amount` 합계 스냅샷이다.
- `total_locked_amount`는 `point_history`나 `point_account` 잔액을 다시 합산해 계산하지 않는다.
- 누가 본인 보증금보다 많이 또는 적게 돌려받았는지는 `deposit_amount`와 `final_amount` 비교로 설명한다.

### 9.3 전원 성공 0회 정산

`전체 인정 성공 횟수 = 0`이면 전원 균등 환급한다.

```text
equal_base = FLOOR(total_locked_amount / participant_count)
remainder = total_locked_amount - (equal_base × participant_count)
```

규칙:

- 모든 참여자의 `base_refund_amount = equal_base`
- 남은 `remainder`는 `deterministic draw` 순위 상위 참여자부터 `1원씩` 나눠준다
- 이 분기에서는 추가 차감 규칙을 두지 않는다.
- 이 분기에서는 기여도 1위가 존재하지 않으므로 일반 정산의 `TOP_1_ALL` 규칙을 사용하지 않는다

### 9.4 잔액 상한

- 참여자 수가 `n`명일 때 절사 후 잔액은 항상 `0 <= remainder < n`이다.
- MVP 최대 인원 `10명` 기준으로 남는 잔액은 `최대 9원`이다.
- 원문 기획안/legacy 기획서의 `1~10원`은 설명용 표현으로 보고, 구현 기준은 위 수학적 상한으로 고정한다.

### 9.5 deterministic draw 규칙

일반 정산의 동점자와 전체 인정 성공 0회 정산의 잔액 분배는 런타임 랜덤이 아니라 재현 가능한 의사 난수 규칙으로 처리한다.

권장 공식:

```text
draw_key = SHA-256(room_id + ":" + settlement_type + ":" + member_id)
정렬 기준: draw_key 오름차순
```

원칙:

- `settlement.id` 같은 실행 시점 생성 PK에 의존하지 않는다.
- `unique(room_id, member_id)` 제약과 `재참여 불가` 정책으로 같은 방의 한 `member`는 하나의 `participant`에만 대응하므로 `member_id` 기반 draw가 안전하다.
- 같은 `room_id`, `settlement_type`, `member_id` 입력이면 재시도, 테스트, 데이터 이관 이후에도 같은 결과가 나와야 한다.
- draw 결과는 `settlement_item.draw_key_snapshot`, `tie_break_rank`로 저장해 사후 설명 가능해야 한다.

## 10. 멱등성 및 동시성 제어

동시성 제어는 `인증 시점`과 `최종 정산 시점`을 분리해 다룬다. 인증은 빠른 기록 보존이 우선이고, 정산은 중복 실행 방지가 우선이다.

### 10.1 인증 시점 동시성 원칙

- 인증 업로드와 `MissionLog` 저장 시점에는 Redisson 락을 기본값으로 두지 않는다.
- 인증은 `MissionLog` append-only 저장을 우선해 원본 로그 유실 가능성을 낮춘다.
- 실시간 대시보드, 예상 지분율, 임시 랭킹은 Redis `INCR`, `SETNX`, Lua script 같은 원자적 캐시 연산으로 갱신할 수 있다.
- 위 실시간 지표는 사용자 안내용 추정값일 뿐이며, 최종 정산 금액의 source of truth가 아니다.
- 실시간 캐시가 일부 지연되거나 누락돼도 최종 정산은 반드시 `MissionLog` 원본을 다시 읽어 확정한다.
- 따라서 인증 시점에는 과도한 분산 락보다 `로그 보존 + 원자적 캐시 갱신 + 사후 재계산 가능성`을 우선한다.

### 10.2 정산 시점 방어선

- 최종 정산 시점에는 중복 정산/중복 지급 방지를 위해 여러 방어선을 함께 사용한다.
- 1차 실행권은 DB 조건부 claim이 결정하고, Redisson은 보조 수단이며, DB unique와 `point_history.idempotency_key`가 최종 방어선이다.
- 아래 10.3~10.6은 같은 방에 대해 배치 워커, 관리자 재시도, 복구 작업이 겹쳐도 결과가 participant 단위로 한 번만 확정되도록 하는 장치다.

### 10.3 상태 기반 claim

- 워커는 `PENDING` 또는 `RETRY_WAIT` 상태의 `Settlement`만 대상으로 한다.
- claim은 조건부 update로 수행한다.
- 같은 `Settlement`를 두 워커가 동시에 집어도 row count `1`을 받은 워커만 실행한다.

### 10.4 분산 락

- `lock:settlement:room:{roomId}` Redisson 락을 사용한다.
- 목적은 배치 워커와 관리자 재시도, 복구 스크립트가 같은 방을 동시에 잡지 못하게 하는 것이다.
- 락이 없어도 DB 조건부 update와 unique 제약이 최종 방어선이므로, 락은 보조 안전장치로 설명한다.
- 즉, Redisson은 `정산 커밋 경계`에서만 제한적으로 사용하고, 인증 로그 저장 전 구간까지 확장하지 않는다.

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
- `room_id`, `settlement_type`, `participant_id`처럼 입력 기반 식별자를 사용해 재시도, 재정산 테스트, 데이터 이관 상황에서도 같은 키가 재현되도록 한다.
- `participant`는 물리 삭제하지 않고 같은 방 재참여도 지원하지 않으므로, 같은 정산 대상에 대한 `participant_id`는 생명주기 동안 안정적으로 유지된다.
- 이 규칙은 `draw_key`와 같은 철학을 따른다. 즉, 런타임 생성값이 아니라 동일 입력이면 동일 결과가 나와야 한다.
- 동일 이벤트는 항상 동일한 `idempotency_key`를 사용한다.
- `POINT_CHARGE`의 API field `payment_id`는 TossPayments `paymentKey`를 의미하며, 하나의 충전 이벤트에만 1:1로 매핑되어야 한다.
- `orderId`는 confirm 검증과 로그 상관관계 추적용이며 `point_history.idempotency_key`에 사용하지 않는다.
- 동일한 `paymentKey`가 재사용되거나 중복 발급되면 `charge:{paymentKey}` 기반 멱등성이 깨지므로, 결제 연동 계층에서 이를 허용하지 않아야 한다.
- provider success 이후 client timeout이 발생해도 같은 `paymentKey` 재시도는 중복 충전이 아니라 기존 원장 재사용으로 수렴해야 한다.
- 재시도 중 중복 insert가 발생하면 unique 제약으로 차단된다.
- 애플리케이션은 동일 키 충돌을 먼저 기존 `point_history`와 요청 payload가 같은 semantic event인지 검증한다. 동일 payload면 기존 원장을 재사용/연결하고, 다른 payload면 idempotency conflict로 실패 처리한다.
- 애플리케이션은 이 충돌을 `이미 지급됨`으로 해석하고 무조건 성공 처리하지 말고, 현재 `Settlement.status`, `settlement_item.point_history_id`, `point_history` payload 일치 여부를 함께 검증해야 한다.
- 이 원칙은 정산 환급뿐 아니라 충전, 보증금 잠금 같은 모든 포인트 이벤트에도 동일하게 적용한다. 따라서 `point_history.idempotency_key`는 항상 `NOT NULL`이어야 한다.

## 11. 실패/재시도 정책

### 11.1 실패 코드 표준

| failure_code           | 의미                                         | 자동 재시도 | 운영 액션                             |
| ---------------------- | -------------------------------------------- | ----------- | ------------------------------------- |
| `INPUT_LOAD_FAILED`    | MissionLog, Participant, Room 입력 로드 실패 | 가능        | `RETRY_WAIT` 후 재시도                |
| `CALCULATION_FAILED`   | 계산 중 예외 또는 불일치 감지                | 불가        | `FAILED`, 원인 분석 후 수동 대응      |
| `POINT_CREDIT_FAILED`  | 포인트 원장 반영 실패                        | 가능        | `RETRY_WAIT` 후 재시도                |
| `DUPLICATE_SETTLEMENT` | 중복 정산 생성 또는 이미 존재하는 정산 감지  | 불가        | 데이터 점검 후 수동 대응              |
| `LOCK_ACQUIRE_FAILED`  | 락 획득 실패                                 | 가능        | 짧은 간격으로 재시도                  |
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
- 이 보정 작업은 재정산이 아니라 FK 연결 보정이며, 새로운 포인트 지급 이벤트를 생성하지 않는다.
- 운영자는 `point_history` 자체가 없는 지급 실패와, `point_history`는 있으나 `settlement_item` 연결만 누락된 상태를 구분해서 대응해야 한다.
- 전자는 미반영 participant에 대한 재시도 대상이고, 후자는 FK 연결 보정 대상이다.
- `Settlement.status`는 `point_history` 생성 여부만이 아니라 `settlement_item`과의 연결 완료 여부까지 포함해 판단한다.
- 모든 `settlement_item`의 `point_history_id`가 채워지고 대응 `point_history` 존재가 검증되기 전까지는 `Settlement.status`를 `SUCCEEDED`로 바꾸지 않는다.

## 12. 예외 케이스

| 시나리오                     | 결정                                                                                                           |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 인원 미달 방 취소            | 방별 `min_participants`를 기준으로 `CANCELLED_BEFORE_START` 정산 생성 후 전액 환급                             |
| 중도 탈퇴                    | `RECRUITING`, `ACTIVE`에서 탈퇴 가능, `withdrawn_at` 이전 성공만 인정, 보증금은 최종 정산 또는 방 취소 시 환급 |
| DAILY 하루 다중 인증         | 같은 날짜 성공 로그는 1회만 인정, 나머지는 제외 근거 저장                                                      |
| SPECIFIC_DAYS 비유효 요일    | `mission_schedule_day`에 없는 요일의 성공 로그는 제외                                                          |
| WEEKLY_N 상한 초과           | 같은 주차에서 `N회` 초과 성공은 제외                                                                           |
| 전체 인정 성공 0회           | 잠겨 있던 보증금을 전액 균등 환급하고, 잔액은 draw 순위대로 `1원씩` 배분                                       |
| 참여자별 보증금 상이         | 총 풀은 합산하되, 결과 설명은 `deposit_amount`, `final_amount`, `share_ratio`로 제공                           |
| `ACTIVE` 이후 신규 참여 요청 | 거절한다. MVP에서는 모집 완료 후 참여자 구성을 고정한다.                                                       |
| 탈퇴 후 동일 방 재참여 요청  | 거절한다. 탈퇴는 해당 방 참여를 종료하는 최종 이벤트다.                                                        |
| 같은 방 중복 정산 시도       | 상태 claim + 락 + unique 제약 + `idempotency_key`로 차단                                                       |
| `Settlement` 누락            | 운영 보정 경로로 `PENDING` 생성, 단 `unique(room_id, settlement_type)` 준수                                    |
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
```

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

## 14. 외부 API 계약

> 외부 API의 최종 요청/응답 계약은 `API-spec-god-saving.md`를 따른다. 이 섹션은 정산 운영 흐름 설명을 위한 보조 설명이며, API 계약과 충돌할 경우 API-spec이 우선한다.

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
  "started_at": "2026-06-01T00:05:10+09:00",
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
      "started_at": "2026-06-01T00:05:10+09:00",
      "finished_at": "2026-06-01T00:05:20+09:00"
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

## 15. 후속 이벤트

정산 커밋 이후에만 아래 후속 작업을 수행한다.

1. 인앱 알림
2. 정산 완료 이메일
3. AI 습관 리포트 생성
4. 운영 모니터링 지표 적재

원칙:

- 후속 작업 실패는 정산 성공을 되돌리지 않는다.
- 따라서 정산 트랜잭션 밖에서 `SettlementCompleted` 이벤트를 소비하게 한다.
- 정산 완료 이메일 실패는 `Settlement.status`, `settlement_item`, `point_history`, 결제 충전 원장을 수정하거나 롤백하지 않는다.
- 이메일 발송은 SMTP 기반 best-effort 후속 작업이다.
- MVP에서는 notification log/outbox를 필수 테이블로 두지 않는다.
- 이메일 실패는 structured log, bounded retry, 운영자 수동 재발송 대상으로만 다룬다.
- structured log는 최소 `settlement_id`, `member_id`, `email_type`, `recipient_hash`, `attempt`, `result`, `smtp_error_code`, `created_at`을 포함한다.

## 16. 골든 데이터 예시

### 예시 A. 일반 정산

조건:

- 참여자 5명
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
기여도 1위 A가 remainder 4원 수령
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
equal_base = FLOOR(300000 / 3) = 100000
remainder = 0
```

최종:

- 전원 `100,000원`
- 잔액이 생기는 경우에는 `draw_key` 순위 상위 참여자부터 `1원씩` 추가 배분한다.

### 예시 C. 1위 동점

조건:

- 참여자 4명
- 총 예치금 40,000원
- 성공 횟수: `A=5`, `B=5`, `C=3`, `D=1`

결정:

- A와 B가 공동 1위
- `draw_key = SHA-256(room_id + ":" + settlement_type + ":" + member_id)` 값이 더 작은 쪽이 잔액 수령자
- 테스트에서는 `room_id`, `settlement_type`, `member_id`를 고정해 기대 결과를 재현한다

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

- `TS-LC-01` host `StartRoom` 성공
  기대 결과: `RECRUITING` 방에서 host가 시작 조건을 만족해 command를 실행하면 `ACTIVE`로 전이하고 `activated_at`이 기록된다.
- `TS-LC-02` `min_participants` 미달 시작 실패
  기대 결과: command 실행 시점의 eligible participant 수가 `min_participants` 미만이면 `ACTIVE`로 전이하지 않는다.
- `TS-LC-03` `start_at` 만료 후 시작 실패
  기대 결과: `start_at` 이후 host start 요청은 만료 오류 또는 이미 취소된 최종 상태로 응답한다.
- `TS-LC-04` 시작 만료 취소 settlement 멱등성
  기대 결과: `RECRUITING -> CANCELLED` batch가 성공하면 `CANCELLED_BEFORE_START` settlement가 1회만 생성되고 재시도해도 중복 환급되지 않는다.
- `TS-LC-05` `StartRoom`과 취소 batch 경합
  기대 결과: 하나의 조건부 전이만 성공하고 loser는 최종 상태를 재조회한다.
- `TS-LC-06` 인증/log eligibility anchor
  기대 결과: `MissionLog.server_time < room.activated_at`이면 `BEFORE_START` 또는 동등 사유로 정산 인정에서 제외한다.

### 18.1 단위 테스트

- `TS-01` `min_participants`가 방별로 적용되는지
  기대 결과: 방 생성 시 host가 값을 설정할 수 있고, 설정하지 않으면 기본값 `2`가 저장된다.
- `TS-02` `min_participants > max_participants`이면 생성 실패
  기대 결과: `2 <= min_participants <= max_participants <= 10` 검증을 통과하지 못하면 요청이 reject된다.
- `TS-03` `RECRUITING`, `ACTIVE` 상태 탈퇴 가능 여부
  기대 결과: `RECRUITING`, `ACTIVE` 방 모두에서 탈퇴가 허용되고, 보증금은 즉시 환급되지 않는다.
- `TS-03A` 참여 시 보증금 잠금 처리
  기대 결과: 참여 시 `point_account.balance`는 보증금만큼 감소하고, 같은 금액이 `room_participant.deposit_amount`에 잠기며 `ROOM_DEPOSIT_LOCK` 원장이 생성된다.
- `TS-03B` 중도 탈퇴 후 보증금 잠금 유지
  기대 결과: `WITHDRAWN` 상태가 되어도 `room_participant.deposit_amount`는 정산 전까지 유지되고 즉시 환급되지 않는다.
- `TS-03C` 보증금 잠금 시 사용 가능 잔액 음수 방지
  기대 결과: `point_account.balance >= deposit_amount` 조건부 update의 row count가 `1`일 때만 잠금이 성공하고, 동시 요청 또는 잔액 부족으로 row count가 `0`이면 참여가 실패한다.
- `TS-04` `WITHDRAWN` 참여자의 인증 요청 차단
  기대 결과: `participant.status == WITHDRAWN`이면 인증 API가 reject된다.
- `TS-04A` `ACTIVE` 이후 신규 참여 불가
  기대 결과: `MissionRoom.status != RECRUITING`이면 신규 참여 요청이 reject된다.
- `TS-04B` 탈퇴 후 동일 방 재참여 불가
  기대 결과: `WITHDRAWN` 이력이 있는 동일 `member`의 재참여 요청이 reject되고 기존 `participant`가 재사용되거나 새로 생성되지 않는다.
- `TS-05` 중도 탈퇴자의 성공 로그 cutoff 처리
  기대 결과: `withdrawn_at` 이후 성공 로그는 `excluded_success_count`에 반영되고 `recognized_success_count`에는 포함되지 않는다.
- `TS-06` DAILY 하루 다중 인증은 1회만 인정
  기대 결과: 같은 날짜 성공 로그가 3건이어도 인정은 1건이고 제외 사유가 저장된다.
- `TS-07` SPECIFIC_DAYS 유효 요일 외 성공 로그 제외
  기대 결과: 스케줄에 없는 요일의 성공 로그는 제외되고 `INVALID_SCHEDULE_DAY` 근거가 저장된다.
- `TS-07A` 인증 시점 실패 사유와 정산 시점 제외 사유가 분리되는지
  기대 결과: `BEFORE_START`, `AFTER_END`, `AFTER_WITHDRAWN` 같은 1차 실패는 `mission_log.failure_reason`에 남고, `DAILY` 중복 제외나 `WEEKLY_N` 상한 제외는 `settlement_item.calculation_reason`에만 남는다.
- `TS-08` WEEKLY_N 주차별 인정 상한 적용
  기대 결과: 주간 버킷별 인정 수가 `N`을 넘지 않는다.
- `TS-08A` `frequency_type`별 recognition strategy가 올바르게 선택되는지
  기대 결과: `DAILY`, `SPECIFIC_DAYS`, `WEEKLY_N` 각각에서 대응 전략 1개만 선택되고, `recognized_success_count`, `recognized_dates_count`, `excluded_success_count`, `calculation_reason` 출력 계약이 동일하게 유지된다.
- `TS-09` 전체 성공 0회 시 균등 환급 적용
  기대 결과: 모든 참여자의 `base_refund_amount`가 `equal_base`로 동일하고 별도 차감 규칙이 적용되지 않는다.
- `TS-10` 전체 성공 0회에서 remainder가 deterministic draw로 분배되는지
  기대 결과: 남은 잔액이 draw 순위대로 `1원씩` 배분되고 재실행 시 동일한 결과가 나온다.
- `TS-11` 참여자별 보증금이 다른 경우
  기대 결과: `total_locked_amount`는 합산되고 각 참여자의 `deposit_amount` 대비 `final_amount`가 일관되게 계산된다.
- `TS-11A` `total_participants`가 `WITHDRAWN` 포함 정산 대상 수로 계산되는지
  기대 결과: 종료 시점까지 locked deposit이 남아 있는 `WITHDRAWN` 참여자는 `total_participants`와 `settlement_item` 생성 대상에 포함된다.
- `TS-11B` 동일 `member`가 여러 방에 참여한 경우 보증금 잠금이 방별로 분리되는지
  기대 결과: 각 방의 `room_participant.deposit_amount`와 관련 `point_history`가 독립적으로 기록되고, 한 방의 정산이 다른 방 잠금 금액을 변경하지 않는다.
- `TS-11C` `total_locked_amount`가 participant 잠금 스냅샷 기준으로 계산되는지
  기대 결과: 정산 실행 시점의 정산 대상 participant `deposit_amount` 합계가 그대로 `total_locked_amount`에 저장되고, `point_history`나 `point_account` 재합산값은 사용되지 않는다.
- `TS-12` 일반 정산 로직이 영향받지 않았는지
  기대 결과: `전체 인정 성공 횟수 > 0`인 경우 기존 지분율 기반 정산과 `TOP_1_ALL` 잔액 정책이 그대로 유지된다.
- `TS-13` deterministic draw 재현성
  기대 결과: 같은 `room_id`, `settlement_type`, `member_id` 집합이면 재실행해도 동일 순서가 나온다.

### 18.2 통합 테스트

- `TS-14` 방 1개 정산 시 participant 단위 지급 트랜잭션에서 `point_history`와 `point_account.balance`가 함께 반영되고, 전체 정산은 partial 복구 가능 상태로 관리되는지
- `TS-14A` `settlement_item`이 먼저 생성되고 이후 `point_history`가 연결되는지
  기대 결과: 계산 스냅샷 없이 원장만 단독 생성되지 않고, `SUCCEEDED` 상태에서는 모든 `settlement_item.point_history_id`가 채워진다.
- `TS-14AA` `SUCCEEDED` 전환 전에 모든 환급 원장 연결이 검증되는지
  기대 결과: `point_history` 생성 또는 `point_history_id` 연결이 하나라도 누락되면 `Settlement.status`는 `SUCCEEDED`로 바뀌지 않고 `RETRY_WAIT` 또는 `FAILED`로 남는다.
- `TS-14B` 정산 환급 후 사용 가능 잔액이 증가하는지
  기대 결과: `ROOM_SETTLEMENT_REFUND` 또는 `ROOM_CANCELLED_REFUND` 기록과 함께 `point_account.balance`가 `final_amount`만큼 증가한다.
- `TS-14C` 포인트 원장 기록이 충전, 잠금, 환급 흐름과 일치하는지
  기대 결과: 같은 `member` 기준으로 `POINT_CHARGE -> ROOM_DEPOSIT_LOCK -> ROOM_SETTLEMENT_REFUND` 또는 `ROOM_CANCELLED_REFUND` 순서의 잔액 변화가 `balance_after`와 함께 일관되게 남는다.
- `TS-14D` `point_account.balance`와 `point_history` 재계산값이 불일치할 때 원장 기준으로 복구되는지
  기대 결과: 운영 검증은 `point_history`를 source of truth로 삼아 불일치 원인을 기록하고, `point_account.balance` 캐시를 원장 재계산값으로 보정한다.
- `TS-15` 배치 재시도 시 `point_history.idempotency_key`로 중복 지급이 차단되는지
  기대 결과: 같은 `room_id`, `settlement_type`, `participant_id` 입력이면 같은 `idempotency_key`가 재사용되고, 동일 payload duplicate는 기존 원장 재사용/연결로 수렴하며, 다른 payload duplicate는 idempotency conflict로 실패한다.
- `TS-15A` `point_history`는 존재하지만 `settlement_item.point_history_id`만 누락된 partial 상태를 안전하게 복구하는지
  기대 결과: 해당 participant는 이미 지급 완료로 간주되고 새 환급 원장은 생성되지 않으며, 관리자 API 또는 배치가 기존 `point_history`를 조회해 FK만 연결한 뒤 전체 연결이 완료되면 `Settlement.status`가 `SUCCEEDED`로 전이된다.
- `TS-16` 취소형 정산과 일반 정산이 같은 조회 API 구조로 반환되는지
- `TS-17` 종료/취소 감지 시 `Settlement(PENDING)`가 먼저 생성되는지
- `TS-17A` `unique(room_id, member_id)` 제약이 같은 방 중복 participant 생성을 막는지
  기대 결과: 동일 `member`가 같은 `MissionRoom`에 두 번째로 참여를 시도하면 DB 제약 또는 동일 수준의 저장 전 검증으로 차단된다.
- `TS-17B` 실시간 대시보드 캐시와 `SUCCEEDED` 전 정산 계산 결과가 일시적으로 달라도 MissionLog 재계산으로 정산값이 확정되는지
  기대 결과: 캐시 누락 또는 지연이 있어도 `settlement_item` 계산값은 `MissionLog` 원본 기준으로 일관되게 생성된다.

### 18.3 동시성 테스트

- `TS-17C` 동시 인증 업로드 시 room 단위 분산 락 없이도 `MissionLog` append-only 저장이 보존되는지
  기대 결과: 같은 시점 인증 요청이 겹쳐도 로그는 유실 없이 저장되고, 실시간 지표는 일부 지연될 수 있지만 최종 정산 결과는 변하지 않는다.
- `TS-18` 같은 방에 대해 동시 정산 요청이 들어온 경우 조건부 claim에서 1개 워커만 성공하는지
- `TS-19` 락 획득 실패 시 실행이 시작되지 않고 재시도 대상으로 남는지
- `TS-20` `unique(room_id, settlement_type)` 제약이 중복 생성 시도를 막는지

### 18.4 운영 검증

- `TS-21` 실패 코드별 `RETRY_WAIT` / `FAILED` 전이가 기대대로 기록되는지
- `TS-22` `FAILED` 건이 관리자 API에서 조회되는지
- `TS-23` `RUNNING` timeout 건을 운영 배치가 `RETRY_WAIT`로 복구할 수 있는지
- `TS-23A` 종료 감지 누락 시 운영 보정으로 `Settlement(PENDING)`를 생성할 수 있는지
  기대 결과: `room_id` 기준 정산 대상 여부 재검사 후 누락된 정산만 보정 생성되고, `unique(room_id, settlement_type)` 제약 위반 시 중복 생성되지 않는다.

## 19. 구현 시작 순서

1. `settlement`, `settlement_item`, `point_history` 확장 컬럼과 unique 제약을 먼저 확정한다.
2. `SettlementPendingCreator`, `SettlementInputAssembler`, `MissionRecognitionStrategy`, `SettlementCalculator` 책임을 분리한다.
3. `TS-01`~`TS-09`와 `TS-08A` 골든/단위 테스트를 먼저 작성한다.
4. 그 다음 `Spring Batch + Redisson + conditional claim + PointHistory idempotency`를 붙인다.
