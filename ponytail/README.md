# ponytail

> 정리일: 2026-11-08 · 상태: ✅ 연동·테스트 완료 · **전역 설치됨 (user scope, 기본 레벨 off)** · 공식: https://github.com/DietrichGebert/ponytail · https://ponytail.dev

## 한 줄 요약

AI 코딩 에이전트에게 **"가장 게으른 시니어 개발자"** 사고방식을 주입하는 프롬프트(Skill) + Claude Code 플러그인.
"필요한 것만, 이미 있는 것·표준 라이브러리부터" 쓰게 해서 코드량·토큰·비용을 줄이되, 검증·에러 처리·보안·접근성은 절대 줄이지 않는다는 규칙이다.
MIT 라이선스, 무료. v5.3.3 (2026-10-29), GitHub ★ 약 18.6만.

## Claude와 같이 쓰면 좋은 점

- **과잉 설계 억제**: 매 세션 시작 시 규칙이 자동 주입돼서, Claude가 라이브러리 추가·래퍼·"나중을 위한" 추상화를 덜 만든다.
  → 실제 테스트에서도 같은 질문에 직접 파서를 짜는 대신 stdlib `urllib.parse.parse_qs`를 골랐다 (아래 테스트 기록).
- **답변 끝에 "건너뛴 것 / 확인 안 한 것 / 위험"을 1~2줄로 요약** → 사람이 검토할 지점이 명확해진다.
- **`/ponytail-review`**: diff만이 아니라 변경이 닿는 코드까지 읽고 버그·보안·부하·테스트 누락을 "무엇인지 / 문제 / 고칠 방법 / 안 고치면" 형식으로 보고한다.
- **`/ponytail-audit`**: 저장소 전체를 같은 기준으로 점검하고 "무엇부터 고칠지" 순위를 매긴다.
- **`shortcut:` 주석 + `/ponytail-debt`**: 일부러 미룬 지름길을 코드에 표시하고, 나중에 목록(장부)으로 모아 준다.
- 서브에이전트(Task)에도 규칙이 주입된다 (`SubagentStart` 훅).

---

## 1. 연동 방법

- 유형: **Claude Code Plugin** (Skill 6개 + Node.js 훅 3개 + "band" UI 모듈)
- 핵심은 프롬프트 하나: [`skills/ponytail/SKILL.md`](plugin/skills/ponytail/SKILL.md) (52줄). 나머지는 이 프롬프트를 여러 에이전트에 넣어 주는 어댑터.
- 설치(테스트용, 이 폴더):
  ```bash
  git clone --depth 1 https://github.com/DietrichGebert/ponytail.git ponytail/plugin
  claude -p "..." --plugin-dir ./plugin < /dev/null
  ```
- 전역 설치 (Claude Code 안에서 **두 번에 나눠** 입력해야 함):
  ```
  /plugin marketplace add DietrichGebert/ponytail
  ```
  ```
  /plugin install ponytail@ponytail
  ```
  삭제: `/plugin uninstall ponytail@ponytail` → `/plugin marketplace remove ponytail` (미검증)
- 인증: 없음 (API 키 불필요)
- 요구사항: `node`가 비대화형 셸 PATH에 있어야 훅이 동작 (이 기기: Node v26.9.0 ✅)
- 다른 에이전트: Codex, Copilot CLI, Cursor, Gemini CLI, OpenCode 등 약 20종 지원 → [plugin/INSTALL.md](plugin/INSTALL.md). 훅이 없는 에이전트는 `AGENTS.md`만 프로젝트에 복사해도 규칙은 적용된다.
- 현재 이 기기 상태: **전역 설치됨** (2026-11-08, user scope, v5.3.3) + **기본 레벨 off**
  - 설치: `claude plugin marketplace add DietrichGebert/ponytail` → `claude plugin install ponytail@ponytail --scope user`
  - 기본 off: `~/.config/ponytail/config.json` = `{"defaultMode": "off"}` (`/ponytail default off`와 같음)
  - 사용법: 코드 작업 세션에서 `/ponytail`로 켜기, 리뷰는 언제든 `/ponytail-review`
  - 삭제: `claude plugin uninstall ponytail@ponytail` → `claude plugin marketplace remove ponytail` → `rm -rf ~/.config/ponytail`
  - 왜 off인가: 항상 켜두면 문서·조사 세션에도 규칙이 적용되고, ECC 같은 다른 워크플로 플러그인과 지시가 충돌할 수 있어서

