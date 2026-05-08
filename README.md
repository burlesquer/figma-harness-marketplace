# figma-harness-marketplace

Figma → 프론트엔드 코드 퍼블리싱을 자동화하는 멀티 에이전트 하네스 마켓플레이스.

## 수록 플러그인

### figma-publish-harness

Figma 디자인 파일을 프로덕션 프론트엔드 코드로 변환하는 멀티 에이전트 하네스. 15개 전문 에이전트와 6개 스킬로 토큰 추출·컴포넌트 매핑·화면 구현·검증·자동 수정 루프를 자동화한다. React/Vue/Svelte/Flutter 등 어느 스택에도 동작.

→ 상세: [`plugins/figma-publish-harness/README.md`](./plugins/figma-publish-harness/README.md)

## 설치

Claude Code에서 마켓플레이스 추가:

```
/plugin marketplace add <이 리포 URL 또는 로컬 경로>
```

이후 플러그인 설치:

```
/plugin install figma-publish-harness@figma-harness-marketplace
```

## 의존성

플러그인 사용 전 다음을 별도 설치:

- **Figma MCP 플러그인** — 디자인 추출용 (Figma 공식 MCP 서버 또는 호환 플러그인)
- **everything-claude-code (ECC) 플러그인** — design-system, dashboard-builder, frontend-patterns, accessibility, browser-qa, e2e-testing, verification-loop, gan-design/gan-build 등

## 구조

```
.
├── .claude-plugin/
│   └── marketplace.json     ← 마켓플레이스 매니페스트
├── plugins/
│   └── figma-publish-harness/
│       ├── .claude-plugin/
│       │   └── plugin.json  ← 플러그인 매니페스트
│       ├── agents/          ← 15개 에이전트
│       ├── skills/          ← 6개 스킬
│       ├── README.md
│       └── CHANGELOG.md
└── README.md
```

## 라이선스

MIT
