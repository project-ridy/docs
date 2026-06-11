# Ridy Phase 1 기본 구현 계획서

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Ridy 카풀 앱의 기반 인프라 구축 — 백엔드 DB 스키마/인증 API + 프론트엔드 디자인 시스템/로그인 화면

**Architecture:** 백엔드는 NestJS 11 모놀리식 + GraphQL schema-first + Prisma 7 adapter. 프론트엔드는 Next.js 16 App Router + shadcn/ui + Tailwind CSS 4. 양쪽 모두 `npm run codegen`으로 generated 타입 사용.

**Tech Stack:** NestJS 11, Prisma 7, Apollo Server 5, Next.js 16, React 19, Tailwind CSS 4, shadcn/ui, Vitest, Jest, Socket.IO

---

## 사전 조건

- [x] BE #1: NestJS 프로젝트 셋업 완료
- [x] FE #1: Next.js 프로젝트 셋업 완료
- [x] GraphQL schema-first + codegen 양쪽 설정 완료
- [x] Health check GraphQL resolver 동작 확인

---

## 백엔드 작업 (BE #2 → #3 → #4)

### Task B1: Prisma 스키마 — ENUM 타입 정의

**Objective:** DATABASE.md 기반 Prisma ENUM 6개 정의

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: ENUM 추가**

```prisma
enum Role {
  passenger
  driver
  both
}

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

enum SettlementStatus {
  PENDING
  PAID
  CANCELLED
}

enum PaymentType {
  CARD
  KAKAO_PAY
  TOSS_PAY
}

enum MessageType {
  TEXT
  IMAGE
  SYSTEM
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): Prisma ENUM 타입 정의"
```

---

### Task B2: Prisma 스키마 — users 테이블

**Objective:** users 테이블 Prisma 모델 작성

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: User 모델 추가**

```prisma
model User {
  id             String   @id @default(uuid()) @map("id")
  email          String   @unique @map("email")
  name           String   @map("name")
  phone          String?  @map("phone")
  imageUrl       String?  @map("image_url")
  provider       String?  @map("provider")
  providerId     String?  @map("provider_id")
  role           Role     @default(passenger) @map("role")
  rating         Decimal  @default(0.0) @map("rating") @db.Decimal(2, 1)
  rideCount      Int      @default(0) @map("ride_count")
  phoneVerified  Boolean  @default(false) @map("phone_verified")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  vehicles       Vehicle[]
  rides          Ride[]         @relation("DriverRides")
  rideRequests   RideRequest[]  @relation("PassengerRequests")
  reviewsFrom    Review[]       @relation("ReviewFrom")
  reviewsTo      Review[]       @relation("ReviewTo")
  chatMessages   Message[]
  paymentMethods PaymentMethod[]

  @@map("users")
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): users 테이블 Prisma 모델 작성"
```

---

### Task B3: Prisma 스키마 — vehicles 테이블

**Objective:** 차주 차량 정보 테이블

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: Vehicle 모델 추가**

```prisma
model Vehicle {
  id        String   @id @default(uuid()) @map("id")
  userId    String   @map("user_id")
  model     String   @map("model")
  color     String   @map("color")
  plate     String   @map("plate")
  capacity  Int      @map("capacity")
  createdAt DateTime @default(now()) @map("created_at")

  user      User     @relation(fields: [userId], references: [id])

  @@map("vehicles")
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): vehicles 테이블 Prisma 모델 작성"
```

---

### Task B4: Prisma 스키마 — rides 테이블

**Objective:** 카풀 운행 테이블

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: Ride 모델 추가**

