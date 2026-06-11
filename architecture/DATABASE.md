# Ridy — 데이터베이스 스키마 (기업 단위 폐쇄형 카풀 서비스)

> Ridy는 기업 단위 폐쇄형 카풀 서비스입니다. 모든 유저는 반드시 소속 기업(Company)이 있어야 하며,
> 카풀 매칭은 **같은 기업(company_id)에 소속된 구성원 간에만** 이루어집니다.

---

## ERD

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│    companies      │      │   invite_codes   │      │      users       │
├──────────────────┤      ├──────────────────┤      ├──────────────────┤
│ id (PK)          │──┐   │ id (PK)          │      │ id (PK)          │
│ name             │  │   │ company_id (FK)   │──┐   │ company_id (FK)  │──┐
│ invite_code      │  │   │ code             │  │   │ employee_id      │  │
│ domain           │  │   │ created_by (FK)  │  │   │ email            │  │
│ admin_id (FK)    │  │   │ max_uses         │  │   │ name             │  │
│ max_members      │  │   │ current_uses     │  │   │ phone            │  │
│ plan             │  │   │ expires_at       │  │   │ image_url        │  │
│ created_at       │  │   │ is_active        │  │   │ provider         │  │
│ updated_at       │  │   └──────────────────┘  │   │ provider_id      │  │
└──────────────────┘  │                         │   │ role             │  │
                      └─────────────────────────┘   │ rating           │  │
                                                    │ ride_count       │  │
┌─────────────┐                                     │ phone_verified   │  │
│  vehicles   │                                     │ created_at       │  │
├─────────────┤                                     │ updated_at       │  │
│ id (PK)     │                                     └────────┬─────────┘  │
│ user_id(FK) │──────────────────────────────────────────────┘            │
│ model       │                                                          │
│ color       │                                                          │
│ plate       │                                                          │
│ capacity    │     ┌──────────────────┐     ┌─────────────────┐         │
│ created_at  │     │      rides       │     │  ride_requests  │         │
└─────────────┘     ├──────────────────┤     ├─────────────────┤         │
                    │ id (PK)          │     │ id (PK)         │         │
┌─────────────┐     │ company_id (FK)  │──┐  │ ride_id (FK)    │         │
│   reviews   │     │ driver_id (FK)   │  │  │ passenger_id(FK)│         │
├─────────────┤     │ departure_lat    │  │  │ pickup_lat      │         │
│ id (PK)     │     │ departure_lng    │  │  │ pickup_lng      │         │
│ ride_id(FK) │     │ departure_addr   │  │  │ pickup_addr     │         │
│ from_id(FK) │     │ arrival_lat      │  │  │ message         │         │
│ to_id (FK)  │     │ arrival_lng      │  │  │ status          │         │
│ rating      │     │ arrival_addr     │  │  │ created_at      │         │
│ comment     │     │ departure_time   │  │  │ updated_at      │         │
│ created_at  │     │ available_seats  │  │  └─────────────────┘         │
└─────────────┘     │ fare             │  │                               │
                    │ recurring_days   │  │  ┌─────────────────┐          │
┌──────────────────┐│ recurring_end    │  │  │   chat_rooms    │          │
│  payment_methods ││ preferences      │  │  ├─────────────────┤          │
├──────────────────┤│ status           │  │  │ id (PK)         │          │
│ id (PK)          ││ created_at       │  │  │ ride_id (FK)    │          │
│ user_id (FK)     ││ updated_at       │  │  │ created_at      │          │
│ type             │└──────────────────┘  │  └────────┬────────┘          │
│ billing_key      │                      │           │                   │
│ alias            │     ┌──────────────────┐  ┌───────▼─────────┐       │
│ is_default       │     │   settlements    │  │   messages       │       │
│ created_at       │     ├──────────────────┤  ├─────────────────┤       │
└──────────────────┘     │ id (PK)          │  │ id (PK)         │       │
                         │ ride_id (FK)     │  │ room_id (FK)    │       │
                         │ passenger_id(FK) │  │ sender_id (FK)  │       │
                         │ amount           │  │ content         │       │
                         │ driver_amount    │  │ type            │       │
                         │ platform_fee     │  │ created_at      │       │
                         │ status           │  └─────────────────┘       │
                         │ due_date         │                            │
                         │ paid_at          │                            │
                         │ created_at       │                            │
                         └──────────────────┘                            │
                                                                        │
             같은 company_id 끼리만 매칭 ◀────────────────────────────────┘
