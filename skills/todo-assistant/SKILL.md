---
name: todo-assistant
description: 사용자가 제공한 문서·URL·코드를 분석해 일일/주간 Todo list를 자동 생성하고, 이전 작업 진행 상황을 추적하며, 각 작업의 수행 가이드(시작점·참조·완료 기준·막힐 때)를 제공한다. 최종 todo는 Notion MCP나 Google Calendar로 등록하거나 markdown 파일로 저장한다. "오늘/이번 주 할 일 정리해줘", "뭘 해야 할지 모르겠다", "다음 단계가 뭐야", "이 자료 보고 작업 분해해줘", "todo 만들어줘", "할 일 추천", "작업 계획 세워줘", "todo 등록해줘", "캘린더에 일정 잡아줘" 같은 요청에 반드시 사용. 자료(문서·링크·코드)와 함께 작업 분해 요청이 오면 무조건 트리거.
---

# Todo Assistant

스스로 작업을 분해하기 어려운 사용자에게, 자료를 받아 **하루치 + 한 주치 Todo**를 만들고, 진행 상황을 추적하며, 결과를 외부 도구(Notion·Calendar)에 등록한다.

핵심 가치: **"자료를 던져주면, 사용자는 첫 항목부터 따라가기만 하면 된다."**

## 워크플로우

```
[Skill 호출]
  ↓
[0] Memory Compaction Gate ── 7일↑ or active≥30 → 압축
  ↓
[1] Memory Load            ── index.json + last_active_id 로드
  ↓
[2] Status Check Gate      ── 이전 미완료 todo 컨펌, 응답 없으면 정지
  ↓
[3] Input Ingestion        ── 문서/URL/코드 정규화·요약
  ↓
[4] Analysis               ── goal/sub_goals/unknowns/risks 추출
  ↓
[5] Ambiguity Gate         ── unknowns 있으면 사용자에게 질문, 정지
  ↓
[6] Decomposition          ── 주간(5~7) → 일일(3~5)로 분해
  ↓
[7] Generation             ── todo + 가이드(시작점/참조/DoD/함정)
  ↓
[8] Validation             ── 추상성/근거/개수·시간·난이도 검사 (재생성 ≤2)
  ↓
[9] User Confirm Gate      ── 초안 보여주기 → 승인 후 저장
  ↓
[10] Output                ── ./todos/YYYY-MM-DD.md 저장
  ↓
[11] Integration (선택)    ── Notion MCP / Google Calendar / md만
  ↓
[12] Memory Update         ── 신규 entry 기록, profile 갱신
```

## Phase 0~2: 진입과 컨텍스트 회복

**Workspace 결정.** 모든 phase의 파일 경로 베이스는 다음 규칙으로 결정한다:

- 입력에 **코드 디렉토리**가 포함되면 → `<cwd>/_workspace/todo-assistant/` (사용자 소스 트리를 오염시키지 않기 위해 격리)
- 그 외(문서/URL/자유 텍스트만) → `<cwd>/`

이 결정값을 `<workspace>`로 표기한다. 이하 모든 경로는 `<workspace>` 기준이다. 디렉토리가 없으면 첫 호출에서 자동 생성한다(자세한 cold start 절차는 `references/memory-schema.md` Bootstrap 섹션 참조).

**Memory Tier — Local + Global.** 이 skill은 **2-tier 메모리**를 운영한다. 자세한 스키마와 lookup 규칙은 `references/memory-schema.md` "Memory Tiers" 절을 참조한다.

| Tier | 위치 | 담는 것 |
|------|------|---------|
| **Local** (프로젝트 단위) | `<workspace>/memory/` | `index.json`, `active/`, `archived/`, (선택) `profile.json` 오버라이드 |
| **Global** (사용자 단위, 모든 프로젝트 공유) | `~/.claude/data/todo-assistant/` | `profile.json` (cross-project 패턴), `projects.json` (workspace 레지스트리), `archived/` (장기 보관) |

