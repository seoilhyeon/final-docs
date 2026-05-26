# PRD: 돈독 (Dondok)

작성 기준 문서: `docs/Dondok_프로젝트기획안.docx` 및 최신 semantic reconciliation / freeze 결과. 이 PRD는 Dondok의 최상위 원본 intent source가 아니라, 최신 기획안과 폐기되지 않은 최종 합의를 안정적으로 정리한 canonical synthesis layer다. 오래된 회의 로그, 중간 검토안, rollback된 semantics, unresolved 논의는 L1 intent authority로 취급하지 않는다.

ERD/API/Settlement/Test 문서는 이 PRD synthesis를 기준으로 후속 propagation 단계에서 정렬되는 derived implementation docs다. 단, PRD 자체도 L1 intent source를 override하는 constitution이 아니며, semantic authority의 최상위 기준은 최신 기획안과 최신 accepted freeze 결과다.

## Canonical Synthesis Register

이 register는 PRD가 독자적으로 제품 의미를 freeze하는 constitution이 아니라, L1 intent authority를 제품/정책 문장으로 안정화한 synthesis임을 전제로 한다. lifecycle, activation, cadence, settlement timing, participant baseline, P0 taxonomy, engagement UX scope의 semantic authority는 최신 기획안과 latest accepted semantic freeze 결과에서 온다. PRD는 그 결과를 downstream alignment가 가능하도록 정리한다.

### Latest accepted product semantics

- Dondok은 남의 실패를 벌주는 서비스가 아니라, 각자가 약속을 얼마나 꾸준히 지켰는지에 따라 공동 보증금을 투명하고 공정하게 다시 배분받는 상호 책임형 습관 계약 플랫폼이다.
- 참여도 차이에 따라 환급금은 달라질 수 있다. 다만 핵심은 타인의 실패를 유도하는 경쟁이 아니라 함께 성실함을 지속하게 만드는 구조에 있다.
- Dondok의 emotional priority는 `계약 신뢰 > 상호 책임 기반 성장 > 경쟁 긴장감`이다. 경쟁 긴장감과 상대적 환급 차이는 허용 가능하지만, 제품의 emotional center는 환급 경쟁이 아니라 “누가 얼마나 꾸준히 함께 버텼는가”에 둔다.
- 정산 UX 우선순위는 `설명 가능성/감정적 수용성 > authoritative 정확성 > 예상값 고정성`이다.
- 예상 환급금은 anxiety reduction과 settlement explanation을 위한 현재 기준 projection이며 최종 정산금이 아니다. 최종 정산은 authoritative batch 결과로 확정된다.
- 실시간 지분율, 상대적 순위/위치, 예상 환급금, 기여도, 결과 카드, 알림 재진입은 engagement visibility로 유지한다. 이는 사용자가 현재 흐름을 이해하고 다시 돌아오게 만드는 UX mechanics이며, 최종 정산·원장·payout authority가 아니다.
- 정산은 deterministic, explainable, replayable 해야 한다.
- 전체 인정 성공 기록이 없는 all-fail 상황에서는 누군가의 실패가 다른 참여자의 추가 환급으로 이어지지 않도록 equal principal refund를 적용한다.
- 사용자 화면에서 도딘(Dodin)은 보증금·환급 UX를 표현하는 user-facing app-money branding이며, authoritative accounting은 point ledger/history가 담당한다. 도딘은 별도 coin, 외부 현금, 인출 가능 자산, 또는 별도 ledger가 아니다.
- 원단위 절사 잔액은 deterministic/replayable calculation rule로 처리한다. 이는 host reward, host authority, 또는 host privilege가 아니다.
- 모집 마감까지 최소 인원 충족 + 승인 + 예치 Lock 완료 + host disband 없음이면 미션은 start_at 기준 자동 ACTIVE가 되며, MVP에서 activated_at = start_at이다. Host는 activation authority가 아니다.
- 인증 검증은 layered trust model을 따른다: `server_time`은 timing 기준, EXIF/hash는 fraud/risk signal, moderation은 contextual review, final batch는 authoritative settlement snapshot이다.
- 방장은 인증 moderation authority를 가지며 이 결정은 정산 입력에 영향을 줄 수 있다. 단, 방장은 정산 엔진, 정산 금액, 포인트 원장을 직접 조작할 수 없다.
- moderation history는 append-only/auditable 해야 한다.
- final settlement 이후 결과는 immutable finality를 가진다. replay는 당시 기준 검증/재현, retry는 실패한 정산 복구, correction은 별도 운영 보정 흐름이며 hidden mutation이나 payout rewrite가 아니다.
- MVP에는 별도 제품 내 dispute/central judgment workflow를 두지 않는다. 예외 상황은 이메일 또는 오픈카톡 등 외부 운영 문의 fallback으로 처리한다.
- P0의 목적은 재미있는 습관 앱 전체가 아니라, 사용자가 실제 돈이 걸린 계약 구조를 신뢰할 수 있게 만드는 trust loop 완성이다.

### Deliberately unresolved implementation details

아래 항목은 PRD synthesis에서 boundary만 유지하고, ERD/API/Settlement/Test propagation 단계에서 별도 정렬한다.

- scheduler mechanics와 timezone storage
- DB field naming / API enum / batch job implementation
- deterministic remainder disclosure level
- moderation enum/state 세부값
- moderation log 기본 공개 범위
- 운영 문의 SLA와 담당자
- 약관/법무 wording
- role-based moderation history visibility matrix
- post-final correction/support workflow 세부 운영
- Redis/Redisson/distributed lock/concurrency control 전략
- notification transport(SSE/FCM/push 등)와 delivery topology
- `point_account`의 물리적 balance shape(`available`, `locked`, `pending`, `total` 등)와 cache/reconciliation 전략
- settlement amount unit 재검토 후보와 기존 정수/절사 baseline 변경 여부

### Downstream alignment rule

이 PRD synthesis와 downstream 문서가 충돌하면, downstream 문서는 latest accepted semantics와 PRD synthesis를 기준으로 후속 propagation 단계에서 정렬한다. 이 정렬은 PRD가 L1 intent source를 override한다는 뜻이 아니라, 최신 기획안과 semantic freeze 결과를 PRD가 canonical하게 정리한 범위 안에서 derived docs를 맞추는 작업이다. 다만 이 PRD는 schema/API field name, batch job implementation, enum value 같은 구현 세부를 premature freeze하지 않는다.

## 1. Summary

Dondok은 소규모 크루가 함께 보증금을 걸고 습관 미션을 수행한 뒤, 각자의 약속 이행 정도에 따라 공동 보증금을 투명하게 다시 배분받는 지분 기반 습관 형성 플랫폼이다. 이 문서는 최신 기획안과 accepted semantic freeze 결과를 바탕으로 제품 철학, MVP 범위, 정산/인증/운영 trust boundary, 그리고 downstream 문서가 참조할 canonical synthesis를 정리한다.

Dondok의 MVP는 기능 풍성함이 아니라 “돈이 걸린 습관 계약의 신뢰 루프”를 증명하는 데 집중한다. 신뢰 루프는 방 생성/참여, 포인트(사용자 화면의 도딘) 예치, 인증 제출, 검증 signal 기록, 방장 moderation, 예상 환급금 projection 설명, final batch 정산, 포인트 환급, 정산 결과 설명까지 끊기지 않고 동작하는 흐름이다.

이 PRD의 emotional constitution은 `계약 신뢰 > 상호 책임 기반 성장 > 경쟁 긴장감`이다. Dondok은 gambling-like reward loop, punitive elimination game, adversarial leaderboard app이 아니며, 경쟁 요소는 사용자가 함께 약속을 버티는 구조를 이해하고 지속하게 만드는 범위에서만 보조적으로 허용한다.

## 2. Contacts

| 이름 | 역할                           | 코멘트                                       |
| ---- | ------------------------------ | -------------------------------------------- |
| 미정 | Product Owner / PM             | 범위, 우선순위, 최종 의사결정 담당           |
| 미정 | Technical / Settlement Lead | 정산 엔진, 배치, downstream alignment 책임   |
| 미정 | Auth / Security Lead           | 인증, 권한, 이미지 검증, 보안 정책 책임      |
| 미정 | Crew / Mission Lead            | 크루, 미션, moderation, 알림 흐름 책임       |
| 미정 | Point / Dashboard Lead         | 포인트, projection, 대시보드, 결과 설명 책임 |
| 미정 | Infra / DevOps Lead            | 배포, CI/CD, 모니터링, 장애 대응 책임        |

