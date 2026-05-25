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

원본 기획안(`.docx`)은 출발점이자 제품/UX 배경 참고 자료다. 현재 활성 제품/UX 참고 문서는 `docs/Dondok_프로젝트기획안_v1.3.docx`이며, 구현과 운영 판단의 기준은 여전히 이 폴더의 Markdown 문서(`docs/*.md`)다. 제안서 표현이 Markdown 문서와 충돌하면 Markdown 문서가 우선한다.

## 2. 추천 읽기 순서

완전히 처음 보는 개발자는 아래 순서로 읽는다. 순서의 목적은 "제품 의도 → 도메인 규칙 → 외부 계약 → 실행 단위"를 끊기지 않게 이해하는 데 있다.

1. [PRD-dondok.md](./PRD-dondok.md)
   제품 목표, MVP 범위, 핵심 비즈니스 규칙을 먼저 이해한다. 이 문서를 읽지 않고 설계나 API부터 보면 왜 그런 제약이 있는지 놓치기 쉽다.
2. [Usecase-dondok.md](./Usecase-dondok.md)
   PRD의 제품 의미가 화면, API, 정산, QA로 전파될 때 흔들리기 쉬운 권한 경계와 drift warning을 확인한다.
3. [Settlement-design.md](./Settlement-design.md) + [ERD-dondok.md](./ERD-dondok.md)
   PRD와 Usecase bridge를 읽은 뒤에는 정산 규칙과 도메인 구조를 같이 본다. `Settlement-design`은 정산/포인트/동시성/재시도 같은 운영 규칙을 설명하고, `ERD`는 데이터 모델의 경계를 설명한다. 두 문서를 함께 봐야 데이터 구조와 비즈니스 계산 규칙이 연결된다.
4. [API-spec-dondok.md](./API-spec-dondok.md)
   앞선 문서들을 기반으로 FE/BE 계약을 확인한다. API는 독립 출발점이 아니라 PRD, Usecase bridge, 정산 설계, ERD를 외부 인터페이스로 고정한 결과물이다.
5. [Dondok_요구사항명세서_v0.7.xlsx](./Dondok_요구사항명세서_v0.7.xlsx), wireframe/QA 자료, 외부 WBS/GitHub Issues
   마지막으로 실행 계획과 검증 단위를 본다. 여기서는 무엇을 만들지 다시 정의하지 않고, 이미 결정된 범위를 어떤 단위로 구현하고 검증할지 확인한다.
6. [Dondok_프로젝트기획안_v1.3.docx](./Dondok_프로젝트기획안_v1.3.docx)
   활성 제품/UX 참고 자료다. 배경, 화면 의도, 소셜 피드 표현을 확인할 때만 본다. 구현 판단은 Markdown source of truth 문서에 승격된 내용만 따른다.

## 3. 문서별 역할 정의

| 문서                                                            | 한 줄 요약                                                            | 언제 읽는가                                     | Source of Truth                        |
| --------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------- |
| [PRD-dondok.md](./PRD-dondok.md)                        | MVP의 목표, 범위, 사용자 가치, 핵심 비즈니스 규칙을 정의한다.         | 프로젝트 온보딩 시작 시, 제품 정책 변경 검토 시 | 비즈니스 요구사항과 MVP 범위           |
| [Usecase-dondok.md](./Usecase-dondok.md)                | PRD 의미가 downstream 문서로 전파될 때 필요한 권한 경계와 drift warning을 정리한다. | 정산/화면/API/QA 해석이 갈릴 때                 | PRD 의미의 semantic bridge             |
| [Settlement-design.md](./Settlement-design.md)                  | 정산 계산, 상태 흐름, 멱등성, 동시성, 실패/재시도 정책을 정의한다.    | 정산/포인트/배치 구현 또는 운영 정책 검토 시    | 정산 도메인 규칙과 운영 안전성 정책    |
| [ERD-dondok.md](./ERD-dondok.md)                        | 핵심 테이블, 관계, 제약, 스냅샷 저장 구조를 정의한다.                 | DB 설계, 쿼리 설계, 영속성 모델 검토 시         | 데이터 구조와 DB 제약                  |
| [API-spec-dondok.md](./API-spec-dondok.md)              | FE/BE가 공유하는 엔드포인트, 요청/응답, 에러, 상태값 계약을 정의한다. | 화면 연동, API 구현, QA 시나리오 작성 시        | 외부 API 계약. 알림 transport 세부는 API 및 외부 implementation/infra decision docs 후속 propagation 대상 |
| [Dondok_요구사항명세서_v0.7.xlsx](./Dondok_요구사항명세서_v0.7.xlsx) | 상세 요구사항 참고 자료다.                                            | 요구사항 세부 항목과 검증 단위를 확인할 때      | Downstream 요구사항 reference          |
| [Dondok_프로젝트기획안_v1.3.docx](./Dondok_프로젝트기획안_v1.3.docx) | 활성 제품/UX 참고 자료다.                                             | 배경, UX 의도, 제안서 표현 확인이 필요할 때     | 현재 source of truth 아님. 참고용 입력 |