> 📌 **왜 글로벌이 `~/.claude/data/todo-assistant/`인가?** plugin 설치 경로(`~/.claude/plugins/cache/.../skills/todo-assistant/`)는 버전이 붙고 업데이트 시 덮어쓰여 사용자 데이터를 잃는다. 따라서 stable한 user data 디렉토리를 사용한다. 이 경로는 OS 사용자 단위로 영구 보존된다.

**Lookup 우선순위 — Local first, Global fallback:**
1. **Entry 데이터(todos, sub_goals, integrations 등)** → Local 전용. Global에는 entry를 두지 않는다(개인정보·repo 종속 정보 누설 방지).
2. **Profile 신호 필드별 조회**(예: `time_factor`, `preferred_hours`):
   - 먼저 `<workspace>/memory/profile.json`에서 해당 필드를 찾고,
   - 없거나 `null`이면 `~/.claude/data/todo-assistant/profile.json`에서 가져오고,
   - 둘 다 없으면 Bootstrap 기본값 사용.
3. **Workspace 자체에 대한 메타 정보**(이전 사용 시점·alias 등) → Global `projects.json`.

**Memory Compaction Gate.** skill 진입 직후 `<workspace>/memory/index.json`을 읽어, `last_compaction_at`이 7일 이상 경과했거나 `active/` 항목이 30개 이상이면 압축을 실행한다. Local archived는 1년 초과 시 Global `archived/`로 승격(이관)할 수 있다. 압축이 끝나야 다음으로 진행한다.

**Memory Load.** Local `index.json` → `last_active_id` → 해당 entry 파일을 읽는다. 그 다음 Global `profile.json`을 읽어 빠진 필드를 보충한다(위 lookup 우선순위 적용). 파일이 손상됐거나 없으면 다음 시나리오로 분기한다:

| 시나리오 | 절차 |
|----------|------|
| Local `index.json` 자체가 없음 | `references/memory-schema.md` Bootstrap 절차로 신규 생성 후 사용자에게 "memory를 새로 시작합니다" 알림 |
| Local `index.json` 존재하지만 JSON parse 실패 | `index.json.bak`로 백업 → Bootstrap 재생성 → 사용자에게 손상 사실과 백업 위치 알림 |
| Local `active/` 디렉토리 누락 | `mkdir -p`로 생성 → 빈 entry로 진행 (index의 `entries`도 비우고 진행) |
| Global `~/.claude/data/todo-assistant/` 자체가 없음 | Global Bootstrap 실행(빈 `profile.json`+`projects.json` 생성). Local은 그대로 진행 |
| Global `profile.json` 손상 | `profile.json.bak`로 백업 → Global Bootstrap 재생성. Local profile만으로 임시 진행 |

세부 복구 시나리오와 다른 edge case는 `references/edge-cases.md` 참조.

**Status Check Gate.** 이전 entry의 `status: pending` todo가 있으면 다음 형식으로 사용자에게 묻는다:

```
지난 todo (2026-05-08) 중 다음이 미완료 상태입니다.
각각 어떤 상태인가요?
  [ ] D2. 검증 스크립트 실행   → (완료 / 진행 중 / 보류 / 폐기)
  [ ] D3. 외부 인증 디버깅      → ...
```

응답을 받기 전에는 다음 단계로 진입하지 않는다. 사용자가 명시적으로 "skip"하면 entry에 `confirmation_skipped: true`를 기록하고 진행한다. 미완료로 남은 항목은 오늘 todo의 "이전 미완료 이월" 섹션에 자동 포함한다.

## Phase 3~5: 입력과 분석

**Input Ingestion.** 입력 타입별 처리:

