# MVP 티켓 분해: 갓세이빙

기준 문서:

- [PRD-god-saving.md](./PRD-god-saving.md)
- [API-spec-god-saving.md](./API-spec-god-saving.md)
- [ERD-god-saving.md](./ERD-god-saving.md)
- [Settlement-design.md](./Settlement-design.md)
- [adr/ADR-mvp-tech-architecture.md](./adr/ADR-mvp-tech-architecture.md)
- [MVP-backlog-user-stories.md](./MVP-backlog-user-stories.md)

## 1. 분해 기준

- 이 문서는 `Epic -> Story -> Ticket` 수준으로 MVP 구현 항목을 쪼갠 실행 문서다.
- 티켓은 가능한 한 `1개 역할이 1~3일 안에 완료 가능한 크기`를 목표로 나눴다.
- 역할은 임시로 `BE`, `FE`, `Infra`, `QA`로 표기했다.
- 각 티켓은 `Why-What-Acceptance` 형식을 따른다.
- 이 문서는 실행 계획이며 API/payment/DB/settlement 계약의 source of truth가 아니다. 문서 간 충돌 시 `PRD -> API-spec -> ERD -> Settlement-design -> ADR -> Tech-stack summary -> Ticket breakdown` 순으로 따른다.
- 실제 팀 구성이 풀스택 중심이면 같은 story 아래의 `BE + FE` 티켓을 하나로 합쳐도 된다.

## 2. 릴리스 티켓 맵

| Ticket | Phase | Role  | Linked Story  | 제목                                            | 선행 의존성 |
| ------ | ----- | ----- | ------------- | ----------------------------------------------- | ----------- |
| T-01   | 1     | BE    | US-01, US-01A | 인증 도메인과 최소 프로필 API 구현              | 없음        |
| T-02   | 1     | FE    | US-01, US-01A | 회원가입/로그인 화면과 프로필 UI 구현           | T-01        |
| T-03   | 1     | BE    | US-02         | 크루 생성 API와 규칙 검증 구현                  | T-01        |
| T-04   | 1     | FE    | US-02         | 크루 생성 폼과 입력 검증 UI 구현                | T-03        |
| T-04A  | 2     | BE    | US-06A        | 방장 미션 시작 API와 lifecycle 전이 구현        | T-03, T-09  |
| T-04B  | 2     | FE    | US-06A        | 방장 미션 시작 버튼과 상태 안내 UI 구현         | T-04A       |
| T-05   | 1     | BE    | US-05         | 포인트 원장과 Toss confirm-only 충전 처리 구현  | T-01        |
| T-06   | 1     | FE    | US-05         | 포인트 충전/이력 화면 구현                      | T-05        |
| T-07   | 2     | BE    | US-03, US-04  | 공개 목록/상세/참여코드 조회 API 구현           | T-03        |
| T-08   | 2     | FE    | US-03, US-04  | 크루 탐색/상세/참여코드 입력 UI 구현            | T-07        |
| T-09   | 2     | BE    | US-06         | 크루 입장과 보증금 예치 트랜잭션 구현           | T-05, T-07  |
| T-10   | 2     | FE    | US-06         | 크루 입장과 잔액 부족 처리 UI 구현              | T-09        |
| T-11   | 2     | Infra | US-07         | 인증 이미지 업로드 저장소와 접근 정책 구성      | T-03        |
| T-12   | 2     | BE    | US-07         | 인증 제출 API와 시간 규칙 검증 구현             | T-09, T-11  |
| T-13   | 2     | BE    | US-08         | Exif 파서와 인증 결과 저장 로직 구현            | T-12        |
| T-14   | 2     | FE    | US-07, US-08  | 인증 업로드/결과 UI 구현                        | T-12, T-13  |
| T-15   | 2     | QA    | US-01~US-08   | 가입-입장-인증 회귀 테스트 정의                 | T-14        |
| T-16   | 3     | BE    | US-09         | 지분율 집계 조회 모델 구현                      | T-13        |
| T-17   | 3     | BE    | US-10         | 정산 규칙 엔진 구현                             | T-09, T-13  |
| T-18   | 3     | BE    | US-10         | 정산 배치, 시작 전 취소, 재시도, 멱등 처리 구현 | T-17, T-04A |
| T-19   | 3     | BE    | US-10, US-11  | 환급 반영과 정산 히스토리 API 구현              | T-18        |
| T-20   | 3     | FE    | US-09         | 실시간 지분율 대시보드 UI 구현                  | T-16        |
| T-21   | 3     | FE    | US-11         | 정산 결과/포인트 히스토리 화면 구현             | T-19        |
| T-22   | 3     | QA    | US-06A, US-09~US-11 | 정산/시작 lifecycle 골든 데이터 테스트 작성 | T-18, T-21  |
| T-23   | 4     | BE    | US-12         | SSE 이벤트 발행과 구독 인증 구현                | T-13, T-16  |
| T-24   | 4     | FE    | US-12         | 인앱 알림 UI와 SSE 클라이언트 구현              | T-23        |
| T-25   | 4     | BE    | US-13         | 정산 완료 이메일 발송 구현                      | T-19        |
| T-26   | 4     | BE    | US-14         | AI 미션 추천 API 연동 구현                      | T-03        |
| T-27   | 4     | FE    | US-14         | AI 미션 추천 UI와 폼 반영 구현                  | T-26        |
| T-28   | 4     | BE    | US-15         | AI 습관 리포트 생성/저장 구현                   | T-19        |
| T-29   | 4     | FE    | US-15         | AI 습관 리포트 조회 UI 구현                     | T-28        |
| T-30   | 4     | BE    | US-16         | 관리자 정산 상태 조회 API 구현                  | T-18        |
| T-31   | 4     | Ops   | US-16         | 관리자 정산 모니터링 화면 구현 — Deferred / MVP 제외 | T-30        |
| T-32   | 4     | Infra | US-16         | 배치/정산 운영 모니터링과 알림 구성             | T-18, T-30  |
| T-33   | 2     | BE    | US-17         | 인증 피드 조회 API 구현                         | T-13        |
| T-34   | 2     | BE    | US-18         | 인증 피드 리액션 API 구현                       | T-33        |

