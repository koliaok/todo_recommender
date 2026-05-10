# Edge Cases & Recovery

skill 운용 중 마주칠 수 있는 비정상 상황과 복구 절차. 카테고리별로 분류했고, 각 행은 **시나리오 → 감지 → 복구 → 사용자 알림**의 4요소를 갖는다.

기본 원칙: **자동 복구가 위험하면 진행을 중단하고 사용자에게 묻는다**. 안전 > 자동화.

## Input

| 시나리오 | 감지 | 복구 | 사용자 알림 |
|----------|------|------|-------------|
| 입력 문서가 토큰 한도 초과 (예: 100p PDF) | 추출 텍스트 길이 측정, 임계 50K 토큰 초과 | 섹션 청킹 후 상위 5개 청크만 요약 입력으로 사용. 나머지는 `analysis.unknowns`에 "후속 청크 검토 필요" 추가 | "문서가 큼 — 상위 5개 섹션만 분석에 사용했습니다. 나머지는 다음 사이클에서 다룰지 알려주세요" |
| URL fetch 실패 (404 / timeout / 인증) | WebFetch 에러 응답 | 재시도 1회 → 실패 시 해당 URL 제외하고 진행. analysis에 누락 표시 | "이 URL을 가져오지 못했습니다: <url> ({이유}). 빈 자료로 진행할까요, 다른 자료를 추가할까요?" 응답 대기 |
| 코드 디렉토리가 너무 큼 (>1000 파일) | `find` count | README/entry point/config만 우선 읽고 나머지는 함수 시그니처 grep만 | "디렉토리가 큼 — README와 entry point 위주로 분석했습니다" |
| 자료가 모두 무관 (domain mismatch) | analysis 단계에서 goal 추출 실패 | Ambiguity Gate를 거쳐 사용자에게 목표 직접 입력 요청 | "자료에서 명확한 목표를 추출하지 못했습니다. 무엇을 달성하고 싶은지 한 줄로 알려주세요" |

## Memory — Local Tier (`<workspace>/memory/`)

| 시나리오 | 감지 | 복구 | 사용자 알림 |
|----------|------|------|-------------|
| Local `index.json` 자체가 없음 (cold start) | 파일 미존재 | `references/memory-schema.md` Bootstrap 절차로 신규 생성 | "memory를 새로 시작합니다 (`<workspace>/memory/`)" |
| Local `index.json` JSON parse 실패 (손상) | `JSON.parse` 예외 | `index.json.bak`로 백업 → Bootstrap 재생성. 단, `active/` 안의 entry 파일은 보존 (수동 복구 가능) | "memory 인덱스가 손상되어 백업 후 재생성했습니다. 백업: `<workspace>/memory/index.json.bak`. active entry는 보존되어 있어 수동 복구 가능합니다" |
| Local `active/` 디렉토리 누락 | `stat` 실패 | `mkdir -p` 후 빈 entries로 진행. index의 `last_active_id` → `null` 갱신 | "이전 entry 디렉토리가 없어 빈 상태로 시작합니다" |
| active entry 파일 일부만 손상 | 특정 entry parse 실패 | 해당 entry만 `<id>.bak`로 백업, index.entries에서 제거. 나머지 정상 entry는 그대로 사용 | "Entry {id}가 손상되어 인덱스에서 제외했습니다. 백업 위치: ..." |
| `profile.json`이 50KB 초과 | 파일 크기 검사 | 압축 (Compaction) 절차의 LLM 재요약을 profile에도 적용 | "profile이 커져 요약했습니다" |
| 동일 날짜 재호출 (이미 `<date>.json` 있음) | 파일 존재 | 기존 entry를 읽어 이어서 진행. 새 todo는 추가, 기존 todo는 status 갱신 (덮어쓰지 않음) | "오늘 entry가 이미 있습니다 — 이어서 진행합니다" |

## Memory — Global Tier (`~/.claude/data/todo-assistant/`)

