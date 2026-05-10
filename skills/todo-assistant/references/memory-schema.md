# Memory Schema

## 디렉토리

```
memory/
├── index.json              # 인덱스 (빠른 조회)
├── profile.json            # 사용자 패턴 (완료율, 시간 추정 보정 등)
├── active/                 # 30일 이내, 상세
│   └── YYYY-MM-DD.json
└── archived/               # 월 단위 요약
    └── YYYY-MM.json
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

## profile.json

```json
{
  "completion_rate": 0.62,
  "time_factor": 1.4,
  "preferred_hours": [9, 10, 14, 15],
  "frequently_postponed": ["문서 작성", "테스트"],
  "updated_at": "2026-05-10"
}
```

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
