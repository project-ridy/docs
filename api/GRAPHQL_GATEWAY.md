# Ridy — GraphQL Gateway API 스펙

## 원칙

Ridy의 API 스펙은 **GraphQL schema-first**로 관리한다.
외부 클라이언트는 `POST /graphql`만 사용하고, Gateway 내부 resolver가 각 bounded context 서비스를 호출해 데이터를 조합한다.

- 스키마 SSoT: `backend/src/graphql/schema.graphql`
- 문서 기준: `docs/api/*.md`, `docs/architecture/MSA.md`
- 타입 생성: backend/frontend 모두 GraphQL codegen 사용
- 내부 서비스 통신: GraphQL이 아니라 service client(gRPC/internal HTTP) + event bus

---

## 엔드포인트

| 용도 | 엔드포인트 | 설명 |
|---|---|---|
| GraphQL API | `POST https://api.ridy.dev/graphql` | Query/Mutation 단일 진입점 |
| GraphQL Playground | `GET https://api.ridy.dev/graphql` | 개발/스테이징에서만 허용 |
| Realtime | `wss://api.ridy.dev/realtime` | 채팅, 라이드 상태 변경, 알림 |
| Health live | `GET https://api.ridy.dev/health/live` | 프로세스 생존 확인 |
| Health ready | `GET https://api.ridy.dev/health/ready` | 내부 의존성 준비 확인 |

---

## Gateway Context

모든 인증 resolver는 다음 context를 받는다.

```typescript
type GraphQLContext = {
  requestId: string;
  viewer: {
    id: string;
    companyId: string;
    role: 'PASSENGER' | 'DRIVER' | 'BOTH' | 'ADMIN';
  } | null;
  authToken: string | null;
};
```

규칙:

1. `viewer.companyId`는 클라이언트 입력으로 받지 않는다.
2. 회사 범위 데이터는 Gateway 또는 내부 서비스에서 반드시 `viewer.companyId`로 필터링한다.
3. 운영자/후속 관리자 기능은 `viewer.role === 'ADMIN'`이어야 한다.
4. 내부 서비스 호출에는 `x-request-id`, `x-company-id`, `x-actor-id`를 전파한다.

---

## 공통 SDL

```graphql
scalar DateTime
scalar JSON

schema {
  query: Query
  mutation: Mutation
}

directive @auth on FIELD_DEFINITION
directive @adminOnly on FIELD_DEFINITION
directive @companyScope on FIELD_DEFINITION

enum CompanyPlan {
  # 후속 B2B 확장 예약. MVP 수수료/기능 분기는 plan을 사용하지 않는다.
  FREE
  PRO
  ENTERPRISE
}

enum Role {
  PASSENGER
  DRIVER
  BOTH
  ADMIN
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

input PaginationInput {
  first: Int = 20
  after: String
}

type ErrorPayload {
  code: String!
  message: String!
  field: String
}

type Query {
  health: Health!
}

type Mutation {
  _empty: Boolean
}

type Health {
  status: String!
  service: String!
}
```

---

## Auth / Company SDL

```graphql
type AuthPayload {
  accessToken: String!
  refreshToken: String!
  user: User!
}

type Company {
  id: ID!
  name: String!
  domain: String!
  plan: CompanyPlan!
  maxMembers: Int!
  memberCount: Int!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User {
  id: ID!
  companyId: ID!
  employeeId: String
  email: String!
  name: String!
  phone: String
  imageUrl: String
  role: Role!
  rating: Float!
  rideCount: Int!
  company: Company!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type InviteCode {
  id: ID!
  code: String!
  maxUses: Int!
  currentUses: Int!
  expiresAt: DateTime
  isActive: Boolean!
  createdAt: DateTime!
}

type EmailVerificationChallenge {
  id: ID!
  companyEmail: String!
  expiresAt: DateTime!
  resendAvailableAt: DateTime!
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
  maxUses: Int = 10
  expiresAt: DateTime
}

type InviteCodeConnection {
  nodes: [InviteCode!]!
  pageInfo: PageInfo!
}

type MemberConnection {
  nodes: [User!]!
  pageInfo: PageInfo!
}

extend type Query {
  me: User! @auth
  myCompany: Company! @auth @companyScope
  validateInviteCode(code: String!): InviteCode!
  inviteCodes(pagination: PaginationInput): InviteCodeConnection! @auth @adminOnly @companyScope
  companyMembers(pagination: PaginationInput): MemberConnection! @auth @adminOnly @companyScope
}

extend type Mutation {
  requestCompanyEmailVerification(input: RequestCompanyEmailVerificationInput!): EmailVerificationChallenge!
  completeEmailPasswordSignup(input: CompleteEmailPasswordSignupInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!
  updateProfile(input: UpdateProfileInput!): User! @auth
  generateInviteCode(input: GenerateInviteCodeInput!): InviteCode! @auth @adminOnly @companyScope
  deactivateInviteCode(id: ID!): InviteCode! @auth @adminOnly @companyScope
}
```

