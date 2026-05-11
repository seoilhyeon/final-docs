# Docs README

## 1. 개요

이 `docs` 폴더는 갓세이빙 MVP의 제품 요구사항, 도메인 설계, 데이터 구조, API 계약, 실행 계획을 한 곳에서 관리하기 위한 문서 루트다. 목적은 세 가지다.

- 제품 규칙과 구현 규칙의 기준 문서를 분리한다.
- 신규 개발자가 코드를 보기 전에 전체 구조를 빠르게 이해할 수 있게 한다.
- 같은 내용을 여러 문서에 반복 작성하지 않도록 source of truth를 고정한다.

문서 계층은 아래 순서로 이해하면 된다.

1. 기획: 무엇을 만들고 왜 만드는가
2. 설계: 어떤 규칙과 구조로 동작하는가
3. 계약: FE/BE가 어떤 인터페이스로 연결되는가
4. 실행: 어떤 순서로 구현하고 검증하는가

원본 기획서(`.docx`)는 출발점이자 제품/UX 배경 참고 자료다. 현재 활성 제품/UX 참고 문서는 `docs/갓세이빙_프로젝트기획서.docx`이며, 구현과 운영 판단의 기준은 여전히 이 폴더의 Markdown 문서(`docs/*.md`)다. 제안서 표현이 Markdown 문서와 충돌하면 Markdown 문서가 우선한다.

## 2. 추천 읽기 순서

완전히 처음 보는 개발자는 아래 순서로 읽는다. 순서의 목적은 "제품 의도 → 도메인 규칙 → 외부 계약 → 실행 단위"를 끊기지 않게 이해하는 데 있다.

1. [PRD-god-saving.md](./PRD-god-saving.md)
   제품 목표, MVP 범위, 핵심 비즈니스 규칙을 먼저 이해한다. 이 문서를 읽지 않고 설계나 API부터 보면 왜 그런 제약이 있는지 놓치기 쉽다.
2. [ERD-god-saving.md](./ERD-god-saving.md) + [Settlement-design.md](./Settlement-design.md)
   PRD를 읽은 뒤에는 도메인 구조와 정산 규칙을 같이 본다. `ERD`는 데이터 모델의 경계를 설명하고, `Settlement-design`은 정산/포인트/동시성/재시도 같은 운영 규칙을 설명한다. 두 문서를 함께 봐야 데이터 구조와 비즈니스 계산 규칙이 연결된다.
3. [API-spec-god-saving.md](./API-spec-god-saving.md)
   앞선 문서들을 기반으로 FE/BE 계약을 확인한다. API는 독립 출발점이 아니라 PRD, ERD, 정산 설계를 외부 인터페이스로 고정한 결과물이다.
4. [MVP-backlog-user-stories.md](./MVP-backlog-user-stories.md) + [MVP-ticket-breakdown.md](./MVP-ticket-breakdown.md)
   마지막으로 실행 계획을 본다. 여기서는 무엇을 만들지 다시 정의하지 않고, 이미 결정된 범위를 어떤 단위로 구현할지 확인한다.
5. [갓세이빙\_프로젝트기획서.docx](./갓세이빙_프로젝트기획서.docx)
   활성 제품/UX 참고 자료다. 배경, 화면 의도, 소셜 피드 표현을 확인할 때만 본다. 구현 판단은 Markdown source of truth 문서에 승격된 내용만 따른다.

## 3. 문서별 역할 정의

