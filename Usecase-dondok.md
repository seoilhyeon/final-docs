# Usecase: Dondok Behavioral Semantic Bridge

> 이 문서는 Dondok의 행동 의미(behavioral semantics)를 안정화하기 위한 브리지 문서다. PRD를 대체하지 않고, API/ERD/Settlement/QA/화면/운영 문서가 같은 신뢰 루프와 권한 경계를 공유하도록 돕는다.

## 1. Purpose and Positioning

### 1.1 왜 이 문서가 존재하는가

Dondok은 돈이 걸린 그룹 습관 계약 플랫폼이다. 따라서 단순한 기능 목록보다 다음 의미가 먼저 안정화되어야 한다.

- 사용자가 언제 참여자가 되는가.
- 예치금은 언제 잠기고 언제 최종 정산 대상이 되는가.
- 인증 이미지는 언제 단순 업로드이고 언제 인증 기록인가.
- 방장(host)은 어디까지 판단할 수 있고 어디부터 판단할 수 없는가.
- projection은 언제 변할 수 있고 왜 최종 정산이 아닌가.
- settlement, retry, replay, correction은 서로 어떻게 다른가.
- append-only history가 기술적으로만 존재하는 것이 아니라 사용자가 어떻게 신뢰할 수 있는가.

이 문서는 PRD의 제품 의미를 normalized behavioral scenarios, authority boundaries, propagation risks로 풀어 downstream drift를 줄이는 행동 의미 브리지(behavioral semantic bridge)다.

### 1.2 문서 계층에서의 위치

| Layer | Role | 이 문서와의 관계 |
|---|---|---|
| PRD | canonical synthesis layer | 제품 의미와 정책의 상위 합성 레이어다. 이 문서는 PRD 의미를 행동 흐름으로 풀어 downstream drift를 줄인다. |
| Usecase-dondok | behavioral semantic bridge | usecase, pressure finding, unresolved semantic, propagation risk를 묶는 브리지다. |
| ERD | derived data model | 이 문서의 append-only, replay, source-of-truth 요구를 데이터 구조로 반영한다. |
| API spec | derived public contract | 이 문서의 권한/상태/용어 경계를 외부 계약으로 반영한다. |
| Settlement design | derived settlement authority design | 이 문서의 deterministic settlement, retry/replay/correction 분리를 계산/복구 정책으로 반영한다. |
| QA/Test | derived behavioral verification | 이 문서의 lifecycle, freeze, projection/final, retry/replay edge를 테스트 매트릭스로 반영한다. |

### 1.3 Authority boundary

이 문서는 새로운 권한을 부여하지 않는다. 특히 다음을 금지한다.

- 방장을 lifecycle, settlement, ledger authority로 승격하지 않는다.
- projection을 최종 정산으로 격상하지 않는다.
- retry를 correction으로 해석하지 않는다.
- replay를 recalculation으로 해석하지 않는다.
- 업로드 객체 존재를 인증 권한으로 해석하지 않는다.
- notification/SSE/AI/social/feed를 canonical state authority로 해석하지 않는다.
- unresolved semantic을 조용히 resolved로 바꾸지 않는다.

### 1.4 Downstream propagation role

Downstream 문서는 이 문서를 다음 방식으로 소비해야 한다.

1. PRD의 canonical synthesis와 충돌하지 않는지 확인한다.
2. 이 문서의 hard blocker를 먼저 해결하거나 명시적으로 unsafe/deferred로 표시한다.
3. propagation warning은 삭제하지 말고 PRD/API/Wireframe/QA/Support에 라벨과 함께 전달한다.
4. Brownfield Conflict는 기존 문서/구현 흔적을 지우지 말고 drift candidate로 표시한다.
5. 실제 구현 세부값은 이 문서가 아니라 API/ERD/Settlement 문서에서 freeze한다.

### 1.5 Canonical constraints preserved here

이 문서 전체에서 다음 제약은 authoritative하다.

- Emotional priority is `계약 신뢰 > 상호 책임 기반 성장 > 경쟁 긴장감`.
- Dondok is not a punitive elimination game, gambling-like reward loop, or adversarial leaderboard app.
- Host has certification input moderation authority only.
- Host is NOT lifecycle, settlement amount, ledger, final settlement, participant baseline, replay/retry/correction authority.
- Projection is a current-basis estimate for anxiety reduction, state visibility, and settlement explanation.
- Projection != final settlement.
- All-fail settlement = equal principal refund; prior zero-refund wording is rejected/brownfield drift only.
- Retry != correction.
- Replay != recalculation.
- Correction after final settlement is a separate support/operations adjustment semantic, not hidden mutation.
- Moderation history is append-only.
- Final settlement must be deterministic, explainable, replayable, and auditable.
- Notifications are non-authoritative hints.
- Server time is authoritative timing source.
- Settlement snapshot + `point_history` become authoritative after settlement success.
- Brownfield conflicts remain visible until intentionally resolved.

## 2. Behavioral Semantic Principles

### 2.1 Trust loop first

Dondok의 핵심은 “기능 풍성함”이 아니라 돈이 걸린 습관 계약의 신뢰 루프다.

제품 감정의 우선순위는 계약 신뢰, 상호 책임 기반 성장, 경쟁 긴장감 순서다. 상대적 결과 차이는 존재할 수 있지만, 행동 의미의 중심은 금전적 우열이 아니라 “누가 얼마나 꾸준히 함께 버텼는가”여야 한다.

```text
crew creation
  -> participant approval / deposit lock
  -> baseline / activation
  -> certification upload and server-time record
  -> EXIF/hash risk signal
  -> host moderation
  -> projection explanation
  -> deterministic settlement
  -> point_history ledger reflection
  -> audit / replay / support explanation
```

이 루프 중 어느 한 지점이라도 권한 경계가 흐려지면 사용자는 “누가 내 돈을 결정했는가”를 이해하지 못한다.

### 2.2 Authority separation

- Host: crew 설정, 모집 맥락, 인증 입력(certification input) moderation actor다.
- System lifecycle: activation/cancellation/freeze 전이를 결정한다.
- Settlement engine: final settlement snapshot을 결정한다.
- Ledger: `point_history`가 금액 변화의 source of truth다.
- Dashboard/feed/notification/support: 설명 및 접근 surface이며 권한 원천이 아니다.

Host는 accepted/rejected certification input과 contextual review만 판단한다. Host는 settlement amount, ledger, final settlement, participant baseline, replay, retry, correction을 결정하지 않는다.

### 2.3 Projection vs settlement

Projection은 사용자의 불안을 낮추고 최종 정산을 설명 가능하게 만드는 current-basis UX estimate다. Projection은 감정적으로 중요하지만 최종 정산 권한은 없고, profit/dopamine loop가 아니다.

- Current-basis projection: 응답 시점에 확인 가능한 현재 입력 기준 예상. 미션 종료 이후에도 final settlement 전에는 이 표현만 사용한다.
- Final settlement: settlement snapshot + settlement item + point history.

### 2.4 Append-only philosophy

Dondok의 신뢰는 “최신값만 맞다”가 아니라 “왜 그렇게 바뀌었는지 추적 가능하다”에서 나온다.

- Mission log는 원본 인증 사건을 보존한다.
- Moderation decision/correction은 기존 사건을 지우지 않고 이력을 추가해야 한다.
- Settlement snapshot과 point history는 성공 후 운영/분쟁/조회 기준이다.
- Correction이 필요하다면 기존 결과를 덮어쓰지 않는 별도 보정/보상 의미가 필요하지만, 이 문서는 그 workflow를 설계하지 않는다.

### 2.5 Replayability

Replay는 final result를 바꾸기 위한 재계산이 아니라, settlement-time input/rule/snapshot으로 결과가 재현 가능한지 확인하는 audit 동작이다.

- Retry != correction.
- Replay != recalculation.
- Final settlement 이후 MissionLog 재계산은 debugging/audit verification일 뿐 지급 결과를 대체하지 않는다.

### 2.6 Moderation boundaries

Host moderation은 인증 입력 상태에 영향을 줄 수 있다. 그러나 host는 다음을 할 수 없다.

