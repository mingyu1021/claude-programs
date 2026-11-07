# ECC (affaan-m/ECC)

> 정리일: 2026-11-08 · 상태: ✅ 연동·테스트 완료 · **전역 설치했다가 삭제함** (테스트용 `ECC/plugin/`만 남음) · 공식: https://github.com/affaan-m/ECC · https://ecc.tools

## 한 줄 요약

Claude Code(및 Codex·Cursor·OpenCode 등)용 **"에이전트 하네스 운영 체계" 플러그인**. 서브에이전트 71개, Skill 314개, 커맨드 98개, hooks, rules를 한 번에 설치해서
`계획 → 테스트 → 구현 → 리뷰 → 검증 → 기억 → 개선` 흐름을 Claude에 강제/자동화한다. MIT 라이선스, 버전 2.2.3 (GitHub ⭐ 약 29.9만, 2026-11-08 기준).

## Claude와 같이 쓰면 좋은 점

- **역할별 서브에이전트**를 바로 씀: `ecc:planner`, `ecc:architect`, `ecc:code-reviewer`, `ecc:tdd-guide`, `ecc:security-reviewer`, `ecc:build-error-resolver`, 언어별 리뷰어(Go/Python/TS/F# …) 등.
- **워크플로 커맨드/스킬**: `/ecc:plan`(구현 계획), `tdd-workflow`, `/build-fix`, `/refactor-clean`, `/context-budget`, `/save-session`·`/resume-session` 등.
- **Hooks로 자동 가드레일**: 파괴적 명령 차단(GateGuard), 첫 Edit 전에 "호출부 확인·지시 인용"을 강제(Fact-Forcing Gate), lint/formatter 설정 수정 차단, 응답 끝날 때 Prettier/Biome·`tsc` 일괄 실행, 세션 상태 저장, 비용 추적.
- **지속 학습**: 세션에서 반복된 패턴을 `~/.claude/skills/learned/`에 스킬로 쌓는 continuous-learning, `/skill-create`로 git 히스토리에서 스킬 생성.
- **AgentShield**: 내 프롬프트·hooks·MCP 설정·권한·비밀값 노출을 스캔 (`/security-scan`, `ecc-agentshield` npm).

---

## 1. 연동 방법

- 유형: **Claude Code Plugin** (marketplace `ecc`, 플러그인 ID `ecc@ecc`) + 선택적 rules 복사 + npm CLI(`ecc-universal`)
- 설치(테스트용, 이 폴더): `git clone --depth 1 https://github.com/affaan-m/ECC.git plugin` → `claude -p "..." --plugin-dir ./plugin`
- 전역 설치 방법 (택 1, **중복 설치 금지** — 스킬·hook이 두 번 뜸):
  - 권장: `npx ecc-universal@2.2.3 setup` (대화형; Global user / Standard hooks 선택)
  - Claude Code 내장: `/plugin marketplace add https://github.com/affaan-m/ECC` → `/plugin install ecc@ecc`
  - 저(低)컨텍스트·hook 없이: `npx ecc-universal@2.2.3 install --profile minimal --target claude`
- 삭제: `/plugin uninstall ecc@ecc` (또는 `npx ecc-universal@2.2.3 uninstall --dry-run` → `uninstall`), 수동 복사한 `~/.claude/rules/ecc/` 삭제
- rules는 플러그인으로 배포되지 않음 → 필요하면 `cp -R plugin/rules/common ~/.claude/rules/ecc/` + 쓰는 언어 1개만 (미검증)
- 인증: 없음 (OSS 플러그인). GitHub App(ecc.tools)은 GitHub 로그인
- 요구사항: Node.js 18+, Git, Claude Code 2.1+ (이 기기: Claude Code 2.1.139, node 있음)
- 현재 이 기기 상태: **삭제됨 (2026-11-08)** — 테스트용 클론 `ECC/plugin/`만 남음. 아래는 설치했다가 지운 기록
  - 삭제 이유: minimal인데도 모든 세션에 +약 13k 토큰, 설정 파일 몰래 수정, 내장 `/code-review`와 이름 충돌
  - 삭제 명령: `npx -y ecc-universal@2.2.3 uninstall` (401개 경로 제거, settings.json 변화 없음) + 남은 것 수동 삭제: `~/.claude/ecc/state.db`(uninstall 후에도 남음), 테스트 때 hook이 만든 `~/.claude/metrics/`, `~/.claude/session-data/`, `~/.claude/skills/learned/`, `~/.claude/bash-commands.log`, `~/.claude/cost-tracker.log` (로그 2개는 메인 세션이 삭제, 2026-11-08)
- (설치했던 내용) **전역 설치 — minimal 프로필(hook 없음)**, 2026-11-08
  - 명령: `npx -y ecc-universal@2.2.3 install --profile minimal --target claude`
  - 설치된 것: `~/.claude/agents/` 68개, `~/.claude/commands/` 94개, `~/.claude/skills/` 약 40개, `~/.claude/rules/ecc/` (common + 언어 22종), 설치 기록 `~/.claude/ecc/install-state.json`
  - 삭제: `npx -y ecc-universal@2.2.3 uninstall --dry-run` → `npx -y ecc-universal@2.2.3 uninstall`
  - ⚠️ 설치 프로그램이 dry-run에 표시되지 않은 `"includeCoAuthoredBy": false`를 `~/.claude/settings.json`에 추가함 (`scripts/lib/claude-commit-attribution.js`) → 사용자가 요청한 게 아니라서 바로 지움
  - 컨텍스트 비용: 세션 입력 토큰 약 21.8k → 34.7k (**모든 프로젝트에서 세션마다 +약 13k**). 언어별 rules는 `paths:`가 있어 해당 파일을 다룰 때만 로드되지만, `rules/ecc/common/` 10개 파일(약 40KB)은 항상 로드됨
  - plugin 방식과 달리 `/ecc:` 접두어가 없음 → `/plan`, `/code-review`처럼 이름이 바로 쓰이고, Claude Code 내장 `/code-review`와 이름이 겹침

### Hook 제어 (환경변수)

| 변수 | 값 | 효과 |
|---|---|---|
| `ECC_HOOKS_ENABLED` | `true`/`false` | hook 전체 켜기/끄기 (플러그인 옵션 `hooks_enabled`와 동일) |
| `ECC_HOOK_PROFILE` | `minimal`/`standard`/`strict` | hook 강도 (기본 standard) |
| `ECC_DISABLED_HOOKS` | `id1,id2` | 개별 hook 끄기 (예: `pre:edit-write:gateguard-fact-force`) |

## 2. 주요 기능 / 도구

| 용도 | 도구·명령 |
|---|---|
| 기능 구현 계획 | `/ecc:plan "설명"` → `ecc:planner` 에이전트 |
| 테스트 먼저 구현 | `tdd-workflow` 스킬 → `ecc:tdd-guide` |
| 코드 리뷰 | `/code-review` → `ecc:code-reviewer` (※ Claude Code 내장 `/code-review` 스킬과 이름이 겹침) |
| 빌드 에러 수정 | `/build-fix` → `ecc:build-error-resolver` |
| 보안 점검 | `/security-scan` → `ecc:security-reviewer`, `agentshield scan --path .` |
| 죽은 코드 정리 | `/refactor-clean` |
| 컨텍스트 점검 | `/context-budget` |
| 세션 저장/재개 | `/save-session`, `/resume-session`, `/learn-eval` |
| GitHub 이슈 검토 | `/ecc:issue-review <번호>` |
| 스킬 생성 | `/skill-create` (로컬 git 히스토리 분석) |

### 주요 Hooks (standard 프로필)

| 이벤트 | Hook | 동작 |
|---|---|---|
| PreToolUse(Bash) | dispatcher / GateGuard | `rm`·강제 `git checkout` 등 파괴적 명령 게이트, tmux 밖 dev 서버 차단, push 전 리마인드, 커밋 전 품질 검사 |
| PreToolUse(Edit/Write) | gateguard-fact-force | **파일별 첫 수정을 거부**하고 "호출부 목록 + 사용자 지시 원문"을 먼저 제시하게 함 |
| PreToolUse(Edit/Write) | config-protection | lint/formatter 설정 수정 차단 |
| PreToolUse(mcp__) | mcp-health-check | 비정상 MCP 호출 차단 |
| SessionStart | session:start | 이전 세션 컨텍스트 로드, 패키지 매니저 감지 |
| Stop | format-typecheck / cost-tracker / evaluate-session / desktop-notify | 포맷·타입체크, 비용 기록, 패턴 추출, macOS 알림 |

## 3. 사용 예시

```
/ecc:plan "div 함수에 0으로 나누기 에러 처리 추가"
tdd-workflow 스킬로 로그인 버그 재현 테스트부터 작성해줘
/build-fix
/context-budget
```

## 4. 요금 · 한도 (2026-11-08 기준, https://ecc.tools/pricing)

| 플랜 | 가격 | 핵심 |
|---|---|---|
| OSS 플러그인 | 무료 (MIT) | 스킬·에이전트·커맨드·hooks·학습 기능 전부 |
| GitHub App Free | $0/월 | **공개 저장소만**, 월 10회 분석, 실행당 200 커밋 |
| Pro | $19/seat/월 (연 $190) | 비공개 저장소, seat당 월 50회(설치 단위 공유), 실행당 1,000 커밋, AgentShield 스캔, PR 자동 트리거 |
| Enterprise | 문의 | 무제한, 실행당 5,000 커밋, SSO·감사 로그 |

- GitHub App FAQ: 코드는 분석 중 메모리에서만 처리하고 영구 저장하지 않는다고 함 (개인정보처리방침 본문은 미확인).

## 5. 테스트 기록

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 플러그인 로드 | `claude -p "Reply with just OK." --plugin-dir ./plugin --output-format stream-json --verbose` | ✅ `ecc@inline` 로드, `ecc:` 에이전트 71개 + 슬래시 항목 400개, SessionStart hook 2개 실행 |
| 2026-11-08 | 컨텍스트 오버헤드 | 같은 프롬프트를 플러그인 없이 실행해 비교 | 입력 토큰 약 21.8k → 31.1k (**+약 9.3k 토큰/세션**), "OK" 한 번에 $0.028 → $0.076 |
| 2026-11-08 | `/ecc:plan` 커맨드 | `cd sandbox && ECC_HOOKS_ENABLED=false claude -p '/ecc:plan "div 함수에 0으로 나누기 에러 처리 추가" …' --plugin-dir ../plugin --allowedTools "Read,Glob,Grep,Skill,Agent"` | ✅ 대상 파일·단계·리스크·복잡도가 담긴 계획 출력 후 "진행할까요?"로 멈춤 (코드 수정 없음), 3턴 $0.11 |
| 2026-11-08 | GateGuard hook | `cd sandbox && claude -p "calc.js div를 Edit로 수정…" --plugin-dir ../plugin --allowedTools "Read,Edit,Glob,Grep"` | ✅ 첫 Edit가 `[Fact-Forcing Gate]`로 거부됨 → Claude가 grep으로 호출부 확인 후 재시도 → 수정 성공. $0.12 |

| 2026-11-08 | 전역 minimal 설치 | `npx -y ecc-universal@2.2.3 install --profile minimal --target claude` (먼저 `--dry-run`) | ✅ 약 400개 파일 설치. ⚠️ settings.json에 `includeCoAuthoredBy:false`가 몰래 추가돼서 지움. 세션당 +약 13k 토큰 |

테스트 중 ECC hooks가 전역 폴더에 만든 것: `~/.claude/metrics/`(비용 기록), `~/.claude/session-data/`(세션 요약), `~/.claude/skills/learned/`(빈 폴더). 설정 파일(`settings.json`, `~/.claude.json`)은 바뀌지 않음.

## 6. 주의사항 · 팁

- **무겁다**: 스킬·에이전트 설명만으로 세션마다 약 9k 토큰이 늘어난다. 필요한 것만 쓰려면 `--profile minimal`이나 선택 설치(`install --modules …`)를 고려.
- **hooks가 동작을 바꾼다**: 첫 Edit 거부, lint 설정 수정 차단, tmux 밖 `npm run dev` 차단 등 → 처음엔 당황할 수 있음. 불편하면 `ECC_HOOK_PROFILE=minimal` 또는 `ECC_DISABLED_HOOKS`.
- hooks가 `~/.claude/` 아래에 세션·비용·학습 데이터를 기록한다. hook 스크립트에서 외부 네트워크 호출은 확인되지 않았음 (`plan-canvas-pending`만 로컬 HTTP).
- **설치는 한 가지 방법만**: 플러그인 + `install.sh --profile full`을 겹치면 스킬·hook이 중복 실행됨. `hooks.json`을 `settings.json`에 직접 붙여넣지 말 것.
- 이름 충돌: ECC의 `/code-review`, `/plan` 등은 Claude Code 내장 스킬과 겹친다. 플러그인 설치 시 `/ecc:` 접두어로 구분해서 부를 것.
- 공식 배포처만 사용: GitHub `affaan-m/ECC`, npm `ecc-universal`/`ecc-agentshield`, 플러그인 `ecc@ecc`, ecc.tools. 비공식 미러는 악성코드 위험 경고가 README에 있음.
- 레포 안 `CLAUDE.md`·`.claude/rules/`는 ECC 개발자용이라, 클론한 폴더에서 Claude를 실행하면 그 규칙이 로드된다. 테스트는 `sandbox/` 같은 별도 폴더에서 할 것.

## 참고 링크

- GitHub: https://github.com/affaan-m/ECC
- 웹사이트·요금: https://ecc.tools · https://ecc.tools/pricing
- npm: https://www.npmjs.com/package/ecc-universal · https://www.npmjs.com/package/ecc-agentshield
- GitHub App: https://github.com/apps/ecc-tools
- 레포 내부 문서: `plugin/hooks/README.md`, `plugin/docs/token-optimization.md`, `plugin/SECURITY.md`
