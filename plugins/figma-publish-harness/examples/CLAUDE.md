# 프로젝트 — Figma Publishing 기본값

> **사용법:** 이 파일을 **자기 프로젝트 루트의 `CLAUDE.md`**로 복사한 뒤 우리 회사·팀 컨벤션에 맞게 수정하세요. 플러그인 자체는 이 파일이 없어도 동작하지만, 매 실행마다 디자인 빈틈 7질문에 답변할 필요가 사라집니다.

---

## 하네스: figma-publish-harness

`figma-publish-harness` 플러그인이 설치되어 있으며, Figma URL/nodeId 또는 "퍼블리싱"·"화면 구현"·"Figma 변환" 표현이 보이면 `figma-publish-orchestrator` 스킬이 자동 트리거됩니다.

## 프로젝트 기본값 (orchestrator의 디자인 빈틈 7질문 자동 답변)

이 섹션을 채우면 orchestrator가 매 실행마다 묻지 않고 바로 진행합니다. 비워두면 사용자에게 묻습니다.

| 질문 | 답변 |
|------|------|
| Components/Variants 페이지 존재? | (예: `있음`, 또는 `없음 — 화면별 변환`) |
| 모바일 mockup 존재? | (예: `있음`, 또는 `없음 — mobile-first 추론`) |
| mock 단계 vs 실제 API? | (예: `mock + zod schema`, 또는 `실제 API: ${API_BASE_URL}`) |
| 아이콘/이미지/폰트 처리 | (예: `Tabler Icons + Google Fonts`) |
| 반응형 범위 | (예: `sm/md/lg/xl 전체`, 또는 `lg만`) |
| container query 사용 | (예: `사용`, 또는 `미사용`) |
| fluid typography(clamp) | (예: `사용`, 또는 `미사용`) |

## 스택 고정 (선택)

`stack-detector`가 자동 감지하지만, 명시적으로 박아두고 싶으면:

```
프레임워크: Vue 3 + Vite + TypeScript
상태관리: Pinia
라우팅: Vue Router
스타일링: Tailwind v4 (또는 SCSS / UnoCSS / Compose / Flutter / SwiftUI)
i18n: vue-i18n (또는 next-intl / react-i18next / svelte-i18n / intl-flutter)
Storybook: 사용 (또는 widgetbook for Flutter / 미사용)
화면 경로: src/pages/{kebab-case}.vue
컴포넌트 경로: src/components/{PascalCase}.vue
에셋 경로: public/assets/
i18n 메시지 경로: src/locales/{locale}.json
```

## 검증 임계치 (선택)

기본은 plugin 내장값. 회사 표준이 다르면 덮어쓰기:

```
visual diff 통과 기준: < 5%
a11y 기준: WCAG AA 위반 0건
LCP 예산: 2.5s
CLS 예산: 0.1
INP 예산: 200ms
번들 크기 예산 (initial JS): 200KB
pixel-diff-loop 최대 반복: 3회
```

## 도메인 규칙 (선택)

회사·서비스 특이 규칙:

```
- 모든 form은 zod 스키마 우선, class-validator는 DTO에서만
- 차트는 chart.js, 위젯 그리드는 gridstack
- 비디오 플레이어는 video.js, 리치 텍스트는 quill (sanitize 필수)
- 모든 API 응답은 envelope 패턴 ({ success, data, error })
- 라우팅은 history mode, 인증 토큰은 httpOnly 쿠키
```

## auto 모드

`퍼블리싱 auto`처럼 "auto" 키워드가 들어오면 orchestrator가 모든 빈틈에 합리적 디폴트를 적용하고 `_workspace/decisions.log`에 기록합니다. 위 기본값이 채워져 있으면 디폴트보다 그것을 우선 사용합니다.

## 변경 이력 (프로젝트 단위)

```
| 날짜 | 변경 내용 | 사유 |
|------|----------|------|
| YYYY-MM-DD | 초기 설정 | - |
```
