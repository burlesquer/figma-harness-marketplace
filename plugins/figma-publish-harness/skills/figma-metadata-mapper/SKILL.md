---
name: figma-metadata-mapper
description: Figma의 Auto Layout, Constraints, Component Properties, Variants 메타데이터를 React/Tailwind/CSS로 정확히 변환하는 매핑 표준. screen-implementer와 component-extractor가 사용. 디자이너 의도(여백/정렬/축/responsive 의도)를 코드에 정확히 보존하기 위한 유일한 경로.
---

# figma-metadata-mapper

`get_design_context`가 주는 코드는 raw 형태로 Figma 의도를 70% 정도만 반영한다. 이 스킬은 Auto Layout/Constraints/Component Property/Variants 4축을 정확히 코드로 매핑하여 나머지 30%를 회수한다.

## 1. Auto Layout → Flexbox/Grid

| Figma 속성 | CSS/Tailwind |
|---|---|
| `layoutMode: "HORIZONTAL"` | `flex flex-row` |
| `layoutMode: "VERTICAL"` | `flex flex-col` |
| `layoutMode: "GRID"` | `grid grid-cols-{n}` (자식 수 또는 명시 설정) |
| `itemSpacing: N` | `gap-[Npx]` 또는 토큰 매칭 시 `gap-{token}` |
| `paddingTop/Right/Bottom/Left` | `pt-/pr-/pb-/pl-` (대칭이면 `px-`/`py-`) |
| `primaryAxisAlignItems: "MIN"` | `justify-start` |
| `primaryAxisAlignItems: "CENTER"` | `justify-center` |
| `primaryAxisAlignItems: "MAX"` | `justify-end` |
| `primaryAxisAlignItems: "SPACE_BETWEEN"` | `justify-between` |
| `counterAxisAlignItems: "MIN/CENTER/MAX"` | `items-start/center/end` |
| `primaryAxisSizingMode: "AUTO"` | width/height: auto (hug) |
| `primaryAxisSizingMode: "FIXED"` | 명시 width/height |
| `layoutGrow: 1` | `flex-1` 또는 `grow` |
| `layoutAlign: "STRETCH"` | `self-stretch` |

**raw hex → 토큰 치환은 design-token-extractor 출력 기준.**

## 2. Constraints → CSS

| Figma horizontal | CSS 정렬 |
|---|---|
| `LEFT` | `left:0` (absolute) 또는 `ml-0` |
| `RIGHT` | `right:0` |
| `CENTER` | `left:50% translate-x-[-50%]` |
| `STRETCH` | `inset-x-0` |
| `SCALE` | container query 또는 `w-full` (비율 보존은 aspect-ratio) |

vertical도 동일 패턴(top/bottom/center/stretch).

**우선순위:** 부모가 Auto Layout이면 Constraint 무시 (Auto Layout이 우선). Constraint는 absolute 자식에만 적용.

## 3. Component Property → React Props

Figma Component Property 4종을 React props로 매핑:

| Figma type | React prop type | 예시 |
|---|---|---|
| `BOOLEAN` | `boolean` | `disabled`, `loading`, `selected` |
| `TEXT` | `string` | `label`, `description` |
| `INSTANCE_SWAP` | `ReactNode` 또는 `Component` | `icon`, `leftSlot` |
| `VARIANT` | `union literal` | `size: 'sm' \| 'md' \| 'lg'` |

**명명 규칙:**
- Figma `Show Icon#abc123` → React `showIcon: boolean`
- Figma `Label#xyz` → React `label: string`
- Figma `Icon#slot` (instance swap) → React `icon?: React.ReactNode`

**기본값:** Figma의 default value를 React `defaultProps` 또는 `?:` 연산자로 매핑.

## 4. Variants → Discriminated Union

Figma Variants는 다축 조합. 예: `Size=md, State=hover, Color=primary`

```tsx
type ButtonProps = {
  size: 'sm' | 'md' | 'lg'
  state?: 'default' | 'hover' | 'active' | 'disabled' // hover/active는 CSS pseudo로 처리, prop 불필요
  color: 'primary' | 'secondary' | 'danger'
  children: React.ReactNode
}
```

**중요:** `hover/focus/active/disabled`는 prop이 아니라 **CSS pseudo class**로 매핑. props로 두면 안 됨.

```tsx
// 좋은 예
<button className="bg-brand-500 hover:bg-brand-600 active:bg-brand-700 disabled:opacity-50">

// 나쁜 예
<button state="hover">
```

`disabled`만 prop이자 attribute (HTML 표준).

## 5. 특수 노드 처리

| Figma 노드 | 변환 |
|---|---|
| `MASK` 그룹 | CSS `clip-path` 또는 `mask-image` |
| `BLEND_MODE != NORMAL` | CSS `mix-blend-mode` |
| `EFFECT: DROP_SHADOW` | `box-shadow` (토큰 매핑 우선) |
| `EFFECT: INNER_SHADOW` | `box-shadow: inset` |
| `EFFECT: LAYER_BLUR` | `filter: blur(Npx)` |
| `EFFECT: BACKGROUND_BLUR` | `backdrop-blur-[Npx]` |
| `STROKE` (border) | `border border-{token} border-[Npx]` |
| `cornerRadius` 비대칭 | `rounded-tl-/tr-/bl-/br-` |

## 6. Hidden / Locked / Ignore-in-export

Figma 노드의 메타 속성을 무시하지 말 것:
- `visible: false` → 코드에서 제외 (렌더링 안 함)
- `locked: true` → 디자이너 의도: 변경 금지. 매핑된 컴포넌트 우선
- 노드 이름에 `_ignore`, `_export-only`, `__skip__` 접두/접미 → export 제외

이는 `get_metadata` 응답에서 확인 가능.

## 7. Prototyping 인터랙션

Figma의 prototype 링크 (클릭 → 다른 frame) 정보를 활용:
- `reactions` 또는 `prototypeStartNodeID` 메타데이터 발견 시
- `onClick={() => router.push('/path')}` 또는 `<Link href="...">` 변환
- 상세는 향후 `figma-prototype-mapper` 분리 가능 (현재는 메모만)

## 8. 검증 체크리스트

변환 후 자체 검증:
- [ ] Auto Layout 노드는 absolute positioning 아님
- [ ] Hover/Focus/Active는 prop 아닌 pseudo class
- [ ] Hidden 노드는 코드에 없음
- [ ] Component Property 이름이 React 컨벤션(camelCase) 따름
- [ ] Variants 중 CSS state(hover 등)는 union에서 제거됨
- [ ] gap/padding은 토큰 우선, 임의 px 최소화

## 9. 사용 시점

- screen-implementer가 화면 코드 생성 시 모든 노드에 적용
- component-extractor가 UI Kit 부품 변환 시 props 도출에 적용
- raw `get_design_context` 출력 → 본 매핑 적용 → 최종 코드
