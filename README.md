# Docs README

## 1. 개요

이 `docs` 폴더는 Dondok MVP의 제품 요구사항, 도메인 설계, 데이터 구조, API 계약, 실행 계획을 한 곳에서 관리하기 위한 문서 루트다. 목적은 세 가지다.

- 제품 규칙과 구현 규칙의 기준 문서를 분리한다.
- 신규 개발자가 코드를 보기 전에 전체 구조를 빠르게 이해할 수 있게 한다.
- 같은 내용을 여러 문서에 반복 작성하지 않도록 source of truth를 고정한다.

문서 계층은 아래 순서로 이해하면 된다.

1. 기획: 무엇을 만들고 왜 만드는가
2. 설계: 어떤 규칙과 구조로 동작하는가
3. 계약: FE/BE가 어떤 인터페이스로 연결되는가
4. 실행: 어떤 순서로 구현하고 검증하는가

원본 기획안(`.docx`)은 출발점이자 제품/UX 배경 참고 자료다. 현재 활성 제품/UX 참고 문서는 `docs/Dondok_프로젝트기획안_v1.9.docx`이며, 구현과 운영 판단의 기준은 여전히 이 폴더의 Markdown 문서(`docs/*.md`)다. 제안서 표현이 Markdown 문서와 충돌하면 Markdown 문서가 우선한다.

## 2. 추천 읽기 순서

완전히 처음 보는 개발자는 먼저 제품 의미와 API authority lane을 분리해서 읽는다. 순서의 목적은 "제품 의도 → semantic guardrail → active API contract → 파생 구현 문서"를 끊기지 않게 이해하는 데 있다.

### 기본 온보딩 순서

1. [PRD-dondok.md](./PRD-dondok.md)
   제품 목표, MVP 범위, 핵심 비즈니스 규칙을 먼저 이해한다. 이 문서는 API authority가 아니라 제품 semantic guardrail의 최상위 문서다.
2. [Usecase-dondok.md](./Usecase-dondok.md)
   PRD의 제품 의미가 화면, API, 정산, QA로 전파될 때 흔들리기 쉬운 권한 경계와 drift warning을 확인한다.
3. `backend/docs/api/overview.md`와 `backend/docs/api/*.md`
   MVP active endpoint inventory, method/path, request/response shape, 공통 API 규칙, active/deferred boundary를 확인한다. 이 경로가 active API source of truth다.
4. [API-spec-dondok.md](./API-spec-dondok.md)
   `backend/docs/api/*` 기준으로 동기화된 integrated API contract를 확인한다. 이 문서는 legacy 단독 authority가 아니며, backend API 계약을 통합해 FE/BE/QA가 소비하기 위한 문서다.
5. [ERD-dondok.md](./ERD-dondok.md), [Schema-migration-spec.md](./Schema-migration-spec.md), [Settlement-design.md](./Settlement-design.md)
   API contract stabilization 이후 정렬되는 derived implementation docs로 읽는다. 데이터 구조, 마이그레이션, 정산 상세 구현 판단에는 여전히 각 문서의 도메인별 상세 규칙을 따른다.
6. [Dondok_요구사항명세서_v0.9.xlsx](./Dondok_요구사항명세서_v0.9.xlsx), wireframe/QA 자료, [Dondok_WBS_v0.6.xlsx](./Dondok_WBS_v0.6.xlsx), 외부 WBS/GitHub Issues
   마지막으로 실행 계획과 검증 단위를 본다. 여기서는 무엇을 만들지 다시 정의하지 않고, 이미 결정된 범위를 어떤 단위로 구현하고 검증할지 확인한다.
7. [Dondok_프로젝트기획안_v1.9.docx](./Dondok_프로젝트기획안_v1.9.docx)
   활성 제품/UX 참고 자료다. 배경, 화면 의도, 소셜 피드 표현을 확인할 때만 본다. 구현 판단은 Markdown source of truth 문서에 승격된 내용만 따른다.

### API contract lane와 semantic guardrail lane

