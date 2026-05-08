---
name: figma-asset-export
description: Figma 에셋(아이콘/일러스트/사진/배경)을 종류별로 분류하여 무손실·고화질로 export하고 srcset/SVG 최적화까지 표준화. asset-handler가 사용. "에셋 추출", "이미지/아이콘/일러스트 export" 요청 시 반드시 사용.
---

# figma-asset-export

에셋의 시각 품질을 보장하는 유일한 경로. **format/scale 옵션 없는 export는 모두 1x PNG로 떨어져 화면이 흐려진다.** 이 스킬을 거치지 않은 export는 금지.

## 에셋 분류 트리

```
노드 식별
  │
  ├─ vector + 단색/2~3색 + 24x24 이하 + 의미적 (검색/메뉴/X)
  │     → 아이콘
  │       ├─ lucide-react / heroicons에 동등
  │       │     → 라이브러리 import (export 불필요)
  │       └─ 커스텀
  │             → SVG export (currentColor 치환)
  │
  ├─ vector + 다색/그라데이션 + 큰 사이즈 + 장식적
  │     → 일러스트
  │       → SVG export (필수, 최적화 필수)
  │
  ├─ raster (사진/스크린샷)
  │     → 사진
  │       → WebP @1x/@2x/@3x + srcset
  │
  ├─ raster (배경/패턴/텍스처)
  │     → 배경
  │       → WebP @1x/@2x (CSS background-image용)
  │
  └─ 폰트
        → 라이브러리 (Google Fonts → next/font, Adobe Fonts → kit)
        → 라이선스 파일 → 사용자 제공 요청
```

## Export 옵션 표준

### 아이콘 (커스텀 SVG)
```
upload_assets({
  fileKey,
  nodeIds: [...],
  format: "svg",
  options: { stripStyles: true, useCurrentColor: true }
})
```
- 후처리: `fill="#XXX"` → `fill="currentColor"` 치환 (CSS로 색 제어)
- 후처리: `id`, `data-*`, Figma 주석 제거
- 저장: `src/assets/icons/{kebab-name}.svg`
- React 컴포넌트화: `src/components/icons/{PascalName}.tsx` (선택)

### 일러스트 (SVG)
```
upload_assets({
  fileKey,
  nodeIds: [...],
  format: "svg"
})
```
- 후처리: SVGO 최적화 (불필요 메타데이터 제거, path 단순화)
- 큰 SVG(>50KB): 한 번 더 검토 — 비트맵 변환이 더 가벼울 수 있음
- 저장: `src/assets/illustrations/{kebab-name}.svg`
- 색이 토큰과 일치 시 → `currentColor` 또는 CSS 변수 치환 검토

### 사진 (raster)
```
upload_assets({ fileKey, nodeIds: [...], format: "webp", scale: 1 })
upload_assets({ fileKey, nodeIds: [...], format: "webp", scale: 2 })
upload_assets({ fileKey, nodeIds: [...], format: "webp", scale: 3 })
```
- 3개 사이즈 모두 추출 (1x/2x/3x)
- 저장: `public/images/{name}.webp`, `{name}@2x.webp`, `{name}@3x.webp`
- 폴백 PNG도 1개 추출 (오래된 브라우저용): `format: "png", scale: 2`
- HTML 사용:
  ```html
  <img
    src="/images/hero.webp"
    srcset="/images/hero.webp 1x, /images/hero@2x.webp 2x, /images/hero@3x.webp 3x"
    alt="..."
    loading="lazy"
  />
  ```
- Next.js `<Image>` 사용 시 src만 지정, 자동 srcset

### 배경 (raster, CSS background)
```
upload_assets({ fileKey, nodeIds: [...], format: "webp", scale: 1 })
upload_assets({ fileKey, nodeIds: [...], format: "webp", scale: 2 })
```
- @1x/@2x 두 개로 충분 (CSS image-set)
- CSS:
  ```css
  background-image: image-set(
    url("/images/bg.webp") 1x,
    url("/images/bg@2x.webp") 2x
  );
  ```

## SVG 최적화 절차

추출된 SVG는 **반드시** 다음을 거친다:

1. **불필요 속성 제거:** `id`, `data-name`, Figma export 주석, 빈 그룹 (`<g></g>`)
2. **viewBox 보존, width/height 제거:** CSS로 사이즈 제어
3. **단색 아이콘:** `fill="#XXX"` → `fill="currentColor"` (한 색만 있을 때)
4. **다색 일러스트:** 토큰과 일치하는 색은 CSS 변수로 치환 가능 시 치환
5. **path 단순화:** SVGO `--multipass`로 처리 (가능한 경우)

도구가 없으면 수동으로 1~3만이라도 적용. 4~5는 옵션.

## get_screenshot 폴백 시 옵션

`upload_assets`가 실패하면 `get_screenshot`로 폴백:

```
get_screenshot({ fileKey, nodeId, scale: 3 })  // 항상 3x로 (고화질 폴백)
```

폴백 시 manifest의 `quality: "fallback"` 마킹. 사용자에게 보고서로 알린다.

## Manifest 스키마

`_workspace/06_assets_manifest.json`:

```json
{
  "version": 2,
  "icons": [
    {
      "name": "search",
      "strategy": "lucide-react",
      "import": "import { Search } from 'lucide-react'"
    },
    {
      "name": "logo",
      "strategy": "svg-export",
      "path": "src/assets/icons/logo.svg",
      "optimized": true,
      "useCurrentColor": true,
      "figmaNodeId": "10:50"
    }
  ],
  "illustrations": [
    {
      "name": "empty-state",
      "path": "src/assets/illustrations/empty-state.svg",
      "size": "12.4KB",
      "figmaNodeId": "10:60"
    }
  ],
  "images": [
    {
      "name": "hero",
      "figmaNodeId": "10:70",
      "variants": [
        { "scale": 1, "format": "webp", "path": "public/images/hero.webp" },
        { "scale": 2, "format": "webp", "path": "public/images/hero@2x.webp" },
        { "scale": 3, "format": "webp", "path": "public/images/hero@3x.webp" },
        { "scale": 2, "format": "png", "path": "public/images/hero@2x.png", "purpose": "fallback" }
      ],
      "alt": "Dashboard hero illustration",
      "lazy": true
    }
  ],
  "backgrounds": [
    {
      "name": "pattern-dots",
      "figmaNodeId": "10:80",
      "variants": [
        { "scale": 1, "format": "webp", "path": "public/images/pattern-dots.webp" },
        { "scale": 2, "format": "webp", "path": "public/images/pattern-dots@2x.webp" }
      ],
      "usage": "css-background"
    }
  ],
  "fonts": [
    { "family": "Inter", "source": "google-fonts", "loaded": "next/font" }
  ]
}
```

## 화질 검증 체크리스트

export 후 반드시 확인:

- [ ] 아이콘 SVG는 `currentColor` 사용 (단색일 때)
- [ ] 일러스트 SVG는 viewBox 있고 width/height 없음
- [ ] 사진은 @1x/@2x/@3x 3종 모두 존재
- [ ] WebP 폴백 PNG가 사진별 1개 존재
- [ ] manifest와 실제 파일 경로 일치
- [ ] visual-verifier가 retina 디스플레이에서 흐림 없음 보고

## 병렬 export

여러 에셋은 항상 병렬 호출:
```
Promise.all([
  upload_assets({ ..., format: "webp", scale: 1 }),
  upload_assets({ ..., format: "webp", scale: 2 }),
  upload_assets({ ..., format: "webp", scale: 3 }),
])
```
동시성 한도는 figma-node-fetcher와 동일 (5건).

## 에러 처리

| 에러 | 처리 |
|---|---|
| `upload_assets` 미지원 format | `get_screenshot(scale=3)` 폴백, manifest에 `"quality": "fallback"` 명시 |
| SVG 추출 결과가 비트맵 포함 | 노드가 잘못 분류됨 — 일러스트가 아니라 사진. 재분류 후 webp 추출 |
| 매우 큰 SVG (>200KB) | 비트맵 변환(WebP @2x) 권장, 사용자 확인 |
| 폰트 라이선스 불명 | 사용자에게 확인 요청, 시스템 폰트 폴백 + 보고서 명시 |

## 재호출 지침

이전 manifest 항목은 재export 금지. 신규 항목만 처리.
"고화질 재추출" 명시 시: 모든 raster 항목을 @3x까지 재추출.
