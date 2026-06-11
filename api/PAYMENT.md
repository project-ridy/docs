# Ridy — 정산/결제 API (GraphQL)

## 개요

거리 기반 자동 비용 계산 + 토스페이먼츠 결제 연동. 플랫폼 수수료 차감 후 차주 정산.

---

## GraphQL 스키마

```graphql
type Settlement {
  id: ID!
  ride: Ride!
  passenger: User!
  amount: Int!           # 탑승자 결제 금액
  driverAmount: Int!     # 차주 수령 금액
  platformFee: Int!      # 플랫폼 수수료
  status: SettlementStatus!
  dueDate: String
  paidAt: String
  createdAt: String!
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

type PaymentMethod {
  id: ID!
  type: PaymentType!
  alias: String
  isDefault: Boolean!
  createdAt: String!
}

type FareCalculation {
  distanceKm: Float!
  durationMin: Int!
  totalFare: Int!
  perPassenger: Int!
  platformFee: Int!
  driverAmount: Int!
}

input CalculateFareInput {
  departureLat: Float!
  departureLng: Float!
  arrivalLat: Float!
  arrivalLng: Float!
  passengers: Int!
}

input RegisterPaymentMethodInput {
  type: PaymentType!
  billingKey: String!
  alias: String
}

input PaySettlementInput {
  settlementId: ID!
  paymentMethodId: ID!
}

type SettlementHistoryResult {
  history: [Settlement!]!
  totalEarned: Int!
  totalSpent: Int!
}

type Query {
  mySettlements(from: String, to: String, role: String): SettlementHistoryResult!
  settlementDetail(id: ID!): Settlement!
  myPaymentMethods: [PaymentMethod!]!
}

type Mutation {
  calculateFare(input: CalculateFareInput!): FareCalculation!
  paySettlement(input: PaySettlementInput!): Settlement!
  registerPaymentMethod(input: RegisterPaymentMethodInput!): PaymentMethod!
  deletePaymentMethod(id: ID!): Boolean!
  setDefaultPaymentMethod(id: ID!): PaymentMethod!
  cancelSettlement(id: ID!): Settlement!
}
```

---

## 기능: 요금 계산 (`calculateFare`)

### 설명

출발지/도착지 거리 기반으로 카풀 요금 자동 계산. 기본요금 + 거리요금 + 플랫폼 수수료.

### 계산 공식

```
baseFare = 3,000원 (기본)
distanceFare = 거리(km) × 420원/km
totalFare = baseFare + distanceFare
perPassenger = ceil(totalFare / passengers)
platformFee = ceil(perPassenger × 0.05)  # 5% 수수료
driverAmount = perPassenger - platformFee
```

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 계산 | 강남→수원 (28.5km), 3명 | totalFare=14,970, perPassenger=4,990, platformFee=250, driverAmount=4,740 |
| A2 | 1명 탑승 | 같은 거리, 1명 | perPassenger=14,970 |
| A3 | 근거리 | 2km, 2명 | baseFare=3,000 + 840 = 3,840, perPassenger=1,920 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | passengers 0 | passengers=0 | `BAD_REQUEST: 탑승자 수는 1 이상이어야 합니다` |
| E2 | passengers 과다 | passengers=100 | `BAD_REQUEST: 탑승자 수는 4 이하여야 합니다` |
| E3 | 출발지=도착지 | 같은 좌표 | `BAD_REQUEST: 출발지와 도착지가 같을 수 없습니다` |
| E4 | 거리 과대 | 서울→부산 (400km+) | `BAD_REQUEST: 카풀 거리는 200km 이내여야 합니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 최소 요금 | 0.5km | baseFare 3,000원 적용 (거리요금 210원, 총 3,210원) |
| X2 | 반올림 | 나누어 떨어지지 않는 금액 | ceil로 올림 처리 |
| X3 | 수수료 최소 단위 | platformFee < 10원 | 최소 10원으로 설정 |

---

## 기능: 정산 결제 (`paySettlement`)

### 설명

탑승자가 등록된 결제 수단으로 정산 금액 결제. 토스페이먼츠 빌링키 결제.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 카드 결제 | settlementId + paymentMethodId(CARD) | status → PAID, paidAt 기록 |
| A2 | 카카오페이 결제 | paymentMethodId(KAKAO_PAY) | 동일하게 PAID 처리 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 이미 결제된 정산 | status=PAID인 정산 | `BAD_REQUEST: 이미 결제 완료된 정산입니다` |
| E2 | 취소된 정산 | status=CANCELLED | `BAD_REQUEST: 취소된 정산입니다` |
| E3 | 본인 아닌 정산 | 다른 탑승자의 정산 | `FORBIDDEN: 본인의 정산만 결제할 수 있습니다` |
| E4 | 결제 수단 없음 | 존재하지 않는 paymentMethodId | `NOT_FOUND: 결제 수단을 찾을 수 없습니다` |
| E5 | 결제 실패 (잔액부족) | 토스페이먼츠에서 거절 | `PAYMENT_FAILED: 결제가 거절되었습니다 (잔액부족)` |
| E6 | 결제 실패 (한도초과) | 한도 초과 | `PAYMENT_FAILED: 결제가 거절되었습니다 (한도초과)` |
| E7 | 토스페이먼츠 장애 | API 타임아웃 | `INTERNAL_ERROR: 결제 서비스에 일시적인 문제가 있습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 기한 초과 | dueDate 경과 후 결제 시도 | `BAD_REQUEST: 정산 기한이 만료되었습니다` |
| X2 | 동시 결제 | 같은 정산에 동시 요청 | 첫 번째만 성공, 두 번째는 E1 에러 |