```prisma
model Ride {
  id             String      @id @default(uuid()) @map("id")
  driverId       String      @map("driver_id")
  departureLat   Decimal     @map("departure_lat") @db.Decimal(10, 7)
  departureLng   Decimal     @map("departure_lng") @db.Decimal(10, 7)
  departureAddr  String?     @map("departure_addr")
  arrivalLat     Decimal     @map("arrival_lat") @db.Decimal(10, 7)
  arrivalLng     Decimal     @map("arrival_lng") @db.Decimal(10, 7)
  arrivalAddr    String?     @map("arrival_addr")
  departureTime  DateTime    @map("departure_time")
  availableSeats Int         @map("available_seats")
  fare           Int?        @map("fare")
  recurringDays  String?     @map("recurring_days")
  recurringEnd   DateTime?   @map("recurring_end") @db.Date
  preferences    Json?       @map("preferences")
  status         RideStatus  @default(OPEN) @map("status")
  createdAt      DateTime    @default(now()) @map("created_at")
  updatedAt      DateTime    @updatedAt @map("updated_at")

  driver         User          @relation("DriverRides", fields: [driverId], references: [id])
  rideRequests   RideRequest[]
  settlements    Settlement[]
  chatRooms      ChatRoom[]
  reviews        Review[]

  @@map("rides")
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): rides 테이블 Prisma 모델 작성"
```

---

### Task B5: Prisma 스키마 — ride_requests 테이블

**Objective:** 탑승 요청 테이블

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: RideRequest 모델 추가**

```prisma
model RideRequest {
  id           String        @id @default(uuid()) @map("id")
  rideId       String        @map("ride_id")
  passengerId  String        @map("passenger_id")
  pickupLat    Decimal?      @map("pickup_lat") @db.Decimal(10, 7)
  pickupLng    Decimal?      @map("pickup_lng") @db.Decimal(10, 7)
  pickupAddr   String?       @map("pickup_addr")
  message      String?       @map("message")
  status       RequestStatus @default(PENDING) @map("status")
  createdAt    DateTime      @default(now()) @map("created_at")
  updatedAt    DateTime      @updatedAt @map("updated_at")

  ride         Ride          @relation(fields: [rideId], references: [id])
  passenger    User          @relation("PassengerRequests", fields: [passengerId], references: [id])

  @@map("ride_requests")
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): ride_requests 테이블 Prisma 모델 작성"
```

---

### Task B6: Prisma 스키마 — 나머지 테이블

**Objective:** reviews, settlements, payment_methods, chat_rooms, messages 테이블 작성

**Files:**
- Modify: `backend/prisma/schema.prisma`

**Step 1: 모델 5개 추가**

```prisma
model Review {
  id        String   @id @default(uuid()) @map("id")
  rideId    String   @map("ride_id")
  fromId    String   @map("from_id")
  toId      String   @map("to_id")
  rating    Int      @map("rating")
  comment   String?  @map("comment")
  createdAt DateTime @default(now()) @map("created_at")

  ride      Ride     @relation(fields: [rideId], references: [id])
  fromUser  User     @relation("ReviewFrom", fields: [fromId], references: [id])
  toUser    User     @relation("ReviewTo", fields: [toId], references: [id])

  @@map("reviews")
}

model Settlement {
  id           String           @id @default(uuid()) @map("id")
  rideId       String           @map("ride_id")
  passengerId  String           @map("passenger_id")
  amount       Int              @map("amount")
  driverAmount Int              @map("driver_amount")
  platformFee  Int              @map("platform_fee")
  status       SettlementStatus @default(PENDING) @map("status")
  dueDate      DateTime?        @map("due_date")
  paidAt       DateTime?        @map("paid_at")
  createdAt    DateTime         @default(now()) @map("created_at")

  ride         Ride             @relation(fields: [rideId], references: [id])

  @@map("settlements")
}

model PaymentMethod {
  id         String      @id @default(uuid()) @map("id")
  userId     String      @map("user_id")
  type       PaymentType @map("type")
  billingKey String?     @map("billing_key")
  alias      String?     @map("alias")
  isDefault  Boolean     @default(false) @map("is_default")
  createdAt  DateTime    @default(now()) @map("created_at")

  user       User        @relation(fields: [userId], references: [id])

  @@map("payment_methods")
}

model ChatRoom {
  id        String    @id @default(uuid()) @map("id")
  rideId    String    @map("ride_id")
  createdAt DateTime  @default(now()) @map("created_at")

  ride      Ride      @relation(fields: [rideId], references: [id])
  messages  Message[]

  @@map("chat_rooms")
}

model Message {
  id        String      @id @default(uuid()) @map("id")
  roomId    String      @map("room_id")
  senderId  String      @map("sender_id")
  content   String      @map("content")
  type      MessageType @default(TEXT) @map("type")
  createdAt DateTime    @default(now()) @map("created_at")

  room      ChatRoom    @relation(fields: [roomId], references: [id])
  sender    User        @relation(fields: [senderId], references: [id])

  @@map("messages")
}
```

