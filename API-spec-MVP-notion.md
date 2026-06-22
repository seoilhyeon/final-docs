# Dondok MVP API 명세 — Notion handoff snapshot

> Derived MVP handoff snapshot, not canonical source of truth. Canonical source of truth is `backend/docs/api/*` and the integrated contract `docs/API-spec-dondok.md`. This file is optimized for Notion import/copy and should be refreshed from the active sources before long-term reuse.

## 1. 문서 목적

- MVP 마무리 시점의 구현 API와 active API docs 상태를 한 문서로 정리한다.
- Notion 이관을 쉽게 하기 위해 단순한 Markdown heading/table 구조만 사용한다.
- 코드/문서 불일치는 숨기지 않고 `확인 필요` / `문서 보강 필요`로 분리한다.

## 2. 현재 상태 요약

- Spring controller `/api` endpoint: **61개**
- Active docs endpoint row: **65개**
  - current matched endpoint: **61개**
  - 의도적으로 남긴 docs-only endpoint: **4개**
- Matched: **61개**
- Code only: **0개**
- Docs only: **4개**
  - `GET /api/crews/{crewId}/moderation-logs`: stale / current endpoint 아님
  - `GET /api/me/verification-history`: feed 통합 사용 / standalone current endpoint 아님
  - `PATCH /api/notification-devices/{deviceId}`: 예정 기능
  - `DELETE /api/notification-devices/{deviceId}`: 예정 기능
- `POST /api/notification-devices` old registration path는 current active docs에서 제거됐고, 현재 경로는 `POST /api/notifications/devices`다.
- `GET /api/me/crews`는 `backend/docs/api/my-crews.md`의 plain heading으로 문서화되어 있으며 code-only가 아니다.
- Frontend build / Backend test 실패는 이 문서 작업의 blocker가 아니라 QA follow-up이다.

## 3. 공통 규칙

### Base URL

```text
/api
```

### 인증

- 대부분의 보호 API는 `Authorization: Bearer {accessToken}` 헤더가 필요하다.
- 인증 예외: `POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/member/signup`.
- Refresh Token은 `refreshToken` HttpOnly cookie로 전달한다.
- JWT subject(`sub`)는 `member.uuid`다. 내부 DB 식별자인 `member.id`는 공개 API에 노출하지 않는다.

### 응답 / 필드 규칙

- API 응답 필드는 `snake_case`를 사용한다.
- 사용자 공개 식별자는 `member_uuid`를 사용한다.
- 시간은 ISO-8601 offset datetime 또는 명세된 `YYYY-MM-DD` / `YYYY-MM` 형식을 사용한다.
- 금액은 원화 정수 단위로 저장/응답한다.
- 이미지 업로드는 presigned URL 흐름을 따른다.

### 공통 에러 형태

```json
{
  "code": "ERROR_CODE",
  "message": "사용자 또는 클라이언트가 이해할 수 있는 오류 메시지",
  "details": {}
}
```

## 4. Endpoint 목록