- API contract lane: `backend/docs/api/*` → [API-spec-dondok.md](./API-spec-dondok.md) → FE/BE/QA 연동과 구현.
- Semantic guardrail lane: [PRD-dondok.md](./PRD-dondok.md) → [Usecase-dondok.md](./Usecase-dondok.md) → API/정산/ERD 해석 경계.
- `backend/docs/api/*`는 active endpoint/method/path/request/response authority지만 PRD/Usecase의 핵심 semantic guardrail을 override하지 않는다.
- PRD/Usecase는 semantic guardrail이지만 `backend/docs/api/*`에 없는 endpoint/status/field를 단독으로 active API contract에 승격하지 않는다.
- `Deferred`, `Brownfield`, `Removed`, `Contract Drift Notes`는 historical/reference only다. future delivery commitment나 implementation permission으로 읽지 않는다.

## 3. 문서별 역할 정의

| 문서 | 한 줄 요약 | 언제 읽는가 | Source of Truth |
| --- | --- | --- | --- |
| [PRD-dondok.md](./PRD-dondok.md) | MVP의 목표, 범위, 사용자 가치, 핵심 비즈니스 규칙을 정의한다. | 프로젝트 온보딩 시작 시, 제품 정책 변경 검토 시 | 비즈니스 요구사항과 MVP semantic guardrail |
| [Usecase-dondok.md](./Usecase-dondok.md) | PRD 의미가 downstream 문서로 전파될 때 필요한 권한 경계와 drift warning을 정리한다. | 정산/화면/API/QA 해석이 갈릴 때 | PRD 의미의 semantic bridge |
| `backend/docs/api/*` | MVP active endpoint inventory, method/path, request/response, 공통 API 규칙을 정의한다. | API 구현, FE 연동, QA contract 확인 시 | MVP active API source |
| [API-spec-dondok.md](./API-spec-dondok.md) | backend API 문서 기준으로 동기화된 통합 API 계약을 제공한다. | 화면 연동, API 구현, QA 시나리오 작성 시 | Integrated synchronized API contract |
| [Implementation-guardrails.md](./Implementation-guardrails.md) | API resurrection 방지와 lifecycle/balance/ledger/settlement implementation guardrail을 고정한다. | 구현 전/리뷰 시 semantic drift를 차단할 때 | 구현 guardrail |
| [Settlement-design.md](./Settlement-design.md) | 정산 계산, 상태 흐름, 멱등성, 동시성, 실패/재시도 정책을 정의한다. | 정산/포인트/배치 구현 또는 운영 정책 검토 시 | API 확정 후 정렬되는 정산 도메인 상세 규칙 |
| [ERD-dondok.md](./ERD-dondok.md) | 핵심 테이블, 관계, 제약, 스냅샷 저장 구조를 정의한다. | DB 설계, 쿼리 설계, 영속성 모델 검토 시 | API 확정 후 정렬되는 데이터 구조와 DB 제약 |
| [Schema-migration-spec.md](./Schema-migration-spec.md) | 스키마 마이그레이션 단위와 검증 기준을 정의한다. | DB 변경/마이그레이션 검토 시 | API 확정 후 정렬되는 migration detail |
| [Dondok_요구사항명세서_v0.9.xlsx](./Dondok_요구사항명세서_v0.9.xlsx) | 상세 요구사항 참고 자료다. | 요구사항 세부 항목과 검증 단위를 확인할 때 | Downstream 요구사항 reference |
| [Dondok_프로젝트기획안_v1.9.docx](./Dondok_프로젝트기획안_v1.9.docx) | 활성 제품/UX 참고 자료다. | 배경, UX 의도, 제안서 표현 확인이 필요할 때 | 현재 source of truth 아님. 참고용 입력 |
| [Dondok_WBS_v0.6.xlsx](./Dondok_WBS_v0.6.xlsx) | MVP 일정/구현 범위 참고 자료다. | 실행 우선순위와 일정 확인이 필요할 때 | Downstream 실행 reference |

문서 충돌 시에는 단일 선형 우선순위가 아니라 소유 영역별 authority를 따른다.

- API surface: `backend/docs/api/*`가 active endpoint/method/path/request/response source이고, [API-spec-dondok.md](./API-spec-dondok.md)는 이를 통합 동기화한 계약이다.
- Product semantics: [PRD-dondok.md](./PRD-dondok.md)와 [Usecase-dondok.md](./Usecase-dondok.md)가 semantic guardrail이다. API 편의 응답이 lifecycle/settlement/ledger authority처럼 보이면 semantic drift로 본다.
- Derived implementation docs: [ERD-dondok.md](./ERD-dondok.md), [Schema-migration-spec.md](./Schema-migration-spec.md), [Settlement-design.md](./Settlement-design.md)는 API contract stabilization 이후 정렬한다. README가 이 문서들을 active API authority로 승격하지 않는다.
- Deferred/Brownfield/Removed/Contract Drift Notes: historical/reference only다. active feature approval, future roadmap commitment, implementation permission이 아니다.
- 요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues·implementation/infra decision docs: 실행·검증·구현 참고이며 상위 authority를 재정의하지 않는다.