**Step 2: 검증**

Run: `cd backend && npm run prisma:generate`
Expected: 성공

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): reviews, settlements, payment_methods, chat_rooms, messages 테이블 작성"
```

---

### Task B7: 마이그레이션 실행 + 시드 데이터

**Objective:** Prisma 마이그레이션으로 DB 테이블 생성 + 시드 데이터

**Files:**
- Create: `backend/prisma/seed.ts`
- Modify: `backend/package.json` (seed 스크립트 추가)

**Step 1: 마이그레이션 실행**

Run: `cd backend && npx prisma migrate dev --name init`
Expected: 마이그레이션 파일 생성 + DB 테이블 생성

**Step 2: 시드 데이터 작성**

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  const driver = await prisma.user.upsert({
    where: { email: 'driver@test.com' },
    update: {},
    create: {
      email: 'driver@test.com',
      name: '박준서',
      phone: '010-1234-5678',
      provider: 'kakao',
      providerId: 'test-driver-001',
      role: 'driver',
      rating: 4.8,
      rideCount: 42,
      phoneVerified: true,
      vehicles: {
        create: { model: '아반떼', color: '흰색', plate: '12가 3456', capacity: 4 },
      },
    },
  });

  const passenger = await prisma.user.upsert({
    where: { email: 'passenger@test.com' },
    update: {},
    create: {
      email: 'passenger@test.com',
      name: '이민수',
      phone: '010-8765-4321',
      provider: 'google',
      providerId: 'test-passenger-001',
      role: 'passenger',
      rating: 4.5,
      rideCount: 28,
      phoneVerified: true,
    },
  });

  console.log({ driver, passenger });
}

main().catch(console.error).finally(() => prisma.$disconnect());
```

**Step 3: package.json에 seed 스크립트 추가**

```json
"prisma": {
  "seed": "ts-node --compiler-options {\"module\":\"commonjs\"} prisma/seed.ts"
}
```

**Step 4: 시드 실행**

Run: `cd backend && npx prisma db seed`
Expected: 시드 데이터 삽입 성공

**Step 5: 커밋**

```bash
git add prisma/seed.ts package.json prisma/migrations/
git commit -m "feat(db): 마이그레이션 실행 및 시드 데이터 작성"
```

---

### Task B8: GraphQL 스키마 — 인증 타입 정의

**Objective:** 인증 관련 GraphQL 타입을 schema.graphql에 추가

**Files:**
- Modify: `backend/src/graphql/schema.graphql`

**Step 1: 인증 타입 추가** (상세 스키마는 `docs/api/AUTH.md` 참조)

```graphql
type AuthPayload {
  accessToken: String!
  refreshToken: String!
  user: User!
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

type Mutation {
  login(input: LoginInput!): AuthPayload!
  refreshToken(token: String!): AuthPayload!
  updateProfile(input: UpdateProfileInput!): User!
}

type Query {
  me: User!
}
```

**Step 2: codegen 실행**

Run: `cd backend && npm run codegen`
Expected: `src/graphql/generated/schema-types.ts` 업데이트

**Step 3: 프론트엔드 codegen도 실행** (양쪽 schema 공유)

Run: `cd frontend && npm run codegen`
Expected: 프론트 generated 타입 업데이트

**Step 4: 커밋**

```bash
git add src/graphql/schema.graphql src/graphql/generated/
git commit -m "feat(auth): 인증 GraphQL 스키마 타입 정의"
```

---

### Task B9: 인증 — AuthService 유닛 테스트 (Red)

**Objective:** 인증 서비스 유닛 테스트 먼저 작성

**Files:**
- Create: `backend/src/modules/auth/auth.service.spec.ts`