`T-33`과 `T-34`는 피드/리액션 API 구현 누락을 보완하는 appended tickets이며, 기존 정산 관련 티켓(`T-17`~`T-22`)의 선행 의존성이나 우선순위를 변경하지 않는다.

## 3. 실행 순서

### Critical Path

1. `T-01 -> T-03 -> T-07 -> T-09 -> T-12 -> T-13 -> T-17 -> T-18 -> T-19`
2. 이 경로가 끝나야 MVP의 핵심 루프인 `가입 -> 생성/참여 -> 예치 -> 인증 -> 정산 -> 환급`이 닫힌다.

### Parallel Lanes

- FE 레인:
  `T-02`, `T-04`, `T-06`은 각 대응 BE API가 고정되면 병렬 진행 가능
- Infra 레인:
  `T-11`, `T-32`는 핵심 도메인 개발과 병렬 진행 가능
- QA 레인:
  `T-15`, `T-22`는 구현 완료 직전이 아니라 API 계약이 보이면 먼저 시나리오를 작성하는 편이 낫다

### Phase Exit Criteria

| Phase   | 종료 기준                                                                    |
| ------- | ---------------------------------------------------------------------------- |
| Phase 1 | 사용자가 가입하고 로그인한 뒤 포인트를 충전하고 크루를 생성할 수 있다.       |
| Phase 2 | 사용자가 공개/비공개 경로로 입장하고 인증 업로드 성공/실패를 확인할 수 있다. |
| Phase 3 | 종료된 미션이 자동 정산되고 환급 결과와 이력을 조회할 수 있다.               |
| Phase 4 | 알림, 이메일, AI, 운영 화면이 핵심 흐름을 깨지 않고 붙는다.                  |

## 3.1 Implementation governance gates

각 티켓의 Acceptance Criteria는 구현 중 refinement될 수 있지만, `docs/implementation-gates.md`의 blocker-level invariant는 구현 중 깨지면 안 된다. 이 섹션은 티켓별 PR 리뷰가 어느 gate를 반드시 참조해야 하는지 지정한다.

| Ticket | Gate source | Blocker-level focus |
| --- | --- | --- |
| T-05 | Payment PR Gate | `payment_id = paymentKey`, `charge:{paymentKey}`, duplicate confirm reuse/conflict, point_history 없는 balance update 금지 |
| T-11/T-12/T-13 | Storage/EXIF PR Gate | presigned URL은 upload delegation only, server-generated key, private object, server-side S3/EXIF validation |
| T-17/T-18/T-19 | Settlement PR Gate | conditional claim, settlement_item snapshot, participant deterministic idempotency, partial recovery, SUCCEEDED FK 검증 |
| T-30/T-32 | Recovery/Ops PR Gate | Admin API/runbook recovery, Redis unavailable DB-claim-only fallback, RUNNING timeout recovery, structured log/CloudWatch minimum |

Admin UI는 T-31로 분리되어 MVP 제외다. 따라서 T-30/T-32는 UI가 아니라 API, log, alarm, runbook만으로 운영 복구가 가능한지를 검증해야 한다.

## 4. 티켓 상세

### T-01. 인증 도메인과 최소 프로필 API 구현

**Why:**  
모든 MVP 기능은 인증된 사용자 기준으로 동작한다. 초기에 인증 경계를 안정적으로 세워야 이후 크루, 포인트, 인증, 정산 권한 제어가 흔들리지 않는다.

**What:**  
회원가입, 로그인, JWT 발급/검증, 보호 API 접근 제어를 구현한다. `member.uuid`를 회원 생성 시 발급하고 JWT subject(`sub`)로 사용한다. Spring Security principal은 내부 처리를 위해 `memberId`와 `memberUuid`를 모두 가질 수 있지만, external boundary의 canonical identifier는 UUID다. `US-01A`를 위해 `GET /api/me` profile read와 `PATCH /api/me/profile` profile update도 같은 Auth / Profile 범위에서 구현한다. 민감한 인증 실패 응답은 과도한 내부 정보를 노출하지 않도록 정리한다.

**Acceptance Criteria:**

- 회원가입과 로그인이 API 수준에서 동작한다.
- 로그인 성공 시 보호 API에 사용할 인증 토큰이 발급된다.
- 인증되지 않은 호출은 보호 API 접근이 차단된다.
- 잘못된 자격 증명 응답에 계정 존재 여부 같은 민감 정보가 노출되지 않는다.
- 사용자는 자신의 프로필(닉네임, 프로필 이미지)을 조회할 수 있다.
- 사용자는 프로필을 수정할 수 있다.
- 프로필 API는 소셜 프로필 기능을 만들지 않는다.

### T-02. 회원가입/로그인 화면과 프로필 UI 구현

**Why:**  
인증 API만 있어서는 사용자 흐름이 닫히지 않는다. 첫 진입 사용자가 실제로 로그인 상태를 만들고 보호 화면으로 이동할 수 있어야 한다.

**What:**  
회원가입/로그인 UI, 입력 오류 처리, 로그인 상태 유지, 보호 라우트 가드를 구현한다. `US-01A`를 위해 닉네임과 프로필 이미지만 확인/수정하는 최소 프로필 UI도 포함한다. 인증 만료 시 재로그인 흐름도 포함한다.

**Acceptance Criteria:**

- 사용자는 화면에서 회원가입과 로그인을 수행할 수 있다.
- 로그인 성공 시 보호 화면 접근이 가능하다.
- 잘못된 입력과 인증 실패가 사용자에게 이해 가능한 메시지로 보인다.
- 로그인 상태가 없거나 만료되면 보호 화면 접근 시 인증 화면으로 이동한다.
- 사용자는 화면에서 닉네임과 프로필 이미지를 확인하고 수정할 수 있다.
- 프로필 UI는 소셜 프로필 기능을 제공하지 않는다.

### T-03. 크루 생성 API와 규칙 검증 구현

**Why:**  
크루 생성은 제품의 핵심 계약 입력 지점이다. 기간, 인원, 보증금 규칙이 이 단계에서 고정돼야 이후 입장, 인증, 정산 계산이 일관된다.