## 3.1 Canonical Freeze v1 적용 메모

이 README는 세부 정책을 재정의하지 않고 후속 패치의 drift를 막는 짧은 guardrail만 둔다. Canonical Freeze v1 기준:

- Host moderation authority는 settlement authority가 아니다. 방장 검수/조정 권한은 정산 결과를 직접 수정하는 권한으로 해석하지 않는다.
- 72h grace는 pre-freeze correction window일 뿐이며, final 3 mission days는 즉시 freeze된다. Post-freeze mutation은 금지된다.
- `NOTIFY-003`은 projection 기반 알림이며 final settlement guarantee가 아니다. 상세 event/API 문구는 `API-spec` 후속 propagation에서 정렬한다.
- `point_history`는 authoritative append-only ledger이고, `point_account.balance`는 projection/cache layer다. 이 경계의 source of truth는 `ERD`다.
- 크루 참여 lifecycle은 `PENDING`(신청 제출 + 예치금 reserve)과 `LOCKED`(방장 승인 후 확정)를 사용한다. `PENDING`은 사용 가능 잔액을 줄이고 reserve projection을 늘리며 capacity reservation에는 포함되지만 activation eligibility/minimum baseline/frozen baseline/settlement eligibility에는 포함되지 않는다. `LOCKED`만 activation/minimum/frozen participant baseline에 포함된다.
- EXIF/`image_hash`는 fraud/risk signal이며 인증/정산 authority가 아니다. `image_hash`는 서버가 S3 object에서 직접 계산한 SHA-256이고 클라이언트 제출 값을 신뢰하지 않는다.
- FCM/알림 inbox는 non-authoritative UX hint다. delivery attempt나 read state를 settlement evidence/lifecycle authority로 끌어올리지 않는다.
- `moderation_history`는 append-only audit trail이다. 기존 레코드를 수정/삭제하지 않으며, host moderation 결정도 mission_log 컬럼 update + history row append로만 진행한다. MVP에서 admin/correction workflow는 존재하지 않는다.

API authority propagation에서도 아래 semantic anchor는 유지한다.

- projection != final settlement.
- notification/inbox/read state != canonical state.
- retry != correction/replay/recalculation.
- host != lifecycle/settlement/ledger authority.
- all-fail = equal principal refund.
- `settlement_item` + `point_history` linkage가 final settlement authority다.
- API convenience/display/projection field != authoritative state.

## 4. 문서 간 관계 구조

문서 간 의존 방향은 아래와 같다.

- `PRD -> Usecase`
  PRD의 제품 의미와 권한 경계를 downstream 문서가 소비할 수 있는 semantic bridge로 정리한다.
- `backend/docs/api/* -> API-spec`
  backend API 문서가 MVP active API source이고, `API-spec`은 이를 통합 동기화한 FE/BE/QA 계약이다.
- `PRD/Usecase -> API-spec 해석`
  API response convenience field, projection, notification/inbox/read 상태가 lifecycle/settlement/ledger authority처럼 읽히면 semantic drift로 본다.
- `API-spec + PRD/Usecase guardrail -> ERD/Schema/Settlement-design`
  API contract stabilization 이후 derived implementation docs를 정렬한다. 이 단계에서 ERD/Settlement가 API authority보다 상위인 것처럼 역전하지 않는다.
- `API-spec + 요구사항 명세서 + wireframe/QA + 외부 WBS/GitHub Issues`
  구현 단위와 검증 단위는 상위 의미를 재정의하지 않고 실행 계획으로만 소비한다.

간단히 보면 구조는 아래와 같다.

```text
제품/의미 lane: PRD -> Usecase semantic bridge
                         │
                         ├─ semantic guardrail
                         ↓
API lane:       backend/docs/api/* -> API-spec-dondok.md
                         │
                         ├─ API 확정 후 propagation
                         ↓
Derived docs:   ERD / Schema-migration / Settlement-design
                         │
                         ↓
Execution:      요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues
```