실명은 아직 문서에 없으므로 추후 보완이 필요하다.

## 3. Background

### Context

많은 습관 앱은 혼자 버티는 구조다. 배지나 포인트를 주더라도 실제 행동을 오래 붙잡아 두기에는 힘이 약하다. 반대로 챌린지형 서비스는 벌금 중심인 경우가 많아, 사용자는 실패를 피하는 데만 집중하게 된다.

Dondok은 이 문제를 다르게 푼다. 사용자는 크루를 만들거나 참여하고, 보증금을 걸고, 정해진 규칙에 따라 사진으로 미션을 인증한다. 미션이 끝나면 각자의 인정된 수행 결과를 기준으로 지분율을 계산하고, 그 비율에 따라 공동 보증금 pool이 재분배된다. 이 구조는 상대적 참여도에 따른 환급금 차이를 숨기지 않지만, 타인의 실패를 유도하는 경쟁이 아니라 함께 성실함을 지속하게 만드는 상호 책임 구조를 지향한다.

### Why Now

지금은 이 구조를 MVP로 만들 수 있는 조건이 갖춰져 있다.

- 팀의 중심 역량이 백엔드에 있어 정산, 배치, 동시성, 보안 같은 trust-loop 핵심 로직에 집중할 수 있다.
- 결제 샌드박스, 파일 저장, 배치, 클라우드 배포 같은 MVP 운영 도구를 바로 붙일 수 있다.
- AI와 소셜 기능은 차별화 요소가 될 수 있지만, MVP에서는 authority flow를 막지 않는 P0 Engagement UX로 제한해야 한다.

### What Recently Became Possible

아래 요소는 예전보다 더 쉽게 구현할 수 있게 됐다.

- 샌드박스 결제와 클라우드 배포 환경 덕분에 실제 서비스와 비슷한 포인트(사용자 화면의 도딘) 예치/환급 흐름을 짧은 기간 안에 시연할 수 있다.
- 이미지 업로드, server_time 기록, EXIF/hash signal, moderation history를 조합해 현실 세계 인증의 신뢰도를 설명할 수 있다.
- AI 미션 추천, 인증 피드/리액션, 알림, 운영 탭, 반응형 UX는 authority 기능이 아니지만 Phase 1 engagement UX로 사용자 이해와 재방문을 돕는다. 실패하더라도 정산/원장/state authority를 차단하지 않는다.

## 4. Objective

Dondok의 목표는 “혼자서는 오래 못 가는 습관”을 “작은 팀과 공정하고 설명 가능한 보증금 재분배 구조”로 오래 가게 만드는 것이다. 사용자는 매일의 성실함이 기록, 현재 기준 예상, 진행 설명으로 보이는 경험을 얻고, 운영자는 재현 가능하고 감사 가능한 정산 구조를 바탕으로 신뢰 가능한 습관 계약 서비스를 만들 수 있다.

이 제품은 아래 원칙에 맞춰 설계한다.

- 상호 책임: 크루는 서로의 습관 지속을 돕는 작은 계약 단위다.
- 투명한 상대성: 참여도 차이에 따라 환급금은 달라질 수 있으나, 그 이유를 사용자가 이해할 수 있어야 한다.
- 협력적 경쟁: 상대적 위치나 기여도는 보여줄 수 있지만, 환급 경쟁보다 “누가 얼마나 꾸준히 함께 버텼는가”를 중심으로 설명해야 한다.
- 설명 가능성: 사용자는 왜 이 인증이 인정/반려되었고 왜 이 환급금이 나왔는지 확인할 수 있어야 한다.
- 재현 가능성: 정산은 동일 입력에 대해 동일 결과를 반환하고, 장애 후에도 다시 검증할 수 있어야 한다.
- 감사 가능성: moderation과 운영자 개입은 기록으로 남아야 하며 임의 변경처럼 보이면 안 된다.

### Key Results

| KR  | 목표                                                                                                                                                | 측정 방식                           |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| KR1 | 첫 릴리스에서 회원가입, 포인트 충전, 크루 참여, 사진 인증, moderation, 정산, 환급까지 trust loop가 끊기지 않고 동작한다.                            | 핵심 E2E 시나리오 통과              |
| KR2 | 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 final settlement batch가 정해진 운영 창 안에 완료되며, 실패 시 재시도 또는 운영 알림이 가능하다. | 배치 로그, 모니터링 알림            |
| KR3 | 정산 결과에서 1원 이상의 설명 불가능한 계산 오차가 발생하지 않는다.                                                                                 | 단위 테스트, 통합 테스트, 표본 검증 |
| KR4 | 정산 핵심 로직 테스트 커버리지가 80% 이상이다.                                                                                                      | 테스트 리포트                       |
| KR5 | EXIF/hash/server_time/moderation 결과가 분리 기록되고, 최종 인증 상태가 왜 그렇게 결정됐는지 설명 가능하다.                                         | 인증/검증/이력 테스트               |
| KR6 | AI·소셜·알림 기능 실패가 방 생성, 자동 시작, 인증, 정산, 환급, 포인트 원장 authority 흐름을 막거나 변경하지 않는다.                                 | 실패 격리 테스트                    |

## 5. Market Segment(s)

이 제품은 나이보다 “어떤 문제를 풀고 싶은가”로 시장을 나눈다.

| 세그먼트                       | 하고 싶은 일                                          | 현재 불편                                         | 제약                                  |
| ------------------------------ | ----------------------------------------------------- | ------------------------------------------------- | ------------------------------------- |
| 혼자 습관을 못 지키는 개인     | 기상, 운동, 공부 같은 루틴을 오래 유지하고 싶다.      | 의지만으로는 금방 흐려진다.                       | 사진 인증이 가능한 습관이어야 한다.   |
| 소규모 목표형 크루             | 친구나 낯선 사람과 함께 목표를 끝까지 해내고 싶다.    | 공정한 규칙과 정산을 직접 관리하기 어렵다.        | 한 크루는 최대 15명으로 제한한다.     |
| 결과를 수치로 보고 싶은 사용자 | 내가 얼마나 지켰고 왜 그 금액을 돌려받는지 알고 싶다. | 기존 앱은 보상이 약하거나 계산 설명이 불투명하다. | MVP에서는 포인트 환급까지만 제공한다. |

### Primary Segment

첫 번째 핵심 타깃은 “규칙적인 생활을 만들고 싶지만 혼자서는 자주 무너지는 2030 사용자”다. 이들은 돈이 걸린 약속, 또래와의 긴장감, 눈에 보이는 예상 보상에 반응할 가능성이 높다. 단, 제품 메시지는 “남의 실패로 이익을 얻는다”가 아니라 “함께 약속을 지키는 정도가 투명하게 반영된다”에 맞춘다.

### Constraints

- 습관 인증은 이미지 업로드 중심으로 시작한다.
- 크루 규모는 최대 15명으로 제한한다.
- 보증금은 포인트로만 다루며, MVP에서는 현금 인출을 제공하지 않는다.
- 미션 규칙은 모두가 이해하기 쉬워야 하며, 계산식은 서버가 일관되게 적용해야 한다.
- P0는 trust loop를 완성하는 범위로 제한한다.

## 6. Value Proposition(s)

| 가치 제안                   | 사용자가 얻는 것                                  | 줄어드는 불편                                             | 경쟁 대비 차별점                                                      |
| --------------------------- | ------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------------------- |
| 참여도에 따라 달라지는 환급 | 내가 약속을 얼마나 지켰는지 환급 결과로 확인한다. | “열심히 해도 똑같다”는 허탈감이 줄어든다.                 | 단순 벌금제가 아니라 상대적 지분 기반 재분배다.                       |
| 예상 환급금 projection      | 현재까지 기준의 예상 흐름과 계산 이유를 확인한다. | 결과를 마지막 날까지 전혀 모르는 답답함이 줄어든다.       | projection을 최종 정산과 구분해 불안 완화와 정산 설명 UX로 안내한다.  |
| 설명 가능한 인증과 정산     | 왜 인정/반려되었고 왜 이 금액인지 이해한다.       | 사진 재사용, 시간 조작, 계산 오차에 대한 불신이 줄어든다. | server_time, EXIF/hash signal, moderation, batch snapshot을 분리한다. |
| 소규모 크루 기반 상호 책임  | 혼자보다 오래 지속할 사회적 긴장감을 얻는다.      | 개인 의지만으로 버티는 부담이 줄어든다.                   | 돈이 걸린 신뢰 기반 습관 계약이라는 차별점이 있다.                    |