| 문서                                                            | 한 줄 요약                                                            | 언제 읽는가                                     | Source of Truth                        |
| --------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------- |
| [PRD-god-saving.md](./PRD-god-saving.md)                        | MVP의 목표, 범위, 사용자 가치, 핵심 비즈니스 규칙을 정의한다.         | 프로젝트 온보딩 시작 시, 제품 정책 변경 검토 시 | 비즈니스 요구사항과 MVP 범위           |
| [ERD-god-saving.md](./ERD-god-saving.md)                        | 핵심 테이블, 관계, 제약, 스냅샷 저장 구조를 정의한다.                 | DB 설계, 쿼리 설계, 영속성 모델 검토 시         | 데이터 구조와 DB 제약                  |
| [Settlement-design.md](./Settlement-design.md)                  | 정산 계산, 상태 흐름, 멱등성, 동시성, 실패/재시도 정책을 정의한다.    | 정산/포인트/배치 구현 또는 운영 정책 검토 시    | 정산 도메인 규칙과 운영 안전성 정책    |
| [API-spec-god-saving.md](./API-spec-god-saving.md)              | FE/BE가 공유하는 엔드포인트, 요청/응답, 에러, 상태값 계약을 정의한다. | 화면 연동, API 구현, QA 시나리오 작성 시        | 외부 API 계약                          |
| [MVP-backlog-user-stories.md](./MVP-backlog-user-stories.md)    | MVP 기능을 사용자 가치 중심의 백로그와 스토리로 정리한다.             | 우선순위 검토, 스프린트 범위 설정 시            | 스토리 레벨 구현 대상과 우선순위       |
| [MVP-ticket-breakdown.md](./MVP-ticket-breakdown.md)            | 스토리를 구현 가능한 티켓과 의존성으로 분해한다.                      | 실제 작업 착수, 일정/병렬화 계획 수립 시        | 구현 단위, 선행 의존성, 실행 순서      |
| [갓세이빙\_프로젝트기획서.docx](./갓세이빙_프로젝트기획서.docx) | 활성 제품/UX 참고 자료다.                                             | 배경, UX 의도, 제안서 표현 확인이 필요할 때     | 현재 source of truth 아님. 참고용 입력 |

문서 충돌 시 우선순위는 아래를 따른다.

1. PRD: 제품 목표와 비즈니스 규칙
2. Settlement-design / ERD: 도메인 규칙과 데이터 구조
3. API-spec: 외부 인터페이스 계약
4. Backlog / Ticket: 실행 계획
5. 활성 제안서 docx: 참고 자료

## 4. 문서 간 관계 구조

문서 간 의존 방향은 아래와 같다.

- `PRD -> ERD`
  제품 요구사항이 어떤 데이터 구조가 필요한지 결정한다.
- `PRD -> Settlement-design`
  비즈니스 규칙이 정산 정책, 예외 처리, 운영 규칙의 상위 기준이 된다.
- `ERD + Settlement-design -> API-spec`
  API는 데이터 모델과 도메인 규칙을 외부 계약으로 노출한 결과다.
- `PRD + API-spec + Settlement-design -> MVP-backlog-user-stories`
  무엇을 어떤 가치로 구현할지 스토리 단위로 정리한다.
- `MVP-backlog-user-stories + API-spec + Settlement-design -> MVP-ticket-breakdown`
  구현 단위, 선행 작업, 병렬 작업 구성을 확정한다.

간단히 보면 구조는 아래와 같다.

```text
활성 제안서(docx)
    ↓
PRD
    ↓
ERD / Settlement-design
    ↓
API-spec
    ↓
MVP-backlog-user-stories
    ↓
MVP-ticket-breakdown
```

## 5. 중복 방지 규칙

아래 규칙을 기준으로 문서 책임을 분리한다.

- 비즈니스 목표, 사용자 가치, MVP 포함/제외 범위는 `PRD`에만 정의한다.
- 정산 로직, 정산 상태 흐름, 재시도, 멱등성, 실패 코드 의미는 `Settlement-design`에만 상세 정의한다.
- 테이블, 컬럼, FK, unique 제약, 스냅샷 저장 구조는 `ERD`에만 상세 정의한다.
- 엔드포인트, 요청/응답 JSON, 에러 응답, API enum은 `API-spec`에만 상세 정의한다.
- 백로그 우선순위와 user story는 `MVP-backlog-user-stories`에만 정의한다.
- 구현 티켓, 선행 의존성, 병렬 실행 순서는 `MVP-ticket-breakdown`에만 정의한다.
- 활성 제안서(`docs/갓세이빙_프로젝트기획서.docx`)의 표현이 현재 Markdown 문서와 다르면 제안서를 직접 구현 기준으로 사용하지 않는다. 필요한 내용은 먼저 소유 Markdown source of truth 문서로 승격한 뒤 구현한다.
- AI 비트랜잭션 경계는 `PRD`, `API-spec`, `ERD`가 같은 정책을 유지해야 한다.
- `point_history.reference_type`의 DB enum/constraint 언어는 `ERD`가 소유하고, `API-spec`은 FE/BE 계약에 필요한 소비자-facing enum과 매핑만 반복한다.

