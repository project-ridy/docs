# Ridy — 정산/결제 API (GraphQL)

## 개요

거리 기반 자동 비용 계산 + 토스페이먼츠 결제 연동. 회사 plan에 따라 수수료 부담 주체가 다름.

---

## GraphQL 스키마

```graphql
type Settlement {
  id: ID!
  ride: Ride!
  passenger: User!
  companyId: String!
  amount: Int!              # 탑승자 결제 금액
  driverAmount: Int!        # 차주 수령 금액
  platformFee: Int!         # 플랫폼 수수료
  companyFee: Int!          # 회사 부담 수수료
  passengerFee: Int!        # 사원 부담 수수료
  dueDate: String
  paidAt: String
  status: SettlementStatus!
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

enum CompanyPlan {
  FREE        # 사원 전액 부담
  PRO         # 회사 50% 부담
  ENTERPRISE  # 회사 100% 부담
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
  companyFee: Int!
  passengerFee: Int!
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

type CompanySettlementSummary {
  totalSettlements: Int!
  totalAmount: Int!
  companyFeeTotal: Int!
  passengerFeeTotal: Int!
  co2SavedKg: Float!
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
  companySettlements(from: String, to: String): CompanySettlementSummary!  @adminOnly
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

거리 기반 요금 자동 계산. 회사 plan에 따라 수수료 부담 비율이 다름.

### 계산 공식

```
baseFare = 3,000원
distanceFare = 거리(km) × 420원/km
totalFare = baseFare + distanceFare
perPassenger = ceil(totalFare / passengers)
platformFee = ceil(perPassenger × 0.05)  # 5%

# 회사 plan별 수수료 분담
FREE:       companyFee=0,            passengerFee=platformFee
PRO:        companyFee=platformFee/2, passengerFee=platformFee/2
ENTERPRISE: companyFee=platformFee,   passengerFee=0

driverAmount = perPassenger - platformFee
```

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | FREE 플랜 | 28.5km, 3명 | perPassenger=4,990, companyFee=0, passengerFee=250 |
| A2 | PRO 플랜 | 같은 거리 | companyFee=125, passengerFee=125 |
| A3 | ENTERPRISE 플랜 | 같은 거리 | companyFee=250, passengerFee=0 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | passengers 0 | passengers=0 | `BAD_REQUEST` |
| E2 | 출발지=도착지 | 같은 좌표 | `BAD_REQUEST` |
| E3 | 거리 과대 | 400km+ | `BAD_REQUEST: 카풀 거리는 200km 이내` |

---

## 기능: 정산 결제 (`paySettlement`)

### 설명

탑승자가 결제. ENTERPRISE 플랜은 사원 부담 0원이므로 자동 PAID 처리.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 일반 결제 | status → PAID, paidAt 기록 |
| A2 | ENTERPRISE 자동 처리 | passengerFee=0 → 즉시 PAID (결제 수단 불필요) |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 이미 결제됨 | `BAD_REQUEST: 이미 결제 완료` |
| E2 | 본인 아닌 정산 | `FORBIDDEN` |
| E3 | 결제 실패 (잔액부족) | `PAYMENT_FAILED` |
| E4 | 토스페이먼츠 장애 | `INTERNAL_ERROR` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 기한 초과 | dueDate 경과 | `BAD_REQUEST: 정산 기한 만료` |
| X2 | 동시 결제 | 같은 정산에 동시 요청 | 첫 번째만 성공 |

---

## 기능: 정산 취소 (`cancelSettlement`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 24시간 내 취소 | status → CANCELLED, 환불 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | PENDING 상태 | `BAD_REQUEST: 결제되지 않은 정산` |
| E2 | 7일 경과 | `BAD_REQUEST: 취소 기한 만료` |

---

## 기능: 회사 정산 통계 (`companySettlements`) — 관리자 전용

### 설명

회사 관리자가 전체 정산 현황 조회. ESG 리포트용 CO₂ 절감량 포함.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 월간 통계 | 총 정산 건수, 회사 부담액, CO₂ 절감량 |
| A2 | 기간 필터 | from/to 범위 내 결과 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 관리자 아님 | `FORBIDDEN` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 데이터 없음 | 첫 달 | 모든 값 0 |
