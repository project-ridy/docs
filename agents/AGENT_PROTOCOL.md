# Ridy — 에이전트 협업 프로토콜

## 개요

Ridy 프로젝트는 3개의 에이전트가 협업합니다:

| 에이전트 | 역할 | 작업 레포 |
|---|---|---|
| **Orchestrator** | 설계, 작업 분배, 품질 관리 | docs |
| **Developer** | 프론트엔드/백엔드 구현 | frontend, backend |
| **Designer** | UI/UX 디자인, 에셋 제작 | frontend (design 파일) |

## 작업 흐름

```
1. Orchestrator가 docs/의 설계 문서를 기준으로 작업 판단
2. agents/DEVELOPER_TASKS.md 또는 agents/DESIGNER_TASKS.md에 작업 추가
3. 각 에이전트가 TASK 파일을 읽고 담당 작업 수행
4. 완료 후 TASK 상태를 COMPLETED로 변경
5. Orchestrator가 결과 확인 후 다음 작업 판단
```

## 작업 등록 형식

```markdown
### TASK-XXX: 작업 제목
- **상태**: PENDING | IN_PROGRESS | COMPLETED | BLOCKED
- **담당**: developer | designer
- **레포**: frontend | backend | docs
- **우선순위**: HIGH | MEDIUM | LOW
- **의존성**: TASK-YYY (선행 작업)
- **설명**: 구체적인 작업 내용
- **완료 조건**: 어떤 상태면 완료인지
- **참고 문서**: docs/의 어떤 파일을 참고할지
```

## 에이전트 규칙

### 공통
1. 작업 시작 전 반드시 관련 docs 문서를 읽는다
2. 브랜치 네이밍: `<type>/<task-id>-<설명>` (예: `feat/TASK-001-auth-login`)
3. main 브랜치에 직접 커밋 금지 — PR로 리뷰
4. 완료 후 TASK 파일 상태 업데이트

### Developer
1. API 작업 시 `docs/api/` 스펙을 정확히 따른다
2. DB 작업 시 `docs/architecture/DATABASE.md` 스키마를 기준으로 한다
3. 프론트엔드는 `docs/design/DESIGN_SYSTEM.md` 토큰을 사용한다
4. 테스트 코드를 반드시 작성한다

### Designer
1. `docs/design/DESIGN_SYSTEM.md`의 컬러/타포를 준수한다
2. 화면 설계는 `docs/design/SCREENS.md` 정의를 따른다
3. 와이어프레임은 `docs/design/WIREFRAMES.md`를 기준으로 구체화한다
4. 산출물은 Figma 링크 또는 코드(HTML/CSS)로 제공한다

## 의사소통

- 에이전트 간 직접 소통 없음 — 모든 정보는 docs를 통해
- 질문/이슈는 TASK의 BLOCKED 상태 + 설명으로 표현
- Orchestrator가 BLOCKED 작업을 확인하고 docs 업데이트로 해결
