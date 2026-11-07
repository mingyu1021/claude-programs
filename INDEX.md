# 도구 목록

상태: ✅ 연동·테스트 완료 · 🟡 정리만 · 🔄 진행중 · ❌ 보류
각 세션은 **자기 행 한 줄만** 추가/수정한다.
어디에(전역/프로젝트) 무엇을 설치할지는 → [install.md](install.md) (메인 세션이 관리)

| 도구 | 유형 | 한 줄 요약 | 상태 | 전역 등록 | 담당 세션 | 갱신일 |
|---|---|---|---|---|---|---|
| [Eraser.io](Eraser.io/README.md) | MCP (HTTP) | 텍스트·자연어로 아키텍처/시퀀스/ERD 다이어그램 생성 | ✅ | user scope | claude-programs-83 | 2026-11-08 |
| [ponytail](ponytail/README.md) | Plugin (Skill+훅) | "게으른 시니어 개발자" 규칙 주입 — 과잉 설계 억제, stdlib 우선, /ponytail-review·audit | ✅ | user scope (`ponytail@ponytail`, 기본 off) | (작업 세션) | 2026-11-08 |
| [draw.io](draw.io/README.md) | Plugin(Skill) + MCP (HTTP/npm) | 무료·무인증. Claude가 Mermaid/XML로 `.drawio` 파일·편집 링크 생성 | ✅ 플러그인 설치·테스트 완료 (npm 서버 미테스트) | user scope (플러그인 `drawio@drawio`) | claude-programs-24 | 2026-11-08 |
| [security-audit-skill](security-audit-skill/README.md) | Skill | Cloudflare 제작. 하위 에이전트로 정찰→헌팅→독립 검증→findings.json→리포트까지 소스코드 보안 감사 | ✅ | user scope (~/.claude/skills) | claude-programs-17 | 2026-11-08 |
| [ECC](ECC/README.md) | Plugin | 에이전트 71·스킬 314·hooks로 계획→TDD→리뷰→학습 흐름을 강제하는 대형 플러그인 (ecc@ecc) | ✅ | — (설치 후 삭제: 세션당 +13k 토큰) | (작업 세션) | 2026-11-08 |
| [impeccable](impeccable/README.md) | Plugin (Skill+훅+에이전트) | 24개 디자인 명령(/impeccable polish·critique·audit…)과 AI slop 탐지기로 프론트엔드 디자인 품질 향상 | 🟡 Skill 지침만 테스트 ✅ (hooks·엔진은 실행 승인 대기) | — | claude-programs-24 | 2026-11-08 |
| [HyperFrames](HyperFrames/README.md) | Plugin (Skill 9) + CLI (+호스팅 MCP 베타) | HTML+GSAP로 MP4 영상 제작 (HeyGen, 무료 로컬 렌더) — Claude가 작성·검사·렌더까지 | ✅ | — (테스트용만) | (작업 세션) | 2026-11-08 |
