# API 명세: Dondok MVP

> Canonical active API source: `backend/docs/api/*`.
> This integrated contract mirrors the backend MVP API documents for FE/BE integration.

## 1. 목적

이 문서는 Dondok MVP의 통합 API 계약 문서다. `backend/docs/api/*`를 active API source of truth로 삼고, 클라이언트가 실제로 호출해야 하는 endpoint, 공통 규칙, enum, response semantics를 한 곳에 모은다.

이번 migration 기준:

- `backend/docs/api/overview.md`의 API 목록이 active endpoint registry다.
- `backend/docs/api/*.md`의 endpoint heading set이 active detail source다.
- backend active API에 없는 endpoint는 active contract가 아니며, 필요하면 `Deferred / Brownfield / Removed Surfaces`에만 둔다.
- ERD / Schema / Settlement 설계 문서는 이 문서 확정 후 정렬하는 derived implementation docs다.

### Semantic guardrails

아래 의미 경계는 API response shape가 편의상 필드를 노출하더라도 깨지면 안 된다.

- Projection은 final settlement가 아니다.
- Retry는 correction, replay, recalculation이 아니다.
- Replay는 감사/디버깅용 재현이며 payout mutation 권한이 아니다.
- Notification / inbox / read state는 canonical domain state가 아니다. 알림 클릭 후 canonical REST API를 refetch한다.
- Host는 lifecycle, settlement, point ledger authority가 아니다.
- 전체 인정 성공이 0인 all-fail 상황은 equal principal refund를 적용한다.
- `settlement_item` 계산 스냅샷과 연결된 `point_history`가 final settlement 이후 운영/분쟁/조회 기준이다.
- API 편의 필드, display field, projection field는 authoritative state처럼 해석하지 않는다.

## 2. 공통 규칙

### Base URL

```
/api
```

### 인증

#### 토큰 전달 방식

| 토큰          | 전달 위치                 | 형식                   | 사용 목적                        |
| ------------- | ------------------------- | ---------------------- | -------------------------------- |
| Access Token  | 요청 헤더 `Authorization` | `Bearer {accessToken}` | 보호된 API 요청 인증             |
| Refresh Token | 쿠키 `refreshToken`       | HttpOnly Cookie        | `POST /api/auth/refresh` 명시적 호출 시 쿠키로 자동 전송 |

#### 요청 헤더 규칙

보호된 API를 호출할 때는 Access Token을 `Authorization` 헤더에 담아 전송한다.

```http
Authorization: Bearer {accessToken}
```

- `Bearer`와 토큰 사이에는 공백 한 칸을 둔다.
- `Authorization` 헤더가 없거나 `Bearer ` 접두사가 없으면 Access Token이 없는 요청으로 처리된다.
- Access Token에는 `type=access` 클레임이 있어야 한다.
- Refresh Token은 `Authorization` 헤더로 보내지 않는다. Refresh Token은 서버가 발급한 `refreshToken` 쿠키로만 전송한다.

#### 로그인 응답 규칙

`POST /api/auth/login` 성공 시 서버는 Access Token을 응답 바디로 내려주고, Refresh Token은 `Set-Cookie` 헤더로 내려준다.

```json
{
  "access_token": "{accessToken}",
  "token_type": "Bearer",
  "expires_in": 1800,
  "member": {
    "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
    "email": "user@example.com",
    "nickname": "돈독러"
  }
}
```

```http
Set-Cookie: refreshToken={refreshToken}; Path=/; Max-Age=604800; HttpOnly; SameSite=Lax
```

- Access Token 만료 시간은 현재 설정 기준 1800초(30분)이다.
- Refresh Token 만료 시간은 현재 설정 기준 604800초(7일)이다.
- 개발 환경에서는 `refreshToken` 쿠키의 `Secure=false`, `SameSite=Lax` 설정을 사용한다.
- 운영 환경에서 크로스 사이트 쿠키 전송이 필요하면 `Secure=true`, `SameSite=None` 설정을 사용한다.

#### 인증 필요한 API 호출 예시

```http
GET /api/me HTTP/1.1
Authorization: Bearer {accessToken}
Cookie: refreshToken={refreshToken}
```

일반적인 보호 API 호출에는 `Authorization` 헤더가 필수이다. Access Token은 FE 메모리에서 관리하며 요청마다 헤더에 직접 포함한다. Credentials 전략(쿠키 자동 전송 등)은 implementation detail로 두며, `POST /api/auth/refresh` · `POST /api/auth/logout` 등 쿠키 수신이 필요한 endpoint는 각 endpoint 명세를 따른다.

#### Access Token 재발급

Access Token이 만료되면 클라이언트는 `POST /api/auth/refresh`를 명시적으로 호출하여 새 Access Token을 발급받는다.

- Refresh Token은 `HttpOnly` 쿠키로만 전달하며, request body로 보내지 않는다.
- 재발급 성공 시 새 `access_token`을 response body로 반환한다. 필요 시 새 Refresh Token을 `Set-Cookie`로 rotate한다.
- 재발급 실패(`REFRESH_TOKEN_INVALID`, `REFRESH_TOKEN_EXPIRED`, `REFRESH_TOKEN_REVOKED`) 시 클라이언트는 로그인 화면으로 유도한다.
- 서버 미들웨어가 자동으로 refresh하거나 응답 `Authorization` 헤더로 새 Access Token을 전달하는 방식은 사용하지 않는다.

#### 로그아웃 규칙

`POST /api/auth/logout`은 인증이 필요한 API이다. 요청 시 Access Token을 `Authorization` 헤더에 포함해야 한다.

```http
POST /api/auth/logout HTTP/1.1
Authorization: Bearer {accessToken}
Cookie: refreshToken={refreshToken}
```

로그아웃 성공 시 서버는 저장된 Refresh Token을 삭제하고, `refreshToken` 쿠키를 만료시킨다.

```http
Set-Cookie: refreshToken=; Path=/; Max-Age=0; HttpOnly; SameSite=Lax
```

#### 인증 예외 API

다음 API는 `Authorization` 헤더 없이 호출할 수 있다.

| Method | Path                  | 설명     |
| ------ | --------------------- | -------- |
| POST   | `/api/auth/login`     | 로그인   |
| POST   | `/api/auth/signup`    | 회원가입 |

그 외 API는 기본적으로 인증이 필요하다.

#### CORS 관련 헤더

현재 서버는 프론트엔드 출처 `http://localhost:3000`을 허용한다.

- 허용 메서드: `GET`, `POST`, `PATCH`, `PUT`, `DELETE`, `OPTIONS`
- 허용 요청 헤더: 전체 허용
- 노출 응답 헤더(`Access-Control-Expose-Headers`): `Authorization`
- 쿠키 인증을 위해 `Access-Control-Allow-Credentials: true`와 명시적 origin 허용이 필요하다.

`Set-Cookie`는 브라우저가 쿠키 처리용으로 직접 소비하는 헤더이며, `Access-Control-Expose-Headers`로 노출하는 JS 읽기 대상이 아니다. 브라우저 클라이언트는 Refresh Token 쿠키 송수신을 위해 요청에 `credentials: include`를 설정해야 한다.

### 식별자

| 식별자        | 용도                                                            |
| ------------- | --------------------------------------------------------------- |
| `member.uuid` | 외부 API 식별자. JWT `sub`, 외부 사용자 식별자로 사용한다.      |
| `member.id`   | DB 내부 FK 전용. API 응답 및 JWT에 사용하지 않는다.             |
| `email`       | 로그인 식별자. routing identity, JWT subject로 사용하지 않는다. |

### 시간

- 모든 시간 값은 offset 포함 ISO-8601 문자열로 주고받는다.
- 예: `2026-05-07T00:05:00+09:00`
- 미션 기간과 정산 판단 기준 시간대는 `Asia/Seoul`로 고정한다.

### 금액

- 모든 금액 필드는 `integer` 원 단위다.

### 에러 응답

모든 4xx/5xx 응답은 아래 형식을 따른다.

```json
{
  "code": "ERROR_CODE",
  "message": "설명",
  "timestamp": "2026-05-07T00:05:00+09:00"
}
```

---

## 3. 도메인 상태 / Enum

### CrewStatus

| 값           | 설명                                                   |
| ------------ | ------------------------------------------------------ |
| `RECRUITING` | 모집 중. `recruitment_deadline` 전 신청/승인/예치 가능 |
| `ACTIVE`     | 진행 중. 시스템이 `start_at`에 자동 전환               |
| `CLOSED`     | 정상 종료                                              |
| `CANCELLED`  | 시작 전 취소                                           |

### ParticipantStatus

| 값          | 설명                                |
| ----------- | ----------------------------------- |
| `PENDING`   | 가입 신청 완료, 보증금 reserve 상태. capacity reservation에 포함하나 activation eligibility, frozen baseline, settlement 대상은 아님 |
| `LOCKED`    | 방장 승인으로 reserve 확정된 참여 상태. 크루 생성 시 호스트는 별도 신청 없이 이 상태로 자동 생성됨. activation eligibility, frozen participant baseline, settlement eligibility의 anchor |
| `REJECTED`  | 방장 거절. reserve는 `CREW_RESERVE_RELEASE`로 반환 |
| `CANCELLED` | 사용자 신청 취소 (승인 전 `PENDING`만). reserve는 `CREW_RESERVE_RELEASE`로 반환. terminal이 아님: 동일 crew에 재신청(reopen) 가능. reopen 시 기존 row를 `CANCELLED → PENDING`으로 in-place 복귀 |
| `EXPIRED`   | 시작 전까지 처리되지 않아 자동 만료. reserve는 `CREW_RESERVE_RELEASE`로 반환 |

### SettlementStatus

| 값           | 설명                                                     |
| ------------ | -------------------------------------------------------- |
| `NONE`       | Settlement row 없음 (projection-only)                    |
| `PENDING`    | 생성됨, 실행 전                                          |
| `RUNNING`    | 실행 중                                                  |
| `SUCCEEDED`  | 완료                                                     |
| `FAILED`     | 실패                                                     |
| `RETRY_WAIT` | 재시도 대기                                              |

### CertificationStatus

`MissionLog.certification_status` 값이다. 리액션은 `SUCCESS` 상태의 로그에만 허용된다.

| 값 | 설명 |
| --- | --- |
| `PENDING_REVIEW` | 호스트 검수 대기 중 |
| `SUCCESS` | 인증 성공. 리액션 대상이 되는 유일한 상태 |
| `FAILED` | 인증 실패 (호스트 거절 또는 시스템 판정) |

### ProjectionStatus

`GET /api/crews/{crewId}/dashboard`의 `projection_status` 값이다. DB에 저장하지 않는 API 응답 전용 값이다.

| 값 | 설명 |
| --- | --- |
| `NOT_STARTED` | 미션 수행 전. 진행/환급 projection 미시작 |
| `LIVE` | `ACTIVE` 상태에서 current-basis estimate 계산 중 |
| `CLOSED_ESTIMATE` | `CLOSED` 상태에서 current-basis estimate 계산 중. 최종값 아님 |
| `NOT_PROVIDED` | `CANCELLED` 등 projection 제공 불가 상태 |
| `SETTLEMENT_SUCCEEDED` | 최종 정산 완료. 최종값은 `GET /api/settlements/{settlementId}` 기준 |

### ProjectionNotice

`GET /api/crews/{crewId}/dashboard`의 `projection_notice` 값이다. DB에 저장하지 않는 API 응답 전용 값이다.

| 값 | 설명 |
| --- | --- |
| `ESTIMATED_NOT_FINAL` | 현재 값은 참고용 estimate이며 최종 정산 결과가 아님 |
| `NOT_STARTED` | 미션 수행 전 |
| `NOT_PROVIDED` | 현재 방 상태에서 projection 미제공 |
| `SETTLEMENT_RESULT_AVAILABLE` | 최종 정산 결과 존재. Settlement API 조회 필요 |
| `INSUFFICIENT_PROJECTION_INPUT` | projection 계산 입력 부족. 일부 추정 필드 `null` |

### 기타 Enum

| Enum                         | 값                                                                                                                         |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `FrequencyType`              | `DAILY`, `SPECIFIC_DAYS`                                                                                                   |
| `PointTransactionType`       | `POINT_CHARGE`, `CREW_DEPOSIT_RESERVE`, `CREW_RESERVE_RELEASE`, `CREW_SETTLEMENT_REFUND`                                   |
| `DailySettlementType`        | `A` (인증마감 09:00 / 정산 12:00), `B` (인증마감 21:00 / 정산 00:00), `C` (인증마감 23:59 / 정산 익일 12:00)               |
| `MissionLogDecisionType`     | `MANUAL_APPROVE`, `MANUAL_REJECT`, `AUTO_APPROVE`, `AUTO_REJECT`                                                           |
| `MissionLogRejectReasonCode` | `TIME_VIOLATION`, `DUPLICATE`, `MISSION_MISMATCH`, `UNCLEAR`, `INAPPROPRIATE`, `OTHER`                                     |
| `MissionLogFailureReason`    | `EXIF_MISSING`, `EXIF_TIME_INVALID`, `BEFORE_START`, `AFTER_END`. `OUT_OF_SCHEDULE`는 MVP에서 사용하지 않음 |
| `SettlementFailureCode`      | `INPUT_LOAD_FAILED`, `CALCULATION_FAILED`, `POINT_CREDIT_FAILED`, `DUPLICATE_SETTLEMENT`, `LOCK_ACQUIRE_FAILED`, `UNKNOWN` |
| `PointHistoryReferenceType`  | `POINT_CHARGE`, `CREW_PARTICIPANT`, `SETTLEMENT_ITEM`                                                                      |
| `MissionLogReactionType`     | 고정 enum이 아님. FE가 선택한 emoji/token string을 그대로 사용한다. 서버는 trim 후 blank 거부, `VARCHAR(20)` 길이 검증만 수행한다 |

