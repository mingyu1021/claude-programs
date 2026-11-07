# HyperFrames

> 정리일: 2026-11-08 · 상태: ✅ 연동·테스트 완료 (테스트용 `--plugin-dir`, 전역 미설치) · 공식: https://github.com/heygen-com/hyperframes · https://hyperframes.heygen.com

## 한 줄 요약

**HTML·CSS·애니메이션(GSAP 등)으로 만든 페이지를 MP4 영상으로 렌더링**하는 오픈소스 프레임워크 (HeyGen 제작, Apache 2.0).
"Write HTML. Render video. Built for agents." — Claude 같은 코딩 에이전트가 영상을 *코드로* 만들도록 설계됐다.
v0.9.20 (2026-11-07), GitHub ★ 약 8만.

## Claude와 같이 쓰면 좋은 점

- **"10초짜리 제품 소개 영상 만들어줘" → MP4**: Claude가 HTML 컴포지션을 쓰고, lint·check·렌더까지 CLI로 직접 돌린다.
  → 실제 테스트에서 프롬프트 한 번으로 6초 1080p 인트로 영상이 나왔다 (아래 테스트 기록).
- 에이전트가 원래 잘 쓰는 **HTML/CSS**가 그대로 영상 포맷이다. React·빌드 과정·독자 포맷이 없다.
- **결정적(deterministic) 렌더링**: headless Chrome으로 프레임을 하나씩 찍고 FFmpeg으로 인코딩한다. 같은 입력이면 같은 영상이 나와서 CI 자동화에 적합하다.
- Skill이 "계획 → HTML 작성 → 애니메이션 → lint → check(브라우저 검사: 레이아웃·대비·모션) → 렌더" 작업 순서를 알려줘서, Claude가 스스로 검증하고 고친다.
- 약 450개 블록(전환 효과, 자막, 차트, 지도, 3D 등)을 `npx hyperframes add <이름>`으로 가져다 쓸 수 있다.
- 활용 예: 기능 출시 영상, 코드 diff를 애니메이션으로 보여주는 PR 설명 영상, 데이터 차트 애니메이션, 문서·PDF를 영상으로 바꾼 설명 영상, 자막 붙은 SNS 영상.

---

## 1. 연동 방법

연동 경로가 셋 있다. **로컬 플러그인(Skill) + CLI가 기본**이고, 테스트도 이 방식으로 했다.

| 경로 | 무엇 | 인증 | 이 기기 |
|---|---|---|---|
| **Plugin (Skill 9개)** | Claude Code에 작업 순서·규칙을 가르침. 훅·MCP 없이 순수 Skill만 | 없음 | 테스트용 (`src/`를 `--plugin-dir`로 로드) |
| **CLI** (`npx hyperframes`) | 프로젝트 생성·lint·check·미리보기·렌더 | 없음 (로컬 렌더 무료) | npx로 실행 (전역 설치 안 함) |
| **호스팅 MCP** (`https://mcp.heygen.com/mcp/hyperframes/`) | 설치 없이 채팅에서 만들고 HeyGen 클라우드로 렌더. **베타** | HeyGen 계정 OAuth | 미테스트 |

- 플러그인 전역 설치:
  ```bash
  claude plugin marketplace add heygen-com/hyperframes
  claude plugin install hyperframes@hyperframes
  ```
  삭제: `claude plugin uninstall hyperframes@hyperframes` → `claude plugin marketplace remove hyperframes`
- Skill만 따로 설치하는 방법 (`~/.claude/skills`에 들어감): `npx hyperframes skills update` (핵심 4개) / `npx skills add heygen-com/hyperframes`
- 요구사항: **Node.js 22 이상, FFmpeg**, Chrome (시스템 Chrome을 쓰거나 `~/.cache/hyperframes/chrome`에 내려받음)
- 이 기기에서 한 일:
  - `brew install ffmpeg` (FFmpeg 9.0.2) — 삭제: `brew uninstall ffmpeg`
  - Node v26.9.0, Chrome, Docker는 원래 있었음 (`npx hyperframes doctor`에서 ✓)
- 이 폴더 구성:
  - `src/`: 저장소 clone, 421MB. Git LFS(테스트용 MP4 약 240MB)는 받지 않음 (`git lfs`가 없어 일반 clone이 실패하므로 LFS 필터를 끄고 clone)
  - `test/`: 테스트 프로젝트 (`hello/`, `intro/`)
  - `.env`: `HYPERFRAMES_NO_TELEMETRY=1`, `DO_NOT_TRACK=1`, `HYPERFRAMES_SKIP_SKILLS=1`

