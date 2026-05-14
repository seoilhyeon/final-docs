# 기술 스택 정리: 갓세이빙 MVP

## 1. 문서 목적

이 문서는 갓세이빙 MVP의 기술 스택을 빠르게 찾기 위한 reference/index다. 구현 정책, API 계약, 데이터 invariant, 운영 복구 규칙을 새로 정의하지 않는다. 세부 기준은 아래 owner 문서를 따른다.

## 2. 확정 기술 스택

| 영역 | 선택 기술 | 기준 문서 |
| --- | --- | --- |
| Backend | Java 17, Spring Boot 3.2 | `adr/ADR-mvp-tech-architecture.md` |
| Database | MySQL 8.0 | `ERD-god-saving.md`, `adr/ADR-mvp-tech-architecture.md` |
| Cache / Lock | Redis, Redisson | `adr/ADR-mvp-tech-architecture.md`, `Settlement-design.md` |
| Batch | Spring Batch 5.x | `adr/ADR-mvp-tech-architecture.md`, `Settlement-design.md` |
| File Storage | AWS S3 | `API-spec-god-saving.md`, `adr/ADR-mvp-tech-architecture.md` |
| Frontend | React, Vite, Axios | `adr/ADR-mvp-tech-architecture.md` |
| Notification | SSE, Email | `PRD-god-saving.md`, `API-spec-god-saving.md`, `adr/ADR-mvp-tech-architecture.md` |
| Payment | 토스페이먼츠 샌드박스 | `API-spec-god-saving.md`, `adr/ADR-mvp-tech-architecture.md` |
| AI | Claude API | `PRD-god-saving.md`, `API-spec-god-saving.md` |
| Infra | AWS EC2, Docker Compose, Nginx, GitHub Actions, CloudWatch | `adr/ADR-mvp-tech-architecture.md`, `docs/runbooks/settlement-recovery.md` |
| API Contract | REST + JSON | `API-spec-god-saving.md` |
| Auth | JWT Bearer token | `API-spec-god-saving.md`, `ERD-god-saving.md` |

## 3. Ownership reference map

| 주제 | Owner |
| --- | --- |
| 제품 목표, 사용자 가치, MVP 범위 | `PRD-god-saving.md` |
| API/Auth/SSE 외부 계약 | `API-spec-god-saving.md` |
| `member.uuid`, DB 구조, persistence invariant | `ERD-god-saving.md` |
| 정산 계산, batch/retry/recovery, point ledger 정책 | `Settlement-design.md` |
| SSE 선택 이유, broker/outbox/replay 제외 이유, 기술 선택 rationale | `adr/ADR-mvp-tech-architecture.md` |
| 운영 절차 | `docs/runbooks/*` |
| 구현 순서와 acceptance evidence | `MVP-ticket-breakdown.md` |

## 4. 사용 규칙

- 이 문서와 owner 문서가 충돌하면 owner 문서를 우선한다.
- 이 문서에는 상세 정책을 복제하지 않는다.
- 기술 선택이 바뀌면 먼저 owner 문서를 수정한 뒤 이 index를 갱신한다.
