# Ridy 작업 규칙

이 문서는 Ridy 프로젝트의 모든 레포(`docs`, `frontend`, `backend`)에서 공통으로 적용되는 작업 규칙입니다.

## 핵심 원칙

1. **모든 작업은 GitHub Organization Project 이슈에서 시작한다.**
   - Project: https://github.com/orgs/project-ridy/projects/1
   - 작업 전 이슈를 자신에게 어사인한다.
   - 가능한 경우 이슈 상태를 `In Progress`로 변경한다.

2. **작업 전 Planner가 구현 계획서를 먼저 작성한다.**
   - 계획서는 반드시 `docs/planning/implementation/`에 저장한다.
   - 로컬 작업 편의를 위해 `.hermes/plans/`에도 같은 내용을 둘 수 있다.
   - 구현자는 계획서의 A/E/X 케이스, 파일 경로, 검증 명령을 기준으로 작업한다.
   - 계획서는 반드시 **구현/테스트 케이스 레지스트리**를 포함한다.
   - 레지스트리는 `Case ID`, `기획 A/E/X 링크`, `구현 파일/단위`, `테스트 파일/테스트명`, `완료 기준`을 연결한다.

3. **docs가 단일 진실 공급원(SSoT)이다.**
   - API: `docs/api/`
   - DB/아키텍처: `docs/architecture/`
   - 디자인/화면: `docs/design/`
   - 이슈별 구현 계획: `docs/planning/implementation/`
   - docs에 없는 스펙은 임의로 구현하지 않고 BLOCKED로 보고한다.

4. **main 브랜치에 직접 커밋하지 않는다.**
   - 항상 기능 브랜치에서 작업한다.
   - PR을 통해 머지한다.

5. **TDD를 기본으로 한다.**
   - Red → Green → Refactor 순서를 지킨다.
   - 테스트 없이 기능 구현을 완료한 것으로 보지 않는다.

6. **GraphQL schema-first를 지킨다.**
   - 백엔드는 `src/graphql/schema.graphql`을 먼저 수정한다.
   - 이후 `npm run codegen`으로 generated 타입을 생성한다.
   - 프론트엔드는 손작성 API 타입을 만들지 않고 generated 타입을 사용한다.

7. **작업 완료 후 실제 검증 결과를 남긴다.**
   - 최소 검증: `npm run test`, `npm run lint`, `npm run build`
   - GraphQL/Prisma 변경 시: `npm run codegen`, `npx prisma validate`, `npx prisma generate`
   - 실행 불가능한 검증은 이유와 대안을 PR 본문에 명시한다.

8. **PR 머지 후 브랜치를 삭제한다.**
   - 로컬 브랜치 삭제: `git branch -d <branch>`
   - 원격 브랜치 삭제: `git push origin --delete <branch>`

---

## 표준 작업 순서

### 1. 이슈 선택

우선순위는 다음 순서로 판단한다.

1. 현재 제품 방향의 기반이 되는 작업
2. 다른 이슈의 선행 조건인 작업
3. 인증/DB/API처럼 하위 기능이 의존하는 작업
4. 사용자에게 직접 보이는 핵심 화면/기능
5. 보조 기능, 리포트, 모니터링

### 2. 구현 계획서 작성

계획서는 다음 형식으로 작성한다.

```txt
planning/implementation/YYYY-MM-DD_<repo><issue>-<short-name>.md
```

예시:

```txt
planning/implementation/2026-06-11_be13-invite-code-system.md
planning/implementation/2026-06-11_fe04-login-onboarding.md
```

계획서에는 반드시 포함한다.

- 목표
- 아키텍처 접근
- 관련 이슈
- 관련 docs 문서
- A/E/X 케이스 분석
  - A: 정상 케이스
  - E: 예외 케이스
  - X: 엣지 케이스
- 구현/테스트 케이스 레지스트리
  - Case ID
  - 기획 A/E/X 링크
  - 구현 파일/단위
  - 테스트 파일/테스트명
  - 완료 기준
