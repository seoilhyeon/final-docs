# Usecase Diagram (External Share Version)

> 외부 공유·발표·README·onboarding용 간소화 다이어그램. "누가 무엇을 어디까지 하는 서비스인가"를 10초 안에 전달하는 것이 목적이다. 내부 semantic verification artifact는 [`Usecase-diagram-dondok.md`](./Usecase-diagram-dondok.md)에 분리되어 있다.

방장은 인증만 검토하고, 최종 정산은 시스템이 자동으로 처리합니다.
현재 기준 예상 환급금을 보며 함께 목표를 지속하는 구조를 지향합니다.

```mermaid
flowchart TB
    %% ===== Actors =====
    Host(["👑 방장"])
    Part(["🙋 참여자"])
    Sys(["⚙️ 시스템"])

    %% ===== Phase 1 =====
    subgraph PHASE1["① 함께 시작"]
        direction TB
        U1["크루 만들기"]
        U2["참가 · 보증금 예치"]
        U1 --> U2
    end

    %% ===== Phase 2 =====
    subgraph PHASE2["② 함께 인증"]
        direction TB
        U3["매일 인증 업로드"]
        U4["방장 인증 검토<br/><i>(정산은 시스템 자동 처리)</i>"]
        U3 --> U4
    end

    %% ===== Phase 3 =====
    subgraph PHASE3["③ 시스템 자동 정산"]
        direction TB
        U5["현재 기준 예상 환급<br/><i>참고값 · 최종 아님</i>"]
        U6["시스템 최종 자동 정산"]
        U7["결과 확인"]
        U5 --> U6 --> U7
    end

    PHASE1 ==> PHASE2 ==> PHASE3

    %% ===== Actor lines =====
    Host --- U1
    Host --- U4
    Part --- U2
    Part --- U3
    Part --- U7
    Sys --- U6

    %% ===== Cooperative note =====
    NOTE["💡 모두가 미달성해도 원금을 돌려받아요<br/>함께 버티는 구조를 지향합니다"]
    U6 -.- NOTE

    classDef actor fill:#FFE4B5,stroke:#333,stroke-width:1px,color:#000
    classDef step fill:#E6F3FF,stroke:#333,stroke-width:1px,color:#000
    classDef ref fill:#F4F4F4,stroke:#999,stroke-width:1px,color:#555
    classDef final fill:#D4EDDA,stroke:#155724,stroke-width:2px,color:#000
    classDef note fill:#FFF8DC,stroke:#888,stroke-width:1px,color:#000

    class Host,Part,Sys actor
    class U1,U2,U3,U4,U7 step
    class U5 ref
    class U6 final
    class NOTE note
```

## 읽는 법

| 색상 | 의미 |
|---|---|
| 🟧 살구색 | 액터 (방장 / 참여자 / 시스템) |
| 🟦 파랑 | 일반 단계 |
| ⬜ 회색 | 참고값 (예상 환급 — 최종 아님) |
| 🟢 진한 초록 | 최종 자동 정산 (시스템 단독 권한) |
| 🟨 노랑 | 협업 안내 |

## 핵심 약속

- **방장**: 인증 검토만 합니다. 환급·정산 권한은 없습니다.
- **참여자**: 보증금을 예치하고, 매일 인증을 올리고, 결과를 확인합니다.
- **시스템**: 정산은 시스템이 자동으로 처리합니다. 사람이 결과를 바꾸지 않습니다.
- **전원 미달성**: 모두에게 원금을 돌려드립니다. 누군가의 실패가 다른 사람의 수익이 되는 구조가 아닙니다.

## 더 자세한 내용

- [`Usecase-diagram-dondok.md`](./Usecase-diagram-dondok.md) — 내부 검증용 semantic diagram (authority boundary · freeze · retry/replay 등 상세)
- [`Usecase-dondok.md`](./Usecase-dondok.md) — usecase 인벤토리 (canonical behavioral semantics)
- [`PRD-dondok.md`](./PRD-dondok.md) — 제품 요구사항 종합