**What:**  
공개/비공개 크루 생성 API와 규칙 검증을 구현한다. 기간 `1주~3개월`, 최대 인원 `10명`, 보증금 `1,000원~100만원`, `1,000원 단위`, 비공개 `6자리 참여 코드`, `min_participants`, `recruitment_deadline`, 예정 시작/수동 시작 만료 시각인 `start_at` 생성 규칙을 포함한다.

**Acceptance Criteria:**

- 주최자는 공개 또는 비공개 크루를 생성할 수 있다.
- 허용 범위를 벗어난 기간, 인원, 보증금은 저장되지 않는다.
- 비공개 크루 생성 시 고유한 참여 코드가 생성된다.
- 생성된 크루는 운영자 정보와 함께 조회 가능하다.
- `min_participants`는 자동 시작 트리거가 아니라 `StartRoom` precondition으로 문서화된다.
- `recruitment_deadline`은 join cutoff, `start_at`은 planned start/latest manual-start deadline으로 저장된다.

### T-04. 크루 생성 폼과 입력 검증 UI 구현

**Why:**  
크루 생성 규칙이 복잡하므로 UI 단계에서도 사전 검증이 필요하다. 서버 실패만으로 피드백을 주면 생성 전환율이 떨어진다.

**What:**  
크루 생성 폼, 공개/비공개 전환, 기간/보증금/인원 입력 검증, 제출 성공 후 상세 화면 이동을 구현한다.

**Acceptance Criteria:**

- 사용자는 화면에서 크루 생성 값을 입력할 수 있다.
- 기간, 보증금, 인원 제약이 UI에서 먼저 안내된다.
- 비공개 선택 시 참여 코드 발급 결과가 확인된다.
- 생성 성공 후 사용자는 새 크루 상세 화면으로 이동한다.

### T-04A. 방장 미션 시작 API와 lifecycle 전이 구현

**Why:**  
최소 인원 충족을 자동 ACTIVE 전이로 해석하면 정산 기준 시각, 인증 가능 시점, participant baseline이 흔들린다. MVP에서는 host `StartRoom` command를 단일 lifecycle commit point로 고정해야 한다.

**What:**  
`POST /api/rooms/{roomId}/start` command를 구현한다. host 권한, `RECRUITING` 상태, `start_at` 만료 여부, command 시점 `min_participants` 재검증, `activated_at` 기록, `ACTIVE` 전이를 하나의 조건부 transaction으로 처리한다.

**Acceptance Criteria:**

- host만 미션 시작을 요청할 수 있다.
- 성공 시 `status = ACTIVE`와 `activated_at = server_now`가 함께 기록된다.
- 이미 `ACTIVE`인 방은 멱등 성공/no-op으로 처리된다.
- `CANCELLED`/`CLOSED` 방은 conflict로 거절된다.
- `min_participants` 미달은 `MIN_PARTICIPANTS_NOT_MET` 계열 오류로 거절된다.
- `start_at` 경과 후 요청은 `ROOM_START_EXPIRED` 또는 terminal-state conflict로 거절된다.
- 동시 start 요청은 하나의 조건부 전이만 성공한다.

### T-04B. 방장 미션 시작 버튼과 상태 안내 UI 구현

**Why:**  
최신 기획안의 “방장이 미션 시작 버튼을 눌러 시작” 정책을 사용자가 이해하려면 상세 화면에서 시작 가능 조건과 만료 상태를 명확히 보여줘야 한다.

**What:**  
host 전용 미션 시작 버튼, 최소 인원 충족 여부, `recruitment_deadline`, `start_at` 시작 가능 만료 안내, 시작 실패/성공 상태 메시지를 구현한다.

**Acceptance Criteria:**

- host는 시작 가능한 `RECRUITING` 방에서 미션 시작 버튼을 볼 수 있다.
- 비 host는 시작 버튼을 볼 수 없거나 권한 없음 안내를 본다.
- 최소 인원 미달, 모집 마감 전/후, 시작 만료 상태가 구분되어 표시된다.
- 시작 성공 후 화면은 `ACTIVE`와 `activated_at` 기준 진행 상태를 보여준다.

### T-05. 포인트 원장과 Toss confirm-only 충전 처리 구현

**Why:**  
보증금 예치와 환급은 모두 포인트 원장 위에서 움직인다. 원장과 충전 반영이 정확하지 않으면 정산 신뢰 자체가 무너진다.

**What:**  
포인트 잔액 모델, `Point_History`, TossPayments sandbox confirm-only 충전 성공/실패 반영, 중복 confirm 재시도 방지 로직을 구현한다. API의 `payment_id`는 Toss `paymentKey`이며, `orderId`는 confirm 검증과 로그 상관관계 추적용으로만 사용한다.

**Acceptance Criteria:**

- 충전 성공 시 포인트 잔액이 증가한다.
- 충전 실패 또는 취소 시 잔액이 증가하지 않는다.
- 모든 충전 결과가 `Point_History`에 기록된다.
- `POINT_CHARGE` idempotency key는 `charge:{paymentKey}`를 사용한다.
- 동일 `paymentKey` + 동일 payload confirm 재시도는 기존 `point_history`를 재사용한다.
- 동일 `paymentKey` + 다른 payload는 idempotency conflict로 실패한다.
- `orderId`는 `point_history.idempotency_key` 구성값으로 사용하지 않는다.

### T-06. 포인트 충전/이력 화면 구현

**Why:**  
사용자는 입장 전에 잔액과 충전 상태를 이해해야 한다. 포인트 이동의 가시성은 이후 환급 신뢰에도 직접 연결된다.

**What:**  
충전 요청 화면, 잔액 표시, 최근 충전/이력 조회 UI를 구현한다. 결제 실패와 취소 상태도 구분해 표시한다.

**Acceptance Criteria:**

- 사용자는 충전 금액을 선택하고 결제를 시작할 수 있다.
- 충전 성공 후 최신 잔액이 반영된다.
- 실패 또는 취소 내역도 이력에서 구분된다.
- 최근 포인트 이동 내역을 화면에서 확인할 수 있다.

### T-07. 공개 목록/상세/참여코드 조회 API 구현

**Why:**  
입장 전 사용자는 어떤 크루인지 판단할 정보가 필요하다. 공개 탐색과 비공개 코드 조회는 서로 다른 유입 경로이므로 API를 명확히 분리해야 한다.