- settlement engine 직접 조작
- final refund amount 직접 조작
- point ledger 직접 조작
- post-freeze authoritative settlement input 소급 변경
- correction workflow 설계 없이 succeeded settlement 변경

### 2.7 Lifecycle ownership

Lifecycle 전이는 system rules가 소유한다. Host는 lifecycle authority가 아니다. Brownfield 문서나 API가 host manual start를 암시하면 Brownfield Conflict / Drift Candidate로 남긴다.

### 2.8 Notification non-authority

Notification은 canonical state가 아니라 hint/deep-link다. 사용자는 알림을 통해 진입할 수 있지만 최종 상태는 canonical API response와 authoritative records가 결정한다.

### 2.9 Emotional trust semantics

Dondok에서 UX copy는 단순 polish가 아니다. 다음 오해를 만들면 semantic risk다.

- projection이 보장 환급처럼 보임
- projection이 `예상 손익`, `실시간 수익 증가`, `더 벌었다`처럼 profit loop로 보임
- host rejection이 몰수처럼 보임
- retry가 지급 재작성처럼 보임
- correction이 몰래 과거 수정처럼 보임
- post-end projection이 final settlement처럼 보임
- all-fail이 전원 0원/house edge/처벌처럼 보임
- ranking/profit 화면이 도박성 경쟁처럼 보임
- 실패자/1위 수익자/공격적 leaderboard가 상대 실패를 기대하게 만듦

원칙은 legalistic warning spam이 아니라 trust-through-visibility다.

Hardening은 engagement mechanics 제거가 아니다. 실시간 지분율, 상대적 위치/순위, 예상 환급 흐름, 기여도, 결과 카드, 피드/리액션, 알림 재진입은 사용자가 현재 상태를 이해하고 다시 돌아오게 만드는 UX intent로 유지한다. 단, 표현은 “현재 기준”, “기여/진행”, “함께 버틴 기록”, “최종 정산 전 변동 가능”을 중심으로 하고, 금전적 우위·타인의 미이행·승자 독식처럼 보이는 framing을 금지한다.

### 2.10 Engagement visibility boundary

아래 mechanics는 Usecase에서 살아 있어야 하지만 authority로 격상하면 안 된다.

- 실시간 지분율/기여도: 현재 입력 기준 projection visibility이며 final settlement input mutation 권한이 아니다.
- 상대적 순위/위치: cooperative persistence를 돕는 상대 위치 표시이며 adversarial leaderboard가 아니다.
- 예상 환급금: 불안 완화와 정산 설명을 위한 current-basis estimate이며 payout promise가 아니다.
- 인증 피드/리액션: social richness와 응원을 위한 engagement surface이며 certification result authority가 아니다. 일반 feed timeline은 `PENDING_REVIEW`/`SUCCESS`/`FAILED` 로그를 보여주는 append-only visible activity stream이고, `NOT_SUBMITTED`는 row 없는 synthetic day/member slot projection이다. 리액션은 OS emoji/free token 기반 social metadata이고, FE가 선택한 token 문자열 그대로 같은 reaction_type으로 취급한다. 동일 token 단위로만 toggle/delete/idempotency를 판단하며 여러 token은 공존할 수 있다.
- Day/member slot과 dashboard/projection은 latest/effective 상태 하나를 사용하는 current-focused summary다. Feed item count와 reaction count는 recognized success count가 아니다.
- 크루 공지/댓글/공지 리액션: 채팅 없는 MVP에서 방장 안내와 참여자 반응을 지원하는 communication surface이며, mission rule override나 settlement/ledger/certification/lifecycle authority가 아니다.
- 결과 카드/공유 욕구: final settlement 이후 completion ritual과 virality intent이며 projection 공유 카드나 금전적 우위 자랑 카드가 아니다.
- 알림 richness: 사용자를 canonical 화면으로 다시 데려오는 hint/deep-link이며 state authority가 아니다.

## 3. Semantic Domain Map

| Domain | Authority Owner | Key Lifecycle Boundaries | Key Semantic Risks | Related Usecases |
|---|---|---|---|---|
| Lifecycle | System lifecycle rules | create/recruiting/baseline/activation/closed/cancelled | host lifecycle authority leakage; activation anchor drift | UC-A01, UC-A04, UC-A05 |
| Recruitment / Activation | System + deposit/participant constraints | recruitment deadline, approval, deposit lock, baseline, automatic activation | `StartRoom` and `/start` brownfield conflict; `PENDING` reserve vs `LOCKED` baseline confusion | UC-A02, UC-A03, UC-A04 |
| Certification / Upload | MissionLog + server validation | upload object, mission-log creation, server_time, validation, risk signal | upload success treated as certification; EXIF/hash treated as final authority | UC-A06, UC-A07, UC-A08, UC-A09 |
| Moderation | Host contextual review only | moderation event, correction-before-freeze, post-freeze no host mutation | host rejection perceived as confiscation; append-only hidden by latest-only UI; host treated as payout approver | UC-A10, UC-A11, UC-A12 |
| Projection | Query-time UX calculation | current-basis estimate, post-end estimate, final settlement handoff | estimate treated as guaranteed payout/profit loop; post-end estimate treated as final | UC-A13, UC-A14, UC-A15 |
| Settlement | Settlement engine + settlement snapshot | input freeze, deterministic calculation, item snapshot, ledger link, succeeded | all-fail zero-refund brownfield drift; host-framed remainder authority; duplicate payout | UC-A15, UC-A16, UC-A17 |
| Replay / Audit | Audit/reconciliation process | replay inputs, version/snapshot, comparison result | replay confused with recalculation; version drift breaks reproducibility | UC-A18 |
| Retry / Recovery | Admin recovery + idempotent system constraints | failed/retry-wait, partial point_history, missing item link | retry treated as correction or rerun payout | UC-A17, UC-A18 |
| Notification / State Drift | Non-authoritative delivery | send, receive, stale payload, reconnect, canonical refresh | notification implies final state; failure is mistaken for settlement retry | UC-A19 |
| Support / Explanation | Support follows source-of-truth hierarchy | pre-settlement explanation vs post-settlement explanation | support cites projection as final; correction/replay promises too much | UC-A20 |
| Crew Communication | Non-authoritative social/communication surface | host notice, comment, notice reaction, visibility status | notice content is mistaken for rule override or settlement/certification authority | UC-A21 |
| Emotional / Trust UX | Product semantics / UX | estimate changes, rejection, contribution visibility, tie, shame, warning density | deceptive certainty, punitive/legalistic product feel, or adversarial ranking | PF-001, PF-004, PF-015–PF-017 |
| Brownfield Conflicts | Canonical semantic register until resolved | manual start, old enums, deferred endpoints, mismatched examples | legacy wording silently becomes active semantics | UC-A04, UC-A12, UC-A16 |

## 4. Core Usecase Inventory

The following inventory consolidates the raw usecase corpus into normalized behavioral scenarios. It intentionally favors semantic boundaries over implementation detail.

### UC-A01 — Crew Creation and Rule Commitment

- **Actors**: Host, system
- **Classification**: actor-performed usecase (Host setup action; no lifecycle/settlement authority granted)
- **Preconditions**: Host authenticated; mission/deposit/recruitment inputs valid.
- **Main Flow**: Host creates a crew with mission rules, deposit amount, schedule, recruitment window, participant limits, and visibility. The same transaction auto-creates the host's own `crew_participant` row as `LOCKED` and reserves/locks the host's deposit identically to a regular participant; the host does not go through the join-request/approval flow.
- **Failure Flow**: Invalid dates, invalid deposit, contradictory participant limits, or insufficient host balance for the host's own deposit reserve prevent creation; partial commits are not allowed.
- **Authority Boundary**: Host configures initial context and is simultaneously a participant under the same `crew_participant` model, but does not gain lifecycle, settlement, ledger, or remainder authority from being host. Host auto-participation is a UX shortcut, not a privilege.
- **Projection Impact**: No performance projection yet; only setup/recruitment state can be displayed.
- **Settlement Impact**: The host's auto-created `LOCKED` participant is a baseline candidate identical to other `LOCKED` participants; the host is also a final settlement target.
- **UX Risk**: Host may assume “creator” means final decision authority.
- **Related Domain Objects**: `mission_room`, `mission_rule`, `mission_schedule_day`.

