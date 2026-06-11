# BE #2: Prisma 스키마 및 마이그레이션 구현 계획서

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** DATABASE.md 기반으로 Prisma 7 스키마 11개 모델 + 6개 ENUM을 작성하고, 마이그레이션 + 시드 데이터까지 완성

**Architecture:** 기업 단위 폐쇄형 카풀 — Company가 최상위 격리 단위, 모든 유저는 company_id 필수, 매칭/채팅/정산은 같은 회사 내에서만

**Tech Stack:** Prisma 7.8.0 + @prisma/adapter-pg + PostgreSQL + NestJS 11

**Issue:** project-ridy/backend #2

---

## 기능 케이스 분석

### 기능: Prisma 스키마 작성

### 설명
DATABASE.md에 정의된 11개 모델, 6개 ENUM을 Prisma 스키마로 구현. 모든 관계, 제약조건, 인덱스 포함.

### 정상 케이스 (A)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | `npx prisma validate` | 스키마 유효성 검사 통과 |
| A2 | `npx prisma generate` | PrismaClient 생성 성공 |
| A3 | `npx prisma migrate dev` | 마이그레이션 파일 생성 + DB 반영 |
| A4 | 시드 스크립트 실행 | 1개 회사 + 관리자 + 초대코드 + 차주 + 탑승자 데이터 생성 |
| A5 | NestJS 서버 기동 | PrismaService 정상 연결 |
| A6 | GraphQL health 쿼리 | 기존 기능 정상 동작 |

### 예외 케이스 (E)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | company_id 없는 User 생성 | Prisma 에러 (필수 필드) |
| E2 | 중복 email User 생성 | Prisma unique 제약 에러 |
| E3 | 중복 inviteCode Company 생성 | Prisma unique 제약 에러 |
| E4 | 같은 회사+사번 중복 | Prisma 복합 unique 제약 에러 |
| E5 | 같은 회사+코드 중복 InviteCode | Prisma 복합 unique 제약 에러 |
| E6 | DB 연결 실패 | PrismaService 에러 로깅 |

### 엣지 케이스 (X)

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 대량 시드 | 50명 유저 + 100건 카풀 | 정상 생성, 타임아웃 없음 |
| X2 | Decimal 좌표 | 위도/경도 10자리 정밀도 | DECIMAL(10,7) 정상 저장 |

---

## Tasks

### Task 1: Prisma 스키마에 ENUM 6개 추가

**Objective:** DATABASE.md의 6개 ENUM을 schema.prisma에 정의

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: ENUM 정의 추가**

기존 `generator client`와 `datasource db` 블록 아래에 추가:

```prisma
enum Plan {
  FREE
  PRO
  ENTERPRISE
}

enum Provider {
  KAKAO
  GOOGLE
}

enum Role {
  PASSENGER
  DRIVER
  BOTH
  ADMIN
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
  COMPLETED
  FAILED
  REFUNDED
}

enum MessageType {
  TEXT
  IMAGE
  SYSTEM
}
```

**Step 2: 스키마 검증**

Run: `cd /home/heeho3/workspace/ridy/backend && npx prisma validate`
Expected: 스키마 유효

**Step 3: 커밋**

```bash
git add prisma/schema.prisma
git commit -m "feat(db): Prisma ENUM 6개 정의"
```

---

### Task 2: Company 모델 추가

**Objective:** 최상위 격리 단위인 Company 모델 작성

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: Company 모델 추가**

```prisma
model Company {
  id          String   @id @default(uuid())
  name        String   @db.VarChar(100)
  inviteCode  String   @unique @map("invite_code") @db.VarChar(20)
  domain      String?  @db.VarChar(100)
  adminId     String   @map("admin_id")
  maxMembers  Int      @default(50) @map("max_members")
  plan        Plan     @default(FREE)

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
```

**Step 2: 검증**

Run: `npx prisma validate`

**Step 3: 커밋**

