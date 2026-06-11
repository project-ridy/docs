# Ridy — 매칭 API (GraphQL)

## 개요

출발지/도착지/시간 기반 카풀 매칭. 차주가 카풀을 등록하고, 탑승자가 검색 후 요청.

---

## GraphQL 스키마

```graphql
type Ride {
  id: ID!
  driver: User!
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
  preferences: String  # JSONB → JSON string
  status: RideStatus!
  currentPassengers: Int!
  createdAt: String!
  updatedAt: String!
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
  pickupLat: Float
  pickupLng: Float
  pickupAddr: String
  message: String
  status: RequestStatus!
  createdAt: String!
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
  depLat: Float!
  depLng: Float!
  arrLat: Float!
  arrLng: Float!
  time: String!
  seats: Int
  radius: Int
}

input CreateRideRequestInput {
  rideId: ID!
  pickupLat: Float
  pickupLng: Float
  pickupAddr: String
  message: String
}

input HandleRequestInput {
  requestId: ID!
  action: String!  # accept | reject
}

type SearchRidesResult {
  rides: [Ride!]!
  total: Int!
}

type Query {
  myRides(status: RideStatus, role: String): [Ride!]!
  searchRides(input: SearchRidesInput!): SearchRidesResult!
  rideDetail(id: ID!): Ride!
  rideRequests(rideId: ID!): [RideRequest!]!
}

type Mutation {
  createRide(input: CreateRideInput!): Ride!
  cancelRide(id: ID!): Ride!
  createRideRequest(input: CreateRideRequestInput!): RideRequest!
  cancelRideRequest(id: ID!): RideRequest!
  handleRideRequest(input: HandleRequestInput!): RideRequest!
  startRide(id: ID!): Ride!
  completeRide(id: ID!): Ride!
}
```

---

## 기능: 카풀 등록 (`createRide`)

### 설명

