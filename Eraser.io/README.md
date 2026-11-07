# Eraser.io

> 정리일: 2026-11-08 · 상태: ✅ 연동·테스트 완료 (전역 등록됨) · 공식 문서: https://docs.eraser.io · 요금: https://www.eraser.io/pricing

## 한 줄 요약

텍스트(DSL)나 자연어 프롬프트로 **아키텍처·시퀀스·ERD·플로우차트·BPMN 다이어그램**을 그려주는 협업 툴.
MCP 서버를 제공해서 **Claude Code에서 바로 다이어그램을 생성·수정**할 수 있다.

## 왜 Claude와 같이 쓰나

- 코드베이스나 설계를 설명하면 Claude가 Eraser에 다이어그램을 만들어주고, 편집 가능한 링크(`fileUrl`)를 돌려준다.
- 결과물이 이미지가 아니라 **DSL 코드**라서 나중에 한 줄만 고쳐도 갱신된다.
- 이미 만들어 둔 다이어그램을 읽어와서(`get_diagram`) 수정 요청을 할 수도 있다.

---

## 1. 연동 방법 (2가지)

### A. MCP 서버 — 현재 이 기기에 설치됨 ✅

`~/.claude.json`(user scope)에 등록되어 있음:

```json
"eraser": {
  "type": "http",
  "url": "https://app.eraser.io/api/mcp"
}
```

새로 설치할 때는:

```bash
claude mcp add --transport http --scope user eraser https://app.eraser.io/api/mcp
```

- 인증: **OAuth 권장**. 대화형 세션에서 `/mcp` → eraser → 로그인. (API 토큰 방식도 있지만 토큰 발급은 유료 플랜 전용)
- 상태 확인: `claude mcp list` → `eraser: ... ✓ Connected`
- 소스: github.com/eraserlabs/eraser-io (`packages/eraser-mcp`)

### B. Agent Skills (MCP 대신)

```bash
npx skills install eraserlabs/eraser-io
```

- Claude가 Eraser DSL을 직접 작성 → API(`/render/elements`)로 이미지 렌더링.
- API 키(`ERASER_API_KEY` 환경변수 또는 프로젝트 루트 `.env`)는 선택. **키 없으면 워터마크 + 저해상도.**

| | MCP | Skills |
|---|---|---|
| 자연어 → 다이어그램 (Eraser AI가 생성) | ✅ `/render/prompt` | ❌ |
| DSL → 다이어그램 | ✅ `/render/elements` | ✅ |
| 파일/폴더/문서 관리 | ✅ | ❌ |
| 결론 | **기본은 MCP** | 가볍게 이미지만 필요할 때 |

---

## 2. 주요 MCP 도구

| 용도 | 도구 |
|---|---|
| 계정/팀 확인 | `get_me`, `list_teams`, `select_team` |
| **자연어로 다이어그램 생성 (권장)** | `create_diagram` — `text`만 넘기고 `diagramType`은 비워두면 Eraser가 알아서 고름 |
| DSL로 직접 생성/수정 | `manually_create_diagram`, `manually_update_diagram` |
| 자연어로 수정 | `update_diagram` |
| 조회 | `list_files`, `get_file`, `list_diagrams`, `get_diagram`, `search` |
| 문서(노트) | `create_document`, `get_document`, `update_document` |
| 내보내기 | `export_diagram`, `export_document` |
| 폴더/정리 | `create_folder`, `list_folders`, `update_folder` |
| AI 프리셋(팀 규칙·템플릿) | `list_presets`, `create_preset`, `update_rules` |

`create_diagram` 주요 옵션:
- 위치: `targetFileId`(기존 파일에 추가) / 생략 시 새 파일 / `isTeamFile: true`(팀 공유) / `folderId`
- 스타일: `colorMode`(pastel·bold·outline), `styleMode`(plain·shadow·watercolor), `typeface`(rough·clean·mono), `theme`(light·dark), `direction`
- `includeImage: true` → PNG URL 반환 (기본 1시간 만료, `urlSettings`로 조정)

### 사용 예시 (Claude Code에서)

```
이 프로젝트의 백엔드 구조를 Eraser 클라우드 아키텍처 다이어그램으로 그려줘. 기존 "로그인 흐름" 파일에 추가해줘.
```

---

## 3. Diagram-as-Code 문법 요약

모든 타입 공통: `노드 [속성]`, 연결 `>` `<` `<>` `-` `--` `-->`, 라벨은 `: 텍스트`, 주석 `//`, 제목 `title ...`