---

## 기능: 정산 취소 (`cancelSettlement`)

### 설명

결제 완료된 정산 취소. 전액 환불 처리.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 결제 후 24시간 내 취소 | status → CANCELLED, 토스페이먼츠 환불 처리 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | PENDING 상태 취소 | `BAD_REQUEST: 결제되지 않은 정산은 취소할 수 없습니다` |
| E2 | 이미 취소됨 | `BAD_REQUEST: 이미 취소된 정산입니다` |
| E3 | 취소 기한 만료 | 결제 후 7일 경과 | `BAD_REQUEST: 취소 기한이 만료되었습니다` |

---

## 기능: 결제 수단 관리

### 등록 (`registerPaymentMethod`)

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 카드 등록 | type=CARD, billingKey="..." | PaymentMethod 생성 |
| A2 | 첫 결제 수단 | 유저의 첫 결제 수단 | isDefault=true 자동 설정 |

| # | 예외 | 기대 결과 |
|---|------|-----------|
| E1 | 빌링키 없음 | `BAD_REQUEST: 빌링키가 필요합니다` |
| E2 | 중복 빌링키 | 이미 등록된 빌링키 | `BAD_REQUEST: 이미 등록된 결제 수단입니다` |
| E3 | 최대 초과 | 결제 수단 5개 초과 | `BAD_REQUEST: 결제 수단은 5개까지 등록 가능합니다` |

### 삭제 (`deletePaymentMethod`)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 일반 삭제 | 결제 수단 삭제 |
| A2 | 기본 결제 수단 삭제 | 삭제 후 남은 수단 중 첫 번째를 기본으로 자동 설정 |

| # | 예외 | 기대 결과 |
|---|------|-----------|
| E1 | 진행 중인 정산 있음 | PENDING 정산이 있는 결제 수단 | `BAD_REQUEST: 진행 중인 정산이 있어 삭제할 수 없습니다` |

### 기본 설정 (`setDefaultPaymentMethod`)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 기본 변경 | 기존 isDefault=false, 선택한 수단 isDefault=true |

| # | 예외 | 기대 결과 |
|---|------|-----------|
| E1 | 본인 아닌 수단 | `FORBIDDEN` |

---

## 기능: 정산 이력 (`mySettlements`)

### 설명

유저의 정산 이력 조회. 차주는 수입, 탑승자는 지출 기준.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 전체 이력 | 모든 정산 내역 + 총 수입/지출 |
| A2 | 기간 필터 | from/to 날짜 범위 내 결과 |
| A3 | 역할 필터 | role=driver → 차주 수입 기준 |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 이력 없음 | 첫 유저 | `{ history: [], totalEarned: 0, totalSpent: 0 }` |