### Value Curve

Dondok은 아래 항목에서 기존 습관 앱보다 높은 가치를 주려 한다.

- 금전적 약속의 몰입감
- 정산의 설명 가능성
- 진행 상황의 가시성
- 인증과 moderation의 감사 가능성
- 소규모 크루 기반의 사회적 압박
- 완주/기록/공동 성취 중심의 결과 framing

AI, 인증 피드/리액션, 알림, 운영 탭, 반응형 UX는 제품 경험에 중요하며 P0 Engagement UX로 둘 수 있다. 단, 정산/원장/state authority를 흔드는 blocker가 되어서는 안 된다.

## 7. Solution

### 7.1 UX / Prototypes

핵심 사용자 흐름은 아래와 같다.

1. 사용자는 공개 크루 탐색에서 카테고리/상태 필터로 참여 가능한 미션을 찾는다.
2. Host는 제목, 기간, 인증 안내/기준 설명 텍스트, 보증금, `min_participants`, `max_participants`, `recruitment_deadline`, `start_at`, 인증·정산 cadence를 설정한다.
3. AI 크루 생성 도우미는 입력 보조로 사용할 수 있지만, 수동 입력 fallback은 항상 가능해야 한다.
4. 참여자는 모집 마감 전 포인트/도딘을 충전하고 참여 신청·승인·예치 Lock을 완료한다.
5. `recruitment_deadline`까지 승인 + 예치 Lock 완료된 참여자만 frozen participant baseline에 포함된다.
6. baseline이 최소 인원을 충족하고 host가 시작 전 해체하지 않았다면 시스템은 `start_at` 기준 자동 `ACTIVE`로 전이하며, MVP에서 `activated_at = start_at`이다.
7. `ACTIVE` 이후 host는 미션을 취소하거나 participant baseline을 바꿀 수 없다.
8. 참여자는 사진+텍스트 인증을 제출하고, EXIF/hash는 fraud/risk signal로 사용된다. Host moderation은 인증 입력에만 영향을 준다.
9. 운영 중 사용자는 projection dashboard에서 예상 도딘/예상 환급금 흐름을 본다. 이는 final settlement가 아니다.
10. 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 final settlement batch가 authoritative snapshot을 생성한다.
11. 결과 화면은 누가 왜 성공/실패했고 도딘이 어떻게 분배되었는지 설명한다.

초기 프로토타입 / Phase 1 범위는 아래 화면을 우선 다뤄야 한다.

- 랜딩 / 홈
- 크루 목록 / 검색 / 카테고리·상태 필터
- 크루 대표 이미지가 포함된 크루 카드/상세 표시
- 크루 생성 폼 + AI 크루 생성 도우미 + 수동 입력 fallback
- 크루 상세 / 참여 / 승인 / 예치 Lock
- 자동 시작 상태 안내(`recruitment_deadline`, `start_at`, baseline)
- 사진+텍스트 인증 업로드
- 인증 피드 / 리액션
- 방장 moderation / moderation history / 운영 탭
- 방장 공지 / 댓글 / 공지 리액션 기반 크루 내 소통
- 예상 환급금 projection 대시보드
- 정산 결과 설명 화면
- 마이페이지 / 포인트·도딘 내역
- best-effort 알림 / 외부 운영 문의 안내
- 모바일/데스크톱 반응형 UX

P1 이후 프로토타입 후보:

- 결과 카드 / 공유 / 다운로드 polish (정산 완료 후 final result 전용)
- 정산 완료 이메일/리포트 polish
- AI 습관 리포트
- retention visual / social richness 확장

단, 결과 카드와 공유 욕구 자체는 단순 polish로 삭제할 수 있는 intent가 아니다. MVP에서 저장/다운로드 구현을 P1로 미루더라도, final settlement 이후 완주 기록·공동 성취·다시 보고 싶은 결과 entry point는 PRD에 살아 있어야 한다. Projection 상태를 공유 카드처럼 포장하지 않고, final result 전용 completion ritual로 다룬다.

핵심 UX 안내 문구:

- 모집 상태 영역에는 아래 문구를 노출한다.
  `모집 마감까지 최소 인원이 모이고 예치가 완료되면 미션 시작일에 자동으로 시작됩니다. 모집 마감 전 방장이 크루를 해체하거나 최소 인원이 충족되지 않으면 미션은 시작되지 않고 예치 도딘이 환급됩니다.`
- 예상 환급금 영역에는 아래 문구를 노출한다.
  `예상 환급금 projection은 현재까지의 인증 결과를 기반으로 계산된 현재 기준 예상입니다. 최종 정산 전까지 변동될 수 있으며, 최종 정산은 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 실행되는 final settlement batch 결과로 확정됩니다.`
- 정산 결과 화면에서 전체 성공 횟수가 `0`인 경우 아래 문구를 노출한다.
  `이번 미션에서는 인정된 성공 기록이 없어, 누군가의 실패가 다른 참여자의 추가 환급으로 이어지지 않도록 원금을 기준으로 정산되었습니다.`
- 알림 UX에는 아래 문구를 적용한다.
  `알림은 놓치지 않도록 돕는 best-effort 안내이며, 인증 제출과 정산 기준은 앱 내 기록과 final batch를 따릅니다.`
  Android-first MVP에서는 FCM을 background/off-app 재진입 transport로 사용한다. FCM payload, 알림 목록, 읽음 상태, 발송/수신/실패 상태는 canonical history나 audit source가 아니며, 알림 클릭 시 클라이언트는 `deep_link`로 이동한 뒤 관련 canonical API state를 다시 조회해야 한다.
- 결과 카드/공유 UX에는 아래 문구 방향을 적용한다.
  `이번 크루에서 꾸준히 참여한 기록입니다. 함께 목표를 향해 버틴 과정을 확인해보세요.`

### 7.2 Key Features

#### P0 Authority: trust loop authority에 반드시 필요한 기능

| 기능                                | 설명                                                                                                                                                                                                | Canonical Boundary                                                                                  |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 회원가입/로그인                     | 사용자 계정 생성, 로그인, 기본 프로필 설정                                                                                                                                                          | Identity prerequisite                                                                               |
| 공개 크루 탐색                      | 공개 미션 목록과 카테고리/상태 필터                                                                                                                                                                 | Discovery aid, not settlement authority                                                             |
| 크루 생성 / 모집 / 입장 / 자동 시작 | Host가 미션명, 기간(7~90일), 보증금(1,000~100,000원, 1,000원 단위), min/max 참여자(2~15), recruitment_deadline, start_at, cadence를 설정한다. 참여자는 모집 마감 전 신청·승인·예치 Lock을 완료한다. | Host는 설정·모집·moderation actor이며 activation/settlement/ledger authority가 아니다.              |
| Participant baseline freeze         | recruitment_deadline까지 승인 + 예치 Lock 완료된 참여자만 frozen participant baseline에 포함된다. 최소 인원 충족 + host disband 없음이면 start_at에 자동 ACTIVE가 된다.                             | MVP에서 activated_at = start_at. ACTIVE 이후 baseline 변경 없음.                                    |
| 예치/포인트/도딘                    | 사용자는 포인트를 충전하고, 사용자 화면에서는 도딘(Dodin)으로 보증금·환급 흐름을 이해한다.                                                                                                          | Authoritative accounting은 point ledger/history. 도딘은 별도 coin/cash/withdrawable asset이 아니다. |
| 인증 업로드                         | 사진 + 텍스트 인증 로그 제출                                                                                                                                                                        | Server time is authoritative timing.                                                                |
| EXIF/hash risk signal               | 이미지 metadata/hash를 fraud/risk signal로 수집                                                                                                                                                     | EXIF/hash는 단독 최종 판정 authority가 아니다.                                                      |
| Host moderation                     | Host가 인증 로그를 contextual review하여 accepted/rejected 등 인증 입력 상태를 정한다.                                                                                                              | Moderation affects certification input only, not settlement/ledger authority.                       |
| Moderation history / 운영 탭        | 인증 검토 이력, 처리 상태, 운영상 주의가 필요한 로그를 사용자에게 설명한다.                                                                                                                         | Visibility/audit UX이며 host ledger 권한이 아니다.                                                  |
| 일일 정산 state                     | 인증 주기별 성공/실패 상태와 예상 지분율을 계산한다.                                                                                                                                                | Projection과 final settlement를 분리한다.                                                           |
| Projection dashboard                | 현재 기준 예상 도딘/예상 환급금, 기여도/진행률, 성공률을 보여준다.                                                                                                                                  | 예상값은 authoritative settlement가 아니며 final batch 전 고정값처럼 표현하지 않는다. 수익/순위 dopamine loop로 표현하지 않는다. |
| Final settlement batch              | 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 authoritative settlement snapshot을 생성한다.                                                                                                    | Deterministic / explainable / replayable.                                                           |
| 결과 설명                           | 최종 인증 상태, 도딘 환급, 대표 성공 로그, 공동 성취/완주 기록을 설명한다.                                                                                                                          | 대표 성공 로그는 created_at asc, mission_log.id asc tie-break로 deterministic하게 선정한다. 낙인/승리 framing을 피한다. |