**What:**  
공개 크루 목록, 상세 조회, 키워드 검색, 비공개 참여 코드 조회 API를 구현한다. 비공개 크루는 공개 목록에서 제외한다.

**Acceptance Criteria:**

- 공개 크루 목록과 상세 조회 API가 동작한다.
- 목록에서 기간, 보증금, 인원, 모집 상태를 조회할 수 있다.
- 비공개 크루는 공개 목록과 검색 결과에서 제외된다.
- 유효한 참여 코드로만 비공개 크루 정보를 조회할 수 있다.

### T-08. 크루 탐색/상세/참여코드 입력 UI 구현

**Why:**  
탐색 경험이 막히면 공개 유입과 초대 유입이 모두 끊긴다. 입장 전 핵심 규칙을 읽을 수 있는 화면이 필요하다.

**What:**  
공개 크루 목록, 상세, 검색, 참여 코드 입력 UI를 구현한다. 모집 종료나 정원 초과 상태도 시각적으로 구분한다.

**Acceptance Criteria:**

- 사용자는 공개 크루 목록과 상세를 볼 수 있다.
- 목록에서 검색 또는 상태 필터를 사용할 수 있다.
- 참여 코드 입력으로 비공개 크루 진입 정보를 확인할 수 있다.
- 모집 종료 또는 정원 초과 상태가 입장 가능 상태와 구분된다.

### T-09. 크루 입장과 보증금 예치 트랜잭션 구현

**Why:**  
입장과 예치는 분리되면 정합성이 깨진다. 사용자 참여 상태와 보증금 잠금은 같은 트랜잭션 경계 안에서 처리돼야 한다.

**What:**  
크루 입장, 잔액 차감, 보증금 잠금, 중복 입장 방지, 인원 초과 방지 로직을 구현한다. 같은 참여 lifecycle 범위에서 방 탈퇴 API도 구현하되, 탈퇴는 즉시 환급이 아니라 `withdrawn_at` 기록과 후속 정산 cutoff 입력으로 남긴다.

**Acceptance Criteria:**

- 잔액이 충분한 사용자만 크루 입장을 완료할 수 있다.
- 입장 성공 시 보증금이 잠금 상태로 반영된다.
- 중복 입장과 정원 초과 입장이 차단된다.
- 입장과 예치 결과가 포인트/참여 상태에 일관되게 기록된다.
- 인증된 기존 참여자만 `POST /api/rooms/{roomId}/withdraw`를 호출할 수 있고, 비참여/권한 없음/이미 탈퇴한 요청은 차단된다.
- `RECRUITING` 또는 `ACTIVE` 참여자가 탈퇴하면 `room_participant.status = WITHDRAWN`과 `withdrawn_at`이 기록되며, 잠긴 보증금은 즉시 환급하지 않고 방 취소 또는 최종 정산까지 유지된다.
- `ACTIVE` 탈퇴 후 인증 제출은 차단되고, 정산/대시보드 계산은 `withdrawn_at` 이전 성공만 인정해 `AFTER_WITHDRAWN`/`AFTER_WITHDRAWN_AT` 경계와 일관된다.

### T-10. 크루 입장과 잔액 부족 처리 UI 구현

**Why:**  
입장 과정에서 가장 자주 만나는 실패는 잔액 부족이다. 이 실패를 명확히 보여주고 충전 흐름으로 연결해야 전환 손실이 줄어든다.

**What:**  
크루 입장 버튼, 예치 확인, 잔액 부족 안내, 충전 화면 이동, 이미 참여한 상태 표시 UI를 구현한다. 참여 중 사용자의 탈퇴 흐름은 lifecycle 상태 표시와 확인 UI만 제공하고 즉시 환급 UX로 오해되지 않게 안내한다.

**Acceptance Criteria:**

- 사용자는 크루 상세에서 입장을 시도할 수 있다.
- 잔액이 부족하면 입장 실패 이유와 다음 행동이 표시된다.
- 입장 성공 후 참여 상태와 예치 금액이 화면에 반영된다.
- 이미 참여 중인 사용자는 중복 입장 대신 현재 상태를 본다.
- 참여 중 사용자는 탈퇴 확인 UI를 통해 방 탈퇴를 요청할 수 있고, 성공 후 `WITHDRAWN` 상태와 탈퇴 시각이 표시된다.
- 탈퇴 후에도 보증금은 정산/취소 전까지 잠긴 상태로 안내되며, 즉시 환급 가능 금액처럼 표시하지 않는다.

### T-11. 인증 이미지 업로드 저장소와 접근 정책 구성

**Why:**  
이미지 인증은 파일 저장과 권한 정책이 먼저 안정되어야 한다. 저장소 구성이 약하면 보안과 운영 비용 둘 다에서 문제가 생긴다.

**What:**  
S3 버킷, 업로드 경로 규칙, 파일 타입/크기 제약, 접근 정책, 환경별 설정을 구성한다. 인증 이미지 저장을 위한 최소 보안 정책을 적용한다.

**Acceptance Criteria:**

- 인증 이미지가 지정된 저장소에 업로드될 수 있다.
- 허용 파일 타입과 크기 제한이 적용된다.
- 인증되지 않은 사용자는 업로드 경로를 임의로 사용할 수 없다.
- 환경별 설정으로 로컬/배포 환경에서 같은 방식으로 연결된다.

### T-12. 인증 제출 API와 시간 규칙 검증 구현

**Why:**  
업로드 성공만으로는 인증이 아니다. 허용 시간과 참여 여부를 먼저 검증해야 Exif 검사 이전 단계에서도 불필요한 집계를 막을 수 있다.

**What:**  
인증 제출 API, 참여 여부 검증, 허용 시간 검증, 서버 수신 시각 저장을 구현한다. 주기별 규칙에 맞지 않는 제출은 실패 상태로 저장한다.

**Acceptance Criteria:**

- 참여자만 인증 제출 API를 호출할 수 있다.
- 서버 수신 시각이 각 제출에 기록된다.
- 허용 시간 밖 제출은 성공 횟수로 집계되지 않는다.
- 제출 결과가 후속 Exif 검증과 연결 가능한 상태로 저장된다.

### T-13. Exif 파서와 인증 결과 저장 로직 구현

