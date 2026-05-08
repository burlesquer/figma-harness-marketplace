---
name: figma-publish-orchestrator
description: Figma 디자인을 코드로 옮기는 전체 퍼블리싱 파이프라인을 조율. Figma URL 또는 nodeId, "퍼블리싱", "화면 구현", "Figma 변환" 같은 요청에서 반드시 사용. 화면 1개부터 9개까지 모두 처리. 재실행/부분수정/업데이트 요청도 이 스킬이 처리.
---

# figma-publish-orchestrator

Figma 디자인 → 프로덕션 프론트엔드 코드 변환의 메인 조율자. 8명의 전문 에이전트와 ECC 스킬, Figma MCP 도구를 엮어 화면을 변환·검증·수정한다.

## Phase 0: 컨텍스트 확인 + 인증 + 스택 감지 + 브랜치 매핑 (필수 첫 단계)

### 0-1. 인증 확인
- figma-node-fetcher 의무 호출에 따라 **`whoami` + `get_libraries` 1회 호출**
- 인증 실패 시 사용자에게 Figma 로그인/Desktop 앱 실행 요청, 후속 작업 막음

### 0-1.5. 스택 감지 (필수)
- **`stack-detector` 에이전트 호출** → `_workspace/_stack.json` 생성
- 기존 코드(`package.json`/`pubspec.yaml` 등) 발견 시 ECC `codebase-onboarding`로 자동 감지
- 빈 프로젝트면 사용자에게 스택 입력 요청
- 감지 결과를 사용자에게 보여주고 확인 단계 거침 (auto 모드면 스킵)
- 후속 모든 에이전트는 `_stack.json`만 보고 출력 형식 결정 → React/Vue/Svelte/Flutter 등 무관하게 동작

### 0-2. 실행 모드 결정
`_workspace/` 디렉토리 상태로:

| 상태 | 모드 | 동작 |
|---|---|---|
| `_workspace/` 없음 | 초기 실행 | Phase 1부터 전체 실행 |
| 있음 + 새 figmaUrl/nodeId | 새 실행 | `_workspace_prev/`로 이동 후 새로 시작 |
| 있음 + 부분 수정 요청 ("Dashboard만 다시", "토큰 재추출") | 부분 재실행 | 해당 에이전트만 재호출 |
| 있음 + 동일 입력 | 재개 | 가장 마지막 미완료 Phase부터 |
| 있음 + 디자인 변경 의심 | **diff 기반 재실행** | `design-diff-watcher` 호출 → 변경 노드만 재변환 |

### 0-3. 브랜치 매핑
Figma 파일이 branch URL이거나 git 브랜치 매핑이 필요할 때:
- `_workspace/_branch_map.json`에 `{figmaBranch, gitBranch}` 기록
- 사용자가 새 git 브랜치를 명시하면 매핑 갱신
- 매핑된 git 브랜치로 체크아웃 권장 (자동 체크아웃 금지, 사용자 확인 필수)

### 0-4. 자동 진행 모드 (`--auto` 플래그 또는 "auto" 키워드)
- 디자인 빈틈 4질문에 합리적 디폴트로 자동 진행:
  - Components/Variants 페이지 → 있으면 사용, 없으면 화면별 변환
  - 모바일 mockup 없으면 → mobile-first 추론
  - mock vs API → 사용자 미지정 시 mock + zod schema
  - 폰트 → Google Fonts 또는 시스템 폰트 폴백
- 모든 결정은 `_workspace/decisions.log`에 기록 (사용자가 사후 검증 가능)
- 단, 인증 실패·치명적 충돌은 자동 진행 금지 (사용자 개입 필수)

## Phase 1: 입력 파싱 + 정찰 + 반응형 합의

1. 사용자 입력에서 추출:
   - Figma URL → fileKey, nodeId (`-` → `:` 변환)
   - 스택 (Next.js 15 + Tailwind v4 + shadcn/ui 권장 / Nuxt 4 / 기존 통합)
   - 우선 화면 (없으면 figma-scout이 인벤토리 출력 후 사용자에게 선택 요청)
2. **디자인 빈틈 정책 — 7개 질문 (auto 모드면 디폴트 적용; 스택은 Phase 0-1.5에서 자동 감지하므로 질문에서 제외)**:
   - Components/Variants 페이지 존재?
   - 모바일 mockup 존재?
   - mock 단계 vs 실제 API?
   - 아이콘/이미지/폰트 처리 방식?
   - **반응형 범위**: sm/md/lg/xl 중 어디까지 추론?
   - **container query** 사용 여부?
   - **fluid typography** (clamp) 사용 여부?
3. **실행 모드: 서브 에이전트** (단일 정찰)
4. `figma-scout` 호출 → `_workspace/01_scout_inventory.json` 생성
5. 인벤토리 검토 후 사용자에게 화면 선택 확인