#### P0 Engagement UX: Phase 1 경험 보조 기능

이 범위는 trust-loop-first를 약화하지 않고 사용자 이해·재방문·운영 가시성을 보강한다. 실패해도 authority flow를 차단하지 않아야 한다.

Authority P0와 Engagement UX P0는 서로 다른 실패 경계를 가진다. Authority P0는 정산, 원장, lifecycle, participant baseline, final settlement authority에 직접 영향을 주는 필수 흐름이다. Engagement UX P0는 사용자의 이해, 재방문, 운영 가시성, 생성 편의성을 돕는 Phase 1 경험이며, 실패하더라도 settlement/ledger/lifecycle authority를 차단하거나 변경해서는 안 된다. 따라서 AI 크루 생성 도우미, 인증 피드/리액션, 알림, 운영 탭, projection dashboard, 반응형 UX, 방장 badge/counter는 trust-loop-first를 약화하지 않는 범위에서 P0 Engagement UX로 유지할 수 있다.

Engagement UX는 위험 문구를 제거한다는 이유로 실시간 가시성 자체를 제거하지 않는다. 현재 기준 지분율, 상대적 위치, 예상 환급 흐름, 기여도, 인증 피드, 리액션, 알림 재진입은 사용자가 “돈이 걸린 약속”을 이해하고 계속 참여하게 만드는 핵심 표면이다. 이 표면은 cooperative persistence framing으로 설명하며, 금전적 우위·타인의 미이행·승패 중심 framing으로 격상하지 않는다.

| 기능                         | 포함 이유                                                        | Boundary                                                                                 |
| ---------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| AI 크루 생성 도우미          | 사용자가 미션명/인증 규칙/문구를 쉽게 작성하도록 돕는다.         | Manual fallback 필수. AI 결과는 policy/settlement authority가 아니다.                    |
| 크루 대표 이미지             | 크루 목록/상세/생성 흐름에서 방의 정체성과 탐색성을 돕는다.       | Display metadata다. lifecycle, settlement, moderation authority가 아니다.                |
| 인증 피드                    | 인증 로그를 참여자에게 보여준다.                                 | Feed ordering/display는 settlement input을 직접 바꾸지 않는다.                           |
| 리액션                       | 참여자 간 응원/가벼운 반응을 제공한다.                           | Reaction은 인증 성공/실패나 지분율에 영향 없음.                                          |
| 크루 공지/댓글/공지 리액션   | 채팅 없는 MVP에서 방장 안내와 참여자 반응을 제공한다.             | Communication surface다. mission rule override, certification, settlement, ledger, lifecycle authority가 아니다. |
| 실시간 현황 / 기여 visibility | 현재 기준 지분율, 상대적 위치, 예상 환급 흐름, 기여도를 보여준다. | Projection/current-basis UX다. final settlement, ledger, payout certainty가 아니다.       |
| 운영 탭                      | 검토 대기/거절/누락 등 상태를 설명한다.                          | Contextual visibility, not ledger control.                                               |
| 최종 결과 entry point        | final settlement 이후 완주 기록과 공동 성취를 다시 보게 한다.     | 결과 카드 intent는 유지하되 저장/공유 polish는 P1 가능. Projection 공유 카드는 금지한다. |
| 알림                         | Android-first FCM, 인앱 토스트, 알림 목록/읽음 UX로 모집/인증/검토/정산 주의가 필요한 순간을 재진입시킨다. | Notification, inbox/read, delivery attempt는 UX/transport state이며 certification, moderation, settlement, ledger authority가 아니다. 클릭 시 canonical API refetch가 필수다. |
| 반응형 UX                    | 모바일/데스크톱에서 핵심 trust loop를 수행할 수 있게 한다.       | Presentation requirement, not authority semantics.                                       |
| Host badge/counter           | Host 역할과 검토 책임을 이해시킨다.                              | Host badge는 activation/settlement authority가 아니다.                                   |

#### P1 / Later 후보

| 기능                                    | 설명                                                | 이유                                                                            |
| --------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------- |
| 결과 카드 저장/공유/다운로드            | final settlement 이후 결과를 카드로 저장하거나 공유 | 구현 polish는 P1이나 completion/virality intent는 PRD에 보존한다. Projection 공유 카드로 오해시키지 않는다. |
| 정산 이메일/리포트 polish               | 정산 결과 알림·요약 고도화                          | MVP trust loop 이후 communication polish                                        |
| AI 습관 리포트                          | 개인 습관 요약/코칭                                 | Settlement input이 아니므로 Phase 1 이후 가능                                   |
| Retention visual / social richness 확장 | 배지, streak, 외부 공유 등                          | P0 authority와 직접 무관                                                        |
| 제품 내 dispute workflow                | 앱 내 이의제기/중재 워크플로                        | MVP에서는 운영 문의 fallback만 둠                                               |

#### Notification / FCM MVP Boundary

| 범위 | MVP 판단 | Boundary |
| --- | --- | --- |
| Android FCM push | 포함 | Background/off-app 재진입 transport. Delivery success/failure는 domain success/failure가 아니다. |
| 인앱 토스트 | 포함 | Foreground 즉시 피드백. Durable history나 canonical state가 아니다. |
| 알림 목록/읽음 | 얇은 후보 | UX hint/read affordance 후보일 뿐 backend persistence 기본값이 아니다. Frontend local state/browser permission으로 충분한 상태는 서버 저장으로 승격하지 않는다. Audit history, certification history, settlement history, ledger history가 아니다. |
| notification event/log | 얇은 후보 | 알림 목록과 운영 추적이 꼭 필요할 때만 검토하는 non-authoritative 기록 후보이며 Core persistence default가 아니다. |
| delivery attempt observability | 후보 포함 | FCM 발송/실패/transport retry 관측 후보. Settlement retry/replay/correction과 분리한다. |
| notification preference matrix | Phase 2 | OS permission 또는 최소 설정 이상은 후속 결정으로 둔다. |
| notification template CMS/table | Phase 2 | MVP는 문서/코드 상수로 시작할 수 있으며 문구 안정화 전 table을 freeze하지 않는다. |
| SSE/Web realtime reliability | Phase 2/drift candidate | Android-first FCM MVP를 역으로 결정하지 않는다. 기존 SSE 문구는 재사용 가능한 non-authority semantics만 흡수한다. |
| campaign/broadcast/advanced analytics | Phase 2 | Trust-loop MVP 이후 확장 후보. |

Notification payload/list item을 도입하더라도 `event_type`, `resource_type`, `resource_id`, `deep_link`, `occurred_at`, `display_text`, `requires_refetch=true` 같은 refetch hint 중심으로 제한한다. 최종 환급금, 인증 truth, ledger truth, settlement retry/replay 상태를 notification-owned truth로 싣지 않는다.


#### Phase 2 Defer