**Why:**  
갓세이빙의 공정성은 Exif 검증에 크게 의존한다. 이 단계가 약하면 재사용 이미지나 시간 조작에 취약해진다.

**What:**  
Exif 추출, 촬영 시각 비교, 성공/실패 사유 분류, 인증 결과 저장 로직을 구현한다. Exif 없음과 촬영 시각 불일치를 구분해 기록한다.

**Acceptance Criteria:**

- 업로드 이미지에서 Exif 읽기를 시도한다.
- Exif 없음과 촬영 시각 불일치가 구분되어 실패 저장된다.
- 성공한 인증만 집계 대상으로 표시된다.
- 실패 사유가 사용자 노출용 메시지와 연결된다.

### T-14. 인증 업로드/결과 UI 구현

**Why:**  
인증은 사용자에게 가장 자주 반복되는 행동이다. 업로드 성공, 검증 중, 실패 사유를 즉시 이해할 수 있어야 이탈이 줄어든다.

**What:**  
인증 업로드 버튼, 진행 상태, 성공/실패 결과, 실패 사유 노출 UI를 구현한다. 반복 인증 흐름이 모바일 기준으로도 무리 없게 설계한다.

**Acceptance Criteria:**

- 사용자는 허용된 화면에서 인증 이미지를 업로드할 수 있다.
- 업로드 후 현재 처리 상태를 확인할 수 있다.
- 실패 시 이해 가능한 사유가 표시된다.
- 성공한 인증과 실패한 인증이 시각적으로 구분된다.
- `GET /api/rooms/{roomId}/mission-logs/me` 기반으로 내 인증 기록 조회 화면/흐름이 동작하고, 성공/실패/처리 중 기록을 API 계약에 맞게 표시한다.

### T-15. 가입-입장-인증 회귀 테스트 정의

**Why:**  
핵심 루프의 초반부는 이후 정산 품질의 전제다. 이 구간을 수동 감으로만 검증하면 후반부 버그가 어디서 시작됐는지 찾기 어려워진다.

**What:**  
가입, 로그인, 크루 생성, 입장, 인증 제출, Exif 실패/성공 케이스를 검증하는 회귀 테스트 시나리오를 정의한다. 자동화 대상과 수동 점검 항목을 함께 구분한다.

**Acceptance Criteria:**

- 핵심 가입-입장-인증 흐름의 테스트 시나리오가 문서화된다.
- Exif 없음, 시간 불일치, 비회원 제출 케이스가 포함된다.
- 공개/비공개 입장 경로가 모두 테스트 대상에 포함된다.
- 자동화 우선순위와 수동 점검 항목이 분리된다.

### T-16. 지분율 집계 조회 모델 구현

**Why:**  
대시보드는 정산 전에도 현재 상태를 보여줘야 한다. 사용자별 성공 횟수와 크루 전체 성공 횟수를 빠르게 읽되, 최종 정산 결과와 혼동되지 않는 `GET /api/rooms/{roomId}/dashboard` projection 계약이 먼저 필요하다.

**What:**  
`MissionLog`와 참여자 상태를 기준으로 사용자 raw 성공 횟수, 추정 인정 성공 횟수, 크루 전체 추정 인정 성공 합계, 예상 환급금, 예상 손익, 추정 순위를 반환하는 Dashboard API 조회 모델을 구현한다. 이 조회 모델은 query-time deterministic estimated projection이며, `MissionLog.server_time`과 `Asia/Seoul` 기준으로 API spec의 representative rule을 적용한다. `point_history`를 projection source로 사용하지 않고, snapshot/cache/aggregate table이나 Redis leaderboard에 projection 값을 source of truth처럼 저장하지 않는다. 전체 추정 인정 성공 횟수 `0`인 경우도 균등 환급 base estimate 또는 projection input 부족 notice로 처리한다.

**Acceptance Criteria:**

- `GET /api/rooms/{roomId}/dashboard`가 API spec의 `projection_status`, `projection_notice`, 필드 nullability 계약을 따른다.
- 사용자별 raw 성공 횟수와 추정 인정 성공 횟수를 구분해 반환한다.
- `DAILY`, `SPECIFIC_DAYS`, `WEEKLY_N` 추정 인정 성공 횟수가 `MissionLog.server_time ASC`, 동률 `MissionLog.id ASC` representative rule로 계산된다.
- 크루 전체 추정 인정 성공 횟수 합계를 조회할 수 있다.
- 예상 환급금은 정산 remainder/draw를 포함하지 않는 deterministic base estimate로 계산된다.
- 전체 추정 인정 성공 횟수 `0`인 경우에도 오류 없이 결과를 반환한다.
- `CLOSED` + not `SUCCEEDED` 상태는 저장된 snapshot이 아니라 `room.end_at` cutoff 기반 `FROZEN` projection으로 재계산된다.
- `Settlement.status = SUCCEEDED` 이후에는 최종값을 복제하지 않고 `settlement_id`로 Settlement API 조회를 유도한다.

### T-17. 정산 규칙 엔진 구현

**Why:**  
정산 로직은 제품 신뢰의 핵심이다. 계산 규칙을 API나 배치 흐름에 섞지 말고 독립 엔진으로 두는 편이 테스트와 변경 대응에 유리하다.

**What:**  
지분율 계산, `BigDecimal` 처리, `원 단위 절사`, `잔액 1위 지급`, `동점 시 성공 횟수 우선 -> 그래도 같으면 랜덤`, `전체 성공 횟수 0`, `중도 탈퇴` 규칙을 구현한다.

**Acceptance Criteria:**

- 지분율과 환급액 계산이 독립 함수 또는 서비스로 동작한다.
- 전체 성공 횟수 `0` 규칙이 정확히 적용된다.
- 절사 후 잔액과 동점 처리 규칙이 반영된다.
- 중도 탈퇴자의 인정 성공 횟수 기준 계산이 가능하다.

### T-18. 정산 배치, 시작 전 취소, 재시도, 멱등 처리 구현

**Why:**  
정산 엔진만 있어도 운영 가능한 서비스는 아니다. 종료 시점 자동 실행, 실패 재시도, 중복 실행 방지가 있어야 실제 시스템이 된다.

