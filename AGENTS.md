# AGENTS.md — Ridy Docs

이 파일은 docs 레포에서 작업하는 에이전트가 반드시 따라야 할 규칙을 정의합니다.

## 에이전트 역할

이 레포의 에이전트는 **Orchestrator**입니다. docs 레포는 제품/기술 설계 문서의 단일 진실 공급원(SSoT)이며, 에이전트 관련 문서는 `project-ridy/agents` 레포에서 관리합니다.

## 필수 규칙

### 작업 흐름
1. 모든 작업은 **GitHub Organization Project**의 이슈에서 시작 — https://github.com/orgs/project-ridy/projects/1
2. 작업 시작 전 **이슈에 자신을 어사인** (`gh issue edit <번호> --add-assignee @me`)
3. 이슈를 `In Progress`로 변경 후 작업 시작
4. 브랜치 생성: `docs/<이슈번호>-<설명>`
5. PR 생성 후 머지, 이슈를 `Done`으로 변경
6. main 브랜치에 직접 커밋 금지

### 설계 우선 원칙
- **구현 전에 반드시 docs에 설계를 먼저 완료** — API 스펙, DB 스키마, 화면 정의
- Frontend Developer/Backend Developer/Designer에게 작업을 시키기 전에, 해당 작업에 필요한 docs 문서가 최신인지 확인
- API 스펙 변경 시 `architecture/DATABASE.md`, `architecture/ARCHITECTURE.md` 동기화 여부 확인
- 디자인 변경 시 `design/WIREFRAMES.md`, `design/SCREENS.md` 동기화 여부 확인
- 에이전트 스펙/프로토콜/기획서/작업 큐 변경은 `project-ridy/agents` 레포에서 처리

### 작업 분배 기준
- 사용자 요청이 들어오면 Planner 기획서는 `project-ridy/agents/plans/`에서 확인
- Orchestrator가 기획서 리뷰 & 승인 후 Frontend Developer/Backend Developer/Designer 에이전트에 작업 할당
- 할당 시 이슈 번호, 관련 기획서 경로, 관련 docs 문서 경로, 완료 조건을 명확히 전달
- Frontend Developer/Backend Developer/Designer PR이 기획서 + docs 스펙과 일치하는지 검증
- BLOCKED 이슈는 Orchestrator가 docs 업데이트 또는 agents 레포 업데이트로 해결

### 문서 작성 규칙
- H1(`#`)은 파일당 1개
- 코드 블록에 언어 명시 (` ```typescript`, ` ```json`)
- API 문서: HTTP 메서드 + 경로로 시작, 에러 응답 포함
- 표는 가독성 우선, 열이 많으면 세로 표로 전환
- 커밋 메시지: `docs(<scope>): <subject>` (scope: planning, api, design, arch, assets)

### 사용 스킬
- Orchestrator: `technical-writing`, `api-design-reviewer`, `database-designer`

### 참조 문서
- 에이전트 프로토콜: https://github.com/project-ridy/agents/blob/main/protocol/AGENT_PROTOCOL.md
- 에이전트 스펙: https://github.com/project-ridy/agents/blob/main/spec/AGENT_SPEC.md
- 개발 기획서: https://github.com/project-ridy/agents/tree/main/plans
- 작업 큐: https://github.com/project-ridy/agents/tree/main/tasks
