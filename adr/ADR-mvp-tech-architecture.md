# ADR: 갓세이빙 MVP 기술 아키텍처 — 복구 가능한 단순함

- 상태: Accepted for MVP
- 작성일: 2026-05-13
- 기준 문서:
  - `docs/PRD-god-saving.md`
  - `docs/Tech-stack-god-saving.md`
  - `docs/Settlement-design.md`
  - `docs/ERD-god-saving.md`
  - `.omx/specs/deep-interview-tech-stack-validation.md`

## 1. 문서 목적

이 문서는 갓세이빙 MVP의 기술 스택을 단순히 나열하지 않는다. 각 기술을 왜 선택하는지, 어떤 문제를 해결하는지, 어디까지 사용할지, 무엇을 하지 않을지, 장애 시 어떤 기준으로 복구할지를 정한다.

갓세이빙은 습관/미션 서비스처럼 보이지만, MVP의 가장 큰 기술 리스크는 **포인트 보증금 lock, 정산, 결제, 멱등성, partial recovery**다. 따라서 이 문서의 성공 기준은 최신 기술 도입 수가 아니라 다음 질문에 답할 수 있는지다.

- 중복 결제/중복 정산/부분 실패가 발생해도 금전성 데이터가 깨지지 않는가?
- 장애 후 어떤 테이블을 기준으로 복구할 수 있는가?
- 운영자가 직접 DB를 고치지 않고도 복구 절차를 실행할 수 있는가?
- 팀이 MVP 기간 안에 구현하고 운영할 수 있는 수준인가?

## 2. 최종 결정 요약

갓세이빙 MVP는 다음 아키텍처 결정을 따른다.

> **MySQL-first recoverable MVP architecture with assistive Redis and bounded Batch.**

한국어로는 다음 역할 분리를 핵심 원칙으로 고정한다.

| 구성요소           | 역할                       |
| ------------------ | -------------------------- |
| **MySQL**          | 최종 정합성 및 복구 기준   |
| **Redis/Redisson** | 운영/성능/동시성 보조 계층 |
| **Spring Batch**   | 정산/재시도/복구 실행 계층 |

핵심 원칙은 다음과 같다.

1. `point_history` append-only ledger가 포인트 금액의 source of truth다.
2. `point_account.balance`는 `point_history`에서 재계산 가능한 파생 상태다.
3. `settlement_item`은 참여자별 정산 계산 스냅샷이며, 성공 정산 이후 운영/분쟁/조회 기준이다.
4. Redis/Redisson은 source of truth가 아니다.
5. unique constraint, conditional update, transaction, `point_history.idempotency_key`가 최종 방어선이다.
6. Spring Batch는 settlement/retry/reconciliation/partial recovery 중심으로 제한한다.
7. Batch/Redis를 전 영역에 남발하지 않는다.
8. 운영 복구는 DB 직접 수정이 아니라 관리자 API 또는 batch trigger를 통해 수행한다.

## 3. 결정 동인

| 우선순위 | 결정 동인        | 의미                                                                                    |
| -------: | ---------------- | --------------------------------------------------------------------------------------- |
|        1 | 데이터 무결성    | 포인트/정산/결제는 중복 반영되면 안 된다.                                               |
|        2 | 정산 안정성      | 배치 실패, 서버 재시작, 재시도 후에도 participant 단위로 안전하게 이어서 처리해야 한다. |
|        3 | 운영 복구 가능성 | 불일치를 완전히 없애기보다, 발생 시 추적·차단·복구 가능해야 한다.                       |
|        4 | MVP 구현 가능성  | 작은 팀이 학습하면서도 일정 안에 완성 가능한 수준이어야 한다.                           |
|        5 | 낮은 운영 복잡도 | Kubernetes/MSA/full observability 같은 운영 부담은 MVP 범위 밖이다.                     |
|        6 | 성장 가능성      | Redis/Batch/QueryDSL은 실제 문제 해결 범위에서 경험하되, 정합성 책임을 넘기지 않는다.   |

## 4. 기술별 ADR

각 기술은 다음 기준으로 판단한다.

- 왜 선택했는가
- 어떤 문제를 해결하는가
- 어디까지 사용할 것인가
- 어디까지는 하지 않을 것인가
- 장애/실패 시 무엇을 기준으로 복구하는가
- 일정이 밀릴 때 어떻게 줄일 수 있는가

### 4.1 Spring Boot / JPA / MySQL

#### Decision

Backend는 Java 17, Spring Boot 3.2, JPA를 사용하고, Database는 MySQL 8을 최종 정합성 기준으로 사용한다.

#### Why

- 트랜잭션, unique constraint, conditional update를 통해 금전성 데이터의 최종 방어선을 구현하기 좋다.
- 팀이 Spring Boot/JPA 기본 서비스 구현 경험을 가지고 있다.
- 정산/포인트/결제는 Redis나 Batch보다 먼저 DB에서 불변조건을 강제해야 한다.

#### Solves

- 포인트 변동의 append-only ledger 저장
- 보증금 lock의 중복 차감 방지
- settlement claim의 단일 실행권 보장
- idempotency key 기반 중복 반영 방지
- 장애 후 `point_history`, `settlement_item`, `Settlement.status` 기준 복구

#### Allowed Use

- `point_history.idempotency_key` unique constraint
- `Settlement(PENDING/RETRY_WAIT -> RUNNING)` 조건부 claim
- 보증금 lock 시 잔액 조건부 update
- `settlement_item.point_history_id`와 `point_history` 연결 검증
- `point_account.balance` reconciliation