**What:**  
`익일 새벽` 일반 정산 배치, `start_at` 만료 미시작 방의 `RECRUITING -> CANCELLED` 취소 batch, 실패 `3회` 재시도, 동일 미션/취소 정산 멱등 처리, 어드민 확인 대상 상태 저장을 구현한다. 필요 시 락/큐/상태 전이를 포함한다.

**Acceptance Criteria:**

- 종료된 미션에 대해 정산 배치가 자동 실행된다.
- `start_at`까지 시작되지 않은 `RECRUITING` 방은 `CANCELLED` 처리되고 취소형 정산이 생성된다.
- `StartRoom`과 취소 batch가 경합해도 하나의 조건부 전이만 성공한다.
- 동일 미션 정산과 취소형 정산이 중복 반영되지 않는다.
- 실패 시 최대 `3회` 재시도 후 운영 확인 상태로 남는다.
- 배치 실행 상태가 추적 가능하게 저장된다.

### T-19. 환급 반영과 정산 히스토리 API 구현

**Why:**  
정산 계산이 맞아도 실제 잔액에 반영되지 않으면 사용자에게는 실패다. 결과 금액 반영과 조회 API는 같은 수준으로 중요하다.

**What:**  
정산 결과를 포인트 잔액과 `Point_History`에 반영하고, 사용자별 정산 결과와 환급 이력을 조회하는 API를 구현한다.

**Acceptance Criteria:**

- 정산 완료 후 환급액이 포인트 잔액에 반영된다.
- 환급 내역이 `Point_History`에 기록된다.
- 사용자는 자신의 정산 결과를 API로 조회할 수 있다.
- 인원 미달 취소 또는 균등 환급 결과도 같은 구조로 조회된다.

### T-20. 실시간 지분율 대시보드 UI 구현

**Why:**  
갓세이빙의 차별점은 결과를 끝난 뒤가 아니라 진행 중에도 보여준다는 점이다. 대시보드가 약하면 제품의 핵심 가치가 사용자에게 전달되지 않는다. 다만 화면은 Dashboard 값을 최종 정산 결과가 아니라 현재 기준 estimated projection으로 일관되게 표현해야 한다.

**What:**  
`GET /api/rooms/{roomId}/dashboard`를 소비해 내 성공 횟수, 추정 인정 성공 횟수, 전체 추정 인정 성공 횟수, 현재 기준 추정 지분율, 예상 환급금, 예상 손익, 추정 수행 순위를 보여주는 대시보드 UI를 구현한다. 전체 추정 인정 성공 `0` 상태도 설명 문구와 함께 처리한다. FE는 Dashboard 값을 최종 정산값처럼 재계산하거나, client-side settlement/remainder/draw 계산을 수행하지 않는다.

**Acceptance Criteria:**

- 사용자는 내 raw 성공 횟수와 추정 인정 성공 횟수를 구분해 볼 수 있다.
- 현재 기준 추정 지분율, 예상 환급금, 예상 손익이 “예상”, “현재 기준”, “추정” 라벨로 표시된다.
- 예상 환급금은 최종 환급금이 아니라 base estimate이며 최종 정산 시 1원 단위 차이가 날 수 있음을 오해 없이 안내한다.
- 전체 추정 인정 성공 횟수 `0`인 경우에도 UI가 정상 동작한다.
- `rank_estimated`는 예상 수익/환급금 순위가 아니라 추정 수행 순위로 표시된다.
- `projection_status = SETTLEMENT_SUCCEEDED`이면 `settlement_id`로 Settlement API를 조회해 최종 결과 화면으로 연결한다.

### T-21. 정산 결과/포인트 히스토리 화면 구현

**Why:**  
정산 결과는 계산보다 설명이 중요하다. 사용자가 왜 이 금액을 받았는지 이해하지 못하면 신뢰가 생기지 않는다.

**What:**  
정산 완료 화면, 환급 요약, 성공 횟수, 지분율, 포인트 이동 이력 UI를 구현한다. 예외 케이스 결과도 읽을 수 있게 정리한다.

**Acceptance Criteria:**

- 사용자는 정산 후 환급 결과 요약을 볼 수 있다.
- 성공 횟수, 지분율, 환급액이 함께 표시된다.
- 관련 포인트 이동 이력을 같은 맥락에서 확인할 수 있다.
- 예외 환급 케이스도 별도 오류 없이 표시된다.

### T-22. 정산/시작 lifecycle 골든 데이터 테스트 작성

**Why:**  
정산은 눈으로 보기엔 맞아도 금액 오차가 숨어들기 쉽다. 고정 입력과 기대 결과를 가진 골든 데이터 세트가 있어야 회귀를 막을 수 있다.

**What:**  
정산 예시 데이터와 기대 환급 결과를 만드는 테스트 세트를 작성한다. `전체 성공 횟수 0`, `잔액 발생`, `1위 동점`, `중도 탈퇴`, `인원 미달`, `start_at` 만료 미시작 취소, `StartRoom`/취소 batch 경합 케이스를 포함한다. Dashboard projection 회귀 fixture는 최종 정산 골든 데이터와 구분해, query-time estimated projection과 final settlement의 차이를 함께 검증한다.

**Acceptance Criteria:**

- 핵심 예외 케이스가 골든 데이터 테스트로 정의된다.
- 기대 환급 결과가 수치로 고정된다.
- Dashboard projection fixture가 `DAILY` duplicate, `SPECIFIC_DAYS` invalid weekday, `WEEKLY_N` overflow, `withdrawn_at` cutoff, `activated_at` 이전 `BEFORE_START`, `CLOSED` cutoff를 포함한다.
- stale reference 검색에서 post-activation eligibility가 `room.start_at`이 아니라 `room.activated_at`을 기준으로 문서화됐는지 확인한다.
- Dashboard projection fixture가 `rank_estimated` 동률 시 `participant_id ASC`, zero-total equal-share base estimate, base estimate와 final settlement remainder/draw 차이를 포함한다.
- 배치 또는 엔진 변경 후 같은 결과를 재검증할 수 있다.
- 금액 오차 또는 규칙 누락을 빠르게 탐지할 수 있다.

### T-23. SSE 이벤트 발행과 구독 인증 구현