### UC-A02 — Join, Approval, and Deposit Lock

- **Actors**: Participant, host, system
- **Classification**: actor-performed usecase (Participant join action; host approval is contextual review context only)
- **Preconditions**: Room recruiting; participant eligible; sufficient point balance.
- **Main Flow**: Participant submits a public-crew join request. The request creates a `PENDING` reserve that decreases available balance and reserves capacity, but only host approval to `LOCKED` can make the participant part of activation/frozen baseline.
- **Lifecycle Semantics**:
  - Before approval, participant cancellation is allowed and the `PENDING` reserve must be refunded/released through ledger-backed balance restoration.
  - Host rejection before `LOCKED` excludes the row from baseline and the `PENDING` reserve must be refunded/released through ledger-backed balance restoration.
  - Unreviewed `PENDING` rows at the relevant recruitment cutoff are auto-rejected/expired for baseline purposes and their reserve must be refunded/released through ledger-backed balance restoration.
  - After approval to `LOCKED`, MVP does not allow participant-side change/cancel; the participant becomes a baseline candidate until activation freeze.
  - A participant who self-cancelled before approval (status `CANCELLED`) may reapply to the same crew while the room is still `RECRUITING`, the server time is before `recruitment_deadline`, capacity is available, and the participant has sufficient balance to re-reserve the deposit. Reapply reuses the existing `crew_participant` row by transitioning `CANCELLED -> PENDING` in place and appending a fresh `CREW_DEPOSIT_RESERVE` ledger entry; no new participant row is created and `unique(crew_id, member_id)` is preserved. The previous `CREW_RESERVE_RELEASE` ledger row is kept as append-only audit and prior cancellation history is retained for moderation/audit. After activation, this reopen path is closed; ACTIVE-phase withdrawal/rejoin remains brownfield/deferred and does not reopen the frozen `LOCKED` baseline.
  - Host-side rejection (`REJECTED`) and pre-start auto-expiry (`EXPIRED`) are terminal exits and are not eligible for reapply; subsequent join attempts on the same crew are blocked.
  - Host auto-created `LOCKED` rows are outside the reapply concept entirely; the host does not reapply to a crew they own.
  - Canonical API/ERD statuses are `PENDING` and `LOCKED`; this usecase intentionally leaves physical cache/column implementation to ERD/API.
- **Failure Flow**: Insufficient balance, duplicate join, deadline passed, participant cancellation before approval, host rejection, auto-rejection/expiration at cutoff, reserve/refund failure, `LOCKED` transition failure.
- **Authority Boundary**: Deposit lock and participant inclusion are system/ledger constrained; host approval here is contextual review context (not payout approval, not deposit-waive authority); host cannot waive settlement rules.
- **Projection Impact**: `reserved_balance`, `locked_balance`, and participant-count visibility may update as UX projection, but these displays do not make `PENDING` part of final baseline before approval to `LOCKED`.
- **Settlement Impact**: Only `LOCKED` participants can become payout baseline candidates.
- **UX Risk**: Users may confuse `PENDING` reserve with fully `LOCKED` baseline status, or interpret rejection/auto-rejection as confiscation.
- **Related Domain Objects**: `crew_participant`, `point_account`, `point_history`.

### UC-A03 — Concurrent Join and Capacity/Balance Race

- **Actors**: Multiple participants, system
- **Classification**: extension / edge case (UC-A02 race variant; DB/account consistency invariant)
- **Preconditions**: Room still accepts participants; concurrent join requests occur.
- **Main Flow**: System commits each join/deposit lock atomically according to balance, capacity, and state.
- **Failure Flow**: Duplicate participant row, double balance deduction, optimistic UI showing locked before lock commits.
- **Authority Boundary**: DB/account consistency wins over client optimism.
- **Projection Impact**: Locked balance and participant count projections may briefly lag.
- **Settlement Impact**: Only committed lock/baseline records count.
- **UX Risk**: User sees a temporary “locked” state and later loses it.
- **Related Domain Objects**: `crew_participant`, `point_history`, participant status.

### UC-A04 — Automatic Activation and Baseline Freeze

- **Actors**: System, host, participants
- **Classification**: system-driven lifecycle event (canonical activation anchor; not host-initiated)
- **Preconditions**: Recruitment conditions, approval, and deposit locks satisfy canonical start rules.
- **Main Flow**: System transitions room to active at the canonical activation anchor and freezes the participant baseline.
- **Failure Flow**: Conditions missing; activation/cancellation race; brownfield host-start wording survives.
- **Authority Boundary**: Host is not lifecycle authority. Manual `StartRoom` / `/start` wording is Brownfield Conflict unless intentionally resolved.
- **Projection Impact**: Dashboard moves from not-started to live estimate after activation.
- **Settlement Impact**: Activation/baseline anchors drive certification eligibility and settlement inputs.
- **UX Risk**: Host or participants think host controls when money-affecting lifecycle starts.
- **Related Domain Objects**: `mission_room`, `crew_participant`.

### UC-A05 — Recruitment Expiry or Start Failure Cancellation

- **Actors**: System, participants, host
- **Classification**: system-driven lifecycle event (cancellation/refund path; system owns expiry rules)
- **Preconditions**: Room remains recruiting when canonical start/cutoff conditions fail.
- **Main Flow**: System cancels or prevents activation and routes deposits to the appropriate cancellation/refund settlement path.
- **Failure Flow**: Room remains indefinitely recruiting; user expects instant refund without settlement/recovery step.
- **Authority Boundary**: System owns expiry and cancellation rules.
- **Projection Impact**: Performance projection is not provided.
- **Settlement Impact**: Cancel-before-start refund settlement may be needed.
- **UX Risk**: “Cancelled” may imply immediate ledger refund even if settlement/refund processing is pending.
- **Related Domain Objects**: `mission_room`, `settlement`, `point_history`.

### UC-A06 — Certification Upload vs MissionLog Authority

- **Actors**: Participant, system
- **Classification**: actor-performed usecase (Participant certification submission; upload ≠ certification success)
- **Preconditions**: Participant is eligible to submit; upload route available.
- **Main Flow**: User uploads image, then creates mission-log/certification record with `image_s3_key` and required 5~100 char `caption` through server validation.
- **Failure Flow**: Upload succeeds but mission-log creation fails; image object orphaned; image-only or caption-only submission is rejected; validation delayed near cutoff.
- **Authority Boundary**: Upload object existence and caption text alone do not establish certification. The server-validated `MissionLog` boundary requires both image object key and caption.
- **Projection Impact**: No projection impact until successful/eligible mission-log candidate exists. Caption may be displayed for feed/replay context but does not decide success/failure by itself.
- **Settlement Impact**: No recognition without authoritative log/input; caption is not settlement input.
- **UX Risk**: User thinks “image uploaded” equals “certification submitted.”
- **Related Domain Objects**: upload object, `mission_log`.

### UC-A07 — Server-Time Certification Eligibility

- **Actors**: Participant, system
- **Classification**: shared input rule / invariant within UC-A06 (server_time eligibility — not a standalone actor-performed action; diagram에서는 UC-A06 라벨에 흡수될 수 있음)
- **Preconditions**: Mission active or near boundary.
- **Main Flow**: System records `server_time` and uses canonical time rules for eligibility.
- **Failure Flow**: Client time, EXIF time, or async processing completion is incorrectly used.
- **Authority Boundary**: Server time is authoritative timing source; EXIF time is signal only.
- **Projection Impact**: Projection bucket/cutoff uses canonical time interpretation.
- **Settlement Impact**: Recognition depends on server-time eligibility.
- **UX Risk**: User disputes near-midnight/near-cutoff result based on camera or local device time.
- **Related Domain Objects**: `mission_log`, `mission_rule`.

### UC-A08 — EXIF/Hash Risk Signal Handling