#### Forbidden Use

- 운영자가 `point_account.balance`를 직접 수정하는 방식의 정상 복구
- 운영자가 `point_history` row를 직접 수정/삭제하는 방식의 정상 복구
- Redis lock 성공 여부를 DB 상태보다 우선하는 정산 판단
- parent `Settlement.status`만 보고 participant별 지급 완료를 단정하는 처리

#### Failure / Recovery Basis

- 최종 복구 기준은 MySQL row다.
- `point_history`는 포인트 금액 source of truth다.
- `settlement_item`은 정산 계산 결과와 participant별 지급 연결의 기준이다.
- `Settlement.status = SUCCEEDED`는 모든 `settlement_item`이 유효한 `point_history`를 가리킬 때만 허용한다.

#### Schedule Pressure Cut

줄이면 안 된다. DB 불변조건, idempotency, conditional update, transaction은 MVP 핵심이다.

---

### 4.2 Redis / Redisson

#### Decision

Redis/Redisson은 운영/성능/동시성 보조 계층으로 사용한다. Redis는 source of truth가 아니며, Redis 장애 시에도 핵심 정산/복구 플로우는 DB 기준으로 복구 가능해야 한다.

#### Why

- 정산 중복 실행, 동시 요청, rate limiting, projection cache 같은 운영/성능 문제를 줄일 수 있다.
- Redisson distributed lock은 중복 실행 가능성을 낮추는 보조 안전장치로 유용하다.
- 팀이 Redis/Redisson을 실제 문제 해결 범위 안에서 경험할 수 있다.

#### Solves

- 같은 방 정산 중복 실행 억제
- 보증금 lock/인증 요청 등의 동시성 보조
- rate limiting
- dashboard/projection cache
- idempotency cache 보조
- SSE/event 보조

#### Allowed Use

- `lock:settlement:room:{roomId}` 같은 짧은 TTL의 settlement duplicate execution guard
- 동시 인증 요청 제어
- rate limiting
- projection/dashboard cache
- DB idempotency를 보조하는 Redis cache
- SSE/event의 비최종 알림 보조

#### Forbidden Use

- 포인트 ledger
- 최종 지급/정산 상태 판단
- 유일한 멱등성 방어선
- Redis 상태만으로 `Settlement.status` 변경
- Redis lock 획득만으로 금전성 operation 성공 처리
- Redis Stream/Kafka 대체 구조를 MVP에 도입

#### Failure / Recovery Basis

Redis 장애 시에도 금전성 정합성은 다음 MySQL 기준으로 보존한다.

- `Settlement.status` 조건부 claim
- `batch_run_key`
- DB transaction
- unique constraint
- `point_history.idempotency_key`
- `settlement_item.point_history_id` 연결 검증

#### Schedule Pressure Cut

일정이 밀리면 다음 순서로 축소한다.

1. SSE/event Redis 보조 구조
2. dashboard/projection cache
3. Redis idempotency cache
4. 일부 rate limit 고도화

단, DB-backed settlement duplicate/concurrency guard로서 Redisson 경험은 MVP 유지 우선순위가 높다. 다만 항상 DB fallback을 문서화해야 한다.

#### SSE / Realtime UX Notification Boundary

SSE notification은 MVP에서 낮은 복잡도로 server-to-client realtime UX signal을 제공하기 위해 선택한다. SSE 성공 여부는 인증, 정산, 포인트 원장, 결제 transaction의 성공 조건이 아니며, REST/DB state가 source of truth다.

Durable broker, outbox, replay, notification inbox, unread sync는 MVP에서 제외한다. 현재 제품 요구는 영속 알림 상태가 아니라 중요한 상태 변화의 best-effort 인지이며, 누락 복구는 API 조회로 충분하다. SSE 외부 계약과 identity 사용 방식은 `API-spec-god-saving.md`가 소유한다.

---

### 4.3 Spring Batch

#### Decision

Spring Batch 5는 정산/재시도/reconciliation/partial recovery 실행 계층으로 사용한다.

#### Why

- 갓세이빙의 정산은 단순 CRUD가 아니라 종료 시점 batch, participant 단위 지급, partial failure, retry, reconciliation이 필요하다.
- 배치 실행 이력, 재시도, 실패 분류, chunk/step/job 구조를 통해 복구 가능한 운영 흐름을 만들 수 있다.
- 팀에게는 신규 학습 영역이지만, 학습 목적이 실제 정산/복구 문제와 직접 연결된다.

#### Solves

- settlement batch
- retry/recovery
- reconciliation
- partial failure recovery
- scheduler 기반 운영 플로우
- RUNNING timeout 복구
- 관리자 batch trigger 기반 복구

#### Allowed Use

- 종료/취소된 방의 settlement 생성/claim/실행
- `RETRY_WAIT` 정산 재시도
- `point_history` 기준 `point_account.balance` reconciliation
- 일부 participant만 성공한 정산의 이어서 처리
- 실패 코드별 재시도/FAILED 전환
- 관리자 복구 API가 호출하는 batch trigger

#### Forbidden Use

- 모든 비동기 작업의 기본 처리 방식
- 단순 CRUD
- 단순 알림 발송 전체
- feed/statistics projection 전체
- 과도한 job dependency graph
- 정산, 통계, 알림, feed를 하나의 job에 섞는 구조

#### Failure / Recovery Basis

Batch 실패 시 복구 기준은 다음이다.

- `Settlement.status`
- `batch_run_key`
- `settlement_item.point_history_id`
- `point_history.idempotency_key`
- retry count / failure code

