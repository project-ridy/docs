# Ridy — 매칭 API

## 카풀 등록 (차주)

### POST /matchings/rides

새로운 카풀 운행 등록

**Request:**
```json
{
  "departure": {
    "lat": 37.4979,
    "lng": 127.0276,
    "address": "강남역",
    "name": "강남역 2번출구"
  },
  "arrival": {
    "lat": 37.2579,
    "lng": 127.0524,
    "address": "수원역",
    "name": "수원역 북측 광장"
  },
  "departureTime": "2026-06-15T08:30:00+09:00",
  "availableSeats": 3,
  "recurring": {
    "days": ["MON", "TUE", "WED", "THU", "FRI"],
    "endDate": "2026-08-15"
  },
  "fare": 5000,
  "preferences": {
    "smoking": false,
    "pet": false,
    "luggage": true,
    "chatLevel": "free"
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "driver": { "id": "uuid", "name": "박준서", "rating": 4.8 },
    "departure": { ... },
    "arrival": { ... },
    "departureTime": "2026-06-15T08:30:00+09:00",
    "availableSeats": 3,
    "currentPassengers": 0,
    "status": "OPEN",
    "fare": 5000
  }
}
```

## 카풀 검색 (탑승자)

### GET /matchings/search?depLat=&depLng=&arrLat=&arrLng&time=&seats=

출발지/도착지/시간 기반 매칭 검색

**Query Params:**
| 파라미터 | 필수 | 설명 |
|---|---|---|
| depLat | O | 출발지 위도 |
| depLng | O | 출발지 경도 |
| arrLat | O | 도착지 위도 |
| arrLng | O | 도착지 경도 |
| time | O | 희망 출발 시간 (ISO 8601) |
| seats | X | 필요 좌석 수 (기본 1) |
| radius | X | 검색 반경 km (기본 3) |

**Response:**
```json
{
  "success": true,
  "data": {
    "matchings": [
      {
        "id": "uuid",
        "driver": { "id": "uuid", "name": "박준서", "rating": 4.8, "rideCount": 42 },
        "departure": { "address": "강남역", "time": "08:30" },
        "arrival": { "address": "수원역", "time": "09:20" },
        "availableSeats": 2,
        "fare": 5000,
        "detourMinutes": 5,
        "preferences": { ... }
      }
    ],
    "total": 5
  }
}
```

## 매칭 요청

### POST /matchings/:id/request

탑승자가 카풀에 참가 요청

**Request:**
```json
{
  "message": "안녕하세요! 매일 강남 가시는군요, 탑승하고 싶습니다.",
  "pickupStop": { "lat": 37.49, "lng": 127.02, "address": "역삼역" }
}
```

## 매칭 수락/거절

### PATCH /matchings/:id/requests/:requestId

**Request:**
```json
{
  "action": "accept" | "reject"
}
```

## 내 매칭 목록

### GET /matchings/mine?status=&role=

| status | 설명 |
|---|---|
| OPEN | 모집 중 |
| MATCHED | 매칭 완료 |
| IN_PROGRESS | 운행 중 |
| COMPLETED | 완료 |
| CANCELLED | 취소 |

| role | 설명 |
|---|---|
| driver | 차주 |
| passenger | 탑승자 |