---

## 4. API 목록

| 도메인      | Method   | Path                                                           | 설명                              |
| ----------- | -------- | -------------------------------------------------------------- | --------------------------------- |
| 인증/회원   | `POST`   | `/api/auth/signup`                                             | 회원가입                          |
| 인증/회원   | `POST`   | `/api/auth/login`                                              | 로그인                            |
| 인증/회원   | `POST`   | `/api/auth/refresh`                                            | access token 재발급               |
| 인증/회원   | `POST`   | `/api/auth/logout`                                             | 로그아웃                          |
| 인증/회원   | `GET`    | `/api/me`                                                      | 내 계정/프로필 조회               |
| 인증/회원   | `PATCH`  | `/api/me/profile`                                              | 내 프로필 수정                    |
| 크루/참여   | `GET`    | `/api/crews`                                                   | 크루 목록 조회                    |
| 크루/참여   | `POST`   | `/api/crews`                                                   | 크루 생성                         |
| 크루/참여   | `GET`    | `/api/crews/{crewId}`                                          | 크루 상세 조회                    |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/participants`                             | 크루 가입 신청                    |
| 크루/참여   | `DELETE` | `/api/crews/{crewId}/participants/me`                          | 가입 신청 취소                    |
| 크루/참여   | `GET`    | `/api/crews/{crewId}/applications`                             | 가입 신청 목록 조회 (방장)        |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/applications/{crewParticipantId}/approve` | 방장 승인                         |
| 크루/참여   | `POST`   | `/api/crews/{crewId}/applications/{crewParticipantId}/reject`  | 방장 거절                         |
| 크루/참여   | `GET`    | `/api/crews/{crewId}/members`                                  | 크루 멤버 목록 조회               |
| 크루 공지   | `GET`    | `/api/crews/{crewId}/notices`                                  | 공지 목록 조회                    |
| 크루 공지   | `POST`   | `/api/crews/{crewId}/notices`                                  | 방장 공지 작성                    |
| 크루 공지   | `PATCH`  | `/api/crews/{crewId}/notices/{noticeId}`                       | 방장 공지 수정                    |
| 크루 공지   | `DELETE` | `/api/crews/{crewId}/notices/{noticeId}`                       | 공지 표시 상태 삭제               |
| 크루 공지   | `GET`    | `/api/crews/{crewId}/notices/{noticeId}/comments`              | 공지 댓글 목록                    |
| 크루 공지   | `POST`   | `/api/crews/{crewId}/notices/{noticeId}/comments`              | 공지 댓글 작성                    |
| 크루 공지   | `PATCH`  | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}`  | 공지 댓글 수정                    |
| 크루 공지   | `DELETE` | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}`  | 댓글 표시 상태 삭제               |
| 크루 공지   | `POST`   | `/api/crews/{crewId}/notices/{noticeId}/reactions`             | 공지 리액션 upsert                |
| 크루 공지   | `DELETE` | `/api/crews/{crewId}/notices/{noticeId}/reactions/me`          | 내 공지 리액션 삭제               |
| 미션 인증   | `POST`   | `/api/uploads/presigned-url`                                   | 이미지 업로드 presigned URL 발급  |
| 미션 인증   | `POST`   | `/api/mission-logs`                                            | 인증 제출                         |
| 미션 인증   | `GET`    | `/api/crews/{crewId}/mission-logs/me`                          | 내 인증 기록 조회                 |
| 미션 인증   | `GET`    | `/api/me/verification-history`                                 | 내 검증 결과 현황 조회            |
| 미션 인증   | `GET`    | `/api/me/mission-feed`                                         | 내 크루별 인증 활동 타임라인 조회 |
| 미션 인증   | `GET`    | `/api/crews/{crewId}/moderation-logs`                          | 방장 검수 이력 조회 (방장 전용)   |
| 미션 인증   | `POST`   | `/api/mission-logs/{missionLogId}/moderation/approve`          | 방장 검수 승인                    |
| 미션 인증   | `POST`   | `/api/mission-logs/{missionLogId}/moderation/reject`           | 방장 검수 거절                    |
| 피드/리액션 | `GET`    | `/api/crews/{crewId}/feed`                                     | 인증 피드 조회                    |
| 피드/리액션 | `POST`   | `/api/mission-logs/{missionLogId}/reactions`                   | 리액션 추가                       |
| 피드/리액션 | `DELETE` | `/api/mission-logs/{missionLogId}/reactions/me`                | 리액션 삭제                       |
| 대시보드    | `GET`    | `/api/crews/{crewId}/dashboard`                                | 진행 현황 및 예상 환급 조회       |
| 정산        | `GET`    | `/api/crews/{crewId}/settlement`                               | 방 기준 정산 상태 조회            |
| 정산        | `GET`    | `/api/settlements/{settlementId}`                              | 정산 결과 상세 조회               |
| AI          | `POST`   | `/api/ai/mission-recommendations`                              | AI 크루 생성 도우미               |
| 알림        | `POST`   | `/api/notification-devices`                                    | FCM 디바이스 등록                 |
| 알림        | `PATCH`  | `/api/notification-devices/{deviceId}`                         | FCM 토큰 갱신                     |
| 알림        | `DELETE` | `/api/notification-devices/{deviceId}`                         | FCM 디바이스 비활성화             |
| 알림        | `GET`    | `/api/notifications`                                           | 알림 목록 조회                    |
| 알림        | `GET`    | `/api/notifications/unread-count`                              | 미읽음 알림 수 조회               |
| 알림        | `PATCH`  | `/api/notifications/{notificationId}/read`                     | 알림 읽음 처리                    |
| 알림        | `PATCH`  | `/api/notifications/read-all`                                  | 전체 읽음 처리                    |
| 포인트      | `POST`   | `/api/points/charges`                                          | 포인트 충전                       |
| 포인트      | `GET`    | `/api/points`                                                  | 포인트 잔액 조회                  |
| 포인트      | `GET`    | `/api/points/history`                                          | 포인트 내역 조회                  |

---

## 5. API 상세

## 5.1 인증 / 회원


### `POST /api/auth/signup`

> 이메일, 비밀번호, 닉네임으로 새 계정을 생성한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `email` | `string` | Y | 로그인 식별자 |
| `password` | `string` | Y | 비밀번호 원문 |
| `nickname` | `string` | Y | 노출 이름 |

**Response** `201 Created`

```json
{
  "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "email": "user@example.com",
  "nickname": "돈독러",
  "status": "ACTIVE",
  "created_at": "2026-05-07T09:00:00+09:00"
}
```

**Error**

- `EMAIL_ALREADY_EXISTS`
- `NICKNAME_ALREADY_EXISTS`
- `VALIDATION_ERROR`

**정책**

- `email`과 `nickname`은 unique다.
- 가입 직후 자동 로그인 여부는 본 명세에서 고정하지 않는다. MVP 기본 흐름은 가입 후 로그인이다.

---

### `POST /api/auth/login`

> 이메일과 비밀번호로 로그인하여 JWT 토큰을 발급한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `email` | `string` | Y | 로그인 식별자 |
| `password` | `string` | Y | 비밀번호 원문 |

**Response** `200 OK`

```json
{
  "access_token": "{accessToken}",
  "token_type": "Bearer",
  "expires_in": 1800,
  "member": {
    "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
    "email": "user@example.com",
    "nickname": "돈독러"
  }
}
```

```http
Set-Cookie: refreshToken={refreshToken}; Path=/; Max-Age=604800; HttpOnly; SameSite=Lax
```

**Error**

- `INVALID_CREDENTIALS`
- `MEMBER_DEACTIVATED`

**정책**

- JWT `sub`는 `member.uuid`다. `email`이나 `member.id`를 subject로 사용하지 않는다.
- refresh token은 `HttpOnly` + `Secure` + `SameSite` cookie로만 전달한다. response body, `localStorage`, `sessionStorage`, JS 접근 대상으로 노출하지 않는다.
- refresh token은 서버에 hash로 저장한다.

---

### `POST /api/auth/refresh`

> refresh token으로 access token을 재발급한다.

**Request** body 없음. 브라우저/클라이언트가 자동 전송하는 refresh token cookie(`HttpOnly`, `Secure`, `SameSite`)를 서버가 읽는다.

**Response** `200 OK`

```json
{
  "access_token": "new-access-token"
}
```

rotation 정책에 따라 새 refresh token이 발급되는 경우 `Set-Cookie` 헤더로 갱신한다.

```http
Set-Cookie: refreshToken={newRefreshToken}; Path=/; HttpOnly; Secure; SameSite=Lax
```

**Error**

- `REFRESH_TOKEN_INVALID`
- `REFRESH_TOKEN_EXPIRED`
- `REFRESH_TOKEN_REVOKED`

**정책**

- 재발급은 refresh cookie 기반이며, request body로 refresh token을 받지 않는다.
- 새 refresh token도 `Set-Cookie`로만 재발급한다 (rotate). token 값을 response body에 포함하지 않는다.

---

### `POST /api/auth/logout`

> refresh token을 폐기하여 로그아웃한다.

**Request** body 없음. 인증이 필요한 API로, `Authorization: Bearer {accessToken}` 헤더와 함께 클라이언트가 자동 전송하는 refresh token cookie를 서버가 읽어 revoke 처리한다.

**Response** `204 No Content`

```http
Set-Cookie: refreshToken=; Path=/; Max-Age=0; HttpOnly; Secure; SameSite=Lax
```

**Error**

- `REFRESH_TOKEN_INVALID`

---

### `GET /api/me`

> 현재 로그인한 사용자의 프로필 정보를 조회한다.

**Response** `200 OK`

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

**정책**

- 수정 가능 필드: `nickname`, `profile_image_url`, `status_message`
- `profile_image_url`은 저장된 `member.profile_image_s3_key`에서 파생한 접근 URL이며, 이미지가 없으면 null일 수 있다.
- `status_message`는 자유 입력 한 줄 상태 메시지다(최대 100자).
- `is_host_ever`, `hosted_crew_count`는 read-time 계산 projection이다.

---

### `PATCH /api/me/profile`

> 닉네임, 프로필 이미지, 상태 메시지를 수정한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `nickname` | `string` | N | 노출 이름 |
| `profile_image_s3_key` | `string \| null` | N | 프로필 이미지 S3 key. `null`이면 이미지 제거 |
| `status_message` | `string \| null` | N | 상태 메시지 (최대 100자). `null`이면 제거 |

**Response** `200 OK`

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

**Error**

- `VALIDATION_ERROR`
- `NICKNAME_ALREADY_EXISTS`
- `PROFILE_IMAGE_NOT_FOUND`

**정책**

- 세 필드 중 하나 이상이 요청에 포함되어야 한다.
- 프로필 이미지는 presigned upload로 먼저 업로드된 S3 key만 참조한다.

## 5.2 크루 / 참여


### `GET /api/crews`

> 크루 목록을 상태 필터로 조회한다.

**Query**

| 필드     | 타입      | 필수 | 설명                |
| -------- | --------- | ---- | ------------------- |
| `status` | `string`  | N    | 기본값 `RECRUITING` |
| `cursor` | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다. |
| `limit`  | `integer` | N    | 기본 20, 최대 100. |

**Response** `200 OK`

```json
{
  "items": [
    {
      "crew_id": 42,
      "title": "새벽 기상 챌린지",
      "image_url": null,
      "status": "RECRUITING",
      "deposit_amount": 100000,
      "min_participants": 2,
      "max_participants": 5,
      "frequency_type": "DAILY",
      "mission_schedule_days": [],
      "recruitment_deadline": "2026-05-09T23:59:59+09:00",
      "start_at": "2026-05-10T00:00:00+09:00",
      "activated_at": null,
      "end_at": "2026-05-31T23:59:59+09:00"
    }
  ],
  "next_cursor": null
}
```

**정책**

- `next_cursor`는 다음 slice가 존재할 때만 응답에 포함하며, 없거나 `null`이면 더 조회할 slice가 없다.

---

### `POST /api/crews`

> 새 크루를 생성한다.

**Request**

| 필드                    | 타입             | 필수 | 설명                                                    |
| ----------------------- | ---------------- | ---- | ------------------------------------------------------- |
| `title`                 | `string`         | Y    | 크루 제목                                               |
| `description`           | `string`         | Y    | 크루 설명                                               |
| `image_s3_key`          | `string \| null` | N    | 사전 업로드된 대표 이미지 S3 key. 표시용 metadata       |
| `category`              | `string`         | Y    | 카테고리                                                |
| `deposit_amount`        | `integer`        | Y    | 보증금 (1,000원 단위, 1,000 ~ 100,000원)                |
| `min_participants`      | `integer`        | N    | 최소 인원. 기본값 `2`                                   |
| `max_participants`      | `integer`        | Y    | 최대 인원 (최대 15)                                     |
| `frequency_type`        | `string`         | Y    | `DAILY` 또는 `SPECIFIC_DAYS`                            |
| `mission_schedule_days` | `string[]`       | N    | `SPECIFIC_DAYS`일 때 필수. 예: `["MONDAY","WEDNESDAY"]` |
| `daily_settlement_type` | `string`         | Y    | `A` (인증마감 09:00 / 정산 12:00), `B` (인증마감 21:00 / 정산 00:00), `C` (인증마감 23:59 / 정산 익일 12:00) |
| `host_agreement`        | `object`         | Y    | 방장 약관 동의 스냅샷 payload                           |
| `recruitment_deadline`  | `string`         | Y    | ISO-8601. 신규 참여 마감 시각                           |
| `start_date`            | `string`         | Y    | `YYYY-MM-DD`. 시작일                                    |
| `end_date`              | `string`         | Y    | `YYYY-MM-DD`. 종료일                                    |