Batch는 복구를 실행하는 계층이지 정합성의 source of truth가 아니다.

#### Schedule Pressure Cut

줄일 수 있는 것:

- step/job 세분화 수준
- 고급 listener/metric
- 통계성 batch

줄이면 안 되는 것:

- settlement batch
- retry/recovery
- reconciliation
- partial failure recovery

---

### 4.4 React + Vite

#### Decision

Frontend는 React + Vite를 사용한다. 이 ADR은 기존 “빌드 도구 미정” 상태를 MVP 기준으로 React + Vite로 결정한다.

#### Why

- 갓세이빙 MVP는 SSR/SEO보다 빠른 UI 구현과 API 연동이 중요하다.
- Next.js의 라우팅/SSR/server action 구조는 현재 요구에 비해 복잡도가 크다.
- React + Vite는 학습 비용과 구현 속도 측면에서 MVP에 적합하다.

#### Solves

- 빠른 화면 개발
- 백엔드 REST API와 분리된 UI 개발
- dashboard, room, mission, settlement 결과 화면 구현

#### Allowed Use

- SPA 중심 UI
- API 응답 projection 표시
- 간단한 client-side routing
- Context/Zustand 수준의 최소 상태관리
- Tailwind 기반 단순 UI 전략

#### Forbidden Use

- 복잡한 SSR 구조
- Next.js 기반 server/client boundary 설계
- 과도한 global state architecture
- FE에서 정산/포인트 최종 상태를 계산하는 구조

#### Failure / Recovery Basis

FE 화면은 projection이다. 정산/포인트 최종 상태는 API가 제공하는 DB 기준 상태를 표시해야 한다. dashboard cache나 ranking은 source of truth가 아니다.

#### Schedule Pressure Cut

- 고급 상태관리 축소
- dashboard polish 축소
- E2E 범위 축소 가능

단, 결제/정산/포인트 핵심 흐름의 사용자 오해를 줄이는 기본 UI는 유지한다.

---

### 4.5 Docker Compose / EC2 / Nginx / GitHub Actions

#### Decision

MVP 배포는 EC2 + Docker Compose + Nginx + GitHub Actions를 기본으로 한다. Docker Compose는 단일 서버 또는 local/dev 재현성을 위한 도구이며, production orchestration platform으로 과장하지 않는다.

#### Why

- 팀은 Kubernetes/MSA 운영 경험이 없다.
- 단일 서버 배포 중심 경험이 있고, MVP 운영 인력도 거의 없다.
- Docker Compose는 애플리케이션, MySQL/Redis 등 의존성을 재현하기 쉽다.
- GitHub Actions로 기본 CI/CD를 구성할 수 있다.

#### Solves

- 배포 재현성
- 단일 서버 운영 단순화
- Nginx reverse proxy
- CI/CD 자동화
- 환경 변수 기반 설정 관리

#### Allowed Use

- local/dev compose
- MVP 단일 서버 배포 compose
- Nginx reverse proxy / TLS termination
- GitHub Actions build/test/deploy
- 수동 rollback runbook

#### Forbidden Use

- Compose를 Kubernetes 대체 오케스트레이터처럼 사용하는 것
- 복잡한 blue-green/canary를 MVP 필수로 넣는 것
- MSA식 다중 서비스 운영 전제

#### Failure / Recovery Basis

배포 실패 시 기준은 다음이다.

- 이전 이미지/컨테이너 rollback
- DB migration rollback/forward 전략
- CloudWatch/log 기반 장애 확인
- batch 실행 중 배포 시 RUNNING timeout recovery

#### Schedule Pressure Cut

- 고급 blue-green/canary 제외
- full IaC 제외
- 자동 rollback 고도화 제외

단, 최소 build/test/deploy와 rollback runbook은 유지한다.

---

### 4.6 CloudWatch

#### Decision

CloudWatch는 MVP 최소 운영 가시성 확보를 위해 사용한다. full observability stack은 도입하지 않는다.

#### Why

- 운영 인력이 거의 없으므로 장애 감지와 로그 조회가 최소한 가능해야 한다.
- Prometheus/Grafana/OpenTelemetry 풀스택은 MVP 시점에 운영 부담이 크다.
- CloudWatch는 EC2/Docker 기반 MVP와 잘 맞는 최소 관측 도구다.

#### Solves

- application error log 수집
- batch failure 감지
- settlement delay 감지
- Redis/DB 연결 실패 감지
- payment confirm duplicate/conflict 감지
- reconciliation mismatch 감지

#### Allowed Use

MVP 필수 알람 후보:

- settlement batch failure
- `RUNNING` timeout
- `RETRY_WAIT` 증가
- DB connection failure
- Redis unavailable
- payment confirm failure
- idempotency conflict
- reconciliation mismatch
- disk usage

#### Forbidden Use

- full distributed tracing
- 복잡한 custom dashboard 전면 도입
- SLO/error budget 운영 체계까지 MVP에 포함

#### Failure / Recovery Basis

CloudWatch는 장애 감지 도구다. 복구 기준은 DB이며, 복구 실행은 관리자 API/batch trigger를 통해 수행한다.

#### Schedule Pressure Cut

- 고급 dashboard 축소
- custom metric 일부 축소

단, batch 실패, Redis/DB 장애, idempotency conflict, reconciliation mismatch 알람은 유지 우선순위가 높다.

---

### 4.7 QueryDSL

#### Decision

