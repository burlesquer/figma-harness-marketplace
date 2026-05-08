# figma-publish-harness

Figma 디자인 파일을 프로덕션 프론트엔드 코드로 변환하는 멀티 에이전트 하네스. 스택 자동 감지부터 검증·자동 수정 루프까지 전체 파이프라인을 자동화한다.

## 무엇을 하는가

Figma URL 또는 nodeId를 받으면:

1. **스택 감지** — 기존 코드베이스 분석 또는 사용자 입력으로 React/Vue/Svelte/Solid/Angular/Flutter/SwiftUI 등 결정 (`_stack.json`)
2. **정찰** — UI Kit(Components/Variants)과 화면(Screens) 분리 식별, 인벤토리 작성
3. **디자인 토큰 추출** — Figma Variables → tokens.css/tailwind config (스택별 출력)
4. **컴포넌트 매핑** — UI Kit 부품을 코드 컴포넌트로 변환, Code Connect 등록
5. **화면 구현** — 토큰·매핑된 컴포넌트로 페이지 코드 생성, isolation 분석으로 병렬 처리
6. **에셋 export** — SVG/PNG/WebP 추출, retina 대응
7. **i18n + Storybook 생성** — 하드코딩 텍스트 키 추출, 컴포넌트별 stories
8. **검증 (3-병렬)** — visual diff + a11y/e2e + Lighthouse/Web Vitals
9. **자동 수정 루프** — diff > 5%면 generator ↔ evaluator GAN 패턴 (최대 3회)
10. **SEO + 보고** — title/description/OG/JSON-LD 생성, 시간/비용 측정 집계

## 트리거

다음 표현이 보이면 `figma-publish-orchestrator` 스킬이 자동 활성화:
- Figma URL (`figma.com/design/...`) 또는 nodeId
- "퍼블리싱", "화면 구현", "Figma 변환"
- 후속 작업: "다시 실행", "Dashboard만 다시", "토큰 재추출", "diff 줄여" 등

## 설치

마켓플레이스 추가 후 플러그인 설치:

```
/plugin marketplace add <repo-url>
/plugin install figma-publish-harness@figma-harness-marketplace
```

또는 로컬에서 테스트:

```
/plugin install C:/path/to/plugins/figma-publish-harness
```

## 의존성

이 플러그인은 다음 외부 자원을 사용한다 (사용자가 별도 설치 필요):

- **Figma MCP 플러그인** — 디자인 추출 (`get_design_context`, `get_screenshot`, `get_metadata`, `get_variable_defs`, `create_design_system_rules`, `get/add_code_connect_map`, `upload_assets`)
- **everything-claude-code (ECC) 플러그인** — `design-system`, `dashboard-builder`, `frontend-patterns`, `nextjs-turbopack`/`nuxt4-patterns`, `accessibility`, `browser-qa`, `e2e-testing`, `verification-loop`, `documentation-lookup` 스킬 + `gan-design`/`gan-build` 명령

## 구성

### 에이전트 (15)

| 에이전트 | 역할 |
|---------|------|
| `stack-detector` | 프로젝트 스택 자동 감지 (`_stack.json`) |
| `figma-scout` | UI Kit과 화면 분리 식별, 인벤토리 출력 |
| `design-token-extractor` | Variables → tokens.css/tailwind config |
| `component-extractor` | UI Kit 부품 → 코드 컴포넌트 (병렬 호출) |
| `component-mapper` | Code Connect 매핑 등록 |
| `screen-isolation-analyzer` | 화면 간 자원 의존 분석 → 병렬 그룹 결정 |
| `screen-implementer` | 화면 노드 → 페이지 코드 변환 (핵심) |
| `asset-handler` | SVG/PNG/WebP export, manifest 작성 |
| `i18n-extractor` | 하드코딩 텍스트 → i18n 키 |
| `story-generator` | Storybook stories 자동 생성 |
| `visual-verifier` | Figma vs 구현 시각 diff + hotspot |
| `quality-auditor` | a11y(WCAG AA) + e2e |
| `perf-auditor` | Lighthouse + Web Vitals + bundle |
| `seo-meta-writer` | title/description/OG/JSON-LD/sitemap |
| `design-diff-watcher` | 노드 hash로 변경분만 재변환 |