---

## Matching SDL

```graphql
enum RideStatus {
  OPEN
  MATCHED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

enum RequestStatus {
  PENDING
  ACCEPTED
  REJECTED
  CANCELLED
}

type LatLng {
  lat: Float!
  lng: Float!
}

input LatLngInput {
  lat: Float!
  lng: Float!
}

type Ride {
  id: ID!
  companyId: ID!
  driver: User!
  departure: LatLng!
  departureAddr: String
  arrival: LatLng!
  arrivalAddr: String
  departureTime: DateTime!
  availableSeats: Int!
  fare: Int
  preferences: JSON
  status: RideStatus!
  requests: [RideRequest!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type RideRequest {
  id: ID!
  ride: Ride!
  passenger: User!
  pickup: LatLng
  pickupAddr: String
  message: String
  status: RequestStatus!
  createdAt: DateTime!
  updatedAt: DateTime!
}

input CreateRideInput {
  departure: LatLngInput!
  departureAddr: String
  arrival: LatLngInput!
  arrivalAddr: String
  departureTime: DateTime!
  availableSeats: Int!
  fare: Int
  preferences: JSON
}

input SearchRidesInput {
  departure: LatLngInput!
  arrival: LatLngInput!
  departureTime: DateTime!
  passengers: Int = 1
  radiusKm: Float = 5.0
}

input RequestRideInput {
  rideId: ID!
  pickup: LatLngInput
  pickupAddr: String
  message: String
}

type RideConnection {
  nodes: [Ride!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

extend type Query {
  searchRides(input: SearchRidesInput!, pagination: PaginationInput): RideConnection! @auth @companyScope
  ride(id: ID!): Ride! @auth @companyScope
  myRides(status: RideStatus, pagination: PaginationInput): RideConnection! @auth @companyScope
  myRideRequests(status: RequestStatus, pagination: PaginationInput): [RideRequest!]! @auth @companyScope
}

extend type Mutation {
  createRide(input: CreateRideInput!): Ride! @auth @companyScope
  updateRide(id: ID!, input: CreateRideInput!): Ride! @auth @companyScope
  cancelRide(id: ID!): Ride! @auth @companyScope
  requestRide(input: RequestRideInput!): RideRequest! @auth @companyScope
  acceptRideRequest(id: ID!): RideRequest! @auth @companyScope
  rejectRideRequest(id: ID!): RideRequest! @auth @companyScope
  cancelRideRequest(id: ID!): RideRequest! @auth @companyScope
}
```

---

## Chat SDL

```graphql
type ChatRoom {
  id: ID!
  ride: Ride!
  lastMessage: Message
  unreadCount: Int!
  createdAt: DateTime!
}

enum MessageType {
  TEXT
  IMAGE
  SYSTEM
}

type Message {
  id: ID!
  roomId: ID!
  sender: User!
  type: MessageType!
  content: String!
  createdAt: DateTime!
}

type MessageConnection {
  nodes: [Message!]!
  pageInfo: PageInfo!
}

extend type Query {
  chatRooms(pagination: PaginationInput): [ChatRoom!]! @auth @companyScope
  messages(roomId: ID!, pagination: PaginationInput): MessageConnection! @auth @companyScope
}
```

실시간 메시지 송수신은 GraphQL Mutation이 아니라 `wss://api.ridy.dev/realtime` 이벤트로 처리한다.

| 이벤트 | 방향 | 설명 |
|---|---|---|
| `chat:join` | client → server | 채팅방 입장 |
| `chat:message` | client → server | 메시지 전송 |
| `chat:messageCreated` | server → client | 새 메시지 브로드캐스트 |
| `ride:statusChanged` | server → client | 라이드 상태 변경 알림 |

