# security-audit-skill (Cloudflare)

> 정리일: 2026-11-08 · 상태: ✅ 연동·테스트 완료 (2026-11-08 전역 설치됨) · 공식: https://github.com/cloudflare/security-audit-skill
> 확인한 커밋: `c1c8a8c` (2026-09-14, "Clarify guidance and full audit modes") · 라이선스 MIT · GitHub 별 약 65k (2026-11-08 `gh api` 기준)

## 한 줄 요약

Claude Code 같은 코딩 에이전트를 **소스코드 보안 감사관**으로 바꾸는 Agent Skill.
정찰 → 커버리지 기반 헌팅 → 독립 검증 → JSON 기록 → 재검증 → 리포트의 6단계를 **서로 격리된 하위 에이전트**로 돌린다.
Cloudflare의 사내 취약점 탐지 하네스가 이 스킬에서 출발했다 (블로그 "Build your own vulnerability harness", 2026-06-18).

## Claude와 같이 쓰면 좋은 점

- "이 코드 보안 감사해줘" 한마디로 **Claude가 하위 에이전트를 여러 개 띄워** 공격 클래스별로 나눠 찾고, 찾은 에이전트와 **다른 에이전트가 반박 검증**한다 → 그냥 "취약점 찾아줘"보다 오탐이 적다.
- 결과가 산문이 아니라 **스키마 검증된 `findings.json`** + 커버리지 장부(`coverage-ledger.json`)로 남아서, 다음 실행 때 빈 곳만 골라 이어서 감사할 수 있다 (실행할수록 누적).
- 가벼운 질문("이 함수 SQLi 가능해?")엔 **가이던스 모드**로 방법론만 빌려 쓰고 파일을 안 만든다.
- 판정 규칙이 엄격하다: 실제 신뢰 경계 침범 + 관찰된 결과가 있어야 `confirmed`, 확인 못 한 건 `needs_validation`(심각도 없음), 반박된 건 `rejected`. "모범사례 미준수"는 취약점으로 안 친다.

---

## 1. 연동 방법

- 유형: **Skill** (Agent Skills 형식, `SKILL.md` + 동반 문서 15개 + Node 검증 스크립트 2개). MCP·API 키 없음.
- 설치(테스트용, 이 폴더): 테스트 대상 프로젝트 안에 프로젝트 범위로 설치
  ```bash
  cd test-target
  npx -y skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --agent claude-code -y
  # → test-target/.claude/skills/security-audit/ 에 복사 + skills-lock.json 생성
  ```
- 전역 등록 (2026-11-08 사용자 승인 후 실행함):
  ```bash
  npx -y skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --agent claude-code --global -y
  # → ~/.claude/skills/security-audit/ 에 복사, ~/.agents/.skill-lock.json 에 출처 기록
  ```
- 삭제: `npx skills remove security-audit --global` (미검증) 또는 `rm -rf ~/.claude/skills/security-audit` 후 `~/.agents/.skill-lock.json`의 `security-audit` 항목 제거
- 업데이트: 자동 업데이트 없음. 새 버전을 받기 전에 GitHub에서 바뀐 내용을 먼저 확인할 것 (전체 에이전트 권한으로 실행되는 지시문이라서)
- 인증: 없음
- 요구사항: 하위 에이전트(Agent/Task 도구)를 쓸 수 있는 에이전트, Node.js(검증 스크립트용, 이 기기 v26.9.0 ✅)
- 현재 이 기기 상태: **전역 설치됨** (`~/.claude/skills/security-audit/`, 커밋 `c1c8a8c`, 검토한 클론과 `diff -r` 동일). 테스트용 복사본(`test-target/.claude/skills/`)도 남아 있어 이 폴더 안에선 두 개가 보인다.
- 이 폴더 구성:
  - `repo/` — GitHub 원본 클론 (`--depth 1`)
  - `test-target/` — 일부러 취약하게 만든 테스트 앱(`app.js`) + 프로젝트 범위로 설치된 스킬
  - `test-runs/` — 헤드리스 테스트 출력물

