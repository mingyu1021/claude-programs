# draw.io (diagrams.net)

> 정리일: 2026-11-08 · 상태: ✅ 플러그인 전역 설치·테스트 완료 (npm 서버는 미테스트) · 공식: https://www.drawio.com · 연동 저장소: https://github.com/jgraph/drawio-mcp

## 한 줄 요약

무료 오픈소스 다이어그램 에디터. 공식 저장소 `jgraph/drawio-mcp`가 **MCP 서버(호스팅·npm)와 Claude Code 플러그인(Skill)** 을 함께 제공해서,
Claude가 **Mermaid나 draw.io XML로 다이어그램을 만들고 → `.drawio` 파일 저장 또는 편집기 링크**로 넘겨준다.

## Claude와 같이 쓰면 좋은 점

- **계정·API 키·요금이 전혀 필요 없다.** (Eraser.io처럼 파일 3개·AI 크레딧 한도가 없음)
- 결과물이 **프로젝트 안의 `.drawio` 파일**(XML 텍스트)이라 git으로 버전 관리·diff가 되고, Claude가 나중에 읽어서 수정할 수 있다.
- Claude가 쓴 Mermaid를 draw.io가 **편집 가능한 도형**으로 바꿔줌 → 사람이 이어서 손으로 다듬기 좋다.
- AWS/Azure/GCP/Cisco/K8s 등 **10,000개+ 공식 아이콘**을 검색해서 쓸 수 있다.
- PNG/SVG/PDF로 내보낼 때 **다이어그램 XML을 파일 안에 넣어서**(`-e`) 이미지 파일을 다시 열면 편집 가능.

### Eraser.io와 비교

| | draw.io | [Eraser.io](../Eraser.io/README.md) |
|---|---|---|
| 비용·한도 | 무료, 무제한 | Free: 파일 3개, AI 크레딧 3개 |
| 인증 | 없음 | OAuth |
| 다이어그램 작성 주체 | **Claude**가 Mermaid/XML 작성 | Eraser AI(`create_diagram`) 또는 Claude가 DSL 작성 |
| 결과물 위치 | 로컬 `.drawio` 파일 / 링크 | Eraser 클라우드 파일 |
| 손 편집 | 정밀 (전통적 에디터) | 코드 + 캔버스 |
| 추천 | **레포에 남길 문서·설계도** | 팀 공유·빠른 AI 초안 |

---

## 1. 연동 방법 (3가지)

| 방식 | 무엇 | Claude Code에서 결과 | 필요 조건 |
|---|---|---|---|
| **A. 플러그인 (공식 추천)** | `drawio` Skill + 호스팅 MCP 번들 | `.drawio` 파일 / PNG·SVG·PDF / URL | Node(내장) · 내보내기는 draw.io Desktop |
| B. 호스팅 MCP App 서버 | `https://mcp.draw.io/mcp` (HTTP) | app.diagrams.net **링크** 반환 (Claude Code는 인라인 표시 안 됨) | 없음 |
| C. MCP Tool 서버 (npm) | `npx -y @drawio/mcp` (stdio, v1.11.0) | 브라우저로 열고 URL 반환, 로컬 `.drawio` 페이지 읽기/쓰기 | Node |

### A. 플러그인 — 권장

```bash
# 전역 설치 — 터미널/Bash에서 (VS Code 확장에서는 /plugin 슬래시 명령이 안 됨 → 이 CLI 사용)  ✅ 2026-11-08 설치함
claude plugin marketplace add jgraph/drawio-mcp
claude plugin install drawio@drawio
# 대화형 터미널 Claude Code에서는 /plugin marketplace add ... , /plugin install ... 도 가능
# 업데이트
claude plugin marketplace update drawio && claude plugin update drawio@drawio
# 삭제
claude plugin uninstall drawio@drawio && claude plugin marketplace remove drawio

# 테스트용 (이 폴더에 받아둔 것, 전역 설정 안 건드림)
claude --plugin-dir ./plugin
```

- 이 폴더의 `plugin/` = 공식 저장소 `plugins/claude-code` (커밋 `639186a`, 2026-11-07, 플러그인 v1.2.6)
- 구성: `skills/drawio/SKILL.md`(작성 규칙), `scripts/find-shapes.js`(아이콘 검색, 네트워크 사용), `.mcp.json`(호스팅 서버 B 번들). **hooks 없음.**
- 사용: `/drawio:drawio <요청>` — 앞에 `png` / `svg` / `pdf` / `url` 을 붙이면 그 형식으로 출력. 없으면 `.drawio` 파일.
- 완전 로컬로 쓰려면 `/mcp`에서 번들된 호스팅 서버를 끄면 된다.

