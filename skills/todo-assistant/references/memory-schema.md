# Memory Schema

## Memory Tiers — Local + Global

이 skill은 2-tier 메모리를 운영한다.

| Tier | 위치 | 단위 | 주요 내용 |
|------|------|------|-----------|
| **Local** | `<workspace>/memory/` | 프로젝트 | entries(`index.json`+`active/`), 선택적 `profile.json` 오버라이드 |
| **Global** | `~/.claude/data/todo-assistant/` | OS 사용자 | 전 프로젝트 누적 `profile.json`, `projects.json` 레지스트리, 장기 `archived/` |

`<workspace>` 결정 규칙은 SKILL.md "Workspace 결정" 참조. 코드 디렉토리 입력 → `<cwd>/_workspace/todo-assistant/`, 그 외 → `<cwd>/`.

**Global이 plugin 설치 경로가 아닌 이유** — `~/.claude/plugins/cache/.../skills/todo-assistant/`는 plugin 버전이 붙고 업데이트 시 덮여 사용자 데이터를 잃기 때문. `~/.claude/data/todo-assistant/`는 OS 사용자 단위로 영구 보존되는 데이터 디렉토리.

### Lookup 우선순위

조회는 항상 **Local first → Global fallback → Bootstrap 기본값** 순서.

| 데이터 | Local | Global | 비고 |
|--------|-------|--------|------|
| entries (todos, sub_goals) | ✓ 전용 | ✗ | repo 종속 정보는 절대 Global에 두지 않는다 |
| profile 필드 (`time_factor`, `preferred_hours`, ...) | 우선 | 폴백 | 필드 단위 머지 — Local에 있으면 Local, 없으면 Global |
| `completion_rate` | 프로젝트별 | 전체 가중평균 | 둘 다 유효 |
| `projects.json` (workspace 레지스트리) | ✗ | ✓ 전용 | 다른 프로젝트와의 연결 정보 |

### 디렉토리 트리

**Local (`<workspace>/memory/`):**

```
<workspace>/memory/
├── index.json              # 이 프로젝트의 entry 인덱스
├── profile.json            # (선택) 프로젝트 오버라이드. 없으면 Global이 그대로 쓰임
├── active/                 # 30일 이내, 상세
│   └── YYYY-MM-DD.json
└── archived/               # 월 단위 요약 (1년 초과 시 Global archived/로 승격)
    └── YYYY-MM.json
```

**Global (`~/.claude/data/todo-assistant/`):**

```
~/.claude/data/todo-assistant/
├── profile.json            # 전 프로젝트 누적 사용자 패턴
├── projects.json           # workspace 레지스트리
└── archived/               # 장기 보관 (Local에서 승격된 것)
    └── <workspace-hash>/YYYY-MM.json
```

## index.json

```json
{
  "entries": [
    {"id": "2026-05-10-001", "date": "2026-05-10", "status": "active", "title": "..."}
  ],
  "last_active_id": "2026-05-10-001",
  "last_compaction_at": "2026-05-01"
}
```

## active entry (`active/YYYY-MM-DD.json`)

```json
{
  "id": "2026-05-10-001",
  "created_at": "2026-05-10T09:00:00+09:00",
  "horizon": "both",
  "input_summary": "...",
  "goal": "...",
  "sub_goals": ["..."],
  "todos_daily": [
    {
      "id": "d1",
      "title": "...",
      "target": "configs/example.yaml",
      "done_when": "정의된 항목 25개+",
      "priority": "P0",
      "status": "pending",
      "depends_on": null,
      "estimated_minutes": 60,
      "difficulty": "med",
      "sources": [{"doc_id": "doc1.md", "locator": "L42-50"}],
      "guide": {"start": "...", "ref": "...", "dod": "...", "pitfall": "..."}
    }
  ],
  "todos_weekly": [
    {"id": "w1", "title": "...", "due": "2026-05-16", "priority": "P0", "status": "pending"}
  ],
  "completion_check_done": false,
  "confirmation_skipped": false,
  "linked_files": ["./todos/2026-05-10.md"],
  "integrations": {"notion": null, "google_calendar": null}
}
```

`integrations` 필드: 외부 등록 결과(페이지 ID, 이벤트 ID)를 저장해 다음 호출 시 중복 등록을 방지한다.

## profile.json (스키마는 Local·Global 동일)

```json
{
  "completion_rate": 0.62,
  "time_factor": 1.4,
  "preferred_hours": [9, 10, 14, 15],
  "frequently_postponed": ["문서 작성", "테스트"],
  "sample_count": 12,
  "updated_at": "2026-05-10"
}
```

`sample_count`: 머지 가중치 계산용. 한 entry 종료마다 +1.

### 필드 단위 lookup 식 (Local first → Global fallback)

```
resolve(field):
    v_local = local.profile[field] if local.profile else None
    if v_local is not None and v_local != []:
        return v_local
    v_global = global.profile[field] if global.profile else None
    if v_global is not None and v_global != []:
        return v_global
    return BOOTSTRAP_DEFAULT[field]
```

빈 배열·`null`은 "값 없음"으로 취급해 다음 tier로 넘어간다.

### Global profile 머지 식 (Phase 12 Memory Update에서 사용)

이번 사이클의 Local 갱신값을 Global에 반영할 때 가중 이동평균을 사용한다. `time_factor`처럼 수치형 필드:

```
n_g = global.sample_count
n_l = local.sample_count_delta_this_cycle   # 이번 사이클의 entry 수, 보통 1
v_g_new = (n_g * v_g_old + n_l * v_l_new) / (n_g + n_l)
global.sample_count = n_g + n_l
```

`preferred_hours`·`frequently_postponed` 같은 집합형 필드는 union 후 빈도 기반 상위 N개를 유지(N=Calendar 슬롯 수, 5개 등).

**왜 단순 덮어쓰기가 아닌가** — 한 프로젝트의 비정상 사이클(예: 마감 직전 이슈로 견적 대비 3배 소요)이 Global에 그대로 반영되면 다른 프로젝트 견적까지 왜곡된다. 가중평균은 표본이 쌓일수록 단일 사이클 영향을 줄여준다.

## projects.json (Global 전용)

```json
{
  "workspaces": [
    {
      "path": "/Users/hyungrak/Desktop/skill/todo_recommender",
      "alias": "todo_recommender",
      "last_active_id": "2026-05-10-001",
      "last_used_at": "2026-05-10",
      "first_seen_at": "2026-05-10"
    }
  ]
}
```

skill 호출 시 현재 cwd가 `workspaces[].path`에 있는지 확인하고, 없으면 새 workspace로 등록한다. alias는 디렉토리명 기본 사용, 사용자가 변경 가능.

## Bootstrap (cold start)

skill을 처음 호출하거나 memory 파일이 없을 때 실행. 사용자에게 별도 입력을 받지 않고 결정값 + 빈 상태로 출발한 뒤, **첫 todo 사이클이 끝나면 실측치로 갱신**한다.

### Local Bootstrap

**1) 디렉토리 생성** — `<workspace>` 결정 후:

```
<workspace>/memory/
├── active/        (mkdir)
└── archived/      (mkdir)
```

**2) Local `index.json` 생성** — 빈 entries:

```json
{
  "entries": [],
  "last_active_id": null,
  "last_compaction_at": "<오늘 날짜>"
}
```

`last_compaction_at`을 오늘로 두면 7일 임계 카운터가 즉시 발동하지 않는다.

**3) Local `profile.json`은 기본 생성하지 않는다** — Global이 cross-project로 같은 역할을 하므로, 프로젝트 오버라이드가 필요할 때만 사용자가 수동/자동 생성한다. 필요 시 Global과 동일 스키마 + 모든 필드를 명시적 오버라이드로 덮는다.

### Global Bootstrap

**1) 디렉토리 생성:**

```
~/.claude/data/todo-assistant/
├── archived/      (mkdir)
```

**2) Global `profile.json` 생성** — 기본값:

```json
{
  "completion_rate": null,
  "time_factor": 1.0,
  "preferred_hours": [],
  "frequently_postponed": [],
  "sample_count": 0,
  "updated_at": "<오늘 날짜>"
}
```

| 필드 | 초기값 | 이유 |
|------|--------|------|
| `completion_rate` | `null` | 0.0이면 "완료 0%"로 오해 → 표본 없음을 명시적으로 |
| `time_factor` | `1.0` | 첫 사이클은 보정 없이 견적 그대로 사용 |
| `preferred_hours` | `[]` | Calendar 통합에서 자동 탐지 모드로 폴백 |
| `frequently_postponed` | `[]` | 누적 학습 대상 |
| `sample_count` | `0` | 머지 식의 가중치 누적용 |

**3) Global `projects.json` 생성:**

```json
{ "workspaces": [] }
```

현재 호출의 `<workspace>` 경로를 첫 entry로 추가한다.

### 우회·갱신 규칙

**4) Status Check 우회** — 이전 entry가 없으므로 Phase 2 Status Check Gate를 자동 통과시키고 `confirmation_skipped: false`로 진입한다(스킵이 아니라 해당 사항 없음).

**5) 첫 사이클 종료 후 갱신** — Phase 12 Memory Update에서:
- Local entry 저장 → `index.last_active_id` 갱신
- Global `profile.json`에 가중평균 머지 (위 식)
- Global `projects.json`의 현재 workspace 항목 `last_active_id`·`last_used_at` 갱신

이렇게 하면 cold start 사용자에게 **잘못된 통계값으로 todo가 왜곡되는 일**을 막고, 동시에 **다른 프로젝트로 이동해도 사용자 패턴이 따라간다**.

## 압축 (Compaction)

호출 시점: skill 진입 직후 `last_compaction_at`이 7일 이상 경과했거나 active 항목이 30개 이상일 때.

규칙:
- 30일 초과 entry를 월 단위(`archived/YYYY-MM.json`)로 묶음
- 각 entry → 3~5문장 요약 + 완료된 핵심 산출물 + 미완료 todo 목록만 유지
- 원본 active 파일은 archive 후 삭제
- profile.json도 50KB 초과 시 오래된 데이터부터 LLM 재요약

압축 프롬프트 골격:

```
다음은 한 달간의 todo 기록이다. 다음을 추출해라:
1. 이 기간 사용자가 실제로 진행한 주요 프로젝트 (3개 이내)
2. 완료된 주요 산출물
3. 끝내지 못하고 미루어진 항목 (다음 달로 이월 가능성)
4. 사용자의 작업 패턴 신호
출력은 JSON.
```

## archived entry (`archived/YYYY-MM.json`)

```json
{
  "month": "2026-04",
  "projects": [{"title": "...", "summary": "..."}],
  "completed_artifacts": ["..."],
  "carryover": [{"id": "...", "title": "...", "from": "2026-04-28"}],
  "patterns": ["주중 후반 집중도 하락"]
}
```