| Domain | Method | Path | Auth | Source | Status |
|---|---:|---|---|---|---|
| AI | POST | `/api/ai/mission-recommendations` | Yes | `backend\docs\api\ai.md`:3 | matched |
| Health / Internal | GET | `/api/health` | Yes | `backend\docs\api\overview.md`:297 | matched |
| 대시보드 | GET | `/api/crews/{crewId}/dashboard` | Yes | `backend\docs\api\overview.md`:204 | matched |
| 대시보드 | GET | `/api/dashboard` | Yes | `backend\docs\api\dashboard.md`:3 | matched |
| 미션 / 피드 | GET | `/api/feed` | Yes | `backend\docs\api\feed.md`:135 | matched |
| 미션 / 피드 | POST | `/api/mission-logs` | Yes | `backend\docs\api\mission.md`:53 | matched |
| 미션 / 피드 | GET | `/api/mission-logs/{missionLogId}` | Yes | `backend\docs\api\feed.md`:89 | matched |
| 미션 / 피드 | POST | `/api/mission-logs/{missionLogId}/moderation/approve` | Yes | `backend\docs\api\mission.md`:290 | matched |
| 미션 / 피드 | POST | `/api/mission-logs/{missionLogId}/moderation/reject` | Yes | `backend\docs\api\mission.md`:363 | matched |
| 미션 / 피드 | POST | `/api/mission-logs/{missionLogId}/moderation/revert` | Yes | `backend\docs\api\mission.md`:448 | matched |
| 미션 / 피드 | POST | `/api/mission-logs/{missionLogId}/reactions` | Yes | `backend\docs\api\feed.md`:140 | matched |
| 미션 / 피드 | DELETE | `/api/mission-logs/{missionLogId}/reactions/me` | Yes | `backend\docs\api\feed.md`:179 | matched |
| 알림 | DELETE | `/api/notification-devices/{deviceId}` | Yes | `backend\docs\api\notification.md`:62 | docs-only: 예정 기능 |
| 알림 | PATCH | `/api/notification-devices/{deviceId}` | Yes | `backend\docs\api\notification.md`:36 | docs-only: 예정 기능 |
| 알림 | GET | `/api/notification-settings` | Yes | `backend\docs\api\notification.md`:72 | matched |
| 알림 | PATCH | `/api/notification-settings` | Yes | `backend\docs\api\notification.md`:101 | matched |
| 알림 | GET | `/api/notifications` | Yes | `backend\docs\api\notification.md`:153 | matched |
| 알림 | POST | `/api/notifications/devices` | Yes | `backend\docs\api\notification.md`:5 | matched |
| 알림 | PATCH | `/api/notifications/read-all` | Yes | `backend\docs\api\notification.md`:251 | matched |
| 알림 | GET | `/api/notifications/unread-count` | Yes | `backend\docs\api\notification.md`:229 | matched |
| 알림 | PATCH | `/api/notifications/{notificationId}/read` | Yes | `backend\docs\api\notification.md`:243 | matched |
| 업로드 | POST | `/api/uploads/presigned-url` | Yes | `backend\docs\api\mission.md`:3 | matched |
| 인증 / 회원 | POST | `/api/auth/login` | No | `backend\docs\api\overview.md`:35 | matched |
| 인증 / 회원 | POST | `/api/auth/logout` | Yes | `backend\docs\api\overview.md`:83 | matched |
| 인증 / 회원 | POST | `/api/auth/oauth2/token` | Yes | `backend\docs\api\auth.md`:89 | matched |
| 인증 / 회원 | POST | `/api/auth/refresh` | No | `backend\docs\api\overview.md`:71 | matched |
| 인증 / 회원 | GET | `/api/me` | Yes | `backend\docs\api\overview.md`:62 | matched |
| 인증 / 회원 | GET | `/api/me/activity-summary` | Yes | `backend\docs\api\member.md`:251 | matched |
| 인증 / 회원 | GET | `/api/me/crews` | Yes | `backend\docs\api\my-crews.md`:12 | matched |
| 인증 / 회원 | GET | `/api/me/host-operation-summary` | Yes | `backend\docs\api\member.md`:140 | matched |
| 인증 / 회원 | PATCH | `/api/me/profile` | Yes | `backend\docs\api\member.md`:261 | matched |
| 인증 / 회원 | GET | `/api/me/verification-history` | Yes | `backend\docs\api\mission.md`:508 | docs-only: feed 통합 사용 |
| 인증 / 회원 | POST | `/api/member/signup` | No | `backend\docs\api\member.md`:3 | matched |
| 인증 / 회원 | GET | `/api/members/{memberUuid}/profile` | Yes | `backend\docs\api\member.md`:190 | matched |
| 정산 | GET | `/api/settlements/{settlementId}` | Yes | `backend\docs\api\settlement.md`:238 | matched |
| 정산 | GET | `/api/settlements/{settlementId}/me` | Yes | `backend\docs\api\settlement.md`:179 | matched |
| 크루 / 참여 | GET | `/api/crews` | Yes | `backend\docs\api\crew.md`:3 | matched |
| 크루 / 참여 | POST | `/api/crews` | Yes | `backend\docs\api\crew.md`:50 | matched |
| 크루 / 참여 | DELETE | `/api/crews/{crewId}` | Yes | `backend\docs\api\crew.md`:177 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}` | Yes | `backend\docs\api\crew.md`:129 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/applications` | Yes | `backend\docs\api\crew.md`:348 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/applications/count` | Yes | `backend\docs\api\crew.md`:392 | matched |
| 크루 / 참여 | POST | `/api/crews/{crewId}/applications/{participantId}/approve` | Yes | `backend\docs\api\crew.md`:281 | matched |
| 크루 / 참여 | POST | `/api/crews/{crewId}/applications/{participantId}/reject` | Yes | `backend\docs\api\crew.md`:315 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/host/mission-logs/reviewable` | Yes | `backend\docs\api\mission.md`:147 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/members` | Yes | `backend\docs\api\crew.md`:419 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/moderation-logs` | Yes | `backend\docs\api\mission.md`:236 | docs-only: stale / 구현 중 불필요해짐 |
| 크루 / 참여 | POST | `/api/crews/{crewId}/participants` | Yes | `backend\docs\api\crew.md`:277 | matched |
| 크루 / 참여 | DELETE | `/api/crews/{crewId}/participants/me` | Yes | `backend\docs\api\crew.md`:250 | matched |
| 크루 / 참여 | GET | `/api/crews/{crewId}/settlement` | Yes | `backend\docs\api\settlement.md`:3 | matched |
| 크루 공지 / 댓글 / 리액션 | GET | `/api/crews/{crewId}/notices` | Yes | `backend\docs\api\notice.md`:5 | matched |
| 크루 공지 / 댓글 / 리액션 | POST | `/api/crews/{crewId}/notices` | Yes | `backend\docs\api\notice.md`:80 | matched |
| 크루 공지 / 댓글 / 리액션 | DELETE | `/api/crews/{crewId}/notices/{noticeId}` | Yes | `backend\docs\api\notice.md`:121 | matched |
| 크루 공지 / 댓글 / 리액션 | GET | `/api/crews/{crewId}/notices/{noticeId}` | Yes | `backend\docs\api\notice.md`:46 | matched |
| 크루 공지 / 댓글 / 리액션 | PATCH | `/api/crews/{crewId}/notices/{noticeId}` | Yes | `backend\docs\api\notice.md`:100 | matched |
| 크루 공지 / 댓글 / 리액션 | GET | `/api/crews/{crewId}/notices/{noticeId}/comments` | Yes | `backend\docs\api\notice.md`:133 | matched |
| 크루 공지 / 댓글 / 리액션 | POST | `/api/crews/{crewId}/notices/{noticeId}/comments` | Yes | `backend\docs\api\notice.md`:174 | matched |
| 크루 공지 / 댓글 / 리액션 | DELETE | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | Yes | `backend\docs\api\notice.md`:213 | matched |
| 크루 공지 / 댓글 / 리액션 | PATCH | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | Yes | `backend\docs\api\notice.md`:194 | matched |
| 크루 공지 / 댓글 / 리액션 | POST | `/api/crews/{crewId}/notices/{noticeId}/reactions` | Yes | `backend\docs\api\notice.md`:225 | matched |
| 크루 공지 / 댓글 / 리액션 | DELETE | `/api/crews/{crewId}/notices/{noticeId}/reactions/me` | Yes | `backend\docs\api\notice.md`:251 | matched |
| 포인트 | GET | `/api/points` | Yes | `backend\docs\api\point.md`:57 | matched |
| 포인트 | POST | `/api/points/charges` | Yes | `backend\docs\api\point.md`:3 | matched |
| 포인트 | GET | `/api/points/history` | Yes | `backend\docs\api\point.md`:187 | matched |
| 포인트 | GET | `/api/points/wallet-history` | Yes | `backend\docs\api\point.md`:185 | matched |