| 시나리오 | 감지 | 복구 | 사용자 알림 |
|----------|------|------|-------------|
| Global 디렉토리가 없음 (최초 사용) | `stat` 실패 | Global Bootstrap 실행 — `mkdir -p ~/.claude/data/todo-assistant/archived` + 빈 `profile.json`/`projects.json` 생성. Local은 그대로 진행 | "Global memory를 처음 만듭니다 — 다른 프로젝트와 사용 패턴이 공유됩니다" |
| Global `profile.json` parse 실패 | JSON 예외 | `profile.json.bak` 백업 → Global Bootstrap 재생성. 이번 사이클은 Local profile만으로 진행 | "전역 profile이 손상되어 재생성했습니다 — 누적 학습이 초기화되었습니다 (백업: `~/.claude/data/todo-assistant/profile.json.bak`)" |
| Global `projects.json` parse 실패 | JSON 예외 | 백업 + 재생성. 현재 workspace를 첫 entry로 추가 | "프로젝트 레지스트리를 재생성했습니다" |
| Local profile vs Global profile 동시에 살아있음 | 둘 다 존재 | Lookup 식에 따라 Local 우선. 머지 시점에 Global로 가중평균 반영 | (정상 동작, 알림 없음) |
| Global 디렉토리 쓰기 권한 없음 | EACCES | Local-only 모드로 폴백(이번 사이클 한정). cross-project 학습은 비활성 | "전역 memory에 쓸 수 없습니다 (`~/.claude/data/todo-assistant/`). Local만 사용합니다 — 다음 호출에서 권한을 확인하세요" |
| Global과 Local의 같은 필드가 충돌 (예: 같은 사용자가 두 머신을 동기화 후 다른 값) | 머지 시 timestamp 비교 | `updated_at`이 더 최근인 쪽 우선, 그 외엔 가중평균 적용 | "전역 profile이 다른 위치에서 갱신된 것 같습니다 — 가중평균으로 머지했습니다" |

## Integration (외부 등록)

| 시나리오 | 감지 | 복구 | 사용자 알림 |
|----------|------|------|-------------|
| Notion MCP가 환경에 등록 안됨 | 도구 호출 시 InputValidationError | 자동 skip 모드. markdown만 저장 | "Notion MCP가 설치되어 있지 않아 markdown 저장으로 진행합니다" |
| Notion 인증 실패 / 권한 없음 | 401/403 응답 | skip 모드 폴백. integration 결과는 `null`로 기록 | "Notion 인증 실패: {원인}. 권한을 확인 후 다시 호출하세요" |
| Notion 다중 등록 중 일부 실패 (예: 5개 중 2개 실패) | 각 페이지 생성 결과 추적 | 성공한 것은 `integrations.notion.pages[]`에 기록. 실패한 todo id를 명시. 사용자가 `다시 등록`을 요청하면 멱등성 규칙으로 미등록만 재시도 | "Notion 등록: 3개 성공, 2개 실패 (D2, D4). 재시도하려면 다시 호출하세요" |
| Calendar 시간 슬롯 충돌 | `create_event`가 conflict 반환하거나 freebusy 검사에서 겹침 발견 | `suggest_time`으로 다음 가용 슬롯 탐색 → 사용자 확인 | "10:00 슬롯이 이미 차있습니다. 14:00로 옮길까요?" |
| Calendar primary 캘린더 ID 없음 | `list_calendars`가 빈 응답 | skip 모드 폴백 | "사용 가능한 캘린더가 없어 markdown 저장만 진행합니다" |

## Output

| 시나리오 | 감지 | 복구 | 사용자 알림 |
|----------|------|------|-------------|
| `<workspace>/todos/`가 `.gitignore`에 없음 (코드베이스 사용 시) | 코드 디렉토리 입력인데 `_workspace/` 모드가 아닌 cwd 모드 | 사용자에게 격리 모드(`_workspace/todo-assistant/`) 사용 권장 알림 | "현재 모드는 cwd 직접 저장입니다. 코드 repo를 오염시키지 않으려면 `_workspace/todo-assistant/` 격리 모드 사용을 권장합니다" |
| 파일 쓰기 권한 거부 | `Write` ENOENT/EACCES | 진행 중단. 사용자에게 다른 경로 또는 권한 수정 요청 | "쓰기 권한이 없습니다: `<workspace>`. 권한을 부여하거나 다른 디렉토리로 호출해주세요" |
| 디스크 공간 부족 | Write ENOSPC | 진행 중단 | "디스크 공간 부족 — 정리 후 다시 시도해주세요" |
| 사용자가 User Confirm Gate에서 응답 없음 (타임아웃) | 응답 부재 | **저장하지 않음**. todo 초안은 메모리에만 남기고 다음 호출에서 사용자가 이어갈 수 있도록 안내 | "초안을 저장하지 않았습니다. 다시 호출해 검토하세요" |

## 카테고리 매트릭스 — 우선순위

| 카테고리 | 복구 가능성 | 사용자 차단 | 우선순위 |
|----------|-------------|-------------|----------|
| Memory 손상 | 높음 (백업 + 재생성) | 낮음 | 자동 복구 |
| Input 일부 실패 | 중간 (부분 진행) | 중간 | 알림 후 진행 |
| Integration 실패 | 중간 (skip 폴백) | 낮음 | 자동 폴백 |
| Output 권한/공간 | 낮음 | 높음 | **진행 중단** |
| User 응답 없음 | n/a | 높음 | **진행 중단** |

위 매트릭스가 의사결정의 기본 가이드. 모호하면 항상 안전 쪽(중단 + 사용자에게 알림)을 택한다.