### 내부 동작 (코드 확인함)

| 훅 | 하는 일 |
|---|---|
| `SessionStart` (`ponytail-activate.js`) | 현재 레벨의 규칙(약 3.4KB ≈ 850토큰)을 컨텍스트에 주입 + 코드베이스 맵(함수·클래스 목록, 최대 2000자) 첨부 + statusLine이 없으면 "설정해 줄까요?" 제안 문구 추가 |
| `UserPromptSubmit` (`ponytail-mode-tracker.js`) | `/ponytail lite\|full\|ultra\|off` 입력을 감지해 레벨 전환 |
| `SubagentStart` (`ponytail-subagent.js`) | 서브에이전트에도 규칙 주입 |
| band 모듈 (`hooks/band/register.tsx`) | 프롬프트 위에 비용·토큰·시간·재사용률·테스트 결과 표시. **Claude Code 2.1.290 이상 필요**, VS Code 채팅 패널·`claude -p`에서는 안 보임 (이 기기 2.1.139 → 미동작) |

## 2. 주요 기능 / 도구

| 용도 | 명령 |
|---|---|
| 강도 설정 / 끄기 | `/ponytail [lite \| full \| ultra \| off]` (기본 full) |
| 기본 레벨 영구 저장 | `/ponytail default <lite\|full\|ultra\|off>` → `~/.config/ponytail/config.json` |
| 변경분 리뷰 | `/ponytail-review` (`uncommitted`, `staged`, `branch`, PR 링크로 범위 지정) |
| 저장소 전체 점검 | `/ponytail-audit` |
| 미룬 지름길 모으기 | `/ponytail-debt` |
| 벤치마크 수치 보기 | `/ponytail-gain` |
| 도움말 | `/ponytail-help` |
| band 숨기기·보이기 / 과거 세션 가져오기 | `/ponytail band [on\|off]`, `/ponytail band import` |

- 레벨: **lite** = 요청대로 만들고 더 작은 대안만 한 줄 제안 / **full** = 규칙 그대로 / **ultra** = 요청 자체에도 "정말 필요한가" 반박
- 끄기: "stop ponytail" 또는 "normal mode"라고 말하거나 `/ponytail off`
- `--plugin-dir`로 로드하면 스킬 이름은 `ponytail:ponytail-review`처럼 네임스페이스가 붙는다.

### 규칙 요약 ("사다리" — 처음 성립하는 칸에서 멈춤)

1. 정말 필요한가? → 2. 이미 코드베이스에 있나? → 3. 표준 라이브러리/플랫폼 기능? → 4. 설치된 의존성? → 5. 한 줄로 되나? → 6. 그래야 최소 코드
- 분기·루프·파서·돈·보안 로직이면 작은 테스트 1개를 남긴다.
- **절대 줄이지 않는 것**: 신뢰 경계 검증, 데이터 손실 방지 에러 처리, 보안, 접근성, 사용자가 요청한 것.

## 3. 사용 예시

```
/ponytail ultra
날짜 선택기 붙여줘            ← 라이브러리 대신 기존 Input + type="date" 같은 답을 유도
/ponytail-review staged
/ponytail-audit
/ponytail off
```

## 4. 요금 · 한도 (2026-11-08 기준)

| 플랜 | 가격 | 핵심 |
|---|---|---|
| 오픈소스 (MIT) | 무료 | 계정·키 불필요. 비용은 Claude 사용량뿐 |

- README 상단의 "Something's coming, join the waitlist"(ponytail.dev/soon)는 별도 제품 예고로 보이며, 내용은 **미확인**.
- README 벤치마크(-53% 코드, -26% 비용, -45% 토큰, 위험 로직 테스트 98% vs 68%)는 **제작자 자체 측정**이다 (Claude Code, Opus 5.5, 39개 작업 × 5회). 독립 검증은 미확인.

