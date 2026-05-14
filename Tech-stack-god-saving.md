# 기술 스택 정리: 갓세이빙 MVP

## 1. 문서 목적

이 문서는 갓세이빙 MVP의 기술 스택을 빠르게 파악하기 위한 요약 문서다. 구현 기준을 새로 정의하지 않고, 이미 확정된 Markdown source of truth 문서에서 기술 선택과 운영 원칙을 모아 정리한다.

우선순위는 아래를 따른다.

1. `PRD-god-saving.md`의 `7.3 Technology`: 기술 스택의 1차 기준
2. `API-spec-god-saving.md`: FE/BE API 계약과 공통 데이터 표현 기준
3. `ERD-god-saving.md`: 데이터 구조, DB 제약, 원장/스냅샷 기준
4. `Settlement-design.md`: 정산, 배치, 락, 멱등성, 운영 복구 기준
5. `MVP-backlog-user-stories.md`, `MVP-ticket-breakdown.md`: 구현 단위와 검증 범위

이 문서와 원문 source of truth 문서가 충돌하면 원문 source of truth 문서를 우선한다.

## 2. 확정 기술 스택

| 영역 | 선택 기술 | 상태 | 기준 문서 |
| --- | --- | --- | --- |
| Backend | Java 17, Spring Boot 3.2 | 확정 | `PRD-god-saving.md` |
| Database | MySQL 8.0 | 확정 | `PRD-god-saving.md`, `ERD-god-saving.md` |
| Cache / Lock | Redis, Redisson | 확정. 단, 최종 정합성 수단이 아니라 보조 락/캐시 | `PRD-god-saving.md`, `Settlement-design.md` |
| Batch | Spring Batch 5.x | 확정 | `PRD-god-saving.md`, `Settlement-design.md` |
| File Storage | AWS S3 | 확정. MVP는 private bucket + server-generated key + presigned URL 업로드를 사용한다 | `PRD-god-saving.md`, `API-spec-god-saving.md` |
| Frontend | React, Vite, Axios | 확정. CSS, 상태관리, 테스트 러너는 미정 | `PRD-god-saving.md`, `ADR-mvp-tech-architecture.md` |
| Notification | SSE, Email | 확정. SSE는 best-effort realtime UX delivery이며 Email은 SMTP 기반 보조 알림이다. 둘 다 실패해도 인증/정산/포인트/결제 트랜잭션을 롤백하지 않는다 | `PRD-god-saving.md`, `API-spec-god-saving.md`, `Settlement-design.md` |
| Payment | 토스페이먼츠 샌드박스 | 확정. MVP는 confirm-only 흐름이며 API의 `payment_id`는 Toss `paymentKey`를 의미한다 | `PRD-god-saving.md`, `API-spec-god-saving.md` |
| AI | Claude API | 확정 | `PRD-god-saving.md`, `API-spec-god-saving.md` |
| Infra | AWS EC2, Docker Compose, Nginx, GitHub Actions, CloudWatch | 확정. Docker Compose는 단일 서버 재현성과 배포 단순화를 위한 도구이며 orchestration platform이 아니다 | `PRD-god-saving.md`, `ADR-mvp-tech-architecture.md` |
| API Contract | REST + JSON | 확정 | `API-spec-god-saving.md` |
| Auth | JWT Bearer token | 확정 | `PRD-god-saving.md`, `API-spec-god-saving.md`, `ERD-god-saving.md` |

## 3. 아키텍처 운영 원칙

### 3.1 API와 데이터 표현

- API는 `REST + JSON` 기준으로 설계한다.
- 비즈니스 시간대는 `Asia/Seoul`로 고정한다.
- API 시각 값은 timezone offset이 포함된 `ISO-8601` 문자열로 주고받는다.
- 금액은 모두 `integer` 원 단위로 표현한다.
- 보증금은 별도 자산 이동이 아니라 포인트를 잠그는 `lock` 모델로 취급한다.

### 3.2 DB source of truth