## 2. 동작 구조

### 두 가지 모드

| 모드 | 언제 | 하는 일 |
|---|---|---|
| 가이던스 모드 (기본) | 보안 질문, 특정 파일/함수 리뷰, 취약점 하나 조사 | 필요한 부분만 사용. 6단계·출력 폴더·파일 생성 **안 함** |
| 전체 감사 모드 | "코드베이스 감사/펜테스트", "전체·포괄적 리뷰", "리포트 파일 달라" | 6단계 전부 실행, 출력 폴더에 산출물 작성 |

애매하면 스킬이 한 번 물어본다.

### 전체 감사 6단계

| 단계 | 내용 | 산출물 |
|---|---|---|
| 1. 정찰 | 아키텍처·신뢰 경계·입력 표면·기존 실행 결과 파악, 커버리지 단위(표면×경계×공격 클래스) 생성 | `architecture.md`, `coverage-ledger.json`, `run-metadata.json` |
| 2. 헌팅 | 장부의 단위별로 격리된 헌터 에이전트 배정, 커버리지 비평가(critic)가 빈틈 찾기 | 후보(candidate) |
| 3. 후보 검증 | 후보마다 **새 검증 에이전트**가 반박 시도 | confirmed / needs_validation / rejected |
| 4. 구조화 출력 | `report-schema.json` + `validate-findings.cjs`, `validate-coverage-ledger.cjs`로 검증 | `findings.json` |
| 5. 독립 재검증 | 또 다른 새 에이전트가 최종 소스 주장 재확인 | (수정 시 재검증) |
| 6. 리포트 | 검증된 기록에서만 생성 | `REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md` |

- 출력 폴더 기본값: `~/security-audit-skill/<repo-name>/run-<N>` (대상 저장소 **밖**). 저장소 안은 git-ignore된 폴더를 명시했을 때만.
- 프로필: `quick`(헌터 1웨이브 + 비평 1회) / `standard`(기본) / `deep`(대형·고위험).
- 범위 지정 실행(경로·서브시스템·두 커밋 diff) 가능, 나머지는 `out_of_scope`로 기록.
- **예산**: "에이전트 호출 N회"로 상한 지정 가능. 정찰 4회 + 비평 + 검증 몫을 먼저 떼고, 부족하면 시작 안 함.

### 동반 문서 (대상에 맞춰 골라 읽음)

| 파일 | 다루는 영역 |
|---|---|
| `ATTACK-CLASSES.md` | 핵심·와일드카드·"뻔한 것" 공격 클래스 |
| `WEB-PROTOCOL-AND-AUTH.md` | HTTP 요청 분할, 캐시, 인증 프로토콜 |
| `CLIENT-SIDE.md` | DOM 주입, postMessage, UI redress, 프로토타입 오염 |
| `AI-AND-LLM.md` | 프롬프트 인젝션, 에이전트/도구 오남용, 출력 처리 |
| `MEMORY-SAFETY-AND-BINARY.md` | 네이티브·바이너리·커널 |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 의존성, CI, 서명, 업데이트, 플러그인 |
| `CLOUD-AND-DEPLOYMENT.md` | IAM, IaC, 컨테이너, 서버리스 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC, 직렬화, 큐, 웹훅 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 쿼터, 큐, 비용 폭증 |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | 테넌트 격리, 백업·삭제·복원 |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | 딥링크, 웹뷰, 로컬 IPC |

### 심각도 기준 (스킬 내 정의)

- **critical** 비인증 원격 코드 실행 / 데이터 저장소 전체 접근 / 임의 계정 탈취
- **high** 명시적 보안 통제를 완전히 무력화 (인증 우회, 교차 테넌트 읽기·쓰기, 인증된 RCE 등)
- **medium** 실제 경계 침범이지만 영향 범위 제한적
- **low** / **informational**

## 3. 사용 예시

