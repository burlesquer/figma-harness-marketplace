---
name: pixel-diff-loop
description: visual-verifier 또는 quality-auditor가 검증 실패 후 자동 수정을 시작할 때 사용. ECC gan-design + gan-build 명령으로 generator(screen-implementer) ↔ evaluator(verifier) GAN 패턴 루프를 돌려 diff를 임계치 아래로 줄인다. 최대 3회.
---

# pixel-diff-loop

검증 실패 → 자동 수정 → 재검증 사이클을 돌리는 GAN 스타일 루프. visual diff와 a11y/e2e 위반 양쪽을 모두 다룬다.

## 트리거 조건

다음 상황에서만 호출:
- `visual-verifier`가 diff > 5% 보고
- `quality-auditor`가 a11y/e2e 위반 보고
- `verification-loop` (ECC) 실패 후

## 루프 구조

```
Round N (1 ≤ N ≤ 3):
  1. evaluator: 현재 구현의 issues 목록 추출
     (visual diff hotspots, a11y violations, e2e failures)

  2. ECC `gan-design` 또는 `gan-build` 명령 호출:
     - generator role: screen-implementer
     - evaluator role: visual-verifier + quality-auditor

  3. generator: issues 받아서 부분 수정만 수행
     (전체 재구현 금지 — diff 위치만 패치)

  4. evaluator: 재검증
     - 통과 → 종료
     - 실패 → Round N+1
     - N == 3 → 사용자에게 보고
```

## generator 호출 프로토콜

screen-implementer를 SendMessage로 호출할 때:

```json
{
  "to": "screen-implementer",
  "type": "fix-request",
  "screenId": "63:30000",
  "issues": [
    {
      "kind": "visual-diff",
      "location": "sidebar",
      "expected": "_workspace/_screenshots/figma_sidebar.png",
      "actual": "_workspace/_screenshots/impl_sidebar.png",
      "hint": "sidebar should collapse to drawer at sm breakpoint"
    },
    {
      "kind": "a11y",
      "rule": "color-contrast",
      "selector": ".muted-text",
      "ratio": 3.8,
      "required": 4.5
    }
  ],
  "round": 1
}
```

## evaluator 호출 프로토콜

visual-verifier / quality-auditor에게 재검증 요청:
- 변경된 파일만 재테스트 (전체 리테스트 금지 — 시간 낭비)
- 결과를 `_workspace/07_visual_report/{screen-id}_round{N}.json`에 저장

## 수정 폭 제한

generator는 다음을 위반하지 않음:
- ❌ 컴포넌트 통째로 재작성 (issues에 명시된 위치만)
- ❌ 새로운 의존성 추가 (사용자 컨펌 없이)
- ❌ 토큰 변경 (design-token-extractor 영역)
- ✅ Tailwind 클래스 조정
- ✅ 누락된 prop 추가
- ✅ a11y 속성 추가
- ✅ 누락된 state 추가

## 종료 조건

| 조건 | 종료 후 동작 |
|---|---|
| diff < 5% AND a11y violations == 0 AND e2e all pass | 화면 완료 마킹, 다음 화면으로 |
| Round 3 도달 | 보고서 작성, 사용자에게 수동 개입 요청 |
| generator가 "수정 불가" 응답 | 디자인 의도 모호 — 사용자에게 명확화 요청 |

## 보고서

각 round 결과를 누적:
```json
{
  "screenId": "63:30000",
  "rounds": [
    { "round": 1, "issues": 5, "fixed": 3, "remaining": 2 },
    { "round": 2, "issues": 2, "fixed": 2, "remaining": 0 }
  ],
  "finalVerdict": "pass",
  "totalDuration": "4m 23s"
}
```

`_workspace/09_loop_report/{screen-id}.json`에 저장.

## ECC gan-design vs gan-build 선택

- **시각 diff 위주** → `gan-design` (UI 매칭 특화)
- **a11y/e2e 위반 위주** → `gan-build` (코드 정확성 중심)
- **혼합** → 두 번 호출 (시각 먼저, 그다음 품질)