**테스트 케이스** (`docs/api/AUTH.md` 기반):

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | A1 | 카카오 기존 유저 로그인 → 토큰 발급 |
| 2 | A2 | 구글 신규 유저 자동 가입 → 토큰 발급 |
| 3 | E1 | 빈 oauthToken → UNAUTHORIZED |
| 4 | E2 | 지원하지 않는 provider → BAD_REQUEST |
| 5 | E3 | 만료된 소셜 토큰 → UNAUTHORIZED |
| 6 | A1(refresh) | 유효한 리프레시 토큰 → 새 토큰 쌍 |
| 7 | E1(refresh) | 만료된 리프레시 토큰 → UNAUTHORIZED |
| 8 | E2(refresh) | 변조된 토큰 → UNAUTHORIZED |

**Step 1: 테스트 작성 → Step 2: 실패 확인**

Run: `cd backend && npm run test -- auth.service.spec`
Expected: FAIL — AuthService 클래스가 아직 없음

---

### Task B10: 인증 — AuthService 구현 (Green)

**Objective:** 인증 서비스 최소 구현 + AuthResolver + AuthModule + Guard

**Files:**
- Create: `backend/src/modules/auth/auth.service.ts`
- Create: `backend/src/modules/auth/auth.module.ts`
- Create: `backend/src/modules/auth/auth.resolver.ts`
- Create: `backend/src/modules/auth/dto/login.input.ts`
- Create: `backend/src/modules/auth/dto/update-profile.input.ts`
- Create: `backend/src/modules/auth/guards/auth.guard.ts`
- Modify: `backend/package.json` (@nestjs/jwt, @nestjs/passport 추가)
- Modify: `backend/src/app.module.ts` (AuthModule 추가)

**Step 1: 패키지 설치**

Run: `cd backend && npm install @nestjs/jwt @nestjs/passport passport passport-jwt && npm install -D @types/passport-jwt`

**Step 2: AuthService 구현** (Task B9 테스트 통과하는 최소 코드)

**Step 3: 테스트 통과 확인**

Run: `cd backend && npm run test -- auth.service.spec`
Expected: PASS

**Step 4: AuthResolver + AuthModule + Guard 구현**

**Step 5: app.module.ts에 AuthModule 추가**

**Step 6: 전체 테스트**

Run: `cd backend && npm run test && npm run test:e2e`
Expected: PASS

**Step 7: 커밋**

```bash
git add src/modules/auth/ package.json src/app.module.ts
git commit -m "feat(auth): 소셜 로그인 인증 서비스 구현"
```

---

## 프론트엔드 작업 (FE #2 → #3 → #4)

### Task F1: shadcn/ui 초기화 + 필수 컴포넌트 설치

**Objective:** shadcn/ui CLI로 초기 설정 + Button, Card, Input, Dialog, Avatar 설치

**Files:**
- Modify: `frontend/package.json`
- Create: `frontend/components.json`
- Create: `frontend/src/components/ui/button.tsx` (shadcn 자동 생성)
- Create: `frontend/src/components/ui/card.tsx`
- Create: `frontend/src/components/ui/input.tsx`
- Create: `frontend/src/components/ui/dialog.tsx`
- Create: `frontend/src/components/ui/avatar.tsx`

**Step 1: shadcn/ui init**

Run: `cd frontend && npx shadcn@latest init -d`

**Step 2: 컴포넌트 설치**

Run: `cd frontend && npx shadcn@latest add button card input dialog avatar`

**Step 3: 검증**

Run: `cd frontend && npm run build`
Expected: 빌드 성공

**Step 4: 커밋**

```bash
git add components.json src/components/ui/ package.json
git commit -m "feat(ui): shadcn/ui 초기화 및 필수 컴포넌트 설치"
```

---

### Task F2: Tailwind 디자인 토큰 적용

**Objective:** DESIGN_SYSTEM.md 컬러/타포 토큰을 Tailwind CSS 4에 반영

**Files:**
- Modify: `frontend/src/app/globals.css`

**Step 1: CSS 커스텀 프로퍼티로 토큰 정의**