```

---

## Prisma 스키마

```prisma
// ============================================================
// 기업 (Company) — 폐쇄형 카풀의 최상위 격리 단위
// ============================================================
model Company {
  id          String   @id @default(uuid())
  name        String   @db.VarChar(100)                    // 기업명
  inviteCode  String   @unique @map("invite_code") @db.VarChar(20) // 기업 고유 초대 코드
  domain      String   @db.VarChar(100)                    // 이메일 도메인 (검증용, 예: "acme.co.kr")
  adminId     String   @map("admin_id")                    // 최초 관리자 user.id
  maxMembers  Int      @default(50) @map("max_members")    // 최대 구성원 수
  plan        Plan     @default(FREE)                      // 요금제

  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt      @map("updated_at")

  // Relations
  admin       User        @relation("CompanyAdmin", fields: [adminId], references: [id])
  users       User[]
  inviteCodes InviteCode[]
  rides       Ride[]
  settlements Settlement[]

  @@map("companies")
}

enum Plan {
  FREE
  PRO
  ENTERPRISE
}

// ============================================================
// 초대 코드 (InviteCode) — 기업 관리자가 생성, 신규 구성원 가입용
// ============================================================
model InviteCode {
  id          String    @id @default(uuid())
  companyId   String    @map("company_id")
  code        String    @db.VarChar(6)                      // 6자리 영숽�드
  createdBy   String    @map("created_by")
  maxUses     Int       @default(10) @map("max_uses")       // 최대 사용 횟수
  currentUses Int       @default(0) @map("current_uses")    // 현재 사용 횟수
  expiresAt   DateTime? @map("expires_at")                  // 만료일 (null=무제한)
  isActive    Boolean   @default(true) @map("is_active")

  // Relations
  company     Company   @relation(fields: [companyId], references: [id])
  creator     User      @relation("InviteCodeCreator", fields: [createdBy], references: [id])

  @@unique([companyId, code])
  @@map("invite_codes")
}

// ============================================================
// 사용자 (User) — 반드시 하나의 기업에 소속
// ============================================================
model User {
  id            String   @id @default(uuid())
  companyId     String   @map("company_id")                 // 필수: 소속 기업
  employeeId    String?  @map("employee_id") @db.VarChar(50) // 선택: 사번
  email         String   @unique @db.VarChar(255)
  name          String   @db.VarChar(100)
  phone         String?  @db.VarChar(20)
  imageUrl      String?  @map("image_url") @db.Text
  provider      Provider?
  providerId    String?  @map("provider_id") @db.VarChar(255)
  role          Role     @default(PASSENGER)
  rating        Decimal  @default(0.0) @db.Decimal(2, 1)
  rideCount     Int      @default(0) @map("ride_count")
  phoneVerified Boolean  @default(false) @map("phone_verified")

  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt      @map("updated_at")

  // Relations
  company       Company        @relation(fields: [companyId], references: [id])
  vehicles      Vehicle[]
  ridesAsDriver Ride[]         @relation("RideDriver")
  rideRequests  RideRequest[]
  reviewsGiven  Review[]       @relation("ReviewFrom")
  reviewsRecv   Review[]       @relation("ReviewTo")
  payments      PaymentMethod[]
  settlements   Settlement[]
  createdInviteCodes InviteCode[] @relation("InviteCodeCreator")
  adminOf       Company?       @relation("CompanyAdmin", fields: [id], references: [adminId])

  @@unique([companyId, employeeId]) // 같은 기업 내 사번 중복 방지
  @@map("users")
}

enum Provider {
  KAKAO
  GOOGLE
  APPLE
}