QueryDSL은 복잡 조회/동적 검색/운영 조회에 제한적으로 사용한다. 이 ADR은 기존 “persistence 세부 기술 미정” 상태를 MVP 기준으로 “Spring Data JPA 기본 + QueryDSL 제한 도입”으로 결정한다.

#### Why

- settlement/point/payment 운영 조회는 동적 조건과 join이 많아질 수 있다.
- 복구용 read-only query, 관리자 API 조회, dashboard projection 조회에 QueryDSL이 유용하다.
- 하지만 단순 CRUD까지 QueryDSL로 작성하면 구현량과 학습 비용이 증가한다.

#### Solves

- settlement 상태별 조회
- idempotency_key 기준 point_history 조회
- payment confirm/idempotency 처리 이력 조회
- reconciliation 대상 조회
- 운영자 read-only diagnosis query

#### Allowed Use

- 복잡한 동적 조건 조회
- 운영/복구 조회 API
- dashboard projection query
- 통계성 read model query

#### Forbidden Use

- 단순 CRUD 전부 QueryDSL화
- 도메인 로직을 query builder에 숨기는 구조
- 정산 금액 계산의 source of truth를 QueryDSL projection에 두는 구조

#### Failure / Recovery Basis

QueryDSL은 조회 편의 도구다. 복구 기준은 QueryDSL 결과가 아니라 MySQL의 원본 row와 불변조건이다.

정산 금액 계산, 지급 확정, partial recovery 판단은 QueryDSL projection이 아니라 MySQL row, 도메인 계산 로직, `settlement_item`, `point_history`, DB unique constraint, deterministic idempotency key를 기준으로 한다.

#### Schedule Pressure Cut

일정이 밀리면 단순 조회는 Spring Data JPA로 유지하고, QueryDSL은 운영/복구 핵심 조회에만 남긴다.

---

### 4.8 Email / SMTP

#### Decision

MVP 이메일은 SMTP 기반으로 발송한다. 이메일은 정산 완료 및 중요 알림의 보조 채널이며, 정산/포인트/결제 트랜잭션의 일부가 아니다.

#### Required Structured Log Fields

- `settlement_id`
- `member_id`
- `email_type`
- `recipient_hash`
- `attempt`
- `result`
- `smtp_error_code`
- `created_at`

#### Allowed Use

- 정산 완료 이메일
- 중요 안내성 이메일
- bounded retry
- 운영자 수동 재발송
- structured log 기반 운영 확인

#### Forbidden Use

- 이메일 실패로 `Settlement.status` 롤백
- 이메일 실패로 `point_history` 또는 결제 충전 원장 롤백
- 이메일 발송을 정산 트랜잭션 내부 필수 단계로 결합
- MVP에서 notification log/outbox를 필수 테이블로 도입

#### Failure / Recovery Basis

이메일 실패는 비금전성 후속 이벤트 실패다. 최종 정산 결과와 포인트 원장은 API와 DB 기준으로 유지되며, 이메일 실패는 structured log, bounded retry, 운영자 수동 재발송으로 처리한다.

---

### 4.9 Storage / S3 Presigned Upload

#### Decision

인증 이미지는 AWS S3 private bucket에 저장한다. 업로드는 서버가 생성한 object key에 대한 presigned URL로 수행한다.

Presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다. 인증 검증은 mission-log 생성 시 서버가 S3 object를 직접 조회해 수행한다.

#### Required Guardrails

- object key는 서버가 생성한다.
- 사용자는 임의 S3 path/key를 지정할 수 없다.
- 권장 key 형식은 `mission/{roomId}/{participantId}/{uuid}`다.
- presigned URL은 짧은 TTL을 가진다.
- bucket/object는 public으로 열지 않는다.
- mission-log 생성 시 서버는 object 존재 여부, size, content-type, key 소유 범위, EXIF를 검증한다.
- EXIF는 클라이언트 입력이 아니라 서버가 업로드 객체에서 추출/검증한 값을 기준으로 한다.
- orphan object는 MVP에서 lifecycle rule 또는 최소 cleanup job으로 정리한다.

#### Forbidden Use

- public bucket 기반 인증 이미지 제공
- 클라이언트 임의 key/path 업로드
- 클라이언트가 제출한 EXIF 값을 authoritative source로 취급
- S3 object 검증 없이 `MissionLog` 생성

---

### 4.10 Payment / Toss Confirm-only Charge

#### Decision

MVP 포인트 충전은 TossPayments sandbox confirm-only 흐름을 사용한다. API의 `payment_id`는 Toss `paymentKey`를 의미한다.

#### Required Guardrails

- `payment_id` = Toss `paymentKey`
- `orderId`는 confirm 검증과 로그 상관관계 추적용이다.
- `orderId`는 `point_history.idempotency_key` 구성값으로 사용하지 않는다.
- 충전 idempotency key는 `charge:{paymentKey}`다.
- 동일 key + 동일 payload는 기존 `point_history`를 재사용한다.
- 동일 key + 다른 payload는 conflict로 실패한다.
- provider success + client timeout 상황에서도 같은 `paymentKey` 재조회/재시도로 중복 충전되지 않아야 한다.
- MVP에서는 별도 payment aggregate 없이 `point_history`를 충전 ledger로 사용한다.

#### Deferred

- webhook/callback 기반 충전 확정
- payment_event/payment_attempt 테이블
- 결제 취소/환불 자동화
- 운영 결제 키 전환

## 5. Redis/Redisson 운영 원칙

### 5.1 사용 원칙

Redis/Redisson은 다음 영역에서 적극적으로 사용해도 MVP 과설계가 아니다.