```css
@import "tailwindcss";

@theme {
  --color-primary: #2563EB;
  --color-primary-light: #3B82F6;
  --color-primary-dark: #1D4ED8;
  --color-secondary: #10B981;
  --color-warning: #F59E0B;
  --color-danger: #EF4444;

  --font-family-sans: 'Pretendard', system-ui, sans-serif;

  --font-size-h1: 28px;
  --font-size-h2: 24px;
  --font-size-h3: 20px;
  --font-size-body: 16px;
  --font-size-caption: 14px;
  --font-size-small: 12px;
}
```

**Step 2: 검증**

Run: `cd frontend && npm run build`
Expected: 빌드 성공

**Step 3: 커밋**

```bash
git add src/app/globals.css
git commit -m "feat(design): Tailwind CSS 4 디자인 토큰 적용"
```

---

### Task F3: Pretendard 폰트 설정

**Objective:** Pretendard 웹폰트 글로벌 적용

**Files:**
- Modify: `frontend/src/app/layout.tsx`
- Create: `frontend/src/fonts/PretendardVariable.woff2`

**Step 1: 폰트 다운로드**

Run: `mkdir -p frontend/src/fonts && curl -L -o frontend/src/fonts/PretendardVariable.woff2 https://cdn.jsdelivr.net/gh/orioncactus/pretendard/public/variable/PretendardVariable.woff2`

**Step 2: next/font/local로 Pretendard 설정**

```typescript
// src/app/layout.tsx
import localFont from 'next/font/local';

const pretendard = localFont({
  src: '../fonts/PretendardVariable.woff2',
  display: 'swap',
  variable: '--font-pretendard',
});
```

**Step 3: 검증 → Step 4: 커밋**

```bash
git add src/fonts/ src/app/layout.tsx
git commit -m "feat(design): Pretendard 폰트 글로벌 적용"
```

---

### Task F4: Ridy 커스텀 컴포넌트 — MatchingCard

**Objective:** 카풀 매칭 카드 컴포넌트 구현

**Files:**
- Create: `frontend/src/components/ridy/matching-card.tsx`
- Create: `frontend/src/components/ridy/matching-card.test.tsx`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | 차주 이름 렌더링 | "박준서" 텍스트 표시 |
| 2 | 평점/운행 횟수 | "4.8 (42회)" 표시 |
| 3 | 요금 포맷팅 | 5000 → "5,000원/인" |
| 4 | 잔여 좌석 | "2/3석" 표시 |
| 5 | 출발→도착 경로 | "강남역 08:30 → 수원역 09:20" |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(ui): MatchingCard 커스텀 컴포넌트 구현"
```

---

### Task F5: Ridy 커스텀 컴포넌트 — BottomNavigation

**Objective:** 하단 내비게이션 바 구현

**Files:**
- Create: `frontend/src/components/ridy/bottom-navigation.tsx`
- Create: `frontend/src/components/ridy/bottom-navigation.test.tsx`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | 4개 탭 렌더링 | 홈, 검색, 채팅, 프로필 |
| 2 | 활성 탭 스타일 | aria-current="page" |
| 3 | 탭 클릭 | onTabChange 콜백 호출 |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(ui): BottomNavigation 컴포넌트 구현"
```

---

### Task F6: Ridy 커스텀 컴포넌트 — RouteInput

**Objective:** 출발지/도착지 입력 컴포넌트 구현

**Files:**
- Create: `frontend/src/components/ridy/route-input.tsx`
- Create: `frontend/src/components/ridy/route-input.test.tsx`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | 출발지/도착지 입력 필드 | 2개 Input 렌더링 |
| 2 | 스왑 버튼 | 출발지↔도착지 교환 |
| 3 | 현재 위치 버튼 | GPS 권한 요청 |
| 4 | 빈 출발지 | 에러 메시지 표시 |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(ui): RouteInput 출발지/도착지 입력 컴포넌트 구현"
```

---

### Task F7: 로그인 화면 — 스플래시 + 소셜 로그인

**Objective:** WIREFRAMES.md 기반 스플래시/로그인 화면 구현

**Files:**
- Create: `frontend/src/app/(auth)/login/page.tsx`
- Create: `frontend/src/app/(auth)/login/login.test.tsx`
- Create: `frontend/src/components/ridy/social-login-buttons.tsx`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | Ridy 로고 + 슬로건 | "Ridy", "함께 타는 길" |
| 2 | 소셜 로그인 버튼 3개 | 카카오, 구글, Apple |
| 3 | 카카오 버튼 클릭 | 카카오 OAuth 리다이렉트 |
| 4 | 구글 버튼 클릭 | 구글 OAuth 리다이렉트 |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(auth): 스플래시 + 소셜 로그인 화면 구현"
```

