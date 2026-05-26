# Dondok 알림명세 v2.2.2 Propagation Map

이 파일은 `Dondok_알림명세_v2.2.2.xlsx`의 Android-first FCM propagation 검토용 companion mapping이다. 알림은 best-effort re-entry hint이며, inbox/read는 UX hint history일 뿐 canonical history/audit/source-of-truth가 아니다. 아래 `event_type`은 앱 라우팅 vocabulary 후보이며 DB enum, audit event catalog, API 구현 freeze가 아니다.

| ID | 알림명 | Propagation 분류 | Candidate `event_type` | Refetch target | Guardrail |
| --- | --- | --- | --- | --- | --- |
| 1 | 크루 가입 신청 | MVP required | `CREW_APPLICATION_SUBMITTED` | Host application/review API | Host action hint. `PENDING` reserve state is canonical crew/application API; push is not capacity or deposit authority. |
| 2 | 크루 가입 신청 철회 | MVP required | `CREW_APPLICATION_CANCELLED` | Crew application + points/reserve API | Cancellation notification is hint only; refetch terminal `CANCELLED` state and reserve refund ledger. |
| 3 | 가입 승인 완료 | MVP required | `CREW_APPLICATION_APPROVED` | Participant/application API | Approval hint; refetch canonical `LOCKED` participant state. Approval push is not deposit/settlement authority. |
| 4 | 가입 거절 | MVP required | `CREW_APPLICATION_REJECTED` | Participant/application + points/reserve API | Rejection hint; refetch terminal `REJECTED` state and reserve refund ledger. Avoid person-level judgment wording. |
| 5 | 새 공지 등록 | MVP candidate | `CREW_NOTICE_CREATED` | Crew notice API | Engagement re-entry; no authority risk if notice API is canonical. |
| 6 | 크루 종료 예정 | MVP candidate | `CREW_ENDED_SOON` | Crew dashboard | Reminder only; missing notification cannot alter mission end/certification cutoff. |
| 7 | 크루 해체 | MVP required | `CREW_CANCELLED` | Crew/participant/points APIs | Send only after canonical cancellation/refund state; notification is not refund proof. |
| 8 | 공지 댓글 등록 | MVP candidate | `CREW_NOTICE_COMMENT_CREATED` | Notice/comment API | Engagement re-entry; avoid notification flood. |
| 9 | 인증 마감 임박 | MVP required | `MISSION_CERTIFICATION_DUE_SOON` | Crew dashboard / mission logs API | Best-effort reminder only; missed push cannot excuse missed certification. |
| 10 | 인증 업로드 완료 | MVP required / toast | `MISSION_LOG_SUBMITTED` | Mission log API | Foreground toast only; upload success is not certification success. |
| 11 | 인증 성공 | MVP required | `MISSION_LOG_APPROVED` | Mission log/dashboard API | Result hint after canonical resolved state; refetch mission log/certification detail. |
| 12 | 인증 실패 | MVP required | `MISSION_LOG_REJECTED` | Mission log/dashboard API | Result hint after canonical resolved state; avoid punitive/person-level failure wording. |
| 13 | 새 인증 업로드 | MVP required | `MISSION_LOG_REVIEW_REQUESTED` | Host review API | Host review hint; moderation state remains canonical. |
| 14 | 미검토 인증 존재 | MVP required | `MISSION_LOG_REVIEW_PENDING` | Host review surface | Host reminder; missing push cannot block daily settlement/canonical cutoff. |
| 15 | 예상 환급금 변동 | MVP candidate / rewrite required | `DASHBOARD_PROJECTION_UPDATED` | Crew dashboard API | Current-basis progress/projection summary only. Avoid failure-profit, “상승/추가 수익”, relative-failure dopamine, guaranteed payout, or final amount framing. |
| 16 | 최종 정산 완료 | MVP required / sequence required | `SETTLEMENT_COMPLETED` | Settlement detail API | Send after canonical settlement/ledger commit. Push should deep-link/refetch; do not treat payload as payout proof. |
| 17 | 환급 완료 | MVP candidate / merge-or-sequence | `POINT_REFUND_COMPLETED` | Points/history API | Potential duplicate with final settlement notification. Merge or strictly sequence after point history commit; payload is not payout proof. |
| 18 | 새 리액션 등록 | MVP candidate | `MISSION_LOG_REACTION_CREATED` | Feed / mission log API | Social engagement only; no certification/settlement effect. |
| 19 | 인기 인증 선정 | Phase 2 | Deferred | Deferred | Popular certification rules and exposure policy deferred. |
| 20 | 서비스 점검 안내 | Phase 2 | Deferred | Deferred | Campaign/broadcast/system notice capability deferred. |
| 21 | 정책 변경 안내 | Phase 2 | Deferred | Deferred | Policy broadcast/email matrix deferred. |
| 22 | 신고 접수 | Phase 2 | Deferred | Deferred | Admin notification/internal channel automation deferred. |
| 23 | 비정상 업로드 감지 | Phase 2 | Deferred | Deferred | Advanced fraud/admin automation deferred; risk signal is not final judgment. |

## Deferred freeze questions

- Final settlement email이 MVP mandatory인지 Phase 2 polish인지 별도 freeze 필요. 이 freeze에서는 email semantics를 다루지 않는다.
- `최종 정산 완료`와 `환급 완료`는 통합 또는 strict sequencing 중 하나로 확정 필요. 어느 쪽이든 notification payload/list item은 payout proof가 아니며 canonical API refetch가 필요하다.
- Preference/template/campaign/advanced analytics는 Phase 2로 유지.