| 기능                                  | Defer 이유                                                                |
| ------------------------------------- | ------------------------------------------------------------------------- |
| 중도 참여 / 중도 탈퇴 / 재참여        | participant baseline, payout inclusion, replayability를 흔들 수 있음      |
| Weekly-N cadence                      | MVP cadence를 매일/특정 요일 + A/B/C 타입으로 제한해 정산 timing을 단순화 |
| 현금 출금 / 외부 교환                 | 도딘의 cash-like misunderstanding 및 규제 리스크                          |
| WebSocket/SSE 기반 realtime transport | projection certainty illusion 및 구현 비용                                |
| 제품 내 dispute workflow              | 운영 문의 fallback으로 MVP 운영 가능                                      |

#### Projection / Dashboard Boundary

Dashboard는 운영 중 사용자에게 예상 상태와 그 이유를 설명하지만, authoritative settlement가 아니다. Projection의 1차 역할은 anxiety reduction과 settlement explanation이며, 실시간 수익 변화나 상대 실패 기반 dopamine feedback을 만드는 것이 아니다.

- Projection input: current participant baseline, accepted/rejected/missing certification input, cadence, elapsed cycles.
- Projection output: 예상 성공률, 예상 기여도/지분율, 현재 기준 예상 도딘/예상 환급금.
- Forbidden wording: “실시간 최종 정산”, “확정 환급금”, “환급금 실시간 상승”, “누군가 실패해서 상승”, “더 벌었다”, “수익”, “예상 손익”, “1위 수익자”.
- Required wording: “현재 기준 예상”, “projection”, “최종 정산 전 변동 가능”, “현재 인증 결과 반영”, “크루 진행 상황 업데이트”.
- Final settlement batch 이전 dashboard value는 replay/debug용 설명값이지 ledger snapshot이 아니다.
- 상대적 위치를 표현해야 할 때는 금지 표현인 “1위 수익자”가 아니라 “현재 상위 기여 그룹”, “크루 평균 이상 달성 중”, “이번 크루에서 꾸준히 참여했어요”처럼 cooperative persistence framing을 사용한다.

#### 핵심 비즈니스 규칙

- `min_participants = 2`, `max_participants = 15`.
- 미션 기간은 7~90일이다.
- 보증금은 1,000~100,000원이며 1,000원 단위로 설정한다.
- `recruitment_deadline`은 참여자 승인 + 예치 Lock eligibility cutoff다.
- `recruitment_deadline`까지 승인 + 예치 Lock 완료된 참여자만 frozen participant baseline에 포함된다.
- `PENDING` 참여 신청 시 보증금 reserve가 발생하고 사용 가능 잔액은 감소하지만, `LOCKED` 승인 전에는 activation eligibility나 frozen participant baseline에 포함되지 않는다.
- 승인 전 참여자는 신청을 철회할 수 있으며, 철회 시 reserve/hold된 도딘은 즉시 환급 또는 release되어야 한다.
- Host 거절 또는 미검토 자동 거절 시 해당 신청은 baseline에 포함되지 않으며 reserve/hold된 도딘은 즉시 환급 또는 release되어야 한다.
- 승인 + 예치 Lock 완료 후에는 MVP에서 참여 변경/취소를 허용하지 않으며, 해당 참여자는 frozen baseline 후보가 된다.
- 위 lifecycle semantics는 사용자 기대와 정산 boundary를 고정하기 위한 것이며, DB schema/API status/point account physical shape를 PRD에서 freeze하지 않는다.
- baseline이 최소 인원을 충족하지 못하면 미션은 시작되지 않고 예치 도딘은 환급된다.
- Host는 `start_at` 전까지 미션을 해체할 수 있다. 해체되면 미션은 시작되지 않고 예치 도딘은 환급된다.
- baseline이 최소 인원을 충족하고 host disband가 없으면 system은 `start_at` 기준 자동 `ACTIVE`로 전이한다.
- MVP에서 `activated_at = start_at`이다. 실제 scheduler 실행 시각/저장 방식은 downstream implementation detail로 freeze하지 않는다.
- `ACTIVE` 이후 Host cancel authority는 없다.
- MVP에서 `ACTIVE` 중도 탈퇴, 중도 참여, 재참여는 지원하지 않는다. 모두 Phase 2로 이연한다.
- Payout 대상은 frozen participant baseline 기준이다. `ACTIVE` 이후 업로드하지 않은 참여자는 실패/미인증 상태로 정산에 포함된다.
- MVP cadence는 매일 또는 특정 요일 반복이다.
- MVP는 A/B/C 일일 인증·정산 타입을 지원한다.
  - A(아침형): 인증 마감 09:00, 일일 정산 12:00.
  - B(표준형): 인증 마감 21:00, 일일 정산 00:00.
  - C(올빼미형): 인증 마감 23:59, 일일 정산 익일 12:00.
- Weekly-N cadence는 Phase 2로 이연한다.
- Final settlement batch는 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 실행된다.
- Grace period는 일반 인증 주기에 72시간을 적용하되, 마지막 3일은 grace 없이 즉시 terminal 상태로 처리한다.
- Projection은 final settlement batch 전까지 항상 현재 기준 예상이며, batch 이후 authoritative 값은 settlement snapshot과 point_history에서만 확인한다.
- 단순 성공 횟수만으로 고정 환급금을 배분하지 않는다. 전체 상대 성공률/지분율 기반으로 정산한다.
- 전체 인정 성공 기록이 없으면 누군가의 실패가 다른 참여자의 추가 환급으로 이어지지 않도록 equal principal refund를 적용한다.
- 정산 중 소수점/절사 잔액은 deterministic/replayable calculation rule로 처리한다. 이는 host reward, host authority, 또는 host privilege가 아니다.
- 최대 15명 기준에서 remainder 설명은 1~14원 범위를 초과하지 않도록 유지한다.

#### 인증 검증 / Moderation

- Server time is the source of truth for certification timing.
- EXIF timestamp, GPS, device metadata, image hash는 fraud/risk signal이다.
- EXIF/hash 실패 또는 부재는 곧바로 최종 실패 판정이 아니다.
- Host moderation은 인증 맥락 검토이며 certification input 상태를 바꿀 수 있다.
- Host moderation은 point ledger, settlement engine, final batch, settlement amount, participant baseline, replay/retry/correction을 직접 수정하지 못한다.
- Host 승인/반려는 payout approval 또는 deposit confiscation이 아니다. Reject reason은 사람에 대한 판결이 아니라 정산 입력에서 제외될 수 있는 인증 상태와 사유를 설명해야 한다.
- 운영자 개입은 audit-backed exceptional recovery boundary에 제한된다.
- 모든 moderation 결정과 변경은 actor, action, reason, time을 추적할 수 있는 append-only history로 남아야 한다.

#### Finality / Replay / Retry / Correction Boundary

- Final settlement batch가 `SUCCEEDED` authoritative snapshot을 만들면 final settlement 자체는 immutable하다.
- Pre-freeze/cutoff 이전에는 moderation/certification correction으로 인증 입력 상태를 바로잡을 수 있다.
- Replay는 당시 기준 입력과 규칙으로 결과를 검증/재현하는 audit 동작이며 payout rewrite가 아니다.
- Retry는 `PENDING`/`FAILED`/`RETRY_WAIT` 등 완료되지 않은 정산 처리를 이어서 복구하는 동작이며 succeeded settlement recalculation이 아니다.
- Correction은 final settlement 이후 별도 운영 기준으로 진행되는 보정/지원 흐름이다. MVP PRD는 correction workflow를 새로 설계하지 않으며, 기존 settlement snapshot을 hidden mutation으로 덮어쓰지 않는다.

#### 예외 처리 원칙

| 예외                              | MVP 처리                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| 모집 마감 전 최소 인원 미달       | 미션 미시작 + 예치 도딘 환급                                                          |
| 모집 마감 전 Host 해체            | 미션 미시작 + 예치 도딘 환급                                                          |
| ACTIVE 이후 취소 요청             | MVP에서 미지원. 운영 문의 fallback만 제공                                             |
| ACTIVE 중도 탈퇴/중도 참여/재참여 | MVP에서 미지원, Phase 2                                                               |
| 인증 업로드 지연                  | Grace rule에 따라 pending/terminal 결정                                               |
| EXIF/hash 부재                    | Risk signal로 표시, host moderation 맥락 검토                                         |
| 알림 미수신                       | User must still be able to certify manually                                           |
| 자동 시작/취소 경합               | recruitment_deadline/start_at 기준으로 CANCELLED 또는 ACTIVE를 deterministic하게 결정 |