---

### Task F8: 로그인 화면 — 프로필 설정 (온보딩)

**Objective:** 신규 유저 프로필 설정 폼 + 차주 토글

**Files:**
- Create: `frontend/src/app/(auth)/onboarding/page.tsx`
- Create: `frontend/src/app/(auth)/onboarding/onboarding.test.tsx`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | 이름 입력 필드 | 필수 입력 |
| 2 | 전화번호 입력 | 형식 검증 |
| 3 | 차주 토글 | role=driver 선택 시 차량 정보 폼 표시 |
| 4 | 차량 정보 입력 | 모델, 색상, 번호판, 좌석 수 |
| 5 | 완료 버튼 | 필수 필드 없으면 비활성화 |
| 6 | 제출 | updateProfile mutation 호출 |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(auth): 온보딩 프로필 설정 화면 구현"
```

---

### Task F9: 인증 상태 관리 — 클라이언트 토큰 + 가드

**Objective:** 클라이언트 토큰 저장 + 인증 미들웨어 가드

**Files:**
- Create: `frontend/src/lib/auth.ts`
- Create: `frontend/src/lib/auth.test.ts`
- Create: `frontend/src/middleware.ts`

**테스트 케이스:**

| # | 케이스 | 설명 |
|---|--------|------|
| 1 | 토큰 저장 | setToken → localStorage에 저장 |
| 2 | 토큰 조회 | getToken → 저장된 토큰 반환 |
| 3 | 토큰 삭제 | clearToken → localStorage 초기화 |
| 4 | 인증 여부 | isAuthenticated → 토큰 있으면 true |
| 5 | 미들웨어 — 비인증 접근 | /home 접근 시 /login 리다이렉트 |
| 6 | 미들웨어 — 인증된 상태에서 로그인 | /login 접근 시 / 리다이렉트 |

**TDD 사이클 → 커밋:**

```bash
git commit -m "feat(auth): 인증 상태 관리 및 미들웨어 가드 구현"
```

---

## 전체 검증

### Task V1: 백엔드 전체 파이프라인

Run: `cd backend && npm run test && npm run test:e2e && npm run prisma:generate && npm run build && npm run lint`
Expected: 전부 PASS

### Task V2: 프론트엔드 전체 파이프라인

Run: `cd frontend && npm run test && npm run lint && npm run build`
Expected: 전부 PASS

### Task V3: PR 생성 + 머지 + 브랜치 정리

```bash
# Backend
cd backend && git checkout -b feat/2-prisma-schema-auth
gh pr create --repo project-ridy/backend --base main --head feat/2-prisma-schema-auth
# 머지 후 브랜치 삭제

# Frontend
cd frontend && git checkout -b feat/2-design-system-login
gh pr create --repo project-ridy/frontend --base main --head feat/2-design-system-login
# 머지 후 브랜치 삭제
```

---

## 작업 순서 요약

| 순서 | Task | 레포 | 이슈 | 예상 시간 |
|------|-------|------|------|-----------|
| 1 | B1~B6 | backend | #2 | 20분 |
| 2 | B7 | backend | #2 | 10분 |
| 3 | B8 | backend | #3 | 5분 |
| 4 | B9~B10 | backend | #3, #4 | 20분 |
| 5 | F1~F3 | frontend | #2 | 15분 |
| 6 | F4~F6 | frontend | #2 | 15분 |
| 7 | F7~F9 | frontend | #4 | 20분 |
| 8 | V1~V3 | both | — | 10분 |

**총 예상 시간:** ~115분 (서브에이전트 병렬 시 ~60분)
