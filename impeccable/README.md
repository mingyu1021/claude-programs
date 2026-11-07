# impeccable

> 정리일: 2026-11-08 · 상태: 🟡 Skill(지침)만 테스트 ✅ / 엔진·hooks 미실행 · 공식: https://impeccable.style · 저장소: https://github.com/pbakaus/impeccable (Apache-2.0, ⭐ 약 105k)

## 한 줄 요약

AI 코딩 에이전트가 만드는 **"AI스러운 뻔한 UI"(AI slop)** 를 막고, 디자인 작업을 `/impeccable polish`, `critique`, `audit` 같은 **24개 디자인 명령**으로 다룰 수 있게 해주는 Claude Code 플러그인(Skill + hooks + 서브에이전트).
Paul Bakaus가 만들었고, Anthropic의 `frontend-design` 스킬에서 출발했다.

## Claude와 같이 쓰면 좋은 점

- "예쁘게 해줘" 대신 **명확한 디자인 어휘**로 지시: `bolder`(과감하게), `quieter`(차분하게), `distill`(덜어내기), `typeset`(타이포), `harden`(에러·i18n·엣지 케이스) 등.
- **리뷰 도구**: `critique`(닐슨 휴리스틱 점수 + 페르소나별 문제), `audit`(접근성·성능·반응형, P0~P3).
- **프로젝트 디자인 맥락 파일**: `init` → `PRODUCT.md`(사용자·목적), `document` → `DESIGN.md`(디자인 시스템). 이후 모든 디자인 작업이 이 파일을 기준으로 함.
- **AI slop 탐지기**(규칙 62개, LLM·API 키 불필요): Inter 폰트 남발, 보라→파랑 그라디언트, 카드 안의 카드, 제목마다 둥근 아이콘 타일 같은 패턴을 hooks로 **편집할 때마다 자동 검사**.

---

## 1. 연동 방법

- 유형: **Plugin** (Skill 1개 + hooks + 서브에이전트 4개). MCP 아님.
- 인증·API 키: 없음 · 요금: 무료(오픈소스, 유료 플랜 안내 없음)

```bash
# A. 플러그인 (Claude Code 대화형 세션 안에서) — 권장
/plugin marketplace add pbakaus/impeccable
/plugin            # → Discover → impeccable 설치. 업데이트도 /plugin 메뉴
# 삭제: /plugin 메뉴에서 uninstall, 또는
claude plugin uninstall impeccable@impeccable && claude plugin marketplace remove impeccable   # (미검증)

# B. 공식 CLI 설치기 (프로젝트 루트에서; 도구 자동 감지, project/global 선택, hook 설치 여부 질문)
npx impeccable install                      # 비대화형: --providers=claude --scope=project
npx impeccable update

# C. 스킬만 (hooks·서브에이전트 없음)
npx skills add pbakaus/impeccable

# 설치 후
/impeccable init        # PRODUCT.md 작성
/impeccable hooks on    # 자동 검사 hook 켜기

# 테스트용 (이 폴더, 전역 설정 안 건드림)
claude --plugin-dir ./plugin
```

- 이 폴더의 `plugin/` = 저장소 `plugin/` (커밋 `687bc38`, 2026-11-06, v4.8.0, 엔진 v0.1.18). 저장소 루트의 `.claude-plugin/`은 `plugin/.marketplace-root-claude-plugin/`으로 보관.
- 현재 이 기기 상태: **미설치** (전역 플러그인·스킬 없음, `~/.impeccable/` 없음). `test-project/.claude/skills/impeccable/`에 **지침 파일만** 복사(테스트용).

### 구성 요소

| 구성 | 위치 | 내용 |
|---|---|---|
| Skill | `plugin/skills/impeccable/SKILL.md` + `reference/*.md` 45개 | 명령별 플레이북, `craft-floor.md`(품질 하한·금지 목록) |
| 엔진 런처 | `skills/impeccable/scripts/impeccable` (sh) | 네이티브 바이너리 실행. **없으면 GitHub Releases에서 다운로드** → `~/.impeccable/bin/` (SHA-256을 `engine-checksums.txt`로 검증) |
| Hooks | `plugin/hooks/hooks.json` | `SessionStart`, `PostToolUse(Edit\|Write)`, `Stop` 에서 `impeccable hook` 실행 (5초/5초/30초 제한) |
| 서브에이전트 | `plugin/agents/` | asset-producer, documenter, finish-reviewer, manual-edit-applier |
| Live 모드 | `scripts/live-browser*.js` | 브라우저에서 요소 골라 변형안 반복 |

## 2. 주요 명령

`/impeccable <명령> [대상]` — 인자 없이 `/impeccable`만 치면 상황별 메뉴.

| 분류 | 명령 |
|---|---|
| 계획·설정 | `shape`(코드 전에 UX 계획), `init`(PRODUCT.md), `document`(DESIGN.md), `extract`(반복 패턴 → 디자인 시스템) |
| 평가 | `critique`(UX 점수 리뷰), `audit`(기술 품질 P0~P3) |
| 다듬기 | `polish`, `bolder`, `quieter`, `distill`, `harden`, `onboard` |
| 강화 | `animate`, `colorize`, `typeset`, `layout`, `delight`, `overdrive` |
| 수정 | `clarify`(UX 문구), `adapt`(기기·화면), `optimize`(성능) |
| 반복 | `live`(브라우저에서 고르며 변형), `generate`(변형안 n개) |
| 기타 | `pin audit`(→ `/audit` 단축), `hooks on`, `doctor`(깨진 hook·오래된 맥락 파일 점검) |