## 2. 주요 기능 / 도구

### Skill (플러그인에 9개 포함, 로드 시 `hyperframes:` 접두사)

| Skill | 용도 |
|---|---|
| `/hyperframes` | **먼저 읽는 핵심 스킬**: 렌더 규칙(contract)과 작업 순서, 포맷·모션·룩·자막 참고 자료 |
| `/hyperframes-cli` | CLI 사용법 (생성·검사·프레임 확인·타임라인 편집·렌더·실패 복구) |
| `/hyperframes-registry` | 약 450개 기성 블록·컴포넌트 (`hyperframes add`) |
| `/media-use` | 음악·효과음·이미지·아이콘·내레이션·자막 만들기 (HeyGen 또는 무료 로컬 도구) |
| `/figma` | Figma 디자인을 영상으로 |
| `/slideshow` | 발표 덱 (MP4가 아니라 넘겨 보는 덱) |
| `/remotion-to-hyperframes` | Remotion 등 기존 영상 코드를 이식 |
| `/reverse-any-video` | 영상 파일을 편집 가능한 프로젝트로 재구성 (**Pro**) |
| `/style-sampling` | 검증된 5가지 스타일로 에피소드 제작 (**Pro**) |

### CLI 주요 명령

| 용도 | 명령 |
|---|---|
| 환경 점검 | `npx hyperframes doctor` |
| 프로젝트 생성 | `npx hyperframes init <이름> --non-interactive --skip-skills` |
| 정적 검사 / 브라우저 검사 | `npx hyperframes lint` / `npx hyperframes check` |
| 미리보기 (실시간 반영) | `npx hyperframes preview` |
| 렌더 | `npx hyperframes render -o out.mp4` (`--gpu`, `--fps`) |
| 블록 추가 | `npx hyperframes add data-chart` |
| 텔레메트리 끄기 | `npx hyperframes telemetry disable` |
| 클라우드 렌더 (유료 크레딧) | `npx hyperframes cloud render` |

### 컴포지션 형식 (요약)

`data-start`, `data-duration`, `data-track-index` 같은 속성으로 타이밍과 트랙을 정하고, 멈춰 둔(paused) GSAP 타임라인을 `window.__timelines["<id>"]`에 등록한다.
렌더러가 이 타임라인을 프레임 단위로 seek하며 캡처한다. 그래서 `Date.now()`, 시드 없는 `Math.random()`, 렌더 중 네트워크 요청은 쓰면 안 된다.

## 3. 사용 예시

```
/hyperframes:hyperframes 로 15초짜리 제품 소개 영상 만들어줘.
배경 영상 위에 제목이 페이드인, 잔잔한 배경음악, 마지막에 로고. lint·check 통과시키고 out.mp4로 렌더해줘.
```
```
이 PR의 변경 내용을 30초짜리 설명 영상으로 만들어줘 (코드 diff 애니메이션 + 자막).
```

## 4. 요금 · 한도 (2026-11-08 기준, 공식 FAQ)

| 항목 | 가격 | 핵심 |
|---|---|---|
| 프레임워크·CLI·Skill | **무료** (Apache 2.0) | 로컬 렌더 무료, 계정 불필요, 상업적 사용 가능, 렌더당 요금 없음 |
| Studio 앱 Free | 무료 | AI 내레이션·자막·음악·효과음, AI 영상 클립 월 3개, 클라우드 내보내기 월 5회 |
| Studio 앱 Pro | **월 $25** | HeyGen 최상위 영상·이미지·음성·음악 모델, 업로드한 영상 스타일 따라 만들기, 클라우드 내보내기 무제한 |
| 클라우드 렌더 / HeyGen 음성·아바타 | HeyGen 크레딧 | `hyperframes cloud render`, `/media-use`의 HeyGen 기능 |

- 실제 비용은 대부분 **Claude 사용량**이다. 테스트에서 6초 인트로 하나에 $0.62 (19턴, 약 4분)가 들었다.

## 5. 테스트 기록