enum Role {
  PASSENGER
  DRIVER
  BOTH
  ADMIN   // 기업 관리자
}

// ============================================================
// 차량 (Vehicle)
// ============================================================
model Vehicle {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  model     String   @db.VarChar(100)
  color     String?  @db.VarChar(50)
  plate     String   @db.VarChar(20)
  capacity  Int      @default(4)

  createdAt DateTime @default(now()) @map("created_at")

  user      User     @relation(fields: [userId], references: [id])

  @@map("vehicles")
}

// ============================================================
// 카풀 운행 (Ride) — 같은 기업 내에서만 생성/매칭
// ============================================================
model Ride {
  id              String       @id @default(uuid())
  companyId       String       @map("company_id")              // ★ 소속 기업 (격리 키)
  driverId        String       @map("driver_id")
  departureLat    Decimal      @map("departure_lat") @db.Decimal(10, 7)
  departureLng    Decimal      @map("departure_lng") @db.Decimal(10, 7)
  departureAddr   String?      @map("departure_addr") @db.VarChar(255)
  arrivalLat      Decimal      @map("arrival_lat") @db.Decimal(10, 7)
  arrivalLng      Decimal      @map("arrival_lng") @db.Decimal(10, 7)
  arrivalAddr     String?      @map("arrival_addr") @db.VarChar(255)
  departureTime   DateTime     @map("departure_time")
  availableSeats  Int          @map("available_seats")
  fare            Int?                                               // 1인 요금 (원)
  recurringDays   String?      @map("recurring_days") @db.VarChar(50)  // MON,TUE,...
  recurringEnd    DateTime?    @map("recurring_end")                     // 정기 카풀 종료일
  preferences     Json?                                             // smoking, pet, luggage 등
  status          RideStatus   @default(OPEN)

  createdAt       DateTime     @default(now()) @map("created_at")
  updatedAt       DateTime     @updatedAt      @map("updated_at")

  // Relations
  company         Company      @relation(fields: [companyId], references: [id])
  driver          User         @relation("RideDriver", fields: [driverId], references: [id])
  requests        RideRequest[]
  reviews         Review[]
  settlements     Settlement[]
  chatRoom        ChatRoom?

  @@map("rides")
}