탐지기 CLI (미검증): `npx impeccable detect src/` · `detect http://localhost:5173` · `detect --json .` · `detect --list-rules` (종료코드 2 = 발견, 1 = 스캔 실패)

## 3. 사용 예시

```
/impeccable init
/impeccable critique 요금제 카드
/impeccable bolder 히어로 섹션
/impeccable harden checkout
/impeccable polish settings 페이지
```

## 4. 요금 · 한도 (2026-11-08 기준)

| 플랜 | 가격 | 핵심 |
|---|---|---|
| 오픈소스 | 무료 | Apache-2.0, API 키 불필요. 공식 요금 페이지는 없음 |
| 실사용 비용 | Claude 토큰 | 테스트에서 critique 1회 ≈ $0.22, 리디자인 1회 ≈ $0.93 (헤드리스, 모델 기본값) |

## 5. 테스트 기록

테스트 대상: 일부러 "AI스럽게" 만든 랜딩 페이지 [test-project/index.html](test-project/index.html) (Inter, 보라→파랑 그라디언트, 카드 안 카드, 이모지 아이콘 타일, 근거 없는 수치)

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 코드 검토 | hooks.json, 런처 스크립트, SKILL.md 직접 읽음 | ✅ 위 "구성 요소"·"주의사항" 참고 |
| 2026-11-08 | `critique` (읽기 전용) | `cd test-project && claude -p "/impeccable critique index.html ..." --allowedTools "Skill,Read,Glob,Grep" --output-format stream-json --verbose` | ✅ 8/20 Poor, 안티패턴 11개, 우선순위 개선안 3개 ([결과](test-output/critique.md)). 런처 없음 → 스킬의 "Launcher unavailable" 대체 절차대로 동작. 4턴, $0.22 |
| 2026-11-08 | 리디자인 (새 파일만 작성) | 같은 방식, `--allowedTools "Skill,Read,Glob,Grep,Write"`, 결과는 `after.html`로 | ✅ 다크 네이비 + 틸 단일 액센트, 이모지 → SVG 아이콘, 비대칭 그리드, UI 목업 히어로, Archivo Black + Manrope ([after.html](test-project/after.html)). 외부 리소스는 Google Fonts뿐, 스크립트 없음. 5턴, $0.93 |
| 2026-11-08 | 플러그인 전체 (hooks·엔진) | `claude --plugin-dir ./plugin` | ⏸ 미실행 — 엔진 바이너리 다운로드·실행이 필요하고, 이 세션 자동 모드가 외부 코드 실행을 막음 (draw.io 때와 동일). 사용자 승인 필요 |
| — | `npx impeccable detect` | — | ⏸ 같은 이유로 미실행 |

관찰:
- 엔진 없이도 **지침만으로 critique·리디자인은 잘 동작**한다. 다만 critique에서 Inter 폰트, "제목마다 아이콘 타일" 같은 **impeccable 고유의 slop 규칙을 명시적으로 짚지는 않았다** → 이 부분은 엔진 탐지기(62규칙)의 몫으로 보인다.
- critique 내용 중 "3열 그리드가 모바일에서 화면 밖으로 넘침"은 과장 (`1fr` 열은 줄어들 뿐 넘치지 않음). 결과는 사람이 한 번 걸러야 함.
- `--allowedTools`에 없는 Bash `ls`가 한 번 실행됨 (읽기 전용 명령이라 기본 권한으로 허용된 것으로 보임).

## 6. 주의사항 · 팁

- ⚠️ **hooks는 내 권한 거부와 무관하게 실행된다.** 설치하면 세션 시작·**모든 Edit/Write**·응답 종료 때마다 엔진이 돈다. 첫 실행에서 GitHub Releases에서 바이너리를 내려받는다 (체크섬 검증 있음). 한 번만 끄려면: `claude --settings '{"disableAllHooks": true}'`.
- ⚠️ **Live 모드의 문구 수정 적용**은 `codex` 또는 `claude` CLI를 **승인·샌드박스 끈 상태**(`--dangerously-bypass-approvals-and-sandbox` / `--permission-mode bypassPermissions`)로 실행한다. 신뢰하는 로컬 저장소에서만 쓰고, 싫으면 `IMPECCABLE_LIVE_COPY_AGENT=chat`(또는 `off`).
- ⚠️ **네트워크·텔레메트리**: 새 디자인을 시작할 때 impeccable.style에서 "시각 방향 카탈로그"를 받아오고, 어떤 방향이 보여졌고 선택됐는지를 익명으로 전송한다 (코드·파일·PRODUCT.md는 안 보낸다고 함). 끄기: `npx impeccable telemetry off` / `DO_NOT_TRACK=1` / `.impeccable/config.json`에 `"telemetry": false`. 꺼도 카탈로그 다운로드는 함.
- 디자인 작업(UI 편집)마다 hooks가 돌아서 **백엔드 위주 프로젝트에는 전역 설치 비추천** → 프론트엔드 프로젝트에만 project scope로 설치하는 게 낫다 (`npx impeccable install --scope=project`).
- 브랜드 폰트로 Inter를 꼭 써야 하면: `npx impeccable ignores add-value overused-font Inter --reason "Brand font"` (미검증).
- `/impeccable` 단독 실행은 메뉴만 보여주고 자동 실행하지 않는다.

## 참고 링크

- 저장소: https://github.com/pbakaus/impeccable
- 공식 사이트: https://impeccable.style
- 문서: https://impeccable.style/docs/
- 플러그인 폴더: https://github.com/pbakaus/impeccable/tree/main/plugin
- 엔진 릴리스: https://github.com/pbakaus/impeccable/releases