```bash
git commit -m "feat(db): Company 모델 추가"
```

---

### Task 3: InviteCode 모델 추가

**Objective:** 기업 관리자가 발급하는 초대 코드 모델

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: InviteCode 모델 추가**

```prisma
model InviteCode {
  id          String    @id @default(uuid())
  companyId   String    @map("company_id")
  code        String    @db.VarChar(6)
  createdBy   String    @map("created_by")
  maxUses     Int       @default(10) @map("max_uses")
  currentUses Int       @default(0) @map("current_uses")
  expiresAt   DateTime? @map("expires_at")
  isActive    Boolean   @default(true) @map("is_active")

  // Relations
  company     Company   @relation(fields: [companyId], references: [id])
  creator     User      @relation("InviteCodeCreator", fields: [createdBy], references: [id])

  @@unique([companyId, code])
  @@map("invite_codes")
}
```

**Step 2: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): InviteCode 모델 추가`

---

### Task 4: User 모델 추가

**Objective:** 회사 소속 유저 모델. company_id 필수, employee_id 선택

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: User 모델 추가**

```prisma
model User {
  id            String   @id @default(uuid())
  companyId     String   @map("company_id")
  employeeId    String?  @map("employee_id") @db.VarChar(50)
  email         String   @unique @db.VarChar(255)
  name          String   @db.VarChar(100)
  phone         String?  @db.VarChar(20)
  imageUrl      String?  @map("image_url") @db.Text
  provider      Provider?
  providerId    String?  @map("provider_id") @db.VarChar(255)
  role          Role     @default(PASSENGER)
  rating        Decimal  @default(0.0) @db.Decimal(2, 1)
  rideCount     Int      @default(0) @map("ride_count")

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

  @@unique([companyId, employeeId])
  @@map("users")
}
```

**Step 2: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): User 모델 추가`

---

### Task 5: Vehicle 모델 추가

**Objective:** 차주의 차량 정보

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: Vehicle 모델 추가**

```prisma
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
```

