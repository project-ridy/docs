# Ridy — 시스템 아키텍처

## 전체 구성도

```
                    ┌──────────────┐
                    │   Client     │
                    │  (Next.js /  │
                    │  React Native│
                    └──────┬───────┘
                           │ HTTPS / WSS
                    ┌──────▼───────┐
                    │  API Gateway │
                    │  (Nginx)     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼──────┐ ┌──▼──────┐
       │  Auth       │ │ Matching│ │  Chat   │
       │  Service    │ │ Service │ │ Service │
       │  (NestJS)   │ │(NestJS) │ │(NestJS) │
       └──────┬──────┘ └──┬──────┘ └──┬──────┘
              │           │           │
       ┌──────▼───────────▼───────────▼──────┐
       │           PostgreSQL                 │
       │    (users, matchings, settlements)   │
       └──────┬──────────────────────────────┘
              │
       ┌──────▼──────┐    ┌──────────────┐
       │   Redis     │    │  S3 (R2)     │
       │ (캐시, 세션) │    │ (이미지, 파일)│
       └─────────────┘    └──────────────┘
```

## 기술 스택

### 프론트엔드
| 기술 | 용도 |
|---|---|
| Next.js 15 (App Router) | 웹 서비스 |
| React Native + Expo | 모바일 앱 |
| TypeScript | 타입 안전성 |
| Tailwind CSS | 스타일링 |
| TanStack Query | 서버 상태 관리 |
| Socket.IO Client | 실시간 채팅 |

### 백엔드
| 기술 | 용도 |
|---|---|
| NestJS | API 프레임워크 |
| TypeScript | 타입 안전성 |
| Prisma | ORM |
| PostgreSQL | 메인 DB |
| Redis | 캐시, 세션, Pub/Sub |
| Socket.IO | WebSocket 채팅 |
| Bull | 작업 큐 (알림, 정산) |

### 인프라
| 기술 | 용도 |
|---|---|
| AWS ECS (Fargate) | 컨테이너 오케스트레이션 |
| AWS RDS (PostgreSQL) | 관리형 DB |
| AWS ElastiCache (Redis) | 관리형 Redis |
| AWS S3 / CloudFront | 정적 자산 CDN |
| AWS SES | 이메일 발송 |
| GitHub Actions | CI/CD |

### 외부 서비스
| 서비스 | 용도 |
|---|---|
| 카카오/구글/Apple | 소셜 로그인 |
| 토스페이먼츠 | 결제 |
| AWS SNS | 푸시 알림 |
| Google Maps / Mapbox | 지도, 경로 |
| CoolSMS | 문자 인증 |

## 마이크로서비스 경계

| 서비스 | 책임 | DB 스키마 |
|---|---|---|
| Auth | 인증, 인가, 세션 | users, sessions |
| Matching | 카풀 등록, 검색, 매칭 | rides, matchings, requests |
| Chat | 실시간 메시지 | messages, rooms |
| Payment | 정산, 결제 | settlements, payments |

> MVP에서는 모놀리식 NestJS로 시작, 트래픽 증가 시 서비스 분리
