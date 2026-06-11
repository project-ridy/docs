# Ridy — 인프라 구성

## AWS 아키텍처 (MVP)

```
                    ┌─────────────────┐
                    │   Route 53      │
                    │  (DNS + CDN)    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  CloudFront     │
                    │  (정적 자산 CDN) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
       ┌──────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
       │  Next.js    │ │  NestJS  │ │  Socket.IO  │
       │  (ECS)      │ │  (ECS)   │ │  (ECS)      │
       └──────┬──────┘ └────┬─────┘ └──────┬──────┘
              │             │              │
       ┌──────▼─────────────▼──────────────▼──────┐
       │              VPC (10.0.0.0/16)            │
       │                                          │
       │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
       │  │   RDS    │  │  Redis   │  │  S3    │ │
       │  │PostgreSQL│  │ElastiCach│  │ (R2)   │ │
       │  └──────────┘  └──────────┘  └────────┘ │
       └──────────────────────────────────────────┘
```

## 환경

| 환경 | 용도 | 도메인 |
|---|---|---|
| dev | 개발 | dev-api.ridy.dev |
| staging | 스테이징 | staging-api.ridy.dev |
| prod | 프로덕션 | api.ridy.dev |

## CI/CD 파이프라인

```
Push to main → GitHub Actions → Test → Build Docker → Push ECR → Deploy ECS
```

### frontend 파이프라인
1. `main` 브랜치 PR → Lint + Type Check + Test
2. Merge → Build → S3 업로드 → CloudFront 무효화

### backend 파이프라인
1. `main` 브랜치 PR → Lint + Unit Test + Integration Test
2. Merge → Docker Build → ECR Push → ECS Rolling Deploy

## 모니터링

| 도구 | 용도 |
|---|---|
| CloudWatch | 로그, 메트릭, 알람 |
| Sentry | 에러 트래킹 |
| Datadog (선택) | APM, 트레이싱 |

## 시크릿 관리

- AWS Secrets Manager: DB 비밀번호, API 키
- GitHub Secrets: CI/CD용 환경 변수