**Step 2: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): Vehicle 모델 추가`

---

### Task 6: Ride + RideRequest 모델 추가

**Objective:** 카풀 운행과 탑승 요청. Ride에 company_id 격리 키 포함

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: Ride 모델 추가**

```prisma
model Ride {
  id              String       @id @default(uuid())
  companyId       String       @map("company_id")
  driverId        String       @map("driver_id")
  departureLat    Decimal      @map("departure_lat") @db.Decimal(10, 7)
  departureLng    Decimal      @map("departure_lng") @db.Decimal(10, 7)
  departureAddr   String?      @map("departure_addr") @db.VarChar(255)
  arrivalLat      Decimal      @map("arrival_lat") @db.Decimal(10, 7)
  arrivalLng      Decimal      @map("arrival_lng") @db.Decimal(10, 7)
  arrivalAddr     String?      @map("arrival_addr") @db.VarChar(255)
  departureTime   DateTime     @map("departure_time")
  availableSeats  Int          @map("available_seats")
  fare            Int?
  recurringDays   String?      @map("recurring_days") @db.VarChar(50)
  recurringEnd    DateTime?    @map("recurring_end")
  preferences     Json?
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
```

**Step 2: RideRequest 모델 추가**

```prisma
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
```

**Step 3: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): Ride, RideRequest 모델 추가`

---

### Task 7: Review + Settlement + PaymentMethod 모델 추가

**Objective:** 리뷰, 정산, 결제수단 모델

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: Review 모델 추가**

```prisma
model Review {
  id        String   @id @default(uuid())
  rideId    String   @map("ride_id")
  fromId    String   @map("from_id")
  toId      String   @map("to_id")
  rating    Int
  comment   String?  @db.Text

  createdAt DateTime @default(now()) @map("created_at")

  ride      Ride     @relation(fields: [rideId], references: [id])
  fromUser  User     @relation("ReviewFrom", fields: [fromId], references: [id])
  toUser    User     @relation("ReviewTo", fields: [toId], references: [id])

  @@map("reviews")
}
```

**Step 2: Settlement 모델 추가**

```prisma
model Settlement {
  id           String           @id @default(uuid())
  rideId       String           @map("ride_id")
  passengerId  String           @map("passenger_id")
  companyId    String?          @map("company_id")
  amount       Int
  driverAmount Int              @map("driver_amount")
  platformFee  Int              @map("platform_fee")
  companyFee   Int              @default(0) @map("company_fee")
  passengerFee Int              @default(0) @map("passenger_fee")
  status       SettlementStatus @default(PENDING)
  dueDate      DateTime?        @map("due_date")
  paidAt       DateTime?        @map("paid_at")

  createdAt    DateTime         @default(now()) @map("created_at")

  ride         Ride             @relation(fields: [rideId], references: [id])
  passenger    User             @relation(fields: [passengerId], references: [id])
  company      Company?         @relation(fields: [companyId], references: [id])

  @@map("settlements")
}
```

**Step 3: PaymentMethod 모델 추가**

```prisma
model PaymentMethod {
  id          String   @id @default(uuid())
  userId      String   @map("user_id")
  type        String   @db.VarChar(30)
  billingKey  String?  @map("billing_key") @db.VarChar(255)
  alias       String?  @db.VarChar(50)
  isDefault   Boolean  @default(false) @map("is_default")

  createdAt   DateTime @default(now()) @map("created_at")

  user        User     @relation(fields: [userId], references: [id])

  @@map("payment_methods")
}
```

**Step 4: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): Review, Settlement, PaymentMethod 모델 추가`

---

### Task 8: ChatRoom + Message 모델 추가

**Objective:** 채팅방과 메시지 모델

**Files:**
- Modify: `prisma/schema.prisma`

**Step 1: ChatRoom 모델 추가**

```prisma
model ChatRoom {
  id        String    @id @default(uuid())
  rideId    String    @unique @map("ride_id")

  createdAt DateTime  @default(now()) @map("created_at")

  ride      Ride      @relation(fields: [rideId], references: [id])
  messages  Message[]

  @@map("chat_rooms")
}
```

**Step 2: Message 모델 추가**

```prisma
model Message {
  id        String     @id @default(uuid())
  roomId    String     @map("room_id")
  senderId  String     @map("sender_id")
  content   String     @db.Text
  type      MessageType @default(TEXT)

  createdAt DateTime   @default(now()) @map("created_at")

  room      ChatRoom   @relation(fields: [roomId], references: [id])
  sender    User       @relation(fields: [senderId], references: [id])

  @@map("messages")
}
```

**Step 3: 검증 & 커밋**

Run: `npx prisma validate`
Commit: `feat(db): ChatRoom, Message 모델 추가`

---

### Task 9: PrismaClient 재생성 + 컴파일 검증

**Objective:** 전체 스키마 기반으로 PrismaClient 생성 + NestJS 빌드 확인

**Files:**
- Generated: `node_modules/.prisma/client/`

**Step 1: PrismaClient 생성**

Run: `cd /home/heeho3/workspace/ridy/backend && npx prisma generate`
Expected: PrismaClient 생성 성공

**Step 2: NestJS 빌드**

Run: `npm run build`
Expected: 빌드 성공 (기존 코드와 충돌 없음)

**Step 3: 기존 테스트 실행**

Run: `npm run test`
Expected: 기존 health 테스트 통과

**Step 4: 커밋**

```bash
git commit -m "chore(db): PrismaClient 재생성 및 빌드 검증"
```

---

### Task 10: 마이그레이션 실행

**Objective:** PostgreSQL에 스키마 반영

**전제조건:** PostgreSQL 실행 중 + DATABASE_URL 설정

**Step 1: Docker로 PostgreSQL 기동 (필요 시)**

```bash
docker run -d --name ridy-postgres \
  -e POSTGRES_USER=ridy \
  -e POSTGRES_PASSWORD=ridy123 \
  -e POSTGRES_DB=ridy \
  -p 5432:5432 \
  postgres:16-alpine