- **Actors**: Participant, system, host
- **Classification**: shared signal rule (EXIF/hash = risk signal only; not standalone authority, not standalone action)
- **Preconditions**: Certification image exists and can be inspected.
- **Main Flow**: EXIF/hash is recorded as fraud/risk signal and may inform moderation or validation.
- **Failure Flow**: Missing EXIF or duplicate hash is treated as final fraud/settlement failure without layered review.
- **Authority Boundary**: EXIF/hash alone is not final authority unless a downstream canonical rule explicitly resolves it.
- **Projection Impact**: May mark candidate as needing review or change estimated input before freeze.
- **Settlement Impact**: Final settlement consumes resolved certification input, not raw signal alone.
- **UX Risk**: User feels accused of cheating.
- **Related Domain Objects**: `mission_log`, file hash, EXIF signal.

### UC-A09 — Duplicate or Excess Certification Under Cadence Rules

- **Actors**: Participant, system
- **Classification**: shared input rule (cadence cap; projection·settlement에 동일 적용; diagram에서는 floating «shared input rule» 노드로 표현될 수 있음)
- **Preconditions**: Multiple successful raw logs exist in the same cadence period.
- **Main Flow**: Raw logs remain append-only; general feed preserves visible activity history, while slot/dashboard projection and settlement recognize only allowed current/effective input according to cadence.
- **Failure Flow**: Feed item count or visible success badge inflates final settlement recognized count.
- **Authority Boundary**: Settlement recognition does not come from feed visibility or reaction activity. Reactions are social metadata only and do not affect certification, payout calculation, point ledger, crew status, or participant status.
- **Projection Impact**: Projection must apply current-basis recognition rules to latest/effective slot state, but remains a non-final estimate until Settlement.status = SUCCEEDED and point_history is committed.
- **Settlement Impact**: Excluded logs need calculation reasons.
- **UX Risk**: User thinks every visible feed post or reaction increases payout.
- **Related Domain Objects**: `mission_log`, `settlement_item`, `calculation_reason`.

### UC-A10 — Host Moderation of Certification Input

- **Actors**: Host, participant, system
- **Classification**: actor-performed usecase (Host moderation of certification input; append-only, pre-freeze only)
- **Preconditions**: Certification log exists and is eligible for review.
- **Main Flow**: Host records contextual certification input review decision with actor, reason category, and time; `mission_log` latest-effective moderation columns update and `moderation_history` receives an append-only row. If the participant reuploads, the feed may keep prior attempts visible as `이전 시도` while slot/projection uses the latest/effective state.
- **Failure Flow**: Host decision overwrites prior history, deletes audit rows, or directly mutates settlement/ledger.
- **Authority Boundary**: Host can affect certification input before freeze; host cannot determine settlement amount, ledger output, final settlement, participant baseline, replay, retry, or correction. Participants receive reason-code-level rejection explanation only; raw `reject_memo` text stays hidden in MVP.
- **Projection Impact**: Current-basis projection may update when effective moderation input changes, with explanation of the changed input state. Feed timeline visibility itself is not a settlement input mutation.
- **Settlement Impact**: Settlement consumes the resolved moderation state at freeze.
- **UX Risk**: Participant interprets rejection as host confiscating money or personally judging the participant.
- **Related Domain Objects**: `mission_log`, moderation history.

### UC-A10A — Account-Level Verification History Summary

- **Actors**: Participant, Host, system
- **Classification**: derived read-model usecase (current/effective verification summary; not audit ledger; not settlement authority)
- **Preconditions**: User is authenticated. Submitted `mission_log` rows or host moderation actions may exist in crews the user can access.
- **Main Flow**: User opens the global “검증 이력” surface. Without `role`, the system returns only the current user's participant-submitted verification summaries across crews. `crew_id` narrows the same summary to one crew. `role=host` explicitly returns summary rows for moderation actions in MVP crews hosted by the user.
- **Failure Flow**: The summary exposes raw moderation transition chain, `before_state`, `after_state`, `reject_memo`, internal actor identifiers, or final settlement counts; `role=host&crew_id` silently falls back to participant visibility for a non-host; a detail endpoint or mini timeline becomes a second audit surface.
- **Authority Boundary**: Verification history is a user-facing current/effective summary. `moderation-logs` remains the history-preserving append-only audit/detail surface. Settlement recognition remains in `settlement_item.calculation_reason` and linked `point_history`; feed visibility and reaction count do not decide recognized success count.
- **Projection Impact**: No projection authority. The summary may link to feed or settlement surfaces but must not present estimated/final payout as its own result.
- **Settlement Impact**: None. `SUCCESS` in this summary can still differ from recognized settlement success.
- **UX Risk**: Users can confuse “검증 이력” summary counts with final recognized success counts unless copy stays neutral: 검토중, 인정됨, 인정되지 않음, 아직 인증 전, 이전 시도, 재업로드.
- **Related Domain Objects**: `mission_log`, `moderation_history`, `crew`, `member`; no dedicated `verification_history` table.

### UC-A11 — Moderation Correction Before Freeze

- **Actors**: Host, participant, system
- **Classification**: extension / variant of UC-A10 (append-only correction event before freeze; not state overwrite)
- **Preconditions**: Prior moderation decision exists; settlement input not frozen.
- **Main Flow**: New moderation event is appended and the `mission_log` latest-effective snapshot may update; current-effective interpretation may change before freeze.
- **Failure Flow**: Prior decision is deleted, silently changed without a history row, or applied after freeze as payout mutation.
- **Authority Boundary**: Correction is append-only and only affects settlement input if before freeze. `reject_memo` remains internal/private context rather than participant-facing canonical explanation.
- **Projection Impact**: Current-basis expected settlement state and contribution/progress visibility may update with explanation.
- **Settlement Impact**: Settlement input changes only before freeze.
- **UX Risk**: User sees estimate change as arbitrary money removal rather than a visible input-state update.
- **Related Domain Objects**: moderation event history, `mission_log`.

### UC-A12 — Post-Freeze / Post-Success Correction Boundary

- **Actors**: Host, admin/support, participant, system
- **Classification**: invariant / boundary (settlement input freeze · post-final immutability; not a usecase action — diagram에서는 FREEZE block으로 표현될 수 있음. Correction workflow는 MVP unresolved/deferred)
- **Preconditions**: Settlement input is frozen or settlement succeeded.
- **Main Flow**: Host correction cannot mutate final settlement input. Any post-success correction lifecycle remains unresolved/defer-as-unsafe unless separately frozen by L1 authority.
- **Failure Flow**: Retry or host correction is used to change final payout.
- **Authority Boundary**: Post-freeze correction is not host moderation, not retry, and not replay. After final settlement, any correction must be separate support/operations adjustment semantics, not hidden mutation.
- **Projection Impact**: Projection should not imply final can still be changed by host action.
- **Settlement Impact**: Final settlement remains authoritative unless a future compensating correction process is explicitly defined.
- **UX Risk**: Users may expect support/host to “fix” payout by editing history.
- **Related Domain Objects**: `settlement`, `settlement_item`, moderation history.

### UC-A13 — Live Projection

- **Actors**: Participant, host, system
- **Classification**: system-driven projection (non-authoritative live estimate; projection ≠ final settlement)
- **Preconditions**: Mission active and enough inputs exist.
- **Main Flow**: Dashboard calculates current-basis estimated progress, expected settlement state, and contribution visibility.
- **Failure Flow**: Estimate is displayed as guaranteed payout, expected profit/loss, or adversarial rank loop.
- **Authority Boundary**: Projection is UX estimate only.
- **Projection Impact**: Core behavior.
- **Settlement Impact**: None directly.
- **UX Risk**: Estimate decrease feels like money being taken away, or increase feels like profiting from someone else's failure.
- **Related Domain Objects**: `mission_log`, dashboard projection response.

### UC-A14 — Post-End Current-Basis Projection Before Final Settlement

- **Actors**: Participant, system
- **Classification**: system-driven projection (non-authoritative current-basis estimate after mission end)
- **Preconditions**: Mission ended; settlement not yet succeeded.
- **Main Flow**: System shows a current-basis estimate from currently resolved inputs and explicitly labels it as not final settlement.
- **Failure Flow**: Post-end estimate is interpreted as final settlement or payout certainty.
- **Authority Boundary**: Post-end estimate remains non-authoritative until Settlement API/ledger finality.
- **Projection Impact**: Projection may become less volatile after mission end but remains a query-time estimate and may still differ from final.
- **Settlement Impact**: Final settlement may differ due to authoritative settlement rules and input freeze.
- **UX Risk**: User disputes final result because it differs from the last estimate.
- **Related Domain Objects**: `mission_room`, projection response, `settlement` status.