| 타입 | 처리 |
|------|------|
| 문서 (md/pdf/docx) | 텍스트 추출 → 섹션 청킹 → 요약 |
| URL | WebFetch → 본문 → 요약 |
| 코드 디렉토리 | README/entry point/config 우선 → 함수 시그니처 추출 |
| 자유 텍스트 | 그대로 컨텍스트에 포함 |

정규화 산출물:

```json
{
  "domain_hint": "...",
  "artifacts": [{"type": "doc", "title": "...", "summary": "...", "key_points": [...]}],
  "constraints": ["마감: ...", "환경: ..."],
  "open_questions": ["..."]
}
```

**Analysis.** 다음 스키마로 정리:

```
goal:             최종 산출물 1~2문장
sub_goals:        3~5개 마일스톤
unknowns:         사용자에게 물어야 할 질문
risks:            예상 블로커
estimated_effort: 총 예상 시간
```

**Ambiguity Gate.** `unknowns`가 비어있지 않으면 todo 생성을 멈추고 사용자에게 먼저 질문한다. 모호한 상태로 만든 todo는 잘못된 길을 굳혀버린다.

## Phase 6~8: 분해·생성·검증

**Decomposition 원칙.**

- 일일 todo: 3~5개, 우선순위 순(D1=가장 중요), 각 1.5h 이내, P0/P1/P2 태그, 의존성은 `depends_on` 명시
- 주간 todo: 5~7개, 마일스톤 단위, 우선순위 순(W1=가장 중요), `due` 날짜 + priority

**Generation — 작업 가이드.** 각 todo에 1~2문장씩 4개 필드:
1. **시작점** — 어디서부터 손대면 되는지 (파일/명령어/링크)
2. **참조** — 관련 자료
3. **완료 기준 (DoD)** — 무엇이 보이면 끝인지
4. **막힐 때** — 흔한 함정 1개

이 부분이 *작업 분해가 어려운 사용자*에게 가장 큰 가치를 준다.

**Validation 파이프라인.** 검증 규칙과 재생성 기준은 `references/validation-rules.md` 참조. 핵심:
- 추상 동사 blocklist (리뷰하기/정리하기/공부하기/확인하기/살펴보기) 매칭 시 실패
- 모든 todo는 `target`(대상)과 `done_when`(완료 조건) 필수
- `sources: [{doc_id, locator}]` 무결성 검사 — 환각 방지
- 일일 ≤5개, 총 ≤6h, hard ≤2개 — 초과는 weekly로 자동 이동
- 실패 시 최대 2회 재생성, 그래도 실패하면 해당 항목만 사용자에게 질문

## Phase 9~11: 컨펌·출력·통합

**User Confirm Gate.** todo 초안을 사용자에게 보여준다. 수정/승인을 받기 전에는 파일에 쓰지 않는다. 바로 저장하면 잘못된 todo가 그대로 굳는다.

**Output.** 승인된 todo를 `<workspace>/todos/YYYY-MM-DD.md`에 저장한다. 템플릿은 `references/output-template.md` 참조.

**Integration Gate (선택).** 출력 직후 사용자에게 묻는다:

```
이 todo를 어디에 등록할까요?
  1) Notion (MCP)        — 데이터베이스/페이지에 todo 항목 생성
  2) Google Calendar     — 일정으로 등록 (due/예상 시간 기반)
  3) 둘 다
  4) markdown 파일만 (skip)
```

각 통합의 호출 방법(MCP 도구·필드 매핑·실패 시 폴백)은 `references/integrations.md` 참조. **MCP가 사용 불가능하거나 사용자가 4번을 선택하면 markdown 저장만으로 종료한다.**

## Phase 12: 기록

**Memory Update — 2-tier 쓰기.** 새 entry를 `<workspace>/memory/active/YYYY-MM-DD.json`에 쓰고 Local `index.json`을 갱신한다. 그 다음 다음 두 단계를 수행:

1. **Local profile 갱신** — 이번 사이클의 완료율·실제 소요 시간 등을 Local `<workspace>/memory/profile.json`에 가중 이동평균으로 반영(프로젝트 단위 학습).
2. **Global profile 머지** — Local의 갱신값을 `~/.claude/data/todo-assistant/profile.json`에 가중 이동평균으로 머지(사용자 단위 학습). 동시에 Global `projects.json`의 해당 workspace 항목에 `last_active_id`·`last_used_at`을 갱신한다.

이렇게 하면 동일 사용자가 다른 프로젝트로 이동해도 시간 추정·선호 시간대 학습이 이어진다. 자세한 머지 식과 가중치는 `references/memory-schema.md` 참조.

## 디렉토리 레이아웃

### Local Tier (`<workspace>` — 프로젝트 단위)

**코드 디렉토리 입력일 때 (격리 모드)** — 사용자 소스 트리를 오염시키지 않기 위해 `_workspace/todo-assistant/` 하위:

```
<user-cwd>/
└── _workspace/
    └── todo-assistant/
        ├── todos/
        │   └── YYYY-MM-DD.md
        └── memory/
            ├── index.json
            ├── profile.json     # (선택) 프로젝트 오버라이드
            ├── active/          # 30일 이내, 상세
            └── archived/        # 월 단위 요약
```

`_workspace/`는 사용자가 `.gitignore`에 추가하는 것을 권장한다.

**그 외 입력 (문서/URL/자유 텍스트)** — 사용자 cwd 직접:

```
<user-cwd>/
├── todos/YYYY-MM-DD.md
└── memory/...
```

### Global Tier (`~/.claude/data/todo-assistant/` — 사용자 단위, 모든 프로젝트 공유)

```
~/.claude/data/todo-assistant/
├── profile.json         # 전 프로젝트 누적 패턴 (time_factor, preferred_hours, ...)
├── projects.json        # workspace 레지스트리 (path → last_active_id, last_used_at)
└── archived/            # Local archived가 1년 초과 시 승격된 장기 보관
```

이 위치는 plugin 설치 경로(`~/.claude/plugins/...`, 버전·캐시 경로)와 다르다. 업데이트로 사용자 데이터가 사라지지 않도록 stable한 별도 위치를 사용한다. skill 자체 source 디렉토리에는 절대 쓰지 않는다.

### Lookup 순서

1. Entry 데이터 → Local만
2. Profile 필드 → Local → Global → Bootstrap 기본값
3. 프로젝트 메타 → Global `projects.json`

## Anti-pattern 방어 요약

| 함정 | 방어 |
|------|------|
| 추상 todo | Validator가 `target`/`done_when` 강제 + 재생성 |
| Memory 무한 누적 | 7일/30개 임계 → 자동 압축 |
| 응답 없이 자동 진행 | Status·Ambiguity·Confirm Gate 3단 |
| 환각 인용 | `sources` 필수 + Validator 무결성 검사 |
| 사용자 압도 | 일일 ≤5, 총 ≤6h, hard ≤2 |
| 사용자 코드 트리 오염 | 코드 입력 시 `_workspace/todo-assistant/`로 격리 |
| 비정상 상황(손상·인증실패·권한·MCP 부재) | `references/edge-cases.md` 매트릭스 — 안전 > 자동화 |

## 트리거 — 반드시 사용

- "오늘 할 일 / 이번 주 할 일 정리해줘"
- "이 문서/링크/코드 보고 뭘 해야 할지 알려줘"
- "다음 단계가 뭐야"
- "todo 만들어줘", "작업 분해해줘"
- 자료(파일/URL/디렉토리)와 함께 "어디서부터 시작해야 할지 모르겠다"
- "todo Notion에 등록", "캘린더에 일정 잡아줘"

## 트리거하지 말 것

- 단일 한 줄 작업 요청 ("이 함수 고쳐줘") — 분해할 필요 없음
- 코드 리뷰 / 디버깅 / 리팩터링 단독 요청
- 단순 질의응답
