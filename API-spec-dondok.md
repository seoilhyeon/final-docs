# API 명세: Dondok MVP

기준 문서:

- [PRD-dondok.md](./PRD-dondok.md)
- [Settlement-design.md](/Users/ilhyeon/Documents/projects/dondok/docs/Settlement-design.md)
- [ERD-dondok.md](./ERD-dondok.md)

## 1. 목적

이 문서는 Dondok MVP에서 FE와 BE가 병렬 개발할 수 있도록 API 계약을 고정하기 위한 문서다.

고정 원칙:

- API는 `REST + JSON` 기준으로 설계한다.
- 비즈니스 시간대는 `Asia/Seoul`이고, API 시각 값은 timezone offset이 포함된 `ISO-8601` 문자열로 주고받는다.
- 금액은 모두 `integer` 원 단위다.
- 보증금은 별도 자산 이동이 아니라 `lock` 모델이다.
- `Settlement.status = SUCCEEDED` 전 정산 계산 입력은 `MissionLog`, frozen `LOCKED` participant baseline, resolved certification state 기반 계산 결과다.
- `Settlement.status = SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다. 이후 `MissionLog` 기반 replay는 감사/디버깅용 검증에만 사용하며 지급 결과를 변경하지 않는다.
- `point_history`는 포인트 금액의 source of truth이고, `point_account.balance`는 재계산 가능한 현재값 캐시다.
- `Settlement.status = SUCCEEDED`는 모든 `settlement_item.point_history_id` 연결과 대응 `point_history` 존재 검증까지 완료된 경우에만 가능하다.
- 정산 재시도와 포인트 중복 지급 방지는 deterministic `idempotency_key`와 DB 제약을 함께 사용한다.

비범위:

- 외부 결제사 webhook/callback 기반 충전 확정과 운영 결제 키 전환 상세
- S3 public bucket 운영, object listing API, 클라이언트 임의 path/key 지정, 고급 upload metadata 관리 UI
- 관리자 FK 보정 API 상세
- AI 모델 비교, 토큰/비용 모니터링, 개인화 고도화, 장기 메모리, 복잡한 품질 평가 계약
- 중도 참여 / 재참여
- 부분 환급 / 중간 해제

## 2. 공통 규칙

### 2.1 Base URL

- MVP 기준 base path는 `/api`다.

### 2.2 인증

- 로그인 이후 API는 `Authorization: Bearer {accessToken}` 헤더를 사용한다.
- refresh token은 별도 API로 재발급한다.

### 2.3 식별자 경계

| Identifier | API/Auth/SSE 역할 |
| ---------- | ----------------- |
| `member.id` / `member_id` | DB 내부 FK / join / persistence identity다. 외부 API/JWT/SSE boundary의 canonical identifier로 사용하지 않는다. |
| `member.uuid` / `member_uuid` | external canonical identifier다. JWT `sub`, 외부 사용자 식별자, SSE 사용자 routing 기준으로 사용한다. |
| `email` | 로그인 식별자, 연락처, 사용자 정보다. PII이고 변경 가능하므로 routing identity, stream identifier, notification recipient key, JWT subject로 사용하지 않는다. |

정책:

- JWT access token subject(`sub`)는 `member.uuid`다.
- API 응답에서 사용자 식별자를 노출해야 하면 `member_uuid`를 사용한다.
- 기존 email 기반 SSE routing 또는 notification routing 표현/구현은 fallback이 아니라 제거 대상 anti-pattern이다.

### 2.4 시간

- 모든 응답 시간은 offset을 포함한 `ISO-8601` 문자열을 사용한다.
- 예시: `2026-05-07T00:05:00+09:00`
- 미션 기간과 정산 판단 기준 시간대는 `Asia/Seoul`로 고정한다.

### 2.5 금액

- 금액 필드는 모두 `integer`다.
- 예시: `100000`

### 2.6 에러 응답

모든 4xx/5xx 응답은 아래 형식을 사용한다.

```json
{
  "code": "ERROR_CODE",
  "message": "설명",
  "timestamp": "2026-05-07T00:05:00+09:00"
}
```

### 2.7 상태값 원칙

- `Crew.status`는 방 상태다. MVP에서 `RECRUITING -> ACTIVE`는 host command가 아니라 시스템 lifecycle 규칙으로 발생하며, `activated_at = start_at`이다.
- `Participant.status`는 참여 상태다.
- `Settlement.status`는 정산 처리 상태의 원천이다.
- `NONE`은 `Settlement` row가 아직 없는 상태를 보여주기 위한 API 응답용 값이다. DB `settlement.status`에는 저장하지 않는다.
- `PENDING`은 `Settlement` row가 생성됐지만 아직 claim되지 않은 실행 전 상태다.
- partial 지급/연결 상태는 복구 가능한 중간 상태이며 `SUCCEEDED`로 노출하지 않고 `RETRY_WAIT` 또는 `FAILED`로 남긴다.

## 3. 도메인 상태 / Enum

### 3.1 CrewStatus

- `RECRUITING`: 모집 중. `recruitment_deadline` 전에는 신청/승인/예치 Lock 흐름이 가능하고, 이후에는 신규 참여 불가.
- `ACTIVE`: 시스템 lifecycle 규칙이 `start_at`에 frozen eligibility를 만족한다고 판단해 `activated_at = start_at`을 기록한 진행 중 상태.
- `CLOSED`: 계획된 `end_at` 이후 정상 종료 상태.
- `CANCELLED`: 시작 전 취소 상태. `start_at` 자동 activation eligibility를 만족하지 못하면 batch가 취소형 정산 대상으로 전이할 수 있다.

### 3.2 ParticipantStatus

- `PENDING`: 사용자가 가입 신청을 완료하고 예치금이 reserve된 상태. 생성 트랜잭션에서 `point_account.balance`(available)가 감소하고 `crew_participant.deposit_amount`가 reserve snapshot이 된다. Capacity reservation에는 포함하지만 activation eligibility, frozen participant baseline, settlement 대상에는 포함하지 않는다.
- `LOCKED`: 방장 승인으로 `PENDING` reserve가 참여 확정으로 전환된 상태. approval은 별도 중간 상태 없이 `PENDING -> LOCKED`를 만들며 activation eligibility, minimum baseline, frozen participant baseline, settlement eligibility의 participant anchor다.
- `REJECTED`: 방장이 가입 신청을 거절한 terminal 상태. 기존 `PENDING` reserve는 취소 환급 원장으로 반환한다.
- `CANCELLED`: 사용자가 승인 전 `PENDING` 상태에서 신청을 취소한 terminal 상태. 기존 reserve는 취소 환급 원장으로 반환한다.
- `EXPIRED`: 시작 전까지 처리되지 않아 자동 만료된 terminal 상태. 기존 reserve는 취소 환급 원장으로 반환한다.
- MVP에서 `APPROVED_LOCK_PENDING` 같은 승인 후 lock 대기 중간 상태는 두지 않는다.
- `WITHDRAWN`/active withdrawal/재참여 semantics는 MVP active contract가 아니라 brownfield/deferred reference다.

### 3.3 FrequencyType

- `DAILY`
- `SPECIFIC_DAYS`
- `WEEKLY_N` (Phase 2 / deferred; MVP active cadence 아님)

### 3.4 SettlementType

- `NORMAL`
- `CANCELLED_BEFORE_START`

### 3.5 SettlementStatus

- `NONE` - API projection only
- `PENDING`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `RETRY_WAIT`

### 3.6 PointTransactionType

- `POINT_CHARGE`
- `CREW_DEPOSIT_LOCK`
- `CREW_SETTLEMENT_REFUND`
- `CREW_CANCELLED_REFUND`

### 3.7 MissionLogFailureReason

- system/timing/upload validation axis다. 호스트 검수자 거절 사유와 의미 vocabulary가 다르다.
- `EXIF_MISSING` (risk/review signal only; automatic final failure 아님)
- `EXIF_TIME_INVALID` (risk/review signal only; automatic final failure 아님)
- `BEFORE_START`
- `AFTER_END`
- `AFTER_WITHDRAWN` (brownfield/deferred; MVP ACTIVE withdrawal 정산 규칙으로 사용하지 않음)

### 3.8 DailySettlementType

- `A`: 일일 인증 마감 `09:00 KST`, 일일 정산 batch `12:00 KST`
- `B`: 일일 인증 마감 `21:00 KST`, 일일 정산 batch `00:00 KST` (익일)
- `C`: 일일 인증 마감 `23:59 KST`, 일일 정산 batch `익일 12:00 KST`
- 방 생성 시 필수. cadence anchor는 `Settlement-design.md`이 소유한다.

### 3.9 MissionLogDecisionType

- host/system moderation 결정 type. 값은 아래 4종으로 freeze한다.
- `MANUAL_APPROVE`
- `MANUAL_REJECT`
- `AUTO_APPROVE`
- `AUTO_REJECT`
- `AUTO_*`는 certification lifecycle의 system moderation outcome일 뿐 AI/admin/support/dispute/override 권한이나 settlement/ledger authority가 아니다.

### 3.10 MissionLogRejectReasonCode

- host moderation rejection reason axis. `MissionLogFailureReason`(system axis)과 의미 vocabulary가 분리된다. 값은 아래 6종으로 freeze한다.
- `TIME_VIOLATION`
- `DUPLICATE`
- `MISSION_MISMATCH`
- `UNCLEAR`
- `INAPPROPRIATE`
- `OTHER`
- `reject_memo`는 일반적으로 nullable이지만 `reject_reason_code = OTHER`일 때 필수이며 최대 50자다. internal/private context이고 participant-facing canonical state나 settlement/ledger authority가 아니다.

### 3.11 SettlementFailureCode

- `INPUT_LOAD_FAILED`
- `CALCULATION_FAILED`
- `POINT_CREDIT_FAILED`
- `DUPLICATE_SETTLEMENT`
- `LOCK_ACQUIRE_FAILED`
- `UNKNOWN`

### 3.12 AiHabitReportStatus — Phase 2 / Deferred

> AI habit report는 ERD에서 MVP Core entity로부터 제거되고 Phase 2 candidate로 격리되었다. 아래 enum은 MVP First Release contract가 아니라 Phase 2 reference로만 유지한다.

- `PENDING`
- `SUCCEEDED`
- `FAILED`

### 3.13 AiHabitReportFailureCode — Phase 2 / Deferred

> AI habit report 격리에 따라 본 enum도 Phase 2 reference로만 유지한다. MVP active contract가 아니다.

- `AI_REPORT_FAILED`
- `AI_RESPONSE_INVALID`
- `UNKNOWN`

### 3.14 ProjectionStatus

대시보드 projection 응답 전용 상태값이다.

이 값들은 DB에 저장되는 enum이 아니며, lifecycle 또는 settlement source-of-truth로 사용하지 않는다.

- `NOT_STARTED`
- `LIVE`
- `CLOSED_ESTIMATE`
- `NOT_PROVIDED`
- `SETTLEMENT_SUCCEEDED`

### 3.15 ProjectionNotice

대시보드 projection 응답의 현재 상태를 설명하기 위한 안내 값이다.

이 값들은 DB에 저장되지 않으며, projection 응답의 보조 설명 용도로만 사용한다.

- `ESTIMATED_NOT_FINAL`
- `NOT_STARTED`
- `NOT_PROVIDED`
- `SETTLEMENT_RESULT_AVAILABLE`
- `INSUFFICIENT_PROJECTION_INPUT`

### 3.16 PointHistoryReferenceType

- `POINT_CHARGE`
- `CREW_PARTICIPANT`
- `SETTLEMENT_ITEM`

### 3.17 MissionLogReactionType

- `reaction_type`은 고정 enum이 아니라 OS 기본 emoji picker 기반 string / normalized emoji token 후보다.
- FE는 사용자가 선택한 emoji grapheme/token 문자열을 그대로 전송한다.
- BE는 `trim`, blank reject, 기존 `VARCHAR(20)` 저장 길이 검증만 수행한다.
- MVP에서는 NFC/NFD 정규화, variation selector collapsing, ZWJ/skin-tone equivalence normalization을 적용하지 않는다. 같은 문자열만 같은 `reaction_type`으로 본다.
- API는 동일 `(mission_log_id, member_id, reaction_type)` 단위로 toggle/delete/idempotency를 판단한다. 한 회원이 같은 feed item에 여러 emoji token을 동시에 남길 수 있지만, 같은 token은 1회만 허용한다.
- 리액션은 소셜 메타데이터 전용이다. 포인트 원장, 정산, 환급, AI 리포트, 상태 생명주기 enum과 연결하지 않는다.

## 4. API 목록

| 도메인      | Method   | Path                                            | 설명                                     |
| ----------- | -------- | ----------------------------------------------- | ---------------------------------------- |
| 인증/회원   | `POST`   | `/api/auth/signup`                              | 회원가입                                 |
| 인증/회원   | `POST`   | `/api/auth/login`                               | 로그인                                   |
| 인증/회원   | `POST`   | `/api/auth/refresh`                             | access token 재발급                      |
| 인증/회원   | `POST`   | `/api/auth/logout`                              | 로그아웃                                 |
| 인증/회원   | `GET`    | `/api/me`                                       | 내 계정/프로필 조회                      |
| 인증/회원   | `PATCH`  | `/api/me/profile`                               | 내 최소 프로필 수정                      |
| 크루/참여   | `GET`    | `/api/crews`                                    | 크루 목록 조회                           |
| 크루/참여   | `POST`   | `/api/crews`                                    | 크루 생성                                |
| 크루/참여   | `GET`    | `/api/crews/{crewId}`                           | 크루 상세 조회                           |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/participants`              | 크루 가입 신청                           |
| 크루/참여   | `DELETE` | `/api/crews/{crewId}/participants/me`           | 가입 신청 취소 (승인 전 `PENDING`만)     |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/applications/{crewParticipantId}/approve` | 방장 승인 → 기존 reserve 확정 → `LOCKED` |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/applications/{crewParticipantId}/reject`  | 방장 거절 → `REJECTED`               |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/withdraw`                  | Brownfield/deferred withdrawal           |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/start`                     | Brownfield/removed manual start          |
| 미션 인증   | `POST`   | `/api/mission-logs`                             | 인증 제출                                |
| 미션 인증   | `GET`    | `/api/crews/{crewId}/mission-logs/me`           | 내 인증 기록 조회                        |
| 미션 인증   | `GET`    | `/api/crews/{crewId}/moderation-logs`           | 방장 검수 audit 조회                     |
| 피드/리액션 | `GET`    | `/api/crews/{crewId}/feed`                      | 방 인증 피드와 파생 일자 상태 조회       |
| 대시보드    | `GET`    | `/api/crews/{crewId}/dashboard`                 | 진행 상황/환급 설명용 current-basis projection 조회 |
| 피드/리액션 | `POST`   | `/api/mission-logs/{missionLogId}/reactions`    | 내 리액션 멱등 upsert                    |
| 피드/리액션 | `DELETE` | `/api/mission-logs/{missionLogId}/reactions/me?reaction_type={reaction_type}` | 내 특정 emoji token 리액션 멱등 삭제 |
| 정산        | `GET`    | `/api/crews/{crewId}/settlement`                | 방 기준 정산 상태/요약 조회              |
| 정산        | `GET`    | `/api/settlements/{settlementId}`               | 정산 결과 상세 조회                      |
| 정산        | `GET`    | `/api/admin/settlements`                        | 관리자 정산 실패/대기 목록 조회          |
| 정산        | `POST`   | `/api/admin/settlements/{settlementId}/retry`   | 관리자 정산 재시도                       |
| AI          | `POST`   | `/api/ai/mission-recommendations`               | AI 크루 생성 도우미. MVP 유지            |
| AI (Phase 2/Deferred) | `POST`   | `/api/crews/{crewId}/ai-habit-report`           | AI 습관 리포트 생성/조회. MVP 제외, Phase 2 격리 |
| AI (Phase 2/Deferred) | `GET`    | `/api/crews/{crewId}/ai-habit-report/me`        | AI 습관 리포트 상태/결과 조회. MVP 제외, Phase 2 격리 |
| AI (Phase 2/Deferred) | `GET`    | `/api/ai-habit-reports/{reportId}`              | AI 습관 리포트 단건 조회. MVP 제외, Phase 2 격리 |
| 알림        | `POST`   | `/api/notification-devices`                    | Android FCM token/device 등록 후보         |
| 알림        | `PATCH`  | `/api/notification-devices/{deviceId}`         | FCM token 갱신/상태 변경 후보              |
| 알림        | `DELETE` | `/api/notification-devices/{deviceId}`         | FCM token/device 비활성화 후보             |
| 알림        | `GET`    | `/api/notifications`                           | 알림 inbox UX hint 목록 조회 후보          |
| 알림        | `GET`    | `/api/notifications/unread-count`              | 미읽음 UX badge count 조회 후보            |
| 알림        | `PATCH`  | `/api/notifications/{notificationId}/read`     | 알림 읽음 UX 상태 처리 후보                |
| 알림        | `GET`    | `/api/notifications/stream`                    | Phase 2/deferred SSE realtime drift 후보   |
| 포인트      | `POST`   | `/api/points/charges`                           | 포인트 충전 반영                         |
| 포인트      | `GET`    | `/api/points`                                   | 사용 가능 잔액 조회                      |
| 포인트      | `GET`    | `/api/points/history`                           | 포인트 내역 조회                         |