## 5. Endpoint 상세 템플릿

Notion에서 endpoint별 상세가 필요하면 아래 템플릿을 각 endpoint row 아래에 복사한다. 반복 boilerplate를 줄이기 위해 이 snapshot에는 표와 active source 링크를 우선 둔다.

```md
### METHOD /api/path
- Purpose: active source 기준 endpoint 목적
- Auth: required / public
- Request: path/query/body table
- Response: status + schema/example
- Errors: domain errors + common errors
- Notes: 정책, 권한, 상태 전이, mismatch
```

## 6. 확인 필요 / 문서 보강 필요

### active docs 반영 완료

| Method | Path | Active source | 상태 |
|---:|---|---|---|
| GET | `/api/crews/{crewId}/applications/count` | `backend\docs\api\crew.md`:392 | 해결됨: active docs 반영 |
| GET | `/api/health` | `backend\docs\api\overview.md`:297 | 해결됨: active docs 반영 |
| GET | `/api/notification-settings` | `backend\docs\api\notification.md`:72 | 해결됨: active docs 반영 |
| PATCH | `/api/notification-settings` | `backend\docs\api\notification.md`:101 | 해결됨: active docs 반영 |
| POST | `/api/notifications/devices` | `backend\docs\api\notification.md`:5 | 해결됨: 현재 구현 경로로 active docs 갱신 |
| POST | `/api/notification-devices` | - | 해결됨: old registration path는 current active docs에서 제거 |

