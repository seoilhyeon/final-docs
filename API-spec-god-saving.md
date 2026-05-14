# API 명세: 갓세이빙 MVP

기준 문서:

- [PRD-god-saving.md](/Users/ilhyeon/Documents/projects/god-saving/docs/PRD-god-saving.md)
- [Settlement-design.md](/Users/ilhyeon/Documents/projects/god-saving/docs/Settlement-design.md)
- [ERD-god-saving.md](/Users/ilhyeon/Documents/projects/god-saving/docs/ERD-god-saving.md)

## 1. 목적

이 문서는 갓세이빙 MVP에서 FE와 BE가 병렬 개발할 수 있도록 API 계약을 고정하기 위한 문서다.

고정 원칙:

- API는 `REST + JSON` 기준으로 설계한다.
- 비즈니스 시간대는 `Asia/Seoul`이고, API 시각 값은 timezone offset이 포함된 `ISO-8601` 문자열로 주고받는다.
- 금액은 모두 `integer` 원 단위다.
- 보증금은 별도 자산 이동이 아니라 `lock` 모델이다.
- `Settlement.status = SUCCEEDED` 전 정산 계산 입력은 `MissionLog`와 참여자 상태 재계산 결과다.
- `Settlement.status = SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item` 계산 스냅샷과 연결된 `point_history` 원장이다. 이후 `MissionLog` 재계산은 감사/디버깅용 검증에만 사용한다.
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

- `MissionRoom.status`는 방 상태다. MVP에서 `RECRUITING -> ACTIVE`는 host `StartRoom` command 성공으로만 발생한다.
- `Participant.status`는 참여 상태다.
- `Settlement.status`는 정산 처리 상태의 원천이다.
- `NONE`은 `Settlement` row가 아직 없는 상태를 보여주기 위한 API 응답용 값이다. DB `settlement.status`에는 저장하지 않는다.
- `PENDING`은 `Settlement` row가 생성됐지만 아직 claim되지 않은 실행 전 상태다.
- partial 지급/연결 상태는 복구 가능한 중간 상태이며 `SUCCEEDED`로 노출하지 않고 `RETRY_WAIT` 또는 `FAILED`로 남긴다.

## 3. 도메인 상태 / Enum

### 3.1 RoomStatus

- `RECRUITING`: 모집 중. `recruitment_deadline` 전에는 신규 참여 가능, 이후에는 신규 참여 불가.
- `ACTIVE`: host `StartRoom` command가 성공해 `activated_at`이 기록된 진행 중 상태.
- `CLOSED`: 계획된 `end_at` 이후 정상 종료 상태.
- `CANCELLED`: 시작 전 취소 상태. `start_at`까지 미시작이면 batch가 취소형 정산 대상으로 전이할 수 있다.

### 3.2 RoomVisibility

- `PUBLIC`
- `PRIVATE`

### 3.3 ParticipantStatus

- `JOINED`
- `WITHDRAWN`

### 3.4 FrequencyType

- `DAILY`
- `SPECIFIC_DAYS`
- `WEEKLY_N`

### 3.5 SettlementType

- `NORMAL`
- `CANCELLED_BEFORE_START`

### 3.6 SettlementStatus

- `NONE` - API projection only
- `PENDING`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `RETRY_WAIT`

### 3.7 PointTransactionType

- `POINT_CHARGE`
- `ROOM_DEPOSIT_LOCK`
- `ROOM_SETTLEMENT_REFUND`
- `ROOM_CANCELLED_REFUND`

### 3.8 MissionLogFailureReason

- `EXIF_MISSING`
- `EXIF_TIME_INVALID`
- `BEFORE_START`
- `AFTER_END`
- `AFTER_WITHDRAWN`

### 3.9 SettlementFailureCode

- `INPUT_LOAD_FAILED`
- `CALCULATION_FAILED`
- `POINT_CREDIT_FAILED`
- `DUPLICATE_SETTLEMENT`
- `LOCK_ACQUIRE_FAILED`
- `UNKNOWN`

### 3.10 AiHabitReportStatus

- `PENDING`
- `SUCCEEDED`
- `FAILED`

### 3.11 AiHabitReportFailureCode

`ai_habit_report.failure_code`의 MVP catalog다.

이 값들은 API/FE/QA의 discoverability를 위한 문서화 목적이며, strict DB enum이나 상세 retry taxonomy를 의미하지 않는다.

- `AI_REPORT_FAILED`
- `AI_RESPONSE_INVALID`
- `UNKNOWN`

### 3.12 ProjectionStatus

대시보드 projection 응답 전용 상태값이다.

이 값들은 DB에 저장되는 enum이 아니며, lifecycle 또는 settlement source-of-truth로 사용하지 않는다.

- `NOT_STARTED`
- `LIVE`
- `FROZEN`
- `NOT_PROVIDED`
- `SETTLEMENT_SUCCEEDED`

### 3.13 ProjectionNotice

대시보드 projection 응답의 현재 상태를 설명하기 위한 안내 값이다.

이 값들은 DB에 저장되지 않으며, projection 응답의 보조 설명 용도로만 사용한다.

- `ESTIMATED_NOT_FINAL`
- `NOT_STARTED`
- `NOT_PROVIDED`
- `SETTLEMENT_RESULT_AVAILABLE`
- `INSUFFICIENT_PROJECTION_INPUT`

### 3.14 PointHistoryReferenceType

- `POINT_CHARGE`
- `ROOM_PARTICIPANT`
- `SETTLEMENT_ITEM`

### 3.15 MissionLogReactionType

- `CHEER`
- `CLAP`
- `FIRE`

리액션 enum은 소셜 메타데이터 전용이다. 포인트 원장, 정산, 환급, AI 리포트, 상태 생명주기 enum과 연결하지 않는다.

## 4. API 목록

