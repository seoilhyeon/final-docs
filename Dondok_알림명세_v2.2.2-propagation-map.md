# Dondok 알림명세 v2.2.2 Propagation Map

이 파일은 `Dondok_알림명세_v2.2.2.xlsx`의 Android-first FCM propagation 검토용 companion mapping이다. 알림은 best-effort re-entry hint이며, inbox/read는 UX hint history일 뿐 canonical history/audit/source-of-truth가 아니다.

| ID | 알림명 | Propagation 분류 | Guardrail |
| --- | --- | --- | --- |
| 1 | 크루 가입 신청 | MVP required | Host action hint. Application state is canonical crew/application API. |
| 2 | 크루 가입 신청 철회 | MVP required | Withdrawal notification is hint only; refetch application state. |
| 3 | 가입 승인 완료 | MVP required | Approval hint; deposit/participant canonical state must be refetched. |
| 4 | 가입 거절 | MVP required | Rejection hint; avoid person-level judgment wording. |
| 5 | 새 공지 등록 | MVP candidate | Engagement re-entry; no authority risk if notice API is canonical. |
| 6 | 크루 종료 예정 | MVP candidate | Reminder only; missing notification cannot alter mission end/certification cutoff. |
| 7 | 크루 해체 | MVP required | Send only after canonical cancellation/refund state; notification is not refund proof. |
| 8 | 공지 댓글 등록 | MVP candidate | Engagement re-entry; avoid notification flood. |
| 9 | 인증 마감 임박 | MVP required | Best-effort reminder only; missed push cannot excuse missed certification. |
| 10 | 인증 업로드 완료 | MVP required / toast | Foreground toast only; upload success is not certification success. |
| 11 | 인증 성공 | MVP required | Result hint; refetch mission log/certification detail. |
| 12 | 인증 실패 | MVP required | Result hint; avoid punitive/person-level failure wording. |
| 13 | 새 인증 업로드 | MVP required | Host review hint; moderation state remains canonical. |
| 14 | 미검토 인증 존재 | MVP required | Host reminder; missing push cannot block daily settlement/canonical cutoff. |
| 15 | 예상 환급금 변동 | MVP candidate / rewrite required | Daily progress/projection summary only. Avoid failure-profit, increase/decrease dopamine, or final amount framing. |
| 16 | 최종 정산 완료 | MVP required / sequence required | Send after canonical settlement/ledger commit. Push should deep-link/refetch; do not treat payload as payout proof. |
| 17 | 환급 완료 | MVP candidate / merge-or-sequence | Potential duplicate with final settlement notification. Merge or strictly sequence after point history commit. |
| 18 | 새 리액션 등록 | MVP candidate | Social engagement only; no certification/settlement effect. |
| 19 | 인기 인증 선정 | Phase 2 | Popular certification rules and exposure policy deferred. |
| 20 | 서비스 점검 안내 | Phase 2 | Campaign/broadcast/system notice capability deferred. |
| 21 | 정책 변경 안내 | Phase 2 | Policy broadcast/email matrix deferred. |
| 22 | 신고 접수 | Phase 2 | Admin notification/internal channel automation deferred. |
| 23 | 비정상 업로드 감지 | Phase 2 | Advanced fraud/admin automation deferred; risk signal is not final judgment. |

## Deferred freeze questions

- Final settlement email이 MVP mandatory인지 Phase 2 polish인지 별도 freeze 필요.
- `최종 정산 완료`와 `환급 완료`는 통합 또는 strict sequencing 중 하나로 확정 필요.
- Preference/template/campaign/advanced analytics는 Phase 2로 유지.