| 날짜 | 무엇을 | 명령 | 결과 |
|---|---|---|---|
| 2026-11-08 | 환경 점검 | `npx -y hyperframes@0.9.20 doctor` | ✅ FFmpeg·FFprobe·Chrome·Docker ✓. 선택 항목인 whisper-cpp(자막), Kokoro TTS, MusicGen만 ✗ |
| 2026-11-08 | 기본 템플릿 렌더 | `init hello --non-interactive --skip-skills` → `lint` → `render -o baseline.mp4` | ✅ lint 0 errors · 10초 1920×1080 H.264 · 29KB · **렌더 6.8초** |
| 2026-11-08 | **Claude가 처음부터 영상 제작** | `claude -p "/hyperframes:hyperframes 로 6초 인트로 …, lint·check 후 out.mp4 렌더" --plugin-dir ./src --allowedTools "Skill" "Read" "Write" "Edit" "Bash(npx hyperframes@0.9.20:*)" …` | ✅ 장면을 `compositions/scene.html`로 분리해 작성. lint·check 0 errors (경고 1개: 첫 프레임이 빔, 페이드인 요청 때문이라 의도된 것). 6초 1080p · 58KB · 렌더 7.2초. **$0.62, 19턴, 약 4분** |
| 2026-11-08 | 결과 영상 프레임 확인 | `ffmpeg -vf select=…` 로 0.5초 / 2초 / 5.7초 프레임 추출 | ✅ 제목만 → 제목+부제목(**한글 정상 렌더**) → 페이드아웃, 요청대로 |
| — | 호스팅 MCP (HeyGen OAuth) | — | 미테스트 (HeyGen 계정 필요) |

- 테스트할 때는 `.env`로 텔레메트리를 끄고(`HYPERFRAMES_NO_TELEMETRY=1`), `--skip-skills`로 전역 `~/.claude/skills`를 건드리지 않았다.
- 결과물: [test/intro/out.mp4](test/intro/out.mp4), [test/hello/baseline.mp4](test/hello/baseline.mp4)

## 6. 주의사항 · 팁

- **텔레메트리가 기본으로 켜져 있다**: 명령 이름, 렌더 성능 같은 사용 데이터를 익명으로 보낸다 (로그인하면 계정과 연결됨). 프로젝트 이름과 영상 내용은 보내지 않는다고 공식 FAQ에 적혀 있다.
  끄기: `npx hyperframes telemetry disable` 또는 환경변수 `HYPERFRAMES_NO_TELEMETRY=1` / `DO_NOT_TRACK=1`.
  이 기기의 `~/.hyperframes/config.json`은 아직 `telemetryEnabled: true`다 (테스트는 환경변수로 껐음).
- **`init`이 전역 Skill을 설치한다**: 기본 동작이 핵심 Skill 4개를 설치하는 것이다. 원치 않으면 `--skip-skills` 또는 `HYPERFRAMES_SKIP_SKILLS=1`. 플러그인을 쓸 거라면 Skill이 중복 설치되지 않게 하는 게 좋다.
- **HyperFrames 저장소 안에서 `npx skills add ... --all`을 실행하지 말 것**: 저장소의 `skills/` 폴더를 지우고 덮어쓴다 (공식 경고).
- 렌더 중에는 네트워크 요청을 하면 안 된다. 다만 기본 템플릿이 GSAP을 jsDelivr CDN에서 불러오므로, 오프라인에서 렌더하려면 로컬 파일로 바꿔야 한다 (미검증).
- 디스크 사용: `~/.cache/hyperframes` 194MB (Chrome·폰트), npx 캐시, 이 폴더의 `src/` 421MB. 플러그인을 마켓플레이스로 설치하면 `src/`는 지워도 된다.
- 호스팅 MCP는 **베타**이고, 클라우드 렌더는 HeyGen 크레딧을 쓴다.
- Remotion과 비교: 둘 다 Chrome+FFmpeg으로 렌더한다. Remotion은 React 기반에 별도 라이선스(source-available)이고, HyperFrames는 순수 HTML에 Apache 2.0이다.

## 참고 링크

- GitHub: https://github.com/heygen-com/hyperframes
- 문서: https://hyperframes.heygen.com/introduction
- Claude Code 가이드: https://hyperframes.heygen.com/guides/with-claude-code
- CLI 레퍼런스: https://hyperframes.heygen.com/packages/cli
- FAQ (요금·텔레메트리): https://hyperframes.heygen.com/faq
- 호스팅 MCP: https://hyperframes.heygen.com/guides/mcp
- 요금: https://hyperframes.studio/pricing
- 카탈로그: https://hyperframes.heygen.com/catalog/blocks/data-chart
