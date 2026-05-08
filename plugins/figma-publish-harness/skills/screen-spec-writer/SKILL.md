---
name: screen-spec-writer
description: Figma get_design_context의 raw 결과를 실제 구현 가능한 화면 스펙으로 변환. 반응형 정책, 상태(states), 컴포넌트 매핑, 데이터 바인딩 결정을 통합한 단일 스펙 문서를 만든다. screen-implementer가 사용.
---

# screen-spec-writer

`get_design_context`의 출력은 정적이고 한 viewport만 반영한다. 이 스킬은 그 결과에 **반응형 정책**, **상태 정책**, **데이터 정책**을 결합해 구현 가능한 스펙을 만든다.

## 입력

1. `mcp__plugin_figma_figma__get_design_context(fileKey, nodeId)` 결과
2. `_workspace/_stack.json` (스택 — 표기 언어 결정)
3. `_workspace/02_design_tokens.json` (토큰)
4. `_workspace/04_code_connect_registry.json` (매핑된 컴포넌트)
5. 5.3장 **디자인 빈틈 정책** 답변 (오케스트레이터가 전달)

## 출력 스펙 구조

스펙 파일 자체는 JSON(`.spec.json`)이며 모든 스택에서 동일 사용. 아래 TS interface는 **읽기 가이드**일 뿐, 실제 출력은 JSON.

```typescript
interface ScreenSpec {
  screenId: string;
  screenName: string;

  layout: {
    rootLayout: "flex" | "grid" | "container";
    breakpoints: {
      sm?: LayoutVariant;
      md?: LayoutVariant;
      lg: LayoutVariant; // 기본 (Figma 원본)
      xl?: LayoutVariant;
    };
  };

  components: Array<{
    figmaNodeId: string;
    role: "mapped" | "new" | "asset";
    codeRef?: string;     // mapped일 때
    spec?: NewSpec;       // new일 때
    props: Record<string, any>;
  }>;

  states: Array<{
    name: "loading" | "empty" | "error" | "success" | string;
    visibility: "always" | "conditional";
    variant: ComponentVariant;
  }>;

  data: {
    source: "mock" | "api";
    schema: ZodSchema;
    fixtures?: any[]; // mock일 때
  };

  interactions: Array<{
    trigger: string;          // "click .save-btn"
    action: string;           // "POST /api/tasks"
    optimistic?: boolean;
  }>;

  a11y: {
    landmarks: string[];      // ["main", "navigation"]
    headingHierarchy: string[]; // ["h1", "h2", "h2", "h3"]
    focusOrder: string[];     // [".btn-primary", ".btn-secondary"]
  };
}
```

## 변환 규칙

### 1. 반응형 추론
Figma 원본이 lg(1440)만 있으면:
- `mobile-first` 정책 → sm/md 레이아웃 추론:
  - sidebar → drawer (sm), narrow (md)
  - 3컬럼 grid → 1컬럼 (sm), 2컬럼 (md)
  - 가로 nav → 햄버거 (sm)
- `desktop-only` 정책 → 추론 생략, lg만 사용

### 2. 상태 추론 — 8상태 매트릭스

`get_design_context` 결과에 명시적 variants 없으면 컴포넌트 종류별 상식적 기본값:

| 컴포넌트 종류 | 필요 상태 (필수) | 추가 상태 (선택) |
|---|---|---|
| Button/Link | default, hover, focus, active, disabled | loading (async action) |
| Input/Textarea/Select | empty, focused, filled, error, disabled | success, hint |
| Form | idle, validating, submitting, success, error | dirty, pristine |
| Data Table/List/Grid | loading-skeleton, empty, error, success | filtered, sorted, paginated |
| Card | default, hover (clickable), disabled | selected, expanded |
| Modal/Dialog | closed, opening, open, closing | full-screen, dismissable |
| Toast/Snackbar | hidden, entering, visible, exiting | success, warning, error variants |
| Navigation/Tab | default, active, hover, disabled | with-badge, with-icon |

**중요 구분:**
- **CSS pseudo class로 처리**: hover/focus/active/disabled — `<button className="hover:..."`. props 아님.
- **데이터 상태 prop**: loading/empty/error/success — `<Table state="loading" />`. 명시적 prop.
- **UI 상태 prop**: open/expanded/selected — `<Modal open />`. 명시적 prop.

### 2-1. Loading skeleton 의무화

데이터 페칭이 있는 화면은 반드시 skeleton 컴포넌트 정의:
- `<Skeleton className="h-4 w-32" />` (단일)
- `<TableSkeleton rows={5} columns={4} />` (테이블)
- 화면 단위 fallback: `<DashboardSkeleton />`

### 2-2. Error boundary

각 화면은 React Error Boundary로 감싸 fallback UI 정의 (`error.tsx` in Next.js).

### 3. 데이터 바인딩
- `mock`: zod schema 정의 + faker로 fixture 5~10개
- `api`: 사용자 제공 OpenAPI 스펙 또는 추정 endpoint
- 차트: recharts/visx 권장, mock 시계열 데이터

### 4. 컴포넌트 매핑 우선순위
1. Code Connect registry에 있음 → `codeRef` 사용
2. `_stack.json`의 `component_library`에 동등 부품 있음 → 해당 라이브러리 컴포넌트 사용
3. 둘 다 없음 → 신규 spec

### 5. raw hex → 토큰 치환
`get_design_context` 출력의 `text-[#3B82F6]` 같은 표현 발견 시:
- `02_design_tokens.json`에서 매칭 토큰 검색
- 매칭 시 `text-brand-500` (Tailwind config 기준)
- 미매칭 시 토큰 누락 보고

## 검증

스펙 작성 후 자체 검증:
- [ ] 모든 컴포넌트에 role 지정됨
- [ ] 토큰만 사용 (raw hex 없음)
- [ ] 모든 상태에 변형 정의됨
- [ ] a11y landmarks/heading 계층 적절
- [ ] mock 데이터 fixture 또는 API endpoint 존재

## 출력 위치

`_workspace/05_specs/{screen-id}.spec.json`

screen-implementer는 이 스펙만 보고 구현하면 됨 (Figma 재호출 불필요).