다른 문서에는 필요한 만큼만 요약하고, 상세 규칙은 source of truth 문서로 링크한다. 같은 표나 enum, 같은 계산식을 여러 문서에 복제하지 않는다.

## 6. 수정 영향 범위 규칙

문서 수정은 아래 영향 범위를 기본으로 점검한다.

- `PRD` 수정:
  제품 규칙이 바뀌면 `Settlement-design`, `ERD`, `API-spec`, `Backlog`, `Ticket`까지 연쇄 영향이 있는지 확인한다.
- `Settlement-design` 수정:
  정산 상태, 계산 규칙, 재시도 정책이 바뀌면 `ERD`, `API-spec`, 관련 티켓을 함께 검토한다.
- `ERD` 수정:
  테이블 구조나 제약이 바뀌면 `API-spec`, 정산 설계, 구현 티켓 영향 여부를 확인한다.
- `API-spec` 수정:
  FE 화면, BE 구현, QA 시나리오, 티켓 정의가 영향을 받는다.
- `MVP-backlog-user-stories` 수정:
  우선순위 변경이 `Ticket` 실행 순서에 반영되어야 한다.
- `MVP-ticket-breakdown` 수정:
  구현 계획 변경이므로 상위 요구사항 자체를 바꾸지 않는다. 상위 규칙이 바뀌면 먼저 `PRD`, `Settlement-design`, `ERD`, `API-spec`를 수정한다.

실행 문서가 상위 문서를 덮어쓰면 안 된다. 규칙 변경은 항상 상위 문서에서 먼저 반영한다.

## 7. 신규 기여자 가이드

코드를 보기 전에 최소한 `README -> PRD -> 자신의 역할에 맞는 기준 문서` 순서로 읽는다.

### Backend 개발자

권장 순서:

1. [PRD-god-saving.md](./PRD-god-saving.md)
2. [Settlement-design.md](./Settlement-design.md)
3. [ERD-god-saving.md](./ERD-god-saving.md)
4. [API-spec-god-saving.md](./API-spec-god-saving.md)
5. [MVP-ticket-breakdown.md](./MVP-ticket-breakdown.md)

이 순서가 필요한 이유:
정산, 포인트, 참가/탈퇴, 원장, 배치 같은 핵심 도메인 규칙이 데이터 구조와 강하게 연결되어 있기 때문이다.

### Frontend 개발자

권장 순서:

1. [PRD-god-saving.md](./PRD-god-saving.md)
2. [API-spec-god-saving.md](./API-spec-god-saving.md)
3. [Settlement-design.md](./Settlement-design.md)
4. [MVP-backlog-user-stories.md](./MVP-backlog-user-stories.md)

이 순서가 필요한 이유:
화면은 제품 흐름과 API 계약에 직접 연결되고, 정산/인증처럼 오해하기 쉬운 상태는 `Settlement-design`의 기준을 같이 알아야 UI 해석 오류를 줄일 수 있다.

### Infra / DevOps

권장 순서:

1. [PRD-god-saving.md](./PRD-god-saving.md)
2. [Settlement-design.md](./Settlement-design.md)
3. [ERD-god-saving.md](./ERD-god-saving.md)
4. [API-spec-god-saving.md](./API-spec-god-saving.md)

이 순서가 필요한 이유:
배치 스케줄, 재시도, 멱등성, 포인트 원장, 운영 복구 경로는 인프라와 운영 설계에 직접 영향을 준다.

## 8. 운영 원칙

- 새로운 정책을 추가할 때는 먼저 source of truth 문서를 결정하고 그 문서에 반영한다. 제안서 내용도 PRD/API/ERD/Settlement-design 중 소유 문서에 승격된 뒤에만 구현 계약이 된다.
- 구현 세부사항이 문서와 충돌하면, 코드보다 먼저 문서 기준이 맞는지 확인한다.
- 더 이상 기준이 아닌 설명, 중복 표, 오래된 표현은 README가 아니라 원문 문서에서 정리한다.
