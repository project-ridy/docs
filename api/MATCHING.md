# Ridy — 매칭 API (GraphQL)

## 개요

동네 출발지에서 회사 근무지로 이동하는 출근 카풀 매칭. **같은 회사 이메일 도메인 구성원끼리만** 매칭 가능하며, MVP에서는 퇴근 카풀·임의 목적지·직장 외 목적지 검색을 지원하지 않는다.

핵심 흐름:

1. 차주가 지도에서 출발 위치를 등록한다.
2. 도착지는 회사 근무지(`Workplace`)로 제한한다.
3. 탑승자는 내 동네/현재 위치 주변 지도 marker에서 카풀 제공자를 선택한다.
4. 승인 전에는 정확한 집 주소를 노출하지 않고 표시용 동네명과 대략 위치만 반환한다.

---

## GraphQL 스키마

```graphql
type Ride {
  id: ID!
  driver: User!
  companyId: ID!
  workplace: Workplace!
  pickupLabel: String!
  pickupPrivacy: PickupPrivacy!
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

type Workplace {
  id: ID!
  companyId: ID!
  name: String!
  lat: Float!
  lng: Float!
  address: String!
  isDefault: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum PickupPrivacy {
  APPROXIMATE
  APPROVED_ONLY
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
  pickup: LatLng
  pickupAddr: String
  message: String
  status: RequestStatus!
  createdAt: DateTime!
  updatedAt: DateTime!
}

input CreateRideInput {
  workplaceId: ID!
  departure: LatLngInput!
  departureAddr: String
  pickupLabel: String!
  pickupPrivacy: PickupPrivacy = APPROXIMATE
  departureTime: DateTime!
  availableSeats: Int!
  fare: Int
  preferences: JSON
}

input NearbyCommuteOffersInput {
  lat: Float!
  lng: Float!
  radiusKm: Float = 3.0
  workplaceId: ID
  departureTimeFrom: DateTime
  departureTimeTo: DateTime
  passengers: Int = 1
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

type Query {
  workplaces: [Workplace!]!
  nearbyCommuteOffers(input: NearbyCommuteOffersInput!, pagination: PaginationInput): RideConnection!
  searchRides(input: SearchRidesInput!, pagination: PaginationInput): RideConnection!
  ride(id: ID!): Ride!
  myRides(status: RideStatus, pagination: PaginationInput): RideConnection!
  myRideRequests(status: RequestStatus, pagination: PaginationInput): [RideRequest!]!
}

type Mutation {
  createRide(input: CreateRideInput!): Ride!
  updateRide(id: ID!, input: CreateRideInput!): Ride!
  cancelRide(id: ID!): Ride!
  requestRide(input: RequestRideInput!): RideRequest!
  acceptRideRequest(id: ID!): RideRequest!
  rejectRideRequest(id: ID!): RideRequest!
  cancelRideRequest(id: ID!): RideRequest!
}
```

---

## 기능: 카풀 등록 (`createRide`)

### 설명

차주가 동네 출발 위치와 회사 근무지 목적지를 등록한다. 자동으로 본인의 `companyId`가 설정되며, `workplaceId`는 같은 회사의 근무지만 허용한다. `departureAddr`는 차주 내부 관리용이며 승인 전 탑승자에게 그대로 노출하지 않는다.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 회사행 출근 카풀 | 출발 위치/근무지/시간/좌석 | Ride 생성, status=OPEN |
| A2 | 선호도 설정 | preferences={noSmoking:true, petAllowed:false} | preferences JSON 저장 |
| A3 | 표시용 동네명 | pickupLabel="성수동 인근" | 지도/카드에는 표시용 동네명 노출 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 좌석 0 | availableSeats=0 | `BAD_REQUEST: 좌석 수는 1 이상이어야 합니다` |
| E2 | 과거 시간 | departureTime=어제 | `BAD_REQUEST: 출발 시간은 현재 이후여야 합니다` |
| E3 | 출발지=근무지 | 같은 좌표 | `BAD_REQUEST: 출발 위치와 근무지가 달라야 합니다` |
| E4 | 중복 등록 | 같은 시간대 이미 등록한 카풀 | `CONFLICT: 해당 시간에 이미 등록한 카풀이 있습니다` |
| E5 | 비차주 | role=passenger | `FORBIDDEN: 차주만 카풀을 등록할 수 있습니다` |
| E6 | 타회사 근무지 | 다른 companyId의 workplaceId | `FORBIDDEN: 같은 회사 근무지만 선택할 수 있습니다` |
| E7 | 출근 방향 아님 | 회사 근무지 외 도착지 직접 입력 | GraphQL validation 실패 또는 `BAD_REQUEST: 회사행 카풀에만 요청할 수 있습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 만석 등록 | availableSeats=1 + 이미 1명 승인 | OPEN 유지 (추가 요청은 E4) |

---

## 기능: 지도 기반 주변 카풀 조회 (`nearbyCommuteOffers`)

### 설명

탑승자가 내 위치 또는 선택한 동네 중심 좌표로 주변 회사행 카풀을 조회한다. **같은 회사 사원의 OPEN 카풀만** 결과에 노출하며, 승인 전 정확한 집 주소를 반환하지 않는다.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 기본 지도 조회 | 성수동 중심 좌표, 오늘 08:00~09:00 | 같은 회사 사원의 회사행 카풀 marker 반환 |
| A2 | 반경 검색 | radiusKm=3.0 | 3km 이내 카풀만 |
| A3 | 다인 검색 | passengers=2 | 2석 이상 남은 카풀만 |
| A4 | 근무지 필터 | workplaceId 지정 | 해당 회사 근무지로 향하는 카풀만 반환 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 타회사 카풀 | 다른 companyId 카풀 존재 | 검색 결과에 미포함 |
| E2 | passengers 0 | passengers=0 | `BAD_REQUEST` |
| E3 | 과거 시간 | departureTime=어제 | `BAD_REQUEST` |
| E4 | 반경 과다 | radiusKm > 10 | `BAD_REQUEST: 조회 반경이 너무 큽니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 검색 결과 없음 | 조건에 맞는 카풀이 없음 | `{ nodes: [], totalCount: 0 }` |
| X2 | 만석 카풀 | 모든 좌석이 찬 카풀 | 검색 결과에서 제외 |
| X3 | 본인 카풀 | 자기가 등록한 카풀 | 검색 결과에서 제외 |
| X4 | 같은 위치 다수 | 같은 동네 marker 밀집 | 클라이언트가 cluster 처리할 수 있도록 좌표/개수 제공 |

---

## 기능: 탑승 요청 (`requestRide`)

### 설명

탑승자가 지도 marker 또는 목록 카드에서 선택한 회사행 카풀에 탑승 요청한다. 같은 회사 사원만 요청 가능하며, 요청 시 입력한 픽업 메모는 차주에게 전달한다.

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
| E6 | 회사행 카풀 아님 | legacy/잘못된 목적지 데이터 | `BAD_REQUEST: 회사행 카풀만 요청할 수 있습니다` |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 동시 요청 | 2명이 마지막 1석에 동시 요청 | 먼저 온 요청만 PENDING, 나머지는 만석 에러 |

---

## 기능: 요청 수락/거절 (`acceptRideRequest` / `rejectRideRequest`)

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
