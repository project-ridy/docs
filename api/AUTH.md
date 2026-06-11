# Ridy — 인증 API (GraphQL)

## 개요

소셜 로그인(카카오/구글/Apple) 기반 인증. JWT 액세스/리프레시 토큰 발급.

---

## GraphQL 스키마

```graphql
type AuthPayload {
  accessToken: String!
  refreshToken: String!
  user: User!
}

input LoginInput {
  provider: String!   # kakao | google | apple
  oauthToken: String! # 소셜 OAuth 액세스 토큰
}

input UpdateProfileInput {
  name: String
  phone: String
  imageUrl: String
  role: Role
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
  rating: Float!
  rideCount: Int!
  phoneVerified: Boolean!
  createdAt: String!
  updatedAt: String!
}

enum Role {
  passenger
  driver
  both
}

type Query {
  me: User!
}

type Mutation {
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!
  updateProfile(input: UpdateProfileInput!): User!
}
```

---

## 기능: 소셜 로그인 (`login`)

### 설명

소셜 제공자의 OAuth 액세스 토큰을 검증하고, 기존 유저면 로그인, 신규 유저면 자동 가입 후 토큰 발급.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 카카오 기존 유저 로그인 | `{ provider: "kakao", oauthToken: "valid_kakao_token" }` | 기존 유저 정보 + JWT 토큰 발급, `isNewUser: false` |
| A2 | 구글 신규 유저 자동 가입 | `{ provider: "google", oauthToken: "valid_google_token" }` | 새 User 레코드 생성 + JWT 토큰 발급, `isNewUser: true` |
| A3 | Apple 로그인 (이메일 선택적) | `{ provider: "apple", oauthToken: "valid_apple_token" }` | Apple에서 이메일 안 주면 임시 이메일 생성 (`apple_{sub}@ridy.app`) |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 빈 oauthToken | `{ provider: "kakao", oauthToken: "" }` | `UNAUTHORIZED: OAuth 토큰이 필요합니다` |
| E2 | 지원하지 않는 provider | `{ provider: "naver", oauthToken: "..." }` | `BAD_REQUEST: 지원하지 않는 로그인 방식입니다` |
| E3 | 만료된 소셜 토큰 | `{ provider: "kakao", oauthToken: "expired_token" }` | `UNAUTHORIZED: 소셜 로그인 토큰이 만료되었습니다` |
| E4 | 유효하지 않은 소셜 토큰 | `{ provider: "google", oauthToken: "invalid_token" }` | `UNAUTHORIZED: 소셜 로그인 인증에 실패했습니다` |
| E5 | 소셜 API 장애 | Kakao/Google API 타임아웃 | `INTERNAL_ERROR: 소셜 로그인 서비스에 일시적인 문제가 있습니다` |
| E6 | 탈퇴한 유저 재로그인 | 탈퇴 처리된 유저의 소셜 토큰 | `FORBIDDEN: 탈퇴한 계정입니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동일 이메일 다른 provider | 카카오로 가입 → 구글 같은 이메일로 로그인 | provider 기준으로 별도 계정 처리 (이메일 UNIQUE는 provider 조합) |
| X2 | 소셜에서 이름 변경 | 카카오 프로필 이름 변경 후 재로그인 | 기존 DB 이름 유지 (자동 덮어쓰지 않음) |
| X3 | Apple private relay 이메일 | `@privaterelay.appleid.com` 도메인 | 일반 이메일과 동일하게 처리 |

---

## 기능: 토큰 갱신 (`refreshToken`)

### 설명

만료된 액세스 토큰을 리프레시 토큰으로 갱신.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 유효한 리프레시 토큰 | `{ token: "valid_refresh_token" }` | 새 액세스 + 리프레시 토큰 쌍 발급 (순환) |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 만료된 리프레시 토큰 | 7일 경과한 토큰 | `UNAUTHORIZED: 리프레시 토큰이 만료되었습니다. 다시 로그인해주세요` |
| E2 | 변조된 토큰 | 잘못된 서명 | `UNAUTHORIZED: 유효하지 않은 토큰입니다` |
| E3 | 빈 토큰 | `{ token: "" }` | `UNAUTHORIZED: 토큰이 필요합니다` |
| E4 | 로그아웃된 토큰 | 로그아웃 처리된 리프레시 토큰 | `UNAUTHORIZED: 로그아웃된 토큰입니다` |

---

## 기능: 내 프로필 조회 (`me`)

### 설명

인증된 유저의 프로필 정보 조회. Authorization 헤더의 Bearer 토큰으로 인증.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 정상 조회 | 유저 전체 필드 반환 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 토큰 없음 | `UNAUTHENTICATED: 로그인이 필요합니다` |
| E2 | 만료된 액세스 토큰 | `UNAUTHENTICATED: 액세스 토큰이 만료되었습니다` |
| E3 | 삭제된 유저 | `NOT_FOUND: 사용자를 찾을 수 없습니다` |

---

## 기능: 프로필 수정 (`updateProfile`)

### 설명

유저 이름, 전화번호, 프로필 이미지, 역할 변경.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 이름 변경 | `{ name: "새이름" }` | 변경된 이름 반영 |
| A2 | 역할 변경 (passenger → both) | `{ role: "both" }` | 차주+탑승자 모두 가능 |
| A3 | 여러 필드 동시 변경 | `{ name: "이름", phone: "01012345678" }` | 전부 반영 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 빈 이름 | `{ name: "" }` | `BAD_REQUEST: 이름은 비워둘 수 없습니다` |
| E2 | 잘못된 전화번호 형식 | `{ phone: "abc" }` | `BAD_REQUEST: 올바른 전화번호 형식이 아닙니다` |
| E3 | 잘못된 role 값 | `{ role: "admin" }` | `BAD_REQUEST: 올바르지 않은 역할입니다` |
| E4 | 인증 안 됨 | 토큰 없음 | `UNAUTHENTICATED: 로그인이 필요합니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 진행 중인 운행 있는 차주가 role 변경 | 차주 → passenger로 변경 시도 | 진행 중인 운행이 있으면 `FORBIDDEN: 진행 중인 운행이 있어 역할을 변경할 수 없습니다` |
| X2 | 아무 필드도 안 보냄 | `{}` | 현재 프로필 그대로 반환 (에러 아님) |

---

## 인증 가드 (AuthGuard)

### 적용 범위

- `me` query
- `updateProfile` mutation
- 이후 모든 인증 필요 mutation/query

### 인증 흐름

1. Authorization 헤더에서 `Bearer <token>` 추출
2. JWT 검증 (서명 + 만료)
3. 페이로드의 `sub`로 유저 조회
4. 컨텍스트에 유저 정보 주입

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| G1 | Authorization 헤더 없음 | `UNAUTHENTICATED` |
| G2 | Bearer 스킴 아님 | `UNAUTHENTICATED` |
| G3 | 토큰 만료 | `UNAUTHENTICATED` |
| G4 | 토큰 서명 무효 | `UNAUTHENTICATED` |
