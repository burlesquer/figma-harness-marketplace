# Changelog

이 플러그인의 모든 주요 변경 내용을 기록한다.

## 0.1.0 — 2026-05-08

### 초기 공개

- **에이전트 15개**: stack-detector, figma-scout, design-token-extractor, component-extractor, component-mapper, screen-isolation-analyzer, screen-implementer, asset-handler, i18n-extractor, story-generator, visual-verifier, quality-auditor, perf-auditor, seo-meta-writer, design-diff-watcher
- **스킬 6개**: figma-publish-orchestrator, figma-node-fetcher, figma-asset-export, figma-metadata-mapper, screen-spec-writer, pixel-diff-loop

### 핵심 설계 원칙

- **Figma MCP 단일 채널** — 시각 추론 기반 웹 UI 스크래핑(Playwright fallback)은 정확도 손실로 폐기. 인증 실패 시 사용자 로그인 요청
- **스택 일반화** — `_stack.json` 기반 분기로 React/Vue/Svelte/Solid/Angular/Flutter/SwiftUI 등 어느 스택에도 동작
- **isolation 그룹 병렬** — screen-isolation-analyzer가 화면 간 자원 공유 분석 후 그룹 단위 동시 처리 (race condition 방지)
- **Code Connect 사전 조회** — 매핑된 노드는 코드 스니펫 즉시 사용, 미매핑만 `get_design_context` 변환
- **스크린샷은 디자인 파악 입력 금지** — 모든 메타데이터는 MCP 응답에 노출됨, 시각 추론은 손실 압축. visual-verifier baseline 캡처에만 사용
- **자동 수정 루프** — `pixel-diff-loop`로 generator(screen-implementer) ↔ evaluator(visual-verifier/quality-auditor) GAN 패턴, 최대 3회

### 주요 기능

- 디자인 빈틈 7질문 (auto 모드면 합리적 디폴트)
- 노드 hash 캐시로 변경분만 재변환 (design-diff-watcher)
- 다크/라이트 모드 단일 모드 처리 (라이트 우선)
- 8상태 매트릭스 (Button/Input/Form/Table/Card/Modal/Toast/Nav)
- WCAG AA + 색맹 시뮬 + prefers-reduced-motion + focus trap + landmark roles
- Lighthouse + Web Vitals (LCP/CLS/INP) + bundle 게이트
- next-intl/react-i18next/vue-i18n/svelte-i18n/angular-localize/intl-flutter + ARB 출력
- Storybook (React/Vue/Svelte/Solid/Angular/RN) + widgetbook(Flutter)
- SEO 메타(title/description/OG/JSON-LD) + sitemap/robots
- 결정 로그(`decisions.log`) + 시간/비용 측정(`_meta/*.json`)
- 부분 재실행/디자인 변경분만 재변환 지원