### 7.3 Technology

PRD는 기술 선택의 상세 정책을 소유하지 않는다. MVP 기술 스택과 운영/아키텍처 결정 사유는 downstream technical docs에서 정렬한다. 단, downstream technical docs가 이 PRD synthesis와 충돌하면, 최신 기획안과 accepted semantic freeze 결과에 맞춰 PRD synthesis를 먼저 확인한 뒤 downstream 문서를 정렬한다. PRD는 기술 세부 구현을 freeze하지 않으며, L1 intent source를 override하지 않는다.

### 7.4 Assumptions

- 사용자는 “보증금을 걸면 더 성실해질 수 있다”는 제안에 실제로 반응한다.
- 대부분의 인증 사진은 EXIF를 유지한 채 업로드되지만, EXIF 유실이나 OS/앱 차이로 인한 오탐 가능성도 존재한다.
- 작은 크루 단위가 몰입감과 운영 난이도 사이에서 가장 현실적이다.
- 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 final settlement batch는 사용자 기대와 정산 안정성 사이의 현실적 기준이다.
- MVP 단계에서는 포인트 환급까지만 있어도 제품 가치 검증이 가능하다.
- AI 크루 생성 도우미, 알림, 인증 피드, 리액션, 크루 공지/댓글/공지 리액션, 운영 탭, 반응형 UX는 P0 Engagement UX일 수 있지만 P0 Authority flow를 막는 조건이 아니다.
- 크루 대표 이미지는 표시 metadata이며, fallback 이미지가 가능하다. 대표 이미지 유무나 값은 정산/생명주기/검수 권한이 아니다.
- 인증 피드, 리액션, 크루 공지/댓글/공지 리액션은 소셜/소통 표현 기능이지만, 정산/환급/포인트/AI/상태 생명주기 기준이 아니다.
- 포인트/도딘 잔액 화면의 현재값은 사용자 표시용 현재값이며, final refund와 authoritative point ledger/history 기준 설명이 우선한다.
- 공개 크루 운영은 초기에는 복잡한 신고/제재 시스템 없이도 가능하다.
- 더 정교한 부정행위 탐지, 현금 인출, 대규모 크루 운영은 후속 버전 과제로 둔다.

## 8. Release

### First Release Scope

첫 릴리스는 아래 질문에 “예”라고 답할 수 있어야 한다.

- 사용자가 가입/로그인 후 닉네임과 프로필 이미지만 포함하는 최소 프로필을 확인하고 수정할 수 있는가?
- 사용자가 크루를 만들고, 모집 마감과 시작일을 이해하며, 자동 시작 lifecycle을 신뢰할 수 있는가?
- 보증금을 안전하게 예치하고, 결과에 따라 도딘/포인트를 환급할 수 있는가?
- 인증 signal과 방장 moderation을 통해 final certification state를 설명할 수 있는가?
- 예상 환급금 projection과 최종 정산 결과를 명확히 구분할 수 있는가?
- 실시간 지분율, 상대적 위치, 예상 환급 흐름, 기여도 visibility가 trust-safe wording으로 유지되는가?
- 정산 결과가 deterministic/replayable/explainable 한가?
- 방장 moderation history와 운영자 예외 개입이 audit 가능하게 남는가?
- AI·소셜·알림 기능이 실패해도 방 생성, 자동 시작, 인증, 정산, 환급 authority 흐름은 계속 사용할 수 있는가?

첫 릴리스의 필수 범위는 아래와 같다.

- 회원가입 / 로그인
- 사용자 프로필
- 공개 크루 생성 및 입장
- 카테고리/상태 필터 기반 공개 크루 탐색
- AI 크루 생성 도우미 + manual fallback
- 모집 마감, 참여 승인, 예치 Lock, 자동 시작 lifecycle
- 최소 2명 / 최대 15명 / 기간 7~90일 / 보증금 1,000~100,000원
- 포인트(사용자 화면의 도딘) 충전, 예치, 환급
- 이미지 업로드와 `server_time` 기록
- EXIF/hash signal 기록
- 방장 moderation과 moderation history / 운영 탭
- 인증 피드 / 리액션
- best-effort 알림
- 예상 환급금 projection 대시보드
- 실시간 지분율 / 상대적 위치 / 기여도 visibility
- 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 deterministic final settlement batch
- authoritative point ledger/history 기반 포인트/도딘 환급
- 정산 결과 설명 화면
- final result 전용 결과 entry point와 completion framing
- 반응형 UX
- 외부 운영 문의 fallback 안내

P1 후보는 아래와 같다.

- 결과 카드 / 공유 / 다운로드 polish (정산 완료 후 final result 전용)
- 정산 완료 이메일/리포트 polish
- AI 습관 리포트
- 리텐션 시각 요소 / social richness 확장

Phase 2 후보는 아래와 같다.

- 중도 참여 / 중도 탈퇴 / 재참여
- Weekly-N cadence
- 현금 인출
- 정식 어드민 / 복잡한 운영자 도구
- complex fraud scoring / advanced anti-cheat
- WebSocket chat / SSE realtime sync / iOS Web Push reliability guarantees
- 대규모 공개 크루와 시즌제 운영
- cooperative contribution visibility / non-adversarial progress framing
- 포인트 만료 구현, 만료 원장 이벤트, 만료 API, 만료 DB 스키마, 만료 transaction type

### Release Phasing

| 단계                                    | 상대 기간 | 목표                                                                                 |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------ |
| 문제 정의 / PRD synthesis stabilization | 약 1주    | L1-aligned product semantics, glossary, trust-loop scope 확정                        |
| 설계 / 초기 셋업                        | 약 1주    | PRD synthesis 기준 downstream ERD/API/Settlement 정렬, 개발 환경, 결제 샌드박스 셋업 |
| 핵심 기능 개발 1                        | 약 1주    | 회원, 크루, 모집/자동 시작, 포인트 예치 기초 완성                                    |
| 핵심 기능 개발 2                        | 약 1주    | 이미지 업로드, server_time, EXIF/hash signal, moderation history 완성                |
| 핵심 기능 개발 3                        | 약 1주    | projection, final batch, authoritative point ledger/history, 예외 처리 완성          |
| 통합 / 설명 UX                          | 약 1주    | 정산 결과 설명, engagement UX, 운영 문의 fallback, 핵심 E2E 완성                     |
| QA / 안정화                             | 약 1주    | trust-loop regression, replay 검증, 배포 안정화, 발표 준비                           |

### Future Release

다음 버전에서 우선 검토할 항목은 아래와 같다.

- 포인트 인출과 실제 운영 결제 키 전환
- 더 강한 부정행위 탐지
- 모바일 앱 또는 PWA 강화
- 대규모 크루와 시즌제 운영
- 추천 미션 개인화 고도화
- 신고, 제재, 운영 정책 도구
- AI 습관 리포트와 결과 공유 고도화
- 포인트 만료 정책과 만료 알림/소멸 처리

### Out of Scope for MVP

- 현금 인출
- 복잡한 금융 정산 정책
- 실시간 양방향 채팅
- perfect realtime sync 보장
- 대규모 소셜 네트워크 기능
- 고급 운영자 리스크 관리 시스템
- AI 개인화 고도화, 장기 메모리, 복잡한 품질 평가 프레임워크
- 모델 비용 최적화 release gate, 모델 비교, 토큰/비용 모니터링 필수화
- 제품 내 dispute/central judgment workflow
- 중도 참여/중도 탈퇴/재참여/계정 통합
- 부분 환급, 중간 해제, 다중 보증금 tranche
- 포인트 만료 구현, 만료 원장 이벤트, 만료 API, 만료 DB 스키마, 만료 transaction type

## 9. Glossary / Terminology Freeze