## 5. 테스트 기록

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 훅 단독 실행 (임시 `CLAUDE_CONFIG_DIR`로 격리) | `echo '{"source":"startup"}' \| CLAUDE_PLUGIN_ROOT=... node hooks/ponytail-activate.js` | ✅ 규칙 3,440자 출력, statusLine 설정 제안 포함. `/ponytail ultra` 입력 시 모드 파일이 `ultra`로 바뀜 |
| 2026-11-08 | 스킬 로드 확인 | `claude -p "ponytail 스킬 이름 나열" --plugin-dir ./plugin` | ✅ `ponytail:ponytail`, `-review`, `-audit`, `-debt`, `-gain`, `-help` 6개 확인 |
| 2026-11-08 | A/B 비교: "쿼리스트링 → dict 파서(중복 키는 리스트)" | 같은 프롬프트를 플러그인 없이 / `--plugin-dir`로 | ✅ 없이: 직접 `split('&')` 파서 14줄 / ponytail: stdlib `parse_qs` 사용 5줄 + 마지막에 "Skipped: ..." 위험 요약. 출력 토큰은 비슷함(180 vs 185), 비용 $0.075 vs $0.079 (규칙 주입분) |
| 2026-11-08 | 전역 설치 후 기본 off 확인 | 새 `claude -p` 세션에 "PONYTAIL MODE ACTIVE가 컨텍스트에 있나?" 질문 | ✅ NO (규칙 미주입), 스킬 6개는 보임 |
| 2026-11-08 | `/ponytail-review`: f-string SQL 인젝션을 심은 diff | `claude -p "/ponytail:ponytail-review uncommitted" --plugin-dir ... --allowedTools "Bash(git diff:*)" ...` | ✅ Must fix로 SQL 인젝션 지적, 같은 파일의 `get_user`처럼 `?` 플레이스홀더로 고치라고 제시, "Verdict: fix 1 first" + "Not checked: 테스트 없음" |

- 질문 1개로 한 A/B라 수치 비교로서는 의미가 작다. 행동 차이(stdlib 우선, 끝맺음 요약)를 확인한 정도로 본다.
- 테스트 중 훅이 `~/.claude/`에 아래 파일을 만들었고, 테스트 후 **모두 삭제**했다:
  `.ponytail-active`, `ponytail-modes/<해시>`, `.ponytail-statusline-nudged`, `ponytail-statusline.sh`

## 6. 주의사항 · 팁

- **전역 디렉터리에 파일을 쓴다** (`settings.json`은 직접 건드리지 않음):
  - `~/.claude/.ponytail-active`, `~/.claude/ponytail-modes/` (현재 레벨), `~/.claude/ponytail-statusline.sh` (상태줄 스크립트 사본), `~/.claude/.ponytail-statusline-nudged`
  - band 사용 시 `~/.config/ponytail/band/<프로젝트>.csv`, `board.html`: 턴마다 비용·토큰·**작업 이름** 기록 (`ledger` 옵션, 기본 켜짐)
- **statusLine 제안**: statusLine이 없으면 첫 세션에 Claude가 "settings.json에 statusLine을 추가할까요?"라고 먼저 제안하도록 지시가 주입된다. 1회만 제안함. 원치 않으면 거절하면 된다.
- **webhook 옵션** (`webhookUrl`, 기본 비어 있음): 설정하면 매 턴 작업 행(작업 이름 포함)이 외부로 전송된다. 기본값이면 외부로 나가는 데이터는 없다 (코드 확인: 외부 `fetch`는 이 webhook 한 곳뿐).
- `/ponytail band import`는 과거 Claude Code 세션 기록 전체를 읽어 board에 넣는다 (로컬 처리).
- 컨텍스트 비용: 세션마다 약 850토큰 + 코드베이스 맵 최대 2000자. 맵은 `PONYTAIL_MAP=0`으로 끌 수 있다.
- 이 폴더의 CLAUDE.md처럼 **문서·조사 작업이 많은 프로젝트**에서는 효과가 작다. 코드 작성 프로젝트에서 의미가 있다.
- 제작자 경고: `DietrichGebert/ponytail`(GitHub) 또는 `@dietrichgebert/ponytail`(npm)에서만 설치할 것. `.exe`/`.dll`을 포함한 사본은 가짜. 인기 저장소라 사칭본에 주의.
- 이 기기 Claude Code 2.1.139 → band UI는 업데이트(2.1.290 이상) 후에만 보인다.

## 참고 링크

- GitHub: https://github.com/DietrichGebert/ponytail
- 설치 안내: https://github.com/DietrichGebert/ponytail/blob/main/INSTALL.md
- 핵심 프롬프트: https://github.com/DietrichGebert/ponytail/blob/main/skills/ponytail/SKILL.md
- 벤치마크 방법·결과: https://github.com/DietrichGebert/ponytail/tree/main/benchmarks
- 릴리스 v5.3.3: https://github.com/DietrichGebert/ponytail/releases/tag/v5.3.3
- 홈페이지: https://ponytail.dev
