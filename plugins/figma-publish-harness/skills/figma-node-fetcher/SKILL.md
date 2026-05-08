---
name: figma-node-fetcher
description: Figma MCP 도구 호출을 표준화하는 유틸리티 스킬. 도구 선택 의사결정 트리, 호출 비용 표, 병렬 호출 규약, 캐싱 정책. 화면/컴포넌트/에셋을 가져오기 전 항상 이 스킬을 따른다. 모든 Figma 관련 에이전트가 사용.
---

# figma-node-fetcher

Figma MCP 호출 시 **올바른 도구를 올바른 순서로** 호출하기 위한 규약. 잘못된 도구 선택 1회는 30초~수분의 시간 낭비를 만든다. 호출 전 반드시 의사결정 트리를 따른다.

## URL 파싱

| URL 패턴 | 추출 |
|---|---|
| `figma.com/design/:fileKey/:fileName?node-id=:nodeId` | fileKey 그대로, nodeId의 `-` → `:` |
| `figma.com/design/:fileKey/branch/:branchKey/:fileName` | branchKey를 fileKey로 사용 |
| `figma.com/board/:fileKey/...` | FigJam — `get_figjam` 사용 |
| `figma.com/make/:makeFileKey/...` | makeFileKey 사용 |

예: `?node-id=63-29809` → MCP에서 `"63:29809"`

## MCP 도구 호출 비용표

호출 전에 **반드시** 이 표를 확인하고 가장 가벼운 도구로 시작한다.

| 도구 | 비용 | 반환 | 사용 시점 |
|---|---|---|---|
| `whoami` | 즉시 | 인증 정보 | **세션 시작 1회** (인증 사전 확인) |
| `get_libraries` | 가벼움 | 외부 라이브러리 목록 | 의존성 확인 (다른 파일 라이브러리 사용 여부) |
| `search_design_system` | 가벼움 | 토큰/컴포넌트 검색 결과 | 토큰 누락 시 폴백, 컴포넌트 탐색 |
| `get_metadata` | 매우 가벼움 | 노드 트리 (depth 1~2) | 구조 파악, 자식 노드 ID 수집 |
| `get_variable_defs` | 가벼움 | 토큰/변수 정의 | 디자인 시스템 추출 |
| `get_code_connect_map` | 가벼움 | 등록된 매핑 목록 | 매핑 존재 확인 |
| `get_context_for_code_connect` | 가벼움 | 매핑된 코드 스니펫 | **매핑된** 컴포넌트의 코드 가져오기 |
| `get_code_connect_suggestions` | 가벼움 | 자동 매핑 제안 | 매핑 후보 탐색 |
| `get_screenshot` | 중간 | PNG (scale 옵션) | 시각 검증, 비교 원본 |
| `upload_assets` | 중간 | export 결과 | 에셋 추출 (옵션 필수) |
| `get_design_context` | **무거움** | 코드+스크린샷+토큰힌트 | **미매핑 노드만** 변환 |
| `create_design_system_rules` | 무거움 | 룰 파일 | 토큰 추출 후 1회만 |

### 세션 시작 시 의무 호출

모든 Figma 워크플로우의 **첫 동작**:

1. MCP `whoami` 호출 — 인증·Dev Mode 라이선스 사전 확인
   - **200 OK**: 후속 작업 진행
   - **401/403**: 사용자에게 Figma 로그인 또는 Desktop 앱 실행 요청 후 중단 (자동 폴백 채널 없음)
   - **Dev Mode license required**: 사용자에게 Dev Mode 좌석 부여 또는 라이선스 보유 계정으로 재인증 요청 후 중단
2. `get_libraries(fileKey)` — 외부 라이브러리 의존성 사전 파악

이 두 호출은 캐시 키 별도 관리(세션 1회만, 사용자가 "재인증" 명시 시 무효화).

> Figma 디자인 추출은 **Figma MCP 단일 채널**을 통해서만 수행한다. 시각 추론에 의존하는 웹 UI 스크래핑(Playwright 기반 figma.com 조작)은 정확도 손실이 커서 사용하지 않는다. 인증/라이선스 문제는 사용자가 직접 해결해야 후속이 진행된다.

### 토큰/컴포넌트 누락 시 폴백

`get_variable_defs`로 못 찾은 토큰이 있을 때:
- `search_design_system(query="brand-500")` → 다른 페이지/외부 라이브러리에서 검색
- 결과 있으면 사용, 없으면 사용자에게 누락 보고

**황금 규칙:** `get_design_context`는 **미매핑이 확실한 노드**에만 호출. 매핑 여부 미확인 시 `get_code_connect_map` → `get_context_for_code_connect` 순으로 먼저 시도.

## 의사결정 트리