`Deferred`, `Brownfield`, `Removed`, `Contract Drift Notes`는 위 active lane 밖의 historical/reference surface다. 별도 승인 없이 구현 범위나 roadmap promise로 해석하지 않는다.

## 5. 중복 방지 규칙

아래 규칙을 기준으로 문서 책임을 분리한다.

- 비즈니스 목표, 사용자 가치, MVP 포함/제외 범위는 `PRD`에만 정의한다.
- PRD 의미의 downstream 해석 경계와 drift warning은 `Usecase`에 둔다.
- active API endpoint inventory, method/path, request/response shape, 공통 API 규칙, API enum, active/deferred boundary는 `backend/docs/api/*`가 소유하고 `API-spec`이 통합 동기화한다.
- 정산 로직, 정산 상태 흐름, 재시도, 멱등성, 실패 코드 의미의 구현 상세는 `Settlement-design`에 둔다. 단, API 확정 전후에 active endpoint를 새로 만들거나 되살리는 authority로 사용하지 않는다.
- 테이블, 컬럼, FK, unique 제약, 스냅샷 저장 구조는 `ERD`에만 상세 정의한다. 단, ERD가 backend active API에 없는 surface를 구현 대상으로 승격하지 않는다.
- 스키마 변경 단위와 마이그레이션 검증은 `Schema-migration-spec`에 둔다. 단, API contract stabilization 이후 정렬 대상으로 다룬다.
- `Deferred`, `Brownfield`, `Removed`, `Contract Drift Notes`는 historical/reference only다. future delivery commitment, active feature approval, implementation permission으로 복제하지 않는다.
- 실행 우선순위, 선행 의존성, 병렬 실행 순서는 요구사항 명세서, wireframe/QA 자료, 외부 WBS, GitHub Issues에서 추적하되 상위 제품/정산/API 의미를 재정의하지 않는다.
- 활성 기획안(`docs/Dondok_프로젝트기획안_v1.9.docx`)의 표현이 현재 Markdown 문서와 다르면 docx를 직접 구현 기준으로 사용하지 않는다. 필요한 내용은 먼저 소유 Markdown source of truth 문서로 승격한 뒤 구현한다.
- AI 비트랜잭션 경계는 `PRD`, `Usecase`, `backend/docs/api/*`, `API-spec`, `ERD`가 같은 정책을 유지해야 한다.
- DB enum/constraint와 `member.uuid` identity persistence invariant의 source of truth는 `ERD`가 소유하고, `API-spec`은 FE/BE 계약에 필요한 consumer-facing enum, auth subject, 알림 recipient/transport contract만 반복한다.

다른 문서에는 필요한 만큼만 요약하고, 상세 규칙은 source of truth 문서로 링크한다. 같은 표나 enum, 같은 계산식, 같은 endpoint 목록을 여러 문서에서 독립적으로 재정의하지 않는다.

## 6. 수정 영향 범위 규칙

문서 수정은 아래 영향 범위를 기본으로 점검한다.

- `backend/docs/api/*` 수정:
  MVP active API source가 바뀐 것이므로 [API-spec-dondok.md](./API-spec-dondok.md)를 먼저 동기화하고, PRD/Usecase semantic guardrail 위반 여부를 확인한 뒤 README/Implementation guardrail 및 derived docs 전파를 검토한다.
- `API-spec` 수정:
  원칙적으로 `backend/docs/api/*`에서 동기화된 integrated contract 변경이어야 한다. FE 화면, BE 구현, QA 시나리오, 요구사항 명세서, 외부 WBS/GitHub Issues가 영향을 받으며, Deferred/Brownfield/Removed surface를 active로 되살리지 않았는지 확인한다.
- `PRD` 수정:
  제품 규칙이 바뀌면 `Usecase`, `backend/docs/api/*`, `API-spec`, `Settlement-design`, `ERD`, `Schema-migration-spec`, 요구사항 명세서, wireframe/QA, 외부 WBS/GitHub Issues까지 연쇄 영향이 있는지 확인한다.
- `Usecase` 수정:
  semantic bridge나 drift warning이 바뀌면 `backend/docs/api/*`, `API-spec`, `Settlement-design`, `ERD`, QA 시나리오, wireframe copy 영향 여부를 확인한다.