**Why:**  
인앱 알림은 실시간성이 중요하지만 핵심 로직을 막아서는 안 된다. 이벤트 발행 구조를 분리하면 실패 격리가 쉬워진다.

**What:**  
인증 결과, 지분 변화 등 알림 이벤트 스키마와 SSE 발행 엔드포인트, 사용자별 구독 인증을 구현한다. SSE emitter registry key와 notification/event routing key는 `member.uuid`를 사용한다. `email`은 로그인 식별자/연락처일 뿐이며 routing key나 stream identifier로 사용하지 않는다.

**Acceptance Criteria:**

- 인증된 사용자만 자신의 알림 스트림에 연결할 수 있다.
- JWT `sub(member.uuid)` 기준으로 현재 사용자 emitter registry에 연결된다.
- 인증 결과와 지분 변화 이벤트가 `member.uuid` recipient routing으로 발행된다.
- 이벤트 payload는 최소 `eventId`, `eventType`, `occurredAt`, `resourceType`, `resourceId`, `message`, `severity`, `uiHint`를 포함한다.
- 같은 이벤트가 중복 발행되지 않도록 제어하고, FE가 `eventId`로 동일 세션 duplicate toast를 방지할 수 있어야 한다.
- SSE 실패가 핵심 도메인 트랜잭션을 롤백시키지 않는다.

### T-24. 인앱 알림 UI와 SSE 클라이언트 구현

**Why:**  
서버 이벤트만 있어서는 사용자가 변화를 체감하지 못한다. 이벤트를 실제 UI 신호로 연결해야 알림 기능이 의미를 가진다.

**What:**  
알림 수신 클라이언트와 toast UI를 구현하고, 이벤트 수신 시 관련 화면의 refetch/invalidate를 trigger한다. badge/count는 best-effort UX projection으로만 다룬다. 연결 실패 시 사용자를 방해하지 않는 조용한 재구독 전략을 최소 수준으로 넣는다.

**Acceptance Criteria:**

- 사용자는 인증 결과 알림을 toast 등 UI에서 받을 수 있다.
- 지분 변화 알림 수신 시 관련 화면이 refetch/invalidate되어 최신 상태 확인이 가능하다.
- 같은 `eventId`에 대해 동일 브라우저 세션에서 duplicate toast가 반복 표시되지 않는다.
- 연결이 끊겨도 사용자에게 불필요한 오류 알림 없이 조용히 재연결을 시도한다.
- 여러 탭에서 duplicate toast가 발생할 수 있음은 known risk로 다루며, BroadcastChannel/localStorage 기반 완화는 선택 사항이다.
- 알림이 없어도 기존 화면 동작은 유지되고 DB/API state로 같은 상태를 확인할 수 있다.

### T-25. 정산 완료 이메일 발송 구현

**Why:**  
이메일은 정산 완료를 앱 밖으로 전달하는 수단이다. 재방문 유도와 결과 전달 품질을 높이되, 핵심 정산 흐름과 결합되면 안 된다.

**What:**  
정산 완료 후 이메일 발송 트리거, 템플릿, 중복 발송 방지, 실패 로깅을 구현한다.

**Acceptance Criteria:**

- 정산 완료 후 결과 안내 이메일 발송이 시도된다.
- 메일 내용에 크루명, 성공 횟수, 환급 결과가 포함된다.
- 동일 정산 건에 대해 중복 발송되지 않는다.
- 발송 실패가 정산 완료 상태를 바꾸지 않는다.

### T-26. AI 미션 추천 API 연동 구현

**Why:**  
크루 생성의 진입 장벽을 낮추려면 AI가 구조화된 초안을 줘야 한다. 이 티켓은 `FR-Required / Non-transactional` 범위로 첫 릴리스에 포함하되, 기본 생성 흐름을 막지 않는 비트랜잭션성 기능으로 유지해야 한다.

**What:**  
AI 프롬프트/응답 스키마, 제목/설명/기간/인증 주기/보증금 추천 응답 정규화, 실패 처리와 API error shape를 구현한다.

**Acceptance Criteria:**

- 사용자가 미션 추천 요청을 보낼 수 있다.
- AI 응답이 구조화된 추천 값으로 정리된다.
- 규칙에 맞지 않는 응답은 저장 전에 걸러진다.
- `AI_RECOMMENDATION_FAILED`, `AI_RESPONSE_INVALID`, `VALIDATION_ERROR` 또는 이에 준하는 오류 형태가 구분된다.
- AI 실패 시 수동 생성 흐름은 그대로 동작한다.

### T-27. AI 미션 추천 UI와 폼 반영 구현

**Why:**  
AI 추천이 있어도 사용자가 바로 고치고 채택할 수 없으면 가치가 반감된다. 추천 결과를 폼으로 자연스럽게 옮기는 UX가 필요하다. 이 티켓은 `FR-Required / Non-transactional` 범위이며, AI 실패가 사용자의 기존 입력을 잃게 하면 안 된다.

**What:**  
크루 생성 화면에서 AI 추천 요청, 결과 미리보기, 값 적용/수정 UI를 구현한다.

**Acceptance Criteria:**

- 사용자는 생성 화면에서 AI 추천을 요청할 수 있다.
- 추천 결과를 보기 전에 현재 입력이 손실되지 않는다.
- 추천 값을 폼에 반영한 뒤 수정할 수 있다.
- AI 실패 시 명확한 실패 UI가 표시되고 수동 입력 UX가 깨지지 않는다.

### T-28. AI 습관 리포트 생성/저장 구현

**Why:**  
정산 이후에도 다시 돌아오게 만들려면 회고 자산이 필요하다. 리포트 생성은 `FR-Required / Non-transactional` 범위의 첫 릴리스 기능이지만, 정산 후 비동기 보조 작업으로 분리하는 편이 안전하다.

**What:**  
정산 데이터 기반 리포트 입력 생성, AI 호출, 결과 저장, 실패 상태 기록, `settlement_id + member_id` 기준 멱등성을 구현한다.

**Acceptance Criteria:**

- 정산 완료 사용자에 대해서만 리포트 생성이 가능하다.
- 리포트는 실제 인증/정산 데이터를 기반으로 생성된다.
- 생성 결과가 저장되어 재조회 가능하다.
- 같은 정산/회원 조합의 중복 생성이 방지된다.
- AI 실패가 정산 완료 상태, 환급 결과, 포인트 원장 상태를 변경하지 않는다.