### 의도적으로 남긴 docs-only endpoint

| Method | Path | Doc source | 상태 |
|---:|---|---|---|
| GET | `/api/crews/{crewId}/moderation-logs` | `backend\docs\api\mission.md`:236 | stale로 표시됨. current endpoint로 취급하지 않음 |
| GET | `/api/me/verification-history` | `backend\docs\api\mission.md`:508 | feed 통합 사용으로 표시됨. standalone current endpoint로 취급하지 않음 |
| DELETE | `/api/notification-devices/{deviceId}` | `backend\docs\api\notification.md`:62 | 예정 기능. 문서에는 유지 |
| PATCH | `/api/notification-devices/{deviceId}` | `backend\docs\api\notification.md`:36 | 예정 기능. 문서에는 유지 |


### path variable naming mismatch

| Endpoint | Code | Active docs | 조치 |
|---|---|---|---|
| `POST /api/crews/{crewId}/applications/{...}/approve` | `{participantId}` in `CrewController` | `{crewParticipantId}` in `backend/docs/api/crew.md` | API path shape is equivalent, but naming should be normalized in docs or explicitly kept as alias. |
| `POST /api/crews/{crewId}/applications/{...}/reject` | `{participantId}` in `CrewController` | `{crewParticipantId}` in `backend/docs/api/crew.md` | API path shape is equivalent, but naming should be normalized in docs or explicitly kept as alias. |

## 7. 도메인별 상세

도메인별 표는 Notion page/DB 분리 기준으로 쓸 수 있다. 상세 request/response 예시는 Source 열의 active source에서 복사한다.

### 인증 / 회원

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| POST | `/api/auth/login` | No | matched | `backend\docs\api\overview.md`:35 | 상세는 active source 참조 |
| POST | `/api/auth/logout` | Yes | matched | `backend\docs\api\overview.md`:83 | 상세는 active source 참조 |
| POST | `/api/auth/oauth2/token` | Yes | matched | `backend\docs\api\auth.md`:89 | 상세는 active source 참조 |
| POST | `/api/auth/refresh` | No | matched | `backend\docs\api\overview.md`:71 | 상세는 active source 참조 |
| GET | `/api/me` | Yes | matched | `backend\docs\api\overview.md`:62 | 상세는 active source 참조 |
| GET | `/api/me/activity-summary` | Yes | matched | `backend\docs\api\member.md`:251 | 상세는 active source 참조 |
| GET | `/api/me/crews` | Yes | matched | `backend\docs\api\my-crews.md`:12 | 상세는 active source 참조 |
| GET | `/api/me/host-operation-summary` | Yes | matched | `backend\docs\api\member.md`:140 | 상세는 active source 참조 |
| PATCH | `/api/me/profile` | Yes | matched | `backend\docs\api\member.md`:261 | 상세는 active source 참조 |
| GET | `/api/me/verification-history` | Yes | docs-only: feed 통합 사용 | `backend\docs\api\mission.md`:508 | 별도 endpoint 대신 feed와 통합 사용 중 |
| POST | `/api/member/signup` | No | matched | `backend\docs\api\member.md`:3 | 상세는 active source 참조 |
| GET | `/api/members/{memberUuid}/profile` | Yes | matched | `backend\docs\api\member.md`:190 | 상세는 active source 참조 |

