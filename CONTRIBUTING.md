# Ridy Docs — 기여 가이드

## 🤖 에이전트 의무 지침

> Orchestrator 에이전트는 이 문서의 규칙을 **반드시** 따릅니다.

### 작업 흐름 (필수)

**모든 작업은 GitHub Organization Project에서 관리됩니다.**
📍 프로젝트 보드: https://github.com/orgs/project-ridy/projects/1

```
1. Project 이슈 할당 → 2. 이슈 In Progress → 3. 브랜치 생성 → 4. 문서 작성/수정 → 5. PR 생성 → 6. 이슈 Done
```

- **main 브랜치에 직접 커밋 금지** — 모든 변경은 PR로
- **Project 이슈 없이 작업 금지** — 모든 작업은 Organization Project의 이슈에서 시작
- **이슈 상태 관리**: 작업 시작 시 `In Progress`, PR 머지 후 `Done`으로 변경

### 에이전트 스킬

Orchestrator 에이전트는 다음 스킬을 로드 후 작업합니다:

| 스킬 | 용도 |
|---|---|
| `technical-writing` | 기술 문서 작성 가이드라인 |
| `api-design-reviewer` | API 스펙 설계 및 리뷰 |
| `database-designer` | DB 스키마 설계 및 ERD |

---

## 📂 문서 구조

```
docs/
├── planning/              # 기획 문서
│   ├── PLANNING.md        # 프로젝트 기획서
│   ├── PERSONAS.md        # 유저 페르소나
│   └── ROADMAP.md         # 로드맵 & 마일스톤
├── api/                   # API 스펙
│   ├── API.md             # 전체 개요
│   ├── AUTH.md            # 인증 API
│   ├── MATCHING.md        # 매칭 API
│   ├── CHAT.md            # 채팅 API
│   └── PAYMENT.md         # 정산/결제 API
├── design/                # 디자인 가이드
│   ├── DESIGN_SYSTEM.md   # 디자인 시스템
│   ├── WIREFRAMES.md      # 와이어프레임
│   └── SCREENS.md         # 화면 정의서
├── architecture/          # 기술 아키텍처
│   ├── ARCHITECTURE.md    # 시스템 아키텍처
│   ├── DATABASE.md        # DB 스키마 & ERD
│   └── INFRA.md           # 인프라 구성도
├── agents/                # 에이전트 작업 지시서
│   ├── AGENT_PROTOCOL.md  # 에이전트 협업 프로토콜
│   ├── DEVELOPER_TASKS.md # 개발 에이전트 작업 큐
│   └── DESIGNER_TASKS.md  # 디자인 에이전트 작업 큐
└── assets/                # 이미지, 파일 에셋
```

---

## 📝 문서 작성 컨벤션

### 마크다운 규칙

- **제목 계층 준수** — H1(`#`)은 파일당 1개, H2(`##`)부터 섹션 분할
- **표는 가독성 우선** — 열이 너무 많으면 세로 표로 전환
- **코드 블록에 언어 명시** — ` ```typescript`, ` ```json`
- **링크는 상대 경로** — 같은 레포 내 문서는 상대 경로로 연결
- **이모지는 제목에만** — 본문에서는 과도한 이모지 자제

### API 문서 규칙

- **모든 엔드포인트는 HTTP 메서드 + 경로로 시작** — `### POST /auth/login`
- **Request/Response는 JSON 코드 블록** 으로 표시
- **필수/선택 필드 명시** — `required`, `optional`
- **에러 응답도 문서화** — 가능한 모든 에러 코드 포함

### 문서 간 일관성

- **API 스펙 변경 시 → DATABASE.md, ARCHITECTURE.md 동기화 확인**
- **디자인 변경 시 → WIREFRAMES.md, SCREENS.md 동기화 확인**
- **새 기능 추가 시 → DEVELOPER_TASKS.md / DESIGNER_TASKS.md에 작업 추가**

---

## 🌿 브랜치 & 커밋 컨벤션

### 브랜치 네이밍

| 타입 | 형식 | 예시 |
|---|---|---|
| 문서 | `docs/<이슈번호>-<설명>` | `docs/5-api-spec-update` |
| 에이전트 | `agents/<이슈번호>-<설명>` | `agents/6-add-task-007` |

### 커밋 메시지

```
docs(<scope>): <subject>
```

| scope | 설명 |
|---|---|
| planning | 기획 문서 |
| api | API 스펙 |
| design | 디자인 가이드 |
| arch | 아키텍처 |
| agents | 에이전트 프로토콜/작업 큐 |

**예시:**
```
docs(api): 매칭 API 응답 필드 추가

- 매칭 검색 결과에 detourMinutes 필드 추가
- DATABASE.md rides 테이블과 동기화

Closes #5
```