문서 충돌 시 우선순위는 아래를 따른다.

1. PRD: 제품 목표와 비즈니스 규칙
2. Usecase: PRD 의미의 semantic bridge와 drift warning
3. Settlement-design: 정산 도메인 규칙과 운영/runtime semantics
4. ERD: 데이터 구조와 DB 제약
5. API-spec: 외부 인터페이스 계약
6. 요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues·implementation/infra decision docs: 실행·검증·구현 참고

단, 외부 API 계약은 `API-spec`이 우선하고, identity persistence invariant와 원장/캐시 경계는 `ERD`가 우선한다. 알림 transport와 NOTIFY-003 semantics는 API-spec 및 외부 implementation/infra decision docs의 후속 propagation 대상이다.
7. 활성 기획안 docx: 참고 자료

## 3.1 Canonical Freeze v1 적용 메모

이 README는 세부 정책을 재정의하지 않고 후속 패치의 drift를 막는 짧은 guardrail만 둔다. Canonical Freeze v1 기준:

- Host moderation authority는 settlement authority가 아니다. 방장 검수/조정 권한은 정산 결과를 직접 수정하는 권한으로 해석하지 않는다.
- 72h grace는 pre-freeze correction window일 뿐이며, final 3 mission days는 즉시 freeze된다. Post-freeze mutation은 금지된다.
- `NOTIFY-003`은 projection 기반 알림이며 final settlement guarantee가 아니다. 상세 event/API 문구는 `API-spec` 후속 propagation에서 정렬한다.
- `point_history`는 authoritative append-only ledger이고, `point_account.balance`는 projection/cache layer다. 이 경계의 source of truth는 `ERD`다.
- 크루 참여 lifecycle 중 `APPLIED`는 보증금 lock 전 신청 상태이고, capacity/activation eligibility/minimum baseline/frozen baseline/settlement eligibility 어디에도 포함되지 않는다. 방장 승인 = 자동 보증금 lock trigger이며 단일 transaction 내 lock 성공 시 즉시 `JOINED`로 전이한다. Activation/min baseline은 `JOINED` participant만 센다.
- EXIF/`image_hash`는 fraud/risk signal이며 인증/정산 authority가 아니다. `image_hash`는 서버가 S3 object에서 직접 계산한 SHA-256이고 클라이언트 제출 값을 신뢰하지 않는다.
- FCM/알림 inbox는 non-authoritative UX hint다. delivery attempt나 read state를 settlement evidence/lifecycle authority로 끌어올리지 않는다.
- `moderation_history`는 append-only audit trail이다. 기존 레코드를 수정/삭제하지 않으며, host moderation 결정도 mission_log 컬럼 update + history row append로만 진행한다. MVP에서 admin/correction workflow는 존재하지 않는다.

## 4. 문서 간 관계 구조

문서 간 의존 방향은 아래와 같다.

- `PRD -> Usecase`
  PRD의 제품 의미와 권한 경계를 downstream 문서가 소비할 수 있는 semantic bridge로 정리한다.
- `Usecase -> Settlement-design`
  drift warning과 권한 경계를 정산 정책, 예외 처리, 운영 규칙에 반영한다.
- `Settlement-design -> ERD`
  정산/runtime semantics가 필요한 스냅샷, 원장, 제약 구조를 데이터 모델에서 확인하게 한다.
- `ERD + Settlement-design -> API-spec`
  API는 데이터 모델과 도메인 규칙을 외부 계약으로 노출한 결과다.
- `API-spec + 요구사항 명세서 + wireframe/QA + 외부 WBS/GitHub Issues`
  구현 단위와 검증 단위는 상위 의미를 재정의하지 않고 실행 계획으로만 소비한다.

간단히 보면 구조는 아래와 같다.

```text
활성 기획안(docx)
    ↓
PRD
    ↓
Usecase semantic bridge
    ↓
Settlement-design
    ↓
ERD
    ↓
API-spec
    ↓
요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues
```

## 5. 중복 방지 규칙

아래 규칙을 기준으로 문서 책임을 분리한다.

- 비즈니스 목표, 사용자 가치, MVP 포함/제외 범위는 `PRD`에만 정의한다.
- 정산 로직, 정산 상태 흐름, 재시도, 멱등성, 실패 코드 의미는 `Settlement-design`에만 상세 정의한다.
- 테이블, 컬럼, FK, unique 제약, 스냅샷 저장 구조는 `ERD`에만 상세 정의한다.
- 엔드포인트, 요청/응답 JSON, 에러 응답, API enum, 알림 transport/API 외부 계약은 `API-spec`에만 상세 정의한다.
- 실행 우선순위, 선행 의존성, 병렬 실행 순서는 요구사항 명세서, wireframe/QA 자료, 외부 WBS, GitHub Issues에서 추적하되 상위 제품/정산/API 의미를 재정의하지 않는다.
- 활성 기획안(`docs/Dondok_프로젝트기획안_v1.3.docx`)의 표현이 현재 Markdown 문서와 다르면 docx를 직접 구현 기준으로 사용하지 않는다. 필요한 내용은 먼저 소유 Markdown source of truth 문서로 승격한 뒤 구현한다.
- AI 비트랜잭션 경계는 `PRD`, `API-spec`, `ERD`가 같은 정책을 유지해야 한다.
- DB enum/constraint와 `member.uuid` identity persistence invariant의 source of truth는 `ERD`가 소유하고, `API-spec`은 FE/BE 계약에 필요한 consumer-facing enum, auth subject, 알림 recipient/transport contract만 반복한다. 예: `point_history.reference_type`, `settlement_status`, `settlement_type`, `frequency_type`, `transaction_type`.