차주가 새 카풀 운행을 등록. 차주(role=driver 또는 both)만 가능.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 일반 카풀 등록 | 강남→수원, 좌석 3, 요금 5000 | Ride 생성, status=OPEN |
| A2 | 정기 카풀 등록 | recurringDays="MON,TUE,WED,THU,FRI", recurringEnd="2026-08-15" | 정기 카풀 생성 |
| A3 | 선호도 포함 등록 | preferences: `{ smoking: false, pet: false, luggage: true }` | preferences JSONB 저장 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 탑승자가 등록 시도 | role=passenger 유저 | `FORBIDDEN: 차주만 카풀을 등록할 수 있습니다` |
| E2 | 좌석 0 | availableSeats=0 | `BAD_REQUEST: 좌석 수는 1 이상이어야 합니다` |
| E3 | 과거 시간 등록 | departureTime이 현재보다 이전 | `BAD_REQUEST: 출발 시간은 현재 이후여야 합니다` |
| E4 | 출발지=도착지 | 위도/경도 동일 | `BAD_REQUEST: 출발지와 도착지가 같을 수 없습니다` |
| E5 | 인증 안 됨 | 토큰 없음 | `UNAUTHENTICATED` |
| E6 | 좌석 수 초과 | availableSeats > 차량 capacity | `BAD_REQUEST: 차량 좌석 수를 초과할 수 없습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 등록 제한 | 한 차주가 OPEN 상태 카풀 5개 이상 | `FORBIDDEN: 동시에 5개까지만 모집할 수 있습니다` |
| X2 | 정기 카풀 종료일 과거 | recurringEnd가 현재보다 이전 | `BAD_REQUEST: 정기 카풀 종료일이 출발 시간보다 이전입니다` |
| X3 | 요금 0원 | fare=0 또는 생략 | 무료 카풀로 등록 허용 (fare nullable) |

---

## 기능: 카풀 검색 (`searchRides`)

### 설명

탑승자가 출발지/도착지/시간으로 카풀 검색. Haversine 공식으로 반경 내 필터링.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 검색 | 강남→수원, 오늘 08:30 | 반경 3km 내 카풀 목록, 거리순 정렬 |
| A2 | 반경 확장 | radius=10 | 10km 내 결과 |
| A3 | 좌석 수 필터 | seats=2 | 2석 이상 남은 카풀만 |
| A4 | 결과 없음 | 검색 조건에 맞는 카풀 없음 | `{ rides: [], total: 0 }` |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 필수 파라미터 누락 | depLat 없음 | `BAD_REQUEST: 출발지 위도가 필요합니다` |
| E2 | 잘못된 좌표 | depLat=999 | `BAD_REQUEST: 올바른 좌표가 아닙니다` |
| E3 | 반경 과대 | radius=1000 | 최대 50km로 클램핑 후 검색 |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 출발 시간 ±30분 | 요청 시간 기준 ±30분 내 카풀 포함 | 시간 범위 내 결과 반환 |
| X2 | MATCHED 상태 카풀 | 좌석 남아있는 MATCHED 카풀 | 검색 결과에 포함 (좌석 남았으면 탑승 가능) |
| X3 | CANCELLED/COMPLETED | 완료/취소된 카풀 | 검색 결과에서 제외 |

---

## 기능: 탑승 요청 (`createRideRequest`)

### 설명

탑승자가 카풀에 참가 요청. 차주가 수락/거절.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 요청 | rideId + 메시지 | RideRequest 생성, status=PENDING |
| A2 | 픽업지 지정 요청 | rideId + pickupLat/Lng/Addr | 픽업지 포함 요청 생성 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 본인 카풀에 요청 | 차주가 자기 카풀에 탑승 요청 | `FORBIDDEN: 본인 카풀에 요청할 수 없습니다` |
| E2 | 만석 카풀 | availableSeats=0인 카풀 | `BAD_REQUEST: 좌석이 모두 찼습니다` |
| E3 | 중복 요청 | 이미 PENDING 요청 존재 | `BAD_REQUEST: 이미 요청하셨습니다` |
| E4 | 취소된 카풀 | status=CANCELLED인 카풀 | `BAD_REQUEST: 취소된 카풀입니다` |
| E5 | 완료된 카풀 | status=COMPLETED인 카풀 | `BAD_REQUEST: 이미 종료된 카풀입니다` |
| E6 | 인증 안 됨 | 토큰 없음 | `UNAUTHENTICATED` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 거절 후 재요청 | 이전 요청 REJECTED → 새 요청 | 새 PENDING 요청 생성 허용 |
| X2 | 동시성 — 같은 좌석 | 마지막 1석에 동시 요청 2개 | 먼저 수락된 것만 성공, 나머지는 만석 에러 |
| X3 | 요청 후 카풀 취소 | PENDING 요청 있는 상태에서 차주가 카풀 취소 | 모든 PENDING 요청 자동 CANCELLED |

---

## 기능: 요청 수락/거절 (`handleRideRequest`)

### 설명

차주가 탑승 요청을 수락하거나 거절.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 수락 | action="accept" | status=ACCEPTED, 카풀 availableSeats 감소 |
| A2 | 거절 | action="reject" | status=REJECTED, 좌석 변동 없음 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 탑승자가 처리 시도 | 차주 아닌 유저 | `FORBIDDEN: 본인 카풀의 요청만 처리할 수 있습니다` |
| E2 | 이미 처리된 요청 | status=ACCEPTED인 요청 다시 수락 | `BAD_REQUEST: 이미 처리된 요청입니다` |
| E3 | 만석 수락 | availableSeats=0인데 수락 | `BAD_REQUEST: 좌석이 모두 찼습니다` |
| E4 | 잘못된 action | action="maybe" | `BAD_REQUEST: 올바르지 않은 처리입니다 (accept/reject)` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 수락 시 카풀 상태 변경 | 마지막 좌석이 채워지면 | 카풀 status → MATCHED |
| X2 | 수락 시 채팅방 자동 생성 | 첫 탑승자 수락 | ChatRoom 자동 생성 + 시스템 메시지 |

---

## 기능: 카풀 상태 변경

### 운행 시작 (`startRide`)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 정상 시작 | status → IN_PROGRESS |
| E1 | 차주 아님 | `FORBIDDEN` |
| E2 | MATCHED가 아닌 상태 | `BAD_REQUEST: 매칭 완료 상태에서만 시작할 수 있습니다` |

### 운행 완료 (`completeRide`)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 정상 완료 | status → COMPLETED, 정산 자동 생성 |
| E1 | IN_PROGRESS가 아닌 상태 | `BAD_REQUEST: 운행 중일 때만 완료할 수 있습니다` |

### 카풀 취소 (`cancelRide`)

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | OPEN 상태 취소 | status → CANCELLED, 모든 PENDING 요청 CANCELLED |
| A2 | MATCHED 상태 취소 | status → CANCELLED, ACCEPTED 요청에 알림 발송 |
| E1 | IN_PROGRESS 상태 취소 | `BAD_REQUEST: 운행 중인 카풀은 취소할 수 없습니다` |
| E2 | 이미 COMPLETED/CANCELLED | `BAD_REQUEST: 이미 종료/취소된 카풀입니다` |
