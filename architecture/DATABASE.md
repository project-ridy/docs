# Ridy — 데이터베이스 스키마

## ERD

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│    users     │     │      rides       │     │   ride_requests │
├─────────────┤     ├──────────────────┤     ├─────────────────┤
│ id (PK)     │──┐  │ id (PK)          │──┐  │ id (PK)         │
│ email       │  │  │ driver_id (FK)   │  │  │ ride_id (FK)    │
│ name        │  └──│ departure_lat    │  └──│ passenger_id(FK)│
│ phone       │     │ departure_lng    │     │ pickup_lat      │
│ image_url   │     │ departure_addr   │     │ pickup_lng      │
│ role        │     │ arrival_lat      │     │ pickup_addr     │
│ rating      │     │ arrival_lng      │     │ message         │
│ ride_count  │     │ arrival_addr     │     │ status          │
│ created_at  │     │ departure_time   │     │ created_at      │
│ updated_at  │     │ available_seats  │     │ updated_at      │
└─────────────┘     │ fare             │     └─────────────────┘
                    │ recurring_days   │
┌─────────────┐     │ recurring_end    │
│  vehicles   │     │ preferences      │     ┌─────────────────┐
├─────────────┤     │ status           │     │   chat_rooms    │
│ id (PK)     │     │ created_at       │     ├─────────────────┤
│ user_id(FK) │     └──────────────────┘     │ id (PK)         │
│ model       │                              │ ride_id (FK)    │
│ color       │     ┌──────────────────┐     │ created_at      │
│ plate       │     │   settlements    │     └────────┬────────┘
│ capacity    │     ├──────────────────┤              │
│ created_at  │     │ id (PK)          │     ┌────────▼────────┐
└─────────────┘     │ ride_id (FK)     │     │   messages      │
                    │ passenger_id(FK) │     ├─────────────────┤
┌─────────────┐     │ amount           │     │ id (PK)         │
│   reviews   │     │ driver_amount    │     │ room_id (FK)    │
├─────────────┤     │ platform_fee     │     │ sender_id (FK)  │
│ id (PK)     │     │ status           │     │ content         │
│ ride_id(FK) │     │ due_date         │     │ type            │
│ from_id(FK) │     │ paid_at          │     │ created_at      │
│ to_id (FK)  │     │ created_at       │     └─────────────────┘
│ rating      │     └──────────────────┘
│ comment     │
│ created_at  │     ┌──────────────────┐
└─────────────┘     │  payment_methods │
                    ├──────────────────┤
                    │ id (PK)          │
                    │ user_id (FK)     │
                    │ type             │
                    │ billing_key      │
                    │ alias            │
                    │ is_default       │
                    │ created_at       │
                    └──────────────────┘
```

## 주요 테이블 상세

### users
| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 유저 고유 ID |
| email | VARCHAR(255) | UNIQUE | 이메일 |
| name | VARCHAR(100) | NOT NULL | 이름 |
| phone | VARCHAR(20) | | 전화번호 |
| image_url | TEXT | | 프로필 이미지 |
| provider | ENUM | | kakao, google, apple |
| provider_id | VARCHAR(255) | | 소셜 ID |
| role | ENUM | DEFAULT passenger | passenger, driver, both |
| rating | DECIMAL(2,1) | DEFAULT 0.0 | 평균 평점 |
| ride_count | INT | DEFAULT 0 | 운행 횟수 |
| phone_verified | BOOLEAN | DEFAULT false | 휴대폰 인증 여부 |
| created_at | TIMESTAMP | | 생성일 |
| updated_at | TIMESTAMP | | 수정일 |

### rides
| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK | 운행 ID |
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