다른 문서에는 필요한 만큼만 요약하고, 상세 규칙은 source of truth 문서로 링크한다. 같은 표나 enum, 같은 계산식을 여러 문서에 복제하지 않는다.

## 6. 수정 영향 범위 규칙

문서 수정은 아래 영향 범위를 기본으로 점검한다.

- `PRD` 수정:
  제품 규칙이 바뀌면 `Usecase`, `Settlement-design`, `ERD`, `API-spec`, 요구사항 명세서, wireframe/QA, 외부 WBS/GitHub Issues까지 연쇄 영향이 있는지 확인한다.
- `Usecase` 수정:
  semantic bridge나 drift warning이 바뀌면 `Settlement-design`, `ERD`, `API-spec`, QA 시나리오, wireframe copy 영향 여부를 확인한다.
- `Settlement-design` 수정:
  정산 상태, 계산 규칙, 재시도 정책이 바뀌면 `ERD`, `API-spec`, 요구사항 명세서, QA 시나리오 영향 여부를 함께 검토한다.
- `ERD` 수정:
  테이블 구조나 제약이 바뀌면 `API-spec`, 정산 설계, 요구사항 명세서, QA 시나리오 영향 여부를 확인한다.
- `API-spec` 수정:
  FE 화면, BE 구현, QA 시나리오, 요구사항 명세서, 외부 WBS/GitHub Issues가 영향을 받는다.
- 요구사항 명세서 / wireframe / QA / 외부 WBS·GitHub Issues 수정:
  실행 계획 변경이므로 상위 요구사항 자체를 바꾸지 않는다. 상위 규칙이 바뀌면 먼저 `PRD`, `Usecase`, `Settlement-design`, `ERD`, `API-spec`를 수정한다.

실행 문서가 상위 문서를 덮어쓰면 안 된다. 규칙 변경은 항상 상위 문서에서 먼저 반영한다.

## 7. 신규 기여자 가이드

코드를 보기 전에 최소한 `README -> PRD -> 자신의 역할에 맞는 기준 문서` 순서로 읽는다.

### Backend 개발자

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. [Settlement-design.md](./Settlement-design.md)
4. [ERD-dondok.md](./ERD-dondok.md)
5. [API-spec-dondok.md](./API-spec-dondok.md)
6. [runbooks/settlement-recovery.md](./runbooks/settlement-recovery.md)
7. 외부 WBS/GitHub Issues

이 순서가 필요한 이유:
정산, 포인트, 참가/탈퇴, 원장, 배치 같은 핵심 도메인 규칙이 데이터 구조와 강하게 연결되어 있기 때문이다.

### Frontend 개발자

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. [API-spec-dondok.md](./API-spec-dondok.md)
4. [Settlement-design.md](./Settlement-design.md)
5. 요구사항 명세서 / wireframe / QA 자료
6. 외부 WBS/GitHub Issues

이 순서가 필요한 이유:
화면은 제품 흐름과 API 계약에 직접 연결되고, 정산/인증처럼 오해하기 쉬운 상태는 `Settlement-design`의 기준을 같이 알아야 UI 해석 오류를 줄일 수 있다.

### Infra / DevOps

권장 순서:

1. [PRD-dondok.md](./PRD-dondok.md)
2. [Usecase-dondok.md](./Usecase-dondok.md)
3. [Settlement-design.md](./Settlement-design.md)
4. [ERD-dondok.md](./ERD-dondok.md)
5. [API-spec-dondok.md](./API-spec-dondok.md)
6. [runbooks/settlement-recovery.md](./runbooks/settlement-recovery.md)
7. 외부 implementation/infra decision docs

이 순서가 필요한 이유:
배치 스케줄, 재시도, 멱등성, 포인트 원장, 운영 복구 경로는 인프라와 운영 설계에 직접 영향을 준다.

## 8. 운영 원칙

- 새로운 정책을 추가할 때는 먼저 source of truth 문서를 결정하고 그 문서에 반영한다. 제안서 내용도 PRD/API/ERD/Settlement-design 중 소유 문서에 승격된 뒤에만 구현 계약이 된다.
- 구현 세부사항이 문서와 충돌하면, 코드보다 먼저 문서 기준이 맞는지 확인한다.
- 더 이상 기준이 아닌 설명, 중복 표, 오래된 표현은 README가 아니라 원문 문서에서 정리한다.