---

## Payment / Analytics SDL

```graphql
enum SettlementStatus {
  PENDING
  PAID
  FAILED
  CANCELLED
}

type FareCalculation {
  distanceKm: Float!
  baseFare: Int!
  platformFee: Int!
  companySubsidy: Int!
  passengerAmount: Int!
  driverAmount: Int!
}

type Settlement {
  id: ID!
  ride: Ride!
  passenger: User!
  amount: Int!
  driverAmount: Int!
  platformFee: Int!
  status: SettlementStatus!
  dueDate: DateTime
  paidAt: DateTime
  createdAt: DateTime!
}

type UsageStat {
  period: String!
  totalRides: Int!
  totalUsers: Int!
  activeUsers: Int!
  totalDistanceKm: Float!
}

type EsgReport {
  period: String!
  totalRides: Int!
  totalDistanceKm: Float!
  co2SavedKg: Float!
  participantCount: Int!
}

type CarbonBadge {
  id: ID!
  label: String!
  description: String!
  achievedAt: DateTime
}

type CarbonImpact {
  period: String!
  totalRides: Int!
  totalDistanceKm: Float!
  co2SavedKg: Float!
  treeEquivalent: Float!
  level: String!
  badges: [CarbonBadge!]!
}

type CarbonHistoryPoint {
  period: String!
  totalRides: Int!
  totalDistanceKm: Float!
  co2SavedKg: Float!
}

type CarbonRideSaving {
  rideId: ID!
  completedAt: DateTime!
  departureAddr: String
  arrivalAddr: String
  distanceKm: Float!
  co2SavedKg: Float!
}

type CarbonHistory {
  monthly: [CarbonHistoryPoint!]!
  rides: [CarbonRideSaving!]!
  pageInfo: PageInfo!
}

input CalculateFareInput {
  departure: LatLngInput!
  arrival: LatLngInput!
  passengers: Int = 1
}

extend type Query {
  calculateFare(input: CalculateFareInput!): FareCalculation! @auth @companyScope
  mySettlements(pagination: PaginationInput): [Settlement!]! @auth @companyScope
  companyUsageStat(period: String!): UsageStat! @auth @adminOnly @companyScope
  companyEsgReport(period: String!): EsgReport! @auth @adminOnly @companyScope
  myCarbonImpact(period: String): CarbonImpact! @auth @companyScope
  carbonHistory(period: String!, pagination: PaginationInput): CarbonHistory! @auth @companyScope
}

extend type Mutation {
  paySettlement(settlementId: ID!, idempotencyKey: String!): Settlement! @auth @companyScope
}
```

---

## 에러 스펙

GraphQL 표준 에러 형식을 사용하고, `extensions.code`를 고정한다.

```json
{
  "errors": [
    {
      "message": "다른 회사의 리소스에 접근할 수 없습니다.",
      "extensions": {
        "code": "FORBIDDEN",
        "requestId": "req_xxx",
        "field": "ride"
      }
    }
  ],
  "data": null
}
```

| code | 의미 |
|---|---|
| `UNAUTHENTICATED` | 로그인 필요 또는 토큰 만료 |
| `FORBIDDEN` | 권한 없음 또는 회사 범위 위반 |
| `NOT_FOUND` | 리소스 없음 |
| `BAD_REQUEST` | 입력값 오류 또는 비즈니스 룰 위반 |
| `CONFLICT` | 중복 요청 또는 상태 충돌 |
| `PAYMENT_FAILED` | 결제 실패 |
| `TOO_MANY_REQUESTS` | 요청 제한 초과 |
| `INTERNAL_ERROR` | 서버 내부 오류 |

---

## 보안/성능 제한

| 항목 | 기준 |
|---|---|
| Query depth | 최대 8 |
| Query complexity | 요청당 최대 1,000점 |
| List size | `first` 최대 50 |
| Timeout | Gateway resolver 전체 5초, 내부 서비스 호출 1~3초 |
| Introspection | production 비활성화 또는 admin token 필요 |
| Persisted Query | production 권장 |
| Rate limit | user/company/IP 단위 적용 |
| Idempotency | 결제/정산 mutation은 `idempotencyKey` 필수 |