### 클라우드 아키텍처

```
title Web Service
VPC Subnet {
  Main Server {
    Server [icon: aws-ec2]
    Data [icon: aws-rds]
  }
}
Storage [icon: aws-s3]
Storage > Server: Cache Hit
```

### 시퀀스

```
title Checkout
Web App [icon: monitor, color: blue] > DB [icon: database, color: green]: Start transaction
activate DB
DB > Web App: OK
deactivate DB
opt [label: if complete] {
  Server > Client: Success
}
```
블록: `loop`, `alt`/`else`, `opt`, `par`/`and`, `break` · 열 순서: `order Server, Client`

### ERD

```
users {
  id string pk
  name string
}
posts {
  id string pk
  authorId string
}
users.id < posts.authorId
```
관계: `<` 1:N, `>` N:1, `-` 1:1, `<>` N:M

### 플로우차트

```
direction right
Start [shape: oval]
Check [shape: diamond]
Start > Check
Check > Done: yes
Check --> Retry: no
```

그 외: BPMN(pool/lane), Freeform(자유 배치)

---

## 4. 요금제 (2026-11-08 기준)

| 플랜 | 가격 | 핵심 |
|---|---|---|
| **Free** | $0 | **파일 3개**, AI 크레딧 3개, 버전 기록 7일, API 호출 ❌, 비공개 파일 ❌ |
| Business | $45/인/월(연간), $60(월간) | 파일 무제한, AI 100크레딧/월, Git repo 1개, API 사용량 과금 |
| Business Plus | $42/인/월, 최소 10석 | AI 250크레딧/월, SAML SSO |
| Enterprise | 문의 | 무제한 |

※ Starter 플랜은 신규 팀에 더 이상 제공 안 됨.

---

## 5. 내 계정 상태 & 주의사항

- 계정: 개인 계정 1개 / 팀 1개 — 무료 플랜으로 추정
- 현재 파일 3개 → **무료 한도(3개) 꽉 참**
  - 바이럴 음식 탐지 시스템 아키텍처
  - 스텔라이브 노래 아카이브 — 전체 아키텍처
  - 로그인 흐름
- ⚠️ 새 파일 생성이 막힐 수 있음 → `create_diagram` 호출 시 **`targetFileId`로 기존 파일에 다이어그램 추가**하거나, 안 쓰는 파일을 정리할 것.
- ⚠️ 세 파일 모두 `linkAccess: anyone-with-link-can-edit` → 링크만 있으면 누구나 편집 가능. 민감한 설계는 공유 설정 확인.
- ⚠️ AI 크레딧이 무료 플랜에선 3개뿐 → 자연어 생성(`create_diagram`/`update_diagram`)을 아끼고 싶으면 Claude가 DSL을 직접 쓰는 `manually_create_diagram`/`manually_update_diagram` 사용. (크레딧 차감 여부는 직접 확인 필요)

## 테스트 기록

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 연결·계정 확인 | `get_me`, `list_files` (메인 세션) | ✅ 팀 1개, 파일 3개 |
| 2026-11-08 | 헤드리스 하위 세션에서 MCP 호출 | `claude -p "... get_me ..." --mcp-config mcp.json --strict-mcp-config --allowedTools "mcp__eraser__get_me" < /dev/null` | ✅ displayName·팀 수 반환 (OAuth 토큰 재사용됨) |
| — | 다이어그램 생성 | 미실행 — 무료 한도(파일 3개) 때문에 보류 | — |

전역 등록 삭제: `claude mcp remove eraser --scope user`

## 6. 팁

- 다이어그램 타입은 명시하지 않는 게 결과가 더 좋음 (Eraser가 프롬프트 보고 고름).
- 생성 후엔 이미지보다 **`fileUrl` 링크로 편집기에서 열어보기**가 낫다.
- 팀 공통 스타일/규칙은 Preset(`update_rules`)에 넣어두면 매번 프롬프트에 안 적어도 됨.

## 참고 링크

- AI 에이전트 연동: https://docs.eraser.io/using-ai-agent-integrations
- Diagram as code: https://docs.eraser.io/diagram-as-code
- 시퀀스 문법: https://docs.eraser.io/docs/syntax-2
- ERD 문법: https://docs.eraser.io/erd-syntax
- 플로우차트 문법: https://docs.eraser.io/docs/syntax-3
- 스타일링: https://docs.eraser.io/docs/styling
- 요금: https://www.eraser.io/pricing
- MCP 카탈로그: https://www.eraser.io/mcp-catalog
