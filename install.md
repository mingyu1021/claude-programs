# 설치 가이드 — 어디에 무엇을 깔까

> 기준일: 2026-11-08 · 대상 도구 7개 (Eraser.io, draw.io, ponytail, ECC, impeccable, security-audit-skill, HyperFrames)
> 각 도구의 자세한 내용은 [INDEX.md](INDEX.md) → 도구별 README 참고.

## 한눈에 보기

| 도구 | 분류 | 현재 이 기기 | 권장 | 세션당 상시 비용 |
|---|---|---|---|---|
| [draw.io](draw.io/README.md) | 🌐 전역 | ✅ 전역 설치됨 | 유지 | 약 190 tok |
| [Eraser.io](Eraser.io/README.md) | 🌐 전역 | ✅ 전역 등록됨 | 유지 (무료 한도 주의) | MCP 도구 설명분 |
| [ponytail](ponytail/README.md) | 🌐 전역 (기본 off) | ✅ 전역 설치됨, off | 유지, 코드 작업 때만 `/ponytail` | off 상태 ≈ 0 (훅은 실행됨) |
| [ECC](ECC/README.md) | 🧰 코드 프로젝트 (필요할 때만) | 전역 삭제됨 (2026-11-08) | 프로젝트 단위, 필요할 때만 | 설치 시 **약 9~13k tok** |
| [impeccable](impeccable/README.md) | 🎨 프론트엔드 프로젝트 | 미설치 | 프로젝트 단위 | — |
| [security-audit-skill](security-audit-skill/README.md) | 🌐 전역 (Skill) | ✅ 전역 설치됨 | 유지 — 질문할 때만 로드됨 | 스킬 설명 한 줄분 |
| [HyperFrames](HyperFrames/README.md) | 🎬 영상 제작 프로젝트 | 미설치 (FFmpeg만 설치됨) | 프로젝트 단위 | — |

> ⚠️ **VS Code 확장에서는 `/plugin` 슬래시 명령이 안 된다.** 아래 명령은 전부 터미널(또는 Claude에게 Bash로 실행 요청)용 `claude plugin ...` CLI다.
> 설치·삭제 후에는 **새 세션을 열어야** 반영된다.

### 범위(scope) 고르기

| scope | 저장 위치 | 언제 |
|---|---|---|
| `user` | `~/.claude/...` | 모든 프로젝트에서 항상 쓸 것 |
| `project` | 프로젝트 `.claude/settings.json` (git에 올라감) | 팀원과 같이 쓸 것 |
| `local` | 프로젝트 `.claude/settings.local.json` (git 제외) | 이 프로젝트에서 나만 쓸 것 |

---

## 🌐 1. 전역 — 모든 프로젝트에서

가볍고(상시 비용 작음), 어떤 프로젝트에서든 쓸 일이 있는 것만.

### draw.io — 다이어그램 (✅ 설치됨)

```bash
claude plugin marketplace add jgraph/drawio-mcp
claude plugin install drawio@drawio
# 삭제
claude plugin uninstall drawio@drawio && claude plugin marketplace remove drawio
```
- 사용: `/drawio:drawio 이 프로젝트 구조 그려줘` → 레포에 `.drawio` 파일
- 같이 깔면 좋은 것: VS Code 확장 **Draw.io Integration** (`hediet.vscode-drawio`), PNG/SVG 내보내기용 `brew install --cask drawio`
- 다이어그램 내용이 `mcp.draw.io`로 전송됨 → 민감한 설계는 `/mcp`에서 번들 서버 끄기

### Eraser.io — 클라우드 다이어그램 (✅ 등록됨)

```bash
claude mcp add --transport http --scope user eraser https://app.eraser.io/api/mcp
# 처음이면 대화형 세션에서 /mcp → eraser 로그인
# 삭제
claude mcp remove eraser --scope user
```
- 무료 플랜: **파일 3개(현재 꽉 참), AI 크레딧 3개** → 새 다이어그램은 기존 파일에 추가
- draw.io와 역할 분담: 레포에 남길 설계도 = draw.io / 팀 공유·빠른 AI 초안 = Eraser

### ponytail — 과잉 설계 억제 (✅ 설치됨, 기본 off)

```bash
claude plugin marketplace add DietrichGebert/ponytail
claude plugin install ponytail@ponytail --scope user
mkdir -p ~/.config/ponytail && echo '{"defaultMode": "off"}' > ~/.config/ponytail/config.json   # 기본 off
# 삭제
claude plugin uninstall ponytail@ponytail && claude plugin marketplace remove ponytail && rm -rf ~/.config/ponytail
```
- 코드 작성 세션에서만 `/ponytail` (또는 `lite`/`ultra`)로 켜기, 리뷰는 언제든 `/ponytail-review`
- off여도 훅 3개(Node)는 매 세션 실행된다. ECC와 같이 켜면 지시가 충돌할 수 있음

---

## 🧰 2. 코드 프로젝트 (백엔드·풀스택 공통)

### ECC — 계획·TDD·리뷰 워크플로 (전역 삭제됨)

2026-11-08에 전역 minimal 설치를 해봤다가 **삭제함**. **전역 설치 금지**, 큰 코드 프로젝트에서 필요할 때만 프로젝트 단위로. 이유:
- 모든 세션(문서·조사 포함)에 **+약 13k 토큰**
- npx 전역 설치본은 `/ecc:` 접두어가 없어 내장 `/code-review`, `/plan`과 **이름이 겹침**
- `~/.claude` 루트에 `AGENTS.md`, `README.md`, `plugin.json` 등 파일을 같이 뿌리고, `settings.json`을 몰래 수정함