### UC-A15 — Final Settlement Success and Ledger Authority

- **Actors**: System, participants, support
- **Classification**: system-driven settlement authority (authoritative final-state producer; settlement snapshot → `point_history` ledger)
- **Preconditions**: Settlement item snapshot and point history linkage are valid.
- **Main Flow**: Settlement succeeds; final source of truth becomes settlement snapshot + `point_history`.
- **Failure Flow**: Settlement marked succeeded before every item is linked to valid ledger history.
- **Authority Boundary**: `point_history` is ledger source; balance/projection/support view is derived.
- **Projection Impact**: Dashboard should hand off to settlement result.
- **Settlement Impact**: Final.
- **UX Risk**: User looks at stale dashboard or notification instead of settlement detail.
- **Related Domain Objects**: `settlement`, `settlement_item`, `point_history`.

### UC-A16 — All-Fail Settlement and Deterministic Remainder

- **Actors**: System, participants, host
- **Classification**: variant / exceptional branch of UC-A15 (all-fail = equal principal refund; remainder = deterministic replayable calculation rule, not host authority/reward)
- **Preconditions**: Settlement input has no recognized successes, or rounding/remainder exists.
- **Main Flow**: All-fail settles by equal principal refund so that nobody's failure becomes another participant's profit. Any separate rounding/remainder rule must be deterministic, replayable, and not host-discretionary.
- **Failure Flow**: “전원 0원 환급”, “환급 없음”, house-edge wording, or host discretionary remainder wording conflicts with canonical settlement semantics.
- **Authority Boundary**: Remainder calculation is deterministic metadata, not host authority/reward/privilege.
- **Projection Impact**: Estimate must not imply discretionary host benefit, punitive pool loss, or failure-profit upside.
- **Settlement Impact**: All-fail equal principal refund is canonical; prior zero-refund wording remains brownfield drift only.
- **UX Risk**: Users see unfair host favoritism or gambling-like pool behavior.
- **Related Domain Objects**: `settlement_item`, settlement calculation reason.

### UC-A17 — Settlement Retry / Partial Recovery

- **Actors**: Admin, system
- **Classification**: recovery (resume incomplete settlement processing; retry ≠ recalculation; snapshot unchanged)
- **Preconditions**: Settlement failed or retry-wait with existing snapshot and possibly partial ledger effects.
- **Main Flow**: Retry resumes existing settlement work, reuses existing idempotent point history, or links missing `point_history_id`.
- **Failure Flow**: Retry recalculates new payout, rewrites final settlement, or creates duplicate point history.
- **Authority Boundary**: Retry is operational recovery, not correction.
- **Projection Impact**: User may see settlement pending/retry state.
- **Settlement Impact**: Snapshot remains same; missing work is completed.
- **UX Risk**: “Retry” sounds like admin can rerun and rewrite payout.
- **Related Domain Objects**: `settlement`, `settlement_item`, `point_history`, idempotency key.

### UC-A18 — Replay / Audit Verification

- **Actors**: Admin/support, system
- **Classification**: audit (settlement-time input reproduction; replay ≠ payout mutation; no ledger change)
- **Preconditions**: Settlement result exists and audit/reconciliation is requested.
- **Main Flow**: System reproduces/verifies result using settlement-time inputs/version/snapshot for audit explanation.
- **Failure Flow**: Replay uses current algorithm and produces different result, or mutates final payout.
- **Authority Boundary**: Replay is audit verification only; it is not retry, correction, recalculation, or payout rewrite.
- **Projection Impact**: None except explanation.
- **Settlement Impact**: No mutation unless a separately defined correction process exists.
- **UX Risk**: User thinks replay can change final result.
- **Related Domain Objects**: `settlement`, `settlement_item`, calculation version/snapshot candidate.

### UC-A19 — Notification and Reconnect State Drift

- **Actors**: Participant, system, client
- **Classification**: cross-cutting non-authoritative semantics (notification = hint only; canonical state is refetched from API)
- **Preconditions**: Notification/SSE/FCM/event delivery exists.
- **Main Flow**: Notification arrives as a best-effort re-entry hint; the client follows `deep_link` and refetches canonical API state before rendering current truth.
- **Failure Flow**: Late, missed, duplicate, or out-of-order notification contradicts current canonical state; canonical API state wins and notification failure does **not** trigger domain retry.
- **Authority Boundary**: Notification, FCM delivery state, inbox/read state, and delivery attempt state are non-authoritative UX/transport surfaces. They do not own crew lifecycle, certification, moderation, settlement, or point ledger truth.
- **Projection Impact**: UI may refresh estimates or final state after canonical refetch; notification payload/list text is not a projection or final settlement snapshot.
- **Settlement Impact**: None; notification failure cannot rollback settlement and notification retry is transport retry, not settlement retry/replay/correction.
- **UX Risk**: User thinks no notification means no payout, stale success means final state, unread means unresolved certification/settlement work, or inbox history is an audit ledger.
- **Related Domain Objects**: notification event/log candidate only if thin inbox persistence is later chosen (`read_at` nullable UX state only), deferred notification delivery attempt candidate, canonical API response. Event catalog names are app routing vocabulary candidates, not DB enum or audit authority.

### UC-A20 — Support Explanation by Lifecycle State

- **Actors**: Participant, support/admin, system
- **Classification**: cross-cutting support semantics (lifecycle-state explanation only; support creates no new authority)
- **Preconditions**: User disputes estimate, moderation, notification, retry, or final settlement.
- **Main Flow**: Support explains using the correct source-of-truth layer for the lifecycle state.
- **Failure Flow**: Support cites dashboard estimate as final, describes retry as correction, or promises support override of immutable settlement.
- **Authority Boundary**: Support explanation cannot create new authority.
- **Projection Impact**: Pre-settlement support explains current-basis projection formula, current inputs, and why it can change before final settlement.
- **Settlement Impact**: Post-settlement support explains settlement item + point history.
- **UX Risk**: Conflicting support answers destroy trust.
- **Related Domain Objects**: projection response, `settlement_item`, `point_history`, moderation timeline.


### UC-A21 — Crew Notice, Comment, and Notice Reaction Surface

- **Actors**: Host, participant/member, system
- **Classification**: cross-cutting non-authoritative communication semantics (crew notice/comment/reaction = social communication only)
- **Preconditions**: Crew exists; actor is authorized for the relevant communication action.
- **Main Flow**: Host posts operational notice; participants/members read, comment, or react to the notice. Hidden/deleted states affect visibility only.
- **Failure Flow**: Notice text is treated as mission rule override, host settlement decision, certification decision, point ledger mutation, or lifecycle command.
- **Authority Boundary**: Crew notice/comment/reaction are communication metadata. They do not own crew lifecycle, certification, moderation, settlement, point ledger, participant baseline, or replay/retry/correction truth.
- **Projection Impact**: UI may show notice/comment/reaction counts or lists as engagement context. Counts are derived social metadata and are not projection/final settlement inputs.
- **Settlement Impact**: None; communication activity cannot change final settlement inputs, settlement snapshots, refund amount, or point history.
- **UX Risk**: Users may follow a notice as if it changed canonical mission rules. Downstream copy must point to canonical crew/mission rule state when rules are discussed.
- **Related Domain Objects**: `crew_notice`, `crew_notice_comment`, `crew_notice_reaction`, `crew`, `mission_rule`.

### 4.A Usecase Classification Taxonomy

각 UC의 `**Classification**` 필드는 아래 카테고리 중 하나로 분류된다. UML usecase diagram 상에서 actor-performed usecase 노드로 표현되는 것과, invariant/boundary/shared-rule로 표현되는 것을 구분하기 위한 분류 체계다.