### 크루 / 참여

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/crews` | Yes | matched | `backend\docs\api\crew.md`:3 | 상세는 active source 참조 |
| POST | `/api/crews` | Yes | matched | `backend\docs\api\crew.md`:50 | 상세는 active source 참조 |
| DELETE | `/api/crews/{crewId}` | Yes | matched | `backend\docs\api\crew.md`:177 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}` | Yes | matched | `backend\docs\api\crew.md`:129 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/applications` | Yes | matched | `backend\docs\api\crew.md`:348 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/applications/count` | Yes | matched | `backend\docs\api\crew.md`:392 | active docs 반영 완료 |
| POST | `/api/crews/{crewId}/applications/{participantId}/approve` | Yes | matched | `backend\docs\api\crew.md`:281 | 상세는 active source 참조 |
| POST | `/api/crews/{crewId}/applications/{participantId}/reject` | Yes | matched | `backend\docs\api\crew.md`:315 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/host/mission-logs/reviewable` | Yes | matched | `backend\docs\api\mission.md`:147 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/members` | Yes | matched | `backend\docs\api\crew.md`:419 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/moderation-logs` | Yes | docs-only: stale | `backend\docs\api\mission.md`:236 | 구현 중 불필요해진 endpoint로 판단 |
| POST | `/api/crews/{crewId}/participants` | Yes | matched | `backend\docs\api\crew.md`:277 | 상세는 active source 참조 |
| DELETE | `/api/crews/{crewId}/participants/me` | Yes | matched | `backend\docs\api\crew.md`:250 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/settlement` | Yes | matched | `backend\docs\api\settlement.md`:3 | 상세는 active source 참조 |

### 크루 공지 / 댓글 / 리액션

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/crews/{crewId}/notices` | Yes | matched | `backend\docs\api\notice.md`:5 | 상세는 active source 참조 |
| POST | `/api/crews/{crewId}/notices` | Yes | matched | `backend\docs\api\notice.md`:80 | 상세는 active source 참조 |
| DELETE | `/api/crews/{crewId}/notices/{noticeId}` | Yes | matched | `backend\docs\api\notice.md`:121 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/notices/{noticeId}` | Yes | matched | `backend\docs\api\notice.md`:46 | 상세는 active source 참조 |
| PATCH | `/api/crews/{crewId}/notices/{noticeId}` | Yes | matched | `backend\docs\api\notice.md`:100 | 상세는 active source 참조 |
| GET | `/api/crews/{crewId}/notices/{noticeId}/comments` | Yes | matched | `backend\docs\api\notice.md`:133 | 상세는 active source 참조 |
| POST | `/api/crews/{crewId}/notices/{noticeId}/comments` | Yes | matched | `backend\docs\api\notice.md`:174 | 상세는 active source 참조 |
| DELETE | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | Yes | matched | `backend\docs\api\notice.md`:213 | 상세는 active source 참조 |
| PATCH | `/api/crews/{crewId}/notices/{noticeId}/comments/{commentId}` | Yes | matched | `backend\docs\api\notice.md`:194 | 상세는 active source 참조 |
| POST | `/api/crews/{crewId}/notices/{noticeId}/reactions` | Yes | matched | `backend\docs\api\notice.md`:225 | 상세는 active source 참조 |
| DELETE | `/api/crews/{crewId}/notices/{noticeId}/reactions/me` | Yes | matched | `backend\docs\api\notice.md`:251 | 상세는 active source 참조 |

### 미션 / 피드

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/feed` | Yes | matched | `backend\docs\api\feed.md`:135 | 상세는 active source 참조 |
| POST | `/api/mission-logs` | Yes | matched | `backend\docs\api\mission.md`:53 | 상세는 active source 참조 |
| GET | `/api/mission-logs/{missionLogId}` | Yes | matched | `backend\docs\api\feed.md`:89 | 상세는 active source 참조 |
| POST | `/api/mission-logs/{missionLogId}/moderation/approve` | Yes | matched | `backend\docs\api\mission.md`:290 | 상세는 active source 참조 |
| POST | `/api/mission-logs/{missionLogId}/moderation/reject` | Yes | matched | `backend\docs\api\mission.md`:363 | 상세는 active source 참조 |
| POST | `/api/mission-logs/{missionLogId}/moderation/revert` | Yes | matched | `backend\docs\api\mission.md`:448 | 상세는 active source 참조 |
| POST | `/api/mission-logs/{missionLogId}/reactions` | Yes | matched | `backend\docs\api\feed.md`:140 | 상세는 active source 참조 |
| DELETE | `/api/mission-logs/{missionLogId}/reactions/me` | Yes | matched | `backend\docs\api\feed.md`:179 | 상세는 active source 참조 |

### 대시보드

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/crews/{crewId}/dashboard` | Yes | matched | `backend\docs\api\overview.md`:204 | 상세는 active source 참조 |
| GET | `/api/dashboard` | Yes | matched | `backend\docs\api\dashboard.md`:3 | 상세는 active source 참조 |

