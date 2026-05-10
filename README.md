# todo-assistant

Claude Code skill — 자료(문서·URL·코드)를 던져주면 **일일 + 주간 Todo list**를 자동 생성하고, 진행 상황을 추적하며, 각 작업의 수행 가이드를 제공한다. 결과는 **Notion (MCP) / Google Calendar / markdown 파일** 중 선택해서 등록한다.

> 핵심 가치: *"자료를 던져주면, 사용자는 그저 첫 항목부터 따라가기만 하면 된다."*

## 설치

### 1. Plugin Marketplace로 설치 (권장)

```bash
/plugin marketplace add koliaok/todo_recommender
/plugin install todo-assistant@todo-recommender-marketplace
```

### 2. Skill 직접 설치

```bash
git clone https://github.com/koliaok/todo_recommender.git
cp -r todo_recommender/skills/todo-assistant ~/.claude/skills/todo-assistant
```

## 사용

자료와 함께 자연어로 호출:

```
"이 PRD 문서랑 코드 보고 오늘 할 일 정리해줘: ./docs/prd.md, ./src/"
"https://example.com/spec — 다음 주 작업 분해해줘"
"어디서부터 시작해야 할지 모르겠어. 이 자료들 참고해서 todo 만들어줘"
```

skill이 자동으로:
1. 이전 todo 진행 상황 컨펌
2. 입력 자료 분석 (모호한 부분 있으면 질문)
3. 일일(3~5) + 주간(5~7) todo 생성 — 우선순위·시간·근거 포함
4. 초안 컨펌 후 `./todos/YYYY-MM-DD.md` 저장
5. **선택**: Notion DB / Google Calendar에 등록

## 외부 등록 옵션

User Confirm 직후 다음 중 선택:

| 선택 | 동작 |
|------|------|
| Notion (MCP) | 데이터베이스 항목 또는 페이지 자식으로 todo 등록 |
| Google Calendar | 예상 시간/마감 기반 이벤트 생성 |
| 둘 다 | 위 두 가지 모두 |
| skip | markdown만 저장 |

MCP 도구가 환경에 없으면 자동으로 markdown 저장으로 폴백한다.

## 산출물 구조

skill을 호출한 작업 디렉토리 하위에 생성:

```
<cwd>/
├── todos/YYYY-MM-DD.md          # 출력 todo
└── memory/
    ├── index.json
    ├── profile.json             # 사용자 작업 패턴
    ├── active/  (30일 이내)
    └── archived/ (월 단위 요약)
```

## 주요 특징

- **3단 Gate** — Status Check / Ambiguity / User Confirm으로 잘못된 todo가 굳는 것을 방지
- **Validation 파이프라인** — 추상 todo blocklist, 근거 무결성, 개수·시간·난이도 한도
- **Memory 자동 압축** — 30일 초과는 월 요약으로 archive, 7일마다 자동 실행
- **Profile 학습** — 완료율, 시간 추정 보정 계수, 선호 시간대를 기록해 개인화

## 라이선스

Apache-2.0