| Category | 의미 | 해당 UC |
|---|---|---|
| actor-performed usecase | actor가 직접 트리거하는 discrete action | UC-A01, UC-A02, UC-A06, UC-A10 |
| system-driven lifecycle event | system이 lifecycle 전이를 소유 | UC-A04, UC-A05 |
| system-driven projection | non-authoritative estimate를 system이 생산 | UC-A13, UC-A14 |
| system-driven settlement authority | authoritative final-state를 system이 생산 | UC-A15 |
| shared input rule / invariant | 여러 UC에 cross-cutting으로 적용되는 규칙·boundary | UC-A07, UC-A08, UC-A09, UC-A12 |
| extension / variant | 상위 UC의 edge-case branch | UC-A03, UC-A11, UC-A16 |
| recovery | 미완료 작업의 operational resume (mutation 없음) | UC-A17 |
| audit | 결과 검증·재현 (mutation 없음) | UC-A18 |
| cross-cutting non-authoritative semantics | hint 또는 설명 레이어 (state authority 없음) | UC-A19, UC-A20, UC-A21 |

### 4.B Diagram Projection Note

`docs/Usecase-diagram-dondok.md`는 UML 가독성과 authority boundary 보호를 위한 non-authoritative visual projection이다. 본 inventory가 behavioral semantic source이며, diagram 표현은 이를 화면화한 보조 산출물로만 소비한다. 본 inventory와 diagram 사이의 현재 매핑은 다음과 같다.

| UC | Diagram 표현 | 이유 |
|---|---|---|
| UC-A07 | UC-A06 라벨에 흡수 (server_time eligibility) | UC-A06 내부 입력 규칙 — standalone actor action이 아님 |
| UC-A08 | UC-A06/UC-A10 라벨에 흡수 (signal only) | risk signal — standalone authority 아님 |
| UC-A09 | floating «shared input rule» 노드 (no edges) | projection·settlement에 동일 적용되는 cross-cutting cadence rule |
| UC-A12 | FREEZE invariant block (boundary) | usecase action이 아닌 lifecycle boundary; post-final immutability |
| UC-A16 | UC-A15의 all-fail 분기 노트 | UC-A15의 variant branch; standalone authority 아님 |
| UC-A17 | ⑥ Recovery lane («non-authoritative» recovery) | retry는 recalculation이 아님 |
| UC-A18 | ⑦ Audit lane («append-only») | replay는 payout mutation이 아님 |
| UC-A19 | ⑧ OPS lane «non-authoritative» | hint layer; canonical state authority 없음 |
| UC-A21 | ⑨ Communication lane «non-authoritative» | crew notice/comment/reaction은 communication surface이며 rule/state authority 없음 |

Diagram에서 노드로 승격되지 않은 UC도 본 inventory에서는 normative semantic으로 보존된다. Diagram은 가독성을 위한 표현 layer이고, 본 inventory가 canonical behavioral semantic source다.

### 4.C Authority Boundary Matrix

각 actor가 보유한 권한을 명시적으로 나열한다. PRD §1.5 canonical constraints 및 PRD §7.2 host 해산 정책과 정합.

| Actor | Lifecycle 시작/종료 | Settlement 결정 | Ledger 변경 | Pre-freeze moderation 입력 | Post-freeze correction |
|---|---|---|---|---|---|
| Host | ✗ (system이 canonical activation anchor 소유; PRD §7.2의 host 해산은 start_at 시점 제한 내에서만 허용되며 본 inventory에서는 확장하지 않음) | ✗ | ✗ | ✓ (append-only contextual review input) | ✗ |
| Participant | ✗ | ✗ | ✗ | ✗ (자기 인증 제출 가능; 타인 입력 영향 없음) | ✗ |
| System | ✓ (canonical activation/cancellation anchor) | ✓ (snapshot · all-fail rule · deterministic remainder) | ✓ (`point_history` ledger) | ✗ (입력을 frozen state로 consume; 직접 mutation 없음) | ✗ (post-final immutable) |
| Admin/Support | ✗ | ✗ (snapshot 변경 불가) | ✗ (별도 correction process 정의 전까지 hidden mutation 금지) | ✗ | ✗ (retry/replay 가능; payout mutation 불가) |

Authority leakage 방지를 위해 위 매트릭스의 ✗는 “현재 캐노니컬 시멘틱에서 부재”를 의미하며, 새로운 권한 확장은 별도 L1 resolution을 거쳐야 한다.

### 4.D Freeze Boundary Clarification

Settlement input freeze는 본 inventory의 가장 critical한 boundary다. Freeze 전/후의 입력 가변성과 authoritative source를 명시한다.

| 시점 | Moderation 입력 | Settlement 입력 | Authoritative source |
|---|---|---|---|
| Pre-freeze | mutable (append-only event history) | not yet committed | 없음 — projection은 estimate only |
| Freeze 시점 | resolved state가 settlement input으로 consumed | committed snapshot 생성 | settlement snapshot 후보 |
| Post-freeze (settlement in-progress) | immutable | immutable | settlement snapshot (in-progress) |
| Final settlement succeeded | immutable | immutable | settlement snapshot + `point_history` ledger |

- Retry (UC-A17): 동일 snapshot으로 미완료 작업 resume — 재계산 아님.
- Replay (UC-A18): 동일 snapshot/version으로 재현·감사 — payout mutation 아님.
- Correction (UC-A12): post-freeze 입력 변경 또는 post-final ledger 변경은 hidden mutation으로 금지되며, MVP에서는 unresolved/deferred 상태. 향후 별도 correction process가 명시적으로 정의되기 전까지 admin/support는 explanation 권한만 보유 (UC-A20).

## 5. Pressure-Test Findings Summary

| PF Ref | Finding | Semantic Danger | Canonical Direction | Downstream Impact |
|---|---|---|---|---|
| PF-001 | Projection as promised payout or profit loop | Money-shaped estimates feel contractual or gambling-like | Always frame as current-basis estimate for anxiety reduction and settlement explanation; final settlement has separate source | PRD, API, Wireframe, QA, Support |
| PF-002 | Post-end projection wording | Post-end estimates can sound final | Use “현재 기준 예상” with settlement status and 변동 가능성; avoid finality wording before settlement succeeds | API, Wireframe, QA |
| PF-003 | Final differs from last estimate | User sees final delta as arbitrary | Provide explanation drivers: moderation, cadence cap, server-time cutoff, withdrawal/defer rule, tie/remainder | Settlement, API, Support, QA |
| PF-004 | Host rejection as confiscation | Moderation feels like money authority | Copy must state certification review input, not deposit/ledger decision | PRD, Wireframe, Support |
| PF-005 | Append-only hidden by latest-only UI | Users suspect tampering | Show timeline/progressive audit visibility where relevant | ERD, API, Wireframe |
| PF-006 | Host inactivity | Participants feel hostage to host | Preserve as warning unless it changes settlement freeze; do not invent workflow here | PRD, QA, Requirements/WBS |
| PF-007 | Bulk moderation | Convenience can amplify wrong decisions | Require per-log audit semantics if bulk exists | API, ERD, QA |
| PF-008 | Late/stale notifications | Notification becomes pseudo-state | Notification deep-links to canonical refresh; avoid final wording unless final is verified | API, Wireframe, QA |
| PF-009 | Upload success misunderstanding | Storage upload treated as certification | Separate upload object from MissionLog authority | API, Wireframe, QA |
| PF-010 | EXIF/hash over-authority | Risk signal becomes accusation/final failure | Treat as risk signal unless canonical rule resolves it | PRD, API, QA |
| PF-011 | Point history exists but item link missing | Wallet and settlement detail diverge | Retry recovers linkage within existing snapshot; do not recalculate or duplicate ledger | Settlement, Runbook, QA |
| PF-012 | Replay version drift | Audit produces different answer later | Store/use settlement-time version/snapshot requirements; replay verifies 당시 기준 result | ERD, Settlement, QA |
| PF-013 | Retry/correction confusion | Admin retry becomes payout editor | Retry resumes failed settlement processing; correction remains separate/deferred unless frozen | API, Runbook, Support |
| PF-014 | Correction as hidden history mutation | Users believe history can be edited | Do not design here; preserve unresolved hard blocker, separate support semantics, and append-only prohibition | PRD, ERD, Settlement |
| PF-015 | Live rank toxicity | Users hope others fail | Prefer contribution/progress/share framing over adversarial leaderboard or “1위 수익자” framing | PRD, Wireframe |
| PF-016 | Failure visibility shame | Accountability becomes humiliation | Avoid public shame mechanics and “실패자” labels; use supportive/private-to-user cues | Wireframe, QA |
| PF-017 | All-fail / tie / remainder fairness | Deterministic can still feel unfair or house-like | All-fail = equal principal refund; remainder must be replayable rule, not host discretion | Settlement detail, Support |
| PF-018 | Brownfield host-start drift | Host lifecycle authority contradicts canonical model | Label/remove/reframe as Drift Candidate | PRD, API, Settlement, QA |
| PF-019 | Support source confusion | Support answers become semantic authority drift | Lifecycle-specific support source hierarchy | Runbook, Support QA |
| PF-020 | Engagement over-hardening | Rank, projection, result card, feed, reaction, and notification are reduced to risk-only surfaces | Preserve mechanic visibility with cooperative wording; harden copy, not the underlying UX intent | PRD, Wireframe, QA |

