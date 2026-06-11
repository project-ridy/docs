# Ridy — 인증 API (GraphQL)

## 개요

초대 코드 기반 가입 + 소셜 로그인(카카오/구글). 같은 회사 사원만 가입 가능.

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
  domain: String
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
  provider: String
  providerId: String
  role: Role!
  companyId: String!
  company: Company!
  employeeId: String
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

input JoinInput {
  inviteCode: String!
  provider: String!
  oauthToken: String!
}

input LoginInput {
  provider: String!
  oauthToken: String!
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
  joinWithInviteCode(input: JoinInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!
  updateProfile(input: UpdateProfileInput!): User!
  generateInviteCode(input: GenerateInviteCodeInput!): InviteCode!  @adminOnly
  deactivateInviteCode(id: ID!): InviteCode!  @adminOnly
}
```

---

## 기능: 초대 코드 가입 (`joinWithInviteCode`)

### 설명

기업 관리자가 발급한 초대 코드 + 소셜 OAuth 토큰으로 가입. 코드 검증 후 회사에 자동 매핑.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 정상 가입 | 유효한 초대 코드 + 카카오 토큰 | AuthPayload 발급, user.companyId 설정 |
| A2 | 구글 가입 | 유효한 초대 코드 + 구글 토큰 | 동일하게 가입 성공 |
| A3 | 만료 임박 코드 | expiresAt 1시간 전 코드 | 가입 성공 (아직 유효) |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 잘못된 초대 코드 | 존재하지 않는 코드 | `NOT_FOUND: 유효하지 않은 초대 코드입니다` |
| E2 | 만료된 코드 | expiresAt 경과 | `BAD_REQUEST: 만료된 초대 코드입니다` |
| E3 | 사용 한도 초과 | currentUses >= maxUses | `BAD_REQUEST: 초대 코드 사용 한도가 초과되었습니다` |
| E4 | 비활성화된 코드 | isActive=false | `BAD_REQUEST: 비활성화된 초대 코드입니다` |
| E5 | 회원 정원 초과 | company.memberCount >= maxMembers | `BAD_REQUEST: 회사 정원이 초과되었습니다` |
| E6 | 이미 가입된 이메일 | 기존 유저 이메일 | `CONFLICT: 이미 가입된 계정입니다` |
| E7 | 빈 OAuth 토큰 | oauthToken="" | `UNAUTHORIZED: OAuth 토큰이 필요합니다` |
| E8 | 지원하지 않는 provider | provider="naver" | `BAD_REQUEST: 지원하지 않는 로그인 방식입니다` |
| E9 | 만료된 소셜 토큰 | 카카오에서 토큰 거절 | `UNAUTHORIZED: 소셜 로그인에 실패했습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 가입 | 같은 코드로 2명 동시 가입 (maxUses=1) | 1명만 성공, 나머지 E3 |
| X2 | 이메일 도메인 불일치 | 회사 domain과 다른 이메일 | 가입은 허용 (도메인 검증은 선택) |
| X3 | 관리자 본인 가입 | admin이 자기 코드로 재가입 | E6 (이미 가입됨) |

---

## 기능: 로그인 (`login`)

### 설명

기존 가입 유저의 소셜 로그인. 이메일 기준으로 유저 조회.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 카카오 로그인 | AuthPayload 발급 |
| A2 | 구글 로그인 | AuthPayload 발급 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 미가입 이메일 | `NOT_FOUND: 가입되지 않은 계정입니다. 초대 코드로 가입해주세요.` |
| E2 | 만료된 소셜 토큰 | `UNAUTHORIZED` |

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

## 기능: 초대 코드 발급 (`generateInviteCode`) — 관리자 전용

### 설명

회사 관리자가 6자리 초대 코드 발급. 코드는 회사별 고유.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 기본 발급 | maxUses=10, expiresAt=7일 후 코드 생성 |
| A2 | 커스텀 발급 | maxUses=50, expiresAt=30일 후 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 관리자 아님 | `FORBIDDEN: 관리자 권한이 필요합니다` |
| E2 | maxUses 과다 | maxUses=9999 | `BAD_REQUEST: 최대 100명까지 설정 가능합니다` |
| E3 | 활성 코드 과다 | 이미 10개 활성 코드 존재 | `BAD_REQUEST: 활성 초대 코드는 10개까지 가능합니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 코드 충돌 | 랜덤 6자리 중복 | 자동 재생성 (최대 3회 시도) |

---

## 기능: 초대 코드 비활성화 (`deactivateInviteCode`) — 관리자 전용

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 비활성화 | isActive=false, 이후 가입 시 E4 에러 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 이미 비활성화됨 | `BAD_REQUEST: 이미 비활성화된 코드입니다` |
| E2 | 다른 회사 코드 | `FORBIDDEN` |