| 영역                      | 사용 목적                 | 최종 방어선                                 |
| ------------------------- | ------------------------- | ------------------------------------------- |
| distributed lock          | 중복 실행 가능성 감소     | DB conditional claim / unique / idempotency |
| idempotency cache         | 중복 요청 빠른 판별       | `point_history.idempotency_key` unique      |
| rate limiting             | 남용/과도한 요청 방지     | API validation / DB 상태                    |
| cache                     | dashboard/projection 성능 | MySQL source data                           |
| duplicate execution guard | settlement 중복 실행 억제 | `Settlement.status`, `batch_run_key`        |
| SSE/event 보조            | 비최종 알림 전달          | DB 상태/API 조회                            |

### 5.2 Redis 장애 모드

#### Normal mode

Redis가 정상일 때 worker는 Redis를 사용해 다음을 수행한다.

- short-lived settlement lock
- duplicate worker suppression
- rate limiting
- retry pacing
- projection cache

하지만 correctness는 Redis가 아니라 MySQL에 의존한다.

#### Redis unavailable — 일반 worker 동작

Redis가 unavailable하면 일반 online worker는 다음 원칙을 따른다.

- 새로운 금전성 지급/정산 실행은 fail-closed 또는 controlled retry로 전환한다.
- read-only inspection은 가능하다.
- Redis lock 상태를 알 수 없다는 이유만으로 payout execution을 강행하지 않는다.
- 장애는 retryable infrastructure failure로 분류한다.

#### Redis unavailable — 명시적 fallback recovery path

Redis 장애 중에도 복구가 필요하면, 별도 관리자 API 또는 batch trigger를 통해 **DB-claim-only fallback path**를 실행할 수 있다.

MVP의 기본 fallback은 settlement-level ownership이다.

- `Settlement.status in (PENDING, RETRY_WAIT)`인 row만 대상이다.
- stale `RUNNING` timeout 후보는 먼저 RUNNING timeout recovery로 `RETRY_WAIT` 전환한 뒤 fallback claim 대상이 된다.
- 조건부 update로 `PENDING/RETRY_WAIT -> RUNNING` claim을 수행한다.
- `batch_run_key`, `started_at`, `retry_count`, failure code를 기록한다.
- update row count가 `1`이면 실행권을 가진다.
- row count가 `0`이면 다른 worker/operator가 먼저 claim한 것이므로 skip한다.
- participant별 중복 지급은 deterministic `point_history.idempotency_key`와 unique constraint가 막는다.

item-level concurrent recovery가 필요해지는 경우, 숨겨진 claim field를 가정하지 않는다. 다음 중 하나를 별도 ADR/스키마 변경으로 도입해야 한다.

- `settlement_item` explicit claim columns
- recovery job table
- explicit `payout_status` state machine

### 5.3 Redis 남용 금지 사례

- Redis에 포인트 잔액을 원장처럼 저장
- Redis lock을 획득했으니 DB 검증 없이 지급
- Redis idempotency cache만 믿고 DB unique 생략
- Redis Stream으로 ledger/event sourcing 흉내
- Redis 장애 시 운영자가 DB를 직접 고쳐야만 복구 가능한 구조

## 6. Spring Batch 운영 원칙

### 6.1 사용 원칙

Spring Batch는 정산/재시도/복구 실행 계층이다.

MVP에서 유지 우선순위가 높은 Batch 기능은 다음이다.

- settlement batch
- retry/recovery batch
- reconciliation batch
- partial failure recovery
- RUNNING timeout recovery
- scheduler 기반 운영 플로우
- 관리자 API가 호출하는 batch trigger

### 6.2 Job/Step 범위 제한

Batch job은 다음 기준으로 나눈다.

| Job                          | 목적                    | 범위                                                       |
| ---------------------------- | ----------------------- | ---------------------------------------------------------- |
| Settlement job               | 종료/취소 room 정산     | settlement claim, calculation snapshot, participant refund |
| Retry recovery job           | 실패/대기 정산 재처리   | `RETRY_WAIT`, timeout, partial 상태 처리                   |
| Reconciliation job           | point balance 검증/복구 | `point_history` 기준 balance rebuild                       |
| Admin-triggered recovery job | 운영 복구               | 특정 settlement/user 대상 제한 실행                        |

Job/Step은 복구 경계가 명확할 정도로만 나눈다. 학습 목적으로 과도하게 세분화하지 않는다.

### 6.3 Batch 남용 금지 사례

- 모든 알림 발송을 Batch로 강제
- feed 생성, 단순 통계, dashboard cache 갱신까지 모두 Batch로 편입
- 정산과 무관한 CRUD 처리에 Batch 사용
- Step dependency graph가 복구보다 디버깅을 어렵게 만드는 구조
- Batch metadata를 비즈니스 source of truth처럼 사용하는 구조

## 7. 장애/복구 철학

### 7.1 강한 무결성 영역

아래 흐름은 불일치 감지 시 fail-safe가 우선이다.

- 결제 승인
- 보증금 lock
- 환급
- 정산 지급
- withdrawal 성격의 포인트 차감

정책:

- 해당 사용자 또는 settlement 단위 operation 차단
- error log + alert 발생
- 관리자 API/batch trigger로 복구
- 필요 시 reconciliation 실행

### 7.2 약한 일관성 영역

아래 영역은 즉시 차단보다 reconciliation/eventual consistency를 허용한다.

- 통계성 projection
- dashboard 잔액 표시
- 실시간 ranking/share
- feed성 데이터
- read model cache
- SSE realtime notification, toast, badge/count UX projection

