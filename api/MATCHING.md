# Ridy — 매칭 API (GraphQL)

## 개요

출발지/도착지/시간 기반 카풀 매칭. **같은 회사 이메일 도메인 구성원끼리만** 매칭 가능.

---

## GraphQL 스키마

```graphql
type Ride {
  id: ID!
  driver: User!
  companyId: String!
  departureLat: Float!
  departureLng: Float!
  departureAddr: String
  arrivalLat: Float!
  arrivalLng: Float!
  arrivalAddr: String
  departureTime: String!
  availableSeats: Int!
  fare: Int
  recurringDays: String
  recurringEnd: String
  preferences: String  # JSON
  status: RideStatus!
  createdAt: String!
  updatedAt: String!
  rideRequests: [RideRequest!]!
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

type RideRequest {
  id: ID!
  ride: Ride!
  passenger: User!
  pickupAddr: String
  message: String
  status: RequestStatus!
  createdAt: String!
  updatedAt: String!
}

input CreateRideInput {
  departureLat: Float!
  departureLng: Float!
  departureAddr: String
  arrivalLat: Float!
  arrivalLng: Float!
  arrivalAddr: String
  departureTime: String!
  availableSeats: Int!
  fare: Int
  recurringDays: String
  recurringEnd: String
  preferences: String
}

input SearchRidesInput {
  departureLat: Float!
  departureLng: Float!
  arrivalLat: Float!
  arrivalLng: Float!
  departureTime: String!
  passengers: Int = 1
  radiusKm: Float = 5.0
}

input RequestRideInput {
  rideId: ID!
  pickupAddr: String
  message: String
}

type SearchRidesResult {
  rides: [Ride!]!
  totalCount: Int!
}

type Query {
  searchRides(input: SearchRidesInput!): SearchRidesResult!
  rideDetail(id: ID!): Ride!
  myRides(status: RideStatus): [Ride!]!
  myRequests(status: RequestStatus): [RideRequest!]!
}

type Mutation {
  createRide(input: CreateRideInput!): Ride!
  updateRide(id: ID!, input: CreateRideInput!): Ride!
  cancelRide(id: ID!): Ride!
  requestRide(input: RequestRideInput!): RideRequest!
  acceptRequest(id: ID!): RideRequest!
  rejectRequest(id: ID!): RideRequest!
  cancelRequest(id: ID!): RideRequest!
}
```

---

## 기능: 카풀 등록 (`createRide`)

### 설명

차주가 카풀을 등록. 자동으로 본인의 companyId가 설정됨.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 일반 카풀 | 출발/도착/시간/좌석 | Ride 생성, status=OPEN |
| A2 | 정기 카풀 | recurringDays="월,화,수,목,금" | 정기 카풀로 등록 |
| A3 | 선호도 설정 | preferences="{noSmoking:true, petAllowed:false}" | preferences JSON 저장 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 좌석 0 | availableSeats=0 | `BAD_REQUEST: 좌석 수는 1 이상이어야 합니다` |
| E2 | 과거 시간 | departureTime=어제 | `BAD_REQUEST: 출발 시간은 현재 이후여야 합니다` |
| E3 | 출발지=도착지 | 같은 좌표 | `BAD_REQUEST: 출발지와 도착지가 달라야 합니다` |
| E4 | 중복 등록 | 같은 시간대 이미 등록한 카풀 | `CONFLICT: 해당 시간에 이미 등록한 카풀이 있습니다` |
| E5 | 비차주 | role=passenger | `FORBIDDEN: 차주만 카풀을 등록할 수 있습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 만석 등록 | availableSeats=1 + 이미 1명 승인 | OPEN 유지 (추가 요청은 E4) |

---

## 기능: 카풀 검색 (`searchRides`)

### 설명

탑승자가 출발지/도착지/시간으로 카풀 검색. **같은 회사 사원의 카풀만** 결과에 노출.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 검색 | 강남→수원, 오늘 08:00 | 같은 회사 사원의 카풀만 반환 |
| A2 | 반경 검색 | radiusKm=3.0 | 3km 이내 카풀만 |
| A3 | 다인 검색 | passengers=2 | 2석 이상 남은 카풀만 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 타회사 카풀 | 다른 companyId 카풀 존재 | 검색 결과에 미포함 |
| E2 | passengers 0 | passengers=0 | `BAD_REQUEST` |
| E3 | 과거 시간 | departureTime=어제 | `BAD_REQUEST` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 검색 결과 없음 | 조건에 맞는 카풀이 없음 | `{ rides: [], totalCount: 0 }` |
| X2 | 만석 카풀 | 모든 좌석이 찬 카풀 | 검색 결과에서 제외 |
| X3 | 본인 카풀 | 자기가 등록한 카풀 | 검색 결과에서 제외 |

---

## 기능: 탑승 요청 (`requestRide`)

### 설명

탑승자가 카풀에 탑승 요청. 같은 회사 사원만 요청 가능.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 정상 요청 | RideRequest 생성, status=PENDING |
| A2 | 메시지 포함 | message="문 앞에서 탈게요" 저장 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 타회사 카풀 | 다른 회사 카풀에 요청 | `FORBIDDEN: 같은 회사 사원만 요청할 수 있습니다` |
| E2 | 만석 | availableSeats=0 | `BAD_REQUEST: 좌석이 모두 찼습니다` |
| E3 | 중복 요청 | 이미 요청한 카풀 | `CONFLICT: 이미 요청한 카풀입니다` |
| E4 | 본인 카풀 | 자기가 등록한 카풀 | `BAD_REQUEST: 본인 카풀에는 요청할 수 없습니다` |
| E5 | 취소된 카풀 | status=CANCELLED | `BAD_REQUEST: 취소된 카풀입니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 요청 | 2명이 마지막 1석에 동시 요청 | 먼저 온 요청만 PENDING, 나머지는 만석 에러 |

---

## 기능: 요청 수락/거절 (`acceptRequest` / `rejectRequest`)

### 설명

차주가 탑승 요청을 수락/거절. 본인 카풀에만 가능.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 수락 | status=ACCEPTED, availableSeats 감소 |
| A2 | 거절 | status=REJECTED |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 본인 아닌 카풀 | 다른 차주의 카풀 | `FORBIDDEN` |
| E2 | 이미 처리됨 | status=ACCEPTED/REJECTED | `BAD_REQUEST: 이미 처리된 요청입니다` |
| E3 | 수락 시 만석 | availableSeats=0 | `BAD_REQUEST: 좌석이 모두 찼습니다` |

---

## 기능: 카풀 상태 변경

### 설명

차주가 카풀 상태를 변경. MATCHED → IN_PROGRESS → COMPLETED.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 출발 | MATCHED → IN_PROGRESS |
| A2 | 완료 | IN_PROGRESS → COMPLETED, 정산 자동 생성 |
| A3 | 취소 | OPEN → CANCELLED |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 잘못된 전이 | OPEN → COMPLETED | `BAD_REQUEST: 유효하지 않은 상태 변경입니다` |
| E2 | 승인된 탑승자 있음 | MATCHED 상태에서 취소 | `BAD_REQUEST: 승인된 탑승자가 있습니다. 먼저 거절해주세요.` |
