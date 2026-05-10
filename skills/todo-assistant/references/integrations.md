# 외부 등록 통합 (Notion MCP / Google Calendar)

User Confirm 직후, markdown 저장 다음 단계에서 사용자에게 묻는다:

```
이 todo를 어디에 등록할까요?
  1) Notion (MCP)
  2) Google Calendar
  3) 둘 다
  4) skip — markdown 파일만
```

선택에 따라 아래 절차를 실행한다. **MCP 도구가 환경에 없으면 자동으로 4번(skip)으로 폴백하고 사용자에게 알린다.**

---

## 1. Notion MCP

도구: `mcp__claude_ai_Notion__notion-create-pages`, `mcp__claude_ai_Notion__notion-search`, `mcp__claude_ai_Notion__notion-create-database`

### 절차

**Step 1. 등록 대상 결정**

사용자에게 묻는다:

```
어디에 등록할까요?
  a) 기존 데이터베이스에 todo 항목으로 추가  → DB URL 입력 받기
  b) 기존 페이지의 자식으로 todo 페이지 생성 → 페이지 URL 입력 받기
  c) 새 데이터베이스 생성                    → 부모 페이지 URL 입력 받기
```

URL/ID는 `notion-fetch`로 유효성을 먼저 검사한다. 실패 시 사용자에게 재입력 요청.

**Step 2. (c 선택 시) DB 생성**

`notion-create-database` 호출. 권장 스키마:

| 속성명 | 타입 | 비고 |
|--------|------|------|
| Title | title | todo title |
| Status | status | pending / in_progress / done / dropped |
| Priority | select | P0 / P1 / P2 |
| Horizon | select | daily / weekly |
| Due | date | weekly의 due, daily는 생성일 |
| Estimated | number | 분 단위 |
| Source | url | 입력 자료 링크 |

**Step 3. 항목 생성**

`notion-create-pages`로 daily + weekly todo 각각 한 페이지씩 생성. 페이지 본문에는 SKILL.md의 가이드(시작점·참조·DoD·막힐 때)와 sources를 포함한다.

**Step 4. 결과 기록**

각 항목의 page URL을 active entry의 `integrations.notion.pages[]`에 저장. 출력 markdown에 `## 🔗 등록 결과`를 추가하고 링크를 나열.

### 멱등성

같은 todo id에 대해 이미 `integrations.notion`에 기록이 있으면 신규 생성 대신 `notion-update-page`로 갱신한다. 중복 생성을 방지한다.

---

## 2. Google Calendar

도구: `mcp__claude_ai_Google_Calendar__list_calendars`, `create_event`, `suggest_time`

### 절차

**Step 1. 캘린더 선택**

`list_calendars`로 사용 가능한 캘린더 조회. 사용자에게 어느 캘린더에 등록할지 묻는다(기본: primary).

**Step 2. 시간 슬롯 결정**

각 todo에 대해:
- **daily todo**: 오늘 날짜에 `estimated_minutes` 길이의 슬롯. profile.json의 `preferred_hours`를 우선 시도.
- **weekly todo**: `due` 날짜 09:00~10:00 (또는 due 직전 영업일).

사용자가 빈 시간을 자동 탐지하길 원하면 `suggest_time` 사용.

**Step 3. 이벤트 생성**

`create_event` 호출. 필드 매핑:

| Calendar 필드 | todo 필드 |
|--------------|-----------|
| summary | `[P0] D1. {title}` |
| description | 가이드 4필드 + sources URL |
| start/end | 슬롯 시간 |
| reminders | 시작 10분 전 |

**Step 4. 결과 기록**

이벤트 ID와 htmlLink를 `integrations.google_calendar.events[]`에 저장.

### 멱등성

기존 이벤트 ID가 있으면 `update_event`로 갱신. 사용자가 todo 일부를 취소하면 `delete_event`로 정리한다.

---

## 3. 둘 다

위 두 절차를 순서대로 실행한다(Notion 먼저, Calendar 나중). 한쪽이 실패해도 나머지는 진행하고, 결과 보고에 실패한 쪽을 명시한다.

---

## 4. Skip — markdown만

`./todos/YYYY-MM-DD.md` 저장으로 종료. active entry의 `integrations` 필드는 둘 다 `null`로 둔다.

---

## 폴백 규칙

다음 경우 자동으로 skip 모드:
- 해당 MCP 서버가 환경에 등록되어 있지 않음 (도구 호출 시 InputValidationError)
- 인증 실패 / 권한 없음
- 사용자가 URL 재입력에 응답하지 않음

폴백 시 사용자에게 알린다: "Notion MCP를 사용할 수 없어 markdown 저장으로 진행합니다. 나중에 등록을 원하면 다시 호출하세요."
