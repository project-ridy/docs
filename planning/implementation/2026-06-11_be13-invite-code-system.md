# BE #13: 초대 코드 시스템 구현 계획서

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** 회사 관리자가 초대 코드를 발급/조회/비활성화하고, 가입 플로우에서 초대 코드를 검증할 수 있는 기반 API와 서비스를 구현한다.

**Architecture:** `Company`가 최상위 격리 단위이며, 초대 코드는 `InviteCode` 테이블에 저장된다. 인증/JWT(#3)는 아직 미구현이므로 Resolver는 `GraphQLContext.currentUser`를 통해 관리자 정보를 받도록 설계하고, 실제 context 주입은 #3에서 연결한다.

**Tech Stack:** NestJS 11 + GraphQL schema-first + Prisma 7 + Jest

**Issue:** project-ridy/backend #13

---

## 범위 조정

기존 #13 이슈에는 `users.invite_code`, `users.invited_by` 필드 추가가 포함되어 있었지만, 회사 기반 재기획 이후 DB 설계는 다음과 같이 변경됨:

- 초대 코드는 `invite_codes` 테이블로 관리
- 유저는 `company_id`로 회사에 소속
- 가입 시 초대 코드 검증 → 해당 `companyId`로 유저 생성 (#3에서 처리)

따라서 이번 작업은 다음까지만 구현:

1. 초대 코드 GraphQL SDL 추가
2. `InviteCodeService` 구현
3. `InviteCodeResolver` 구현
4. 관리자 context 타입 정의
5. 유닛 테스트 작성

`joinWithInviteCode` 가입/JWT 발급은 #3 인증 API에서 구현.

---

## 기능 케이스 분석

### 기능: 초대 코드 발급 (`generateInviteCode`)

### 설명
회사 관리자가 6자리 영숫자 초대 코드를 발급한다. 같은 회사 내 코드는 중복될 수 없으며, 활성 코드는 회사당 최대 10개까지 허용한다.

### 정상 케이스 (A)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 발급 | maxUses 미입력 | 6자리 코드, maxUses=10, isActive=true |
| A2 | 커스텀 발급 | maxUses=50, expiresAt 지정 | 입력값 반영 |
| A3 | 코드 충돌 후 재시도 | 첫 생성 코드가 중복 | 새 코드로 재시도 후 생성 |

### 예외 케이스 (E)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 미인증 사용자 | currentUser 없음 | `UnauthorizedException` |
| E2 | 관리자 아님 | role=PASSENGER | `ForbiddenException` |
| E3 | maxUses 0 이하 | maxUses=0 | `BadRequestException` |
| E4 | maxUses 과다 | maxUses=101 | `BadRequestException` |
| E5 | 활성 코드 10개 이상 | activeCount=10 | `BadRequestException` |
| E6 | 코드 3회 충돌 | 3회 모두 중복 | `BadRequestException` |

### 엣지 케이스 (X)

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 만료일 과거 | expiresAt이 현재보다 과거 | `BadRequestException` |
| X2 | maxUses 경계값 | maxUses=1, 100 | 정상 생성 |

---

### 기능: 초대 코드 검증 (`validateInviteCode`)

### 설명
가입 전에 초대 코드가 유효한지 확인하고 소속 회사 정보를 반환한다.

### 정상 케이스 (A)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 유효 코드 | active, 미만료, currentUses < maxUses | InviteCode + Company 반환 |

### 예외 케이스 (E)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 존재하지 않는 코드 | unknown | `NotFoundException` |
| E2 | 비활성 코드 | isActive=false | `BadRequestException` |
| E3 | 만료 코드 | expiresAt < now | `BadRequestException` |
| E4 | 사용 한도 초과 | currentUses >= maxUses | `BadRequestException` |
| E5 | 회사 정원 초과 | memberCount >= maxMembers | `BadRequestException` |

### 엣지 케이스 (X)

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 코드 소문자 입력 | abc123 | 대문자로 정규화 후 조회 |
| X2 | 코드 공백 포함 | ` ABC123 ` | trim 후 조회 |

---

### 기능: 초대 코드 비활성화 (`deactivateInviteCode`)

### 설명
관리자가 자기 회사의 초대 코드를 비활성화한다.

### 정상 케이스 (A)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 정상 비활성화 | 활성 코드 id | isActive=false |

### 예외 케이스 (E)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 미인증 사용자 | currentUser 없음 | `UnauthorizedException` |
| E2 | 관리자 아님 | role=PASSENGER | `ForbiddenException` |
| E3 | 없는 코드 | unknown id | `NotFoundException` |
| E4 | 다른 회사 코드 | companyId 불일치 | `ForbiddenException` |
| E5 | 이미 비활성화 | isActive=false | `BadRequestException` |

---

## Tasks

### Task 1: #13 이슈 본문 최신화

**Objective:** 오래된 `users.invite_code` 설명을 제거하고 현재 DB 스키마 기준으로 이슈를 갱신한다.

**Files:** GitHub Issue #13

**Steps:**
1. `gh issue edit 13 --repo project-ridy/backend --body-file /tmp/be13-body.md`
2. 선행 이슈를 #2 완료, #3 연동 예정으로 수정

---

### Task 2: 작업 브랜치 생성

**Objective:** main 최신 상태에서 기능 브랜치 생성

```bash
cd /home/heeho3/workspace/ridy/backend
git checkout main && git pull origin main
git checkout -b feat/13-invite-code-system
```

---

### Task 3: GraphQL SDL 먼저 수정

**Objective:** schema-first 원칙에 따라 초대 코드 타입/쿼리/뮤테이션 정의

**Files:**
- Modify: `src/graphql/schema.graphql`

**Add:**
- `scalar DateTime`
- `enum CompanyPlan`
- `enum Role`
- `type Company`
- `type User`
- `type InviteCode`
- `input GenerateInviteCodeInput`
- Query: `validateInviteCode(code: String!): InviteCode!`, `inviteCodes: [InviteCode!]!`
- Mutation: `generateInviteCode(input: GenerateInviteCodeInput!): InviteCode!`, `deactivateInviteCode(id: ID!): InviteCode!`

**Verify:**
```bash
npm run codegen
```

---

### Task 4: Context 타입 추가

**Objective:** 인증 구현 전에도 Resolver 테스트가 가능하도록 `currentUser` 타입 정의

**Files:**
- Modify: `src/graphql/context.ts`

```ts
export type CurrentUser = {
  readonly id: string;
  readonly companyId: string;
  readonly role: 'PASSENGER' | 'DRIVER' | 'BOTH' | 'ADMIN';
};

export type GraphQLContext = {
  readonly currentUser?: CurrentUser;
};
```

---

### Task 5: RED — InviteCodeService 테스트 작성

**Objective:** 구현 전 실패하는 유닛 테스트 작성

**Files:**
- Create: `src/modules/invite-code/invite-code.service.spec.ts`

**Test cases:**
- A1 기본 발급
- E2 관리자 아님
- E4 maxUses 과다
- E5 활성 코드 10개 이상
- A1 validate 유효 코드
- E1 validate 없는 코드
- E3 validate 만료 코드
- A1 deactivate 정상
- E4 deactivate 다른 회사 코드

**Run:**
```bash
npm run test -- --runTestsByPath src/modules/invite-code/invite-code.service.spec.ts
```
Expected: FAIL — `Cannot find module './invite-code.service'`

---

### Task 6: GREEN — InviteCodeService 구현

**Objective:** 테스트를 통과하는 최소 서비스 구현

**Files:**
- Create: `src/modules/invite-code/invite-code.service.ts`

**Methods:**
- `generateInviteCode(currentUser, input)`
- `validateInviteCode(code)`
- `deactivateInviteCode(currentUser, id)`
- private `assertAdmin(currentUser)`
- private `normalizeCode(code)`
- private `createUniqueCode(companyId)`

**Rules:**
- code는 `trim().toUpperCase()`
- maxUses 기본값 10, 허용 범위 1~100
- 활성 코드 10개 제한
- 코드 충돌 최대 3회 재시도
- validate 시 company.users count로 정원 확인

---

### Task 7: Resolver + Module 구현

**Objective:** GraphQL entrypoint 구성

**Files:**
- Create: `src/modules/invite-code/invite-code.resolver.ts`
- Create: `src/modules/invite-code/invite-code.module.ts`
- Modify: `src/app.module.ts`

**Resolver:**
- `@Query('validateInviteCode')`
- `@Query('inviteCodes')`
- `@Mutation('generateInviteCode')`
- `@Mutation('deactivateInviteCode')`

---

### Task 8: 전체 검증

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

Expected: 모두 exit 0

---

### Task 9: 커밋/PR/머지/브랜치 정리

```bash
git add -A
git commit -m "feat(invite): 초대 코드 발급/검증/비활성화 구현

Closes #13"
git push -u origin feat/13-invite-code-system
# PR 생성, 머지, 브랜치 삭제
```