enum RideStatus {
  OPEN
  MATCHED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

// ============================================================
// 카풀 요청 (RideRequest) — 같은 기업 내에서만 매칭
// ============================================================
model RideRequest {
  id           String          @id @default(uuid())
  rideId       String          @map("ride_id")
  passengerId  String          @map("passenger_id")
  pickupLat    Decimal?        @map("pickup_lat") @db.Decimal(10, 7)
  pickupLng    Decimal?        @map("pickup_lng") @db.Decimal(10, 7)
  pickupAddr   String?         @map("pickup_addr") @db.VarChar(255)
  message      String?         @db.Text
  status       RequestStatus   @default(PENDING)

  createdAt    DateTime        @default(now()) @map("created_at")
  updatedAt    DateTime        @updatedAt      @map("updated_at")

  // Relations
  ride         Ride            @relation(fields: [rideId], references: [id])
  passenger    User            @relation(fields: [passengerId], references: [id])

  @@map("ride_requests")
}

enum RequestStatus {
  PENDING
  ACCEPTED
  REJECTED
  CANCELLED
}

// ============================================================
// 리뷰 (Review)
// ============================================================
model Review {
  id        String   @id @default(uuid())
  rideId    String   @map("ride_id")
  fromId    String   @map("from_id")
  toId      String   @map("to_id")
  rating    Int                                          // 1~5
  comment   String?  @db.Text

  createdAt DateTime @default(now()) @map("created_at")

  ride      Ride     @relation(fields: [rideId], references: [id])
  fromUser  User     @relation("ReviewFrom", fields: [fromId], references: [id])
  toUser    User     @relation("ReviewTo", fields: [toId], references: [id])

  @@map("reviews")
}

// ============================================================
// 정산 (Settlement)
// ============================================================
model Settlement {
  id           String          @id @default(uuid())
  rideId       String          @map("ride_id")
  passengerId  String          @map("passenger_id")
  amount       Int                                       // 승객 결제 금액
  driverAmount Int             @map("driver_amount")      // 차주 수령 금액
  platformFee  Int             @map("platform_fee")       // 플랫폼 수수료
  status       SettlementStatus @default(PENDING)
  dueDate      DateTime?       @map("due_date")
  paidAt       DateTime?       @map("paid_at")

  createdAt    DateTime        @default(now()) @map("created_at")

  ride         Ride            @relation(fields: [rideId], references: [id])
  passenger    User            @relation(fields: [passengerId], references: [id])
  company      Company?        @relation(fields: [companyId], references: [id]) // 선택: 기업별 정산 조회

  @@map("settlements")
}

enum SettlementStatus {
  PENDING
  COMPLETED
  FAILED
  REFUNDED
}

// ============================================================
// 결제 수단 (PaymentMethod)
// ============================================================
model PaymentMethod {
  id          String   @id @default(uuid())
  userId      String   @map("user_id")
  type        String   @db.VarChar(30)                     // card, kakao_pay, toss_pay ...
  billingKey  String?  @map("billing_key") @db.VarChar(255)
  alias       String?  @db.VarChar(50)
  isDefault   Boolean  @default(false) @map("is_default")

  createdAt   DateTime @default(now()) @map("created_at")

  user        User     @relation(fields: [userId], references: [id])

  @@map("payment_methods")
}

// ============================================================
// 채팅방 (ChatRoom)
// ============================================================
model ChatRoom {
  id        String    @id @default(uuid())
  rideId    String    @unique @map("ride_id")

  createdAt DateTime  @default(now()) @map("created_at")

  ride      Ride      @relation(fields: [rideId], references: [id])
  messages  Message[]

  @@map("chat_rooms")
}

// ============================================================
// 메시지 (Message)
// ============================================================
model Message {
  id        String   @id @default(uuid())
  roomId    String   @map("room_id")
  senderId  String   @map("sender_id")
  content   String   @db.Text
  type      MessageType @default(TEXT)

  createdAt DateTime @default(now()) @map("created_at")

  room      ChatRoom @relation(fields: [roomId], references: [id])
  sender    User     @relation(fields: [senderId], references: [id])

  @@map("messages")
}

enum MessageType {
  TEXT
  IMAGE
  SYSTEM
}
```

---

## 주요 테이블 상세

### companies

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 기업 고유 ID |
| name | VARCHAR(100) | NOT NULL | 기업명 |
| invite_code | VARCHAR(20) | UNIQUE | 기업 고유 초대 코드 (가입 시 사용) |
| domain | VARCHAR(100) | | 이메일 도메인 (예: `acme.co.kr`). 도메인 기반 자동 소속 검증용 |
| admin_id | UUID | FK → users | 기업 관리자 (최초 생성자) |
| max_members | INT | DEFAULT 50 | 최대 구성원 수 (요금제별 상이) |
| plan | ENUM | DEFAULT FREE | FREE / PRO / ENTERPRISE |
| created_at | TIMESTAMP | | 생성일 |
| updated_at | TIMESTAMP | | 수정일 |

**plan별 기본 max_members**:
| plan | max_members | 비고 |
|---|---|---|
| FREE | 50 | 무료 |
| PRO | 200 | 월 구독 |
| ENTERPRISE | 무제한 | 커스텀 계약 |

### invite_codes

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 초대 코드 ID |
| company_id | UUID | FK → companies, NOT NULL | 소속 기업 |
| code | VARCHAR(6) | NOT NULL | 6자리 영숫자 코드 |
| created_by | UUID | FK → users | 코드 생성자 (관리자) |
| max_uses | INT | DEFAULT 10 | 최대 사용 가능 횟수 |
| current_uses | INT | DEFAULT 0 | 현재까지 사용 횟수 |
| expires_at | TIMESTAMP | | 만료일 (NULL = 무제한) |
| is_active | BOOLEAN | DEFAULT true | 활성 여부 |

**제약**: `(company_id, code)` 복합 유니크 — 같은 기업 내 코드 중복 방지

### users

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 유저 고유 ID |
| **company_id** | **UUID** | **FK → companies, NOT NULL** | **소속 기업 (필수)** |
| **employee_id** | **VARCHAR(50)** | **선택** | **사번 (같은 기업 내 유니크)** |
| email | VARCHAR(255) | UNIQUE | 이메일 |
| name | VARCHAR(100) | NOT NULL | 이름 |
| phone | VARCHAR(20) | | 전화번호 |
| image_url | TEXT | | 프로필 이미지 |
| provider | ENUM | | kakao, google, apple |
| provider_id | VARCHAR(255) | | 소셜 ID |
| role | ENUM | DEFAULT PASSENGER | PASSENGER, DRIVER, BOTH, ADMIN |
| rating | DECIMAL(2,1) | DEFAULT 0.0 | 평균 평점 |
| ride_count | INT | DEFAULT 0 | 운행 횟수 |
| phone_verified | BOOLEAN | DEFAULT false | 휴대폰 인증 여부 |
| created_at | TIMESTAMP | | 생성일 |
| updated_at | TIMESTAMP | | 수정일 |

**제약**: `(company_id, employee_id)` 복합 유니크 — 같은 기업 내 사번 중복 방지

### rides

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 운행 ID |
| **company_id** | **UUID** | **FK → companies, NOT NULL** | **소속 기업 (격리 키)** |
| driver_id | UUID | FK → users | 차주 |
| departure_lat | DECIMAL(10,7) | NOT NULL | 출발지 위도 |
| departure_lng | DECIMAL(10,7) | NOT NULL | 출발지 경도 |
| departure_addr | VARCHAR(255) | | 출발지 주소 |
| arrival_lat | DECIMAL(10,7) | NOT NULL | 도착지 위도 |
| arrival_lng | DECIMAL(10,7) | NOT NULL | 도착지 경도 |
| arrival_addr | VARCHAR(255) | | 도착지 주소 |
| departure_time | TIMESTAMP | NOT NULL | 출발 시간 |
| available_seats | INT | NOT NULL | 가능 좌석 |
| fare | INT | | 1인 요금 (원) |
| recurring_days | VARCHAR(50) | | 요일 (MON,TUE,...) |
| recurring_end | DATE | | 정기 카풀 종료일 |
| preferences | JSONB | | smoking, pet, luggage 등 |
| status | ENUM | | OPEN, MATCHED, IN_PROGRESS, COMPLETED, CANCELLED |
| created_at | TIMESTAMP | | |
| updated_at | TIMESTAMP | | |

### ride_requests

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 요청 ID |
| ride_id | UUID | FK → rides | 대상 운행 |
| passenger_id | UUID | FK → users | 탑승자 |
| pickup_lat | DECIMAL(10,7) | | 픽업 위도 |
| pickup_lng | DECIMAL(10,7) | | 픽업 경도 |
| pickup_addr | VARCHAR(255) | | 픽업 주소 |
| message | TEXT | | 요청 메시지 |
| status | ENUM | | PENDING, ACCEPTED, REJECTED, CANCELLED |
| created_at | TIMESTAMP | | |
| updated_at | TIMESTAMP | | |

### reviews

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 리뷰 ID |
| ride_id | UUID | FK → rides | 운행 |
| from_id | UUID | FK → users | 작성자 |
| to_id | UUID | FK → users | 대상자 |
| rating | INT | 1~5 | 평점 |
| comment | TEXT | | 코멘트 |
| created_at | TIMESTAMP | | |

### settlements

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 정산 ID |
| ride_id | UUID | FK → rides | 운행 |
| passenger_id | UUID | FK → users | 승객 |
| amount | INT | | 승객 결제 금액 |
| driver_amount | INT | | 차주 수령 금액 |
| platform_fee | INT | | 플랫폼 수수료 |
| status | ENUM | | PENDING, COMPLETED, FAILED, REFUNDED |
| due_date | TIMESTAMP | | 정산 예정일 |
| paid_at | TIMESTAMP | | 정산 완료일 |
| created_at | TIMESTAMP | | |

### payment_methods

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 결제수단 ID |
| user_id | UUID | FK → users | 소유자 |
| type | VARCHAR(30) | | card, kakao_pay, toss_pay ... |
| billing_key | VARCHAR(255) | | 빌링 키 |
| alias | VARCHAR(50) | | 별칭 |
| is_default | BOOLEAN | DEFAULT false | 기본 결제수단 여부 |
| created_at | TIMESTAMP | | |

### chat_rooms

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 채팅방 ID |
| ride_id | UUID | FK → rides, UNIQUE | 연결된 운행 |
| created_at | TIMESTAMP | | |

### messages

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 메시지 ID |
| room_id | UUID | FK → chat_rooms | 채팅방 |
| sender_id | UUID | FK → users | 발신자 |
| content | TEXT | | 내용 |
| type | ENUM | | TEXT, IMAGE, SYSTEM |
| created_at | TIMESTAMP | | |

---

## 기업 격리 (Multi-tenancy) 규칙

### 매칭 시 같은 company_id 조건

모든 카풀 매칭 쿼리는 **반드시 같은 `company_id` 조건**을 포함해야 합니다.

```sql
-- ✅ 올바른 매칭 쿼리: 같은 기업 내에서만 매칭
SELECT r.*
FROM rides r
JOIN users u ON u.id = r.driver_id
WHERE r.company_id = :current_user_company_id
  AND r.status = 'OPEN'
  AND r.departure_time > NOW()
ORDER BY r.departure_time ASC;

-- ❌ 금지: company_id 없는 전체 검색
SELECT * FROM rides WHERE status = 'OPEN';  -- 절대 금지
```

**Prisma 예시**:
```typescript
// 같은 기업 내 운행 검색
const rides = await prisma.ride.findMany({
  where: {
    companyId: user.companyId,   // ★ 필수 조건
    status: 'OPEN',
    departureTime: { gt: new Date() },
  },
});

// 같은 기업 내 매칭 요청
const request = await prisma.rideRequest.create({
  data: {
    rideId: ride.id,
    passengerId: user.id,
    // ride.companyId === user.companyId 검증은 서비스 레이어에서 수행
  },
});
```

### 서비스 레이어 검증 체크리스트

모든 서비스 메서드는 아래 검증을 수행해야 합니다:

| 액션 | 검증 조건 |
|---|---|
| 운행 생성 | `driver.companyId`로 `ride.companyId` 설정 |
| 운행 검색 | `WHERE companyId = currentUser.companyId` |
| 탑승 요청 | `ride.companyId === passenger.companyId` |
| 채팅 참여 | `ride.companyId === user.companyId` |
| 리뷰 작성 | `ride.companyId === reviewer.companyId` |
| 정산 조회 | 관리자: 본인 기업 정산만 조회 가능 |

---

## 관리자 대시보드용 통계 뷰

기업 관리자(role=ADMIN)는 자사 대시보드에서 아래 통계를 조회할 수 있습니다.

### 1. 기업 활성 사용자 통계

```sql
CREATE OR REPLACE VIEW v_company_user_stats AS
SELECT
  c.id                AS company_id,
  c.name              AS company_name,
  COUNT(u.id)         AS total_users,
  COUNT(u.id) FILTER (WHERE u.role IN ('DRIVER', 'BOTH')) AS driver_count,
  COUNT(u.id) FILTER (WHERE u.role IN ('PASSENGER', 'BOTH')) AS passenger_count,
  AVG(u.rating)       AS avg_rating,
  SUM(u.ride_count)   AS total_rides
FROM companies c
LEFT JOIN users u ON u.company_id = c.id
GROUP BY c.id, c.name;
```

### 2. 기업 일별 카풀 통계

```sql
CREATE OR REPLACE VIEW v_company_ride_daily AS
SELECT
  company_id,
  DATE(departure_time)        AS ride_date,
  COUNT(*)                    AS total_rides,
  COUNT(*) FILTER (WHERE status = 'COMPLETED') AS completed_rides,
  COUNT(*) FILTER (WHERE status = 'CANCELLED') AS cancelled_rides,
  AVG(available_seats)        AS avg_available_seats,
  SUM(fare)                   AS total_fare
FROM rides
GROUP BY company_id, DATE(departure_time);
```

### 3. 기업 월별 정산 요약

```sql
CREATE OR REPLACE VIEW v_company_settlement_monthly AS
SELECT
  c.id                  AS company_id,
  c.name                AS company_name,
  DATE_TRUNC('month', s.created_at) AS settlement_month,
  COUNT(s.id)           AS total_settlements,
  SUM(s.amount)         AS total_amount,
  SUM(s.driver_amount)  AS total_driver_amount,
  SUM(s.platform_fee)   AS total_platform_fee,
  COUNT(s.id) FILTER (WHERE s.status = 'COMPLETED') AS completed_count,
  COUNT(s.id) FILTER (WHERE s.status = 'PENDING')   AS pending_count
FROM settlements s
JOIN rides r ON r.id = s.ride_id
JOIN companies c ON c.id = r.company_id
GROUP BY c.id, c.name, DATE_TRUNC('month', s.created_at);
```

### 4. 초대 코드 사용 현황

```sql
CREATE OR REPLACE VIEW v_invite_code_usage AS
SELECT
  ic.company_id,
  c.name          AS company_name,
  ic.code,
  ic.max_uses,
  ic.current_uses,
  ic.expires_at,
  ic.is_active,
  CASE
    WHEN ic.current_uses >= ic.max_uses THEN 'EXHAUSTED'
    WHEN ic.expires_at IS NOT NULL AND ic.expires_at < NOW() THEN 'EXPIRED'
    WHEN ic.is_active = false THEN 'DISABLED'
    ELSE 'ACTIVE'
  END              AS status
FROM invite_codes ic
JOIN companies c ON c.id = ic.company_id;
```

### 대시보드 API 엔드포인트 (예정)

| 엔드포인트 | 설명 | 사용 뷰 |
|---|---|---|
| `GET /admin/stats/users` | 기업 사용자 통계 | `v_company_user_stats` |
| `GET /admin/stats/rides/daily` | 일별 카풀 통계 | `v_company_ride_daily` |
| `GET /admin/stats/settlements/monthly` | 월별 정산 요약 | `v_company_settlement_monthly` |
| `GET /admin/invite-codes` | 초대 코드 현황 | `v_invite_code_usage` |

> **주의**: 모든 관리자 API는 요청자의 `company_id`로 자동 필터링되며, 타 기업 데이터 접근은 불가합니다.

---

## 가입 플로우

```
1. 사용자가 초대 코드 입력 (또는 이메일 도메인 자동 감지)
   ↓
2. invite_codes 테이블에서 코드 검증
   - is_active = true
   - current_uses < max_uses
   - expires_at IS NULL OR expires_at > NOW()
   ↓
3. companies.domain 과 사용자 이메일 도메인 일치 확인
   ↓
4. users 테이블에 company_id 설정하여 생성
   - employee_id (선택) 입력
   ↓
5. invite_codes.current_uses += 1
   ↓
6. 가입 완료 → 같은 기업 카풀 매칭 가능
```

---

## 인덱스 전략

```sql
-- 기업 격리 쿼리 최적화
CREATE INDEX idx_users_company_id ON users(company_id);
CREATE INDEX idx_rides_company_id_status ON rides(company_id, status);
CREATE INDEX idx_rides_company_id_departure ON rides(company_id, departure_time);
CREATE INDEX idx_ride_requests_passenger ON ride_requests(passenger_id);
CREATE INDEX idx_invite_codes_company_code ON invite_codes(company_id, code);
CREATE INDEX idx_settlements_ride_company ON settlements(ride_id);

-- 사번 검색
CREATE INDEX idx_users_company_employee ON users(company_id, employee_id);

-- 이메일 도메인 검증
CREATE INDEX idx_companies_domain ON companies(domain);
```
