# Changelog

All notable changes to the `todo-assistant` skill are documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.1.2] - 2026-05-10

### Added
- **2-tier 메모리** — Local(`<workspace>/memory/`) + Global(`~/.claude/data/todo-assistant/`). Entry는 Local 전용, profile/projects는 Global 누적.
- `SKILL.md` "Memory Tier — Local + Global" 절: tier 표, lookup 우선순위(Local→Global→Bootstrap), 2-tier 쓰기 규칙.
- `references/memory-schema.md` Global Bootstrap 절차, `projects.json` 스키마, profile 머지 식(가중 이동평균).
- `references/edge-cases.md` Global Tier 6개 시나리오(디렉토리 부재, profile/projects 손상, Local-Global 동시 존재, 권한 거부, 동시 갱신 충돌).
- `README.md` Global tier 산출물 구조 안내.

### Changed
- 기본 정책: Local `profile.json`은 더 이상 자동 생성하지 않는다(Global이 cross-project로 같은 역할). 프로젝트 오버라이드가 필요할 때만 사용자가 명시적으로 생성.
- profile 스키마에 `sample_count` 추가 (가중평균 머지의 가중치).
- Phase 12 Memory Update: Local entry 저장 → Local profile 갱신 → Global profile 가중머지 → Global `projects.json`의 last_active_id/last_used_at 갱신.

## [0.1.1] - 2026-05-10

### Added
- `references/edge-cases.md` — Input·Memory·Integration·Output 카테고리 11+ 시나리오에 대한 감지·복구·알림 매트릭스. 결정 우선순위 표 포함.
- `references/memory-schema.md` — **Bootstrap (cold start)** 섹션. 디렉토리 생성, `index.json`/`profile.json` 초기값, 첫 사이클 후 갱신 규칙 명시.
- `SKILL.md` — **Workspace 결정** 절. 코드 디렉토리 입력 시 `<cwd>/_workspace/todo-assistant/`로 격리, 그 외에는 cwd 직접 사용.
- `SKILL.md` Phase 0~2 — `index.json` 누락/parse 실패/`active/` 부재 3개 시나리오 표.
- `README.md` — **실행 예제** 섹션 (입력 → 흐름 → 출력 발췌 → edge-cases 링크).
- `.gitignore` — `_workspace/` 무시.

### Changed
- `SKILL.md` 출력 경로: `./todos/...` → `<workspace>/todos/...`. memory 경로도 동일하게 `<workspace>/memory/...`.
- `SKILL.md` 디렉토리 레이아웃 — 격리 모드와 cwd 모드 두 트리로 분리.
- Anti-pattern 표에 "사용자 코드 트리 오염" / "비정상 상황" 2행 추가.
- `README.md` 산출물 구조 — 격리 모드 우선 안내.
- `references/memory-schema.md` 디렉토리 트리 — `<workspace>/memory/` 접두 명시.

## [0.1.0] - 2026-05-10

### Added
- 초기 릴리스. SKILL.md + 4개 reference (memory-schema, validation-rules, integrations, output-template).