**Response** `201 Created`

```json
{
  "crew_id": 42,
  "title": "새벽 기상 챌린지",
  "description": "매일 새벽 6시 전 기상 인증",
  "image_url": null,
  "category": "EXERCISE",
  "status": "RECRUITING",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "SPECIFIC_DAYS",
  "mission_schedule_days": ["MONDAY", "WEDNESDAY", "FRIDAY"],
  "daily_settlement_type": "A",
  "host_agreement_version": "v1",
  "host_agreed_at": "2026-05-07T09:00:00+09:00",
  "recruitment_deadline": "2026-05-09T23:59:59+09:00",
  "start_at": "2026-05-10T00:00:00+09:00",
  "activated_at": null,
  "end_at": "2026-05-31T23:59:59+09:00",
  "created_at": "2026-05-07T09:00:00+09:00",
  "my_participation": {
    "crew_participant_id": 100,
    "status": "LOCKED",
    "deposit_locked_amount": 100000,
    "locked_at": "2026-05-07T09:00:00+09:00"
  }
}
```

**Error**

- `VALIDATION_ERROR`
- `INVALID_DEPOSIT_AMOUNT`
- `INVALID_FREQUENCY_RULE`
- `INVALID_CATEGORY`
- `INVALID_DAILY_SETTLEMENT_TYPE`
- `HOST_AGREEMENT_REQUIRED`
- `INSUFFICIENT_BALANCE`

**정책**

- `2 <= min_participants <= max_participants <= 15`
- `start_date`, `end_date`는 서버에서 `Asia/Seoul` 기준 `start_at`, `end_at`으로 변환한다.
- `RECRUITING → ACTIVE` 전환은 `start_at`에 시스템이 자동으로 수행한다. host/admin manual 전환은 없다.
- 크루 생성 트랜잭션은 `crew` row insert와 함께 호스트용 `crew_participant` row를 `status=LOCKED`로 자동 생성하고, `CREW_DEPOSIT_RESERVE point_history` row를 함께 insert한다. 호스트는 별도 `POST /api/crews/{crewId}/participants` 신청 + 방장 승인 흐름을 거치지 않는다.
- host auto-created `LOCKED` participant의 보증금은 생성 트랜잭션에서 `point_account.available_balance -= crew.deposit_amount` / `locked_balance += crew.deposit_amount`로 직접 반영한다. host는 처음부터 `LOCKED`이므로 `PENDING` reserve bucket을 거치지 않는다.
- 호스트 잔액이 `crew.deposit_amount` 미만이면 lock 처리가 실패하므로 크루 생성 자체를 `INSUFFICIENT_BALANCE`로 거절한다. 호스트에게 별도 보증금 면제/예외는 없다.
- 호스트 auto-created `LOCKED` participant는 일반 `LOCKED` 참여자와 동일하게 capacity, `min_participants` baseline, activation eligibility, frozen participant baseline, settlement eligibility에 포함되며 최종 정산 대상이다.
- 호스트의 `CREW_DEPOSIT_RESERVE` 원장은 일반 신청 reserve와 동일한 `transaction_type`을 사용하지만 bucket destination은 `locked_balance`다. 별도 `HOST_*` enum/type을 만들지 않는다.
- 응답의 `my_participation`은 호스트 본인의 auto-created `LOCKED` participant snapshot이다.

---

### `GET /api/crews/{crewId}`

> 특정 크루의 상세 정보와 내 참여 현황을 조회한다.

**Response** `200 OK`

```json
{
  "crew_id": 42,
  "host_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
  "title": "새벽 기상 챌린지",
  "description": "매일 아침 6시 전에 인증",
  "image_url": null,
  "category": "EXERCISE",
  "status": "ACTIVE",
  "settlement_status": "NONE",
  "deposit_amount": 100000,
  "min_participants": 2,
  "max_participants": 5,
  "frequency_type": "DAILY",
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
    "locked_at": "2026-05-08T13:00:00+09:00"
  }
}
```

**Error**

- `CREW_NOT_FOUND`

**정책**

- `my_participation`은 참여 이력이 없으면 `null`이다.
- `settlement_status`는 Settlement-design §5.3의 crew settlement state projection이다. 정산 상태의 원천은 항상 `Settlement.status`다.

---

### `POST /api/crews/{crewId}/participants`

> 크루에 참여를 신청하고 보증금을 예약한다.

**Request** body 없음

**Response** `201 Created`

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

**Error**

- `CREW_NOT_FOUND`
- `CREW_NOT_RECRUITING`
- `CAPACITY_FULL`
- `INSUFFICIENT_BALANCE`
- `ALREADY_PARTICIPATING`
- `APPLICATION_NOT_ALLOWED`

**정책**

- `RECRUITING` 상태이고 `recruitment_deadline` 이전일 때만 신청 가능하다.
- 일반 참여 신청 시 `deposit_amount`만큼 `available_balance`를 차감해 `reserved_balance`로 reserve한다.
- `PENDING` 상태는 capacity에 포함하나 정산 대상은 아니다.
- `CANCELLED` 상태(자진 취소)에서는 재신청이 허용된다. 재신청 조건은 일반 신청과 동일하다: `crew.status = RECRUITING` + 서버 시간이 `recruitment_deadline` 전 + capacity 가능(`PENDING + LOCKED < max_participants`) + reserve 가능(`available_balance >= crew.deposit_amount`). 재신청 시 새 row를 생성하지 않고 기존 `crew_participant` row를 `CANCELLED → PENDING`으로 reopen한다(row resurrection / in-place reopen semantics). `unique(crew_id, member_id)` 제약은 그대로 유지되며 soft delete나 제약 완화 없이 기존 row를 그대로 재사용한다. reopen 시 `released_point_history_id`를 `null`로 reset하고 `pending_at`을 현재 시각으로 갱신한다. 보증금 reserve는 `point_history` append-only 방식으로 새 cycle을 추가하며, idempotency key는 `crew:{crewId}:participant:{participantId}:reserve:{cycle}` 형식으로 cycle별 구분한다.
- `REJECTED`, `EXPIRED` 상태에서 재신청은 `APPLICATION_NOT_ALLOWED`로 거절한다. MVP에서는 이 두 상태에서 재참여/row 재사용/status 되돌리기를 허용하지 않는다.
- 호스트는 자신이 생성한 크루에 대해 이 endpoint로 다시 신청하지 않는다. `POST /api/crews` 시점에 host용 `crew_participant` row가 이미 `LOCKED`로 auto-created되어 있으므로 호스트의 추가 신청 시도는 `ALREADY_PARTICIPATING`로 거절된다.

---

### `DELETE /api/crews/{crewId}/participants/me`

> 크루 참여 신청을 취소한다.

**Request** body 없음

**Response** `200 OK`

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "CANCELLED",
  "cancelled_at": "2026-05-08T14:00:00+09:00"
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `APPLICATION_NOT_CANCELLABLE`

**정책**

- `PENDING` 상태에서만 취소 가능하다.
- 취소 시 reserved 보증금은 `CREW_RESERVE_RELEASE` point_history로 반환하고, `point_account.available_balance`를 같은 금액만큼 복구한다.
- 취소(`CANCELLED`) 이후 동일 크루에 재신청할 수 있다. 재신청은 `POST /api/crews/{crewId}/participants`를 다시 호출하며, 기존 `crew_participant` row를 `CANCELLED → PENDING`으로 reopen한다. 새 row는 생성되지 않으며 `unique(crew_id, member_id)` 제약은 유지된다. `cancelled_at`은 reopen 시 갱신되는 `pending_at`과 별개로 audit 용도로 row에 남는다.

---

### `POST /api/crews/{crewId}/applications/{crewParticipantId}/approve`

> 방장이 크루 참여 신청을 승인한다.

**Request** body 없음

**Response** `200 OK`

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "LOCKED",
  "deposit_locked_amount": 100000,
  "locked_at": "2026-05-08T15:00:00+09:00"
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `APPLICATION_NOT_APPROVABLE`

**정책**

- 호출자가 해당 크루의 host여야 한다.
- `PENDING` 상태에서만 승인 가능하다. 다른 상태는 `APPLICATION_NOT_APPROVABLE`로 거절한다.
- 승인 시 기존 reserve를 `LOCKED`로 확정한다. 추가 잔액 차감은 없으며 새 `point_history`를 만들지 않는다(`reserved_balance → locked_balance` bucket transition만 수행).
- 이 endpoint는 일반 참여자의 `PENDING` row에만 사용한다. `POST /api/crews` 시점에 auto-created된 호스트 본인의 `LOCKED` row는 승인 대상이 아니며, 호출 시 `APPLICATION_NOT_APPROVABLE`로 거절한다.

---

### `POST /api/crews/{crewId}/applications/{crewParticipantId}/reject`

> 방장이 크루 참여 신청을 거절한다.

**Request** body 없음

**Response** `200 OK`

