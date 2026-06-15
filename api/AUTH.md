# Ridy — 인증 API (GraphQL)

## 개요

가입 코드 + 회사 이메일 인증코드 + 비밀번호 기반 인증. 같은 회사 이메일 도메인 구성원만 가입 가능하며, OAuth 간편로그인(카카오/구글)은 인증 SSoT에서 제거한다.

인증 플로우는 다음 순서를 따른다.

1. 사용자가 가입 코드와 회사 이메일을 입력한다.
2. backend가 가입 코드와 회사 도메인을 검증한 뒤 회사 이메일로 6자리 인증코드를 발송한다.
3. 사용자가 인증코드와 비밀번호를 입력한다.
4. backend가 인증코드를 검증하고 비밀번호 해시를 저장한 뒤 AuthPayload를 발급한다.
5. 기존 사용자는 회사 이메일 + 비밀번호로 로그인한다.

---

## GraphQL 스키마

```graphql
type AuthPayload {
  accessToken: String!
  refreshToken: String!
  user: User!
}

type Company {
  id: ID!
  name: String!
  inviteCode: String!
  domain: String!
  plan: CompanyPlan!
  maxMembers: Int!
  memberCount: Int!
  createdAt: String!
}

enum CompanyPlan {
  FREE
  PRO
  ENTERPRISE
}

enum Role {
  passenger
  driver
  both
  admin
}

type User {
  id: ID!
  email: String!
  name: String!
  phone: String
  imageUrl: String
  role: Role!
  companyId: String!
  company: Company!
  employeeId: String
  emailVerifiedAt: String
  rating: Float!
  rideCount: Int!
  createdAt: String!
  updatedAt: String!
}

type InviteCode {
  id: ID!
  code: String!
  companyId: String!
  createdBy: User!
  maxUses: Int!
  currentUses: Int!
  expiresAt: String
  isActive: Boolean!
  createdAt: String!
}

type EmailVerificationChallenge {
  id: ID!
  companyEmail: String!
  expiresAt: String!
  resendAvailableAt: String!
}

input RequestCompanyEmailVerificationInput {
  inviteCode: String!
  companyEmail: String!
}

input CompleteEmailPasswordSignupInput {
  challengeId: ID!
  verificationCode: String!
  password: String!
}

input LoginInput {
  companyEmail: String!
  password: String!
}

input UpdateProfileInput {
  name: String
  phone: String
  imageUrl: String
  role: Role
  employeeId: String
}

input GenerateInviteCodeInput {
  maxUses: Int
  expiresAt: String
}

type Query {
  me: User!
  myCompany: Company!
  inviteCodes(companyId: ID!): [InviteCode!]!  @adminOnly
}

type Mutation {
  requestCompanyEmailVerification(input: RequestCompanyEmailVerificationInput!): EmailVerificationChallenge!
  completeEmailPasswordSignup(input: CompleteEmailPasswordSignupInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!
  updateProfile(input: UpdateProfileInput!): User!
  generateInviteCode(input: GenerateInviteCodeInput!): InviteCode!  @adminOnly
  deactivateInviteCode(id: ID!): InviteCode!  @adminOnly
}
```

---

## 기능: 회사 이메일 인증코드 발송 (`requestCompanyEmailVerification`)

### 설명

운영자가 배포한 가입 코드와 회사 이메일을 검증한 뒤, 회사 이메일로 6자리 인증코드를 발송한다. 이 mutation은 계정을 생성하지 않고 `EmailVerificationChallenge`만 생성한다.

검증 규칙:

- 가입 코드는 trim + uppercase 정규화 후 조회한다.
- 회사 이메일은 trim + lowercase 정규화 후 저장한다.
- `companyEmail` 도메인은 가입 코드가 속한 `Company.domain`과 일치해야 한다.
- 인증코드는 원문 저장 금지. backend는 인증코드 해시만 저장한다.
- 인증코드는 10분 뒤 만료된다.
- 재발송은 같은 이메일 기준 60초 후 허용한다.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 정상 인증코드 발송 | 유효한 가입 코드 + 회사 이메일 | 인증코드 이메일 발송, challenge 반환 |
| A2 | 가입 코드 소문자/공백 | ` abc123 ` + 회사 이메일 | `ABC123`으로 정규화 후 성공 |
| A3 | 회사 이메일 대소문자 | `Jane@Acme.CO.KR` | lowercase 정규화 후 성공 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 잘못된 가입 코드 | 존재하지 않는 코드 | `NOT_FOUND: 유효하지 않은 가입 코드입니다` |
| E2 | 만료된 코드 | expiresAt 경과 | `BAD_REQUEST: 만료된 가입 코드입니다` |
| E3 | 사용 한도 초과 | currentUses >= maxUses | `BAD_REQUEST: 가입 코드 사용 한도가 초과되었습니다` |
| E4 | 비활성화된 코드 | isActive=false | `BAD_REQUEST: 비활성화된 가입 코드입니다` |
| E5 | 회원 정원 초과 | company.memberCount >= maxMembers | `BAD_REQUEST: 회사 정원이 초과되었습니다` |
| E6 | 이미 가입된 이메일 | 기존 user.email | `CONFLICT: 이미 가입된 계정입니다` |
| E7 | 회사 이메일 누락 | companyEmail="" | `BAD_REQUEST: 회사 이메일은 필수입니다` |
| E8 | 회사 이메일 형식 오류 | `jane-company` | `BAD_REQUEST: 올바른 회사 이메일을 입력해주세요` |
| E9 | 회사 이메일 도메인 불일치 | company.domain과 다른 companyEmail | `FORBIDDEN: 회사 이메일 도메인이 일치하지 않습니다` |
| E10 | 재발송 제한 | 60초 이내 재요청 | `TOO_MANY_REQUESTS: 인증코드는 60초 후 다시 요청할 수 있습니다` |
| E11 | 이메일 발송 실패 | 메일 provider 장애 | `SERVICE_UNAVAILABLE: 인증 이메일 발송에 실패했습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 발송 요청 | 같은 이메일로 동시에 요청 | 하나의 최신 challenge만 유효 |
| X2 | 도메인 대소문자 차이 | `Acme.CO.KR` vs `acme.co.kr` | 정규화 후 일치하면 성공 |
| X3 | 이전 challenge 존재 | 만료 전 재요청 | 이전 challenge 무효화 후 새 challenge 발급 |

---

## 기능: 인증코드 검증 + 비밀번호 설정 가입 (`completeEmailPasswordSignup`)

### 설명

사용자가 이메일로 받은 인증코드와 비밀번호를 제출하면, backend가 challenge를 검증하고 유저를 생성한다. 비밀번호는 강한 단방향 해시로 저장하며 원문 저장은 금지한다.

검증 규칙:

- challenge는 존재하고 만료되지 않아야 한다.
- 인증코드는 challenge에 저장된 해시와 일치해야 한다.
- 인증 성공 시 같은 challenge는 재사용할 수 없다.
- 가입 코드 사용 횟수(`InviteCode.currentUses`)와 회사 구성원 수(`Company.memberCount`) 증가는 유저 생성과 같은 transaction에서 처리한다.
- 비밀번호 정책: 최소 10자, 영문/숫자 포함.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 정상 가입 완료 | 유효 challenge + 인증코드 + 강한 비밀번호 | User 생성, emailVerifiedAt 설정, AuthPayload 발급 |
| A2 | 인증코드 앞뒤 공백 | ` 123456 ` | trim 후 일치하면 가입 성공 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 존재하지 않는 challenge | invalid id | `NOT_FOUND: 인증 요청을 찾을 수 없습니다` |
| E2 | 만료된 인증코드 | expiresAt 경과 | `BAD_REQUEST: 인증코드가 만료되었습니다` |
| E3 | 잘못된 인증코드 | 불일치 코드 | `BAD_REQUEST: 인증코드가 일치하지 않습니다` |
| E4 | 이미 사용된 challenge | usedAt 존재 | `BAD_REQUEST: 이미 사용된 인증 요청입니다` |
| E5 | 약한 비밀번호 | 정책 미충족 | `BAD_REQUEST: 비밀번호는 10자 이상이며 영문과 숫자를 포함해야 합니다` |
| E6 | 이미 가입된 이메일 | race로 기존 User 생성됨 | `CONFLICT: 이미 가입된 계정입니다` |
| E7 | 회사 정원 초과 | 검증 후 정원 초과 | `BAD_REQUEST: 회사 정원이 초과되었습니다` |
| E8 | 가입 코드 사용 한도 초과 | 검증 후 maxUses 도달 | `BAD_REQUEST: 가입 코드 사용 한도가 초과되었습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 가입 완료 | 같은 challenge 동시 제출 | 하나만 성공, 나머지는 E4 또는 E6 |
| X2 | maxUses=1 race | 마지막 사용 횟수 경쟁 | transaction으로 1명만 성공 |

