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
4. 초안 컨펌 후 `<workspace>/todos/YYYY-MM-DD.md` 저장 (코드베이스 입력은 `_workspace/todo-assistant/` 하위에 격리)
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

**코드 디렉토리 입력 (격리 모드)** — 사용자 소스 트리를 오염시키지 않기 위해 `_workspace/todo-assistant/` 하위에 격리:

```
<cwd>/
└── _workspace/
    └── todo-assistant/
        ├── todos/YYYY-MM-DD.md       # 출력 todo
        └── memory/
            ├── index.json
            ├── profile.json          # 사용자 작업 패턴
            ├── active/  (30일 이내)
            └── archived/ (월 단위 요약)
```

`_workspace/`는 `.gitignore`에 추가 권장.

**그 외 입력 (문서/URL/자유 텍스트)** — cwd 직접:

```
<cwd>/
├── todos/YYYY-MM-DD.md
└── memory/...
```

**Global tier (모든 프로젝트 공유)** — OS 사용자 단위 영구 보존:

```
~/.claude/data/todo-assistant/
├── profile.json         # 전 프로젝트 누적 사용자 패턴 (time_factor, preferred_hours, ...)
├── projects.json        # workspace 레지스트리
└── archived/            # 1년 초과 시 Local에서 승격된 장기 보관
```

skill은 항상 **Local 먼저 → Global 폴백** 순으로 조회한다. entry 데이터(todos)는 Local 전용, 사용자 패턴은 Global 누적 — 다른 프로젝트로 이동해도 시간 추정과 선호 시간대가 따라간다. 자세한 머지 식·lookup 규칙은 [memory-schema.md](skills/todo-assistant/references/memory-schema.md) 참조.

## 실행 예제

**입력 (사용자):**

```
"./src 보고 오늘 할 일 정리해줘 — auth 미들웨어 리팩터링이 목표야"
```

**skill 동작 흐름:**

1. Status Check Gate — 이전 entry 미완료 todo 확인 (없으면 cold start, [Bootstrap](skills/todo-assistant/references/memory-schema.md) 자동 실행)
2. `src/` 분석 → goal/sub_goals/unknowns 추출
3. Ambiguity Gate — 모호한 점 있으면 질문, 응답 받기 전 정지
4. 일일 todo 3~5개 생성 → User Confirm
5. 승인 후 `_workspace/todo-assistant/todos/2026-05-10.md` 저장 (코드 입력이므로 격리 모드)

**출력 발췌 (`_workspace/todo-assistant/todos/2026-05-10.md`):**

```markdown
# Todo — 2026-05-10 (일)

> **목표**: auth 미들웨어 리팩터링 v1

## 🌞 오늘 할 일

### [ ] D1. `[P0]` `src/middleware/auth.ts` 토큰 검증 분리
- **시작점**: `auth.ts` L24-58 verifyToken 추출 → `src/middleware/jwt.ts`
- **참조**: 기존 unit test `tests/auth.test.ts`
- **완료 기준**: 신규 파일 + 기존 테스트 통과 + import 경로 갱신
- **막힐 때**: 순환 import 발생 시 타입을 별도 `types/auth.d.ts`로
- **예상 시간**: 60분
- 📎 근거: src/middleware/auth.ts L24-58
```

비정상 상황(memory 손상, MCP 부재, 입력 토큰 한도 초과 등)은 [edge-cases.md](skills/todo-assistant/references/edge-cases.md) 매트릭스에 따라 자동 복구·폴백·중단 중 안전한 쪽을 택한다.

## 주요 특징

- **3단 Gate** — Status Check / Ambiguity / User Confirm으로 잘못된 todo가 굳는 것을 방지
- **Validation 파이프라인** — 추상 todo blocklist, 근거 무결성, 개수·시간·난이도 한도
- **Memory 자동 압축** — 30일 초과는 월 요약으로 archive, 7일마다 자동 실행
- **Profile 학습** — 완료율, 시간 추정 보정 계수, 선호 시간대를 기록해 개인화

## 라이선스

Apache-2.0