- 수정/생성할 파일 경로
- TDD 순서
- 실행할 검증 명령
- 완료 조건

### 3. 이슈 본문 최신화

이슈 내용이 현재 docs와 다르면 먼저 업데이트한다.

- 기능 설명
- 관련 문서 링크
- 작업 범위
- 범위 제외
- TDD 사이클
- 완료 조건
- 구현 계획서 경로
- Case ID(A/E/X)별 구현 예정 파일/단위와 테스트 예정 파일/테스트명

### 4. 브랜치 생성

브랜치 형식:

```txt
<type>/<issue-number>-<short-description>
```

예시:

```txt
feat/13-invite-code-system
docs/10-workflow-rules
test/4-auth-api-tests
```

### 5. 구현

- 계획서 순서대로 진행한다.
- 테스트를 먼저 작성한다.
- 실패를 확인한 뒤 최소 구현한다.
- codegen/generated 산출물은 사람이 직접 수정하지 않는다.

### 6. 검증

레포별 기본 검증:

```bash
npm run test
npm run lint
npm run build
```

백엔드 GraphQL/Prisma 변경 시:

```bash
npm run codegen
npx prisma validate
npx prisma generate
```

프론트 GraphQL 변경 시:

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

### 7. 커밋

커밋/PR 제목 형식:

```txt
<type>(<scope>): <subject 한글>
```

예시:

```txt
feat(auth): 초대 코드 가입 플로우 구현
test(auth): 인증 API E2E 테스트 추가
docs(planning): 이슈별 구현 계획서 추가
```

규칙:

- type은 영어
- subject는 한국어
- 이슈를 닫는 PR/커밋에는 `Closes #<issue-number>` 포함

### 8. PR 생성

PR 본문에는 다음을 포함한다.

- 변경 내용
- 관련 이슈
- 관련 계획서 경로
- 기획서 Case ID 구현/테스트 확인표
  - 모든 A/E/X Case ID
  - 구현 파일/단위
  - 테스트 파일/테스트명
  - 검증 결과
- 검증 결과
- 범위 제외 또는 후속 작업

프론트엔드/백엔드 PR에서 Case ID 확인표가 비어 있거나 계획서 Case ID를 누락하면 완료로 보지 않는다. 계획서 또는 docs에 없는 동작은 구현하지 않고 BLOCKED 처리한 뒤 docs/계획서를 먼저 갱신한다.

### 9. 머지 및 정리

- PR을 squash merge한다.
- main을 pull한다.
- 로컬/원격 브랜치를 삭제한다.

```bash
git checkout main && git pull origin main
git branch -d <branch>
git push origin --delete <branch>
```

---

## 레포별 추가 규칙

### docs

- 제품/기술/디자인 스펙의 SSoT이다.
- 구현 계획서는 `planning/implementation/`에 저장한다.
- API/DB/디자인 변경 시 관련 문서를 함께 갱신한다.

### backend

- NestJS 11 + GraphQL schema-first + Prisma 7 기준.
- `src/graphql/schema.graphql` 수정 → `npm run codegen` → Resolver/Service 구현 순서를 지킨다.
- Prisma schema 변경 시 `docs/architecture/DATABASE.md`와 일치해야 한다.

### frontend

- Next.js 16 App Router + React 19 + Tailwind CSS 4 기준.
- GraphQL operation을 먼저 작성하고 `npm run codegen`을 실행한다.
- 손작성 API 타입을 만들지 않는다.
- 디자인은 `docs/design/DESIGN_SYSTEM.md`, `WIREFRAMES.md`, `SCREENS.md`를 기준으로 한다.

---

## 금지 사항

- main 직접 커밋
- 계획서 없이 다단계 기능 구현 시작
- docs에 없는 스펙 임의 추가
- 테스트 없이 기능 완료 처리
- generated 파일 수동 수정
- 커밋/PR 제목의 type을 한국어로 작성
- PR 머지 후 브랜치 방치