| 용어                                     | 사용자 관점 의미                               | 시스템 관점 의미                                                                                                     | 혼동 금지                                                          | Source of Truth                                    | 관련 정책                |
| ---------------------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------- | ------------------------ |
| Dondok / 돈독                            | 돈이 걸린 습관 계약 서비스                     | 제품명 / PRD synthesis name                                                                                          | 비-Dondok legacy 명칭과 혼용 금지                                  | PRD synthesis                                      | naming stabilization     |
| 크루 / 방                                | 함께 미션을 수행하는 작은 그룹                 | 미션 규칙과 참여자 집합이 묶이는 단위                                                                                | 공개 SNS 그룹 전체와 다름                                          | PRD synthesis, downstream room model               | trust loop               |
| 방장 / host                              | 크루를 만들고 운영하는 사람                    | 크루 설정, 신청 승인, 시작 전 해체, moderation authority를 가진 actor                                                | Activation/settlement/ledger authority 아님                        | moderation history                                 | host boundary            |
| member                                   | 서비스 사용자 계정                             | 포인트 환급 대상 계정                                                                                                | participant와 혼동 금지                                            | account/auth docs                                  | payout identity          |
| participant                              | 특정 크루에 참여한 단위                        | 정산 계산 단위                                                                                                       | member와 항상 같은 개념 아님                                       | settlement input                                   | calculation identity     |
| recruitment_deadline                     | 모집 마감 시각                                 | 참여자 승인 + 예치 Lock eligibility cutoff                                                                           | activation time 아님                                               | PRD lifecycle                                      | activation freeze        |
| start_at                                 | 미션 시작 예정 시각                            | 자동 ACTIVE 전이의 유효 시작 시각                                                                                    | host command deadline 아님                                         | PRD lifecycle                                      | activation freeze        |
| activated_at                             | 미션이 활성화된 기준 시각                      | MVP에서 start_at과 동일한 effective activation anchor                                                                | scheduler 실행 timestamp/storage 설계 아님                         | PRD lifecycle                                      | activation freeze        |
| frozen participant baseline              | 최종 정산 기준 참여자 집합                     | recruitment_deadline까지 승인 + 예치 Lock 완료된 participant set                                                     | ACTIVE 이후 중도 변경되는 명단 아님                                | PRD lifecycle, settlement input                    | payout inclusion         |
| Dodin / 도딘                             | 사용자가 화면에서 보는 app-money 브랜드명      | 보증금·환급 UX를 표현하는 user-facing app-money branding                                                             | 외부 교환·인출 가능 자산, 별도 coin, 별도 ledger 아님              | PRD glossary, point ledger/history                 | display boundary         |
| point                                    | 보증금 예치와 환급에 쓰는 내부 단위            | 포인트 잔액/원장 기록 대상                                                                                           | MVP 현금 출금 수단 아님; 화면에서는 도딘으로 표시 가능             | authoritative point ledger/history                 | point trust              |
| balance / 잔액                           | 현재 보유하거나 묶인 포인트/도딘 표시          | ledger/history 또는 projection에서 파생되는 조회값                                                                   | 최종 환급금, 출금 가능액, settlement authority와 혼동 금지         | point ledger/history by context                    | balance boundary         |
| 보증금 / deposit                         | 미션 참여를 위해 맡기는 금액                   | lock 대상 금액                                                                                                       | 즉시 환급되는 예치금 아님                                          | point lock/history                                 | trust loop               |
| 예치 / lock                              | 미션 종료 전까지 사용할 수 없게 묶인 상태      | spendable balance에서 분리된 locked amount                                                                           | 결제 취소와 동일 개념 아님                                         | point lock/history                                 | deposit policy           |
| 환급 / refund                            | 정산 후 돌려받는 포인트/도딘                   | final settlement 결과로 발생하는 point movement                                                                      | 예상 환급금 projection과 다름                                      | final batch, point ledger/history                  | settlement               |
| 인증 제출 / upload                       | 미션 수행 사진을 올리는 행동                   | server_time과 file signal을 남기는 event                                                                             | 최종 인정과 동일하지 않음                                          | certification record                               | layered trust            |
| 인정 인증 / final certification state    | 최종 정산에 반영되는 인증 상태                 | signal + rule + moderation이 반영된 resolved state                                                                   | raw EXIF/hash 결과와 동일하지 않음                                 | final certification state                          | layered trust            |
| 예상 환급금 / expected refund projection | 현재까지 기준으로 예상되는 환급 금액           | anxiety reduction과 settlement explanation을 위한 projection 계산 결과                                                | 최종 settlement/refund, 수익, 손익, 실시간 payout certainty와 동일 개념 아님 | dashboard projection                               | projection != settlement |
| 대시보드 / projection polling            | 현재 상황을 확인하는 화면                      | 주기적 또는 요청 시 계산된 projection                                                                                | final settlement 또는 perfect realtime 아님                        | dashboard response                                 | projection boundary      |
| 최종 정산 / settlement                   | 최종 환급 결과                                 | final batch가 만든 settlement snapshot                                                                               | 예상 환급금 아님                                                   | authoritative batch                                | final settlement         |
| authoritative batch                      | 최종 결과를 확정하는 처리                      | 마지막 인증 주기의 일일 정산 완료 시점 + 24시간 후 실행되는 batch                                                    | 실시간 대시보드 계산 아님                                          | batch result                                       | batch authority          |
| server_time                              | 서버가 인증 요청을 받은 시간                   | timing 판단 기준                                                                                                     | EXIF 촬영 시간과 다름                                              | server receive time                                | layered trust            |
| EXIF signal                              | 사진 메타데이터 참고 정보                      | fraud/risk signal                                                                                                    | 최종 판정 아님                                                     | extracted metadata signal                          | layered trust            |
| hash signal                              | 동일 파일 재사용 탐지 신호                     | duplicate/risk signal                                                                                                | 단독 실패 판정 아님                                                | file hash signal                                   | layered trust            |
| moderation                               | 방장이 인증이 규칙에 맞는지 확인하는 과정      | certification input/state에 영향을 주는 contextual review action                                                     | settlement/ledger authority 또는 중앙 운영자 판결 아님             | moderation history                                 | host authority           |
| 방장 moderation authority                | 방장이 인증을 판단할 수 있음                   | settlement input에 영향 가능                                                                                         | ledger 수정 권한 아님                                              | moderation decision                                | host boundary            |
| moderation history                       | 승인/반려/예외 승인 이력                       | append-only audit trail                                                                                              | 삭제 가능한 임시 로그 아님                                         | moderation history                                 | auditability             |
| override                                 | signal 이상이나 예외를 승인하는 결정           | moderation action의 한 종류                                                                                          | 운영자 임의 정산 수정 아님                                         | moderation history                                 | contextual review        |
| grace period / 유예기간                  | 인증 상태 불확실성을 흡수하는 안정화 구간      | 일반 인증 주기 72시간 지연 허용 window                                                                               | 마지막 3일은 grace 없이 terminal 처리, batch 이후 무제한 수정 아님 | PRD lifecycle                                      | layered trust            |
| trust loop                               | 돈이 걸린 계약을 신뢰하게 만드는 핵심 흐름     | 예치→인증→moderation→projection→정산→환급                                                                            | 재미 기능 전체 아님                                                | PRD P0                                             | MVP cutline              |
| replayability                            | 나중에 당시 기준으로 다시 검증 가능한 성질     | same input → same output 검증 가능성                                                                                 | 수동 감으로 조정하거나 지급 결과를 다시 쓰는 운영 아님             | settlement inputs/snapshot                         | deterministic            |
| deterministic settlement                 | 항상 같은 입력에 같은 결과를 내는 정산         | random 없는 계산                                                                                                     | 룰렛/랜덤 보상 아님                                                | settlement formula                                 | no random                |
| append-only                              | 이력이 덮어써지지 않음                         | audit trail 누적                                                                                                     | 수정/삭제 가능한 임시 상태 아님                                    | history/audit log                                  | auditability             |
| auditability                             | 사후 검증 가능성                               | actor/action/reason/time 추적                                                                                        | 운영자 임의 처리 아님                                              | audit log                                          | operator boundary        |
| authoritative point ledger/history       | 포인트 증감 기준 기록                          | balance/refund financial history                                                                                     | PRD에서 특정 table명 고정 금지                                     | point ledger/history                               | point trust              |
| authoritative source                     | 최종 판단 기준                                 | 제품 intent는 최신 기획안과 accepted semantic freeze, PRD는 그 canonical synthesis, 정산 결과는 final batch snapshot | PRD가 L1 intent source를 override하는 constitution이 아님          | L1 intent / PRD synthesis / final batch by context | source hierarchy         |
| 운영자 개입                              | 예외 문의/복구                                 | 장애·오류·악용 대응                                                                                                  | 일반 dispute system 아님                                           | audit-backed ops                                   | limited intervention     |
| settlement snapshot                      | 정산 확정 시점 기록                            | final result freeze                                                                                                  | 이후 projection 재계산 아님                                        | final batch output                                 | batch authority          |
| retry                                    | 완료되지 않은 정산 처리 복구                   | 실패/중단된 settlement processing continuation                                                                       | `SUCCEEDED` 정산 상태를 다시 산정하거나 지급 결과를 덮어쓰는 동작 아님 | settlement status                                  | recovery boundary        |
| replay                                   | 당시 기준 결과 검증/재현                       | settlement-time input/rule/snapshot으로 동일 결과 확인                                                              | payout mutation 또는 결과 재산정 아님                              | settlement snapshot                                | audit boundary           |
| correction                               | 별도 운영 보정/지원 흐름                       | final settlement 이후 별도 append-only 대응                                                                          | hidden mutation 또는 settlement overwrite 아님                     | audit-backed ops                                   | support boundary         |
| layered trust model                      | 여러 신뢰 신호를 조합                          | time/signal/moderation/batch 분리                                                                                    | 단일 hard truth 아님                                               | PRD synthesis                                      | certification policy     |
| downstream propagation                   | PRD synthesis를 ERD/API/Settlement/Test에 반영 | L1-aligned PRD wording을 derived docs로 정렬하는 후속 작업                                                           | downstream이 L1 intent나 PRD synthesis를 역으로 결정하는 것 아님   | PRD synthesis after L1 alignment                   | source hierarchy         |