```
입력: nodeId
  │
  ├─ "이 노드는 무엇인가?" 모름
  │     → get_metadata(depth=1) 1회 → 분류
  │
  ├─ 토큰 페이지 (Colors/Typography/Spacing)
  │     → get_variable_defs
  │
  ├─ 컴포넌트 (UI Kit)
  │     → get_code_connect_map(fileKey) 캐시 확인
  │       ├─ 매핑됨: get_context_for_code_connect (코드 스니펫 즉시 반환, 변환 불필요)
  │       └─ 미매핑: get_design_context → 신규 변환
  │
  ├─ 화면 (Screen)
  │     → get_metadata(depth=2)로 자식 노드 수집
  │     → 자식별 매핑 여부 분류
  │       ├─ 매핑된 자식: get_context_for_code_connect로 코드 스니펫만 수집
  │       └─ 미매핑 자식: get_design_context로 변환
  │     → 둘을 조립하여 화면 코드 생성
  │
  ├─ 에셋 (이미지/일러스트/아이콘)
  │     → figma-asset-export 스킬 위임 (upload_assets + 옵션)
  │
  └─ visual-verifier baseline (검증 단계 진입 시점만)
        → get_screenshot(scale=2) — 디자인 파악 단계에서는 호출 금지
```

## 병렬 호출 규약

**병렬 가능 (반드시 병렬화):**
- 동일 fileKey의 **서로 다른 nodeId** 호출 (예: 컴포넌트 50개의 `get_design_context`)
- `get_metadata` + `get_variable_defs` (다른 종류, 의존 없음)
- 화면 N개의 자식 분해 단계 (Phase 1 정찰)

**직렬 필수:**
- `get_metadata` → `get_design_context` (구조 파악 후 변환)
- `get_code_connect_map` → `get_context_for_code_connect` (매핑 확인 후 코드 조회)
- `add_code_connect_map` → `send_code_connect_mappings` (등록 후 전송)

**병렬 호출 시 동시성 한도:**
- MCP 채널: 최대 5건 동시 (Rate limit 회피 + 메모리 보호)

```
// 좋은 예
Promise.all([
  get_design_context(file, comp1),
  get_design_context(file, comp2),
  get_design_context(file, comp3),
  get_design_context(file, comp4),
  get_design_context(file, comp5),
])

// 나쁜 예
for (comp of comps) await get_design_context(file, comp) // 직렬
```

## 도구별 옵션 표준

### get_metadata
- `depth=1`: 직계 자식만 (인벤토리 분류용)
- `depth=2`: 손자까지 (화면 자식 노드 분해)
- `depth>=3`: 금지 — 응답 폭발

### get_screenshot
- **호출 시점**: visual-verifier 진입 시점 한정. **디자인 파악·get_design_context·screen-implementer 입력으로 호출 금지** — Figma는 모든 메타데이터를 MCP 응답에 노출하므로 시각 추론은 불필요한 손실 압축.
- `scale=2` 기본 (비교용 정확도)
- `scale=3`: retina 배포 자산 추출 시
- `scale=1`: 빠른 미리보기만

### upload_assets (에셋용 — figma-asset-export 위임)
- `format`: `svg` | `png` | `webp` | `jpg`
- `scale`: 1 | 2 | 3 (raster만)
- 상세 규칙은 `figma-asset-export` 스킬 참조.

### get_design_context
- 호출 자체는 옵션 없음 (가장 무거움)
- **항상** 매핑 여부 사전 확인 필수

## 캐싱 정책

캐시 키: `{fileKey}__{nodeId}__{tool}__{optionsHash}`

- `tool`: get_metadata / get_design_context / get_screenshot / ...
- `optionsHash`: depth / scale / format 등 옵션 해시 (다른 옵션은 다른 캐시)

저장 위치: `_workspace/_cache/{key}.json` (또는 `.png` for screenshots)

**무효화 규칙:**
- 같은 세션 내 동일 키 호출 → 캐시 사용 (네트워크 호출 금지)
- 사용자가 "재추출/재정찰" 명시 → 해당 키만 무효화
- 사용자가 "전체 재실행" 명시 → 캐시 전부 폐기
- Figma 측 변경(브랜치 전환 등) → fileKey 자체가 다르므로 자동 무효화

**캐시 우선 호출 패턴:**
```
function fetch(key) {
  if (cache[key]) return cache[key]
  const result = call_mcp(...)
  cache[key] = result
  return result
}
```

## 에러 처리

| 에러 | 처리 |
|---|---|
| 인증 실패 (401/403) | 사용자에게 Figma 로그인 또는 Desktop 앱 실행 요청 후 중단. 자동 우회 채널 없음 (시각 기반 웹 UI 스크래핑은 정확도 손실로 폐기됨) |
| Dev Mode license required | 사용자에게 Dev Mode 좌석 부여 또는 라이선스 보유 계정으로 재인증 요청 후 중단 |
| 노드 없음 (404) | nodeId가 부모 페이지 ID일 가능성 — `get_metadata(depth=1)`로 자식 확인 후 재호출 |
| 타임아웃 | 1회 재시도. 재실패 시 더 작은 노드(자식)로 분할 호출 |
| Rate limit | exponential backoff (1s, 2s, 4s) 최대 3회 |
| 파일 너무 큼 | `get_metadata` → 자식 단위 분할, 절대 단일 호출 강행 금지 |
| `get_design_context` 응답 비정상 | nodeId가 컴포넌트가 아닌 페이지/프레임일 가능성 — `get_metadata`로 종류 재확인 |

## 출력 정규화 (참고용)

`get_design_context`는 React+Tailwind 코드를 반환하지만 **참조용**:
- shadcn/ui 매핑 가능한 부분은 shadcn 컴포넌트로 교체
- raw hex → 토큰 변수로 치환
- absolute positioning은 flexbox/grid로 재작성
- 매핑된 자식이 있으면 `get_context_for_code_connect`의 결과로 교체