## Phase 1.5: 화면 그룹핑 + 공유 레이아웃 추출 (대규모 처리)

화면이 10개 이상일 때 또는 공통 레이아웃(Header/Sidebar/Footer)이 명확할 때:

1. figma-scout 인벤토리에서 화면 클러스터링 (이름 prefix·폴더 구조 기반)
2. 공유 레이아웃 후보 식별:
   - 모든 화면에 동일 Header 존재 → 공통 레이아웃 1개 추출
   - 동일 Sidebar/Nav → 공통 레이아웃에 포함
3. Next.js `app/(group)/layout.tsx` 또는 Nuxt `layouts/`로 변환
4. 화면들은 layout 내부 page만 담당 (재구현 비용 감소)

## Phase 2: 토큰 + 컴포넌트 + 매핑

**실행 모드: 서브 에이전트 (순차 + 내부 병렬)**

순차 실행 (의존성: token → component → mapper). 단, 각 에이전트 **내부에서는 figma-node-fetcher 규약에 따라 MCP 호출을 병렬화**한다 (동시성 5).

1. `design-token-extractor` → `_workspace/02_design_tokens.json` + tokens.css/tailwind config
2. `component-extractor` → `_workspace/03_component_map.json` + 실제 컴포넌트 파일들 (UI Kit 50개라도 병렬 호출)
3. `component-mapper` → `_workspace/04_code_connect_registry.json`

## Phase 2.5: 화면 isolation 분석 (병렬화 안전성)

화면 처리 전에 `screen-isolation-analyzer` 호출:
- 입력: 사용자가 선택한 화면 ID 리스트
- 출력: `_workspace/_isolation_groups.json` (그룹별 `parallelSafe`, `sharedResources` 명시)
- 이후 Phase 3는 이 분석 결과대로 동시 인스턴스 수와 작업 분배 결정

## Phase 3: 에셋 선행 + 화면 구현 (팀, isolation-aware 병렬)

**실행 모드: 에이전트 팀**

**핵심 변경:**
1. asset-handler를 screen-implementer **앞**에 배치 (에셋이 먼저 자리잡혀야 import 가능).
2. isolation 그룹 N개에 따라 screen-implementer 인스턴스 N개 spawn (group 1개당 instance 1개, 최대 5).
3. 그룹 내부는 직렬 처리(자원 공유 안전), 그룹 간 병렬.

```
TeamCreate("figma-publish-impl", [
  "asset-handler",            // 공유, fan-in
  "i18n-extractor",            // 공유, fan-in (Phase 3.5와 통합)
  "screen-implementer-G1",    // group G1 화면 처리
  "screen-implementer-G2",    // group G2 화면 처리
  "screen-implementer-G3"     // group G3 화면 처리 (isolation 분석 결과대로)
])
```

순서:
1. asset-handler가 모든 그룹의 에셋 인벤토리를 fan-in 통합 → manifest 작성 → 모든 screen-implementer 인스턴스에 import 경로 통지
2. 각 screen-implementer-Gx 인스턴스가 자기 그룹의 화면을 직렬 처리 (그룹 내부)
3. 인스턴스 간은 병렬 (그룹 간 자원 공유 없음, isolation-analyzer가 보장)
4. 공유 자원(에셋/i18n)은 fan-in 단계에서 통합 (race condition 방지)

### 화면 처리 병렬화 임계치 (isolation-analyzer 결과로 결정)

| isolation 그룹 수 | 처리 방식 |
|---|---|
| 그룹 1개 (모두 공유 자원) | 직렬 처리 (race 위험) |
| 그룹 2~3개 | screen-implementer 인스턴스 2~3개 병렬 |
| 그룹 4~5개 | 인스턴스 4~5개 병렬 (한도 5) |
| 그룹 6개+ | 5개씩 배치 처리 (그룹 큐) |

화면별 검증은 Phase 4·5에서 진행. 검증 실패한 화면만 수정 루프 진입.

> 위 임계치는 isolation-analyzer가 모든 화면을 그룹화한 결과로 자동 결정. 사용자가 수동 강제하려면 `--no-parallel` 플래그로 직렬화 가능.

## Phase 3.5: i18n + Storybook 생성 (병렬)

화면 코드 생성 직후, 다음 두 작업을 **병렬**로:
- `i18n-extractor` — 하드코딩 텍스트 추출 + `messages/{locale}.json` 생성
- `story-generator` — 화면·컴포넌트별 Storybook stories 생성

이 단계는 화면 코드가 만들어진 직후, Phase 4 검증 진입 전에 수행.

## Phase 4: 검증 (팀, 3-병렬)

**실행 모드: 에이전트 팀**

```
TeamCreate("figma-publish-verify", [
  "visual-verifier",
  "quality-auditor",
  "perf-auditor"
])
```