이 영역은 source of truth가 아니며, 사용자에게 최종 정산 기준이 아님을 명확히 할 수 있다.

### 7.3 point_history ledger

- `point_history`는 append-only에 가깝게 다룬다.
- 모든 포인트 이벤트는 deterministic `idempotency_key`를 가진다.
- 동일 이벤트는 항상 동일 idempotency key를 사용한다.
- 동일 key + 동일 payload 재시도는 기존 원장을 재사용/연결한다.
- 동일 key + 다른 payload는 idempotency conflict로 실패한다.

### 7.4 Settlement item derived payout state

MVP에서는 schema churn을 줄이기 위해 별도 `payout_status` 컬럼을 즉시 추가하지 않고, `settlement_item.point_history_id`와 대응 `point_history` 존재 여부로 participant별 지급 상태를 파생한다.

| Derived state          | Rule                                                                         |
| ---------------------- | ---------------------------------------------------------------------------- |
| `PENDING_PAYOUT`       | `point_history_id IS NULL` and parent `Settlement.status` is not `SUCCEEDED` |
| `PAYOUT_SUCCEEDED`     | `point_history_id IS NOT NULL` and referenced `point_history` exists         |
| `INVALID_INCONSISTENT` | `point_history_id IS NOT NULL` but referenced `point_history` does not exist |
| `BLOCKED_AMBIGUOUS`    | parent `Settlement.status` conflicts with item-derived state                 |

Parent status rule:

- `Settlement.status = SUCCEEDED`는 모든 item이 `PAYOUT_SUCCEEDED`일 때만 유효하다.
- 하나라도 `PENDING_PAYOUT`이면 parent settlement는 `SUCCEEDED`가 될 수 없다.
- 하나라도 `INVALID_INCONSISTENT`이면 복구/조사가 필요하며 succeeded로 취급하지 않는다.
- parent status만으로 participant별 지급 완료를 추론하지 않는다.

금지되는 ambiguous state:

1. parent `SUCCEEDED`인데 item `point_history_id IS NULL`
2. item `point_history_id IS NOT NULL`인데 대응 `point_history` row 없음
3. 이미 `PAYOUT_SUCCEEDED`인 item에 대한 재지급
4. Redis lock state를 payout state로 취급
5. provider timeout만으로 성공 처리

향후 다음 조건이 생기면 explicit `payout_status` 컬럼 도입을 재검토한다.

- item lifecycle이 pending/succeeded/recovery-needed 이상으로 복잡해짐
- provider reconciliation에 durable intermediate state가 필요함
- 운영 조회에서 item status filter가 자주 필요함
- derived rule만으로 복구 로직을 설명하기 어려워짐

### 7.5 결제/정산 용어 경계

MVP settlement refund에서 “payout”은 외부 송금/현금 출금이 아니라 내부 `point_history` credit과 `point_account.balance` update를 의미한다.

provider timeout 검증은 다음에 적용한다.

- point charge/payment approval
- 향후 외부 payout/remittance integration

현재 MVP settlement refund에 외부 지급 provider가 있다고 가정하지 않는다.

### 7.6 관리자 복구 원칙

MVP에서 Admin UI는 만들지 않는다.

허용:

- DB read-only query
- CloudWatch/log 조회
- 관리자 API
- batch trigger
- recovery runbook
- 복구용 SQL 조회 쿼리 모음

금지:

- `point_account.balance` 직접 수정
- `point_history` 직접 수정/삭제
- `settlement_item` 결과 금액 직접 변경
- 임의로 `SUCCEEDED` 처리

Direct DB mutation은 정상 복구 경로가 아니다. break-glass emergency에서만 허용하며, 이 경우에도 incident procedure, audit record, post-repair invariant validation이 필요하다.

## 8. Testable Architecture Invariants

### Settlement correctness

1. `Settlement.status = SUCCEEDED`는 모든 `settlement_item.point_history_id`가 non-null이고 대응 `point_history`가 존재할 때만 가능하다.
2. `settlement_item.point_history_id IS NULL`인 item은 지급 완료로 보지 않는다.
3. 같은 business payout intent는 두 번 지급될 수 없다.
4. partial 상태는 `RETRY_WAIT` 또는 `FAILED`로 남아야 하며 succeeded로 취급하지 않는다.

### Idempotency

1. 모든 금전성 이벤트는 deterministic `idempotency_key`를 가진다.
2. 결제 충전은 API field `payment_id`에 담긴 Toss `paymentKey`를 사용해 `charge:{paymentKey}`를 생성한다.
3. 동일 `paymentKey`는 하나의 충전 이벤트만 의미해야 하며, `orderId`는 confirm 검증과 상관관계 추적용으로만 사용한다.
4. 동일 idempotency key + 동일 payload는 기존 원장을 재사용/연결한다.
5. 동일 idempotency key + 다른 payload는 conflict로 실패한다.

### Redis safety

1. Redis lock 획득 실패가 unsafe payout execution으로 이어지면 안 된다.
2. Redis 부재가 직접 DB 수정 필요성을 만들면 안 된다.
3. fallback recovery는 MySQL ownership semantics를 사용해야 한다.

### SSE notification safety

1. SSE 연결 실패, reconnect, 서버 재시작은 인증/정산/포인트 원장 transaction을 롤백하거나 차단하면 안 된다.
2. SSE는 source-of-truth state가 아니라 best-effort UX signal이다.
3. Durable broker, outbox, replay, notification inbox, unread sync는 MVP 필수 요구가 아니다.
4. SSE 외부 계약, identity, payload, reconnect semantics는 `API-spec-god-saving.md`의 알림/SSE 계약을 따른다.