```

**Step 2: .env에 DATABASE_URL 설정**

```
DATABASE_URL=postgresql://ridy:ridy123@localhost:5432/ridy
```

**Step 3: 마이그레이션 실행**

Run: `npx prisma migrate dev --name init`
Expected: 마이그레이션 파일 생성 + DB 반영

**Step 4: DB 테이블 확인**

Run: `npx prisma db execute --stdin <<< "\\dt"`
Expected: 11개 테이블 + _prisma_migrations 확인

**Step 5: 커밋**

```bash
git add prisma/migrations/
git commit -m "migrate(db): 초기 마이그레이션 (11개 테이블)"
```

---

### Task 11: 시드 스크립트 작성

**Objective:** 개발용 초기 데이터 생성 스크립트

**Files:**
- Create: `prisma/seed.ts`

**Step 1: 시드 스크립트 작성**

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  // 1. 회사 생성
  const company = await prisma.company.create({
    data: {
      name: '테크스타터',
      inviteCode: 'TECH01',
      domain: 'techstarter.co.kr',
      maxMembers: 50,
      plan: 'PRO',
      admin: {
        create: {
          email: 'admin@techstarter.co.kr',
          name: '최서연',
          phone: '010-0000-0001',
          provider: 'KAKAO',
          providerId: 'kakao_admin',
          role: 'ADMIN',
          employeeId: 'TS-001',
        },
      },
    },
  });

  // 2. 차주 생성
  const driver = await prisma.user.create({
    data: {
      companyId: company.id,
      email: 'driver@techstarter.co.kr',
      name: '박준서',
      phone: '010-0000-0002',
      provider: 'GOOGLE',
      providerId: 'google_driver',
      role: 'DRIVER',
      employeeId: 'TS-035',
      rating: 4.8,
      rideCount: 42,
      vehicles: {
        create: {
          model: '아반떼',
          color: '흰색',
          plate: '12가 3456',
          capacity: 4,
        },
      },
    },
  });

  // 3. 탑승자 생성
  const passenger = await prisma.user.create({
    data: {
      companyId: company.id,
      email: 'passenger@techstarter.co.kr',
      name: '김도윤',
      phone: '010-0000-0003',
      provider: 'KAKAO',
      providerId: 'kakao_passenger',
      role: 'PASSENGER',
      employeeId: 'TS-042',
    },
  });

  // 4. 초대 코드 생성
  await prisma.inviteCode.create({
    data: {
      companyId: company.id,
      code: 'ABC123',
      createdBy: company.adminId,
      maxUses: 10,
    },
  });

  // 5. 샘플 카풀
  const ride = await prisma.ride.create({
    data: {
      companyId: company.id,
      driverId: driver.id,
      departureLat: 37.4979,
      departureLng: 127.0276,
      departureAddr: '강남역',
      arrivalLat: 37.2636,
      arrivalLng: 127.0286,
      arrivalAddr: '수원역',
      departureTime: new Date(Date.now() + 86400000),
      availableSeats: 3,
      fare: 5000,
      recurringDays: 'MON,TUE,WED,THU,FRI',
      preferences: { noSmoking: true },
      status: 'OPEN',
    },
  });

  // 6. 채팅방
  await prisma.chatRoom.create({
    data: {
      rideId: ride.id,
    },
  });

  console.log('✅ Seed data created successfully');
  console.log(`   Company: ${company.name} (${company.id})`);
  console.log(`   Admin: 최서연, Driver: 박준서, Passenger: 김도윤`);
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

**Step 2: package.json에 seed 스크립트 추가**

```json
"prisma": {
  "seed": "npx ts-node --compiler-options {\"module\":\"commonjs\"} prisma/seed.ts"
}
```

**Step 3: 시드 실행**

Run: `npx prisma db seed`
Expected: 데이터 생성 성공 로그

**Step 4: 데이터 확인**

Run: `npx prisma studio` 또는 쿼리로 확인

**Step 5: 커밋**

```bash
git add prisma/seed.ts package.json
git commit -m "feat(db): 개발용 시드 스크립트 작성"
```

---

### Task 12: PrismaService 테스트 작성 (TDD Red)

**Objective:** PrismaService 연결 + 기본 CRUD 동작 검증

**Files:**
- Create: `src/prisma/prisma.service.spec.ts`

**Step 1: 테스트 파일 작성**

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { PrismaService } from './prisma.service';

describe('PrismaService', () => {
  let service: PrismaService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [PrismaService],
    }).compile();

    service = module.get<PrismaService>(PrismaService);
  });

  afterAll(async () => {
    await service.$disconnect();
  });

  it('서비스가 정의되어야 함', () => {
    expect(service).toBeDefined();
  });

  it('DB 연결이 가능해야 함', async () => {
    await expect(service.$queryRaw`SELECT 1`).resolves.toBeDefined();
  });

  it('Company 생성이 가능해야 함', async () => {
    const company = await service.company.create({
      data: {
        name: '테스트 회사',
        inviteCode: 'TEST01',
        admin: {
          create: {
            email: 'test@test.com',
            name: '테스트 관리자',
            role: 'ADMIN',
          },
        },
      },
    });
    expect(company.name).toBe('테스트 회사');
    expect(company.inviteCode).toBe('TEST01');
  });

  it('company_id 없는 User 생성 시 에러', async () => {
    await expect(
      service.user.create({
        data: {
          email: 'no-company@test.com',
          name: '무소속',
        },
      }),
    ).rejects.toThrow();
  });

  it('중복 email User 생성 시 에러', async () => {
    // 먼저 유저 생성
    const company = await service.company.findFirst();
    await service.user.create({
      data: {
        companyId: company!.id,
        email: 'duplicate@test.com',
        name: '유저1',
      },
    });

    // 같은 이메일로 재생성
    await expect(
      service.user.create({
        data: {
          companyId: company!.id,
          email: 'duplicate@test.com',
          name: '유저2',
        },
      }),
    ).rejects.toThrow();
  });
});
```

