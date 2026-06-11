# Ridy — 에이전트 협업 프로토콜

## 개요

Ridy 프로젝트는 4개의 에이전트가 협업합니다:

| 에이전트 | 역할 | 작업 레포 |
|---|---|---|
| **Orchestrator** | 전체 관리, 작업 분배, 품질 관리 | docs |
| **Planner** | 사용자 요청 분석, 개발 기획서 작성 | docs (plans/) |
| **Developer** | 프론트엔드/백엔드 구현 | frontend, backend |
| **Designer** | UI/UX 디자인, 에셋 제작 | frontend (design 파일) |

## GitHub Project

**프로젝트 보드**: https://github.com/orgs/project-ridy/projects/1

모든 작업은 GitHub Project를 기준으로 관리됩니다.

### 커스텀 필드

| 필드 | 옵션 | 설명 |
|---|---|---|
| **Phase** | Phase 1~5 | 개발 단계 |
| **레포** | docs, frontend, backend | 작업 대상 레포 |
| **작업 유형** | 기능, 버그 수정, 테스트, 디자인, 문서, 인프라 | 작업 종류 |
| **우선순위** | P0-Critical, P1-High, P2-Medium, P3-Low | 우선순위 |

### Phase별 이슈 현황

| Phase | 이슈 수 | 주요 작업 |
|---|---|---|
| Phase 1: 기획 & 설계 | 4 | API 상세 설계, DB 상세 설계, 와이어프레임, 기술 스택 |
| Phase 2: MVP 개발 | 16 | 프로젝트 셋업, 인증/매칭/채팅 API + 화면, 테스트 |
| Phase 3: 정산 + 평점 | 5 | 정산/결제 API, 평점/리뷰, 화면, 테스트 |
| Phase 4: 클로즈드 베타 | 3 | 초대 코드, 모니터링, 마이페이지 |
| Phase 5: 오픈 베타 | 3 | B2B 기획, 랜딩 페이지, 친환경 대시보드 |

## 작업 흐름

```
1. 사용자가 기능 요청 → Orchestrator가 Planner에게 기획 지시
2. Planner가 요청 분석 & 개발 기획서 작성 (docs/plans/)
3. Orchestrator가 기획서 리뷰 & 승인
4. Orchestrator가 기획서 기반으로 GitHub Project에 이슈 생성
5. 이슈에 자신을 어사인 (`gh issue edit <번호> --add-assignee @me`)
6. 이슈 상태를 "In Progress"로 변경
7. Developer/Designer 에이전트가 기획서 + docs 문서를 읽고 작업 수행
8. 브랜치 생성 → TDD 사이클 → PR 생성
9. PR 머지 후 이슈를 "Done"으로 변경
10. 기능 작업인 경우 후속 테스트 이슈를 "In Progress"로 변경
11. Orchestrator가 결과 확인 후 다음 작업 판단
```

### 작업 우선순위 규칙

1. **같은 Phase 내에서는 P0 > P1 > P2 > P3 순서**
2. **Phase 순서 준수** — Phase 2는 Phase 1 이슈가 Done 이후 시작
3. **기능 이슈 → 후속 테스트 이슈** — 기능이 Done이 되면 대응되는 테스트 이슈를 바로 이어서 진행
4. **의존성 확인** — 이슈 본문에 명시된 선행 이슈가 Done인지 확인 후 시작

### 이슈 ↔ 레포 매핑

| 레포 | 이슈 범위 |
|---|---|
| project-ridy/docs | #1~#5 (기획, 문서, 디자인 가이드) |
| project-ridy/backend | #1~#14 (API, DB, 인프라) |
| project-ridy/frontend | #1~#13 (화면, 컴포넌트, 테스트) |

## 에이전트 규칙

### 공통
1. 작업 시작 전 **이슈에 자신을 어사인** (`gh issue edit <번호> --add-assignee @me`)
2. 작업 시작 전 **GitHub Project에서 이슈 상태를 "In Progress"로 변경**
3. 작업 시작 전 반드시 관련 docs 문서를 읽는다
4. 브랜치 네이밍: `<type>/<이슈번호>-<설명>` (예: `feat/3-auth-login`)
5. main 브랜치에 직접 커밋 금지 — PR로 리뷰
6. 완료 후 **이슈를 "Done"으로 변경**
7. 기능 작업 후 **후속 테스트 이슈를 진행** (없는 경우 생성)

### Developer
1. API 작업 시 `docs/api/` 스펙을 정확히 따른다
2. DB 작업 시 `docs/architecture/DATABASE.md` 스키마를 기준으로 한다
3. 프론트엔드는 `docs/design/DESIGN_SYSTEM.md` 토큰을 사용한다
4. **TDD 필수** — Red → Green → Refactor 사이클 (CONTRIBUTING.md 참고)
5. 기능 PR 머지 후 대응되는 테스트 이슈를 진행
6. **기획서 준수** — `docs/plans/`의 기획서에 정의된 코드 구조, 예외 처리, 테스트 시나리오를 따른다

### Planner
1. 사용자 요청을 기능 단위로 분해하여 개발 기획서를 작성한다
2. 기획서에는 코드 구조, 함수 시그니처, 예외 처리, 엣지 케이스, 테스트 시나리오를 포함한다
3. 기존 docs 스펙(API, DB, 디자인)과의 충돌 여부를 사전에 확인한다
4. 기획서는 `docs/plans/` 디렉토리에 작성한다
5. 기획서 머지 전 Orchestrator의 리뷰를 받는다

### Designer
1. `docs/design/DESIGN_SYSTEM.md`의 컬러/타포를 준수한다
2. 화면 설계는 `docs/design/SCREENS.md` 정의를 따른다
3. 와이어프레임은 `docs/design/WIREFRAMES.md`를 기준으로 구체화한다
4. 산출물은 Figma 링크 또는 코드(HTML/CSS/React)로 제공한다

## 의사소통

- 에이전트 간 직접 소통 없음 — 모든 정보는 docs + GitHub Project를 통해
- 질문/이슈는 GitHub 이슈에 BLOCKED 라벨 + 코멘트로 표현
- Orchestrator가 BLOCKED 이슈를 확인하고 docs 업데이트로 해결