- `Implementation-guardrails` 수정:
  구현 금지/허용 경계가 바뀌면 `backend/docs/api/*`, `API-spec`, PRD/Usecase semantic guardrail과 충돌하지 않는지 확인한다.
- `Settlement-design` 수정:
  정산 상태, 계산 규칙, 재시도 정책이 바뀌면 `backend/docs/api/*`, `API-spec`, `ERD`, 요구사항 명세서, QA 시나리오 영향 여부를 함께 검토한다. API active inventory를 우회해 새 endpoint/status를 만들지 않는다.
- `ERD` 수정:
  테이블 구조나 제약이 바뀌면 `backend/docs/api/*`, `API-spec`, 정산 설계, 요구사항 명세서, QA 시나리오 영향 여부를 확인한다.
- `Schema-migration-spec` 수정:
  마이그레이션 기준이 바뀌면 ERD와 API contract alignment를 확인한다.
- 요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues 수정:
  실행 계획 변경이므로 상위 요구사항 자체를 바꾸지 않는다. 상위 규칙이 바뀌면 먼저 소유 source 문서를 수정한다.

실행 문서가 상위 문서를 덮어쓰면 안 된다. 규칙 변경은 항상 소유 source 문서에서 먼저 반영한 뒤 downstream 문서로 전파한다.

## 7. 신규 기여자 가이드

코드를 보기 전에 최소한 `README -> PRD -> 자신의 역할에 맞는 기준 문서` 순서로 읽는다.

### Backend 개발자

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. `backend/docs/api/overview.md` + 담당 도메인 `backend/docs/api/*.md`
4. [API-spec-dondok.md](./API-spec-dondok.md)
5. [Implementation-guardrails.md](./Implementation-guardrails.md)
6. [Settlement-design.md](./Settlement-design.md), [ERD-dondok.md](./ERD-dondok.md), [Schema-migration-spec.md](./Schema-migration-spec.md)
7. [runbooks/settlement-recovery.md](./runbooks/settlement-recovery.md)
8. 외부 WBS/GitHub Issues

이 순서가 필요한 이유:
백엔드 구현은 active API source를 먼저 고정하고, PRD/Usecase semantic guardrail과 implementation guardrail을 함께 확인해야 legacy/candidate surface를 되살리지 않는다.

### Frontend 개발자

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. [API-spec-dondok.md](./API-spec-dondok.md)
4. 필요 시 `backend/docs/api/overview.md` + 담당 도메인 `backend/docs/api/*.md`
5. [Settlement-design.md](./Settlement-design.md)
6. 요구사항 명세서 / wireframe / QA 자료
7. 외부 WBS/GitHub Issues

이 순서가 필요한 이유:
화면은 제품 흐름과 API 계약에 직접 연결된다. `API-spec`은 통합 계약이지만 active source는 `backend/docs/api/*`이며, 정산/인증/알림처럼 오해하기 쉬운 상태는 semantic guardrail을 같이 알아야 UI 해석 오류를 줄일 수 있다.

### Infra / DevOps

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. `backend/docs/api/overview.md` + 운영 영향 도메인 `backend/docs/api/*.md`
4. [API-spec-dondok.md](./API-spec-dondok.md)
5. [Settlement-design.md](./Settlement-design.md)
6. [ERD-dondok.md](./ERD-dondok.md), [Schema-migration-spec.md](./Schema-migration-spec.md)
7. [runbooks/settlement-recovery.md](./runbooks/settlement-recovery.md)
8. 외부 implementation/infra decision docs

이 순서가 필요한 이유:
배치 스케줄, 재시도, 멱등성, 포인트 원장, 운영 복구 경로는 API contract와 도메인 guardrail에 직접 영향을 받는다.

## 8. 운영 원칙

- 새로운 정책을 추가할 때는 먼저 source of truth 문서를 결정하고 그 문서에 반영한다. API surface는 `backend/docs/api/*`에서 시작해 `API-spec`으로 동기화하고, 제품 의미는 PRD/Usecase guardrail을 유지한다.
- 구현 세부사항이 문서와 충돌하면, 코드보다 먼저 문서 기준이 맞는지 확인한다.
- 더 이상 기준이 아닌 설명, 중복 표, 오래된 표현은 README가 아니라 원문 문서에서 정리한다.