**Step 2: 테스트 실행 (Red 확인)**

Run: `npm run test -- --testPathPattern=prisma.service.spec`
Expected: DB 연결 관련 에러 (아직 DB 없음) 또는 통과 (DB 있으면)

**Step 3: 커밋**

```bash
git add src/prisma/prisma.service.spec.ts
git commit -m "test(db): PrismaService 연결 및 제약조건 테스트"
```

---

### Task 13: 전체 검증 + 최종 커밋

**Objective:** 전체 파이프라인 검증

**Step 1: 전체 테스트**

Run: `npm run test`
Expected: 전체 통과

**Step 2: 린트**

Run: `npm run lint`
Expected: 에러 없음

**Step 3: 빌드**

Run: `npm run build`
Expected: 빌드 성공

**Step 4: 서버 기동 확인**

Run: `npm run start:dev` → GraphQL Playground에서 `query { health { status service } }` 확인

**Step 5: codegen 실행**

Run: `npm run codegen`
Expected: generated 타입 갱신

**Step 6: 최종 커밋 (필요 시)**

```bash
git add -A
git commit -m "chore(db): 전체 검증 완료"
```

---

## 검증 체크리스트

- [ ] `npx prisma validate` 통과
- [ ] `npx prisma generate` 성공
- [ ] `npx prisma migrate dev` 성공
- [ ] 11개 테이블 생성 확인
- [ ] 시드 스크립트 실행 성공
- [ ] `npm run test` 통과
- [ ] `npm run lint` 통과
- [ ] `npm run build` 성공
- [ ] GraphQL health 쿼리 정상
- [ ] E1~E5 제약조건 테스트 통과