## 6. Lifecycle Dependency Graph

```text
1. Crew rules created
   authority: host config only
   risk: host authority overread

2. Recruitment / approval / deposit lock
   authority: system + ledger constraints
   risk: pending vs locked baseline confusion

3. Frozen participant baseline
   authority: canonical lifecycle rules
   risk: ACTIVE 이후 baseline mutation

4. Automatic activation
   authority: system lifecycle
   risk: StartRoom / host manual start drift

5. Certification upload
   authority: storage evidence only
   risk: upload success != certification

6. MissionLog creation with server_time
   authority: certification event boundary
   risk: client/EXIF/processing time replaces server_time

7. EXIF/hash risk signals
   authority: signal only
   risk: signal treated as final fraud/failure

8. Host moderation
   authority: contextual certification input review
   risk: moderation perceived as settlement/ledger authority

9. Live projection
   authority: non-authoritative query-time current-basis estimate
   risk: estimate treated as contract, profit/loss, or failure-profit loop

10. Mission end / post-end current-basis projection
    authority: non-authoritative estimate using currently resolved inputs
    risk: post-end estimate treated as final settlement

11. Settlement input freeze
    authority: canonical freeze boundary
    risk: post-freeze moderation changes payout

12. Deterministic settlement batch
    authority: settlement engine
    risk: non-replayable algorithm, all-fail zero-refund drift, or brownfield payout mismatch

13. Settlement item snapshot
    authority: participant-level calculation result
    risk: snapshot absent or mutable

14. Point history ledger movement
    authority: money source of truth
    risk: balance cache or retry duplicates ledger

15. Settlement succeeded
    authority: settlement item + point_history final
    risk: notification/projection/support view overrides final source

16. Replay/audit/support explanation
    authority: audit verification and explanation only
    risk: replay/correction/retry conflation or support override authority
```

### Timing and freeze boundaries

- `recruitment_deadline`: participant approval/deposit-lock eligibility cutoff.
- `start_at`: planned activation anchor in PRD synthesis.
- `activated_at`: effective activation anchor; any divergence must be explicitly resolved across PRD/API/Settlement.
- `server_time`: certification timing authority.
- `end_at`: mission/certification cutoff and one input boundary for current-basis projection.
- settlement input freeze: hard blocker because it determines whether moderation/projection inputs can still affect final settlement.
- `Settlement.status = SUCCEEDED`: final authority transition only after settlement item and point history consistency are verified.

## 7. Unresolved Semantic Registry

| Semantic | Classification | Ambiguity | Authority Risk | Replay / Settlement Impact | Downstream Propagation Risk |
|---|---|---|---|---|---|
| Upload cutoff authority | Hard Blocker | Is eligibility based on request receipt, MissionLog creation, or validation completion? | Wrong timing authority can replace server_time/log boundary | Near-cutoff submissions can settle differently | API, Settlement, QA may encode incompatible rules |
| Settlement input freeze timing | Hard Blocker | Exact point where moderation/projection inputs stop affecting settlement is not fully centralized | Post-freeze host action may appear payout-affecting | Different final settlement inputs | Moderation, Projection, Settlement tests diverge |
| Settlement eligibility anchors | Hard Blocker | `start_at`, `activated_at`, `server_time`, withdrawal/deferred semantics can drift | Authoritative lifecycle interpretation splits | Two valid replay paths | PRD/API/Settlement mismatch |
| Replay/version snapshot requirements | Hard Blocker | Minimum data for settlement-time replay not fully frozen | Replay can become current-rule recalculation | Audit reproduction can differ | ERD/Settlement cannot prove replayability |
| Post-success correction lifecycle | Hard Blocker / Deferred Semantic | Formal correction/dispute workflow is not MVP-frozen | Hidden mutation or admin payout editing risk | Final settlement could be overwritten without append-only model | Do not design in API/ERD until L1 freezes it |
| All-fail refund mismatch | Resolved upstream / Brownfield Conflict | PRD now requires all-fail equal principal refund, but prior docs may still say all fail => 0 refund | Settlement constitution conflict if old wording propagates | Direct payout difference | Settlement/ERD/API/requirements/QA must remove or label zero-refund wording before propagation |
| Deterministic remainder | Hard Blocker / UX Warning | Remainder is replayable calculation metadata, not host reward/privilege | Host authority leakage | Remainder replay rule misunderstood as host privilege | PRD/Settlement/API/Support wording drift |
| Host manual start / `/start` | Brownfield Conflict | Existing docs/API may imply host lifecycle authority | Host becomes activation authority | Eligibility and projection anchors drift | Must be removed, deferred, or labeled before propagation |
| Moderation timeout / inactive host | Propagation Warning, possibly blocker if it affects freeze | What happens when host does not moderate before freeze? | Participant may feel hostage to host | If unresolved input affects final settlement, can become hard blocker | PRD/API/QA need label; no invented workflow here |
| Moderation visibility scope | Propagation Warning | Who sees full history vs latest effective result? | Append-only guarantee may be invisible | No direct payout change | Wireframe/API/support drift |
| Post-end projection wording | Propagation Warning | Post-end estimate sounds final | Projection becomes pseudo-settlement | Users dispute final delta | API/Wireframe copy drift |
| Notification stale-state reconciliation | Propagation Warning | Reconnect/late notification behavior wording incomplete | Notification becomes pseudo-authority | No direct settlement change | Client/QA/support drift |
| Support explanation hierarchy | Propagation Warning | Support may cite wrong source depending on lifecycle | Support becomes informal authority | Disputes handled from projection instead of settlement item | Runbook/support QA drift |
| Emotional trust framing | Propagation Warning | Ranking, rejection, warnings can feel punitive/gambling-like | Product trust erodes | No direct payout change | Wireframe/PRD/requirements/QA may drop as polish |
| Competitive mechanics vs emotional framing | Propagation Warning | Relative contribution visibility may be copied as adversarial leaderboard | Users start hoping others fail | No direct payout change | Wireframe/API/requirements/QA may encode first-place earner or failure-profit copy |

## 8. Downstream Propagation Guidance

### 8.1 How derived docs should consume this document

- PRD should use this document to check whether synthesis wording creates downstream ambiguity.
- ERD should use this document to identify data evidence needed for append-only history, replay, and source-of-truth guarantees.
- API spec should use this document to keep public state/copy from implying wrong authority.
- Settlement design should use this document to protect deterministic, replayable, explainable finality.
- QA should use this document to build scenario matrices around authority boundaries, not just endpoint success.
- Wireframes should use this document to avoid trust-breaking labels and misleading finality.
- Support runbooks should use this document to explain from the right source-of-truth layer.

### 8.2 Terms that must not drift