- `point_history`는 포인트 금액의 source of truth다.
- `point_account.balance`는 `point_history`에서 재계산 가능한 현재값 캐시다.
- `settlement_item`은 참여자별 정산 계산 스냅샷의 기준이다.
- `Settlement.status = SUCCEEDED` 이후 운영, 분쟁, 조회 기준은 `settlement_item`과 연결된 `point_history`다.
- 실시간 지분율, 통계성 캐시, 대시보드 projection은 최종 정산 금액의 source of truth가 아니다.

### 3.3 정산, 락, 멱등성

- 정산 배치는 `Spring Batch` 기반으로 설계한다.
- 정산 실행권은 DB 조건부 update로 `PENDING/RETRY_WAIT -> RUNNING` claim에 성공한 워커가 가진다.
- Redis/Redisson 락은 배치 워커, 관리자 재시도, 복구 작업의 동시 실행을 줄이는 보조 안전장치다.
- 최종 방어선은 DB 조건부 claim, DB unique 제약, `point_history.idempotency_key`다.
- 정산 금액 계산은 Java `BigDecimal`과 `MathContext.DECIMAL128` 기준으로 수행하고, 최종 지급액은 원 단위로 절사한다.

### 3.4 파일, 알림, AI의 트랜잭션 경계

- 인증 이미지는 AWS S3에 저장한다.
- SSE와 Email은 사용자 알림 채널이며, 실패해도 인증/정산/포인트 원장 트랜잭션을 롤백시키지 않는다. SSE는 toast, 관련 화면 갱신, badge/count 같은 realtime UX 반응을 위한 best-effort delivery이고 DB/API state를 대체하지 않는다.
- Claude API 기반 AI 미션 추천과 AI 습관 리포트는 첫 릴리스 기능 gate를 통과해야 하지만, 실패해도 수동 방 생성, 정산 완료, 환급, 포인트 원장 흐름을 차단하지 않는다.
- AI 리포트는 정산 완료 데이터를 읽어 생성되는 후행 기능이며, 정산/환급/포인트 원장의 source of truth가 아니다.

### 3.5 Persistence 경계

- Spring Data JPA를 기본 persistence 방식으로 사용한다.
- 단순 CRUD는 Spring Data JPA Repository로 처리한다.
- QueryDSL은 복잡 조회, 운영 조회, 복구 조회, 동적 조건 조회에만 제한적으로 사용한다.
- QueryDSL projection은 settlement correctness 또는 source of truth가 아니다.
- 정산 금액 계산과 복구 판단 기준은 MySQL row, `settlement_item`, `point_history`, DB 제약, deterministic idempotency key다.

### 3.6 Identity / Auth / Realtime boundary

- `member.id`는 DB 내부 FK / join / persistence identity다.
- `member.uuid`는 external canonical identifier이며 JWT `sub`, API 사용자 식별자, SSE emitter registry key, notification/event routing key로 사용한다.
- `email`은 로그인 식별자와 연락처/사용자 정보이며, 변경 가능하고 PII이므로 routing key, stream identifier, notification recipient key, JWT subject로 사용하지 않는다.
- SSE는 state source가 아니라 REST refetch/invalidate를 유도하는 best-effort realtime signal이다.

### 3.7 Storage / Email / Payment 경계

- Presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- S3 object key는 서버가 생성하며 사용자는 임의 path/key를 지정할 수 없다.
- mission-log 생성 시 서버는 S3 object를 직접 조회해 존재 여부, size, content-type, ownership, EXIF를 검증한다.
- EXIF는 서버가 S3 object에서 추출/검증한 값이 기준이다.
- MVP 이메일은 SMTP 기반 보조 알림이며 structured log, bounded retry, 운영자 수동 재발송 수준으로 운영한다.
- notification log/outbox, notification inbox, cross-device unread sync, broker replay/fan-out은 MVP core 범위에서 제외한다.
- TossPayments 충전 API의 `payment_id`는 Toss `paymentKey`를 의미한다.
- `orderId`는 confirm 검증과 로그 상관관계 추적용이며 `point_history.idempotency_key` 구성값으로 사용하지 않는다.

## 4. 문서 기준 확정 항목과 후속 보강 항목

아래 항목은 deep-interview / ADR pressure-pass 이후 확정된 세부 결정과, 구현 전에 참조해야 할 소유 문서를 정리한다.