## 5. API 상세

## 5.1 인증 / 회원

### `POST /api/auth/signup`

역할:

- 이메일 회원가입을 생성한다.

Request:

| 필드       | 타입     | 필수 | 설명          |
| ---------- | -------- | ---- | ------------- |
| `email`    | `string` | Y    | 로그인 식별자 |
| `password` | `string` | Y    | 비밀번호 원문 |
| `nickname` | `string` | Y    | 노출 이름     |

Response `201 Created`:

```json
{
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "email": "user@example.com",
  "nickname": "돈독러",
  "status": "ACTIVE",
  "created_at": "2026-05-07T09:00:00+09:00"
}
```

Error:

- `EMAIL_ALREADY_EXISTS`
- `VALIDATION_ERROR`

정책:

- `email`은 unique다.
- 가입 직후 자동 로그인 여부는 본 명세에서 고정하지 않는다. MVP 기본 흐름은 가입 후 로그인이다.

### `POST /api/auth/login`

역할:

- access token과 refresh token을 발급한다.

Request:

| 필드       | 타입     | 필수 | 설명          |
| ---------- | -------- | ---- | ------------- |
| `email`    | `string` | Y    | 로그인 식별자 |
| `password` | `string` | Y    | 비밀번호 원문 |

Response `200 OK`:

```json
{
  "access_token": "jwt-access-token",
  "refresh_token": "jwt-refresh-token",
  "member": {
    "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
    "email": "user@example.com",
    "nickname": "돈독러",
    "status": "ACTIVE"
  }
}
```

Error:

- `INVALID_CREDENTIALS`
- `MEMBER_DEACTIVATED`

정책:

- access token JWT subject(`sub`)는 `member.uuid`다. `email`이나 Long `member.id`를 subject로 사용하지 않는다.
- refresh token은 서버에 raw value가 아니라 hash로 저장한다.

### `POST /api/auth/refresh`

역할:

- refresh token으로 access token을 재발급한다.

Request:

| 필드            | 타입     | 필수 | 설명             |
| --------------- | -------- | ---- | ---------------- |
| `refresh_token` | `string` | Y    | 재발급 요청 토큰 |

Response `200 OK`:

```json
{
  "access_token": "new-access-token",
  "refresh_token": "new-refresh-token"
}
```

Error:

- `REFRESH_TOKEN_INVALID`
- `REFRESH_TOKEN_EXPIRED`
- `REFRESH_TOKEN_REVOKED`

정책:

- refresh token rotate 여부는 구현 선택이지만, 본 명세 예시는 rotate를 전제로 한다.

### `POST /api/auth/logout`

역할:

- refresh token을 revoke한다.

Request:

| 필드            | 타입     | 필수 | 설명        |
| --------------- | -------- | ---- | ----------- |
| `refresh_token` | `string` | Y    | 폐기할 토큰 |

Response `204 No Content`

Error:

- `REFRESH_TOKEN_INVALID`

### `GET /api/me`

역할:

- 현재 로그인한 회원 정보와 최소 프로필을 조회한다.
- 이 API는 사용자 프로필 조회(US-01A)를 포함한다.

Response `200 OK`:

```json
{
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "email": "user@example.com",
  "nickname": "돈독러",
  "profile_image_url": "https://cdn.example.com/profile/018f4fd2-6d7a-7a41-9f58-6d07f5c3c901/avatar.jpg",
  "status_message": "오늘도 한 걸음 더",
  "is_host_ever": true,
  "hosted_crew_count": 2,
  "status": "ACTIVE",
  "created_at": "2026-05-01T12:00:00+09:00"
}
```

정책:

- 프로필은 `nickname`, `profile_image_url`, `status_message`만 사용자 수정 가능 영역으로 노출한다.
- `profile_image_url`은 저장된 `member.profile_image_s3_key`에서 파생한 접근 URL이며, 이미지가 없으면 `null`일 수 있다.
- `status_message`는 자유 입력 한 줄 상태 메시지다. 최대 100자.
- `is_host_ever`, `hosted_crew_count`는 `member` 저장 컬럼이 아니라 profile read-model projection이다. 호스트 활동 이력에서 read-time으로 계산하며, 클라이언트 입력값으로 받지 않고 mutable counter로 다루지 않는다.
- 소셜 프로필 기능은 이 계약에 포함하지 않는다.

### `PATCH /api/me/profile`

역할:

- 현재 로그인한 회원의 최소 프로필을 수정한다.
- 닉네임과 프로필 이미지를 업데이트한다.
- 부분 업데이트를 지원한다.
- 소셜 프로필 기능은 포함하지 않는다.
- 이 API는 `US-01A`의 profile update 계약이다.

Request:

| 필드                   | 타입             | 필수 | 설명                                                                 |
| ---------------------- | ---------------- | ---- | -------------------------------------------------------------------- |
| `nickname`             | `string`         | N    | 수정할 노출 이름                                                     |
| `profile_image_s3_key` | `string \| null` | N    | 사전 업로드된 프로필 이미지 키. `null`이면 프로필 이미지를 제거한다. |
| `status_message`       | `string \| null` | N    | 자유 입력 상태 메시지(최대 100자). `null`이면 상태 메시지를 제거한다. |

Response `200 OK`:

```json
{
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "email": "user@example.com",
  "nickname": "돈독러",
  "profile_image_url": "https://cdn.example.com/profile/018f4fd2-6d7a-7a41-9f58-6d07f5c3c901/avatar.jpg",
  "status_message": "오늘도 한 걸음 더",
  "is_host_ever": true,
  "hosted_crew_count": 2,
  "status": "ACTIVE",
  "updated_at": "2026-05-01T12:10:00+09:00"
}
```

Error:

- `VALIDATION_ERROR`
- `PROFILE_IMAGE_NOT_FOUND`

정책:

- 요청에는 `nickname`, `profile_image_s3_key`, `status_message` 중 하나 이상이 있어야 한다.
- 프로필 이미지는 별도 presigned upload 흐름으로 먼저 업로드된 S3 key만 참조한다.
- 수정 범위는 닉네임, 프로필 이미지, 상태 메시지로 한정한다.
- `is_host_ever`, `hosted_crew_count`는 `member` 저장 컬럼이 아닌 profile projection이라 이 API로 수정할 수 없다. 응답에는 호스트 이력에서 파생한 현재 값을 노출한다.
- 인증/session 로직에 영향을 주지 않는다.
- 정산/포인트 시스템에 영향을 주지 않는다.
- 환급, 상태 생명주기는 변경하지 않는다.
- 소셜 프로필 기능은 이 API에 포함하지 않는다.

## 5.2 크루 / 참여

### `GET /api/crews`

역할:

- 크루 목록을 조회한다.

Query:

| 필드         | 타입     | 필수 | 설명                |
| ------------ | -------- | ---- | ------------------- |
| `status`     | `string` | N    | 기본값 `RECRUITING` |

Response `200 OK`:

```json
{
  "items": [
    {
      "crew_id": 42,
      "title": "새벽 기상 챌린지",
      "status": "RECRUITING",
      "deposit_amount": 100000,
      "min_participants": 2,
      "max_participants": 5,
      "frequency_type": "DAILY",
      "frequency_count": null,
      "mission_schedule_days": [],
      "recruitment_deadline": "2026-05-09T23:59:59+09:00",
      "start_at": "2026-05-10T00:00:00+09:00",
      "activated_at": null,
      "end_at": "2026-05-31T23:59:59+09:00"
    }
  ]
}
```

정책:

- MVP는 공개 크루만 지원한다.
- 참여자 수 같은 집계 필드는 본 명세의 필수 응답에 포함하지 않는다.

### `POST /api/crews`

역할:

- 크루 방과 인증 규칙을 생성한다.

Request:

| 필드                    | 타입       | 필수 | 설명                                                    |
| ----------------------- | ---------- | ---- | ------------------------------------------------------- |
| `title`                 | `string`   | Y    | 크루 제목. 표시용 텍스트이며 lifecycle/moderation/settlement authority가 아니다. |
| `description`           | `string`   | Y    | 크루 설명. 표시용 텍스트이며 lifecycle/moderation/settlement authority가 아니다. |
| `category`              | `string`   | Y    | 방 카테고리. 값 catalog는 deferred decision이며 string으로 받는다. 자세한 사항은 §3 enum 정책을 따른다. |
| `deposit_amount`        | `integer`  | Y    | 기본 보증금                                             |
| `min_participants`      | `integer`  | N    | 기본값 `2`                                              |
| `max_participants`      | `integer`  | Y    | 최대 인원                                               |
| `frequency_type`        | `string`   | Y    | MVP: `DAILY` / `SPECIFIC_DAYS`; `WEEKLY_N`은 Phase 2/deferred |
| `frequency_count`       | `integer`  | N    | Phase 2 `WEEKLY_N` reference 전용                       |
| `mission_schedule_days` | `string[]` | N    | `SPECIFIC_DAYS`일 때 필수. 예: `["MONDAY","WEDNESDAY"]` |
| `daily_settlement_type` | `string`   | Y    | `A` / `B` / `C`. 일일 정산 cadence. §3.8 참조 |
| `host_agreement`        | `object`   | Y    | 방장 동의/약관 스냅샷 payload. payload shape는 deferred decision이며 서버는 시점 스냅샷으로 보관한다. |
| `recruitment_deadline`  | `string`   | Y    | ISO-8601. 신규 참여 마감 시각                           |
| `start_date`            | `string`   | Y    | `YYYY-MM-DD`. 예정 시작일                               |
| `end_date`              | `string`   | Y    | `YYYY-MM-DD`. 계획된 종료일                             |

Response `201 Created`:

```json
{
  "crew_id": 42,
  "title": "새벽 기상 챌린지",
  "description": "매일 새벽 6시 전 기상 인증",
  "category": "EXERCISE",
  "status": "RECRUITING",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "SPECIFIC_DAYS",
  "frequency_count": null,
  "mission_schedule_days": ["MONDAY", "WEDNESDAY", "FRIDAY"],
  "daily_settlement_type": "A",
  "host_agreement_version": "v1",
  "host_agreed_at": "2026-05-07T09:00:00+09:00",
  "recruitment_deadline": "2026-05-09T23:59:59+09:00",
  "start_at": "2026-05-10T00:00:00+09:00",
  "activated_at": null,
  "end_at": "2026-05-31T23:59:59+09:00",
  "created_at": "2026-05-07T09:00:00+09:00"
}
```

Error:

- `VALIDATION_ERROR`
- `INVALID_DEPOSIT_AMOUNT`
- `INVALID_FREQUENCY_RULE`
- `INVALID_CATEGORY`
- `INVALID_DAILY_SETTLEMENT_TYPE`
- `HOST_AGREEMENT_REQUIRED`

정책:

- `deposit_amount`는 PRD synthesis 기준 `1,000원 ~ 100,000원`, `1,000원 단위`를 만족해야 한다.
- `min_participants`는 기본값 `2`고, `2 <= min_participants <= max_participants <= 15`를 만족해야 한다.
- `SPECIFIC_DAYS`는 특정 날짜가 아니라 반복 요일 규칙이며 `mission_schedule_day` 원본으로 저장한다.
- `recruitment_deadline`은 신규 참여 마감 시각이며 activation/settlement 기준이 아니다.
- `start_date`, `end_date`는 서버에서 `Asia/Seoul` 기준 `start_at`, `end_at`으로 정규화한다. `start_at`은 시스템 자동 activation anchor이며 MVP에서 `activated_at = start_at`이다.
- `end_at`은 계획된 미션 종료 cutoff이며 activation 지연으로 자동 이동하지 않는다.
- `category`는 필수다. 정확한 enum 값 catalog는 deferred decision이라 contract에서 freeze하지 않는다. 서버는 알려진 값만 통과시키고 unknown 값은 `INVALID_CATEGORY`로 거절한다.
- `daily_settlement_type`은 필수다. cadence anchor는 `Settlement-design.md`가 소유한다.
- `host_agreement`는 방 생성 시점 약관/규칙 동의의 스냅샷이다. 서버는 `host_agreement_snapshot`(JSON), `host_agreement_version`, `host_agreed_at`을 함께 저장한다. 이후 약관 본문이 바뀌어도 이 방의 동의 컨텍스트는 변하지 않는다.

### `GET /api/crews/{crewId}`

역할:

- 방 상세와 내 참여 상태를 조회한다.

Response `200 OK`:

```json
{
  "crew_id": 42,
  "host_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "title": "새벽 기상 챌린지",
  "description": "매일 아침 6시 전에 인증",
  "category": "EXERCISE",
  "status": "ACTIVE",
  "settlement_status": "NONE",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "DAILY",
  "frequency_count": null,
  "mission_schedule_days": [],
  "daily_settlement_type": "A",
  "host_agreement_version": "v1",
  "host_agreed_at": "2026-05-07T09:00:00+09:00",
  "recruitment_deadline": "2026-05-09T23:59:59+09:00",
  "start_at": "2026-05-10T00:00:00+09:00",
  "activated_at": "2026-05-10T00:00:00+09:00",
  "end_at": "2026-05-31T23:59:59+09:00",
  "my_participation": {
    "crew_participant_id": 101,
    "status": "LOCKED",
    "deposit_locked_amount": 100000,
    "locked_at": "2026-05-08T13:00:00+09:00",
    "withdrawn_at": null
  }
}
```

Error:

- `CREW_NOT_FOUND`

정책:

- `settlement_status`는 조회 최적화용 projection이다.
- 정산 처리의 원천 상태는 `Settlement.status`다.
- `my_participation`이 없으면 아직 참여하지 않은 회원이다.
- `category`, `daily_settlement_type`, `host_agreement_version`, `host_agreed_at`은 방 생성 시점 컨텍스트의 read-only 노출이며 변경할 수 없다.
- `host_agreement_snapshot` JSON 본문은 본 응답에서 직접 노출하지 않는다. 본문 노출 방식은 deferred decision이다.

### `POST /api/crews/{crewId}/participants`

역할:

- 크루 가입 신청을 생성하고 예치금을 reserve한다.
- MVP에서는 신청 생성 시 `PENDING` 상태로 시작하며, 같은 트랜잭션에서 사용 가능 잔액을 `crew.deposit_amount`만큼 차감하고 reserve snapshot을 남긴다.
- 방장 승인은 별도 보증금 차감 없이 기존 `PENDING` reserve를 `LOCKED` 참여로 확정한다.

Request:

- body 없음

Response `201 Created`:

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
  "status": "PENDING",
  "deposit_reserved_amount": 100000,
  "deposit_locked_amount": 0,
  "locked_at": null,
  "pending_at": "2026-05-08T13:00:00+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `CREW_NOT_RECRUITING`
- `CAPACITY_FULL`
- `INSUFFICIENT_BALANCE`
- `ALREADY_PARTICIPATING`
- `APPLICATION_NOT_ALLOWED`

정책:

- 신규 신청은 `RECRUITING` 상태이면서 서버 시간이 `recruitment_deadline` 전일 때만 허용한다.
- `recruitment_deadline` 이후에는 `CREW_RECRUITMENT_CLOSED` 또는 `CREW_NOT_RECRUITING` 계열 오류로 거절한다.
- 같은 `member`는 같은 `crew`에 단 하나의 `crew_participant` row만 가질 수 있다 (`unique(crew_id, member_id)`). 한 번 생성된 row는 lifecycle 종료 후에도 재사용/재생성하지 않는다.
- 진행 중 상태(`PENDING`, `LOCKED`)에서 동일 사용자가 신청 시도하면 `ALREADY_PARTICIPATING`로 거절한다.
- terminal 상태(`REJECTED`, `CANCELLED`, `EXPIRED`)에서 동일 사용자가 같은 방에 재신청 시도하면 `APPLICATION_NOT_ALLOWED`로 거절한다. MVP에서는 재참여/row 재사용/status 되돌리기를 허용하지 않는다.
- `PENDING` 상태는 capacity reservation에 포함한다. 신청 생성 시 capacity 확인은 `PENDING + LOCKED < crew.max_participants` 기준이다.
- `PENDING`은 `deposit_reserved_amount`를 갖지만 activation/minimum/frozen baseline과 settlement eligibility에는 포함하지 않는다. `LOCKED` 전이는 방장 승인 endpoint가 reserve를 확정하는 단일 상태 전이다.

### `DELETE /api/crews/{crewId}/participants/me`

역할:

- 본인의 `PENDING` 신청을 취소한다.
- 방장 승인 전(`PENDING` 상태)에만 가능하다.

Request:

- body 없음

Response `200 OK`:

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "CANCELLED",
  "cancelled_at": "2026-05-08T14:00:00+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `APPLICATION_NOT_CANCELLABLE`

정책:

- 취소는 `PENDING` 상태일 때만 허용한다. `LOCKED`, `REJECTED`, `EXPIRED`, `CANCELLED`는 `APPLICATION_NOT_CANCELLABLE`로 거절한다.
- 기존 reserve는 `CREW_CANCELLED_REFUND` 계열 point_history로 반환하고, `point_account.balance`를 같은 금액만큼 복구한다. terminal 전이와 reserve release는 같은 transaction에서 처리하며, release는 `crew_participant.id`당 한 번만 허용한다. 구현은 `released_point_history_id` 또는 `reserve_released_at` guard로 중복 release를 막는다.
- 멱등성: 동일 사용자가 이미 `CANCELLED`된 신청에 대해 다시 호출하면 `APPLICATION_NOT_CANCELLABLE`을 반환한다. 동일 idempotency 응답을 원하면 클라이언트가 `204 No Content` polling 패턴을 별도로 처리한다.
- `CANCELLED`는 pre-start exit 상태이며 capacity/baseline/settlement 대상이 아니다.

### `POST /api/crews/{crewId}/applications/{crewParticipantId}/approve`

역할:

- 방장이 `PENDING` 신청을 승인한다.
- 승인 처리는 기존 reserve의 참여 확정과 `LOCKED` 전이를 단일 트랜잭션으로 수행한다. 추가 예치금 차감이나 host settlement 권한은 발생하지 않는다.

Request:

- body 없음

Response `200 OK`:

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "LOCKED",
  "deposit_locked_amount": 100000,
  "locked_at": "2026-05-08T15:00:00+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `APPLICATION_NOT_APPROVABLE`

정책:

- 호출자는 해당 `crew.host_member_id`와 일치하는 사용자여야 한다. 아니면 `FORBIDDEN_NOT_HOST`.
- 승인은 `PENDING` 상태에서만 가능하다. `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`는 `APPLICATION_NOT_APPROVABLE`로 거절한다.
- 승인 트랜잭션 단계:
  1. 대상 row가 `PENDING`이고 기존 reserve가 존재하는지 확인한다.
  2. `crew_participant.status = LOCKED`
  3. `locked_at` 기록
  4. reserve snapshot(`deposit_amount`)은 그대로 유지해 activation/frozen baseline과 정산 입력으로 사용한다.
- 위 단계 중 하나라도 실패하면 `LOCKED`로 전이하지 않는다. 별도 중간 상태를 만들지 않고, 신청자 `crew_participant.status`는 `PENDING`로 유지하거나 승인 실패 응답으로 처리한다.
- `INSUFFICIENT_BALANCE`는 신청 생성 시 reserve에 실패할 때 발생한다. 승인 endpoint는 추가 잔액 차감을 수행하지 않는다.
- `CREW_DEPOSIT_LOCK point_history`는 신청 생성 시점의 reserve 성공 원장으로 생성된다. 사용자 측 별도 deposit-lock endpoint는 제공하지 않는다.

### `POST /api/crews/{crewId}/applications/{crewParticipantId}/reject`

역할:

- 방장이 `PENDING` 신청을 거절한다.

Request:

- body 없음

Response `200 OK`:

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "REJECTED",
  "rejected_at": "2026-05-08T15:00:00+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `APPLICATION_NOT_REJECTABLE`

정책:

- 호출자는 해당 `crew.host_member_id`와 일치해야 한다. 아니면 `FORBIDDEN_NOT_HOST`.
- 거절은 `PENDING` 상태에서만 가능하다. 다른 상태는 `APPLICATION_NOT_REJECTABLE`로 거절한다.
- 기존 reserve는 `CREW_CANCELLED_REFUND` 계열 point_history로 반환하고, `point_account.balance`를 같은 금액만큼 복구한다. terminal 전이와 reserve release는 같은 transaction에서 처리하며, release는 `crew_participant.id`당 한 번만 허용한다. 구현은 `released_point_history_id` 또는 `reserve_released_at` guard로 중복 release를 막는다.
- `REJECTED`는 terminal pre-start exit 상태이며 capacity/baseline/settlement 대상이 아니다. 동일 crew 재신청은 MVP에서 허용하지 않는다.

### `POST /api/crews/{crewId}/start` (Brownfield / removed from MVP active contract)

역할:

- 이 endpoint는 과거 host manual start 설계의 brownfield 흔적이며, MVP active API contract로 제공하지 않는다.
- MVP에서 `RECRUITING -> ACTIVE` 전이는 host/admin command가 아니라 시스템 lifecycle 규칙이 `start_at`에 수행한다.

Canonical replacement:

- `start_at`에 시스템은 frozen eligibility를 평가한다.
- activation eligibility는 `recruitment_deadline` 경과, `LOCKED` participant 수 `min_participants` 이상, host의 시작 전 해체 없음으로 판단한다. `LOCKED`는 reserve가 참여 확정으로 전환된 상태이므로 별도 승인 대기 확인이 필요 없다.
- `PENDING`, `REJECTED`, `CANCELLED`, `EXPIRED`는 activation/minimum/frozen baseline에는 포함하지 않는다. 단 `PENDING`은 승인 전 capacity reservation에는 포함한다.
- 조건을 만족하면 `status = ACTIVE`, `activated_at = start_at`이 된다.
- 조건을 만족하지 못하면 방은 시작 전 취소 정산 대상이 된다.
- scheduler 실제 실행 시각은 운영 구현 상세이며 product authority는 `start_at`이다.

정책:

- Host는 activation authority가 아니다.
- Admin도 manual ACTIVE transition authority가 아니다.
- 기존 클라이언트/구현 흔적이 이 endpoint를 참조하더라도, downstream propagation에서는 drift candidate로 취급하고 신규 ERD/API/QA authority로 확장하지 않는다.

### `POST /api/crews/{crewId}/withdraw` (Brownfield / deferred)

역할:

- ACTIVE withdrawal/재참여 semantics는 MVP active contract가 아니라 brownfield/deferred 영역이다.
- frozen participant baseline은 `start_at` 자동 activation 시점의 `LOCKED` 집합이며, withdrawal wording이 이 baseline을 소급 변경하는 권한으로 해석되면 안 된다.

Request/Response:

- MVP active contract에서는 제공하지 않는다.
- 기존 brownfield response shape가 남아 있더라도 frozen baseline, final settlement, point ledger 변경 권한으로 해석하지 않는다.

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `WITHDRAW_NOT_ALLOWED`

정책:

- 이 endpoint는 MVP active semantics로 고정하지 않는다.
- `RECRUITING` 중 사용자 신청 취소는 `DELETE /api/crews/{crewId}/participants/me`로 처리하며 `PENDING -> CANCELLED` 전이만 발생한다. `LOCKED` frozen baseline과 구분한다.
- `ACTIVE` 이후 withdrawal은 deferred이며, 정산에서는 frozen `LOCKED` baseline과 resolved certification state를 소급 변경하지 않는다는 원칙을 우선한다.
- 향후 withdrawal을 재도입하더라도 즉시 환급, final settlement mutation, point ledger 직접 변경으로 해석하면 안 된다.
- 구현 가이드: MVP 공개 계약은 `WITHDRAW_NOT_ALLOWED` 단일 코드를 사용한다.
- 구현 단계에서는 필요 시 `WITHDRAW_ALREADY_DONE`, `WITHDRAW_FORBIDDEN`, `WITHDRAW_NOT_ALLOWED`로 세분화할 수 있다.
- 세분화하더라도 API 응답 구조와 상위 코드 체계는 유지해야 한다.

## 5.3 미션 인증

### `POST /api/uploads/presigned-url`

역할:

- 인증 이미지와 프로필 이미지 업로드를 위한 private S3 presigned URL을 발급한다.
- S3 object key는 클라이언트가 정하지 않고 서버가 생성한다.

Request:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `purpose` | `string` | Y | `MISSION_IMAGE` 또는 `PROFILE_IMAGE` |
| `crew_id` | `integer` | N | mission image 업로드 시 대상 방 |
| `crew_participant_id` | `integer` | N | mission image 업로드 시 대상 참여자 |
| `content_type` | `string` | Y | 허용된 이미지 content type |
| `content_length` | `integer` | Y | 업로드 예정 파일 크기 |

Response `200 OK`:

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `upload_url` | `string` | 짧은 TTL의 presigned upload URL |
| `s3_key` | `string` | 서버가 생성한 object key |
| `expires_at` | `string` | 만료 시각 |

정책:

- S3 bucket/object는 private이다.
- object key는 서버가 생성한다.
- 사용자는 임의 S3 path/key를 지정할 수 없다.
- mission 인증 이미지의 권장 key 형식은 `mission/{crewId}/{crewParticipantId}/{uuid}`다. `{crewParticipantId}`는 `crew_participant.id`를 의미한다.
- presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- 서버는 발급 시점에 사용자, crew, participant 권한을 검증한다.
- 클라이언트는 발급받은 URL로 S3에 직접 업로드한다.
- 이후 클라이언트는 `image_s3_key`로 mission-log 생성 요청을 보낸다.
- 서버는 mission-log 생성 시 S3 object를 직접 조회해 존재 여부, size, content-type, ownership, EXIF를 검증한다.

### `POST /api/mission-logs`

역할:

- 미션 인증 로그를 append-only로 저장한다.

Request:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `crew_id` | `integer` | Y | 대상 방 |
| `image_s3_key` | `string` | Y | presigned upload API로 발급되고 업로드 완료된 이미지 key |
| `caption` | `string` | Y | 사진과 함께 제출하는 필수 인증 텍스트. 5~100자 |

Response `201 Created`:

```json
{
  "mission_log_id": 9001,
  "crew_id": 42,
  "crew_participant_id": 101,
  "image_url": "https://cdn.example.com/mission/9001.jpg",
  "image_s3_key": "mission/42/101/9001.jpg",
  "caption": "오늘도 미션 완료했습니다",
  "image_hash": "9b74c9897bac770ffc029102a200c5de8c0e9e5b9d3c9c7e5f4f5c1a2b3c4d5e",
  "server_time": "2026-05-11T05:58:10+09:00",
  "certification_status": "SUCCESS",
  "failure_reason": null,
  "decision_type": null,
  "reject_reason_code": null,
  "reject_memo": null
}
```

인증은 성공했지만 정산에서 제외될 수 있는 예시:

`note`는 문서 설명용이며 실제 API 응답 필드는 아니다.

```json
{
  "mission_log_id": 9003,
  "crew_id": 42,
  "crew_participant_id": 101,
  "caption": "아침 미션 인증합니다",
  "server_time": "2026-05-12T08:30:00+09:00",
  "certification_status": "SUCCESS",
  "failure_reason": null,
  "note": "SPECIFIC_DAYS 비해당 요일로 정산 시 제외될 수 있음. `certification_status`는 인증 성공 표시이며 settlement_item.calculation_reason이 최종 인정 여부를 결정한다."
}
```

실패 판정이지만 로그는 저장된 경우:

```json
{
  "mission_log_id": 9002,
  "crew_id": 42,
  "crew_participant_id": 101,
  "image_url": "https://cdn.example.com/mission/9002.jpg",
  "image_s3_key": "mission/42/101/9002.jpg",
  "caption": "시작 전 촬영한 인증입니다",
  "server_time": "2026-05-11T00:01:02+09:00",
  "certification_status": "FAILED",
  "failure_reason": "BEFORE_START"
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `PARTICIPANT_WITHDRAWN`

`PARTICIPANT_WITHDRAWN`와 `AFTER_WITHDRAWN`은 brownfield/deferred withdrawal 상태를 방어적으로 표현하는 코드다. MVP active settlement에서 frozen `LOCKED` baseline을 변경하는 근거가 아니다.

정책:

- 인증 시점에는 crew 단위 Redisson 락을 기본으로 사용하지 않는다.
- 인증은 `MissionLog` 원본 보존이 우선이다.
- 이미지 업로드 자체는 별도 presigned upload 계약으로 처리하고, 이 API는 업로드 완료된 `image_s3_key`와 필수 `caption`을 함께 받는다.
- 유효한 mission-log creation에는 서버가 검증한 `image_s3_key`와 5~100자 `caption`이 모두 필요하다. image-only 또는 caption-only 인증 생성은 허용하지 않는다.
- `image_url`은 조회/서빙용 nullable URL이며, 이미지 존재/범위 검증의 기준은 `image_s3_key`와 서버의 S3 object validation이다.
- `caption`은 feed/display/replay evidence 용도이고 단독 인증 성공/실패, 정산, 원장 기준이 아니다.
- Presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- 서버는 `image_s3_key`가 현재 사용자/participant/crew 범위에 속하는지 검증한다.
- 서버는 S3 object를 직접 조회해 존재 여부, size, content-type, ownership, EXIF를 검증한다.
- 클라이언트는 `exif_taken_at`을 authoritative source로 제출하지 않는다.
- 서버는 S3 object에서 EXIF/hash 등 risk signal을 추출하고 가능한 범위에서 검증한다.
- `image_hash`는 서버가 S3 object 바이트에서 직접 계산한 SHA-256 hex 값이다. 클라이언트가 제출한 hash를 신뢰하지 않고, 요청 body로도 받지 않는다. fraud/duplicate detection signal이며 authority가 아니다.
- `MissionLog.exif_taken_at`은 서버가 추출한 보조 metadata 저장값이며 authoritative timing source가 아니다.
- EXIF 부재나 이상은 단독 automatic failure가 아니라 fraud/risk signal이다. 필요한 경우 moderation/review flow로 라우팅한다.
- 정산 인정 판단의 timing anchor는 `server_time` 기준으로 수행한다.
- `server_time`은 서버가 인증 요청을 수신한 시각이다.
- `certification_status`는 인증 요청의 resolved certification state를 나타내며, 최종 정산 인정 여부를 보장하지 않는다. 권장 enum: `PENDING_REVIEW`, `SUCCESS`, `FAILED`.
- 아래 조건을 검토해 `certification_status`를 결정한다.
  - 업로드 object의 소유/범위/기본 무결성
  - EXIF/hash risk signal과 review 필요 여부
  - 미션 기간 내 요청 여부(`server_time` 기준)
  - frozen baseline / participant 상태 적합성
- `certification_status = SUCCESS`는 인증 성공 표시이며, 최종 정산에서 인정된다는 의미는 아니다.
- `certification_status = FAILED`여도 원본 로그는 저장할 수 있다.
- `certification_status = PENDING_REVIEW`는 업로드 직후 검수/판정 대기 상태다.
- `certification_status`는 인증 피드 badge, dashboard projection, 알림 input에 쓰이는 resolved state이며 EXIF/hash raw signal이나 host moderation `decision_type`/`reject_reason_code`와 동일 axis로 해석하지 않는다.
- `mission_log.failure_reason`은 인증 시점 실패 사유(system/timing axis)다.
- `decision_type`, `reject_reason_code`, `reject_memo`는 호스트 검수자 결과 axis이며 시스템 `failure_reason`과 의미 vocabulary가 다르다. 자세한 사항은 §3.9/§3.10 참조.
- POST 응답에서 `decision_type`, `reject_reason_code`, `reject_memo`는 검수가 일어나지 않은 시점에는 `null`이다. 검수 갱신은 별도 흐름이며 이 API는 검수 결과를 입력받지 않는다.
- `settlement_item.calculation_reason`은 정산 시점 포함/제외 근거다.
- MVP 인증 API에서 `OUT_OF_SCHEDULE`는 사용하지 않는다.
- 최종 정산에서의 인정 여부는 `certification_status`가 아니라 `Settlement` 계산 단계에서 결정된다.
- `SPECIFIC_DAYS`, `DAILY` 중복처럼 인증은 성공했지만 정산에서 제외되는 경우는 `mission_log.failure_reason`이 아니라 `settlement_item.calculation_reason`으로만 표현한다. `WEEKLY_N` 초과는 Phase 2/deferred reference다.
- 따라서 인증 시점 성공 로그도 최종 정산에서 제외될 수 있다. 예: `DAILY` 중복, `SPECIFIC_DAYS` 비유효 요일, pre-freeze host moderation으로 resolved certification state가 미인정인 경우.
- Dashboard projection은 추정값이고, `SUCCEEDED` 전 정산 계산값은 `MissionLog`, frozen `LOCKED` participant baseline, resolved certification state 기준으로 확정한다.

### `GET /api/crews/{crewId}/mission-logs/me`

역할:

- 현재 로그인한 회원의 해당 방 인증 기록을 조회한다.

Response `200 OK`:

```json
{
  "items": [
    {
      "mission_log_id": 9001,
      "crew_participant_id": 101,
      "image_url": "https://cdn.example.com/mission/9001.jpg",
      "caption": "오늘도 미션 완료했습니다",
      "image_hash": "9b74c9897bac770ffc029102a200c5de8c0e9e5b9d3c9c7e5f4f5c1a2b3c4d5e",
      "server_time": "2026-05-11T05:58:10+09:00",
      "exif_taken_at": "2026-05-11T05:57:58+09:00",
      "certification_status": "SUCCESS",
      "failure_reason": null,
      "decision_type": null,
      "reject_reason_code": null
    }
  ]
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`

정책:

- 이 API는 원시 인증 기록 조회용이다.
- 정산 인정 판단 기준 시간은 `MissionLog.server_time`이다.
- `exif_taken_at`은 서버가 S3 object에서 추출/검증한 촬영 시각 보조 정보이며, 최종 정산 인정 시각 기준으로 사용하지 않는다.
- `image_hash`는 서버 계산 SHA-256 결과의 read-only 노출이며, 동일 인증 사진 중복 의심 신호일 뿐 authority가 아니다.
- `certification_status`는 인증 요청의 resolved certification state(`PENDING_REVIEW`/`SUCCESS`/`FAILED`)이며, 정산에서 인정된 횟수를 나타내는 값이 아니다.
- `decision_type`, `reject_reason_code`는 현재 latest-effective 검수 결과 projection이다. 참여자-facing 응답은 `reject_reason_code`만 제공하고 `reject_memo`를 포함하지 않는다. `reject_memo`는 internal/private context다.
- FE는 이 값을 `최종 성공 횟수` 또는 `정산 인정 횟수`로 사용하면 안 된다.
- 최종 인정 여부와 인정 횟수는 반드시 정산 결과 API `GET /api/settlements/{settlementId}`를 기준으로 판단해야 한다.

### `GET /api/crews/{crewId}/moderation-logs`

역할:

- 방장 검수 audit trail을 read-only로 조회한다.
- `moderation_history` append-only 레코드를 기반으로 한다.

Query:

| 필드             | 타입       | 필수 | 설명                                  |
| ---------------- | ---------- | ---- | ------------------------------------- |
| `mission_log_id` | `integer`  | N    | 특정 인증 로그로 필터                 |
| `cursor`         | `string`   | N    | 페이지네이션 커서                     |
| `limit`          | `integer`  | N    | 기본 50, 최대 200                     |

Response `200 OK`:

```json
{
  "items": [
    {
      "moderation_history_id": 7001,
      "mission_log_id": 9001,
      "before_state": { "decision_type": null },
      "after_state": { "decision_type": "MANUAL_APPROVE" },
      "decision_type": "MANUAL_APPROVE",
      "reject_reason_code": null,
      "moderator_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
      "changed_at": "2026-05-12T11:00:00+09:00"
    }
  ],
  "next_cursor": null
}
```

Error:

- `CREW_NOT_FOUND`
- `FORBIDDEN`

정책:

- 본 API는 read-only audit 조회 전용이다. 검수 결정을 새로 만들거나 수정하지 않는다.
- `moderation_history`는 append-only다. 본 API는 기존 레코드를 변경/삭제하지 않는다.
- 조회 권한 매트릭스(누가 어디까지 볼 수 있는지)는 deferred decision이다. MVP 1차 구현 범위는 호스트 본인 + 본인 인증 로그에 대한 본인 참여자로 한정한다.
- `decision_type`은 `MANUAL_APPROVE`, `MANUAL_REJECT`, `AUTO_APPROVE`, `AUTO_REJECT`만 사용한다.
- `reject_reason_code`는 `TIME_VIOLATION`, `DUPLICATE`, `MISSION_MISMATCH`, `UNCLEAR`, `INAPPROPRIATE`, `OTHER`만 사용한다.
- `reject_memo`는 일반적으로 nullable이지만 `OTHER`일 때 필수이며 최대 50자다. internal/private context이므로 participant-facing 응답에는 포함하지 않는다. `OTHER`여도 참여자는 raw memo text가 아니라 `reject_reason_code`만 받는다.
- `before_state`, `after_state`는 검수 결정 시점의 latest-effective snapshot JSON이다. 정산 결과를 재계산하는 입력으로 사용하지 않는다.
- 검수자 식별은 `moderator_member_uuid`로만 노출한다. internal FK `moderator_id`는 응답에 포함하지 않는다.
- 이 API는 운영 admin 권한 endpoint가 아니다. MVP에서는 admin/correction workflow를 발명하지 않는다.

## 5.4 인증 피드 / 리액션

이 섹션은 성공 인증을 보여주는 소셜 projection 계약이다. `feed_items[]`와 `day_statuses[]` / `participant_day_slots[]`는 같은 화면에서 함께 쓰일 수 있지만 의미가 다르다.

- `feed_items[]`는 feed-eligible `MissionLog` 게시물 목록이다. feed-eligible은 `mission_log.certification_status = 'SUCCESS'` 인증 성공 로그만 뜻한다.
- `day_statuses[]`와 `participant_day_slots[]`는 참여자/일자 표시용 파생 상태다. 값은 `SUCCESS`, `FAILED`, `NOT_SUBMITTED`만 사용한다.
- 파생 상태는 DB 상태가 아니고 피드 게시물이 아니며 정산 입력도 아니다.
- feed eligibility는 정산 인정, 환급, 포인트 적립, AI 리포트 입력, `Settlement.status` 또는 크루/참여 생명주기 전이를 의미하지 않는다.
- Canonical rule: Feed success does NOT guarantee settlement inclusion. `feed_items[].certification_status = 'SUCCESS'`는 UX/social layer 표시 기준이며, 정산 포함 여부는 `MissionLog`, frozen `LOCKED` baseline, resolved certification state 기준의 settlement calculation과 `settlement_item.calculation_reason`이 결정한다.
- 정산 인정 여부와 최종 성공 횟수는 정산 API와 `settlement_item.calculation_reason`을 기준으로 판단한다.

### `GET /api/crews/{crewId}/feed`

역할:

- 방의 인증 성공 피드와 참여자/일자 파생 상태를 함께 조회한다.
- 성공 인증 원본 게시물과 실패/미제출 표시 상태를 명확히 분리한다.

Query:

| 필드     | 타입      | 필수 | 설명                                |
| -------- | --------- | ---- | ----------------------------------- |
| `limit`  | `integer` | N    | feed_items 페이지 크기. 기본 20     |
| `cursor` | `string`  | N    | feed_items 페이지 커서              |
| `from`   | `string`  | N    | 파생 상태 조회 시작일. `YYYY-MM-DD` |
| `to`     | `string`  | N    | 파생 상태 조회 종료일. `YYYY-MM-DD` |

Response `200 OK`:

```json
{
  "crew_id": 42,
  "feed_items": [
    {
      "mission_log_id": 9001,
      "crew_participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "nickname": "돈독러",
      "image_url": "https://cdn.example.com/mission/9001.jpg",
      "caption": "오늘도 미션 완료했습니다",
      "server_time": "2026-05-11T05:58:10+09:00",
      "created_at": "2026-05-11T05:58:10+09:00",
      "certification_status": "SUCCESS",
      "reaction_counts": {
        "👏": 2,
        "🔥": 1
      },
      "my_reactions": ["👏"]
    }
  ],
  "next_cursor": "2026-05-11T05:58:10+09:00_9001",
  "day_statuses": [
    {
      "date": "2026-05-11",
      "status": "SUCCESS",
      "representative_mission_log_id": 9001
    },
    {
      "date": "2026-05-12",
      "status": "NOT_SUBMITTED",
      "representative_mission_log_id": null
    }
  ],
  "participant_day_slots": [
    {
      "crew_participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "date": "2026-05-11",
      "status": "SUCCESS",
      "representative_mission_log_id": 9001
    },
    {
      "crew_participant_id": 102,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c908",
      "date": "2026-05-11",
      "status": "FAILED",
      "representative_mission_log_id": null
    }
  ]
}
```

Error:

- `CREW_NOT_FOUND`
- `CREW_ACCESS_DENIED`

정책:

- `feed_items[]`에는 `mission_log.certification_status = 'SUCCESS'`인 인증 성공 로그만 포함한다.
- `certification_status = 'FAILED'` 실패 로그와 `certification_status = 'PENDING_REVIEW'` 검수 대기 로그, 미제출일은 `feed_items[]`에 포함하지 않는다.
- 같은 참여자/같은 날짜에 성공 인증 로그가 여러 개 있으면 endpoint pagination/filtering에서 별도 제한하지 않는 한 raw successful feed post로 모두 남을 수 있다.
- 참여자/일자 파생 상태 대표 규칙:
  - 성공 로그가 하나 이상 있으면 `SUCCESS`다.
  - 대표 성공 로그는 가장 이른 successful `created_at`, 동률이면 가장 낮은 `mission_log.id`다.
  - 성공 로그가 없고 실패 시도가 하나 이상 있으면 `FAILED`다.
  - 성공/실패 로그가 모두 없으면 `NOT_SUBMITTED`다.
- 대표 규칙은 표시/API payload용이다. 원본 성공 피드 게시물을 삭제, 병합, 수정, 숨김 처리하지 않는다.
- `caption`은 feed item의 display/replay evidence로 포함될 수 있으며 단독 인증/정산 기준이 아니다.
- reaction counts는 `mission_log_reaction`에서 파생한다. `mission_log`에 저장 카운터를 두거나 갱신하지 않는다. `reaction_counts`는 emoji token을 key로 하는 동적 map이다.
- 이 API의 상태 projection은 정산 인정 횟수, 환급액, 포인트 잔액, AI 리포트 상태, lifecycle status의 source of truth가 아니다.

### `POST /api/mission-logs/{missionLogId}/reactions`

역할:

- 현재 로그인한 회원의 해당 인증 성공 게시물 리액션을 emoji token 단위로 멱등 toggle/create한다.

Request:

| 필드            | 타입     | 필수 | 설명                      |
| --------------- | -------- | ---- | ------------------------- |
| `reaction_type` | `string` | Y    | OS emoji string / normalized emoji token 후보 |

```json
{
  "reaction_type": "👏"
}
```

Response `200 OK`:

```json
{
  "mission_log_id": 9001,
  "my_reactions": ["👏", "🔥"],
  "reaction_counts": {
    "👏": 3,
    "🔥": 1
  }
}
```

Error:

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`
- `INVALID_REACTION_TYPE`

정책:

- 리액션 대상은 `mission_log.certification_status = 'SUCCESS'`인 feed-eligible `MissionLog`로 제한한다.
- `POST`는 `(mission_log_id, member_id, reaction_type)` 기준 멱등 create/toggle이다. 같은 emoji token이 이미 있으면 동일 token 단위로 idempotent하게 처리하고, 다른 emoji token은 별도 row로 공존할 수 있다.
- 구현은 `(mission_log_id, member_id, reaction_type)` unique constraint 기반의 DB-level idempotency를 MUST로 한다. SQL 문법은 실제 MySQL 8.0 stack에 맞춘다.
- 동일 `(mission_log_id, member_id, reaction_type)`에 대한 동시 중복 요청은 DB unique conflict 때문에 API 에러가 되어서는 안 되며, 최종 상태는 해당 token 1개 존재로 수렴해야 한다.
- 한 회원은 한 `MissionLog`에 여러 emoji token을 남길 수 있지만, 동일 token은 1회만 허용한다.
- Emoji token minimal-freeze: FE-selected token을 서버가 새 등가 규칙으로 바꾸지 않는다. BE는 trim 후 blank를 거절하고 `VARCHAR(20)` 저장 길이를 검증한다. NFC/NFD, variation selector, ZWJ/skin-tone 동등성 처리는 MVP에서 적용하지 않는다.
- 리액션 생성/수정은 `mission_log`를 mutate하지 않는다.
- 리액션은 정산, 환급, 포인트 원장, AI 리포트, 방/참여/정산 상태 전이에 side effect를 만들지 않는다.

### `DELETE /api/mission-logs/{missionLogId}/reactions/me?reaction_type={reaction_type}`

역할:

- 현재 로그인한 회원의 해당 인증 성공 게시물에서 지정한 emoji token 리액션을 멱등 삭제한다.

Query:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `reaction_type` | `string` | Y | 삭제할 OS emoji string / normalized emoji token 후보. URL encoding 필요 |

Response `200 OK`:

```json
{
  "mission_log_id": 9001,
  "my_reactions": ["🔥"],
  "reaction_counts": {
    "👏": 2,
    "🔥": 1
  }
}
```

Error:

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`

정책:

- 리액션이 이미 없어도 성공 응답을 반환한다.
- 삭제는 `(mission_log_id, member_id, reaction_type)` 기준 멱등 delete다. 같은 token만 삭제하며 다른 emoji token row는 유지한다.
- `reaction_type` query parameter는 required다. 클라이언트는 emoji token을 URL encoding해서 전송해야 하며, 서버는 POST와 같은 trim/blank/length 검증을 적용한다.
- 삭제도 `mission_log` 원본, 정산, 환급, 포인트 원장, AI 리포트, 상태 생명주기에 side effect를 만들지 않는다.

## 5.5 크루 대시보드

### `GET /api/crews/{crewId}/dashboard`

역할:

- 미션 방 화면에서 내 현재 수행 현황, 추정 인정 성공 횟수, 현재 기준 예상 환급금, 보증금 대비 예상 차이, 추정 지분율, 진행/참여도 표시 순서를 보여주는 UX용 projection API다.
- Dashboard는 단순 `MissionLog` 조회가 아니라 `MissionLog` 기반 진행 상황/환급 설명용 current-basis projection API다.
- Dashboard 값은 `Settlement.status = SUCCEEDED` 전까지 최종 정산 결과가 아니며, 정산 source of truth가 아니다.
- `Settlement.status = SUCCEEDED` 이후 최종 인정 성공 횟수, 최종 환급금, 최종 지분율은 `GET /api/settlements/{settlementId}`의 `settlement_item` 기준으로 표시한다.
- Dashboard projection과 최종 settlement 결과가 달라도 그 자체를 시스템 오류로 간주하지 않는다.

Response `200 OK`:

```json
{
  "crew_id": 101,
  "crew_participant_id": 1001,
  "settlement_id": null,
  "crew_status": "ACTIVE",
  "settlement_status": "NONE",
  "projection_status": "LIVE",
  "projection_notice": "ESTIMATED_NOT_FINAL",
  "my_deposit_amount": 6800,
  "my_success_count": 5,
  "my_recognized_success_count_estimated": 4,
  "total_recognized_success_count_estimated": 31,
  "my_share_ratio_estimated": "0.12903200",
  "my_expected_refund_amount": 8000,
  "my_expected_refund_delta_amount": 1200,
  "rank_estimated": 3,
  "updated_at": "2026-05-11T00:00:00+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `CREW_ACCESS_DENIED`

정책:

- `projection_status`와 `projection_notice`는 API 응답용 값이며 DB enum이나 도메인 상태 원천으로 저장하지 않는다.
- `settlement_status = NONE`은 해당 방의 `Settlement` row가 아직 없다는 뜻이다. Dashboard projection을 계산할 수 없다는 뜻이 아니다.
- `updated_at`은 현재 Dashboard projection 계산/응답 생성 시각이다. `MissionLog`의 최신 수정 시각이나 `settlement.finished_at`을 뜻하지 않는다.
- 모든 문서화된 필드는 응답에 포함한다. 적용할 수 없는 projection 필드는 생략하지 않고 `null`로 내려준다.
- `my_recognized_success_count_estimated`, `total_recognized_success_count_estimated`, `my_share_ratio_estimated`, `my_expected_refund_amount`, `my_expected_refund_delta_amount`, `rank_estimated`(= `current_rank`)는 모두 `MissionLog` / `crew_participant` / `crew` 입력에서 응답 시점에 계산하는 logical projection이며 저장 컬럼이 아니다. 모두 "현재 기준 예상" 값이며 "최종 정산 전 변동 가능"하다.

#### ProjectionStatus

| 값 | 의미 |
| --- | --- |
| `NOT_STARTED` | `RECRUITING` 등 미션 수행 전 상태라 진행/환급 projection이 아직 시작되지 않았다. |
| `LIVE` | `ACTIVE` 상태에서 현재 `MissionLog`와 참여자 상태를 기준으로 current-basis estimate를 계산했다. |
| `CLOSED_ESTIMATE` | `CLOSED` 상태에서 매 요청 시 현재 확인 가능한 입력으로 current-basis estimate를 계산한다. 저장된 dashboard snapshot이 아니며 최종값도 아니다. |
| `NOT_PROVIDED` | `CANCELLED` 등 수행 projection을 제공하지 않는 상태다. 환급/정산 안내는 Settlement API 기준이다. |
| `SETTLEMENT_SUCCEEDED` | 최종 정산이 성공했다. Dashboard는 최종값을 복제하지 않고 `settlement_id`로 Settlement API 조회를 유도한다. |

#### ProjectionNotice

| 값 | 의미 |
| --- | --- |
| `ESTIMATED_NOT_FINAL` | 현재 값은 참고용 current-basis estimate이며 최종 정산 결과가 아니다. |
| `NOT_STARTED` | 미션 수행 전이라 성과/보상 projection이 아직 시작되지 않았다. |
| `NOT_PROVIDED` | 현재 방 상태에서는 Dashboard 진행/환급 projection을 제공하지 않는다. |
| `SETTLEMENT_RESULT_AVAILABLE` | 최종 정산 결과가 존재하므로 `settlement_id`로 Settlement API를 조회해야 한다. |
| `INSUFFICIENT_PROJECTION_INPUT` | projection 계산에 필요한 참여자/보증금 입력을 충분히 확정할 수 없어 일부 추정 필드를 `null`로 반환한다. |

#### 상태별 필드 계약

| `projection_status` | 일반 crew status | `settlement_id` | `my_deposit_amount` | `my_success_count` | `my_recognized_success_count_estimated` | `total_recognized_success_count_estimated` | `my_share_ratio_estimated` | `my_expected_refund_amount` | `my_expected_refund_delta_amount` | `rank_estimated` | `updated_at` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `NOT_STARTED` | `RECRUITING` | `null` | value | `0` | `0` | `0` | `null` | `null` | `null` | `null` | value |
| `LIVE` | `ACTIVE` | nullable | value | value | value | value | value 또는 `null` | value 또는 `null` | value 또는 `null` | value 또는 `null` | value |
| `CLOSED_ESTIMATE` | `CLOSED` + not `SUCCEEDED` | nullable | value | value | value | value | value 또는 `null` | value 또는 `null` | value 또는 `null` | value 또는 `null` | value |
| `NOT_PROVIDED` | `CANCELLED` | nullable | value | `null` | `null` | `null` | `null` | `null` | `null` | `null` | value |
| `SETTLEMENT_SUCCEEDED` | any + `Settlement.status = SUCCEEDED` | value | value | `null` | `null` | `null` | `null` | `null` | `null` | `null` | value |

- `LIVE` / `CLOSED_ESTIMATE`에서 `settlement_id`는 실제 `Settlement` row가 있으면 값이고, `settlement_status = NONE`이면 `null`이다.
- `SETTLEMENT_SUCCEEDED`에서 성과/보상 projection 필드를 `null`로 내려주는 이유는 데이터가 없어서가 아니라 최종값의 source of truth가 `GET /api/settlements/{settlementId}`이기 때문이다.
- 대표 `projection_notice`는 `NOT_STARTED -> NOT_STARTED`, `LIVE/CLOSED_ESTIMATE -> ESTIMATED_NOT_FINAL`, `NOT_PROVIDED -> NOT_PROVIDED`, `SETTLEMENT_SUCCEEDED -> SETTLEMENT_RESULT_AVAILABLE`이다.
- `LIVE` / `CLOSED_ESTIMATE`에서 denominator 등 필수 projection 입력이 부족하면 해당 추정 필드는 `null`이고 `projection_notice = INSUFFICIENT_PROJECTION_INPUT`을 사용한다.

#### Projection source 역할

| Source | Dashboard에서의 역할 |
| --- | --- |
| `mission_log` | 성공 후보와 수행 현황의 primary event source다. `mission_log.certification_status = 'SUCCESS'` 로그만 후보로 사용하고, 인정 판단 시간은 `MissionLog.server_time` 기준이다. |
| `crew_participant` | 참여자 식별, frozen `LOCKED` baseline, `deposit_amount` 보증금 금액 source다. `withdrawn_at`은 brownfield/deferred reference다. |
| `crew` | 방 상태, 기간, 미션 주기/규칙 컨텍스트다. 총 보증금 source가 아니다. |
| `settlement` | `SUCCEEDED` 여부와 최종값 전환 판단용이다. `SUCCEEDED` 전 Dashboard projection 계산 source가 아니다. |
| `point_history` | 포인트 원장 source of truth다. Dashboard projection 계산 source가 아니다. 최종 환급/잔액 반영은 `Settlement.status = SUCCEEDED` 이후 Settlement API와 `point_history` 기준으로 확인한다. |
| `point_account` | 현재 사용 가능 잔액 캐시다. Dashboard projection 계산 source가 아니며 `locked_balance`와 `my_expected_refund_amount`를 합산/차감해 최종 금액을 추론하지 않는다. |

#### 계산 규칙

- Dashboard는 deterministic estimated projection이다. 같은 source 입력과 입력 상한을 사용하면 BE/FE/QA가 같은 projection 결과를 기대할 수 있어야 한다.
- `my_success_count`는 raw `mission_log.certification_status = 'SUCCESS'` 성공 로그 수다. 정산 인정 성공 수가 아니다.
- `my_recognized_success_count_estimated`는 현재 시점에서 정산 규칙을 가능한 범위로 반영한 추정 인정 성공 수다.
- 추정 인정 성공 수는 `MissionLog.server_time`을 `Asia/Seoul` 기준 날짜/요일/주차로 해석해 계산한다.
- projection 후보 로그는 `mission_log.certification_status = 'SUCCESS'`이고, `crew.activated_at <= MissionLog.server_time <= projection_input_until_at`을 만족해야 한다. `activated_at`이 `null`이면 post-activation projection을 계산하지 않는다.
  - `LIVE`에서는 `projection_input_until_at = min(응답 생성 시각, crew.end_at)`이다.
  - `CLOSED_ESTIMATE`에서는 `projection_input_until_at = crew.end_at`이며, 이는 query-time current-basis estimate의 입력 상한일 뿐 최종 정산 snapshot이 아니다.
  - `withdrawn_at` 기준은 brownfield/deferred reference이며 MVP Dashboard projection에서 `LOCKED` frozen baseline을 소급 변경하지 않는다.
- 대표 success 선택은 모든 frequency projection에서 동일하게 `MissionLog.server_time ASC`, 동률이면 `MissionLog.id ASC` 순서를 사용한다.
- `DAILY`는 같은 KST date의 첫 success만 인정하고 나머지 success는 duplicate로 제외한다.
- `SPECIFIC_DAYS`는 `mission_schedule_day`에 포함된 KST weekday의 success만 후보로 삼고, valid KST date별 첫 success만 인정한다.
- `WEEKLY_N`은 Phase 2/deferred cadence다. MVP Dashboard projection active contract에서는 계산하지 않는다.
  - 향후 재도입 시에도 activation anchor는 `crew.activated_at = start_at`이며 host/admin manual activation이 아니다.
- `total_recognized_success_count_estimated`는 참여자별 추정 인정 성공 수 합계다.
- `my_share_ratio_estimated`는 소수 정밀도 오해를 줄이기 위해 문자열 decimal로 반환한다.
- `my_expected_refund_amount`는 deterministic base UX estimate다. `total_recognized_success_count_estimated > 0`이면 `FLOOR(total_locked_amount × my_share_ratio_estimated)`로 계산한다.
- Dashboard는 정산의 `remainder`, `remainder_policy`, deterministic remainder allocation, 1원 단위 잔액 처리를 계산하거나 반영하지 않는다. 해당 최종 지급 차이는 `Settlement.status = SUCCEEDED` 이후 Settlement API에서만 확인한다.
- `my_expected_refund_delta_amount = my_expected_refund_amount - my_deposit_amount`다. 이 값은 수익 권위값이 아니라 현재 기준 환급 설명용 차이값이다.
- `rank_estimated`는 예상 환급금/수익/지분율/보증금 기준 순위가 아니라 추정 수행/참여도 표시 순서다. 정렬 기준은 `recognized_success_count_estimated DESC`, 동률이면 `crew_participant_id ASC`다.
- `total_recognized_success_count_estimated = 0`인 `LIVE` / `CLOSED_ESTIMATE` projection은 0으로 나누지 않고 all-fail equal-principal refund estimate를 적용한다.
- 전체 인정 성공 추정값이 `0`인 경우 Dashboard는 all-fail equal-principal refund 철학에 맞춰 각 참여자의 `deposit_amount`를 current-basis refund estimate로 보여준다. `WITHDRAWN`/ACTIVE withdrawal은 brownfield-deferred semantics다.
- zero-total base estimate의 `my_expected_refund_amount = my_deposit_amount`이며, 이 경우 Dashboard에서도 host/winner/draw remainder를 수행하지 않는다.
- denominator를 확정할 수 없으면 `my_share_ratio_estimated`, `my_expected_refund_amount`, `my_expected_refund_delta_amount`, `rank_estimated`는 `null`이고 `projection_notice = INSUFFICIENT_PROJECTION_INPUT`이다.
- `Settlement.status = SUCCEEDED` 이후 최종 인정 성공 횟수, 최종 환급금, 최종 지분율은 Dashboard projection보다 Settlement API가 우선하며, `settlement_item`과 연결된 `point_history`가 final source of truth다.
- Dashboard projection과 최종 settlement 결과가 달라도 시스템 오류로 보지 않는다.

#### Crew status별 동작

| Crew status | Dashboard 동작 |
| --- | --- |
| `RECRUITING` | 진행/환급 projection은 시작 전이다. 보증금, 방 규칙, `recruitment_deadline`, `start_at` 중심으로 표시한다. |
| `ACTIVE` | current-basis estimated projection을 계산한다. 모든 금액/비율/표시 순서는 추정값이다. |
| `CLOSED` | query-time current-basis estimate를 계산해 보여준다. 저장된 snapshot이 아니며, `Settlement.status = SUCCEEDED` 전까지 최종값이 아니므로 pending/running/retry 상태 안내를 함께 제공한다. |
| `CANCELLED` | 수행 성과 projection을 제공하지 않는다. 시작 전 취소 정산/환급은 Settlement API 기준으로 안내한다. |

#### locked_balance와의 관계

- `GET /api/points`의 `locked_balance`는 계정 단위 현재 잠긴 보증금 UX projection이다.
- Dashboard의 `my_expected_refund_amount`는 특정 crew/crew_participant 기준 예상 환급금 projection이다.
- FE는 `locked_balance`, `available_balance`, `total_balance`, `my_expected_refund_amount`를 합산하거나 차감해서 최종 보유 포인트, 출금 가능 금액, 최종 정산 후 확정되는 환급금을 계산하면 안 된다.
- `total_balance = available_balance + locked_balance` 관계는 포인트 요약 화면 전용이다.
- `CLOSED`지만 `Settlement.status != SUCCEEDED`인 방은 `locked_balance`에 아직 남을 수 있고, Dashboard는 non-final current-basis estimate인 `my_expected_refund_amount`를 보여줄 수 있다.
- 최종 환급 여부와 금액은 `Settlement.status = SUCCEEDED` 이후 Settlement API와 `point_history` 원장 기준이다.

## 5.6 정산

### `GET /api/crews/{crewId}/settlement`

역할:

- 방 기준으로 현재 정산 상태와 정산 식별자를 조회한다.

Response `200 OK`:

정산 row가 아직 없는 경우:

```json
{
  "crew_id": 42,
  "settlement_id": null,
  "settlement_type": null,
  "status": "NONE",
  "retry_count": 0,
  "failure_code": null,
  "failure_message": null,
  "started_at": null,
  "finished_at": null
}
```

정산 row가 있는 경우:

```json
{
  "crew_id": 42,
  "settlement_id": 501,
  "settlement_type": "NORMAL",
  "status": "RUNNING",
  "retry_count": 1,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-01T13:12:10+09:00",
  "finished_at": null
}
```

Error:

- `CREW_NOT_FOUND`

정책:

- `NONE`은 API projection이다.
- `PENDING -> RUNNING -> SUCCEEDED / RETRY_WAIT / FAILED`는 `Settlement.status` 원천 상태를 그대로 반영한다.
- `finished_at`은 성공/실패 종료 시각이다.
- `started_at`/`finished_at`은 runtime execution fact다. Lifecycle/cutoff authority는 `start_at`, crew timezone, daily cutoff, mission period end 같은 scheduled semantic anchor에 남는다.

### `GET /api/settlements/{settlementId}`

역할:

- 정산 스냅샷과 참여자별 결과를 조회한다.

Response `200 OK`:

```json
{
  "settlement_id": 501,
  "crew_id": 42,
  "settlement_type": "NORMAL",
  "status": "SUCCEEDED",
  "retry_count": 1,
  "total_participants": 5,
  "total_locked_amount": 500000,
  "total_recognized_success": 390,
  "total_base_refund_amount": 499996,
  "total_remainder_amount": 4,
  "remainder_policy": "DETERMINISTIC_REMAINDER_ALLOCATION",
  "remainder_winner_crew_participant_id": null,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-01T13:12:10+09:00",
  "finished_at": "2026-06-01T13:12:18+09:00",
  "items": [
    {
      "settlement_item_id": 7001,
      "crew_participant_id": 101,
      "participant_status_snapshot": "LOCKED",
      "deposit_amount": 100000,
      "success_count_raw": 92,
      "recognized_success_count": 90,
      "recognized_dates_count": 30,
      "excluded_success_count": 2,
      "withdrawn_at_snapshot": null,
      "share_ratio": "0.23076923",
      "base_refund_amount": 115384,
      "remainder_bonus_amount": 4,
      "reward_amount": 15388,
      "refund_amount": 115388,
      "final_amount": 115388,
      "point_history_id": 99001,
      "calculation_reason": {
        "includedDates": ["2026-05-01", "2026-05-02"],
        "excludedLogs": [
          {
            "serverTime": "2026-05-02T07:10:11+09:00",
            "code": "DAILY_DUPLICATE"
          }
        ]
      }
    }
  ]
}
```

Error:

- `SETTLEMENT_NOT_FOUND`

정책:

- `settlement_item`은 참여자별 계산 스냅샷의 source of truth다.
- `point_history`는 그 결과를 실제 잔액에 반영한 금액 source of truth다.
- `SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item + point_history`이며, `MissionLog` 기반 replay는 감사/디버깅용 검증에만 사용하고 지급 결과를 변경하지 않는다.
- Replay는 historical semantic truth reconstruction이다. API가 calculation context를 노출하는 경우에도 이는 당시 algorithm/rule/moderation/reason-code/lifecycle semantics 설명용이며 current-engine reinterpretation, payout rewrite, mutable recalculation 권한이 아니다.
- Versioned semantic replay는 v2 runtime이 v1 settlement semantics를 해석 가능하게 하는 것이다. v2 API/엔진이 v1 final settlement를 현재 규칙으로 덮어쓰는 migration-forward reinterpretation을 허용하지 않는다.
- `SUCCEEDED`는 모든 `settlement_item.point_history_id`가 채워지고 대응 `point_history` 존재가 검증된 상태를 뜻한다.
- partial 상태에서는 일부 item의 `point_history_id`가 `null`일 수 있고, 이 경우 `status`는 `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`다.
- 일반 정산에서 절사 후 남은 잔액은 deterministic remainder allocation rule로 처리한다. Brownfield `HOST_REMAINDER` 명칭은 legacy alias일 뿐 host reward/authority/privilege가 아니다.
- 전체 인정 성공 `0`이면 all-fail equal-principal refund를 적용한다. 각 참여자는 자기 `deposit_amount`를 환급받고 host/winner/draw remainder 추가 지급은 발생하지 않는다.
- `settlement.algorithm_version`, `settlement.rule_context_snapshot`, `settlement_item.effective_moderation_snapshot`, `settlement_item.moderation_chain_ref`은 정산 시점 컨텍스트 스냅샷 source-of-truth다(ERD §정산/Settlement-design §7 참조). 이 컨텍스트를 API 응답에 어떤 모양으로 노출할지는 deferred decision이다. 노출하더라도 read-only replay/audit 컨텍스트이고, 현재 엔진으로의 reinterpretation/recalculation 권한이 아니다.
- `settlement_item`에는 저장 `rank` 컬럼이 없다. UI 표시용 최종 순위가 필요하면 `final_rank`라는 logical projection으로 노출하고, `recognized_success_count DESC`, 동률이면 `crew_participant_id ASC` 기준으로 read-time 계산한다. `final_rank`는 payout authority가 아니며 지급 결과 변경에 사용하지 않는다.

### `GET /api/admin/settlements`

역할:

- 운영자가 실패/재시도 대기 정산을 조회한다.

Query:

| 필드     | 타입     | 필수 | 설명                            |
| -------- | -------- | ---- | ------------------------------- |
| `status` | `string` | Y    | `FAILED` 또는 `RETRY_WAIT` 권장 |

Response `200 OK`:

```json
{
  "items": [
    {
      "settlement_id": 501,
      "crew_id": 42,
      "settlement_type": "NORMAL",
      "status": "FAILED",
      "retry_count": 3,
      "failure_code": "POINT_CREDIT_FAILED",
      "failure_message": "point_history insert timeout",
      "started_at": "2026-06-01T13:12:10+09:00",
      "finished_at": "2026-06-01T13:12:20+09:00"
    }
  ]
}
```

### `POST /api/admin/settlements/{settlementId}/retry`

역할:

- 특정 `Settlement` row를 다시 claim해서 재시도한다.

Request:

- body 없음

Response `202 Accepted`:

```json
{
  "settlement_id": 501,
  "crew_id": 42,
  "status": "RUNNING",
  "retry_count": 2
}
```

Error:

- `SETTLEMENT_NOT_FOUND`
- `SETTLEMENT_NOT_RETRYABLE`
- `SETTLEMENT_ALREADY_SUCCEEDED`

정책:

- `FAILED` 또는 `RETRY_WAIT` 상태에서만 허용한다.
- retry 대상은 특정 `Settlement` row다.
- 같은 `crew`라도 `settlement_type`에 따라 별도 `Settlement`가 존재할 수 있으므로 retry 기준은 `crewId`가 아니라 `settlementId`다.
- 같은 방에 새 `Settlement`를 만드는 것이 아니라 지정된 기존 row를 재사용한다.
- 이미 생성된 `point_history`는 deterministic `idempotency_key`로 중복 지급이 차단된다.
- 동일 `idempotency_key`와 동일 payload의 중복은 기존 `point_history`를 재사용하거나 연결하고, 동일 키에 다른 payload가 확인되면 idempotency conflict로 실패 처리한다.
- partial 상태에서는 미지급 participant만 이어서 처리하거나, 이미 원장이 있으나 FK만 누락된 경우 기존 `point_history`를 재사용해 연결만 보정한다.
- retry는 unfinished execution completion authority다. 기존 snapshot/ledger/item을 교체하거나 current engine으로 재계산해 payout을 바꾸는 API가 아니다.
- admin retry는 별도 support adjustment 경로가 아니다. Retry는 기존 `Settlement`/`settlement_item` 기준의 interrupted execution 복구이며, frozen certification outcome, succeeded settlement snapshot, authoritative daily/final result를 변경하지 않는다.
- replay가 필요하더라도 replay는 historical reproducibility 검증용이며 payout mutation이나 payout rewrite endpoint가 아니다.

## 5.7 AI

AI API는 trust-loop authority가 아니다. AI 실패, 무응답, 유효하지 않은 응답은 비트랜잭션성 기능 실패이지 시스템 실패가 아니다. 따라서 수동 방 생성, 정산 결과 조회, 환급, 포인트 원장, `Settlement.status`를 차단하거나 변경하지 않는다.

MVP에 포함되는 AI 기능은 크루 생성 도우미(`POST /api/ai/mission-recommendations`) 하나뿐이다. AI habit report 계열 endpoint는 MVP First Release 계약에서 제외되며, Phase 2 / Deferred 섹션으로 격리한다. AI 크루 생성 도우미와 AI habit report는 별개 기능이며 혼동하지 않는다.

### `POST /api/ai/mission-recommendations`

역할:

- 크루 생성 전 사용자의 seed text 또는 일부 입력값을 받아 구조화된 미션 초안을 추천한다.
- 추천 결과는 저장이 아니라 폼에 반영할 수 있는 draft다.

Request:

| 필드                    | 타입            | 필수 | 설명                                   |
| ----------------------- | --------------- | ---- | -------------------------------------- |
| `seed_text`             | `string`        | Y    | 사용자의 목표/습관 설명                |
| `title`                 | `string`        | N    | 사용자가 이미 입력한 방 제목           |
| `description`           | `string`        | N    | 사용자가 이미 입력한 설명              |
| `frequency_type`        | `string`        | N    | MVP: `DAILY` / `SPECIFIC_DAYS`; `WEEKLY_N`은 Phase 2/deferred |
| `frequency_count`       | `integer`       | N    | Phase 2 `WEEKLY_N` reference 전용      |
| `mission_schedule_days` | `array<string>` | N    | 반복 요일 후보                         |
| `deposit_amount`        | `integer`       | N    | 보증금 후보                            |
| `duration_days`         | `integer`       | N    | 기간 후보                              |

Response `200 OK`:

```json
{
  "draft": {
    "title": "아침 20분 독서 인증",
    "description": "매일 아침 독서한 책 페이지를 사진으로 인증합니다.",
    "frequency_type": "DAILY",
    "frequency_count": 1,
    "mission_schedule_days": [],
    "deposit_amount": 50000,
    "duration_days": 30
  },
  "validation_warnings": [
    {
      "field": "deposit_amount",
      "message": "권장 보증금은 1,000원 단위로 조정되었습니다."
    }
  ]
}
```

Error:

- `AI_RECOMMENDATION_FAILED`
- `AI_RESPONSE_INVALID`
- `VALIDATION_ERROR`

정책:

- 추천 응답은 사용자가 확인/수정한 뒤 `POST /api/crews`로 별도 저장한다.
- 유효하지 않은 AI 응답은 자동 저장하지 않는다.
- 실패 응답을 받아도 FE는 기존 입력값을 유지하고 수동 생성 흐름을 계속 제공해야 한다.

### Phase 2 / Deferred: AI habit report

> 아래 endpoint는 MVP First Release 계약이 아니다. ERD에서도 `ai_habit_report`는 Core/MVP entity에서 제외되었고 Phase 2 candidate note로만 유지된다. AI habit report는 settlement / `point_history` / `MissionLog.certification_status` / lifecycle authority가 아니며, MVP runtime이 의존하지 않는다. 새 AI entity/API를 invent하지 않는다.

#### `POST /api/crews/{crewId}/ai-habit-report` — Phase 2 / Deferred

역할:

- 정산 성공 후 현재 사용자의 AI 습관 리포트 생성을 요청한다.
- 이미 같은 `settlement_id + member_id` 리포트가 있으면 새로 만들지 않고 기존 리포트 상태를 반환한다.
- 생성 기준은 성공한 정산 데이터이며, 리포트는 정산/환급/포인트 원장의 source of truth가 아니다.

Request:

- body 없음

Response `202 Accepted`:

```json
{
  "report_id": 9001,
  "crew_id": 42,
  "settlement_id": 501,
  "status": "PENDING"
}
```

이미 성공한 리포트 Response `200 OK`:

```json
{
  "report_id": 9001,
  "crew_id": 42,
  "settlement_id": 501,
  "status": "SUCCEEDED",
  "report_body": "30일 중 27일을 성공했고, 오전 루틴의 지속성이 높았습니다.",
  "created_at": "2026-06-01T00:10:00+09:00",
  "completed_at": "2026-06-01T00:10:08+09:00"
}
```

이미 실패한 리포트 Response `200 OK`:

```json
{
  "report_id": 9001,
  "crew_id": 42,
  "settlement_id": 501,
  "status": "FAILED",
  "report_body": null,
  "failure_code": "AI_REPORT_FAILED",
  "created_at": "2026-06-01T00:10:00+09:00",
  "completed_at": "2026-06-01T00:10:08+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `SETTLEMENT_NOT_SUCCEEDED`

정책:

- 리포트 입력은 `Settlement.status = SUCCEEDED`인 정산 결과와 실제 인증/정산 데이터를 기준으로 만든다.
- 멱등성 기준은 `settlement_id + member_id`이며, 영속성 방어선은 `unique(settlement_id, member_id)`다.
- 이 POST는 create-or-return-existing 계약이다.
- 상태별 동작은 아래와 같다.

| 기존 리포트 / 자격 상태                             | 동작                                                                                                                                                 | HTTP           |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| 성공한 정산이 없거나 현재 사용자가 대상 회원이 아님 | `SETTLEMENT_NOT_SUCCEEDED` 등 현재 4xx 에러를 반환하고 리포트 row를 만들지 않는다.                                                                   | 4xx            |
| 기존 리포트 없음                                    | `PENDING` 리포트 row 하나를 만들고 비동기 생성을 시작한다.                                                                                           | `202 Accepted` |
| 기존 `PENDING`                                      | 기존 pending 리포트를 반환하며 새 row를 삽입하거나 중복 생성을 시작하지 않는다.                                                                      | `202 Accepted` |
| 기존 `SUCCEEDED`                                    | 기존 completed 리포트를 반환한다.                                                                                                                    | `200 OK`       |
| 기존 `FAILED`                                       | `report_body: null`, `failure_code`가 포함된 기존 failed 리포트를 반환한다. 같은 POST는 재시도, 새 row 삽입, 정산/환급/원장 상태 변경을 하지 않는다. | `200 OK`       |

- 생성 실패는 리포트 상태를 `FAILED`로 남기되, 정산 성공 상태, 환급, `Settlement.status`, `settlement_item`, `point_history`를 변경하지 않는다.
- 향후 재시도를 추가한다면 별도 명시적 operation 또는 같은 row의 명시적 상태 전이로 설계해야 한다.
- AI report/explanation은 versioned + stale/invalidation-aware artifact로 발전할 수 있으나 non-authoritative다. Prompt/policy/model/input snapshot metadata를 추가하더라도 settlement authority, replay authority, payout truth가 되지 않는다.
- Regeneration append semantics, provider-level determinism, full AI replay reproducibility는 Phase 2 hardening registry에 남기며 이 MVP endpoint의 create-or-return-existing 계약을 바꾸지 않는다.

#### `GET /api/crews/{crewId}/ai-habit-report/me` — Phase 2 / Deferred

역할:

- 현재 사용자의 저장된 AI 습관 리포트 상태와 결과를 조회한다.

Response `200 OK`:

```json
{
  "report_id": 9001,
  "crew_id": 42,
  "settlement_id": 501,
  "status": "SUCCEEDED",
  "report_body": "30일 중 27일을 성공했고, 오전 루틴의 지속성이 높았습니다.",
  "failure_code": null,
  "created_at": "2026-06-01T00:10:00+09:00",
  "completed_at": "2026-06-01T00:10:08+09:00"
}
```

Error:

- `CREW_NOT_FOUND`
- `AI_REPORT_NOT_FOUND`

정책:

- `status`는 `PENDING`, `SUCCEEDED`, `FAILED` 중 하나다.
- `FAILED`여도 정산 결과 화면과 포인트 히스토리 조회는 그대로 가능해야 한다.
- AI 리포트의 stale/failed 상태는 정산 결과의 stale/failed 상태가 아니다. 정산 truth는 Settlement API와 point history 원장 기준이다.

#### `GET /api/ai-habit-reports/{reportId}` — Phase 2 / Deferred

역할:

- 리포트 식별자로 저장된 AI 습관 리포트 단건을 조회한다.

정책:

- 응답 구조와 상태 정책은 `GET /api/crews/{crewId}/ai-habit-report/me`와 동일하다.
- 리포트 소유자만 조회할 수 있다.

## 5.8 포인트

### `POST /api/points/charges`

역할:

- 외부 결제 승인 이후 포인트 충전 결과를 반영한다.

Request:

| 필드         | 타입      | 필수 | 설명                           |
| ------------ | --------- | ---- | ------------------------------ |
| `payment_id` | `string`  | Y    | TossPayments `paymentKey`. 기존 API 필드명은 유지하지만 값의 의미는 Toss 결제 승인 키다. |
| `order_id`   | `string`  | Y    | TossPayments `orderId`. confirm 검증과 로그 상관관계 추적용이며 멱등성 키로 사용하지 않는다. |
| `amount`     | `integer` | Y    | 충전 금액 |

Response `201 Created`:

```json
{
  "point_history_id": 3001,
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "amount": 50000,
  "balance_after": 350000,
  "transaction_type": "POINT_CHARGE",
  "created_at": "2026-05-07T09:30:00+09:00"
}
```

동일 `payment_id` 재시도 Response `200 OK`:

```json
{
  "point_history_id": 3001,
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "amount": 50000,
  "balance_after": 350000,
  "transaction_type": "POINT_CHARGE",
  "created_at": "2026-05-07T09:30:00+09:00"
}
```

Error:

- `INVALID_AMOUNT`
- `INVALID_PAYMENT_ID`
- `PAYMENT_ID_REUSED_WITH_DIFFERENT_AMOUNT`

정책:

- MVP는 TossPayments sandbox confirm-only 흐름을 사용한다.
- 서버는 Toss confirm 성공을 확인한 뒤에만 `POINT_CHARGE` 원장을 생성한다.
- `payment_id`는 TossPayments `paymentKey`이며 하나의 충전 이벤트만 의미해야 한다.
- `POINT_CHARGE` idempotency key는 `charge:{paymentKey}`다. API field 기준으로는 `charge:{payment_id}`다.
- `order_id`는 Toss `orderId`이며 confirm 검증과 로그 상관관계 추적용이다. `point_history.idempotency_key` 구성값으로 사용하지 않는다.
- 동일 `payment_id`와 동일 payload 재시도는 기존 `point_history`를 반환한다.
- 동일 `payment_id`와 다른 payload는 idempotency conflict로 실패한다.
- webhook/callback 기반 충전 확정과 별도 payment aggregate는 MVP 범위에서 제외한다.

### `GET /api/points`

역할:

- 현재 사용 가능한 포인트 잔액과 정산 전 묶인 보증금 projection을 함께 조회한다.

Response `200 OK`:

```json
{
  "available_balance": 350000,
  "reserved_balance": 100000,
  "active_locked_amount": 60000,
  "settlement_pending_amount": 40000,
  "locked_balance": 100000,
  "total_balance": 550000,
  "updated_at": "2026-05-07T09:30:00+09:00"
}
```

정책:

- `available_balance`는 `point_account.balance`이며, `PENDING` reserve 또는 `LOCKED` deposit으로 이미 차감된 뒤 현재 사용 가능한 포인트 잔액만 의미한다.
- `reserved_balance`, `active_locked_amount`, `settlement_pending_amount`, `locked_balance`는 DB 컬럼이 아니라 API 응답에서만 제공하는 wallet/projection 필드다. `settlement_pending_amount`는 DB/account column이 아니며 별도 settlement pending balance 컬럼도 두지 않는다.
- `reserved_balance`는 승인 전 `PENDING` reserve 표시용이고, `active_locked_amount`는 진행/모집 중 `LOCKED` deposit, `settlement_pending_amount`는 종료 후 최종 정산 전 `LOCKED` deposit 표시용이다. `locked_balance = active_locked_amount + settlement_pending_amount`다. 모두 포인트 원장의 source of truth가 아니다.
- MVP 기준 `reserved_balance`는 사용자의 `PENDING` 상태 `crew_participant.deposit_amount`, `locked_balance` 계열은 `LOCKED` 상태 `crew_participant.deposit_amount`를 `crew`과 조인해 계산한다. `REJECTED`/`CANCELLED`/`EXPIRED`는 반환 완료 terminal 상태라 합산 대상이 아니다.

```sql
SELECT rp.status, COALESCE(SUM(rp.deposit_amount), 0) AS amount
FROM crew_participant rp
JOIN crew mr ON mr.id = rp.crew_id
WHERE rp.member_id = :memberId
  AND rp.status IN ('PENDING', 'LOCKED')
  AND rp.deposit_amount > 0
  AND mr.status IN ('RECRUITING', 'ACTIVE', 'CLOSED')
GROUP BY rp.status
-- response layer maps PENDING -> reserved_balance; LOCKED + RECRUITING/ACTIVE -> active_locked_amount; LOCKED + CLOSED pre-SUCCEEDED -> settlement_pending_amount; locked_balance = active_locked_amount + settlement_pending_amount
```

- `RECRUITING`은 신청/승인 후 아직 시작 전이지만 보증금이 reserve 또는 locked 상태다.
- `ACTIVE`는 미션 진행 중이므로 보증금이 잠겨 있는 상태다.
- `CLOSED`는 정산 완료 전까지 `settlement_pending_amount`로 표시하며, 이 금액은 `locked_balance`에 포함한다.
- `WITHDRAWN`/ACTIVE withdrawal은 brownfield-deferred다. MVP locked balance projection은 frozen `LOCKED` baseline과 settlement status를 기준으로 해석한다.
- `Settlement.status = SUCCEEDED` 이후에는 해당 방의 보증금 lock이 해제된 것으로 본다.
- 다만 MVP projection은 settlement 조인을 강제하지 않고 `crew.status` 기반으로 시작한다. 따라서 `CLOSED` 포함은 정산 전 잠금 표시를 위한 근사값이며, 더 정확한 정산 상태 기반 제외 조건은 Settlement 조회/정산 구현 단계에서 보강할 수 있다.
- `total_balance = available_balance + reserved_balance + locked_balance`다.
- `locked_balance`와 `total_balance`는 출금 가능 여부, 환급 가능 여부, 분쟁 처리, 정산 결과 판단 기준으로 사용하지 않는다.
- 최종 정산/환급 결과는 정산 API와 `point_history`를 기준으로 판단한다.
- 내부 검증에서 `available_balance`와 `point_history` 재계산값이 다르면 `point_history` 기준으로 조사/보정한다.

### `GET /api/points/history`

역할:

- 포인트 원장 내역을 조회한다.

Query:

| 필드     | 타입      | 필수 | 설명                    |
| -------- | --------- | ---- | ----------------------- |
| `limit`  | `integer` | N    | 기본값 `20`, 최대 `100` |
| `cursor` | `string`  | N    | 다음 페이지 조회용 커서 |

Response `200 OK`:

```json
{
  "items": [
    {
      "point_history_id": 3001,
      "amount": 50000,
      "balance_after": 350000,
      "transaction_type": "POINT_CHARGE",
      "reference_type": "POINT_CHARGE",
      "reference_id": 3001,
      "created_at": "2026-05-07T09:30:00+09:00"
    }
  ],
  "page": {
    "limit": 20,
    "next_cursor": "2026-05-07T09:30:00+09:00_3001",
    "has_next": true
  }
}
```

예시 요청:

```http
GET /api/points/history?limit=20
GET /api/points/history?limit=20&cursor=2026-05-07T09:30:00+09:00_3001
```

Error:

- `INVALID_LIMIT`
- `INVALID_CURSOR`

정책:

- 포인트 내역은 최신순 `created_at DESC, point_history_id DESC`로 조회한다.
- 동일 `created_at`이 있을 수 있으므로 `point_history_id`를 보조 정렬 키로 사용한다.
- `cursor`는 마지막으로 조회한 항목의 `created_at + point_history_id`를 기반으로 생성한다.
- `cursor`는 클라이언트가 직접 해석하지 않고 다음 요청에 그대로 전달하는 값으로 취급한다.
- `limit`이 `1` 미만이거나 `100`을 초과하면 `INVALID_LIMIT`를 반환한다.
- `cursor` 형식이 잘못되었거나 해석할 수 없으면 `INVALID_CURSOR`를 반환한다.
- `CREW_DEPOSIT_LOCK`는 자산 이동이 아니라 lock 이벤트다.
- `CREW_SETTLEMENT_REFUND`와 `CREW_CANCELLED_REFUND`는 실제 사용 가능 잔액 증가 이벤트다.
- `reference_type`은 `POINT_CHARGE`, `CREW_PARTICIPANT`, `SETTLEMENT_ITEM`만 사용한다.
- `reference_type` / `reference_id` 매핑은 아래와 같다.

| 도메인 동작         | `transaction_type`       | `reference_type`   | `reference_id` 규칙                                                                                                 | `idempotency_key` 예시 |
| ------------------- | ------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| 포인트 충전         | `POINT_CHARGE`           | `POINT_CHARGE`     | MVP에서는 생성된 `point_history.id`를 사용한다. API의 `payment_id`에 담긴 Toss `paymentKey`는 `idempotency_key = charge:{paymentKey}`에 남긴다. | `charge:{paymentKey}` |
| 크루 참여 보증금 reserve | `CREW_DEPOSIT_LOCK`      | `CREW_PARTICIPANT` | `crew_participant.id`                                                                                               | `crew:{crewId}:participant:{participantId}:reserve` |
| PENDING reserve release | `CREW_CANCELLED_REFUND`  | `CREW_PARTICIPANT` | `crew_participant.id`                                                                                               | `crew:{crewId}:participant:{participantId}:reserve-release` |
| 일반 정산 환급      | `CREW_SETTLEMENT_REFUND` | `SETTLEMENT_ITEM`  | `settlement_item.id`                                                                                                | `crew:{crewId}:participant:{participantId}:settlement-refund:{settlementId}` |

`{participantId}` placeholder는 내부적으로 `crew_participant.id`를 가리킨다. API field로 직접 노출할 때는 `crewParticipantId`로 정렬한다. `{settlementType}`은 §3.8의 `daily_settlement_type` (`A` / `B` / `C`) 값이다.


## 5.9 알림 / Android FCM / Inbox / SSE drift

알림 API의 MVP 기준은 Android-first FCM이다. FCM은 delivery transport이고, notification은 best-effort re-entry hint다. 알림 payload, inbox list item, read/unread state, delivery attempt success/failure는 crew lifecycle, certification, moderation, settlement, point ledger/history의 canonical state authority가 아니다.

### MVP boundary

| 범위 | MVP 판단 | 권한 경계 |
| ---- | -------- | --------- |
| FCM token/device lifecycle | 포함 후보 | token/device transport state만 변경한다 |
| notification inbox/list/read | 포함 후보 | UX hint history/read affordance이며 audit/canonical history가 아니다 |
| unread count | 포함 후보 | badge 표시용 UX count이며 unresolved settlement/certification task가 아니다 |
| delivery attempt observability | 포함 후보 | FCM send attempt 관측/transport retry용이며 settlement evidence가 아니다 |
| SSE realtime stream | Phase 2/deferred drift 후보 | Android-first FCM MVP의 source가 아니며 realtime reliability 보장은 deferred다 |
| notification preference matrix | Phase 2 | 채널/이벤트별 수신 설정 freeze 대상 아님 |
| notification template CMS/table | Phase 2 | 문구 CMS/table freeze 대상 아님 |
| campaign/broadcast/advanced analytics | Phase 2 | 운영/마케팅 자동화는 MVP 밖 |

### FCM token/device lifecycle 후보

> Path naming은 후보이며, ERD/API propagation 단계에서 세부 스키마와 identifier 이름을 premature freeze하지 않는다.

| Method | Candidate path | 역할 |
| ------ | -------------- | ---- |
| `POST` | `/api/notification-devices` | 현재 인증 사용자의 Android FCM token/device 등록 |
| `PATCH` | `/api/notification-devices/{deviceId}` | token refresh, app version/platform metadata, enabled/disabled 상태 갱신 |
| `DELETE` | `/api/notification-devices/{deviceId}` | logout, uninstall signal, invalid token 처리에 따른 token/device 비활성화 |

Request 후보:

```json
{
  "platform": "ANDROID",
  "fcm_token": "fcm-token",
  "device_id": "client-generated-or-installation-id",
  "app_version": "1.0.0"
}
```

Policy:

- 서버는 현재 인증 사용자(JWT `sub = member.uuid`)의 token/device만 등록하거나 갱신한다. `email`이나 DB 내부 Long `member.id`를 routing identity로 사용하지 않는다.
- invalid token, token refresh, deactivate는 notification device/token 상태만 변경한다. crew lifecycle, certification, moderation, settlement, point ledger/history를 변경하지 않는다.
- FCM token은 delivery credential에 가까운 민감 데이터로 취급하고, public response에서 불필요하게 재노출하지 않는다.

### Notification inbox/list/read 후보

| Method | Candidate path | 역할 |
| ------ | -------------- | ---- |
| `GET` | `/api/notifications` | 내 알림 UX hint 목록 조회 |
| `GET` | `/api/notifications/unread-count` | badge 표시용 미읽음 count 조회 |
| `PATCH` | `/api/notifications/{notificationId}/read` | 단건 읽음 처리 |
| `PATCH` | `/api/notifications/read-all` | 전체 읽음 처리 후보 |

Response item 후보:

```json
{
  "notification_id": "uuid-or-id",
  "event_type": "MISSION_LOG_VERIFICATION_RESULT",
  "resource_type": "mission_log",
  "resource_id": "1201",
  "deep_link": "dondok://crews/42/mission-logs/1201",
  "occurred_at": "2026-05-13T07:31:08+09:00",
  "display_text": "인증 결과가 반영되었습니다.",
  "requires_refetch": true,
  "read_at": null
}
```

Field policy:

| 필드 | 설명 |
| ---- | ---- |
| `notification_id` | inbox/read UX 상태 처리를 위한 알림 식별자. 도메인 aggregate id가 아니다 |
| `event_type` | 클라이언트 반응을 결정하는 notification event catalog 후보. DB strict enum freeze가 아니다 |
| `resource_type` | refetch 대상 canonical resource 종류 |
| `resource_id` | refetch/deep-link route에 사용할 resource 식별자 |
| `deep_link` | 클릭 후 이동할 client route. 이동 직후 canonical API refetch가 필요하다 |
| `occurred_at` | 알림 대상 product event 발생 시각 후보. 정산/원장 발생 시각 source of truth가 아니다 |
| `display_text` | 사용자 표시 문구. 클라이언트 분기 조건이나 payout/certification proof로 사용하지 않는다 |
| `requires_refetch` | MVP에서는 항상 `true`로 취급한다 |
| `read_at` | UX read state. audit history나 미해결 domain task 상태가 아니다 |

Click/refetch contract:

- 사용자가 push 또는 inbox item을 클릭하면 클라이언트는 `deep_link`로 이동한 뒤 `resource_type`/`resource_id`에 맞는 canonical REST API를 다시 조회한다.
- stale, duplicate, out-of-order, missed notification이 있어도 화면 표시와 domain action 가능 여부는 refetched canonical API state가 결정한다.
- payload/list item에 authoritative payout snapshot, authoritative certification snapshot, ledger truth, settlement retry/replay/correction directive를 넣지 않는다.
- notification read/unread는 badge와 목록 정리에만 쓰며 unresolved settlement/certification/moderation/ledger task로 표시하지 않는다.

### Daily mission result notification projection

`MISSION_LOG_VERIFICATION_RESULT`, `DASHBOARD_PROJECTION_UPDATED` 등 일일 결과 알림 문구 후보:

> `[크루명][날짜] 현재 인증 결과가 반영되었습니다. 현재 기준 예상 환급금은 [금액]도딘이며 최종 정산 전까지 변동될 수 있습니다.`

- 문구에 인터폴레이트되는 `crewTitle`, `missionDate`, `certificationStatus`, `successMemberCount`, `failedMemberCount`, `pendingMemberCount`, `expectedRefundAmount`, `expectedRefundDelta`는 모두 알림 생성 시점 projection이며 저장 컬럼이 아니다.
- `crewTitle`은 `crew.title` 현재 값이고, `missionDate`는 `Asia/Seoul` 기준 KST date다.
- `certificationStatus`는 알림 대상 `MissionLog`의 resolved `certification_status`(`PENDING_REVIEW`/`SUCCESS`/`FAILED`)다.
- `successMemberCount`, `failedMemberCount`, `pendingMemberCount`는 해당 `missionDate`에 대한 `mission_log` × `crew_participant` projection이며 dashboard projection과 동일 source/current-basis 규칙을 따른다.
- `expectedRefundAmount`는 알림 대상 참여자의 `dashboard.my_expected_refund_amount`와 동일한 current-basis projection이다. `expectedRefundDelta`는 직전 알림 발송 시점 대비 변화 hint이며 ledger/settlement authority가 아니다.
- 알림은 hint/deep-link이고 canonical state가 아니다. 알림 payload는 stale일 수 있고, 알림 클릭/진입 시 클라이언트는 `deep_link`로 이동한 뒤 dashboard / settlement / mission-log canonical API를 refetch해야 한다.
- 알림 발송 성공/실패는 `certification_status`, `Settlement.status`, `point_history` 상태를 변경하지 않는다.

### Delivery attempt observability 후보

- `notification_delivery_attempt` 또는 동등한 내부 log는 FCM send attempt, provider response, invalid token, bounded transport retry 관측용 후보다.
- delivery attempt failure는 notification transport failure일 뿐 domain failure가 아니다. settlement retry, replay, correction, payout mutation trigger로 사용하지 않는다.
- 동일 event에 대한 중복 알림은 클라이언트와 서버 모두 idempotent하게 처리한다. 중복 알림이 중복 정산/중복 ledger entry를 만들면 안 된다.

### Event taxonomy 후보

| `event_type` 후보 | 설명 | Refetch target 예시 |
| ----------------- | ---- | ------------------ |
| `CREW_APPLICATION_CREATED` | 가입 신청 발생 | crew applications / host review API |
| `CREW_APPLICATION_DECIDED` | 가입 승인/거절 결과 | crew participant/application API |
| `CREW_NOTICE_CREATED` | 새 공지/댓글 등 engagement re-entry | crew notice API |
| `MISSION_CERTIFICATION_DUE_SOON` | 인증 마감 reminder | crew dashboard / mission logs API |
| `MISSION_LOG_UPLOADED` | 방장 검수 대상 인증 업로드 | mission log review API |
| `MISSION_LOG_VERIFICATION_RESULT` | 인증 결과 반영 hint | mission log detail / dashboard API |
| `DASHBOARD_PROJECTION_UPDATED` | 현재 기준 projection 요약 변화 | crew dashboard API |
| `SETTLEMENT_RESULT_READY` | 정산 결과 조회 가능 | crew settlement / settlement detail API |
| `POINT_HISTORY_UPDATED` | 포인트 내역 반영 hint | points/history API |
| `REACTION_CREATED` | 리액션 engagement hint | feed / mission log API |

이 taxonomy는 notification routing 후보이며 DB enum이나 authoritative audit event catalog가 아니다. 새 event는 canonical REST API로 refetch 가능한 product event에만 추가한다.

### `GET /api/notifications/stream` — Phase 2/deferred SSE drift 후보

역할:

- 이 endpoint는 기존 문서/구현 흔적을 보존하기 위한 Phase 2/deferred realtime 후보이며 Android-first FCM MVP의 authoritative notification contract가 아니다.
- 도입하더라도 best-effort realtime UX delivery만 제공하고, notification inbox/read, replay cursor, durable delivery, cross-device unread sync를 보장하지 않는다.
- Public API contract는 "현재 인증 사용자 stream"이다. 서버는 JWT `sub = member.uuid`로 인증 사용자를 식별하고 해당 사용자 대상 이벤트만 전달한다.
- `email`은 변경 가능하고 PII이므로 SSE routing identity, stream identifier, notification recipient key로 사용하지 않는다.

Request:

```http
GET /api/notifications/stream
Authorization: Bearer {accessToken}
Accept: text/event-stream
```

SSE payload policy:

- SSE payload에는 email, Long `member.id`, 불필요한 사용자 PII를 넣지 않는다.
- SSE payload는 상태 snapshot이 아니라 REST refetch/invalidate를 유도하는 signal이다.
- SSE `eventType`, `resourceType`, `resourceId`, message/ui hint가 있더라도 canonical 화면 상태는 REST API refetch 결과가 결정한다.
- SSE 연결 실패는 핵심 도메인 transaction 실패로 해석하지 않는다.

Reconnect / delivery semantics:

- SSE delivery는 best-effort realtime UX delivery다.
- 서버 재시작, 네트워크 단절, 브라우저 재연결 중 이벤트가 누락될 수 있다.
- missed event 복구는 settlement replay가 아니라 기존 REST API 화면 재조회로 처리한다.
- stale/duplicate/out-of-order event가 있어도 reconnect 또는 화면 진입 시 authoritative REST state가 우선한다.
- 이벤트 순서, durable delivery, cross-device unread state를 보장하지 않는다.

## 6. 상태 흐름 다이어그램

### 6.1 Crew

```text
RECRUITING --system activation at start_at / activated_at = start_at--> ACTIVE -> CLOSED
RECRUITING --start_at eligibility failure cancellation batch--> CANCELLED
```

### 6.2 Participant

```text
PENDING --host approve + reserve 확정--> LOCKED
PENDING --user cancel (DELETE /participants/me)--> CANCELLED
PENDING --host reject--> REJECTED
PENDING --시작 전까지 처리 안 됨--> EXPIRED
WITHDRAWN / ACTIVE withdrawal: brownfield-deferred, not MVP active baseline authority
```

### 6.3 Settlement

```text
NONE (API projection only)
-> PENDING
-> RUNNING
-> SUCCEEDED

RUNNING
-> RETRY_WAIT
-> RUNNING
-> SUCCEEDED or FAILED
```

## 7. FE에서 바로 써야 하는 필드 설명

- 방 화면은 `status`, `frequency_type`, `frequency_count`, `mission_schedule_days`, `deposit_amount`, `my_participation`을 기준으로 버튼 상태를 결정한다. `my_participation.status`(`PENDING`/`LOCKED`/`REJECTED`/`CANCELLED`/`EXPIRED`)에 따라 가입 신청/취소/대기 UX를 분기한다.
- 계정/포인트 요약 화면은 `GET /api/points.available_balance`, `GET /api/points.reserved_balance`, `GET /api/points.active_locked_amount`, `GET /api/points.settlement_pending_amount`, `GET /api/points.locked_balance`, `GET /api/points.total_balance`를 기준으로 표시한다.
- FE는 여러 방의 `my_participation.deposit_reserved_amount` / `deposit_locked_amount`를 직접 합산해 계정 단위 reserve/lock 잔액을 만들지 않고, `GET /api/points`의 projection 필드를 표시한다.
- `my_participation.deposit_reserved_amount`와 `my_participation.deposit_locked_amount`는 방 상세의 해당 참여 보증금 표시용 필드이며, 계정 단위 `reserved_balance`, `locked_balance`, `total_balance`의 source of truth가 아니다.
- `reserved_balance`, `locked_balance`, `total_balance`는 UX 표시용이며, 출금 가능 여부, 환급 가능 여부, 분쟁 처리, 정산 결과 판단 기준으로 사용하면 안 된다.
- 탈퇴 버튼/withdrawal UX는 MVP active contract가 아니라 brownfield-deferred로 취급한다. 노출하더라도 frozen baseline, final settlement, point ledger를 직접 변경한다고 안내하면 안 된다.
- FE는 `certification_status`를 인증 요청의 resolved certification state로만 사용해야 하고, 최종 정산 인정 여부 판단 기준으로 사용하면 안 된다.
- 피드 화면에서 `feed_items[]`는 성공 인증 게시물이고, `day_statuses[]` / `participant_day_slots[]`는 `SUCCESS`, `FAILED`, `NOT_SUBMITTED` 표시용 projection이다. 둘을 정산 결과나 포인트 원장으로 해석하면 안 된다.
- 리액션은 `mission_log_reaction` 기반 social metadata이며, `reaction_counts`는 파생값이다. FE는 리액션이 인증 성공 여부, 정산 인정, 환급, 포인트, AI 리포트 상태를 바꾼다고 표시하면 안 된다.
- 인증 제출 직후에는 `certification_status`와 `failure_reason`만 신뢰한다. 최종 인정 횟수는 정산 전까지 확정되지 않는다.
- 인증 직후에는 성공으로 표시할 수 있지만, 최종 결과 화면의 인정/미인정 표시는 정산 결과 기준으로 별도 표시해야 한다.
- 인증 기록 화면과 정산 결과 화면은 서로 다른 기준을 사용해야 한다.
- 인증 기록 화면은 `certification_status` 기준으로 `검수 대기/인증 성공/인증 실패`만 표시한다.
- 정산 결과 화면은 `settlement_item.calculation_reason` 기준으로 `최종 인정/미인정`을 표시한다.
- 두 기준을 혼용하면 잘못된 UX가 발생하므로 반드시 분리해서 사용해야 한다.
- 최종 정산 인정 시각 판단은 `server_time` 기준이며, `exif_taken_at`은 촬영 시각 검증용 보조 정보로만 사용해야 한다.
- `failure_reason = null`이어도 최종 정산에서 제외될 수 있으므로, `DAILY` 중복이나 `SPECIFIC_DAYS` 제외 여부는 `settlement_item.calculation_reason`이 포함된 정산 결과 화면에서 확인해야 한다.
- 정산 결과 화면은 먼저 `GET /api/crews/{crewId}/settlement`를 polling하고, `status = SUCCEEDED`가 되면 `settlement_id`로 `GET /api/settlements/{settlementId}`를 호출한다.
- 포인트 내역 화면은 `transaction_type` 그대로 내려받고, UI에서 `POINT_CHARGE`, `CREW_DEPOSIT_LOCK`, `CREW_SETTLEMENT_REFUND`, `CREW_CANCELLED_REFUND`를 한국어 라벨로 매핑한다.
- 포인트 내역 화면의 `next_cursor`는 UI가 직접 해석하지 말고 다음 요청에 그대로 전달해야 한다.
- `Settlement.status = SUCCEEDED` 전에는 일부 `point_history_id`가 비어 있을 수 있으므로, 정산 상세의 item 금액과 포인트 내역 표시 시 상태를 함께 봐야 한다.
- Dashboard 화면의 금액, 비율, 순위는 “예상”, “현재 기준”, “추정” 라벨로 표시하고 “확정”, “최종”, “정산 완료” 라벨은 Settlement API 결과에만 사용한다.
- Dashboard의 `my_expected_refund_amount`와 `GET /api/points`의 `locked_balance`, `available_balance`, `total_balance`를 조합해 최종 보유 포인트, 출금 가능 금액, 최종 정산 후 확정되는 환급금을 계산하면 안 된다.
- Dashboard의 `projection_status = SETTLEMENT_SUCCEEDED`이면 `settlement_id`로 `GET /api/settlements/{settlementId}`를 호출해 최종 인정 성공 수, 최종 지분율, 최종 환급금을 표시한다.
- Dashboard projection과 최종 settlement 결과가 달라도 시스템 오류로 간주하지 않고, 차이 설명은 Settlement API의 `settlement_item.calculation_reason`을 기준으로 한다.


### Notification click/refetch UX handling

Android FCM push, notification inbox item, deferred SSE signal 모두 `deep_link` 또는 refetch target으로 이동한 뒤 canonical REST API state를 다시 조회한다. 알림 payload/list item/read state는 상태 snapshot이나 canonical state authority가 아니다. `GET /api/notifications/stream`은 Phase 2/deferred drift 후보로만 유지한다.

## 8. 구현 메모

- 인증 시점에는 과도한 분산 락보다 `MissionLog append-only 저장 + 캐시 원자 연산 + 최종 계산/replay 가능성`을 우선한다.
- `SUCCEEDED` 전 최종 정산 계산은 `MissionLog.server_time`, frozen `LOCKED` participant baseline, resolved certification state를 기준으로 수행한다.
- `exif_taken_at`은 서버가 S3 object에서 추출/검증한 이미지 조작 또는 촬영 시각 이상 여부 검증 보조 정보이며, 인정 횟수 계산 기준 시간으로 사용하지 않는다.
- 정산 시점에는 DB 조건부 `Settlement(PENDING/RETRY_WAIT -> RUNNING)` claim을 1차 기준으로 사용하고, Redisson crew lock은 보조 수단, DB unique 제약과 `point_history.idempotency_key`는 최종 방어선으로 사용한다.
- `total_locked_amount`는 정산 시점의 `crew_participant.deposit_amount` 합계 스냅샷이며, `point_account`나 `point_history`를 다시 합산하지 않는다.
- 일반 정산 remainder는 deterministic remainder allocation rule로 처리한다. 이는 host reward/authority/privilege, winner 지급, draw/random 지급이 아니라 replayable floor-remainder 규칙이다.
- 취소형 정산에서도 조회 구조는 동일하고 `settlement_type = CANCELLED_BEFORE_START`만 달라진다.
- 포인트 원장 기록과 `point_account.balance` 갱신은 participant 지급 단위로 같은 트랜잭션에서 처리하되, 전체 정산은 partial 복구가 가능하도록 이미 생성된 원장을 idempotency key로 재사용한다.
- Replay/retry는 finality를 약화하지 않는다. Replay는 historical semantic truth reconstruction이고, retry는 missing item/linkage completion이다. 둘 다 succeeded settlement snapshot이나 point ledger overwrite 권한이 아니다.
- Scheduler delay는 운영 감사 fact다. API/배치는 delayed execution을 기록할 수 있지만 contract lifecycle authority는 scheduled semantic anchor 기준이다.
- Moderation persistence는 authoritative transition ledger와 non-authoritative operational context를 분리한다. Human memo/support note/UX wording은 REST/API 정산 truth가 아니다.
- Phase 2 hardening registry: audit-grade notification durability, notification preference matrix, notification template CMS/table, SSE/Web realtime reliability, campaign/broadcast system, advanced notification analytics, full AI replay reproducibility, immutable event sourcing migration, provider-level AI determinism, distributed replay engine, full provenance governance는 MVP API active contract가 아니다.

## 9. 확정 메모

- Withdrawal/탈퇴 semantics는 brownfield/deferred다. MVP active contract에서는 frozen `LOCKED` baseline, final settlement, point ledger를 직접 변경하지 않는다.
- MVP 인증 API의 `mission_log.failure_reason`은 인증 시점 실패 사유만 표현하고, `OUT_OF_SCHEDULE`는 사용하지 않는다.
- 관리자 정산 재시도는 `crewId`가 아니라 `settlementId` 기준으로 수행한다.