### 업로드

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| POST | `/api/uploads/presigned-url` | Yes | matched | `backend\docs\api\mission.md`:3 | 상세는 active source 참조 |

### 포인트

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/points` | Yes | matched | `backend\docs\api\point.md`:57 | 상세는 active source 참조 |
| POST | `/api/points/charges` | Yes | matched | `backend\docs\api\point.md`:3 | 상세는 active source 참조 |
| GET | `/api/points/history` | Yes | matched | `backend\docs\api\point.md`:187 | 상세는 active source 참조 |
| GET | `/api/points/wallet-history` | Yes | matched | `backend\docs\api\point.md`:185 | 상세는 active source 참조 |

### 정산

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/settlements/{settlementId}` | Yes | matched | `backend\docs\api\settlement.md`:238 | 상세는 active source 참조 |
| GET | `/api/settlements/{settlementId}/me` | Yes | matched | `backend\docs\api\settlement.md`:179 | 상세는 active source 참조 |

### 알림

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| DELETE | `/api/notification-devices/{deviceId}` | Yes | docs-only: 예정 기능 | `backend\docs\api\notification.md`:62 | 일정 보고 추가 예정, 문서 유지 |
| PATCH | `/api/notification-devices/{deviceId}` | Yes | docs-only: 예정 기능 | `backend\docs\api\notification.md`:36 | 일정 보고 추가 예정, 문서 유지 |
| GET | `/api/notification-settings` | Yes | matched | `backend\docs\api\notification.md`:72 | active docs 반영 완료 |
| PATCH | `/api/notification-settings` | Yes | matched | `backend\docs\api\notification.md`:101 | active docs 반영 완료 |
| GET | `/api/notifications` | Yes | matched | `backend\docs\api\notification.md`:153 | 상세는 active source 참조 |
| POST | `/api/notifications/devices` | Yes | matched | `backend\docs\api\notification.md`:5 | 현재 구현 경로 active docs 반영 완료 |
| PATCH | `/api/notifications/read-all` | Yes | matched | `backend\docs\api\notification.md`:251 | 상세는 active source 참조 |
| GET | `/api/notifications/unread-count` | Yes | matched | `backend\docs\api\notification.md`:229 | 상세는 active source 참조 |
| PATCH | `/api/notifications/{notificationId}/read` | Yes | matched | `backend\docs\api\notification.md`:243 | 상세는 active source 참조 |

### AI

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| POST | `/api/ai/mission-recommendations` | Yes | matched | `backend\docs\api\ai.md`:3 | 상세는 active source 참조 |

### Health / Internal

| Method | Path | Auth | Status | Source | Notes |
|---:|---|---|---|---|---|
| GET | `/api/health` | Yes | matched | `backend\docs\api\overview.md`:297 | active docs 반영 완료 |


## 8. Snapshot lifecycle

- 이 파일은 Notion 이관용 임시 snapshot이다. 장기 canonical 문서가 아니다.
- Notion 이관 후에는 이 파일을 삭제하거나 archive/export 위치로 이동한다.
- 계속 보관해야 한다면 `backend/docs/api/*` 또는 `docs/API-spec-dondok.md` 변경 시 이 파일을 재생성/검수한다.

## 9. QA Follow-up

- Frontend `npm run build` 실패: lockfile JSON parse 문제와 `firebase`, `html-to-image` module resolution 실패를 별도 작업으로 처리한다.
- Backend `./gradlew.bat test --no-daemon` 실패: `backend/build/test-results/test/binary/output.bin` file lock 삭제 실패를 정리한 뒤 재실행한다.
- 이 문서 작업은 API 동작 변경을 포함하지 않는다.

## 10. Verification checklist

- [ ] endpoint matrix 재실행: controller/docs/matched/code-only/docs-only count 확인
- [ ] `GET /api/me/crews` matched 확인
- [ ] duplicate endpoint heading 없음
- [ ] 새 문서에 `.env` 값/secret-looking token 없음
- [ ] Auth token flow spot check
- [ ] Point/settlement money fields integer KRW spot check
- [ ] Upload presigned URL flow spot check