---

## 기능: 로그인 (`login`)

### 설명

기존 가입 유저가 회사 이메일과 비밀번호로 로그인한다. OAuth provider/oauthToken 기반 로그인은 제공하지 않는다.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 정상 로그인 | companyEmail + password | AuthPayload 발급 |
| A2 | 이메일 대소문자/공백 | ` Jane@Acme.CO.KR ` | lowercase + trim 정규화 후 로그인 성공 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 미가입 이메일 | `NOT_FOUND: 가입되지 않은 계정입니다. 가입 코드와 회사 이메일 인증으로 가입해주세요.` |
| E2 | 비밀번호 불일치 | `UNAUTHORIZED: 이메일 또는 비밀번호가 올바르지 않습니다` |
| E3 | 이메일 미검증 계정 | `FORBIDDEN: 회사 이메일 인증이 필요합니다` |
| E4 | 빈 이메일 | `BAD_REQUEST: 회사 이메일은 필수입니다` |
| E5 | 빈 비밀번호 | `BAD_REQUEST: 비밀번호는 필수입니다` |

---

## 기능: 토큰 갱신 (`refreshToken`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 유효한 리프레시 토큰 | 새 액세스/리프레시 토큰 쌍 발급 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 만료된 리프레시 토큰 | `UNAUTHORIZED: 로그인이 만료되었습니다. 다시 로그인해주세요.` |
| E2 | 변조된 토큰 | `UNAUTHORIZED: 유효하지 않은 토큰입니다` |

---

## 기능: 프로필 수정 (`updateProfile`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 이름 변경 | name 업데이트 |
| A2 | 차주 전환 | role → driver (또는 both) |
| A3 | 사번 등록 | employeeId 설정 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | admin 권한 변경 시도 | role=admin 설정 불가 (관리자는 별도 승격) |
| E2 | 빈 이름 | name="" | `BAD_REQUEST: 이름은 필수입니다` |

---

## 기능: 가입 코드 발급 (`generateInviteCode`) — 운영자 전용

### 설명

도메인 운영자가 6자리 가입 코드를 발급한다. 코드는 회사별 고유이며, 가입 시 회사 이메일 도메인 검증과 함께 사용된다.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 기본 발급 | maxUses=10, expiresAt=7일 후 코드 생성 |
| A2 | 커스텀 발급 | maxUses=50, expiresAt=30일 후 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 운영자 아님 | `FORBIDDEN: 운영자 권한이 필요합니다` |
| E2 | maxUses 과다 | maxUses=9999 | `BAD_REQUEST: 최대 100명까지 설정 가능합니다` |
| E3 | 활성 코드 과다 | 이미 10개 활성 코드 존재 | `BAD_REQUEST: 활성 가입 코드는 10개까지 가능합니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 코드 충돌 | 랜덤 6자리 중복 | 자동 재생성 (최대 3회 시도) |

---

## 기능: 가입 코드 비활성화 (`deactivateInviteCode`) — 운영자 전용

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 비활성화 | isActive=false, 이후 가입 시 E4 에러 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 이미 비활성화됨 | `BAD_REQUEST: 이미 비활성화된 코드입니다` |
| E2 | 다른 회사 코드 | `FORBIDDEN` |