### 스킬 (6)

| 스킬 | 역할 |
|------|------|
| `figma-publish-orchestrator` | 전체 파이프라인 조율 (메인) |
| `figma-node-fetcher` | Figma MCP 호출 표준화·캐싱·병렬 규약 |
| `figma-asset-export` | 에셋 분류 트리·format/scale·SVG 최적화 |
| `figma-metadata-mapper` | Auto Layout/Constraints/Property/Variants → 코드 매핑 |
| `screen-spec-writer` | 8상태 매트릭스로 화면 spec 작성 |
| `pixel-diff-loop` | GAN 패턴 자동 수정 루프 (최대 3회) |

## 디자인 빈틈 정책

부족한 정보는 7가지 질문으로 사용자 확인 (auto 모드 또는 프로젝트 CLAUDE.md에 기본값이 있으면 자동 적용):

1. Components/Variants 페이지 존재?
2. 모바일 mockup 존재?
3. mock 단계 vs 실제 API?
4. 아이콘/이미지/폰트 처리 방식?
5. 반응형 범위 (sm/md/lg/xl)?
6. container query 사용?
7. fluid typography 사용?

## 프로젝트 기본값 사전 등록 (선택, 권장)

매 실행마다 7질문에 답변하기 싫으면, [`examples/CLAUDE.md`](./examples/CLAUDE.md)를 자기 프로젝트 루트의 `CLAUDE.md`로 복사·수정하세요. 회사 컨벤션(스택·반응형 범위·검증 임계치·도메인 규칙)을 박아두면 orchestrator가 묻지 않고 바로 진행합니다.

> 이 파일은 **선택 사항**입니다. 플러그인 자체는 이 파일 없이도 동작합니다 (트리거는 스킬 description에 이미 들어있음).

## 핵심 설계 결정

- **Figma MCP 단일 채널** — 시각 추론 기반 웹 UI 스크래핑은 정확도 손실로 폐기. 인증 실패 시 사용자 로그인 요청
- **스크린샷은 디자인 파악 입력 금지** — 모든 메타데이터는 MCP 응답에 노출됨, 시각 추론은 손실 압축. 스크린샷은 `visual-verifier` baseline 캡처에만 사용
- **스택 일반화** — `_stack.json` 기반으로 모든 에이전트 분기 (Tailwind/CSS/UnoCSS/Compose/Flutter/SwiftUI 등)
- **isolation 그룹 병렬** — 화면 간 자원 공유 분석 후 그룹 단위 동시 처리 (race condition 방지)
- **Code Connect 사전 조회** — 매핑된 노드는 즉시 코드 스니펫 사용, 미매핑만 변환

## 출력

```
{프로젝트 루트}/
├── _workspace/           ← 중간 산출물 (감사 추적용)
│   ├── _stack.json
│   ├── 01_scout_inventory.json
│   ├── 02_design_tokens.json
│   ├── 03_component_map.json
│   ├── 04_code_connect_registry.json
│   ├── 05_screens/{screen-id}.json
│   ├── 06_asset_manifest.json
│   ├── 07_visual_report/{screen-id}.json
│   ├── 08_quality_report/{screen-id}.json
│   ├── 09_loop_report/{screen-id}.json
│   ├── decisions.log
│   └── REPORT.md
├── src/                  ← 최종 화면 코드 (스택별 경로)
├── public/assets/        ← 추출된 에셋
└── messages/             ← i18n 메시지
```

## 변경 이력

CHANGELOG.md 참조. 핵심:
- 2026-05-07: 초기 구성 (15 에이전트 + 6 스킬)
- 2026-05-08: Figma MCP 단일 채널로 일원화 (Playwright 웹 UI 스크래핑 폐기)

## 라이선스

MIT