```
security audit this codebase                       # 전체 감사 (standard)
security audit this codebase, quick profile, budget 12 agents, output to ~/audits/myapp
find security vulnerabilities in ./src             # 범위 지정
이 라우트에 IDOR 있는지 봐줘                          # 가이던스 모드 (파일 생성 없음)
```

## 4. 요금 · 한도 (2026-11-08 기준)

| 항목 | 내용 |
|---|---|
| 스킬 자체 | 무료 (MIT) |
| 실제 비용 | Claude 사용량. 전체 감사는 하위 에이전트를 여러 개 띄우므로 **토큰 소모가 큼** → 예산(에이전트 호출 수) 지정 권장 |
| 블로그 수치 (Cloudflare 사내 하네스 기준, 이 스킬 단독 아님) | 약 3만 LOC 저장소: 초기 발견 ~100건에 3~4시간. 1회 실행은 반복 실행 대비 **약 절반**만 찾음 |

## 5. 테스트 기록

테스트 대상: `test-target/app.js` — 일부러 만든 66줄짜리 Node HTTP 서버. **정답(심어둔 취약점, 대상 코드엔 힌트 없음)**:
① `/invoices/:id` 소유자 검사 없음(IDOR) ② `/ping` 명령 주입(`exec` + 쿼리값) ③ `/files` 경로 탐색(`path.join`만 사용). 미끼: `/me`(안전).

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 동봉 검증기 단위 테스트 | `node --test validate-findings.test.cjs validate-coverage-ledger.test.cjs` | ✅ 65/65 통과 (0.9초) |
| 2026-11-08 | Skills CLI로 프로젝트 범위 설치 | `npx -y skills add ... --agent claude-code -y` (test-target 안) | ✅ 설치본이 클론과 `diff -r` 동일, 전역 폴더 변화 없음 |
| 2026-11-08 | 가이던스 모드 (읽기 전용 도구만) | `claude -p "Use the security-audit skill to do a focused security review of app.js..." --allowedTools "Skill Read Grep Glob" --disallowedTools "Bash Write Edit" --max-budget-usd 3` | ✅ 정답 3개 모두 confirmed로 탐지 (명령 주입 high, 경로 탐색 high, IDOR medium) + 하드코딩 토큰 informational 1건. 파일 생성 없음. 26초, 5턴, API 환산 $0.14 |
| 2026-11-08 | 전체 감사 모드 (quick, 에이전트 12회 예산, 샌드박스 없음 명시) | 아래 명령 | ✅ `run_status: complete`. 정답 3개 모두 confirmed **high**. 총 6건 confirmed(high 5, low 1), deferred 1단위(자원 고갈). 두 검증기 내가 직접 재실행해도 PASS. 대상 코드 변경 없음(`git status` 깨끗). **29.2분, 64턴, 에이전트 12회(정찰 4·헌터 1·비평 1·검증 6), API 환산 $4.02** |
| 2026-11-08 | 전역 설치 후 다른 폴더의 새 세션에서 인식 확인 | `claude -p "Is a skill named security-audit listed ...?" --tools "Skill"` (scratchpad 폴더에서) | ✅ `SKILL_FOUND`. 참고: `--tools ""`로 도구를 다 끄면 Skill 도구도 꺼져서 `SKILL_MISSING`으로 나옴 |

전체 감사 명령 (출력: `test-runs/run-1/`, 원본 로그: `test-runs/full-run1.jsonl`):

```bash
cd test-target && claude -p "Security audit this codebase (full audit mode) using the security-audit skill. \
Use the quick profile with a budget of 12 agent invocations. Output directory: <절대경로>/test-runs/run-1 . \
There is no OS-enforced sandbox available, so do not execute target code. Do not ask questions; ..." \
  --add-dir <절대경로>/test-runs \
  --allowedTools "Skill Read Grep Glob Agent Task Write Edit Bash(node *) Bash(mkdir *) Bash(git *) Bash(ls *) ..." \
  --max-budget-usd 15 --output-format stream-json --verbose < /dev/null
```