### 제거하거나 주의할 혼용 표현

- realtime projection을 최종 정산금처럼 표현하지 않는다. 현재 기준 값은 “예상 환급금 projection”으로 표기한다.
- projection을 수익/손익/실시간 상승/타인 실패 기반 상승처럼 표현하지 않는다. “현재 기준 예상”, “최종 정산 전 변동 가능”, “현재 인증 결과 반영”으로 설명한다.
- 상대 비교를 “1위 수익자”, “실패자”, “누가 돈을 가장 많이 벌었는가”처럼 표현하지 않는다. 필요하면 “기여 구간”, “크루 평균 이상”, “함께 달성한 인증 수”처럼 협력적 진행 표현을 사용한다.
- EXIF/hash signal 이상을 최종 불인정으로 단정하지 않는다. “review/유예기간 대상이 될 수 있다”로 설명한다.
- host가 돈을 나누거나 정산한다는 표현을 쓰지 않는다. “방장은 인증 상태를 moderation하고, final batch가 정산한다”로 설명한다.
- 방장 승인/반려가 원장 수정 권한처럼 읽히는 표현을 금지한다.
- 반려/거절을 “몰수”, “처벌”, “수익 박탈”, “문제 사용자”, “부정행위 확정”처럼 표현하지 않는다. 정산 입력에 반영될 인증 상태와 사유로 설명한다.
- retry/replay/correction을 “정산을 다시 계산”, “결과를 다시 산정”, “몰래 수정”처럼 표현하지 않는다. Notification retry는 FCM delivery attempt 복구일 뿐 settlement retry/replay/correction이 아니다.
- AI 크루 생성 도우미는 P0 Engagement UX로 둘 수 있지만, manual fallback 없는 mandatory flow나 policy/settlement authority처럼 표현하지 않는다.
- 도딘은 보증금·환급 UX를 표현하는 user-facing app-money branding으로 설명하되, 외부 교환·인출 가능 자산이나 별도 ledger처럼 표현하지 않는다.

### 발표자료 / FAQ wording 주의

- “남의 실패로 돈 번다” 금지.
- “예상 환급금은 확정 금액”처럼 표현 금지.
- “예상 손익”, “실시간 수익 증가”, “더 벌었다”, “1위 수익자” 금지.
- “인증 실패자”, “문제 사용자”, “몰수”, “처벌”, “수익 박탈” 금지.
- “EXIF가 없으면 무조건 끝” 금지.
- “방장이 돈을 나눠준다” 금지.
- “방장 승인으로 환급 확정” 금지.
- “retry/replay로 정산을 다시 계산한다” 금지.
- 현재 기준 projection을 최종 환급금 확인처럼 표현 금지.
- 알림 목록/읽음/미수신을 canonical history, audit log, unresolved settlement/certification task처럼 표현 금지.

### 비개발 직군 설명용 단순화

- Projection: “현재까지 기준으로 보여주는 예상값.”
- Settlement: “최종 검토 후 확정된 결과.”
- Moderation: “방장이 인증이 규칙에 맞는지 확인하는 과정.”
- Retry: “완료되지 않은 정산 처리를 이어서 복구하는 과정.”
- Replay: “당시 기준으로 결과를 다시 검증하는 과정.”
- Correction: “최종 정산 이후 별도 운영 기준으로 진행되는 보정/지원 처리.”
- Layered trust: “사진 정보, 제출 시간, 방장 확인을 함께 보는 구조.”
- Replayability: “당시 기준 입력으로 다시 검증해도 같은 결과가 확인되는 구조.”

### 외부 노출 금지 또는 주의할 backend-only jargon

- `Settlement.status`
- `PointHistory`
- `idempotency_key`
- `resolved certification state`
- `append-only`
- `batch snapshot`
- `calculation_reason`
- `projection polling`

외부에는 필요 시 “최종 정산 상태”, “포인트 기록”, “중복 지급 방지”, “최종 인증 상태”, “변경 이력”, “정산 확정 기록”처럼 풀어서 설명한다.

## 10. Downstream Propagation Preparation

이 섹션은 후속 ERD/API/Settlement/Test 정렬을 위한 영향도 지도다. PRD는 제품 의미와 정책 boundary를 정리하되, API specification, ERD, requirements specification, wireframe/QA, 외부 WBS/GitHub Issues, implementation gate, settlement recovery runbook 계열 downstream 문서를 직접 수정하거나 구현 세부를 freeze하지 않는다.

### Policy impact map

| Policy               | ERD 영향                         | API 영향                      | Settlement 영향         | Test 영향                                     |
| -------------------- | -------------------------------- | ----------------------------- | ----------------------- | --------------------------------------------- |
| Layered trust        | signal/state 분리 필요           | 인증 상태와 signal 응답 구분  | resolved state 입력     | EXIF/hash override case                       |
| Moderation authority | moderation history 필요          | approve/reject/override 계약  | moderation 결과 반영    | audit/history tests                           |
| Projection boundary  | projection 저장/계산 정책 검토   | dashboard disclaimer/status   | final과 독립            | projection != final tests                     |
| Deterministic remainder | settlement snapshot 설명 필요 | settlement detail reason | replayable remainder calculation, not host reward/authority | deterministic remainder tests |
| P0 trust loop        | authority scope 확인             | endpoint priority             | batch/replay priority   | E2E trust-loop tests                          |
| Activation lifecycle | participant baseline 필요        | 모집/자동 시작 상태 응답 검토 | baseline freeze input   | auto-start/cancel race tests                  |
| Cadence A/B/C        | cadence 설정/일일 정산 기준 검토 | cadence display/validation    | daily settlement anchor | A/B/C timing tests                            |
| P0 Engagement UX     | authority table과 분리           | failure isolation contract    | 정산 side effect 없음   | AI/feed/reaction/notification isolation tests |

### High-risk contamination sources

1. EXIF hard-fail assumption
2. realtime projection certainty / profit-like projection wording
3. AI/social/communication/notification mandatory-authority wording
4. downstream docs as intent/policy authority wording
5. remainder recipient conflict
6. unrestricted admin/operator mutation wording
7. all-fail punitive or zero-refund wording
8. host approval/rejection as payout approval/confiscation wording
9. retry/replay/correction as hidden mutation wording
10. adversarial ranking / winner payout framing

### Propagation dependency order

1. L1 intent authority 확인
2. PRD synthesis wording stabilization
3. Glossary authority wording alignment
4. Usecase semantic bridge alignment
5. Settlement semantics wording confirmation
6. ERD impact mapping
7. API contract patch
8. Requirements / WBS / GitHub Issues / wireframe / QA scenario alignment

### Freeze-before-propagation areas

- scheduler mechanics와 timezone storage
- DB field naming / API enum / batch job implementation
- moderation enum/state 세부값
- moderation log 공개 범위
- 운영 문의 SLA
- 약관/법무 wording
- role-based moderation history visibility matrix
- post-final correction/support workflow 세부 운영