```bash
# 프로젝트에서만 플러그인으로 (접두어 /ecc: 로 구분됨)
cd <프로젝트>
claude plugin marketplace add https://github.com/affaan-m/ECC
claude plugin install ecc@ecc --scope local
# 삭제
claude plugin uninstall ecc@ecc --scope local && claude plugin marketplace remove ecc

# 한 번만 써보기 (설치 없이)
claude --plugin-dir /Users/mingyupark/2026-2/Claude_programs/ECC/plugin
```
- 훅 강도 조절: `ECC_HOOK_PROFILE=minimal|standard|strict`, 끄기 `ECC_HOOKS_ENABLED=false`
- 설치 방법은 **하나만** (npx 설치 + 플러그인 중복 시 스킬·훅이 두 번 뜸)
- ⚠️ 설치 프로그램이 `settings.json`에 `"includeCoAuthoredBy": false`를 몰래 넣은 적 있음 → 설치 후 확인

### ponytail 켜기
전역에 이미 있음 → 코드 작업 세션에서 `/ponytail`.

---

## 🎨 3. 프론트엔드 프로젝트

### impeccable — 디자인 명령 + AI slop 탐지 (미설치)

```bash
cd <프론트엔드 프로젝트>
# A. 플러그인 (Skill + 훅 + 에이전트)
claude plugin marketplace add pbakaus/impeccable
claude plugin install impeccable@impeccable --scope local      # 팀 공유면 --scope project
# B. 공식 설치기 (프로젝트 범위)
npx impeccable install --providers=claude --scope=project
# C. 훅 없이 지침(Skill)만 — 가장 가벼움, 테스트 검증된 방식
npx skills add pbakaus/impeccable

# 설치 후
/impeccable init         # PRODUCT.md
/impeccable hooks on     # 자동 검사 (원할 때만)
```
- **전역 설치 비추천**: 훅이 **모든 Edit/Write마다** 실행됨
- 첫 실행 때 GitHub에서 엔진 바이너리 다운로드 (`~/.impeccable/bin/`, 체크섬 검증)
- 텔레메트리 끄기: `npx impeccable telemetry off` 또는 `DO_NOT_TRACK=1`
- ⚠️ **live 모드**는 claude를 `bypassPermissions`로 실행 → 신뢰하는 로컬 저장소에서만, 아니면 `IMPECCABLE_LIVE_COPY_AGENT=chat`
- 테스트 상태: Skill 지침만 검증됨 (critique·리디자인 ✅). 훅·엔진은 미검증

### 같이 쓰면 좋은 것
- draw.io (전역) — 화면 흐름·컴포넌트 구조도
- ECC의 프론트엔드 리뷰어(`react-reviewer`, `vue-reviewer`, `a11y-architect`) — ECC를 프로젝트에 깐 경우

---

## 🔒 4. 보안 점검

### security-audit-skill (Cloudflare) — ✅ 전역 설치됨 (2026-11-08)

Skill은 훅이 없고, 보안 질문을 할 때만 본문이 로드되므로 전역에 둬도 상시 비용이 거의 없다.

```bash
# 전역 (현재 상태: ~/.claude/skills/security-audit, 커밋 c1c8a8c, 출처 기록 ~/.agents/.skill-lock.json)
npx -y skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --global
# 프로젝트에만
cd <프로젝트> && npx -y skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --agent claude-code -y
# 삭제 (전역)
rm -rf ~/.claude/skills/security-audit
```
- 사용: `이 라우트에 IDOR 있는지 봐줘`(가벼운 모드) / `security audit this codebase, quick profile, budget 12 agents`(전체 감사)
- 전체 감사는 하위 에이전트를 많이 띄움 → **quick 프로필 + 예산 지정**부터
- 결과는 기본으로 저장소 밖 `~/security-audit-skill/<repo>/run-N`에 쌓임
- 테스트 비용: 가벼운 모드 $0.14·26초 / 전체 감사 quick $4·29분 (에이전트 12회)

---

## 🎬 5. 영상 제작 프로젝트

### HyperFrames — HTML로 MP4 만들기 (미설치, FFmpeg는 설치됨)

```bash
cd <영상 프로젝트>
claude plugin marketplace add heygen-com/hyperframes
claude plugin install hyperframes@hyperframes --scope local
# 텔레메트리 끄기 (기본 켜짐) — 프로젝트 .env 또는 셸에
export HYPERFRAMES_NO_TELEMETRY=1 DO_NOT_TRACK=1
# 삭제
claude plugin uninstall hyperframes@hyperframes && claude plugin marketplace remove hyperframes
```
- 요구사항: Node 22+, FFmpeg(`brew install ffmpeg` ✅ 설치됨), Chrome
- ⚠️ `npx hyperframes init`은 기본으로 **전역 `~/.claude/skills`에 Skill 4개를 설치**함 → 플러그인을 쓰면 `--skip-skills` 또는 `HYPERFRAMES_SKIP_SKILLS=1`로 중복 방지
- 사용: `/hyperframes 10초짜리 제품 소개 영상 만들어줘`

---

## 상태 점검 · 정리 명령

```bash
claude plugin list                    # 설치된 플러그인
claude plugin details <플러그인>       # 상시 토큰 비용
claude plugin marketplace list
claude mcp list                       # MCP 연결 상태
ls ~/.claude/skills ~/.claude/agents ~/.claude/commands
```

### 현재 전역 상태 (2026-11-08 확인)
- 플러그인: `clangd-lsp`, `drawio`, `ponytail` / MCP: `eraser`, `plugin:drawio:drawio`
- 스킬: `~/.claude/skills/security-audit` (+ Claude가 관리하는 `synced/`)
- `~/.claude/agents`, `commands`, `rules` 없음
- ECC 테스트 찌꺼기(`metrics/`, `session-data/`, `skills/learned/`, `bash-commands.log`, `cost-tracker.log`) 모두 삭제됨
