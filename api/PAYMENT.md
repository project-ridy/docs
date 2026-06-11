# Ridy — 정산/결제 API

## 비용 계산

### POST /payments/calculate

거리 기반 카풀 비용 자동 계산

**Request:**
```json
{
  "departure": { "lat": 37.4979, "lng": 127.0276 },
  "arrival": { "lat": 37.2579, "lng": 127.0524 },
  "passengers": 3
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "distanceKm": 28.5,
    "durationMin": 45,
    "totalFare": 15000,
    "perPassenger": 5000,
    "breakdown": {
      "baseFare": 3000,
      "distanceFare": 12000,
      "platformFee": 0
    }
  }
}
```

## 정산 요청

### POST /payments/settlements

운행 완료 후 정산 요청

**Request:**
```json
{
  "matchingId": "uuid",
  "autoConfirm": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "matchingId": "uuid",
    "totalAmount": 15000,
    "passengers": [
      { "userId": "uuid", "name": "김도윤", "amount": 5000, "status": "PENDING" }
    ],
    "driverAmount": 14250,
    "platformFee": 750,
    "status": "PENDING",
    "dueDate": "2026-06-16T23:59:59+09:00"
  }
}
```

## 정산 확인

### PATCH /payments/settlements/:id/confirm

탑승자가 정산 확인

**Request:**
```json
{
  "paymentMethodId": "uuid"
}
```

## 결제 수단 관리

### GET /payments/methods

등록된 결제 수단 목록

### POST /payments/methods

결제 수단 등록 (토스페이먼츠 빌링키)

**Request:**
```json
{
  "type": "card" | "bank",
  "billingKey": "string",
  "alias": "신한카드"
}
```

## 정산 이력

### GET /payments/history?from=&to=&role=

**Response:**
```json
{
  "success": true,
  "data": {
    "history": [
      {
        "id": "uuid",
        "date": "2026-06-15",
        "matchingId": "uuid",
        "role": "passenger",
        "amount": 5000,
        "status": "COMPLETED"
      }
    ],
    "totalEarned": 142500,
    "totalSpent": 50000
  }
}
```