| 도메인      | Method   | Path                                            | 설명                                     |
| ----------- | -------- | ----------------------------------------------- | ---------------------------------------- |
| 인증/회원   | `POST`   | `/api/auth/signup`                              | 회원가입                                 |
| 인증/회원   | `POST`   | `/api/auth/login`                               | 로그인                                   |
| 인증/회원   | `POST`   | `/api/auth/refresh`                             | access token 재발급                      |
| 인증/회원   | `POST`   | `/api/auth/logout`                              | 로그아웃                                 |
| 인증/회원   | `GET`    | `/api/me`                                       | 내 계정/프로필 조회                      |
| 인증/회원   | `PATCH`  | `/api/me/profile`                               | 내 최소 프로필 수정                      |
| 크루/참여   | `GET`    | `/api/rooms`                                    | 공개 방 목록 조회                        |
| 크루/참여   | `POST`   | `/api/rooms`                                    | 방 생성                                  |
| 크루/참여   | `GET`    | `/api/rooms/{roomId}`                           | 방 상세 조회                             |
| 크루/참여   | `GET`    | `/api/rooms/join-code/{joinCode}`               | 참여 코드로 방 조회                      |
| 크루/참여   | `POST`   | `/api/rooms/{roomId}/participants`              | 방 참여 및 보증금 lock                   |
| 크루/참여   | `POST`   | `/api/rooms/{roomId}/withdraw`                  | 방 탈퇴                                  |
| 크루/참여   | `POST`   | `/api/rooms/{roomId}/start`                     | host 수동 미션 시작                      |
| 미션 인증   | `POST`   | `/api/mission-logs`                             | 인증 제출                                |
| 미션 인증   | `GET`    | `/api/rooms/{roomId}/mission-logs/me`           | 내 인증 기록 조회                        |
| 피드/리액션 | `GET`    | `/api/rooms/{roomId}/feed`                      | 방 인증 피드와 파생 일자 상태 조회       |
| 대시보드    | `GET`    | `/api/rooms/{roomId}/dashboard`                 | 방 내 성과/보상 실시간 추정 projection 조회 |
| 피드/리액션 | `POST`   | `/api/mission-logs/{missionLogId}/reactions`    | 내 리액션 멱등 upsert                    |
| 피드/리액션 | `DELETE` | `/api/mission-logs/{missionLogId}/reactions/me` | 내 리액션 멱등 삭제                      |
| 정산        | `GET`    | `/api/rooms/{roomId}/settlement`                | 방 기준 정산 상태/요약 조회              |
| 정산        | `GET`    | `/api/settlements/{settlementId}`               | 정산 결과 상세 조회                      |
| 정산        | `GET`    | `/api/admin/settlements`                        | 관리자 정산 실패/대기 목록 조회          |
| 정산        | `POST`   | `/api/admin/settlements/{settlementId}/retry`   | 관리자 정산 재시도                       |
| AI          | `POST`   | `/api/ai/mission-recommendations`               | AI 미션 추천 초안 생성                   |
| AI          | `POST`   | `/api/rooms/{roomId}/ai-habit-report`           | 정산 완료 후 내 AI 습관 리포트 생성/조회 |
| AI          | `GET`    | `/api/rooms/{roomId}/ai-habit-report/me`        | 내 AI 습관 리포트 상태/결과 조회         |
| AI          | `GET`    | `/api/ai-habit-reports/{reportId}`              | AI 습관 리포트 단건 조회                 |
| 알림        | `GET`    | `/api/notifications/stream`                  | 내 실시간 알림 SSE stream 구독             |
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
  "nickname": "갓세이빙러",
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
    "nickname": "갓세이빙러",
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
  "nickname": "갓세이빙러",
  "profile_image_url": "https://cdn.example.com/profile/018f4fd2-6d7a-7a41-9f58-6d07f5c3c901/avatar.jpg",
  "status": "ACTIVE",
  "created_at": "2026-05-01T12:00:00+09:00"
}
```

정책:

- 프로필은 `nickname`과 `profile_image_url`만 노출한다.
- `profile_image_url`은 저장된 `member.profile_image_s3_key`에서 파생한 접근 URL이며, 이미지가 없으면 `null`일 수 있다.
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

Response `200 OK`:

```json
{
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "email": "user@example.com",
  "nickname": "saving-cat",
  "profile_image_url": "https://cdn.example.com/profile/018f4fd2-6d7a-7a41-9f58-6d07f5c3c901/avatar.jpg",
  "status": "ACTIVE",
  "updated_at": "2026-05-01T12:10:00+09:00"
}
```

Error:

- `VALIDATION_ERROR`
- `PROFILE_IMAGE_NOT_FOUND`

정책:

- 요청에는 `nickname` 또는 `profile_image_s3_key` 중 하나 이상이 있어야 한다.
- 프로필 이미지는 별도 presigned upload 흐름으로 먼저 업로드된 S3 key만 참조한다.
- 수정 범위는 닉네임과 프로필 이미지로 한정한다.
- 인증/session 로직에 영향을 주지 않는다.
- 정산/포인트 시스템에 영향을 주지 않는다.
- 환급, 상태 생명주기는 변경하지 않는다.
- 소셜 프로필 기능은 이 API에 포함하지 않는다.

## 5.2 크루(방) / 참여

### `GET /api/rooms`

역할:

- 공개 모집 방 목록을 조회한다.

Query:

| 필드         | 타입     | 필수 | 설명                |
| ------------ | -------- | ---- | ------------------- |
| `status`     | `string` | N    | 기본값 `RECRUITING` |
| `visibility` | `string` | N    | 기본값 `PUBLIC`     |

Response `200 OK`:

```json
{
  "items": [
    {
      "room_id": 42,
      "title": "새벽 기상 챌린지",
      "visibility": "PUBLIC",
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

- MVP 목록 API는 공개 방만 대상으로 시작한다.
- 참여자 수 같은 집계 필드는 본 명세의 필수 응답에 포함하지 않는다.

### `POST /api/rooms`

역할:

- 크루 방과 인증 규칙을 생성한다.

Request:

| 필드                    | 타입       | 필수 | 설명                                                    |
| ----------------------- | ---------- | ---- | ------------------------------------------------------- |
| `title`                 | `string`   | Y    | 방 제목                                                 |
| `description`           | `string`   | N    | 방 설명                                                 |
| `visibility`            | `string`   | Y    | `PUBLIC` / `PRIVATE`                                    |
| `deposit_amount`        | `integer`  | Y    | 기본 보증금                                             |
| `min_participants`      | `integer`  | N    | 기본값 `2`                                              |
| `max_participants`      | `integer`  | Y    | 최대 인원                                               |
| `frequency_type`        | `string`   | Y    | `DAILY` / `SPECIFIC_DAYS` / `WEEKLY_N`                  |
| `frequency_count`       | `integer`  | N    | `WEEKLY_N`일 때 필수                                    |
| `mission_schedule_days` | `string[]` | N    | `SPECIFIC_DAYS`일 때 필수. 예: `["MONDAY","WEDNESDAY"]` |
| `recruitment_deadline`  | `string`   | Y    | ISO-8601. 신규 참여 마감 시각                           |
| `start_date`            | `string`   | Y    | `YYYY-MM-DD`. 예정 시작일                               |
| `end_date`              | `string`   | Y    | `YYYY-MM-DD`. 계획된 종료일                             |

Response `201 Created`:

```json
{
  "room_id": 42,
  "title": "새벽 기상 챌린지",
  "visibility": "PRIVATE",
  "join_code": "A1B2C3",
  "status": "RECRUITING",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "SPECIFIC_DAYS",
  "frequency_count": null,
  "mission_schedule_days": ["MONDAY", "WEDNESDAY", "FRIDAY"],
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

정책:

- `deposit_amount`는 `1,000원 ~ 1,000,000원`, `1,000원 단위`를 만족해야 한다.
- `min_participants`는 기본값 `2`고, `2 <= min_participants <= max_participants <= 10`을 만족해야 한다.
- `SPECIFIC_DAYS`는 특정 날짜가 아니라 반복 요일 규칙이며 `mission_schedule_day` 원본으로 저장한다.
- `recruitment_deadline`은 신규 참여 마감 시각이며 activation/settlement 기준이 아니다.
- `start_date`, `end_date`는 서버에서 `Asia/Seoul` 기준 `start_at`, `end_at`으로 정규화한다. `start_at`은 예정 시작 및 MVP 수동 시작 가능 만료 시각이고, 실제 ACTIVE 전이 시각은 `activated_at`이다.
- `end_at`은 계획된 미션 종료 cutoff이며 activation 지연으로 자동 이동하지 않는다.

### `GET /api/rooms/{roomId}`

역할:

- 방 상세와 내 참여 상태를 조회한다.

Response `200 OK`:

```json
{
  "room_id": 42,
  "host_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "title": "새벽 기상 챌린지",
  "description": "매일 아침 6시 전에 인증",
  "visibility": "PUBLIC",
  "status": "ACTIVE",
  "settlement_status": "NONE",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "DAILY",
  "frequency_count": null,
  "mission_schedule_days": [],
  "recruitment_deadline": "2026-05-09T23:59:59+09:00",
  "start_at": "2026-05-10T00:00:00+09:00",
  "activated_at": "2026-05-09T23:30:00+09:00",
  "end_at": "2026-05-31T23:59:59+09:00",
  "my_participation": {
    "participant_id": 101,
    "status": "JOINED",
    "deposit_locked_amount": 100000,
    "joined_at": "2026-05-08T13:00:00+09:00",
    "withdrawn_at": null
  }
}
```

Error:

- `ROOM_NOT_FOUND`

정책:

- `settlement_status`는 조회 최적화용 projection이다.
- 정산 처리의 원천 상태는 `Settlement.status`다.
- `my_participation`이 없으면 아직 참여하지 않은 회원이다.

### `GET /api/rooms/join-code/{joinCode}`

역할:

- 비공개 참여 코드를 방 조회용으로 변환한다.

Response `200 OK`:

```json
{
  "room_id": 42,
  "title": "새벽 기상 챌린지",
  "visibility": "PRIVATE",
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
```

Error:

- `INVALID_JOIN_CODE`
- `ROOM_NOT_FOUND`

### `POST /api/rooms/{roomId}/participants`

역할:

- 방 참여를 생성하고 보증금을 lock한다.

Request:

- body 없음

Response `201 Created`:

```json
{
  "participant_id": 101,
  "room_id": 42,
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
  "status": "JOINED",
  "deposit_locked_amount": 100000,
  "joined_at": "2026-05-08T13:00:00+09:00"
}
```

Error:

- `ROOM_NOT_FOUND`
- `ROOM_NOT_RECRUITING`
- `CAPACITY_FULL`
- `ALREADY_JOINED`
- `INSUFFICIENT_BALANCE`

정책:

- 신규 참여는 `RECRUITING` 상태이면서 서버 시간이 `recruitment_deadline` 전일 때만 허용한다.
- `recruitment_deadline` 이후에는 `ROOM_RECRUITMENT_CLOSED` 또는 `ROOM_NOT_RECRUITING` 계열 오류로 거절한다.
- 같은 `member`는 같은 방에 하나의 `participant`만 가질 수 있다.
- 참여 처리에서는 아래 세 단계가 하나의 트랜잭션으로 함께 성공하거나 함께 롤백되어야 한다.
  - `point_account.balance` 조건부 차감
  - `room_participant` 생성
  - `ROOM_DEPOSIT_LOCK point_history` 생성
- 잔액 차감은 반드시 `WHERE balance >= deposit_amount` 조건부 update로 수행하고, row count가 `1`일 때만 성공으로 간주한다.
- `INSUFFICIENT_BALANCE`는 잔액 부족뿐 아니라 동시 요청으로 조건부 update row count가 `0`이 된 경우도 포함한다.

### `POST /api/rooms/{roomId}/start`

역할:

- host가 모집 중인 방을 수동으로 시작한다.
- MVP에서 이 command의 성공 transaction만 `RECRUITING -> ACTIVE` 전이를 만들 수 있다.

Request:

- body 없음

Response `200 OK` 또는 `204 No Content`:

```json
{
  "room_id": 42,
  "status": "ACTIVE",
  "min_participants": 2,
  "current_participant_count": 5,
  "start_at": "2026-05-10T00:00:00+09:00",
  "activated_at": "2026-05-09T23:30:00+09:00"
}
```

Error:

- `ROOM_NOT_FOUND`
- `ROOM_START_FORBIDDEN`
- `ROOM_NOT_RECRUITING`
- `MIN_PARTICIPANTS_NOT_MET`
- `ROOM_START_EXPIRED`
- `CONFLICT`

정책:

- caller는 해당 방의 host여야 한다.
- room은 `RECRUITING`이어야 한다. 단 이미 `ACTIVE`인 방에 대한 중복 요청은 idempotent success/no-op으로 처리한다.
- 서버 시간이 `start_at`을 넘으면 `ROOM_START_EXPIRED` 또는 이미 batch가 취소한 경우 terminal-state conflict로 응답한다.
- command 실행 시점에 eligible participant 수가 `min_participants` 이상인지 재검증한다.
- 성공 transaction은 `status = ACTIVE`, `activated_at = server_now`를 함께 기록한다. 이 정책상 성공한 activation은 `activated_at <= start_at`을 만족하며, 이는 별도 DB constraint가 아니라 StartRoom 만료 검증과 시작 만료 취소 batch에서 파생되는 MVP invariant다.
- 동시에 여러 start 요청이 오면 하나의 조건부 전이만 성공한다. loser는 최종 room 상태를 재조회해 deterministic response를 반환한다.
- `CANCELLED`/`CLOSED` 같은 terminal 상태에는 `CONFLICT` 또는 `ROOM_NOT_RECRUITING` 계열 오류로 응답한다.
- `activated_at` 이후 post-activation 인증, 정산, projection/log eligibility는 `room.start_at`이 아니라 `room.activated_at`을 기준으로 한다.

### `POST /api/rooms/{roomId}/withdraw`

역할:

- 내 참여 상태를 `WITHDRAWN`으로 전환한다.

Request:

- body 없음

Response `200 OK`:

```json
{
  "participant_id": 101,
  "room_id": 42,
  "status": "WITHDRAWN",
  "withdrawn_at": "2026-05-20T10:00:00+09:00",
  "deposit_locked_amount": 100000
}
```

Error:

- `ROOM_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `WITHDRAW_NOT_ALLOWED`

정책:

- 탈퇴는 `RECRUITING`, `ACTIVE` 상태 모두에서 허용한다.
- `RECRUITING` 상태 탈퇴는 참여 취소로 간주하지만, `participant.status`는 `WITHDRAWN`으로 유지한다.
- `ACTIVE` 상태 탈퇴는 중도 탈퇴로 간주하고, 정산에서는 `withdrawn_at` 이전 성공만 인정한다.
- 중도 탈퇴자와 모집 중 탈퇴자 모두 즉시 환급되지 않는다. 보증금은 최종 정산 또는 방 취소 시 환급된다.
- 탈퇴 후 인증은 차단되고, 정산에서는 `withdrawn_at` 이전 성공만 인정한다.
- MVP에서는 탈퇴 후 동일 방 재참여를 지원하지 않는다.
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
| `room_id` | `integer` | N | mission image 업로드 시 대상 방 |
| `participant_id` | `integer` | N | mission image 업로드 시 대상 참여자 |
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
- mission 인증 이미지의 권장 key 형식은 `mission/{roomId}/{participantId}/{uuid}`다.
- presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- 서버는 발급 시점에 사용자, room, participant 권한을 검증한다.
- 클라이언트는 발급받은 URL로 S3에 직접 업로드한다.
- 이후 클라이언트는 `image_s3_key`로 mission-log 생성 요청을 보낸다.
- 서버는 mission-log 생성 시 S3 object를 직접 조회해 존재 여부, size, content-type, ownership, EXIF를 검증한다.

### `POST /api/mission-logs`

역할:

- 미션 인증 로그를 append-only로 저장한다.

Request:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `room_id` | `integer` | Y | 대상 방 |
| `image_s3_key` | `string` | Y | presigned upload API로 발급되고 업로드 완료된 이미지 key |

Response `201 Created`:

```json
{
  "mission_log_id": 9001,
  "room_id": 42,
  "participant_id": 101,
  "image_url": "https://cdn.example.com/mission/9001.jpg",
  "image_s3_key": "mission/42/101/9001.jpg",
  "server_time": "2026-05-11T05:58:10+09:00",
  "is_success": true,
  "failure_reason": null
}
```

인증은 성공했지만 정산에서 제외될 수 있는 예시:

`note`는 문서 설명용이며 실제 API 응답 필드는 아니다.

```json
{
  "mission_log_id": 9003,
  "room_id": 42,
  "participant_id": 101,
  "server_time": "2026-05-12T08:30:00+09:00",
  "is_success": true,
  "failure_reason": null,
  "note": "SPECIFIC_DAYS 비해당 요일로 정산 시 제외될 수 있음"
}
```

실패 판정이지만 로그는 저장된 경우:

```json
{
  "mission_log_id": 9002,
  "room_id": 42,
  "participant_id": 101,
  "image_url": "https://cdn.example.com/mission/9002.jpg",
  "image_s3_key": "mission/42/101/9002.jpg",
  "server_time": "2026-05-11T00:01:02+09:00",
  "is_success": false,
  "failure_reason": "BEFORE_START"
}
```

Error:

- `ROOM_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `PARTICIPANT_WITHDRAWN`

`PARTICIPANT_WITHDRAWN`는 참여 상태 기반 비즈니스 에러 코드이며, `AFTER_WITHDRAWN`은 MissionLog 검증 실패 사유(enum)다.

정책:

- 인증 시점에는 room 단위 Redisson 락을 기본으로 사용하지 않는다.
- 인증은 `MissionLog` 원본 보존이 우선이다.
- 이미지 업로드 자체는 별도 presigned upload 계약으로 처리하고, 이 API는 업로드 완료된 `image_s3_key`만 받는다.
- Presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- 서버는 `image_s3_key`가 현재 사용자/participant/room 범위에 속하는지 검증한다.
- 서버는 S3 object를 직접 조회해 존재 여부, size, content-type, ownership, EXIF를 검증한다.
- 클라이언트는 `exif_taken_at`을 authoritative source로 제출하지 않는다.
- 서버는 S3 object에서 EXIF를 추출하고 검증한다.
- `MissionLog.exif_taken_at`은 서버 검증 결과 저장값이다.
- EXIF가 없으면 `EXIF_MISSING`으로 실패 처리한다.
- EXIF가 유효하지 않으면 `EXIF_TIME_INVALID`로 실패 처리한다.
- 정산 인정 판단은 `server_time` 기준으로 수행한다.
- `server_time`은 서버가 인증 요청을 수신한 시각이다.
- `is_success`는 인증 요청이 유효성 검증을 통과했는지를 나타낸다.
- 아래 조건을 통과하면 `is_success = true`로 기록한다.
  - `EXIF` 존재 여부
  - `EXIF` 시간 유효성
  - 미션 기간 내 요청 여부
  - 탈퇴 이후 요청 여부
- `is_success = true`는 인증 성공을 뜻하지만, 최종 정산에서 인정된다는 의미는 아니다.
- `is_success = false`여도 원본 로그는 저장할 수 있다.
- `mission_log.failure_reason`은 인증 시점 실패 사유다.
- `settlement_item.calculation_reason`은 정산 시점 포함/제외 근거다.
- MVP 인증 API에서 `OUT_OF_SCHEDULE`는 사용하지 않는다.
- 최종 정산에서의 인정 여부는 `is_success`가 아니라 `Settlement` 계산 단계에서 결정된다.
- `SPECIFIC_DAYS`, `DAILY` 중복, `WEEKLY_N` 초과처럼 인증은 성공했지만 정산에서 제외되는 경우는 `mission_log.failure_reason`이 아니라 `settlement_item.calculation_reason`으로만 표현한다.
- 따라서 인증 시점 성공 로그도 최종 정산에서 제외될 수 있다. 예: `DAILY` 중복, `WEEKLY_N` 상한, `SPECIFIC_DAYS` 비유효 요일.
- 실시간 대시보드는 추정값이고, `SUCCEEDED` 전 정산 계산값은 `MissionLog` 재계산 결과로 확정한다.

### `GET /api/rooms/{roomId}/mission-logs/me`

역할:

- 현재 로그인한 회원의 해당 방 인증 기록을 조회한다.

Response `200 OK`:

```json
{
  "items": [
    {
      "mission_log_id": 9001,
      "participant_id": 101,
      "image_url": "https://cdn.example.com/mission/9001.jpg",
      "server_time": "2026-05-11T05:58:10+09:00",
      "exif_taken_at": "2026-05-11T05:57:58+09:00",
      "is_success": true,
      "failure_reason": null
    }
  ]
}
```

Error:

- `ROOM_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`

정책:

- 이 API는 원시 인증 기록 조회용이다.
- 정산 인정 판단 기준 시간은 `MissionLog.server_time`이다.
- `exif_taken_at`은 서버가 S3 object에서 추출/검증한 촬영 시각 보조 정보이며, 최종 정산 인정 시각 기준으로 사용하지 않는다.
- `is_success`는 인증 요청의 유효성 통과 여부를 의미하며, 정산에서 인정된 횟수를 나타내는 값이 아니다.
- FE는 이 값을 `최종 성공 횟수` 또는 `정산 인정 횟수`로 사용하면 안 된다.
- 최종 인정 여부와 인정 횟수는 반드시 정산 결과 API `GET /api/settlements/{settlementId}`를 기준으로 판단해야 한다.

## 5.4 인증 피드 / 리액션

이 섹션은 성공 인증을 보여주는 소셜 projection 계약이다. `feed_items[]`와 `day_statuses[]` / `participant_day_slots[]`는 같은 화면에서 함께 쓰일 수 있지만 의미가 다르다.

- `feed_items[]`는 feed-eligible `MissionLog` 게시물 목록이다. feed-eligible은 `mission_log.is_success = true` 인증 성공 로그만 뜻한다.
- `day_statuses[]`와 `participant_day_slots[]`는 참여자/일자 표시용 파생 상태다. 값은 `SUCCESS`, `FAILED`, `NOT_SUBMITTED`만 사용한다.
- 파생 상태는 DB 상태가 아니고 피드 게시물이 아니며 정산 입력도 아니다.
- feed eligibility는 정산 인정, 환급, 포인트 적립, AI 리포트 입력, `Settlement.status` 또는 방/참여 생명주기 전이를 의미하지 않는다.
- Canonical rule: Feed success does NOT guarantee settlement inclusion. `feed_items[].is_success = true`는 UX/social layer 표시 기준이며, 정산 포함 여부는 정산 시점 재계산 결과와 `settlement_item.calculation_reason`이 결정한다.
- 정산 인정 여부와 최종 성공 횟수는 정산 API와 `settlement_item.calculation_reason`을 기준으로 판단한다.

### `GET /api/rooms/{roomId}/feed`

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
  "room_id": 42,
  "feed_items": [
    {
      "mission_log_id": 9001,
      "participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "nickname": "saving-cat",
      "image_url": "https://cdn.example.com/mission/9001.jpg",
      "server_time": "2026-05-11T05:58:10+09:00",
      "created_at": "2026-05-11T05:58:10+09:00",
      "is_success": true,
      "reaction_counts": {
        "CHEER": 2,
        "CLAP": 1,
        "FIRE": 0
      },
      "my_reaction_type": "CHEER"
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
      "participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "date": "2026-05-11",
      "status": "SUCCESS",
      "representative_mission_log_id": 9001
    },
    {
      "participant_id": 102,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c908",
      "date": "2026-05-11",
      "status": "FAILED",
      "representative_mission_log_id": null
    }
  ]
}
```

Error:

- `ROOM_NOT_FOUND`
- `ROOM_ACCESS_DENIED`

정책:

- `feed_items[]`에는 `mission_log.is_success = true`인 인증 성공 로그만 포함한다.
- `is_success = false` 실패 로그와 미제출일은 `feed_items[]`에 포함하지 않는다.
- 같은 참여자/같은 날짜에 성공 인증 로그가 여러 개 있으면 endpoint pagination/filtering에서 별도 제한하지 않는 한 raw successful feed post로 모두 남을 수 있다.
- 참여자/일자 파생 상태 대표 규칙:
  - 성공 로그가 하나 이상 있으면 `SUCCESS`다.
  - 대표 성공 로그는 가장 이른 successful `created_at`, 동률이면 가장 낮은 `mission_log.id`다.
  - 성공 로그가 없고 실패 시도가 하나 이상 있으면 `FAILED`다.
  - 성공/실패 로그가 모두 없으면 `NOT_SUBMITTED`다.
- 대표 규칙은 표시/API payload용이다. 원본 성공 피드 게시물을 삭제, 병합, 수정, 숨김 처리하지 않는다.
- reaction counts는 `mission_log_reaction`에서 파생한다. `mission_log`에 저장 카운터를 두거나 갱신하지 않는다.
- 이 API의 상태 projection은 정산 인정 횟수, 환급액, 포인트 잔액, AI 리포트 상태, lifecycle status의 source of truth가 아니다.

### `POST /api/mission-logs/{missionLogId}/reactions`

역할:

- 현재 로그인한 회원의 해당 인증 성공 게시물 리액션을 멱등 upsert한다.

Request:

| 필드            | 타입     | 필수 | 설명                      |
| --------------- | -------- | ---- | ------------------------- |
| `reaction_type` | `string` | Y    | `CHEER` / `CLAP` / `FIRE` |

```json
{
  "reaction_type": "CHEER"
}
```

Response `200 OK`:

```json
{
  "mission_log_id": 9001,
  "my_reaction_type": "CHEER",
  "reaction_counts": {
    "CHEER": 3,
    "CLAP": 1,
    "FIRE": 0
  }
}
```

Error:

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`
- `INVALID_REACTION_TYPE`

정책:

- 리액션 대상은 `mission_log.is_success = true`인 feed-eligible `MissionLog`로 제한한다.
- `POST`는 `(mission_log_id, member_id)` 기준 멱등 upsert다. 기존 리액션이 있으면 같은 row의 `reaction_type`을 교체하고, 없으면 생성한다.
- 구현은 `(mission_log_id, member_id)` unique constraint 기반의 DB-level idempotent upsert를 MUST로 한다. SQL 문법은 실제 MySQL 8.0 stack에 맞춘다.

- 동일 `(mission_log_id, member_id)`에 대한 동시 중복 요청은 DB unique conflict 때문에 API 에러가 되어서는 안 되며, 최종 상태는 하나의 일관된 `reaction_type`으로 성공적으로 수렴해야 한다.
- 한 회원은 한 `MissionLog`에 하나의 리액션만 가진다.
- 리액션 생성/수정은 `mission_log`를 mutate하지 않는다.
- 리액션은 정산, 환급, 포인트 원장, AI 리포트, 방/참여/정산 상태 전이에 side effect를 만들지 않는다.

### `DELETE /api/mission-logs/{missionLogId}/reactions/me`

역할:

- 현재 로그인한 회원의 해당 인증 성공 게시물 리액션을 멱등 삭제한다.

Response `200 OK`:

```json
{
  "mission_log_id": 9001,
  "my_reaction_type": null,
  "reaction_counts": {
    "CHEER": 2,
    "CLAP": 1,
    "FIRE": 0
  }
}
```

Error:

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`

정책:

- 리액션이 이미 없어도 성공 응답을 반환한다.
- 삭제는 `(mission_log_id, member_id)` 기준 멱등 delete다.
- 삭제도 `mission_log` 원본, 정산, 환급, 포인트 원장, AI 리포트, 상태 생명주기에 side effect를 만들지 않는다.

## 5.5 룸 대시보드

### `GET /api/rooms/{roomId}/dashboard`

역할:

- 미션 방 화면에서 내 현재 수행 현황, 추정 인정 성공 횟수, 예상 환급금, 예상 손익, 추정 지분율, 추정 순위를 보여주는 UX용 projection API다.
- Dashboard는 단순 `MissionLog` 조회가 아니라 `MissionLog` 기반 성과/보상 estimated projection API다.
- Dashboard 값은 `Settlement.status = SUCCEEDED` 전까지 최종 정산 결과가 아니며, 정산 source of truth가 아니다.
- `Settlement.status = SUCCEEDED` 이후 최종 인정 성공 횟수, 최종 환급금, 최종 지분율은 `GET /api/settlements/{settlementId}`의 `settlement_item` 기준으로 표시한다.
- Dashboard projection과 최종 settlement 결과가 달라도 그 자체를 시스템 오류로 간주하지 않는다.

Response `200 OK`:

```json
{
  "room_id": 101,
  "participant_id": 1001,
  "settlement_id": null,
  "room_status": "ACTIVE",
  "settlement_status": "NONE",
  "projection_status": "LIVE",
  "projection_notice": "ESTIMATED_NOT_FINAL",
  "my_deposit_amount": 6800,
  "my_success_count": 5,
  "my_recognized_success_count_estimated": 4,
  "total_recognized_success_count_estimated": 31,
  "my_share_ratio_estimated": "0.12903200",
  "my_expected_refund_amount": 8000,
  "my_expected_net_profit_amount": 1200,
  "rank_estimated": 3,
  "updated_at": "2026-05-11T00:00:00+09:00"
}
```

Error:

- `ROOM_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `ROOM_ACCESS_DENIED`

정책:

- `projection_status`와 `projection_notice`는 API 응답용 값이며 DB enum이나 도메인 상태 원천으로 저장하지 않는다.
- `settlement_status = NONE`은 해당 방의 `Settlement` row가 아직 없다는 뜻이다. Dashboard projection을 계산할 수 없다는 뜻이 아니다.
- `updated_at`은 현재 Dashboard projection 계산/응답 생성 시각이다. `MissionLog`의 최신 수정 시각이나 `settlement.finished_at`을 뜻하지 않는다.
- 모든 문서화된 필드는 응답에 포함한다. 적용할 수 없는 projection 필드는 생략하지 않고 `null`로 내려준다.

#### ProjectionStatus

| 값 | 의미 |
| --- | --- |
| `NOT_STARTED` | `RECRUITING` 등 미션 수행 전 상태라 성과/보상 projection이 아직 시작되지 않았다. |
| `LIVE` | `ACTIVE` 상태에서 현재 `MissionLog`와 참여자 상태를 기준으로 실시간 추정값을 계산했다. |
| `FROZEN` | `CLOSED` 상태에서 `room.end_at` cutoff로 매 요청 시 deterministic하게 재계산한 frozen projection을 보여준다. 저장된 dashboard snapshot이 아니며 최종값도 아니다. |
| `NOT_PROVIDED` | `CANCELLED` 등 수행 성과 projection을 제공하지 않는 상태다. 환급/정산 안내는 Settlement API 기준이다. |
| `SETTLEMENT_SUCCEEDED` | 최종 정산이 성공했다. Dashboard는 최종값을 복제하지 않고 `settlement_id`로 Settlement API 조회를 유도한다. |

#### ProjectionNotice

| 값 | 의미 |
| --- | --- |
| `ESTIMATED_NOT_FINAL` | 현재 값은 실시간 참고용 추정값이며 최종 정산 결과가 아니다. |
| `NOT_STARTED` | 미션 수행 전이라 성과/보상 projection이 아직 시작되지 않았다. |
| `NOT_PROVIDED` | 현재 방 상태에서는 Dashboard 성과/보상 projection을 제공하지 않는다. |
| `SETTLEMENT_RESULT_AVAILABLE` | 최종 정산 결과가 존재하므로 `settlement_id`로 Settlement API를 조회해야 한다. |
| `INSUFFICIENT_PROJECTION_INPUT` | projection 계산에 필요한 참여자/보증금 입력을 충분히 확정할 수 없어 일부 추정 필드를 `null`로 반환한다. |

#### 상태별 필드 계약

| `projection_status` | 일반 room status | `settlement_id` | `my_deposit_amount` | `my_success_count` | `my_recognized_success_count_estimated` | `total_recognized_success_count_estimated` | `my_share_ratio_estimated` | `my_expected_refund_amount` | `my_expected_net_profit_amount` | `rank_estimated` | `updated_at` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `NOT_STARTED` | `RECRUITING` | `null` | value | `0` | `0` | `0` | `null` | `null` | `null` | `null` | value |
| `LIVE` | `ACTIVE` | nullable | value | value | value | value | value 또는 `null` | value 또는 `null` | value 또는 `null` | value 또는 `null` | value |
| `FROZEN` | `CLOSED` + not `SUCCEEDED` | nullable | value | value | value | value | value 또는 `null` | value 또는 `null` | value 또는 `null` | value 또는 `null` | value |
| `NOT_PROVIDED` | `CANCELLED` | nullable | value | `null` | `null` | `null` | `null` | `null` | `null` | `null` | value |
| `SETTLEMENT_SUCCEEDED` | any + `Settlement.status = SUCCEEDED` | value | value | `null` | `null` | `null` | `null` | `null` | `null` | `null` | value |

- `LIVE` / `FROZEN`에서 `settlement_id`는 실제 `Settlement` row가 있으면 값이고, `settlement_status = NONE`이면 `null`이다.
- `SETTLEMENT_SUCCEEDED`에서 성과/보상 projection 필드를 `null`로 내려주는 이유는 데이터가 없어서가 아니라 최종값의 source of truth가 `GET /api/settlements/{settlementId}`이기 때문이다.
- 대표 `projection_notice`는 `NOT_STARTED -> NOT_STARTED`, `LIVE/FROZEN -> ESTIMATED_NOT_FINAL`, `NOT_PROVIDED -> NOT_PROVIDED`, `SETTLEMENT_SUCCEEDED -> SETTLEMENT_RESULT_AVAILABLE`이다.
- `LIVE` / `FROZEN`에서 denominator 등 필수 projection 입력이 부족하면 해당 추정 필드는 `null`이고 `projection_notice = INSUFFICIENT_PROJECTION_INPUT`을 사용한다.

#### Projection source 역할

| Source | Dashboard에서의 역할 |
| --- | --- |
| `mission_log` | 성공 후보와 수행 현황의 primary event source다. `mission_log.is_success = true` 로그만 후보로 사용하고, 인정 판단 시간은 `MissionLog.server_time` 기준이다. |
| `room_participant` | 참여자 식별, 참여 상태, `deposit_amount`, `withdrawn_at`, 보증금 금액 source다. |
| `mission_room` | 방 상태, 기간, 미션 주기/규칙 컨텍스트다. 총 보증금 source가 아니다. |
| `settlement` | `SUCCEEDED` 여부와 최종값 전환 판단용이다. `SUCCEEDED` 전 Dashboard projection 계산 source가 아니다. |
| `point_history` | 포인트 원장 source of truth다. Dashboard projection 계산 source가 아니다. 최종 환급/잔액 반영은 `Settlement.status = SUCCEEDED` 이후 Settlement API와 `point_history` 기준으로 확인한다. |
| `point_account` | 현재 사용 가능 잔액 캐시다. Dashboard projection 계산 source가 아니며 `locked_balance`와 `my_expected_refund_amount`를 합산/차감해 최종 금액을 추론하지 않는다. |

#### 계산 규칙

- Dashboard는 deterministic estimated projection이다. 같은 source 입력과 cutoff를 사용하면 BE/FE/QA가 같은 projection 결과를 기대할 수 있어야 한다.
- `my_success_count`는 raw `mission_log.is_success = true` 성공 로그 수다. 정산 인정 성공 수가 아니다.
- `my_recognized_success_count_estimated`는 현재 시점에서 정산 규칙을 가능한 범위로 반영한 추정 인정 성공 수다.
- 추정 인정 성공 수는 `MissionLog.server_time`을 `Asia/Seoul` 기준 날짜/요일/주차로 해석해 계산한다.
- projection 후보 로그는 `mission_log.is_success = true`이고, `room.activated_at <= MissionLog.server_time <= projection_cutoff_at`을 만족해야 한다. `activated_at`이 `null`이면 post-activation projection을 계산하지 않는다.
  - `LIVE`에서는 `projection_cutoff_at = min(응답 생성 시각, room.end_at)`이다.
  - `FROZEN`에서는 `projection_cutoff_at = room.end_at`이다.
  - `room_participant.withdrawn_at`이 있으면 `MissionLog.server_time < withdrawn_at`인 success 로그만 후보로 사용한다.
- 대표 success 선택은 모든 frequency projection에서 동일하게 `MissionLog.server_time ASC`, 동률이면 `MissionLog.id ASC` 순서를 사용한다.
- `DAILY`는 같은 KST date의 첫 success만 인정하고 나머지 success는 duplicate로 제외한다.
- `SPECIFIC_DAYS`는 `mission_schedule_day`에 포함된 KST weekday의 success만 후보로 삼고, valid KST date별 첫 success만 인정한다.
- `WEEKLY_N`은 calendar week가 아니라 실제 activation anchor인 `room.activated_at`의 KST date를 기준으로 7일 bucket을 만들고, bucket별 정렬 상위 `frequency_count`개만 인정한다.
  - 예: `week_index = floor(days_between(kst_date(room.activated_at), kst_date(MissionLog.server_time)) / 7) + 1`
- `total_recognized_success_count_estimated`는 참여자별 추정 인정 성공 수 합계다.
- `my_share_ratio_estimated`는 소수 정밀도 오해를 줄이기 위해 문자열 decimal로 반환한다.
- `my_expected_refund_amount`는 deterministic base UX estimate다. `total_recognized_success_count_estimated > 0`이면 `FLOOR(total_locked_amount × my_share_ratio_estimated)`로 계산한다.
- Dashboard는 정산의 `remainder`, `remainder_policy`, deterministic draw winner, 1원 단위 잔액 배분을 계산하거나 반영하지 않는다. 해당 최종 지급 차이는 `Settlement.status = SUCCEEDED` 이후 Settlement API에서만 확인한다.
- `my_expected_net_profit_amount = my_expected_refund_amount - my_deposit_amount`다.
- `rank_estimated`는 예상 환급금/예상 손익/지분율/보증금 기준 순위가 아니라 추정 수행 순위다. 정렬 기준은 `recognized_success_count_estimated DESC`, 동률이면 `participant_id ASC`다.
- `total_recognized_success_count_estimated = 0`인 `LIVE` / `FROZEN` projection은 0으로 나누지 않고 균등 환급 base estimate를 적용한다.
- 균등 환급 추정 denominator는 현재 normal settlement projection에 포함되는 `room_participant.deposit_amount > 0` 참여자 전체다. `WITHDRAWN` 참여자도 정산 전 보증금이 남아 있으면 포함한다.
- zero-total base estimate의 `my_expected_refund_amount`는 `FLOOR(total_locked_amount / participant_count)`이며, 이 경우에도 remainder/draw 1원 배분은 Dashboard에서 수행하지 않는다.
- denominator를 확정할 수 없으면 `my_share_ratio_estimated`, `my_expected_refund_amount`, `my_expected_net_profit_amount`, `rank_estimated`는 `null`이고 `projection_notice = INSUFFICIENT_PROJECTION_INPUT`이다.
- `Settlement.status = SUCCEEDED` 이후 최종 인정 성공 횟수, 최종 환급금, 최종 지분율은 Dashboard projection보다 Settlement API가 우선하며, `settlement_item`과 연결된 `point_history`가 final source of truth다.
- Dashboard projection과 최종 settlement 결과가 달라도 시스템 오류로 보지 않는다.

#### Room status별 동작

| Room status | Dashboard 동작 |
| --- | --- |
| `RECRUITING` | 성과/보상 projection은 시작 전이다. 보증금, 방 규칙, `recruitment_deadline`, `start_at` 중심으로 표시한다. |
| `ACTIVE` | 실시간 estimated projection을 계산한다. 모든 금액/비율/순위는 추정값이다. |
| `CLOSED` | `room.end_at` cutoff로 query-time deterministic frozen projection을 재계산해 보여준다. 저장된 snapshot이 아니며, `Settlement.status = SUCCEEDED` 전까지 최종값이 아니므로 pending/running/retry 상태 안내를 함께 제공한다. |
| `CANCELLED` | 수행 성과 projection을 제공하지 않는다. 시작 전 취소 정산/환급은 Settlement API 기준으로 안내한다. |

#### locked_balance와의 관계

- `GET /api/points`의 `locked_balance`는 계정 단위 현재 잠긴 보증금 UX projection이다.
- Dashboard의 `my_expected_refund_amount`는 특정 room/participant 기준 예상 환급금 projection이다.
- FE는 `locked_balance`, `available_balance`, `total_balance`, `my_expected_refund_amount`를 합산하거나 차감해서 최종 보유 포인트, 출금 가능 금액, 확정 환급금을 계산하면 안 된다.
- `total_balance = available_balance + locked_balance` 관계는 포인트 요약 화면 전용이다.
- `CLOSED`지만 `Settlement.status != SUCCEEDED`인 방은 `locked_balance`에 아직 남을 수 있고, Dashboard는 frozen `my_expected_refund_amount`를 보여줄 수 있다.
- 최종 환급 여부와 금액은 `Settlement.status = SUCCEEDED` 이후 Settlement API와 `point_history` 원장 기준이다.

## 5.6 정산

### `GET /api/rooms/{roomId}/settlement`

역할:

- 방 기준으로 현재 정산 상태와 정산 식별자를 조회한다.

Response `200 OK`:

정산 row가 아직 없는 경우:

```json
{
  "room_id": 42,
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
  "room_id": 42,
  "settlement_id": 501,
  "settlement_type": "NORMAL",
  "status": "RUNNING",
  "retry_count": 1,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-01T00:05:10+09:00",
  "finished_at": null
}
```

Error:

- `ROOM_NOT_FOUND`

정책:

- `NONE`은 API projection이다.
- `PENDING -> RUNNING -> SUCCEEDED / RETRY_WAIT / FAILED`는 `Settlement.status` 원천 상태를 그대로 반영한다.
- `finished_at`은 성공/실패 종료 시각이다.

### `GET /api/settlements/{settlementId}`

역할:

- 정산 스냅샷과 참여자별 결과를 조회한다.

Response `200 OK`:

```json
{
  "settlement_id": 501,
  "room_id": 42,
  "settlement_type": "NORMAL",
  "status": "SUCCEEDED",
  "retry_count": 1,
  "total_participants": 5,
  "total_locked_amount": 500000,
  "total_recognized_success": 390,
  "total_base_refund_amount": 499996,
  "total_remainder_amount": 4,
  "remainder_policy": "TOP_1_ALL",
  "remainder_winner_participant_id": 101,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-01T00:05:10+09:00",
  "finished_at": "2026-06-01T00:05:18+09:00",
  "items": [
    {
      "settlement_item_id": 7001,
      "participant_id": 101,
      "participant_status_snapshot": "JOINED",
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
- `SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item + point_history`이며, `MissionLog` 재계산은 감사/디버깅용 검증에만 사용한다.
- `SUCCEEDED`는 모든 `settlement_item.point_history_id`가 채워지고 대응 `point_history` 존재가 검증된 상태를 뜻한다.
- partial 상태에서는 일부 item의 `point_history_id`가 `null`일 수 있고, 이 경우 `status`는 `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`다.
- 일반 정산에서 절사 후 남은 잔액은 기여도 1위 참여자에게 지급한다. 기여도 1위가 동점이면 성공 횟수를 비교하고, 그래도 같으면 재현 가능한 draw 규칙으로 1명을 결정한다.
- 전체 인정 성공 `0`이면 균등 환급 후, 남은 잔액은 같은 재현 가능한 규칙으로 `1원씩` 배분한다.

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
      "room_id": 42,
      "settlement_type": "NORMAL",
      "status": "FAILED",
      "retry_count": 3,
      "failure_code": "POINT_CREDIT_FAILED",
      "failure_message": "point_history insert timeout",
      "started_at": "2026-06-01T00:05:10+09:00",
      "finished_at": "2026-06-01T00:05:20+09:00"
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
  "room_id": 42,
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
- 같은 `room`이라도 `settlement_type`에 따라 별도 `Settlement`가 존재할 수 있으므로 retry 기준은 `roomId`가 아니라 `settlementId`다.
- 같은 방에 새 `Settlement`를 만드는 것이 아니라 지정된 기존 row를 재사용한다.
- 이미 생성된 `point_history`는 deterministic `idempotency_key`로 중복 지급이 차단된다.
- 동일 `idempotency_key`와 동일 payload의 중복은 기존 `point_history`를 재사용하거나 연결하고, 동일 키에 다른 payload가 확인되면 idempotency conflict로 실패 처리한다.
- partial 상태에서는 미지급 participant만 이어서 처리하거나, 이미 원장이 있으나 FK만 누락된 경우 기존 `point_history`를 재사용해 연결만 보정한다.

## 5.7 AI

AI API는 첫 릴리스 필수 사용자 기능을 위한 최소 계약만 고정한다. AI 실패, 무응답, 유효하지 않은 응답은 비트랜잭션성 기능 실패이지 시스템 실패가 아니다. 따라서 수동 방 생성, 정산 결과 조회, 환급, 포인트 원장, `Settlement.status`를 차단하거나 변경하지 않는다.

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
| `frequency_type`        | `string`        | N    | `DAILY` / `SPECIFIC_DAYS` / `WEEKLY_N` |
| `frequency_count`       | `integer`       | N    | 주 N회 등 반복 횟수                    |
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

- 추천 응답은 사용자가 확인/수정한 뒤 `POST /api/rooms`로 별도 저장한다.
- 유효하지 않은 AI 응답은 자동 저장하지 않는다.
- 실패 응답을 받아도 FE는 기존 입력값을 유지하고 수동 생성 흐름을 계속 제공해야 한다.

### `POST /api/rooms/{roomId}/ai-habit-report`

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
  "room_id": 42,
  "settlement_id": 501,
  "status": "PENDING"
}
```

이미 성공한 리포트 Response `200 OK`:

```json
{
  "report_id": 9001,
  "room_id": 42,
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
  "room_id": 42,
  "settlement_id": 501,
  "status": "FAILED",
  "report_body": null,
  "failure_code": "AI_REPORT_FAILED",
  "created_at": "2026-06-01T00:10:00+09:00",
  "completed_at": "2026-06-01T00:10:08+09:00"
}
```

Error:

- `ROOM_NOT_FOUND`
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

### `GET /api/rooms/{roomId}/ai-habit-report/me`

역할:

- 현재 사용자의 저장된 AI 습관 리포트 상태와 결과를 조회한다.

Response `200 OK`:

```json
{
  "report_id": 9001,
  "room_id": 42,
  "settlement_id": 501,
  "status": "SUCCEEDED",
  "report_body": "30일 중 27일을 성공했고, 오전 루틴의 지속성이 높았습니다.",
  "failure_code": null,
  "created_at": "2026-06-01T00:10:00+09:00",
  "completed_at": "2026-06-01T00:10:08+09:00"
}
```

Error:

- `ROOM_NOT_FOUND`
- `AI_REPORT_NOT_FOUND`

정책:

- `status`는 `PENDING`, `SUCCEEDED`, `FAILED` 중 하나다.
- `FAILED`여도 정산 결과 화면과 포인트 히스토리 조회는 그대로 가능해야 한다.

### `GET /api/ai-habit-reports/{reportId}`

역할:

- 리포트 식별자로 저장된 AI 습관 리포트 단건을 조회한다.

정책:

- 응답 구조와 상태 정책은 `GET /api/rooms/{roomId}/ai-habit-report/me`와 동일하다.
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
  "locked_balance": 100000,
  "total_balance": 450000,
  "updated_at": "2026-05-07T09:30:00+09:00"
}
```

정책:

- `available_balance`는 `point_account.balance`이며, 현재 사용 가능한 포인트 잔액만 의미한다.
- `locked_balance`는 DB 컬럼이 아니라 API 응답에서만 제공하는 projection 필드다.
- `locked_balance`는 정산 전 묶인 보증금 표시를 위한 UX 파생값이며, 포인트 원장의 source of truth가 아니다.
- MVP 기준 `locked_balance`는 사용자의 양수 `room_participant.deposit_amount`를 `mission_room`과 조인해 계산한다.

```sql
SELECT COALESCE(SUM(rp.deposit_amount), 0)
FROM room_participant rp
JOIN mission_room mr ON mr.id = rp.room_id
WHERE rp.member_id = :memberId
  AND rp.deposit_amount > 0
  AND mr.status IN ('RECRUITING', 'ACTIVE', 'CLOSED')
```

- `RECRUITING`은 참여 후 아직 시작 전이지만 보증금이 잠겨 있는 상태다.
- `ACTIVE`는 미션 진행 중이므로 보증금이 잠겨 있는 상태다.
- `CLOSED`는 정산 완료 전까지 보증금이 잠겨 있는 것으로 표시한다.
- `WITHDRAWN` 참여자도 즉시 환급되지 않으므로 정산 완료 전이면 `locked_balance`에 포함한다.
- `Settlement.status = SUCCEEDED` 이후에는 해당 방의 보증금 lock이 해제된 것으로 본다.
- 다만 MVP projection은 settlement 조인을 강제하지 않고 `mission_room.status` 기반으로 시작한다. 따라서 `CLOSED` 포함은 정산 전 잠금 표시를 위한 근사값이며, 더 정확한 정산 상태 기반 제외 조건은 Settlement 조회/정산 구현 단계에서 보강할 수 있다.
- `total_balance = available_balance + locked_balance`다.
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
- `ROOM_DEPOSIT_LOCK`는 자산 이동이 아니라 lock 이벤트다.
- `ROOM_SETTLEMENT_REFUND`와 `ROOM_CANCELLED_REFUND`는 실제 사용 가능 잔액 증가 이벤트다.
- `reference_type`은 `POINT_CHARGE`, `ROOM_PARTICIPANT`, `SETTLEMENT_ITEM`만 사용한다.
- `reference_type` / `reference_id` 매핑은 아래와 같다.

| 도메인 동작         | `transaction_type`       | `reference_type`   | `reference_id` 규칙                                                                                                 |
| ------------------- | ------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| 포인트 충전         | `POINT_CHARGE`           | `POINT_CHARGE`     | MVP에서는 생성된 `point_history.id`를 사용한다. API의 `payment_id`에 담긴 Toss `paymentKey`는 `idempotency_key = charge:{paymentKey}`에 남긴다. |
| 방 참여 보증금 잠금 | `ROOM_DEPOSIT_LOCK`      | `ROOM_PARTICIPANT` | `room_participant.id`                                                                                               |
| 일반 정산 환급      | `ROOM_SETTLEMENT_REFUND` | `SETTLEMENT_ITEM`  | `settlement_item.id`                                                                                                |
| 시작 전 취소 환급   | `ROOM_CANCELLED_REFUND`  | `SETTLEMENT_ITEM`  | `settlement_item.id`                                                                                                |


## 5.9 알림 / SSE

### `GET /api/notifications/stream`

역할:

- 현재 로그인한 사용자의 best-effort realtime notification stream을 SSE로 구독한다.
- 이 endpoint는 notification inbox, unread sync, replay cursor를 제공하지 않는다.
- Public API contract는 "현재 인증 사용자 stream"이다. 서버는 JWT `sub = member.uuid`로 인증 사용자를 식별하고 해당 사용자 대상 이벤트만 전달한다.
- `email`은 변경 가능하고 PII이므로 SSE routing identity, stream identifier, notification recipient key로 사용하지 않는다.

Request:

```http
GET /api/notifications/stream
Authorization: Bearer {accessToken}
Accept: text/event-stream
```

Response:

- `Content-Type: text/event-stream`
- 서버는 연결 유지 중 인증된 현재 사용자 대상 이벤트를 SSE data payload로 전달한다.
- 연결이 끊기면 클라이언트는 사용자에게 불필요한 오류를 노출하지 않고 재구독할 수 있다.

Event payload contract:

```json
{
  "eventId": "uuid-or-deterministic-id",
  "eventType": "MISSION_LOG_VERIFICATION_RESULT",
  "occurredAt": "2026-05-13T07:31:08+09:00",
  "resourceType": "missionLog",
  "resourceId": "1201",
  "message": "인증 결과가 반영되었습니다.",
  "severity": "success",
  "uiHint": {
    "toast": true,
    "refreshTargets": ["missionLogs", "dashboard"],
    "badgeDelta": 1
  }
}
```

Field policy:

| 필드 | 설명 |
| ---- | ---- |
| `eventId` | 클라이언트 세션 내 중복 이벤트 처리를 위한 이벤트 식별자 |
| `eventType` | 클라이언트 반응을 결정하는 SSE event catalog 값. DB enum이 아니다 |
| `occurredAt` | 이벤트 발생 시각. API 공통 시간 규칙을 따른다 |
| `resourceType` | 이벤트가 가리키는 도메인 리소스 종류 |
| `resourceId` | refetch, invalidate, route 이동 등에 사용할 리소스 식별자 |
| `message` | 사용자 표시용 fallback 문구. 클라이언트 분기 조건의 유일한 기준으로 사용하지 않는다 |
| `severity` | `info`, `success`, `warning`, `error` 중 하나의 표시 강도 |
| `uiHint` | 클라이언트 갱신을 돕는 UX hint. source of truth가 아니다 |

Payload 최소화 원칙:

- SSE payload에는 email, Long `member.id`, 불필요한 사용자 PII를 넣지 않는다.
- recipient 식별은 인증된 현재 사용자 기준으로 처리하고, 클라이언트가 필요로 하는 도메인 refetch 단서만 payload에 포함한다.
- SSE payload는 상태 snapshot이 아니라 REST refetch/invalidate를 유도하는 signal이다.

Initial event catalog:

- `MISSION_LOG_VERIFICATION_RESULT` — 인증 성공/실패 결과 반영
- `DASHBOARD_PROJECTION_CHANGED` — 예상 환급금, 지분율, 순위 같은 dashboard projection 변화
- `SETTLEMENT_STATUS_CHANGED` — 정산 상태 또는 결과 조회 가능 상태 변화
- `AI_MISSION_RECOMMENDATION_COMPLETED` — AI 미션 추천 초안 생성 완료

이 catalog는 DB enum이나 notification persistence schema를 의미하지 않는다. 새 eventType은 기존 REST API 화면으로 source-of-truth 상태를 다시 조회할 수 있는 도메인 변화에만 추가한다.

Reconnect / delivery semantics:

- SSE delivery는 best-effort realtime UX delivery다.
- 서버 재시작, 네트워크 단절, 브라우저 재연결 중 이벤트가 누락될 수 있다.
- missed event 복구는 replay가 아니라 기존 REST API 화면 재조회로 처리한다.
- 이벤트 순서, durable delivery, cross-device unread state를 보장하지 않는다.
- 같은 `eventId`가 다시 수신될 수 있으므로 클라이언트는 동일 세션에서 중복 이벤트를 idempotent하게 처리해야 한다.

Client reaction expectations:

- 클라이언트는 `message` 문자열만 해석하지 않고 `eventType`, `resourceType`, `resourceId`, `uiHint`를 기준으로 필요한 REST API를 다시 조회한다.
- SSE payload와 `uiHint`는 상태 snapshot이나 source of truth가 아니다. DB/API state가 source of truth다.
- notification persistence/read-state/unread sync는 MVP API 계약에 포함되지 않는다.

Error / close policy:

- 인증이 없거나 만료된 사용자는 stream에 연결할 수 없다.
- logout 또는 token invalidation 시 클라이언트는 SSE 연결을 닫아야 한다.
- SSE 연결 실패는 핵심 도메인 transaction 실패로 해석하지 않는다.

## 6. 상태 흐름 다이어그램

### 6.1 Room

```text
RECRUITING --host StartRoom success / activated_at set--> ACTIVE -> CLOSED
RECRUITING --start_at expired cancellation batch--> CANCELLED
```

### 6.2 Participant

```text
JOINED -> WITHDRAWN
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

- 방 화면은 `status`, `visibility`, `frequency_type`, `frequency_count`, `mission_schedule_days`, `deposit_amount`, `my_participation`을 기준으로 버튼 상태를 결정한다.
- 계정/포인트 요약 화면은 `GET /api/points.available_balance`, `GET /api/points.locked_balance`, `GET /api/points.total_balance`를 기준으로 표시한다.
- FE는 여러 방의 `my_participation.deposit_locked_amount`를 직접 합산해 계정 단위 잠금 잔액을 만들지 않고, `GET /api/points.locked_balance`를 표시한다.
- `my_participation.deposit_locked_amount`는 방 상세의 해당 참여 보증금 표시용 필드이며, 계정 단위 `locked_balance`나 `total_balance`의 source of truth가 아니다.
- `locked_balance`와 `total_balance`는 UX 표시용이며, 출금 가능 여부, 환급 가능 여부, 분쟁 처리, 정산 결과 판단 기준으로 사용하면 안 된다.
- 탈퇴 버튼은 `RECRUITING`, `ACTIVE`에서만 노출하고, 탈퇴 직후에도 보증금은 즉시 반환되지 않는다고 안내해야 한다.
- FE는 `is_success`를 인증 성공 여부로만 사용해야 하고, 최종 인정 여부 판단 기준으로 사용하면 안 된다.
- 피드 화면에서 `feed_items[]`는 성공 인증 게시물이고, `day_statuses[]` / `participant_day_slots[]`는 `SUCCESS`, `FAILED`, `NOT_SUBMITTED` 표시용 projection이다. 둘을 정산 결과나 포인트 원장으로 해석하면 안 된다.
- 리액션은 `mission_log_reaction` 기반 social metadata이며, `reaction_counts`는 파생값이다. FE는 리액션이 인증 성공 여부, 정산 인정, 환급, 포인트, AI 리포트 상태를 바꾼다고 표시하면 안 된다.
- 인증 제출 직후에는 `is_success`와 `failure_reason`만 신뢰한다. 최종 인정 횟수는 정산 전까지 확정되지 않는다.
- 인증 직후에는 성공으로 표시할 수 있지만, 최종 결과 화면의 인정/미인정 표시는 정산 결과 기준으로 별도 표시해야 한다.
- 인증 기록 화면과 정산 결과 화면은 서로 다른 기준을 사용해야 한다.
- 인증 기록 화면은 `is_success` 기준으로 `인증 성공/실패`만 표시한다.
- 정산 결과 화면은 `settlement_item.calculation_reason` 기준으로 `최종 인정/미인정`을 표시한다.
- 두 기준을 혼용하면 잘못된 UX가 발생하므로 반드시 분리해서 사용해야 한다.
- 최종 정산 인정 시각 판단은 `server_time` 기준이며, `exif_taken_at`은 촬영 시각 검증용 보조 정보로만 사용해야 한다.
- `failure_reason = null`이어도 최종 정산에서 제외될 수 있으므로, `DAILY` 중복이나 `SPECIFIC_DAYS` 제외 여부는 `settlement_item.calculation_reason`이 포함된 정산 결과 화면에서 확인해야 한다.
- 정산 결과 화면은 먼저 `GET /api/rooms/{roomId}/settlement`를 polling하고, `status = SUCCEEDED`가 되면 `settlement_id`로 `GET /api/settlements/{settlementId}`를 호출한다.
- 포인트 내역 화면은 `transaction_type` 그대로 내려받고, UI에서 `POINT_CHARGE`, `ROOM_DEPOSIT_LOCK`, `ROOM_SETTLEMENT_REFUND`, `ROOM_CANCELLED_REFUND`를 한국어 라벨로 매핑한다.
- 포인트 내역 화면의 `next_cursor`는 UI가 직접 해석하지 말고 다음 요청에 그대로 전달해야 한다.
- `Settlement.status = SUCCEEDED` 전에는 일부 `point_history_id`가 비어 있을 수 있으므로, 정산 상세의 item 금액과 포인트 내역 표시 시 상태를 함께 봐야 한다.
- Dashboard 화면의 금액, 비율, 순위는 “예상”, “현재 기준”, “추정” 라벨로 표시하고 “확정”, “최종”, “정산 완료” 라벨은 Settlement API 결과에만 사용한다.
- Dashboard의 `my_expected_refund_amount`와 `GET /api/points`의 `locked_balance`, `available_balance`, `total_balance`를 조합해 최종 보유 포인트, 출금 가능 금액, 확정 환급금을 계산하면 안 된다.
- Dashboard의 `projection_status = SETTLEMENT_SUCCEEDED`이면 `settlement_id`로 `GET /api/settlements/{settlementId}`를 호출해 최종 인정 성공 수, 최종 지분율, 최종 환급금을 표시한다.
- Dashboard projection과 최종 settlement 결과가 달라도 시스템 오류로 간주하지 않고, 차이 설명은 Settlement API의 `settlement_item.calculation_reason`을 기준으로 한다.


### SSE realtime UX handling

SSE realtime UX handling은 `GET /api/notifications/stream` 계약의 client reaction expectations와 reconnect / delivery semantics를 따른다.

## 8. 구현 메모

- 인증 시점에는 과도한 분산 락보다 `MissionLog append-only 저장 + 캐시 원자 연산 + 최종 재계산`을 우선한다.
- `SUCCEEDED` 전 최종 정산 계산은 `MissionLog.server_time`과 room/participant 상태를 기준으로 수행한다.
- `exif_taken_at`은 서버가 S3 object에서 추출/검증한 이미지 조작 또는 촬영 시각 이상 여부 검증 보조 정보이며, 인정 횟수 계산 기준 시간으로 사용하지 않는다.
- 정산 시점에는 DB 조건부 `Settlement(PENDING/RETRY_WAIT -> RUNNING)` claim을 1차 기준으로 사용하고, Redisson room lock은 보조 수단, DB unique 제약과 `point_history.idempotency_key`는 최종 방어선으로 사용한다.
- `total_locked_amount`는 정산 시점의 `room_participant.deposit_amount` 합계 스냅샷이며, `point_account`나 `point_history`를 다시 합산하지 않는다.
- 일반 정산 remainder는 `기여도 1위 -> 성공 횟수 비교 -> 재현 가능한 draw` 순서로 결정한다.
- 취소형 정산에서도 조회 구조는 동일하고 `settlement_type = CANCELLED_BEFORE_START`만 달라진다.
- 포인트 원장 기록과 `point_account.balance` 갱신은 participant 지급 단위로 같은 트랜잭션에서 처리하되, 전체 정산은 partial 복구가 가능하도록 이미 생성된 원장을 idempotency key로 재사용한다.

## 9. 확정 메모

- 탈퇴는 `RECRUITING`, `ACTIVE` 모두에서 허용하되, 즉시 환급은 없다.
- MVP 인증 API의 `mission_log.failure_reason`은 인증 시점 실패 사유만 표현하고, `OUT_OF_SCHEDULE`는 사용하지 않는다.
- 관리자 정산 재시도는 `roomId`가 아니라 `settlementId` 기준으로 수행한다.