| 영역 | 결정/보강 항목 | 기준 문서 |
| --- | --- | --- |
| Frontend | CSS/UI 라이브러리 | 별도 Frontend ADR |
| Frontend | 상태관리 도구 | 별도 Frontend ADR |
| Frontend | 테스트 러너와 E2E 도구 | 별도 QA/Frontend ADR |
| Backend persistence | Spring Data JPA 기본 + QueryDSL 제한 도입 | `ADR-mvp-tech-architecture.md`, `Settlement-design.md` |
| Email | SMTP 기반 보조 알림. notification log/outbox는 MVP 제외 | `docs/runbooks/settlement-recovery.md` |
| Storage | S3 private bucket + server-generated key + presigned URL | `API-spec-god-saving.md` |
| Payment | Toss confirm-only. `payment_id` = Toss `paymentKey`, idempotency key = `charge:{paymentKey}` | `API-spec-god-saving.md`, `ERD-god-saving.md`, `Settlement-design.md` |
| Infra | EC2 단일 서버 + Docker Compose + Nginx + GitHub Actions + CloudWatch | `ADR-mvp-tech-architecture.md`, `docs/runbooks/settlement-recovery.md` |

문서에 없는 항목을 구현자가 임의로 확정하면 이후 PRD/API/ERD/Settlement 설계와 어긋날 수 있다. 구현 전에는 관련 source of truth 문서에 반영된 결정만 따른다.

## 5. 구현 핸드오프 메모

### Backend

- Java 17, Spring Boot 3.2, MySQL 8.0, Redis/Redisson, Spring Batch 5.x를 기본 전제로 둔다.
- 정산과 포인트 원장 구현은 `Settlement-design.md`와 `ERD-god-saving.md`를 먼저 확인한다.
- Redis/Redisson을 최종 정합성 수단으로 확대하지 않는다.
- 포인트 금액 정합성은 `point_history`, DB 제약, 멱등성 키를 기준으로 검증한다.

### Frontend

- React, Vite, Axios를 기본 전제로 둔다.
- 화면 구현은 `API-spec-god-saving.md`의 응답 projection과 상태값 계약을 따른다.
- Dashboard projection, feed success, settlement result의 의미를 혼동하지 않는다.
- CSS, 상태관리, 테스트 도구는 아직 확정되지 않았으므로 구현 전에 별도 결정이 필요하다.

### Infra / DevOps

- AWS EC2, Docker Compose, Nginx, GitHub Actions, CloudWatch를 MVP 운영 기본값으로 둔다.
- 배치 지연, 실패율, 재시도, 알림은 CloudWatch와 로그 기반으로 관찰 가능해야 한다.
- Docker Compose는 orchestration platform이 아니라 단일 서버 재현성과 배포 단순화를 위한 도구다. IaC, 네트워크, 환경 분리 전략은 MVP 이후 보강 대상으로 둔다.

## 6. ADR 요약

### Decision

갓세이빙 MVP의 문서 기준 기술 스택은 `PRD-god-saving.md`의 Technology 표를 따른다. 세부 구현 선택은 문서에 근거가 있는 경우에만 확정으로 표기하고, 문서에 없는 항목은 미정으로 분리한다.

### Drivers

1. 정산과 포인트 원장의 데이터 무결성이 가장 중요하다.
2. MVP는 빠르게 구현 가능해야 하지만 운영 복구와 재시도 가능성을 가져야 한다.
3. 기술 스택 문서는 구현자가 문서에 없는 세부 선택까지 확정된 것으로 오해하지 않게 해야 한다.

### Alternatives considered

- 모든 세부 구현 기술까지 임의로 채운다.
  - 기각 이유: `docs 기준`이라는 조건과 충돌하고, source of truth 문서에 없는 선택을 확정하게 된다.
- PRD Technology 표만 복사한다.
  - 기각 이유: 정산/포인트/락/멱등성 같은 운영상 중요한 제약을 놓칠 수 있다.

### Consequences

- 구현자는 확정 스택과 미정 항목을 분리해서 볼 수 있다.
- 미정 항목은 구현 전에 ADR 또는 원문 source of truth 문서 업데이트가 필요하다.
- 기술 스택이 변경되면 먼저 소유 문서를 수정하고, 이 요약 문서를 함께 갱신해야 한다.