### B. 호스팅 MCP 서버

```bash
claude mcp add --transport http --scope user drawio https://mcp.draw.io/mcp   # 전역 등록 (미실행)
claude mcp remove drawio --scope user                                          # 삭제
```
테스트용 설정: [mcp.hosted.json](mcp.hosted.json)

### C. npm Tool 서버

```bash
claude mcp add --scope user drawio -- npx -y @drawio/mcp   # 전역 등록 (미실행)
```
테스트용 설정: [mcp.json](mcp.json) (`drawio` = npm 서버 + `DRAWIO_BROWSER=off`, `drawio-app` = 호스팅 서버)

### draw.io Desktop (선택 — Mermaid 변환·자동 레이아웃·이미지 내보내기용)

```bash
brew install --cask drawio          # 설치 (미실행)
brew uninstall --cask drawio        # 삭제
# CLI는 앱 번들 안에 있음 (밖으로 심볼릭 링크하면 "Unable to find helper app"으로 죽음)
/Applications/draw.io.app/Contents/MacOS/draw.io -x -f png -e -b 10 -o a.drawio.png a.drawio
```

- 인증: **없음** (모든 방식)
- 현재 이 기기 상태: **플러그인 `drawio@drawio` v1.2.6 전역(user scope) 설치됨** (2026-11-08)
  - 번들 MCP `plugin:drawio:drawio` (https://mcp.draw.io/mcp) ✓ Connected
  - 상시 토큰 비용 약 187 tok/세션, Skill 호출 시 약 7.2k tok (`claude plugin details drawio@drawio`)
  - draw.io Desktop 미설치, VS Code draw.io 확장 미설치

## 2. 주요 기능 / 도구

**호스팅 서버 (B)** — 플러그인 설치 시 같이 들어옴

| 용도 | 도구 |
|---|---|
| 다이어그램 생성 → 링크 | `create_diagram` (`xml` 또는 `mermaid` 중 하나, `postLayout: "elk"`, `direction`) |
| 작성 규칙 참조 | `get_drawio_reference` (`format`: xml / mermaid) |
| 아이콘 검색 | `search_shapes` |

**npm 서버 (C)**

| 용도 | 도구 |
|---|---|
| 편집기에서 열기 | `open_drawio_xml`, `open_drawio_mermaid`, `open_drawio_csv` (`lightbox`, `dark`, `postLayout`, `routing: "libavoid"`) |
| 참조·아이콘 | `get_drawio_reference`, `search_shapes` (`query`, `limit` ≤ 50) |
| **로컬 `.drawio` 파일 페이지 편집** | `list_pages`, `get_page`, `set_page` |

**draw.io Desktop CLI** (플러그인이 사용)

| 용도 | 명령 |
|---|---|
| Mermaid → .drawio | `draw.io -x -f xml -o d.drawio d.mmd` |
| 자동 레이아웃 | `--layout verticalFlow` / `horizontalFlow` / `verticalTree` / `organic` / `libavoid` |
| 내보내기 (편집 가능 이미지) | `draw.io -x -f png -e -b 10 -o d.drawio.png d.drawio` |

## 3. 사용 예시

```
/drawio:drawio 이 프로젝트의 백엔드 구조를 AWS 아이콘으로 그려줘
/drawio:drawio svg 회원가입 시퀀스 다이어그램
/drawio:drawio url users/posts/comments 테이블 ERD
docs/architecture.drawio 열어서 Redis 캐시 노드 추가해줘
```

## 4. 요금 · 한도 (2026-11-08 기준)

| 플랜 | 가격 | 핵심 |
|---|---|---|
| 웹(app.diagrams.net)·Desktop | 무료 | 다이어그램 무제한, 워터마크 없음, 계정 불필요 |
| MCP 서버·플러그인 | 무료 | Apache-2.0, 인증 없음 |
| Confluence/Jira 앱 | 유료 (Atlassian Marketplace) | 정확한 가격 미확인 |

## 5. 테스트 기록

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | npm 패키지 정보 | `npm view @drawio/mcp version license` | ✅ 1.11.0, Apache-2.0 |
| 2026-11-08 | 호스팅 서버 연결 (읽기 전용) | `claude -p "... get_drawio_reference(format=xml) ..." --mcp-config mcp.hosted.json --strict-mcp-config --allowedTools "mcp__drawio-app__get_drawio_reference" < /dev/null` | ✅ `# draw.io XML Reference`, 약 24K자 |
| 2026-11-08 | 호스팅 서버 다이어그램 생성 | 같은 방식, `mcp__drawio-app__create_diagram`에 Mermaid 로그인 플로우 | ✅ app.diagrams.net `#create=` 링크 반환. 링크 payload 디코딩 → 보낸 Mermaid와 동일 확인 ([결과](test-output/hosted-create-result.txt)) |
| 2026-11-08 | `.drawio` 파일 형식 | Claude가 SKILL.md 규칙대로 XML 직접 작성 → Python으로 파싱·셀 참조 검증, `#R` URL 생성·역변환 | ✅ 셀 10·엣지 4, URL 왕복 일치 ([login-flow.drawio](test-output/login-flow.drawio), [링크](test-output/login-flow.url.txt)) — 플러그인 경유는 아님 |
| 2026-11-08 | npm 서버 (`npx -y @drawio/mcp`) | `claude -p ... --mcp-config mcp.json` | ⏸ 미실행 — 자동 모드 분류기가 "외부 코드 실행"으로 차단. 사용자 승인 필요 |
| 2026-11-08 | 플러그인 전역 설치 | `claude plugin marketplace add jgraph/drawio-mcp` → `claude plugin install drawio@drawio` | ✅ v1.2.6, user scope (사용자 요청으로 메인 세션이 설치) |
| 2026-11-08 | 플러그인 Skill (전역 설치본) | 빈 폴더에서 `claude -p "/drawio:drawio 사용자 → 웹서버 → DB ... arch.drawio" --allowedTools "Read,Write,Bash(ls:*)" < /dev/null` | ✅ `arch.drawio` 생성 (셀 3개 + 화살표, 유효한 mxGraphModel XML) |
| 2026-11-08 | 실제 다이어그램 보기 | CLAUDE.md 워크플로를 XML로 작성 → `#R` 링크 생성 → `open`으로 브라우저 열기 | ✅ 셀 29개, URL 2.9K자 ([파일](test-output/claude-programs-workflow.drawio), [링크](test-output/claude-programs-workflow.url.txt)) |
| — | PNG/SVG 내보내기 | draw.io Desktop 필요 | — 미설치라 보류 |

## 6. 주의사항 · 팁

- **개인정보 흐름**
  - 호스팅 서버(B)·플러그인 기본값: 다이어그램 내용이 `mcp.draw.io`로 전송됨 (공식 설명상 저장 안 함). 민감한 설계는 `/mcp`에서 끄고 로컬(.drawio 파일·npm 서버)로.
  - 링크 방식: 다이어그램이 URL의 `#` 뒤(fragment)에 들어가서 **브라우저가 서버로 보내지 않음**. 단, 링크를 공유하면 내용이 그대로 공유됨.
  - `search_shapes`/`find-shapes.js`는 검색어를 `icons.diagrams.net`으로 보냄 (`DRAWIO_ICON_SERVICE_URL=off`로 차단).
- Claude Code는 다이어그램을 **채팅 안에 그려주지 못함** → 링크나 파일로 받는다. VS Code에서는 확장 **Draw.io Integration (`hediet.vscode-drawio`)** 을 깔면 `.drawio`/`.drawio.svg`/`.drawio.png`를 에디터 탭에서 바로 편집할 수 있다 (플러그인도 이 확장이 있으면 `code <file>`로 연다).
- Desktop 없이도 **XML 작성 → `.drawio` 파일 / URL** 은 된다. Mermaid 변환·ELK 레이아웃·이미지 내보내기만 Desktop 필요.
- Mermaid → PNG 직접 내보내기(`-e`)는 현재 Desktop에서 깨짐 → **반드시 `.drawio`로 먼저 변환 후** 내보내기.
- 내보낸 이미지는 `이름.drawio.png`처럼 이중 확장자로 — "XML이 들어있음" 표시.
- npm 서버의 `open_drawio_*`는 기본으로 브라우저를 띄운다. 헤드리스/자동화에서는 `DRAWIO_BROWSER=off`.
- 아주 큰 다이어그램은 URL 길이 제한에 걸릴 수 있음 → 파일로 열기.

## 참고 링크

- 연동 저장소 (README·도구 목록): https://github.com/jgraph/drawio-mcp
- Tool 서버: https://github.com/jgraph/drawio-mcp/blob/main/mcp-tool-server/README.md
- App 서버: https://github.com/jgraph/drawio-mcp/blob/main/mcp-app-server/README.md
- Claude Code 플러그인: https://github.com/jgraph/drawio-mcp/tree/main/plugins/claude-code
- 공식 문서: https://www.drawio.com/docs/manual/generate/drawio-mcp-server
- AI 어시스턴트 개인정보 정책: https://www.drawio.com/trust/privacy-ai-assistants/
- Desktop CLI: https://github.com/jgraph/drawio/wiki/Desktop-App
- npm: https://www.npmjs.com/package/@drawio/mcp