### Operator recovery

1. 운영자는 approved API/batch command를 통해서만 금전성 복구를 실행한다.
2. 복구 command는 idempotent하고 resumable해야 한다.
3. 복구 실행자는 누가, 언제, 무엇을 실행했는지 audit log를 남긴다.

### 8.1 Implementation gate overlay

구현자는 `docs/implementation-gates.md`를 PR 리뷰용 gate checklist로 함께 사용한다. 이 overlay는 새로운 source of truth가 아니라, 본 ADR과 `API-spec`, `ERD`, `Settlement-design`, runbook의 blocker-level invariant를 구현 중 반복 확인하기 위한 문서다.

Blocker로 취급한다:

- `point_history` source of truth 위반
- deterministic idempotency key 위반
- Redis/Batch/CloudWatch를 correctness source로 승격
- `Settlement.status = SUCCEEDED`인데 `settlement_item.point_history_id` 또는 대응 `point_history`가 누락된 상태
- client-provided EXIF를 authoritative로 저장
- 관리자 API/runbook 없이 직접 DB mutation을 정상 복구 경로로 안내

구현 중 조정 가능하지만 blocker invariant를 깨면 안 된다:

- CloudWatch alarm threshold
- orphan cleanup 주기
- retry interval/backoff
- structured log 추가 필드
- batch step 세분화 수준
- presigned URL TTL과 content-type whitelist 상세값

## 9. Cut line / Scope Reduction 전략

### 9.1 MVP에서 유지 우선순위가 높은 핵심 기능

| 기능                                   | 유지 이유                                   |
| -------------------------------------- | ------------------------------------------- |
| DB idempotency unique constraints      | 중복 결제/정산 방지 최종 방어선             |
| DB conditional updates/transactions    | settlement claim과 보증금 lock 안전성       |
| `point_history` append-only ledger     | 포인트 금액 복구 기준                       |
| `settlement_item` calculation snapshot | participant별 정산 근거/분쟁 기준           |
| settlement batch                       | 정산 실행의 기본 경로                       |
| reconciliation batch                   | balance 불일치 복구 경로                    |
| participant-level partial retry        | 일부 성공/일부 실패 복구                    |
| Redisson duplicate/concurrency guard   | 운영/동시성 보조 계층 경험과 중복 실행 억제 |
| rate limiting minimum version          | 남용/과도한 요청 방지                       |
| 관리자 recovery API/batch trigger      | DB 직접 수정 없는 복구                      |
| runbook + read-only SQL query set      | 운영자가 장애를 판단할 수 있는 최소 도구    |
| targeted tests                         | duplicate/partial/reconciliation 회귀 방지  |

### 9.2 일정이 밀릴 때 우선 축소할 기능

| 축소 순서 | 축소 대상                     | 줄일 수 있는 이유                                               | 유지해야 할 대체 기준                    |
| --------: | ----------------------------- | --------------------------------------------------------------- | ---------------------------------------- |
|         1 | SSE/event Redis 보조 구조     | 알림은 최종 정산 기준이 아니며 replay/fan-out은 MVP 필수가 아님 | DB 상태/API 조회, Email                  |
|         2 | dashboard/projection cache    | projection은 source of truth가 아님                             | MySQL 조회 또는 stale 표시               |
|         3 | Redis idempotency cache       | DB idempotency가 최종 방어선                                    | `point_history.idempotency_key` unique   |
|         4 | Batch step/job 세분화         | 복구 경계만 명확하면 세분화는 줄일 수 있음                      | settlement/retry/reconciliation job 유지 |
|         5 | Advanced CloudWatch dashboard | 알람/로그가 먼저 중요                                           | 핵심 error/metric alarm 유지             |
|         6 | QueryDSL 사용 범위            | 단순 조회는 JPA로 가능                                          | 운영/복구 핵심 조회만 QueryDSL           |
|         7 | Admin API 편의 기능           | 편의 기능은 나중에 가능                                         | 핵심 recovery command 유지               |

### 9.3 줄이면 안 되는 것

- `point_history` ledger
- idempotency key 설계
- DB unique/transaction/conditional update
- settlement partial recovery
- reconciliation
- 직접 DB 수정 금지 원칙
- Redis가 source of truth가 아니라는 원칙

## 10. 대안과 제외 이유

| 대안                                    | 장점                                             | MVP에서 제외하는 이유                                        | 재검토 조건                                               |
| --------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| Kafka/Event Broker                      | durable event stream, consumer 분리, async scale | 운영 부담과 event contract 복잡도가 MVP 범위를 초과          | DB polling/batch 처리량 한계, 다수 downstream replay 필요 |
| CDC                                     | MySQL 변경 전파 자동화                           | connector/runtime 복잡도와 downstream 보장 필요              | analytics/audit feed가 near-real-time으로 필요            |
| Event Sourcing                          | complete transition history, replayability       | 설계/구현 비용이 큼. 현재 relational settlement model로 충분 | audit/replay가 핵심 제품 요구가 됨                        |
| Kubernetes/MSA                          | service isolation, independent scaling           | 팀 운영 경험 부족, MVP에는 단일 deployable app이 빠름        | 팀/트래픽/배포 빈도/격리 요구 증가                        |
| Full Terraform/IaC                      | 재현 가능한 인프라                               | 초기 인프라 footprint가 작고 학습 비용 큼                    | 다중 환경, DR, 규제성 배포 증적 필요                      |
| Full OpenTelemetry + Prometheus/Grafana | tracing/metrics/observability 강화               | MVP 전면 도입은 운영 복잡도 과다                             | SLO/error budget 또는 분산 추적 필요                      |
| Admin UI                                | 운영 실수 감소                                   | 일정/보안/권한 관리 부담                                     | 비개발자 운영자 투입, 복구 빈도 증가                      |
| Direct DB mutation                      | 빠른 emergency repair                            | 중복 지급, 불변조건 위반, 감사 누락 위험                     | 정상 경로로는 재검토하지 않음. break-glass만 허용         |
| Next.js                                 | SSR, file routing, fullstack capability          | MVP는 SEO/SSR보다 빠른 SPA 구현이 중요                       | 공개 SEO/SSR 요구가 제품 핵심이 될 때                     |