### T-29. AI 습관 리포트 조회 UI 구현

**Why:**  
리포트는 저장만으로 끝나지 않는다. 사용자가 이해하기 쉬운 화면으로 보여줘야 재참여 동기라는 목적을 달성할 수 있다. 이 티켓은 `FR-Required / Non-transactional` 범위이며, 리포트 실패 상태에서도 정산 결과 화면은 계속 사용할 수 있어야 한다.

**What:**  
마이페이지 또는 크루 결과 화면에서 리포트 조회 UI를 구현한다. 생성 전/생성 실패 상태도 구분해 표현한다.

**Acceptance Criteria:**

- 사용자는 생성된 개인 리포트를 화면에서 볼 수 있다.
- 아직 생성되지 않았거나 `PENDING`, `FAILED`, `SUCCEEDED` 상태가 구분된다.
- 리포트는 미션 종료 후에만 노출된다.
- 결과 화면과 자연스럽게 연결된다.
- 리포트 실패 상태에서도 기존 정산 결과와 포인트 히스토리 조회는 유지된다.

### T-30. 관리자 정산 상태 조회 API 구현

**Why:**  
운영자가 실패 배치를 찾지 못하면 수동 대응도 할 수 없다. 사용자용 결과 API와 별개로 운영용 상태 조회 API가 필요하다.

**What:**  
미션별 정산 상태, 재시도 횟수, 오류 요약, 확인 필요 상태를 조회하는 관리자 API를 구현한다.

**Acceptance Criteria:**

- 관리자 권한이 있는 사용자만 API를 호출할 수 있다.
- 미션별 배치 상태와 재시도 횟수를 조회할 수 있다.
- 실패 원인 요약 또는 최근 오류 정보가 포함된다.
- 운영 확인이 필요한 건을 구분해 반환한다.

### T-31. 관리자 정산 모니터링 화면 구현 — Deferred / MVP 제외

**Why:**  
MVP에서는 운영 복구를 관리자 API, CloudWatch 알람, structured log, runbook으로 수행한다. 별도 Admin UI는 FE 범위를 넓히므로 MVP 이후로 미룬다.

**What:**  
MVP에서는 관리자 화면을 구현하지 않는다. 운영자는 아래 수단으로 정산 실패와 복구를 처리한다.

- `GET /api/admin/settlements`
- `POST /api/admin/settlements/{settlementId}/retry`
- CloudWatch alarm/log
- `docs/runbooks/settlement-recovery.md`

**Acceptance Criteria:**

- Admin UI 없이 실패 정산을 식별할 수 있다.
- Admin UI 없이 특정 settlement를 재시도할 수 있다.
- 운영자가 DB를 직접 수정하지 않아도 runbook과 관리자 API로 복구할 수 있다.

### T-32. 배치/정산 운영 모니터링과 알림 구성

**Why:**  
운영 화면만으로는 늦을 수 있다. 배치 실패나 비정상 지연을 시스템 수준에서 감지해야 실제 서비스 운영이 가능하다.

**What:**  
배치 실행 상태, 실패율, 재시도, 지연 시간에 대한 로그/메트릭/알림을 구성한다. CloudWatch 등 운영 도구에 연결할 수 있는 형태를 만든다.

**Acceptance Criteria:**

- 정산 배치 상태를 메트릭 또는 로그로 추적할 수 있다.
- 실패 또는 과도한 지연 시 운영 알림이 발생한다.
- 재시도 횟수와 최종 실패 건을 구분해 볼 수 있다.
- 운영자가 관리자 API, runbook, CloudWatch/log에서 상태를 확인할 수 있다.
- CloudWatch 최소 알람은 `settlement batch failure`, `RUNNING timeout`, `RETRY_WAIT 증가`, `DB connection failure`, `Redis unavailable`, `payment confirm failure`, `idempotency conflict`, `reconciliation mismatch`, `disk usage`를 포함한다.
- Kubernetes, MSA, full Terraform, blue-green/canary, full observability stack, full OpenTelemetry는 MVP 범위에 포함하지 않는다.

### T-33. 인증 피드 조회 API 구현

**Why:**  
사용자가 방 내 인증 활동을 소셜 피드 형태로 확인하고, 자신의 참여 상태를 직관적으로 이해할 수 있어야 한다.

**What:**  
`GET /api/rooms/{roomId}/feed` API를 구현한다.  
성공 인증(`mission_log.is_success = true`)만 `feed_items`로 반환하고,  
`day_statuses` 또는 `participant_day_slots`를 통해 일자별 상태(SUCCESS / FAILED / NOT_SUBMITTED)를 별도로 제공한다.

**Acceptance Criteria:**

- 성공 인증 로그만 feed_items에 포함된다.
- FAILED / NOT_SUBMITTED 상태는 feed_items가 아니라 day_statuses로 표현된다.
- feed_items만으로 일자 상태를 유추할 수 없다.
- 동일 조건에서 항상 동일한 응답을 반환한다.

### T-34. 인증 피드 리액션 API 구현

**Why:**  
사용자가 다른 참여자의 인증에 반응하면서 소셜 참여감을 느끼고 지속적인 행동 동기를 얻을 수 있어야 한다.

**What:**  
`POST /api/mission-logs/{missionLogId}/reactions` 및  
`DELETE /api/mission-logs/{missionLogId}/reactions/me` API를 구현한다.  
리액션은 `(mission_log_id, member_id)` 기준 멱등 upsert로 처리하며,  
DB-level upsert를 사용해 동시 요청에서도 오류 없이 일관된 상태로 수렴해야 한다. MySQL 8.0 구현에서는 unique key 기반 `INSERT ... ON DUPLICATE KEY UPDATE` 같은 stack-compatible upsert를 사용한다.

**Acceptance Criteria:**

- 같은 사용자가 동일 로그에 대해 하나의 리액션만 가진다.
- POST는 upsert로 동작하며 기존 리액션을 교체한다.
- DELETE는 멱등하게 동작한다.
- 동시 요청에서도 unique constraint 오류가 발생하지 않는다.
- 리액션은 정산, 포인트, 상태 계산에 영향을 주지 않는다.