```json
{
  "crew_participant_id": 101,
  "crew_id": 42,
  "status": "REJECTED",
  "rejected_at": "2026-05-08T15:00:00+09:00"
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `APPLICATION_NOT_REJECTABLE`

**정책**

- 호출자가 해당 크루의 host여야 한다.
- `PENDING` 상태에서만 거절 가능하다. 다른 상태는 `APPLICATION_NOT_REJECTABLE`로 거절한다.
- 거절 시 reserved 보증금은 `CREW_RESERVE_RELEASE` point_history로 반환하고, `point_account.available_balance`를 같은 금액만큼 복구한다.
- 이 endpoint는 일반 참여자의 `PENDING` row에만 사용한다. `POST /api/crews` 시점에 auto-created된 호스트 본인의 `LOCKED` row는 거절 대상이 아니며, 호출 시 `APPLICATION_NOT_REJECTABLE`로 거절한다.

---

### `GET /api/crews/{crewId}/applications`

> 방장이 크루 가입 신청 목록을 조회한다.

**Query**

| 필드     | 타입      | 필수 | 설명                                                                       |
| -------- | --------- | ---- | -------------------------------------------------------------------------- |
| `status` | `string`  | N    | `PENDING`, `LOCKED`, `REJECTED`, `CANCELLED`, `EXPIRED`. 생략 시 `PENDING` |
| `cursor` | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다                          |
| `limit`  | `integer` | N    | 기본 50, 최대 200                                                          |

**Response** `200 OK`

```json
{
  "items": [
    {
      "crew_participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "nickname": "돈독러",
      "profile_image_url": null,
      "status": "PENDING",
      "applied_at": "2026-05-08T13:00:00+09:00",
      "decided_at": null
    }
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`

**정책**

- 호출자가 해당 크루의 host여야 한다.
- 기본 정렬은 `applied_at DESC`이며, `status` 필터를 생략하면 `PENDING` 목록만 반환한다.
- 이 API는 신청 처리를 위한 읽기 전용 조회 surface다. 승인/거절/취소는 별도 endpoint를 사용한다.

---

### `GET /api/crews/{crewId}/members`

> 크루 멤버 목록을 조회한다.

**Query**

| 필드     | 타입      | 필수 | 설명                                                              |
| -------- | --------- | ---- | ----------------------------------------------------------------- |
| `state`  | `string`  | N    | `ACTIVE` / `LOCKED`. 생략 시 `ACTIVE`. `ACTIVE`는 ParticipantStatus enum 값이 아니라 "active membership" alias이며 MVP에서는 `LOCKED` participant 집합을 의미한다. `LOCKED`는 ParticipantStatus enum 값을 직접 지정한다. |
| `cursor` | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다                 |
| `limit`  | `integer` | N    | 기본 50, 최대 200                                                 |

**Response** `200 OK`

```json
{
  "items": [
    {
      "crew_participant_id": 101,
      "member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "nickname": "돈독러",
      "profile_image_url": null,
      "role": "HOST",
      "status": "LOCKED",
      "joined_at": "2026-05-08T13:00:00+09:00"
    }
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `CREW_ACCESS_DENIED`

**정책**

- 해당 크루의 참여자 또는 호스트여야 한다.
- `role`은 `HOST` 또는 `MEMBER`이며, `crew.host_member_id` 매칭에서 파생한 projection이다. 권한 부여 컬럼이 아니다.
- `joined_at`은 `crew_participant.locked_at`에 해당하는 projection이다.
- 정산 결과(인정 횟수, 환급액 등) 필드는 포함하지 않는다. 정산 결과는 `GET /api/settlements/{settlementId}`로 조회한다.

---

### 보증금 Lifecycle 요약

크루 참여에서 정산까지 보증금 흐름의 핵심 규칙을 요약한다.

| 단계 | 처리 |
| --- | --- |
| 크루 생성 (`POST /api/crews`) | host auto-created `LOCKED` participant 생성 + `available_balance → locked_balance` bucket update + `CREW_DEPOSIT_RESERVE` point_history insert를 같은 트랜잭션에서 처리 |
| 일반 참여 신청 (`POST participants`) | `PENDING` row 생성과 함께 `deposit_amount` reserve (`available_balance → reserved_balance`) |
| 승인 (`approve`) | 추가 잔액 차감 없이 기존 reserve를 `LOCKED`로 확정 (`reserved_balance → locked_balance` bucket transition만 수행, 새 `point_history` row 생성 안 함) |
| 취소 / 거절 / 만료 | `CREW_RESERVE_RELEASE` point_history로 잔액 복구 (`reserved_balance → available_balance`)를 같은 트랜잭션에서 처리 |
| `CANCELLED → PENDING` reopen | 기존 release row를 append-only로 유지한 채 새 reserve cycle 추가. idempotency key는 `crew:{crewId}:participant:{participantId}:reserve:{cycle}` 형식으로 cycle별 구분 |

**재신청 가능 여부**

| 현재 상태 | 재신청 결과 |
| --- | --- |
| `PENDING` / `LOCKED` | `ALREADY_PARTICIPATING` |
| `CANCELLED` | reopen 가능 (`CANCELLED → PENDING`) |
| `REJECTED` / `EXPIRED` | `APPLICATION_NOT_ALLOWED` (terminal, MVP에서 재참여 불가) |
| host auto-created `LOCKED` row | reopen 대상 아님 |

**시스템 Lifecycle 전이**

- `RECRUITING → ACTIVE`: `start_at` 기준 시스템이 자동 전환. host/admin manual 전환 없음.

---

### 공지 / 댓글 / 리액션

> 크루 내 방장 공지, 댓글, 공지 리액션 communication surface를 제공한다. 상세 명세는 `notice.md` 참조.

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/api/crews/{crewId}/notices` | 공지 목록 조회 |
| `POST` | `/api/crews/{crewId}/notices` | 방장 공지 작성 |
| `PATCH` | `/api/crews/{crewId}/notices/{noticeId}` | 방장 공지 수정 |
| `DELETE` | `/api/crews/{crewId}/notices/{noticeId}` | 공지 표시 상태 삭제 |
| `GET` | `/api/crews/{crewId}/notices/{noticeId}/comments` | 공지 댓글 목록 |
| `POST` | `/api/crews/{crewId}/notices/{noticeId}/comments` | 공지 댓글 작성 |
| `PATCH` | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | 공지 댓글 수정 |
| `DELETE` | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | 댓글 표시 상태 삭제 |
| `POST` | `/api/crews/{crewId}/notices/{noticeId}/reactions` | 공지 리액션 멱등 upsert |
| `DELETE` | `/api/crews/{crewId}/notices/{noticeId}/reactions/me` | 내 공지 리액션 멱등 삭제 |

**정책**

- 공지 작성/수정 권한은 host 중심으로 검증한다.
- 공지 본문은 `crew`, `mission_rule`, `mission_log`, `settlement`, `point_history`의 canonical rule/state를 변경하지 않는다.
- 댓글과 공지 리액션은 social interaction only이며, 정산 인정 횟수, 환급액, 포인트 원장, 인증 성공/실패, lifecycle 전이에 side effect를 만들지 않는다.
- `reaction_type`은 FE-selected emoji/token string이며 고정 enum으로 freeze하지 않는다.
- 삭제 계열은 물리 삭제가 아니라 표시 상태 전이(`HIDDEN`/`DELETED`)를 우선한다.

## 5.3 크루 공지 / 댓글 / 리액션


> 채팅 없는 MVP에서 크루 내 소통 수단을 제공한다. 공지 본문은 `crew`, `mission_rule`, `mission_log`, `settlement`, `point_history`의 canonical rule/state를 변경하지 않는다.

### `GET /api/crews/{crewId}/notices`

> 크루의 공지 목록을 조회한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `cursor` | `string` | N | 이전 응답의 `next_cursor`로 다음 slice를 조회한다. |
| `limit` | `integer` | N | 기본 20, 최대 100. |

**Response** `200 OK`

```json
{
  "items": [
    {
      "notice_id": 1,
      "crew_id": 42,
      "author_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c901",
      "title": "이번 주 인증 안내",
      "content": "매일 오전 9시 전까지 인증해주세요.",
      "created_at": "2026-05-11T10:00:00+09:00"
    }
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `CREW_ACCESS_DENIED`

**정책**

- 최신순(`created_at DESC, notice_id DESC`) 정렬.
- `next_cursor`는 다음 slice가 존재할 때만 응답에 포함하며, 없거나 `null`이면 더 조회할 slice가 없다.

---

### `POST /api/crews/{crewId}/notices`

> 방장이 공지를 작성한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | `string` | Y | 공지 제목 |
| `content` | `string` | Y | 공지 내용 |

**Response** `201 Created`

**Error**

- `CREW_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`

---

### `PATCH /api/crews/{crewId}/notices/{noticeId}`

> 방장이 공지를 수정한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | `string` | N | 공지 제목 |
| `content` | `string` | N | 공지 내용 |

**Response** `200 OK`

**Error**

- `CREW_NOT_FOUND`
- `NOTICE_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`

---

### `DELETE /api/crews/{crewId}/notices/{noticeId}`

> 공지를 표시 상태(`HIDDEN`/`DELETED`)로 삭제 처리한다.

**Response** `200 OK`

**정책**

- 물리 삭제가 아니라 표시 상태 전이를 사용한다.

---

### `GET /api/crews/{crewId}/notices/{noticeId}/comments`

> 공지의 댓글 목록을 조회한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `cursor` | `string` | N | 이전 응답의 `next_cursor`로 다음 slice를 조회한다. |
| `limit` | `integer` | N | 기본 20, 최대 100. |

**Response** `200 OK`

```json
{
  "items": [
    {
      "comment_id": 101,
      "notice_id": 1,
      "author_member_uuid": "018f4fd2-6d7a-7a41-9f58-6d07f5c3c907",
      "nickname": "돈독러",
      "content": "확인했습니다!",
      "created_at": "2026-05-11T10:30:00+09:00"
    }
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `NOTICE_NOT_FOUND`

**정책**

- 오래된순(`created_at ASC, comment_id ASC`) 정렬.
- `next_cursor`는 다음 slice가 존재할 때만 응답에 포함하며, 없거나 `null`이면 더 조회할 slice가 없다.

---

### `POST /api/crews/{crewId}/notices/{noticeId}/comments`

> 공지에 댓글을 작성한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `content` | `string` | Y | 댓글 내용 |

**Response** `201 Created`

**Error**

- `CREW_NOT_FOUND`
- `NOTICE_NOT_FOUND`
- `CREW_ACCESS_DENIED`

---

### `PATCH /api/crews/{crewId}/notices/{noticeId}/comments/{commentId}`

> 본인이 작성한 댓글을 수정한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `content` | `string` | Y | 수정할 댓글 내용 |

**Response** `200 OK`

**Error**

- `COMMENT_NOT_FOUND`
- `FORBIDDEN`

---

### `DELETE /api/crews/{crewId}/notices/{noticeId}/comments/{commentId}`

> 댓글을 표시 상태(`HIDDEN`/`DELETED`)로 삭제 처리한다.

**Response** `200 OK`

**정책**

- 물리 삭제가 아니라 표시 상태 전이를 사용한다.

---

### `POST /api/crews/{crewId}/notices/{noticeId}/reactions`

> 공지에 이모지 리액션을 멱등 추가한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `reaction_type` | `string` | Y | emoji token 문자열 |

**Response** `200 OK`

```json
{
  "notice_id": 1,
  "my_reactions": ["👍"],
  "reaction_counts": { "👍": 3 }
}
```

**정책**

- `(notice_id, member_id, reaction_type)` 기준 멱등 upsert다.

---

### `DELETE /api/crews/{crewId}/notices/{noticeId}/reactions/me`

> 공지에 남긴 내 이모지 리액션을 멱등 삭제한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `reaction_type` | `string` | Y | 삭제할 emoji token (URL encoding 필요) |

**Response** `200 OK`

```json
{
  "notice_id": 1,
  "my_reactions": [],
  "reaction_counts": { "👍": 2 }
}
```

**정책**

- 리액션이 이미 없어도 성공 응답을 반환한다.

## 5.4 미션 인증 / 검수 / 이력


### `POST /api/uploads/presigned-url`

> S3 직접 업로드를 위한 presigned URL을 발급한다.

**Request**

| 필드                  | 타입      | 필수 | 설명                                                |
| --------------------- | --------- | ---- | --------------------------------------------------- |
| `purpose`             | `string`  | Y    | `MISSION_IMAGE`, `PROFILE_IMAGE`, 또는 `CREW_IMAGE` |
| `crew_id`             | `integer` | N    | 미션 이미지 업로드 시 대상 크루                     |
| `crew_participant_id` | `integer` | N    | 미션 이미지 업로드 시 대상 참여자                   |
| `content_type`        | `string`  | Y    | 이미지 content type                                 |
| `content_length`      | `integer` | Y    | 파일 크기 (bytes)                                   |

**Response** `200 OK`

| 필드         | 타입     | 설명                          |
| ------------ | -------- | ----------------------------- |
| `upload_url` | `string` | 단기 TTL presigned upload URL |
| `s3_key`     | `string` | 서버가 생성한 S3 object key   |
| `expires_at` | `string` | URL 만료 시각                 |

**Error**

- `ALREADY_CERTIFIED_TODAY`
- `CERTIFICATION_IN_REVIEW`

**정책**

- S3 object key는 서버가 생성한다. 클라이언트가 임의 path를 지정할 수 없다.
- S3 bucket/object는 private이다. presigned URL은 upload delegation 수단이지 validation delegation 수단이 아니다.
- 이미지 종류별 권장 key 형식:
  - 미션 이미지: `mission/{crewId}/{crewParticipantId}/{uuid}` (`crewParticipantId`는 `crew_participant.id`)
  - 프로필 이미지: `profile/{memberUuid}/{uuid}`
  - 크루 대표 이미지: `crew/{memberUuid}/{uuid}` (`memberUuid`는 발급 요청자, 즉 미래의 호스트)
- `purpose = CREW_IMAGE`는 크루 생성/수정 흐름에서 사용되며 `crew_id`/`crew_participant_id`를 요구하지 않는다. 발급 자체는 settlement/lifecycle authority가 아니다.
- 발급 시점에 사용자/크루/참여자 권한을 검증한다.
- `purpose = MISSION_IMAGE`인 경우, 당일(`server_time` 기준 `Asia/Seoul` 날짜의 cadence slot) `certification_status = SUCCESS`인 인증 로그가 존재하면 `ALREADY_CERTIFIED_TODAY`, `PENDING_REVIEW`인 인증 로그가 존재하면 `CERTIFICATION_IN_REVIEW`를 반환한다.
- **이 검사는 UX pre-check(최선 시도)이며 certification authority가 아니다.** 경쟁 상태나 presigned 발급 이후 상태 변경으로 인해 pre-check를 통과한 요청이 `POST /api/mission-logs`에서 거절될 수 있다. 최종 authoritative guard는 `POST /api/mission-logs`다.
- 클라이언트는 발급받은 URL로 S3에 직접 업로드한 뒤, `s3_key`로 미션 로그 생성을 요청한다.

---

### `POST /api/mission-logs`

> 업로드된 이미지로 미션 인증 로그를 생성한다.

**Request**

| 필드           | 타입      | 필수 | 설명                        |
| -------------- | --------- | ---- | --------------------------- |
| `crew_id`      | `integer` | Y    | 대상 크루                   |
| `image_s3_key` | `string`  | Y    | 업로드 완료된 이미지 S3 key |
| `caption`      | `string`  | Y    | 인증 텍스트 (5~100자)       |

**Response** `201 Created`

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
  "reject_reason_code": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `PARTICIPANT_NOT_ELIGIBLE`
- `ALREADY_CERTIFIED_TODAY`
- `CERTIFICATION_IN_REVIEW`
- `NOT_MISSION_DAY`

**정책**

- 인증 시점에는 crew 단위 Redisson 락을 기본으로 사용하지 않는다.
- 인증은 `MissionLog` 원본 보존이 우선이다.
- `LOCKED` 상태인 참여자만 인증을 제출할 수 있다. `PENDING` 등 비`LOCKED` 상태에서는 `PARTICIPANT_NOT_ELIGIBLE`을 반환한다.
- `SPECIFIC_DAYS` 크루에서 `server_time` 기준 해당 요일이 `mission_schedule_days`에 포함되지 않으면 `NOT_MISSION_DAY`를 반환하고 제출 자체를 거절한다. 로그를 생성한 뒤 정산에서 exclude하는 방식은 사용하지 않는다.
- **재업로드 정책**: 당일(`server_time` 기준 `Asia/Seoul` 날짜의 cadence slot) 인증 상태에 따라 아래와 같이 처리한다.
  - `SUCCESS` 로그가 있으면 `ALREADY_CERTIFIED_TODAY`를 반환하고 거절한다. `ALREADY_CERTIFIED_TODAY`는 `DAILY`/`SPECIFIC_DAYS` 구분 없이 동일 cadence slot의 기인증 완료 상태를 의미한다.
  - `PENDING_REVIEW` 로그가 있으면 `CERTIFICATION_IN_REVIEW`를 반환하고 거절한다.
  - `FAILED` 로그만 있으면 재업로드를 허용한다.
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
- `certification_status`는 인증 요청의 resolved certification state를 나타내며, 최종 정산 인정 여부를 보장하지 않는다.
- `certification_status` 결정 시 아래 조건을 검토한다.
  - 업로드 object의 소유/범위/기본 무결성
  - EXIF/hash risk signal과 review 필요 여부
  - 미션 기간 내 요청 여부 (`server_time` 기준)
  - frozen baseline / participant 상태 적합성
- `certification_status = SUCCESS`는 인증 성공 표시이며, 최종 정산에서 인정된다는 의미는 아니다.
- `certification_status = FAILED`여도 원본 로그는 저장할 수 있다.
- `certification_status = PENDING_REVIEW`는 업로드 직후 검수/판정 대기 상태다.
- `certification_status`는 인증 피드 badge, dashboard projection, 알림 input에 쓰이는 resolved state이며 EXIF/hash raw signal이나 host moderation `decision_type`/`reject_reason_code`와 동일 axis로 해석하지 않는다.
- `mission_log.failure_reason`은 인증 시점 실패 사유(system/timing axis)다.
- `decision_type`, `reject_reason_code`는 호스트 검수자 결과 axis이며 시스템 `failure_reason`과 의미 vocabulary가 다르다.
- POST 응답에서 `decision_type`, `reject_reason_code`는 검수가 일어나지 않은 시점에는 `null`이다. `reject_memo`는 participant-facing 응답 필드가 아니다.
- `settlement_item.calculation_reason`은 정산 시점 포함/제외 근거다.
- MVP 인증 API에서 `OUT_OF_SCHEDULE`는 사용하지 않는다.
- 최종 정산에서의 인정 여부는 `certification_status`가 아니라 Settlement 계산 단계에서 결정된다.
- 인증은 성공했지만 정산에서 제외되는 경우(예: 동일 cadence slot 내 중복 인정)는 `mission_log.failure_reason`이 아니라 `settlement_item.calculation_reason`으로만 표현한다. `SPECIFIC_DAYS` 비해당 요일은 제출 시점에 `NOT_MISSION_DAY`로 거절되므로 settlement exclude 대상이 아니다.
- 인증 시점 성공 로그도 최종 정산에서 제외될 수 있다.

---

### `GET /api/crews/{crewId}/mission-logs/me`

> 특정 크루에서 내 미션 인증 로그 목록을 조회한다.

**Query**

| 필드     | 타입      | 필수 | 설명 |
| -------- | --------- | ---- | ---- |
| `cursor` | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다. |
| `limit`  | `integer` | N    | 기본 20, 최대 100. |

**Response** `200 OK`

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
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`

**정책**

- 이 API는 원시 인증 기록 조회용이다.
- 정산 인정 판단 기준 시간은 `MissionLog.server_time`이다.
- `exif_taken_at`은 서버가 S3 object에서 추출/검증한 촬영 시각 보조 정보이며, 최종 정산 인정 시각 기준으로 사용하지 않는다.
- `image_hash`는 서버 계산 SHA-256 결과의 read-only 노출이며, 동일 인증 사진 중복 의심 신호일 뿐 authority가 아니다.
- `certification_status`는 인증 요청의 resolved certification state(`PENDING_REVIEW`/`SUCCESS`/`FAILED`)이며, 정산에서 인정된 횟수를 나타내는 값이 아니다.
- `decision_type`, `reject_reason_code`는 현재 latest-effective 검수 결과 projection이다. 참여자-facing 응답은 `reject_reason_code`만 제공하고 `reject_memo`를 포함하지 않는다. `reject_memo`는 internal/private context다.
- FE는 이 값을 `최종 성공 횟수` 또는 `정산 인정 횟수`로 사용하면 안 된다.
- 최종 인정 여부와 인정 횟수는 반드시 정산 결과 API `GET /api/settlements/{settlementId}`를 기준으로 판단해야 한다.

---

### `GET /api/crews/{crewId}/moderation-logs`

> 크루 내 인증 검수 이력을 조회한다.

**Query**

| 필드             | 타입      | 필수 | 설명                |
| ---------------- | --------- | ---- | ------------------- |
| `mission_log_id` | `integer` | N    | 특정 인증 로그 필터 |
| `cursor`         | `string`  | N    | 페이지네이션 커서   |
| `limit`          | `integer` | N    | 기본 50, 최대 200   |

**Response** `200 OK`

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

**Error**

- `CREW_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`

**정책**

- 본 API는 읽기 전용 감사 조회 전용이다. 검수 결정을 새로 만들거나 수정하지 않는다.
- `moderation_history`는 추가만 가능하다. 본 API는 기존 레코드를 변경/삭제하지 않는다.
- 호출자가 해당 크루의 host여야 한다. host가 아니면 `FORBIDDEN_NOT_HOST`를 반환한다.
- 크루원이 본인의 검수 결과를 확인할 때는 `GET /api/me/verification-history`를 사용한다.
- `decision_type`은 `MANUAL_APPROVE`, `MANUAL_REJECT`, `AUTO_APPROVE`, `AUTO_REJECT`만 사용한다.
- `reject_reason_code`는 `TIME_VIOLATION`, `DUPLICATE`, `MISSION_MISMATCH`, `UNCLEAR`, `INAPPROPRIATE`, `OTHER`만 사용한다.
- `reject_memo`는 일반적으로 nullable이지만 `OTHER`일 때 필수이며 최대 50자다. 내부 전용 컨텍스트이므로 참여자 응답에는 포함하지 않는다. `OTHER`여도 참여자는 raw memo text가 아니라 `reject_reason_code`만 받는다.
- `before_state`, `after_state`는 검수 결정 시점의 최신 유효 스냅샷 JSON이다. 정산 결과를 재계산하는 입력으로 사용하지 않는다.
- 검수자 식별은 `moderator_member_uuid`로만 노출한다. 내부 FK `moderator_id`는 응답에 포함하지 않는다.
- 이 API는 운영 관리자 권한 엔드포인트가 아니다. MVP에서는 관리자 수정 워크플로를 추가하지 않는다.

---

### `POST /api/mission-logs/{missionLogId}/moderation/approve`

> 방장이 검수 대기 중인 인증을 승인한다.

**Request** body 없음

**Response** `200 OK`

```json
{
  "mission_log_id": 9001,
  "crew_id": 42,
  "crew_participant_id": 101,
  "certification_status": "SUCCESS",
  "decision_type": "MANUAL_APPROVE",
  "reject_reason_code": null,
  "decided_at": "2026-05-12T11:00:00+09:00",
  "moderation_history_id": 7001
}
```

**Error**

- `MISSION_LOG_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `MISSION_LOG_NOT_REVIEWABLE`
- `SETTLEMENT_INPUT_FROZEN`

**정책**

- 호출자가 해당 미션 로그가 속한 크루의 host여야 한다.
- `PENDING_REVIEW` 상태인 인증 로그만 승인 가능하다. 이미 `SUCCESS`/`FAILED`이면 `MISSION_LOG_NOT_REVIEWABLE`로 거절한다.
- 정산 입력이 freeze된 이후에는 `SETTLEMENT_INPUT_FROZEN`으로 거절한다.
- 승인 시 `certification_status`가 `SUCCESS`로 전환된다.
- `moderation_history`에 `MANUAL_APPROVE` 기록이 append-only로 추가된다.
- 이 API는 settlement 재계산을 trigger하지 않는다. `point_history`, `settlement_item`을 직접 변경하지 않는다.

---

### `POST /api/mission-logs/{missionLogId}/moderation/reject`

> 방장이 검수 대기 중인 인증을 거절한다.

**Request**

| 필드                 | 타입     | 필수 | 설명                                                                                           |
| -------------------- | -------- | ---- | ---------------------------------------------------------------------------------------------- |
| `reject_reason_code` | `string` | Y    | `TIME_VIOLATION`, `DUPLICATE`, `MISSION_MISMATCH`, `UNCLEAR`, `INAPPROPRIATE`, `OTHER` 중 하나 |
| `reject_memo`        | `string` | N    | 거절 사유 메모. `OTHER`일 때 필수, 최대 50자                                                   |

**Response** `200 OK`

```json
{
  "mission_log_id": 9001,
  "crew_id": 42,
  "crew_participant_id": 101,
  "certification_status": "FAILED",
  "decision_type": "MANUAL_REJECT",
  "reject_reason_code": "MISSION_MISMATCH",
  "decided_at": "2026-05-12T11:10:00+09:00",
  "moderation_history_id": 7002
}
```

**Error**

- `MISSION_LOG_NOT_FOUND`
- `FORBIDDEN_NOT_HOST`
- `MISSION_LOG_NOT_REVIEWABLE`
- `SETTLEMENT_INPUT_FROZEN`
- `INVALID_REJECT_REASON_CODE`
- `REJECT_MEMO_REQUIRED`
- `REJECT_MEMO_TOO_LONG`

**정책**

- 호출자가 해당 미션 로그가 속한 크루의 host여야 한다.
- `PENDING_REVIEW` 상태인 인증 로그만 거절 가능하다. 이미 `SUCCESS`/`FAILED`이면 `MISSION_LOG_NOT_REVIEWABLE`로 거절한다.
- 정산 입력이 freeze된 이후에는 `SETTLEMENT_INPUT_FROZEN`으로 거절한다.
- `reject_reason_code`는 `TIME_VIOLATION`, `DUPLICATE`, `MISSION_MISMATCH`, `UNCLEAR`, `INAPPROPRIATE`, `OTHER` 중 하나여야 한다. 그 외 값은 `INVALID_REJECT_REASON_CODE`로 거절한다.
- `reject_reason_code = OTHER`이면 `reject_memo`가 필수이며 최대 50자다. 누락 시 `REJECT_MEMO_REQUIRED`, 초과 시 `REJECT_MEMO_TOO_LONG`을 반환한다.
- `reject_memo`는 내부 기록용이며 참여자 응답에는 포함하지 않는다. 참여자는 `reject_reason_code`만 확인할 수 있다.
- `moderation_history`에 `MANUAL_REJECT` 기록이 append-only로 추가된다.
- 이 API는 settlement 재계산을 trigger하지 않는다. 거절은 인증 입력 결정이며 즉시 환급/원장 변경이 아니다.

---

### `GET /api/me/verification-history`

> 내 미션 인증 검증 결과 현황을 조회한다. 내가 제출한 인증들의 상태(`PENDING_REVIEW`, `SUCCESS`, `FAILED`)를 크루별로 확인할 수 있다.

**Query**

| 필드      | 타입      | 필수 | 설명                                              |
| --------- | --------- | ---- | ------------------------------------------------- |
| `crew_id` | `integer` | N    | 특정 크루로 범위를 좁힌다                         |
| `role`    | `string`  | N    | `participant` 또는 `host`. 생략 시 `participant`  |
| `status`  | `string`  | N    | `PENDING_REVIEW`, `SUCCESS`, `FAILED` 필터        |
| `cursor`  | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다 |

**Response** `200 OK`

```json
{
  "items": [
    {
      "verification_history_item_id": "participant:9001",
      "perspective": "participant",
      "crew_id": 42,
      "crew_title": "새벽 기상 챌린지",
      "mission_log_id": 9001,
      "occurred_at": "2026-05-11T05:58:10+09:00",
      "verification_status": "SUCCESS",
      "reject_reason_code": null,
      "signal_summary": {
        "exif": "NORMAL",
        "reviewer": "HOST"
      },
      "links": {
        "feed": "/api/crews/42/feed",
        "settlement": null
      }
    }
  ],
  "next_cursor": null
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `FORBIDDEN`

**정책**

- 기본(`role` 생략 또는 `role = participant`)은 본인이 제출한 인증 로그 summary만 반환한다.
- `role = host`이면 본인이 방장으로 검수한 활동 summary를 반환한다. 호스트가 아닌 크루에 대한 정보는 포함하지 않는다.
- `role = host&crew_id={id}`로 조회 시 해당 크루의 host가 아니면 `FORBIDDEN`을 반환한다.
- `reject_reason_code`는 participant-facing 거절 사유이며, `reject_memo`는 internal/private context이므로 응답에 포함하지 않는다.
- `verification_status`는 현재 resolved 상태(`PENDING_REVIEW`/`SUCCESS`/`FAILED`)다. `SUCCESS`는 인증 상태 요약이지 최종 정산 인정을 보장하지 않는다.

---

### `GET /api/me/mission-feed`

> 내 크루별 인증 활동 타임라인을 조회한다. 내가 참여 중인 모든 크루의 인증 기록을 최신순으로 조회한다.

**Query**

| 필드      | 타입      | 필수 | 설명                                                                          |
| --------- | --------- | ---- | ----------------------------------------------------------------------------- |
| `crew_id` | `integer` | N    | 특정 크루로 범위를 좁힌다. 호출자가 참여자인 크루여야 한다                    |
| `status`  | `string`  | N    | `PENDING_REVIEW`, `SUCCESS`, `FAILED` 필터. `NOT_SUBMITTED`는 허용하지 않는다 |
| `cursor`  | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다                             |
| `limit`   | `integer` | N    | 기본 20, 최대 100                                                             |

**Response** `200 OK`

```json
{
  "items": [
    {
      "mission_log_id": 9101,
      "crew_id": 42,
      "crew_title": "새벽 기상 챌린지",
      "crew_participant_id": 101,
      "image_url": "https://cdn.example.com/mission/9101.jpg",
      "caption": "오늘 미션 인증합니다",
      "server_time": "2026-05-12T06:05:00+09:00",
      "certification_status": "SUCCESS",
      "reject_reason_code": null,
      "reaction_counts": { "👏": 2, "🔥": 1 },
      "my_reactions": ["👏"],
      "links": {
        "crew_feed": "/api/crews/42/feed"
      }
    },
    {
      "mission_log_id": 9003,
      "crew_id": 42,
      "crew_title": "새벽 기상 챌린지",
      "crew_participant_id": 101,
      "image_url": "https://cdn.example.com/mission/9003.jpg",
      "caption": "재업로드 후 다시 검토를 기다리고 있습니다",
      "server_time": "2026-05-11T07:10:02+09:00",
      "certification_status": "PENDING_REVIEW",
      "reject_reason_code": null,
      "reaction_counts": {},
      "my_reactions": [],
      "links": {
        "crew_feed": "/api/crews/42/feed"
      }
    }
  ],
  "next_cursor": "2026-05-11T07:10:02+09:00_9003"
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `INVALID_FEED_STATUS_FILTER`

**정책**

- 본인이 제출한 인증 로그만 포함된다. cross-crew append-only mission activity timeline이다.
- 모든 `certification_status`(`PENDING_REVIEW`, `SUCCESS`, `FAILED`)가 기본 포함되며, `status` 파라미터로 필터링할 수 있다.
- `NOT_SUBMITTED`는 `mission_log` row가 없는 synthetic slot projection이므로 이 API에 포함하지 않는다.
- 최신순(`server_time DESC`) 정렬이며, 같은 날짜에 여러 시도가 있으면 각 시도가 별도 item으로 노출된다. (`FAILED`/`PENDING_REVIEW` 재업로드 이력 보존)
- `reaction_counts`와 `my_reactions`는 `GET /api/crews/{crewId}/feed`와 동일한 범위로 제공된다. `certification_status = SUCCESS`인 item에만 값이 있으며, `FAILED`/`PENDING_REVIEW` item은 빈 map/빈 list로 응답된다.
- `reject_reason_code`는 participant-facing 거절 사유이며, `reject_memo`는 internal/private context이므로 응답에 포함하지 않는다.
- `SUCCESS` item이라도 최종 정산 인정을 보장하지 않는다. 최종 인정 여부는 `settlement_item.calculation_reason`을 기준으로 판단한다.

## 5.5 피드 / 미션 로그 리액션


### `GET /api/crews/{crewId}/feed`

> 크루 피드(인증 사진 목록)와 일별 미션 현황을 조회한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `limit` | `integer` | N | feed_items 페이지 크기. 기본 20 |
| `cursor` | `string` | N | 페이지네이션 커서 |
| `status` | `string` | N | `PENDING_REVIEW`, `SUCCESS`, `FAILED` 필터. 생략 시 전체 |
| `from` | `string` | N | 파생 상태 조회 시작일 `YYYY-MM-DD` |
| `to` | `string` | N | 파생 상태 조회 종료일 `YYYY-MM-DD` |

**Response** `200 OK`

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
      "reject_reason_code": null,
      "reaction_counts": { "👏": 2, "🔥": 1 },
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
    }
  ]
}
```

**Error**

- `CREW_NOT_FOUND`
- `CREW_ACCESS_DENIED`
- `INVALID_FEED_STATUS_FILTER`

**정책**

- `feed_items[]`는 실제 `mission_log` row가 있는 인증 활동을 append-only stream으로 노출한다. 기본 status set은 `SUCCESS`, `PENDING_REVIEW`, `FAILED`다. `status` 쿼리 파라미터로 특정 상태만 필터링할 수 있다.
- `day_statuses[]`와 `participant_day_slots[]`는 참여자/일자 표시용 latest/effective summary다. `NOT_SUBMITTED`는 `mission_log` row가 없는 synthetic slot projection이며 feed item이 아니다.
- 같은 참여자/같은 날짜/cadence slot에 여러 `mission_log` row가 생기는 경우는 `FAILED`/`PENDING_REVIEW` 재업로드와 host moderation 상태 전이로 한정된다. 이전 `FAILED`/`PENDING_REVIEW` item도 visible item으로 유지하며, 삭제/overwrite로 정산 입력을 바꾸지 않는다.
- `reject_reason_code`는 호스트 검수 거절 사유이며, 해당 없으면 `null`이다. `reject_memo`는 internal/private context이므로 feed 응답에 포함하지 않는다.
- 참여자/일자 summary 대표 규칙:
    - 성공 로그(`SUCCESS`)가 하나 이상 있으면 `SUCCESS`. 대표 로그는 가장 이른 `server_time`, 동률이면 가장 낮은 `mission_log.id`.
    - 성공 로그가 없고 검수 대기 로그(`PENDING_REVIEW`)가 있으면 `PENDING_REVIEW`.
    - 성공/검수 대기 로그가 없고 실패 로그(`FAILED`)가 하나 이상 있으면 `FAILED`.
    - 어떤 로그도 없으면 `NOT_SUBMITTED`.
- `server_time`은 서버가 인증 요청을 수신한 시각으로 **인증/정산 인정 timing anchor**다. `created_at`은 row 생성/feed 정렬/페이지네이션 보조 시각으로 두 값은 의미 axis가 다르다. 정산 인정 시각 기준은 항상 `MissionLog.server_time`이다.
- `caption`은 피드 표시용이며 단독 인증/정산 기준이 아니다.
- `reaction_counts`는 `mission_log_reaction`에서 파생한다. `mission_log`에 저장 카운터를 두거나 갱신하지 않는다. `reaction_counts`는 emoji token을 key로 하는 동적 map이다.
- MVP에서 리액션은 `certification_status = SUCCESS`인 로그에만 허용한다. `FAILED`/`PENDING_REVIEW` item의 `reaction_counts`는 빈 map, `my_reactions`는 빈 list로 응답한다.
- 피드 성공 여부는 정산 인정 여부를 보장하지 않는다. 정산 포함 여부는 `settlement_item.calculation_reason`이 결정한다.
- 이 API의 상태 projection은 정산 인정 횟수, 환급액, 포인트 잔액, lifecycle status의 기준이 아니다.

---

### `POST /api/mission-logs/{missionLogId}/reactions`

> 미션 인증 로그에 이모지 리액션을 추가한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `reaction_type` | `string` | Y | emoji token 문자열 |

**Response** `200 OK`

```json
{
  "mission_log_id": 9001,
  "my_reactions": ["👏", "🔥"],
  "reaction_counts": { "👏": 3, "🔥": 1 }
}
```

**Error**

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`
- `INVALID_REACTION_TYPE`

**정책**

- 리액션 대상은 `certification_status = SUCCESS`인 로그로 제한한다.
- `(mission_log_id, member_id, reaction_type)` 기준 멱등 upsert다. 같은 emoji token이 이미 있으면 동일 token 단위로 멱등 처리하고, 다른 emoji token은 별도 row로 공존할 수 있다.
- `(mission_log_id, member_id, reaction_type)` unique constraint 기반의 DB 레벨 멱등성을 보장해야 한다.
- 동일 `(mission_log_id, member_id, reaction_type)`에 대한 동시 중복 요청은 API 에러가 되어서는 안 되며, 최종 상태는 해당 token 1개 존재로 수렴해야 한다.
- 한 회원이 같은 로그에 여러 emoji token을 남길 수 있으나, 동일 token은 1회만 허용한다.
- FE가 선택한 emoji token을 서버가 다른 값으로 변환하지 않는다. trim 후 blank 거부, `VARCHAR(20)` 길이 검증만 수행한다. NFC/NFD 정규화, variation selector collapsing, ZWJ/skin-tone 동등성 정규화는 MVP에서 적용하지 않는다.
- 리액션 생성/수정은 `mission_log`를 변경하지 않는다.
- 리액션은 정산, 환급, 포인트 원장, 크루/참여/정산 상태 전이에 영향을 주지 않는다.

---

### `DELETE /api/mission-logs/{missionLogId}/reactions/me`

> 내 이모지 리액션을 삭제한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `reaction_type` | `string` | Y | 삭제할 emoji token (URL encoding 필요) |

**Response** `200 OK`

```json
{
  "mission_log_id": 9001,
  "my_reactions": ["🔥"],
  "reaction_counts": { "👏": 2, "🔥": 1 }
}
```

**Error**

- `MISSION_LOG_NOT_FOUND`
- `REACTION_NOT_ALLOWED`

**정책**

- 리액션이 이미 없어도 성공 응답을 반환한다.
- `(mission_log_id, member_id, reaction_type)` 기준 멱등 삭제다. 같은 저장 문자열만 삭제하며 다른 emoji token row는 유지한다.
- `reaction_type` query parameter는 필수다. 클라이언트는 emoji token을 URL encoding해서 전송해야 하며, 서버는 POST와 같은 trim/blank/length 검증을 적용한다.
- 삭제도 `mission_log` 원본, 정산, 환급, 포인트 원장, 상태 생명주기에 영향을 주지 않는다.

## 5.6 대시보드


### `GET /api/crews/{crewId}/dashboard`

> 내 보증금 현황, 예상 환급액, 랭킹 등 크루 대시보드 정보를 조회한다.

**Response** `200 OK`

```json
{
  "crew_id": 42,
  "crew_participant_id": 101,
  "settlement_id": null,
  "crew_status": "ACTIVE",
  "settlement_status": "NONE",
  "projection_status": "LIVE",
  "projection_notice": "ESTIMATED_NOT_FINAL",
  "my_deposit_amount": 100000,
  "my_success_count": 5,
  "my_recognized_success_count_estimated": 4,
  "total_recognized_success_count_estimated": 31,
  "my_share_ratio_estimated": "0.12903200",
  "my_expected_refund_amount": 103226,
  "my_expected_refund_delta_amount": 3226,
  "rank_estimated": 3,
  "updated_at": "2026-05-11T00:00:00+09:00"
}
```

**Error**

- `CREW_NOT_FOUND`
- `PARTICIPANT_NOT_FOUND`
- `CREW_ACCESS_DENIED`

**ProjectionStatus**

| 값 | 설명 |
|----|------|
| `NOT_STARTED` | `RECRUITING` 등 미션 수행 전 상태라 진행/환급 projection이 아직 시작되지 않았다 |
| `LIVE` | `ACTIVE` 상태에서 현재 `MissionLog`와 참여자 상태를 기준으로 current-basis estimate를 계산했다 |
| `CLOSED_ESTIMATE` | `CLOSED` 상태에서 매 요청 시 현재 확인 가능한 입력으로 current-basis estimate를 계산한다. 저장된 dashboard snapshot이 아니며 최종값도 아니다 |
| `NOT_PROVIDED` | `CANCELLED` 등 수행 projection을 제공하지 않는 상태다. 환급/정산 안내는 Settlement API 기준이다 |
| `SETTLEMENT_SUCCEEDED` | 최종 정산이 성공했다. Dashboard는 최종값을 복제하지 않고 `settlement_id`로 Settlement API 조회를 유도한다 |

**ProjectionNotice**

| 값 | 설명 |
|----|------|
| `ESTIMATED_NOT_FINAL` | 현재 값은 참고용 current-basis estimate이며 최종 정산 결과가 아니다 |
| `NOT_STARTED` | 미션 수행 전이라 성과/보상 projection이 아직 시작되지 않았다 |
| `NOT_PROVIDED` | 현재 방 상태에서는 Dashboard 진행/환급 projection을 제공하지 않는다 |
| `SETTLEMENT_RESULT_AVAILABLE` | 최종 정산 결과가 존재하므로 `settlement_id`로 Settlement API를 조회해야 한다 |
| `INSUFFICIENT_PROJECTION_INPUT` | projection 계산에 필요한 참여자/보증금 입력을 충분히 확정할 수 없어 일부 추정 필드를 `null`로 반환한다 |

**상태별 필드 계약**

| `projection_status` | `settlement_id` | `my_success_count` | 추정 필드들 (`my_recognized_*`, `share_ratio`, `refund`, `rank`) | `updated_at` |
|---|---|---|---|---|
| `NOT_STARTED` | `null` | `0` | 모두 `null` | value |
| `LIVE` | nullable | value | value 또는 `null` | value |
| `CLOSED_ESTIMATE` | nullable | value | value 또는 `null` | value |
| `NOT_PROVIDED` | nullable | `null` | 모두 `null` | value |
| `SETTLEMENT_SUCCEEDED` | value | `null` | 모두 `null` | value |

- `LIVE` / `CLOSED_ESTIMATE`에서 denominator 등 필수 projection 입력이 부족하면 해당 추정 필드는 `null`이고 `projection_notice = INSUFFICIENT_PROJECTION_INPUT`을 사용한다.
- `SETTLEMENT_SUCCEEDED`에서 추정 필드를 `null`로 내려주는 이유는 데이터가 없어서가 아니라 최종값의 source of truth가 `GET /api/settlements/{settlementId}`이기 때문이다.

**응답 필드 설명**

| 필드 | 설명 |
|------|------|
| `my_success_count` | latest/effective slot summary 기준의 현재 성공 표시 수. 일반 feed item 수나 정산 인정 성공 수가 아니다 |
| `my_recognized_success_count_estimated` | 현재 시점에서 정산 규칙을 가능한 범위로 반영한 추정 인정 성공 수 |
| `total_recognized_success_count_estimated` | 참여자별 추정 인정 성공 수 합계 |
| `my_share_ratio_estimated` | 전체 인정 성공 중 내 비율. 소수점 정밀도 오해 방지를 위해 string decimal로 반환 |
| `my_expected_refund_amount` | `total_recognized_success_count_estimated > 0`이면 `FLOOR(total_locked_amount × my_share_ratio_estimated)`. 0인 경우 all-fail equal-principal refund로 `my_deposit_amount` 반환 |
| `my_expected_refund_delta_amount` | `my_expected_refund_amount - my_deposit_amount`. 수익 확정값이 아닌 현재 기준 설명용 차이값 |
| `rank_estimated` | 추정 수행/참여도 표시 순서. `recognized_success_count_estimated DESC`, 동률이면 `crew_participant_id ASC` 기준 |
| `updated_at` | Dashboard 응답을 생성한 시각. 참여자 상태 변경 시각이나 최근 미션 로그 제출 시각이 아니다 |

**Projection source 역할**

| Source | Dashboard에서의 역할 |
|--------|---------------------|
| `mission_log` | 성공 후보와 수행 현황의 primary event source. `certification_status = 'SUCCESS'` 로그만 후보로 사용하고, 인정 판단 시간은 `MissionLog.server_time` 기준이다 |
| `crew_participant` | 참여자 식별, frozen `LOCKED` baseline, `deposit_amount` 보증금 금액 source다 |
| `crew` | 방 상태, 기간, 미션 주기/규칙 컨텍스트다. 총 보증금 source가 아니다 |
| `settlement` | `SUCCEEDED` 여부와 최종값 전환 판단용이다. `SUCCEEDED` 전 Dashboard projection 계산 source가 아니다 |
| `point_history` | Dashboard projection 계산 source가 아니다. 최종 환급/잔액은 Settlement API와 `point_history` 기준이다 |

**`locked_balance`와의 관계**

- `GET /api/points`의 `locked_balance`는 계정 단위 현재 잠긴 보증금 UX projection이다.
- Dashboard의 `my_expected_refund_amount`는 특정 crew/crew_participant 기준 예상 환급금 projection이다.
- FE는 `locked_balance`, `available_balance`, `my_expected_refund_amount`를 합산하거나 차감해 최종 보유 포인트, 출금 가능 금액, 최종 정산 후 확정 환급금을 계산하면 안 된다.
- 최종 환급 여부와 금액은 `Settlement.status = SUCCEEDED` 이후 Settlement API와 `point_history` 원장 기준이다.

**정책**

- Dashboard는 `Settlement.status = SUCCEEDED` 전까지 최종 정산 결과가 아니며, 정산 source of truth가 아니다.
- Dashboard projection과 최종 settlement 결과가 달라도 시스템 오류로 간주하지 않는다.
- `projection_status`, `projection_notice`는 API 응답용 값이며 DB enum이나 도메인 상태 원천으로 저장하지 않는다.
- `settlement_status = NONE`은 해당 방의 `Settlement` row가 아직 없다는 뜻이며, Dashboard projection을 계산할 수 없다는 의미가 아니다. 저장/authority 규칙은 Settlement-design §5.3을 따른다.
- `my_share_ratio_estimated`는 소수 오해 방지를 위해 string decimal로 반환한다.
- 적용 불가 필드는 생략하지 않고 `null`로 반환한다.
- Dashboard는 정산의 `remainder`, `remainder_policy`, deterministic remainder allocation, 1원 단위 잔액 처리를 계산하거나 반영하지 않는다. 해당 최종 지급 차이는 Settlement API에서만 확인한다.
- `SETTLEMENT_SUCCEEDED` 이후 최종 인정 성공 횟수, 최종 환급금, 최종 지분율은 `GET /api/settlements/{settlementId}`의 `settlement_item` 기준으로 확인한다.

## 5.7 정산


### `GET /api/crews/{crewId}/settlement`

> 크루의 정산 진행 상태를 조회한다.

**Response** `200 OK`

정산 row가 없는 경우:

```json
{
  "crew_id": 42,
  "settlement_id": null,
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
  "status": "RUNNING",
  "retry_count": 1,
  "failure_code": null,
  "failure_message": null,
  "started_at": "2026-06-01T13:12:10+09:00",
  "finished_at": null
}
```

**정책**

| 필드 | 설명 |
|------|------|
| `settlement_id` | 정산 식별자. `Settlement` row가 없으면 `null` |
| `status` | 정산 처리 상태. `NONE`은 Settlement row 없음 projection이다 |
| `retry_count` | 재시도 횟수 |
| `failure_code` | 실패 사유 코드. 실패하지 않았으면 `null`. 값 목록: `INPUT_LOAD_FAILED`, `CALCULATION_FAILED`, `POINT_CREDIT_FAILED`, `DUPLICATE_SETTLEMENT`, `LOCK_ACQUIRE_FAILED`, `UNKNOWN` |
| `failure_message` | 실패 상세 메시지 (내부 로그용) |
| `started_at` | 정산 실행 시작 시각 |
| `finished_at` | 정산 성공/실패 종료 시각. 진행 중이면 `null` |

- no row → `NONE`, row exists → corresponding `Settlement.status` (Settlement-design §5.3).
- `PENDING → RUNNING → SUCCEEDED / RETRY_WAIT / FAILED`는 `Settlement.status` 원천 상태를 그대로 반영한다.
- `started_at` / `finished_at`은 runtime execution fact다. lifecycle/cutoff authority는 `start_at`, crew timezone, daily cutoff 같은 scheduled semantic anchor에 남는다.

**Error**

- `CREW_NOT_FOUND`

---

### `GET /api/settlements/{settlementId}`

> 완료된 정산의 상세 결과와 참여자별 환급 내역을 조회한다.

**Response** `200 OK`

```json
{
  "settlement_id": 501,
  "crew_id": 42,
  "status": "SUCCEEDED",
  "retry_count": 1,
  "total_participants": 5,
  "total_locked_amount": 500000,
  "total_recognized_success": 390,
  "total_base_refund_amount": 499996,
  "total_remainder_amount": 4,
  "remainder_policy": "HOST_REMAINDER",
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
      "share_ratio": "0.23076923",
      "base_refund_amount": 115384,
      "remainder_bonus_amount": 4,
      "refund_amount": 115388,
      "point_history_id": 99001,
      "calculation_reason": {
        "included_dates": ["2026-05-01", "2026-05-02"],
        "excluded_logs": [
          {
            "server_time": "2026-05-02T07:10:11+09:00",
            "code": "DAILY_DUPLICATE"
          }
        ]
      }
    }
  ]
}
```

**Error**

- `SETTLEMENT_NOT_FOUND`

**헤더 필드 정책**

| 필드 | 설명 |
| --- | --- |
| `total_participants` | 정산 대상 frozen `LOCKED` participant 수 |
| `total_locked_amount` | 정산 시점의 `crew_participant.deposit_amount` 합계 스냅샷. `point_account`/`point_history`를 재합산하지 않는다 |
| `total_recognized_success` | 전체 참여자의 인정 성공 수 합계 |
| `total_base_refund_amount` | floor 연산 후 전체 기본 환급 합계 |
| `total_remainder_amount` | floor 연산 후 남은 잔액 |
| `remainder_policy` | 잔액 처리 정책. MVP active 값은 `HOST_REMAINDER`이며 의미는 “절사 후 남은 1~14원 수준의 remainder를 host의 `crew_participant`에 deterministic하게 배정하는 fixed policy”다. 이는 host discretion, random winner, payout mutation authority, settlement/ledger authority가 아니다. host item의 remainder bonus 금액은 해당 `settlement_item.remainder_bonus_amount`로만 확인하며, persisted payout authority는 `settlement_item.refund_amount`와 연결된 `point_history`다 |

**정산 항목(item) 필드 정책**

| 필드 | 설명 |
| --- | --- |
| `participant_status_snapshot` | 정산 시점의 참여 상태 스냅샷. 정산 후 row가 변경되어도 이 값이 계산 기준 |
| `deposit_amount` | 해당 참여자의 보증금 |
| `success_count_raw` | 중복/비유효 포함 raw 성공 로그 수 |
| `recognized_success_count` | 중복·비유효 제외 후 정산에서 실제 인정된 성공 수 |
| `recognized_dates_count` | 인정된 날짜 수 |
| `excluded_success_count` | 중복 등 제외된 성공 수 (`success_count_raw - recognized_success_count`) |
| `share_ratio` | 전체 인정 성공 중 해당 참여자 비율. 소수점 정밀도 오해 방지를 위해 string decimal로 반환 |
| `base_refund_amount` | `FLOOR(total_locked_amount × share_ratio)`. FLOOR 적용 후, `remainder_bonus_amount` 합산 전 기본 환급액 |
| `remainder_bonus_amount` | `HOST_REMAINDER` fixed policy에 따라 host item에 deterministic하게 배정된 절사 잔액 스냅샷. host discretion/authority가 아니며 payout authority는 `refund_amount`와 연결된 `point_history`다 |
| `refund_amount` | 실제 환급된 금액 (`base_refund_amount + remainder_bonus_amount`). **persisted 최종 환급 source of truth**이며 연결된 `point_history`와 함께 payout authority다 |
| `point_history_id` | 연결된 포인트 원장 ID. `null`이면 아직 지급 미완료 상태 |
| `calculation_reason` | 정산 포함/제외 근거. `includedDates`(인정된 날짜 목록)와 `excludedLogs`(제외된 로그의 `serverTime` + 제외 `code`) |

**`final_rank` (응답 포함 시)**

- 저장 컬럼이 아닌 logical projection이다.
- `recognized_success_count DESC`, 동률이면 `crew_participant_id ASC` 기준 read-time 계산한다.
- payout authority가 아니며 지급 결과 변경에 사용하지 않는다.

**정책**

- `settlement_item`은 참여자별 계산 스냅샷의 source of truth다.
- `point_history`는 실제 잔액에 반영된 금액 source of truth다.
- `SUCCEEDED`는 모든 `settlement_item.point_history_id`가 채워지고 대응 `point_history` 존재가 검증된 상태다.
- partial 상태에서는 일부 item의 `point_history_id`가 `null`일 수 있으며, 이 경우 `status`는 `SUCCEEDED`가 아니라 `RETRY_WAIT` 또는 `FAILED`다.
- `SUCCEEDED` 이후 운영/분쟁/조회 기준은 `settlement_item + point_history`다. `MissionLog` 기반 replay는 감사/디버깅용 검증에만 사용하며 지급 결과를 변경하지 않는다.
- 전체 인정 성공이 `0`이면 all-fail equal-principal refund를 적용한다:
  - 모든 참여자의 `refund_amount = deposit_amount` (보증금 원금 그대로 환급).
  - `remainder_bonus_amount = 0`이며 추가 지급 없음.
- `total_remainder_amount > 0`이면 MVP `HOST_REMAINDER` fixed policy에 따라 host `crew_participant` item의 `remainder_bonus_amount`/`refund_amount`에 deterministic하게 persisted된다. 이는 host가 금액, 수령자, 원장, 최종 정산을 선택하거나 override할 권한을 가진다는 뜻이 아니다.

---

## 5.8 AI


### `POST /api/ai/mission-recommendations`

> 목표 텍스트를 기반으로 AI가 미션 설정 초안을 추천한다.

**인증**

`Authorization: Bearer {access_token}` 필수

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `seed_text` | `string` | Y | 목표/습관 설명 |

**Response** `200 OK`

```json
{
  "draft": {
    "title": "아침 20분 독서 인증",
    "description": "매일 아침 독서한 책 페이지를 사진으로 인증합니다.",
    "frequency_type": "DAILY",
    "mission_schedule_days": [],
    "daily_settlement_type": "A",
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

**Error**

- `AI_RECOMMENDATION_FAILED`
- `AI_RESPONSE_INVALID`
- `VALIDATION_ERROR`

**정책**

- 추천 결과는 draft이며 자동 저장되지 않는다. 사용자 확인 후 `POST /api/crews`로 별도 저장한다.
- AI 실패는 시스템 실패가 아니다. `AI_RECOMMENDATION_FAILED`는 정산, 포인트 원장, 크루 생성 흐름을 차단하지 않는다. FE는 기존 입력값을 유지하고 수동 생성 흐름을 제공해야 한다.

## 5.9 알림 / Android FCM / Inbox


> FCM(Firebase Cloud Messaging)을 통한 Android-first 알림이 기준이다. 알림 서비스는 canonical state authority가 아닌 **best-effort re-entry hint** 역할만 한다. FE는 알림 payload의 값(display_text, resource_id 등)을 최종 상태로 신뢰하지 않아야 한다. 알림(push 또는 inbox item) 클릭 시 클라이언트는 `deep_link`로 이동 후 canonical REST API를 refetch해야 한다.

### `POST /api/notification-devices`

> FCM 알림 수신을 위해 기기를 등록한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `platform` | `string` | Y | `ANDROID` |
| `fcm_token` | `string` | Y | FCM 토큰 |
| `device_id` | `string` | Y | 클라이언트 기기 식별자 |
| `app_version` | `string` | N | 앱 버전 |

**Response** `201 Created`

```json
{
  "device_id": "client-generated-or-installation-id",
  "platform": "ANDROID",
  "enabled": true,
  "created_at": "2026-05-07T09:00:00+09:00"
}
```

**정책**

- JWT `sub = member.uuid`로 현재 인증 사용자의 디바이스만 등록한다.

---

### `PATCH /api/notification-devices/{deviceId}`

> 등록된 기기의 FCM 토큰 또는 앱 버전을 갱신한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `fcm_token` | `string` | N | 갱신할 FCM 토큰 |
| `app_version` | `string` | N | 앱 버전 |

**Response** `200 OK`

**Error**

- `DEVICE_NOT_FOUND`

**정책**

- `fcm_token` 또는 `app_version` 중 최소 하나는 제공되어야 한다. 둘 다 없으면 `VALIDATION_ERROR`를 반환한다.
- 제공되지 않은 필드는 기존 값을 유지한다.

---

### `DELETE /api/notification-devices/{deviceId}`

> 기기 알림 등록을 삭제한다.

**Response** `204 No Content`

---

### `GET /api/notifications`

> 내 알림 목록을 조회한다.

**Query**

| 필드     | 타입      | 필수 | 설명 |
| -------- | --------- | ---- | ---- |
| `cursor` | `string`  | N    | 이전 응답의 `next_cursor`로 다음 slice를 조회한다. |
| `limit`  | `integer` | N    | 기본 20, 최대 100. |

**Response** `200 OK`

```json
{
  "items": [
    {
      "notification_id": "uuid",
      "event_type": "MISSION_LOG_VERIFICATION_RESULT",
      "resource_type": "mission_log",
      "resource_id": "9001",
      "deep_link": "dondok://crews/42/mission-logs/9001",
      "occurred_at": "2026-05-13T07:31:08+09:00",
      "display_text": "인증 결과가 반영되었습니다.",
      "requires_refetch": true,
      "read_at": null
    }
  ],
  "next_cursor": null
}
```

**알림 항목 필드 정책**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `notification_id` | `string (uuid)` | 알림 고유 식별자 |
| `event_type` | `string` | 이벤트 유형 코드. 클라이언트 라우팅/UI 분기 기준 |
| `resource_type` | `string` | 연관 리소스 유형 (`mission_log`, `crew`, `settlement` 등) |
| `resource_id` | `string` | 연관 리소스 식별자 |
| `deep_link` | `string` | 알림 클릭 시 이동할 앱 내 URL scheme |
| `occurred_at` | `string (ISO-8601)` | 이벤트 발생 시각 |
| `display_text` | `string` | 서버가 생성한 알림 표시 문구. UI 직접 렌더링용 참고값 |
| `requires_refetch` | `boolean` | 클릭 후 canonical REST API refetch 필요 여부. **MVP에서는 항상 `true`로 취급한다** |
| `read_at` | `string \| null` | 읽음 처리 시각. 미읽음이면 `null` |

**정책**

- 최신순(`occurred_at DESC, notification_id DESC`) 정렬.
- `next_cursor`는 다음 slice가 존재할 때만 응답에 포함하며, 없거나 `null`이면 더 조회할 slice가 없다.
- `read_at = null`이면 읽지 않은 알림이다.

**click → refetch 계약**

알림(push 수신 또는 inbox item) 클릭 시 클라이언트는 다음 순서로 처리한다:

1. `deep_link` URL scheme으로 이동한다.
2. `deep_link`가 가리키는 리소스에 대해 canonical REST API를 refetch한다.
3. 알림 payload(`display_text`, `resource_id` 등)를 최종 상태 값으로 사용하지 않는다.

`requires_refetch: true`는 이 계약이 항상 적용됨을 나타낸다. MVP에서는 모든 알림이 `true`이므로, 클라이언트는 이 값의 `false` 분기를 MVP 단계에서 구현할 필요가 없다.

**event_type 목록**

> 구현 시 추가·변경될 수 있다.

| `event_type` | `resource_type` | 설명 |
| --- | --- | --- |
| `MISSION_LOG_VERIFICATION_RESULT` | `mission_log` | 방장이 인증 로그를 검수한 결과 (승인 또는 거절) |
| `CREW_APPLICATION_APPROVED` | `crew` | 내 크루 참여 신청이 승인됨 |
| `CREW_APPLICATION_REJECTED` | `crew` | 내 크루 참여 신청이 거절됨 |
| `CREW_ACTIVATED` | `crew` | 크루가 활성화(미션 시작)됨 |
| `SETTLEMENT_COMPLETED` | `settlement` | 크루 정산이 완료됨 |

---

### `GET /api/notifications/unread-count`

> 읽지 않은 알림 수를 조회한다.

**Response** `200 OK`

```json
{
  "unread_count": 3
}
```

---

### `PATCH /api/notifications/{notificationId}/read`

> 알림을 읽음 처리한다.

**Response** `200 OK`

---

### `PATCH /api/notifications/read-all`

> 알림을 전체 읽음 처리한다.

**Request** body 없음

**Response** `200 OK`

```json
{
  "updated_count": 5
}
```

**정책**

- 현재 인증 사용자의 읽지 않은 알림 전체를 읽음 처리한다.
- 이미 읽은 알림은 변경하지 않는다.
- `updated_count`는 실제로 읽음 처리된 알림 수다.

## 5.10 포인트


### `POST /api/points/charges`

> TossPayments 결제를 확인하여 포인트를 충전한다.

**Request**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `payment_id` | `string` | Y | TossPayments `paymentKey`. 하나의 충전 이벤트만 의미해야 한다 |
| `order_id` | `string` | Y | TossPayments `orderId`. confirm 검증과 로그 상관관계 추적용이며 idempotency key로 사용하지 않는다 |
| `amount` | `integer` | Y | 충전 금액 |

**Response** `201 Created`

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

동일 `payment_id` 재시도 시 `200 OK`로 기존 결과를 반환한다.

**Error**

- `INVALID_AMOUNT`
- `INVALID_PAYMENT_ID`
- `PAYMENT_ID_REUSED_WITH_DIFFERENT_AMOUNT`

**정책**

- Toss confirm 성공 확인 후에만 포인트 원장을 생성한다.
- idempotency key: `charge:{payment_id}`
- 동일 `payment_id` + 다른 금액은 conflict로 실패한다.

---

### `GET /api/points`

> 현재 포인트 잔액 상세(가용·예약·잠금)를 조회한다.

**Response** `200 OK`

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

**정책**

| 필드 | 설명 |
|------|------|
| `available_balance` | 현재 사용 가능한 잔액 |
| `reserved_balance` | `PENDING` 상태 참여의 보증금 |
| `active_locked_amount` | `RECRUITING`/`ACTIVE` 크루의 `LOCKED` 보증금 |
| `settlement_pending_amount` | `CLOSED` 크루의 정산 전 `LOCKED` 보증금 |
| `locked_balance` | `active_locked_amount + settlement_pending_amount` |
| `total_balance` | `available_balance + reserved_balance + locked_balance` |

- 이 필드들은 `point_account`의 DB 원천 컬럼이 아닌 read-time projection이다. `point_history`, `crew_participant` 상태 등에서 집계·계산하며, 별도 컬럼으로 저장하지 않는다.
- 이 필드들은 출금 가능 여부, 정산 결과 판단에 사용하지 않는다.
- `CANCELLED` 상태의 reserve는 반환 완료 상태이므로 `reserved_balance` 합산 대상이 아니다. 동일 row가 이후 reopen되어 `PENDING`으로 복귀하면 새 사이클의 reserve가 `reserved_balance` projection에 다시 합산된다.

---

### `GET /api/points/history`

> 포인트 입출금 거래 이력을 최신순으로 페이지네이션 조회한다.

**Query**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `limit` | `integer` | N | 기본 20, 최대 100 |
| `cursor` | `string` | N | 페이지네이션 커서 |

**Response** `200 OK`

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
  "next_cursor": "2026-05-07T09:30:00+09:00_3001"
}
```

**Error**

- `INVALID_LIMIT`
- `INVALID_CURSOR`

예시 요청:

```http
GET /api/points/history?limit=20
GET /api/points/history?limit=20&cursor=2026-05-07T09:30:00+09:00_3001
```

**정책**

- 최신순(`created_at DESC, point_history_id DESC`) 정렬
- `cursor`는 클라이언트가 해석하지 않고 다음 요청에 그대로 전달한다.
- `next_cursor`는 다음 slice가 존재할 때만 응답에 포함하며, 없거나 `null`이면 더 조회할 slice가 없다.
- `has_next`, `total_count` 같은 page total 필드는 MVP 필수 contract가 아니다.
- `limit`이 `1` 미만이거나 `100`을 초과하면 `INVALID_LIMIT`를 반환한다.
- `cursor` 형식이 잘못되었거나 해석할 수 없으면 `INVALID_CURSOR`를 반환한다.

**`balance_after` 필드 매핑**

- API 응답의 `balance_after`는 `point_history.available_after`의 read-only projection/alias다. 별도 persisted column이 아니다.
- `point_history.reserved_after`, `point_history.locked_after`는 reconciliation/debug snapshot이며 API 응답에 노출하지 않는다. 필요 시 운영/감사 채널로만 확인한다.
- `balance_after`는 convenience field이며, append-only ledger의 authoritative ordering/idempotency authority는 `point_history` row 자체다.

**`reference_type` / `reference_id` / `idempotency_key` 매핑**

| 도메인 동작 | `transaction_type` | `reference_type` | `reference_id` | `idempotency_key` |
|---|---|---|---|---|
| 포인트 충전 | `POINT_CHARGE` | `POINT_CHARGE` | 생성된 `point_history.id` | `charge:{paymentKey}` |
| 크루 참여 보증금 reserve/lock event | `CREW_DEPOSIT_RESERVE` | `CREW_PARTICIPANT` | `crew_participant.id` | `crew:{crewId}:participant:{participantId}:reserve:{cycle}` |
| PENDING reserve 반환 | `CREW_RESERVE_RELEASE` | `CREW_PARTICIPANT` | `crew_participant.id` | `crew:{crewId}:participant:{participantId}:reserve-release:{cycle}` |
| 일반 정산 환급 | `CREW_SETTLEMENT_REFUND` | `SETTLEMENT_ITEM` | `settlement_item.id` | `crew:{crewId}:participant:{participantId}:settlement-refund:{settlementId}` |

- `CREW_DEPOSIT_RESERVE`는 자산 이동이 아니라 보증금 reserve/lock 이벤트다. 일반 참여자의 `PENDING` 신청은 `available_balance -= deposit_amount` / `reserved_balance += deposit_amount`이고, host auto-created `LOCKED` 참여는 `available_balance -= deposit_amount` / `locked_balance += deposit_amount`다. 두 경우 모두 같은 `transaction_type`을 사용하며 별도 host 전용 transaction type을 만들지 않는다.
- 승인 시점 `PENDING → LOCKED` 전이는 `reserved_balance → locked_balance` bucket transition만 수행하며 새 `point_history` row를 만들지 않는다.
- `{cycle}`은 `CANCELLED → PENDING` reopen 시 증가하여 이전 사이클과 중복 처리를 방지한다. 최초 생성은 cycle `1`이며, 사이클 numbering은 implementation detail이다.

## 6. 상태 흐름

### Crew

```
RECRUITING ──(start_at 시스템 자동 활성화)──▶ ACTIVE ──▶ CLOSED
RECRUITING ──(start_at eligibility 미충족)──▶ CANCELLED
```

### Participant

```
PENDING ──(host 승인)──▶ LOCKED
PENDING ──(본인 취소)──▶ CANCELLED
PENDING ──(host 거절)──▶ REJECTED
PENDING ──(자동 만료)──▶ EXPIRED
CANCELLED ──(재신청 reopen)──▶ PENDING
```

- `REJECTED` / `EXPIRED`: terminal. 동일 crew 재신청은 `APPLICATION_NOT_ALLOWED`로 차단.
- host auto-created `LOCKED` row는 reopen 경로에 포함되지 않는다.

### Settlement

```
NONE → PENDING → RUNNING → SUCCEEDED
                  RUNNING → RETRY_WAIT → RUNNING → SUCCEEDED / FAILED
```

## 7. Contract Drift Notes

이 절은 backend active API와 PRD/Usecase semantic guardrail 사이에서 오해될 수 있는 host authority leakage를 containment한다. 이 절은 새 settlement policy를 설계하거나 backend active response shape를 임의 삭제하지 않는다.

### HOST_REMAINDER

`backend/docs/api/settlement.md`의 active settlement response는 `remainder_policy = HOST_REMAINDER`를 노출한다. PRD/Usecase/Settlement/ERD 계열 guardrail은 이를 host lifecycle/settlement/ledger authority나 host discretionary payout privilege로 해석하지 말 것을 요구한다.

이 migration에서는 backend 문서의 현재 active API response shape를 삭제하지 않고, 다음 해석을 고정한다.

- `HOST_REMAINDER`는 MVP active deterministic fixed policy다. 절사 후 남는 1~14원 수준의 remainder recipient를 host `crew_participant`로 고정한다.
- `HOST_REMAINDER`는 host discretion, random winner, lottery, payout mutation authority, ledger correction authority, lifecycle transition authority를 뜻하지 않는다.
- host item에 배정된 remainder 금액은 해당 `settlement_item.remainder_bonus_amount` 스냅샷으로만 확인한다. 별도 `remainder_winner_*` persisted column이나 active response field는 두지 않는다.
- persisted payout/refund authority는 각 `settlement_item.refund_amount`와 연결된 `point_history`다.
- clients/FE/support 문구는 `HOST_REMAINDER`라는 이름을 “방장이 임의로 가져감/정함”으로 설명하지 않고, deterministic fixed policy로 설명한다.
- all-fail equal principal refund에서는 추가 remainder 지급이 발생하지 않는다.

## 8. Deferred / Brownfield / Removed Surfaces

아래 항목은 MVP active API contract가 아니다. 새 구현이나 FE 연동에서 active endpoint처럼 사용하지 않는다.

- Crew manual start: `POST /api/crews/{crewId}/start`는 MVP active API가 아니다. `RECRUITING -> ACTIVE` 전이는 `start_at` 기준 시스템 자동 전환이다. Host/admin manual activation authority는 없다.
- Active withdrawal / rejoin: `POST /api/crews/{crewId}/withdraw`, `WITHDRAWN`, 중도탈퇴, 재참여 semantics는 MVP active lifecycle이 아니다. Frozen `LOCKED` baseline, final settlement, point ledger를 변경하는 근거로 쓰지 않는다.
- Admin settlement surface: `GET /api/admin/settlements`, `POST /api/admin/settlements/{settlementId}/retry`, admin/manual settlement, correction API는 MVP active API가 아니다. Backend `settlement.md`도 Admin Settlement API를 deferred로 둔다.
- AI habit report: `POST /api/crews/{crewId}/ai-habit-report`, `GET /api/crews/{crewId}/ai-habit-report/me`, `GET /api/ai-habit-reports/{reportId}`, 관련 habit report enum은 Phase 2 reference다. Settlement, `point_history`, mission certification, lifecycle authority가 아니다.
- SSE realtime stream: `GET /api/notifications/stream`은 Phase 2/deferred realtime drift reference다. MVP active notification baseline은 Android-first FCM device lifecycle과 notification inbox/read REST API다.
- `WEEKLY_N`, notification preference matrix, notification template CMS/table, delivery attempt observability, campaign/broadcast analytics, distributed replay engine, correction workflow, automatic replay recovery는 MVP active API contract가 아니다.

## 9. 구현 메모

- 인증 시점에는 `MissionLog` 저장과 검수/정산에서 재현 가능한 입력 보존을 우선한다.
- 정산 시점에는 `Settlement` row 상태와 `settlement_item` / `point_history` linkage가 final authority다.
- 알림 payload와 inbox item은 re-entry hint다. `deep_link` 이동 후 관련 canonical REST API를 다시 조회한다.
- 공지/댓글/리액션은 communication/social interaction surface다. `crew`, `mission_rule`, `mission_log`, `settlement`, `point_history`의 canonical rule/state를 변경하지 않는다.