| Separation | Required Meaning |
|---|---|
| projection vs final settlement | Projection is current-basis estimate for explanation/visibility. Final is settlement snapshot + point history. |
| retry vs correction | Retry recovers existing settlement work. Correction is unresolved/deferred unless separately frozen. |
| replay vs recalculation | Replay verifies past result. Recalculation must not mutate final payout. |
| upload object vs mission-log authority | Upload object is evidence. MissionLog/server validation is certification event boundary. |
| moderation vs settlement authority | Host review can affect input before freeze; host cannot decide money. |
| notification/inbox/read vs canonical API state | Notification hints and inbox/read UX history are non-authoritative. Canonical API/state records decide current truth after deep-link/refetch. |
| crew notice/comment/reaction vs canonical rule/state | Crew communication surfaces explain and coordinate, but canonical rule/state remains in crew, mission_rule, mission_log, settlement, and point_history records. |
| append-only history vs visible latest state | Latest state may be summarized, but history must remain auditable. |
| competitive mechanics vs competitive framing | Contribution/progress visibility is allowed; adversarial “winner/profit from failure” framing is not. |
| all-fail refund vs punishment | All-fail equal principal refund prevents monetizing everyone’s failure; zero-refund/punitive wording is brownfield drift only. |

### 8.3 Dangerous wording patterns

- “Host starts/activates mission” without Drift Candidate label.
- “Expected refund” without current-basis qualifier.
- “예상 손익”, “실시간 수익 증가”, “더 벌었다”, or “수익률” as projection framing.
- “누군가 실패해서 상승”, “1위 수익자”, “지분왕”, or other failure-profit leaderboard copy.
- Post-end estimate phrased as final settlement.
- “Approved” implying payout approval.
- “Rejected” implying confiscation, punishment, or person-level judgment.
- “전원 0원”, “환급 없음”, “몰수”, “처벌”, or house-edge all-fail wording as canonical settlement.
- “Retry settlement” implying recalculation.
- “Replay settlement” implying payout can change.
- “Correction” implying hidden mutation of final settlement.
- “Upload complete” implying certification complete.
- “Notification success” implying canonical final state.

### 8.4 Propagation order

1. PRD semantic patch and authority synthesis.
2. Usecase semantic bridge alignment.
3. Settlement semantic patch.
4. ERD propagation.
5. API wording/contract propagation.
6. Requirements / WBS / GitHub Issues cleanup.
7. QA semantic matrix.
8. Wireframe/copy guidance.
9. Support runbook scripts.

## 9. UX Semantics Guidance

### 9.1 Philosophy

Dondok should not hide uncertainty, but it also should not make every screen feel like a legal disclaimer. The desired tone is calm, explanatory, and state-aware.

Safe competition is allowed only when it reinforces persistence, contribution, and shared progress. It must not become an emotional loop where users feel rewarded by another participant's failure.

Preferred:

- calm explanatory wording
- progressive disclosure
- visible auditability
- trust-through-visibility
- context-specific explanations
- non-punitive state labels
- contribution/progress/cooperative achievement framing

Avoid:

- legalistic warning spam
- deceptive certainty
- fear-based fraud language
- host-dictator framing
- leaderboard toxicity
- public shame mechanics
- failure-profit dopamine framing
- adversarial winner/loser or first-place earner framing

### 9.2 Safer wording examples

| Situation | Safer Direction | Dangerous Direction |
|---|---|---|
| Live estimate | “현재 기준 예상 환급입니다. 최종 정산 전 변동될 수 있어요.” | “받을 환급금” |
| Post-end estimate | “현재 기준 예상입니다. 최종 정산 전까지 변동될 수 있어요.” | “확정 금액 / 받을 환급금” |
| Projection update | “현재 인증 결과가 반영되었습니다. 크루 전체 진행 상황이 업데이트되었습니다.” | “누군가 실패해서 상승 / 더 벌었습니다” |
| Feed pending | “검토중입니다.” / “재업로드가 접수되었습니다.” | “곧 성공 처리됩니다 / 정산 반영 확정” |
| Feed failed | “인정되지 않음” / “이전 시도” | “사람을 낙인찍는 표현 / 처벌 확정처럼 보이는 표현” |
| Not submitted slot | “아직 인증 전” | “미제출 feed 게시물 / 금전 처벌 대상처럼 보이는 표현” |
| Host moderation accepted | “방장이 인증 내용을 검토했어요. 정산 입력에 반영될 수 있습니다.” | “환급 승인 완료” |
| Host moderation rejected | “인증 검토 결과 정산 입력에서 제외될 수 있어요. 사유와 이력을 확인할 수 있습니다.” | “몰수 / 실패 확정 / 문제 사용자” |
| Upload success | “이미지가 업로드되었습니다. 인증 제출 처리를 완료해야 합니다.” | “인증 성공” |
| EXIF/hash issue | “추가 확인이 필요한 이미지 신호가 있습니다.” | “부정행위 확정” |
| Retry | “기존 정산 복구를 이어서 처리 중입니다.” | “정산을 다시 계산합니다.” |
| Replay | “감사용으로 당시 기준 결과를 재현합니다.” | “결과를 다시 산정합니다.” |
| Correction | “최종 정산 이후 별도 운영 기준으로 진행되는 보정/지원 처리입니다.” | “정산 결과를 수정했습니다 / 몰래 보정했습니다” |
| Notification | “알림을 눌러 최신 상태를 확인하세요.” / “알림은 놓치지 않도록 돕는 안내이며, 최신 상태는 앱에서 다시 확인합니다.” | “이 알림이 최종 상태입니다.” / “알림 실패로 정산을 재시도합니다.” |
| All-fail | “이번 미션에서는 인정된 성공 기록이 없어, 누군가의 실패가 다른 참여자의 추가 환급으로 이어지지 않도록 원금을 기준으로 정산되었습니다.” | “전원 0원 / 환급 없음 / 실패자 몰수” |
| Tie/remainder | “정해진 deterministic rule에 따라 처리됩니다.” | “방장에게 임의 지급됩니다.” |
| Contribution visibility | “현재 상위 기여 그룹입니다 / 크루 평균 이상 달성 중입니다.” | “1위 수익자 / 지분왕 / 실패자 덕분에 상승” |
| Result share card | “이번 크루를 끝까지 버텼습니다 / 함께 인증을 쌓았습니다.” | “남들을 이겨서 벌었습니다” |

### 9.3 Emotional trust checks

A UX state/copy is semantically risky if it causes users to believe any of the following.

- Projection is a contract.
- Projection is expected profit/loss.
- Projection increase means somebody else's failure was monetized.
- Host can take money.
- Retry rewrites payout.
- Correction hides history mutation.
- Post-end estimate is final settlement.
- Ranking rewards other people’s failure.
- All-fail means punishment, house edge, or zero-refund elimination.
- Reject reason is a moral judgment about the person.
- Missing notification means no state change.
- Support can override ledger history by promise.

## 10. Appendix / References

### 10.1 Source documents

- `docs/PRD-dondok.md` — canonical synthesis layer.
- `docs/API-spec-dondok.md` — derived public API contract.
- `docs/ERD-dondok.md` — derived data model.
- `docs/Settlement-design.md` — derived settlement/recovery design.
- `docs/runbooks/settlement-recovery.md` — operational recovery guidance.

### 10.2 OMX semantic artifacts

- `.omx/specs/deep-interview-dondok-usecase-explosion.md`
- `.omx/interviews/dondok-usecase-explosion-20260521T080000Z.md`
- `.omx/specs/deep-interview-dondok-usecase-pressure-round-2.md`
- `.omx/interviews/dondok-usecase-pressure-round-2-20260521T082000Z.md`
- `.omx/specs/deep-interview-dondok-usecase-corpus-consolidation.md`
- `.omx/interviews/dondok-usecase-corpus-consolidation-20260521T083000Z.md`
- `.omx/plans/dondok-controlled-propagation-stabilization.md`
- `.omx/plans/dondok-semantic-propagation-stabilization.md`

### 10.3 Canonical review checklist for future edits

Before changing PRD/API/ERD/Settlement/QA/Wireframe/Support docs, check whether the change affects:

- lifecycle authority
- host authority boundary
- participant baseline
- deposit/ledger source of truth
- upload vs mission-log authority
- server-time eligibility
- moderation append-only history
- projection/final separation
- settlement input freeze
- retry/correction/replay separation
- notification state authority
- emotional trust semantics
- brownfield conflict visibility

If yes, update or cross-check this behavioral semantic bridge before propagation.