3개 검사 병렬:
- `visual-verifier`: Figma vs 구현 시각 diff + hotspot 시각화 + VRT 베이스라인 등록
- `quality-auditor`: a11y(색맹/모션/포커스/landmarks) + e2e
- `perf-auditor`: Lighthouse + Web Vitals + bundle

세 검사 모두 통과해야 화면 완료. 하나라도 실패 → Phase 5로.


## Phase 5: 자동 수정 루프 (서브)

**실행 모드: 서브 에이전트**

`pixel-diff-loop` 스킬 호출 → ECC `gan-design` + `gan-build` 명령으로 generator(screen-implementer) ↔ evaluator(visual-verifier/quality-auditor) 자동 루프.

- 최대 3회 반복
- 3회 후에도 실패: 사용자에게 보고하고 수동 개입 요청
- 통과: 다음 화면으로

## Phase 5.5: SEO 메타데이터 (서브)

검증 통과한 화면들에 대해 `seo-meta-writer` 호출:
- title/description/OG image/JSON-LD/canonical 생성
- sitemap.xml + robots.txt 1회 생성

## Phase 6: 최종 보고 + 측정 집계

`_workspace/REPORT.md` 생성:
- 화면별 상태 (완료 / 부분 / 실패)
- 시각 diff %, a11y 위반 수, e2e 통과율, **Lighthouse 점수, LCP/CLS/INP**
- **diff hotspot** 위치 시각화 (PNG 첨부)
- 수동 개입 필요 항목 목록
- **시간/비용 측정 집계** (`_workspace/_meta/*.json` 합산):
  - 총 소요 시간, Phase별 시간
  - MCP 호출 횟수 (도구별)
  - 토큰 사용량 (가능한 경우)
- **결정 로그 요약** (`_workspace/decisions.log`):
  - 자동 결정 N건, 사용자 확인 N건
  - 합리적 디폴트 적용 항목

## 데이터 전달 프로토콜

- **메시지 기반**: 팀 모드에서 SendMessage (`screen-implementer ↔ asset-handler`, `visual-verifier ↔ screen-implementer`)
- **태스크 기반**: TaskCreate로 화면별 작업 추적
- **파일 기반**: 모든 중간 산출물은 `_workspace/{phase}_{agent}_{artifact}.{ext}`
- 최종 화면 코드는 사용자 지정 src 경로

### 관측·추적 (모든 Phase 공통)

각 에이전트는 작업 시작·종료 시 다음을 기록:

1. **결정 로그** (`_workspace/decisions.log` — TSV 형식):
   ```
   2026-05-07T12:34:56Z\tscreen-implementer\tnew\tno-mapping-found-for-63:30210
   2026-05-07T12:35:02Z\tasset-handler\tsvg-export\tcustom-icon-not-in-lucide
   ```
   - 컬럼: `timestamp(ISO 8601 UTC)`, `agent`, `choice`, `reason`
   - 합리적 디폴트 적용·매핑 결정·라이브러리 선택 등 모두 기록

2. **시간/비용 측정** (`_workspace/_meta/{phase}_{agent}.json`):
   ```json
   { "duration_ms": 12480, "mcpCallCount": 23, "tokens": 18450 }
   ```
   - Phase 6 최종 보고에 집계

## 에러 핸들링

- 에이전트 1회 재시도 후 재실패: 보고서에 누락 명시하고 다음으로 진행
- 상충 정보(예: 토큰 충돌): 삭제하지 않고 원본·변환본 병기
- 의존성 미설치(Playwright 등): 오케스트레이터가 사용자에게 설치 요청

## 테스트 시나리오

**정상 흐름**:
1. 사용자: "https://figma.com/design/.../63-29809 Dashboard 화면 Next.js로 퍼블리싱"
2. Phase 0: `_workspace/` 없음 → 초기 실행
3. Phase 1: figma-scout이 인벤토리 출력 → 사용자에게 Dashboard 화면 1개 확인
4. Phase 2~5 자동 진행 → `src/app/dashboard/page.tsx` 생성
5. Phase 6: REPORT.md, 화면 완료 1/1

**에러 흐름**:
1. visual-verifier diff 8% → pixel-diff-loop 1회 → diff 6%
2. 2회 → diff 4% (목표 5% 통과)
3. 다음 화면으로

## 후속 작업 키워드

이 스킬을 트리거할 후속 표현:
- "다시 실행", "재실행", "퍼블리싱 다시"
- "Dashboard만 다시", "토큰 재추출"
- "이 부분 수정해서 재구현"
- "결과 개선", "diff 줄여"
- "auto 모드", "자동 진행", "질문 없이"
- "디자인 변경분만", "diff 기반 재변환"
- "i18n 추가", "다국어", "스토리북", "Storybook"
- "다크모드", "테마"
- "SEO", "OG 이미지", "메타데이터"
- "성능 측정", "Lighthouse", "Web Vitals"