산출물: `REPORT.md`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`, `findings.json`, `coverage-ledger.json`, `architecture.md`, `run-metadata.json`, `agents/<에이전트별 scratch·artifacts>`

전체 감사 결과에서 눈여겨본 점:
- 👍 예산(12회)을 정확히 지켰고, 정찰→헌팅→비평→후보별 검증→JSON 검증→리포트 순서를 그대로 따랐다. 다 못 본 영역(자원 고갈)은 숨기지 않고 `deferred`로 남기고 "부분 커버리지"라고 리포트에 적었다.
- 👍 가이던스 모드에선 medium이던 IDOR을 전체 감사에선 스킬 기준대로 high로 매겼다.
- 👎 **과대 평가·중복**: 테스트용 하드코딩 토큰을 별개 2건(소스에 토큰 노출 / 만료 없음)으로 나눠 둘 다 high로 매김. 사람이 보면 하나로 묶고 낮출 내용.
- 👎 **리포트 문구와 JSON 불일치**: 최종 요약은 "5 confirmed (4× high, 1× low)"인데 `findings.json`·리포트 표는 6건(high 5). 스킬이 스스로 금지한 안티패턴 #10("prose와 JSON 불일치")이 작은 대상에서도 생겼다.
- ⚠️ 코드를 실행하지 않았는데도 `confirmed`가 나왔다. `observed_result`가 "Source confirms ..." 같은 정적 분석 서술이다. 정적으로 확실하면 confirmed로 인정하는 셈이니, 실행 증거로 받아들이면 안 된다.
- 💸 같은 66줄 파일 기준으로 가이던스 모드($0.14, 26초)의 **약 30배 비용, 70배 시간**. 큰 저장소에선 예산 지정이 필수.

## 6. 주의사항 · 팁

- **스킬은 에이전트 전체 권한으로 실행된다** (Skills CLI도 설치 후 "Review skills before use" 경고). 이 저장소는 마크다운 지시문 + 의존성 없는 Node 검증기(`fs`/`path`/`util`만 사용, 네트워크·프로세스 실행 없음)라 확인 결과 문제없음.
- **샌드박스가 없으면 대상 코드를 실행하지 않는다** — 빌드·테스트·PoC 실행은 네트워크 차단·환경변수 비움·쓰기 제한·자원 제한이 OS 수준으로 다 갖춰졌을 때만. 아니면 `needs_validation`으로 남긴다. macOS 일반 Claude Code 세션은 이 조건을 다 만족한다고 보기 어렵다 → 사실상 **정적(소스) 분석 위주**.
- 배포된 서비스·실제 엔드포인트에 프로브하지 않는다 (설계상 금지). 실서비스 침투 테스트 도구가 아님.
- 1회 실행 = 전체 커버리지 아님. 같은 출력 루트로 여러 번 돌리면 이전 장부를 읽고 빈 곳을 메운다.
- 심각도는 사람이 다시 봐야 한다: 가이던스 모드는 IDOR을 medium(스킬 기준상 high), 전체 감사는 테스트용 토큰을 high 2건으로 과대 평가.
- 전체 감사는 하위 에이전트 수가 많아 사용량을 빨리 쓴다 (66줄에 $4, 29분) → `quick` 프로필 + "budget N agents" 지정으로 먼저 시험. 평소 빠른 확인은 가이던스 모드로 충분.
- 출력 폴더는 대상 저장소 **밖**에 두는 게 기본이다. 저장소 안에 두려면 git-ignore된 폴더여야 한다.
- 헤드리스(`claude -p`)로 돌릴 땐 `Agent`/`Task` 도구와 출력 폴더 쓰기 권한(`--add-dir`)을 허용해야 6단계가 돈다.

## 참고 링크

- GitHub: https://github.com/cloudflare/security-audit-skill
- Cloudflare 블로그 "Build your own vulnerability harness" (2026-06-18): https://blog.cloudflare.com/build-your-own-vulnerability-harness
- Skills CLI: https://skills.sh