## 11. 검증 매트릭스

| Scenario                                 | Test level               | Setup                                          | Expected result                                               |
| ---------------------------------------- | ------------------------ | ---------------------------------------------- | ------------------------------------------------------------- |
| Duplicate payment confirm retry          | Integration              | 같은 Toss `paymentKey` confirm 재시도 2회 수신 | `charge:{paymentKey}`로 `point_history` 1건만 생성/재사용     |
| Duplicate settlement run                 | Integration/batch        | 같은 room settlement 동시 실행                 | DB claim 1개만 성공, 중복 지급 없음                           |
| Redis unavailable during normal worker   | Integration              | Redis 연결 실패                                | worker fail-closed 또는 retry, unsafe payout 없음             |
| Redis unavailable fallback recovery      | Batch/integration        | Redis off, DB-claim-only recovery 실행         | `Settlement.status` 조건부 claim으로 1개 실행권, 중복 없음    |
| Provider timeout after successful charge | Integration/recovery     | provider는 성공, client timeout                | 같은 idempotency key로 재조회/재시도, 중복 충전 없음          |
| Partial batch failure and resume         | Batch/recovery           | 일부 participant 성공 후 job 실패              | 완료 item 유지, 미완료 item만 재시도                          |
| Balance rebuild                          | Batch/reconciliation     | `point_account.balance` 불일치 fixture         | `point_history` 합계 기준으로 재계산/복구                     |
| Operator recovery without DB mutation    | Recovery                 | stuck settlement 대상 admin API 실행           | app/batch path로 복구, audit log 생성                         |
| Parent `SUCCEEDED` with null item FK     | Unit/data validation     | inconsistent fixture                           | succeeded 취급 거부, recovery 필요                            |
| Non-null item FK missing point_history   | Unit/data validation     | orphan fixture                                 | `INVALID_INCONSISTENT`, investigation 필요                    |
| Concurrent recovery workers              | Integration              | recovery worker 2개 동시 실행                  | MySQL claim으로 item/settlement 중복 처리 없음                |
| Break-glass validation                   | Operational test/runbook | 긴급 DB 수정 가정                              | incident record, audit, post-repair invariant validation 필요 |

## 12. 운영 Runbook 최소 요구

MVP에는 다음 문서를 함께 둔다.

- settlement 상태 조회 SQL
- `settlement_item` + `point_history` 연결 검증 SQL
- idempotency key 기준 `point_history` 조회 SQL
- payment confirm/idempotency 처리 이력 조회 SQL/API
- `point_account.balance` reconciliation 실행 절차
- Redis 장애 시 일반 worker fail-closed 확인 절차
- Redis 장애 시 DB-claim-only fallback 실행 절차
- break-glass incident procedure
- post-repair invariant validation checklist

## 13. 결과와 영향

### Positive Consequences

- 금전성 correctness가 Redis/Batch가 아니라 MySQL 불변조건에 고정된다.
- Redis/Redisson과 Spring Batch를 실제 운영 문제에 연결된 범위에서 학습할 수 있다.
- 장애 후 복구 기준과 축소 기준이 명확하다.
- 작은 팀 MVP에 맞게 K8s/MSA/Kafka 등 과한 운영 복잡도를 피한다.

### Negative Consequences

- Redis/Batch 학습 비용은 여전히 존재한다.
- derived payout state는 명확한 validation 없이는 운영자가 이해하기 어려울 수 있다.
- Admin UI가 없으므로 초기 운영은 runbook과 API/SQL 숙련도에 의존한다.
- CloudWatch 최소 구성만으로는 고급 원인 분석이 제한될 수 있다.

### Follow-ups

- CSS/UI, 상태관리, 테스트 러너는 별도 Frontend/QA ADR에서 결정
- Redis degraded mode와 DB-claim-only fallback의 실제 운영 SQL/API 예시는 runbook에 지속 보강
- derived item payout state validation 또는 향후 `payout_status` 도입 조건은 ERD 후속 보강
- 정산/멱등성/복구 테스트 시나리오 구현

## 14. 최종 기술 철학

갓세이빙 MVP의 기술 아키텍처는 빠른 출시보다 **복구 불가능한 빠름**을 경계한다.

> Redis/Redisson은 운영/동시성 보조 계층,  
> Spring Batch는 정산/재시도/복구 실행 계층,  
> MySQL은 최종 정합성 및 복구 기준으로 역할을 고정한다.

Redis는 운영을 돕고, Batch는 재시도와 복구를 실행하며, MySQL은 진실을 남긴다. MVP 성공 기준은 기술 도입 수가 아니라 장애 후에도 `point_history`, `settlement_item`, DB 제약, idempotency key를 기준으로 안전하게 복구할 수 있는지다.
