# 모니터링 및 로깅 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#14` — `[FEAT] 모니터링 및 로깅`
- **목적**: 운영 환경에서 backend 상태를 확인하고 Prometheus가 수집 가능한 메트릭과 JSON 구조화 로그를 제공한다.
- **범위**: GraphQL `health` 확장, REST `/health/live`, `/health/ready`, `/metrics`, 요청 로깅 middleware, 구조화 logger, 테스트.
- **범위 제외**: 실제 Redis 연결 확인, 실제 Sentry SDK 전송, Slack/Discord webhook 발송, Dockerfile healthcheck 추가. 현재 backend repo에 Redis/Sentry/webhook/Docker 설정이 없으므로 후속 이슈로 분리한다.

## 관련 이슈

- `project-ridy/backend#14` — 모니터링 및 로깅

## 관련 docs 문서

- `docs/api/GRAPHQL_GATEWAY.md` — `/health/live`, `/health/ready`
- `docs/architecture/MSA.md` — structured log, correlationId, readiness
- `docs/architecture/INFRA.md` — Sentry, AWS 운영 인프라

## 현재 제약과 결정

- 현재 backend에는 Dockerfile이 없다. Docker healthcheck는 이번 PR에서 추가하지 않는다.
- 현재 backend에는 Redis client 설정이 없다. Redis readiness는 `disabled`로 명시하고 실제 Redis check는 후속으로 둔다.
- 현재 backend에는 Sentry SDK 의존성이 없다. 실제 Sentry 전송은 후속으로 두고, 이번에는 JSON error log와 설정 확장 지점을 만든다.
- 새 외부 의존성은 최소화한다. Prometheus text format은 직접 생성 가능한 기본 메트릭으로 시작한다.
- `health` GraphQL 타입은 기존 클라이언트 호환을 위해 `status`, `service`를 유지하고 `database`, `redis`, `uptimeSec`, `timestamp`를 추가한다.

## 아키텍처 접근

- `HealthService`가 liveness/readiness/GraphQL health payload를 만든다.
- DB readiness는 `PrismaService.$queryRaw` 또는 `$connect` mock 가능 메서드로 확인한다.
- Redis readiness는 config가 없으므로 `{ status: "disabled" }`로 반환한다.
- `HealthController`를 추가해 `GET /health/live`, `GET /health/ready`를 제공한다.
- `MetricsController`를 추가해 `GET /metrics`에서 Prometheus text/plain payload를 반환한다.
- `MetricsService`는 process uptime, memory, request count, request duration sum/count를 보관한다.
- `RequestLoggingMiddleware`는 요청 완료 시 JSON 로그를 남기고 metrics counter를 증가시킨다.
- GraphQL operation name 추적은 HTTP body의 `operationName`을 best-effort로 읽는다. body parser 순서에 따라 없을 수 있으므로 필수값으로 가정하지 않는다.

## API 설계

### GraphQL Health 확장

```graphql
type DependencyHealth {
  status: String!
  latencyMs: Int
  message: String
}

type Health {
  status: String!
  service: String!
  database: DependencyHealth!
  redis: DependencyHealth!
  uptimeSec: Int!
  timestamp: DateTime!
}
```

### REST Health

```http
GET /health/live
```

```json
{
  "status": "ok",
  "service": "ridy-backend",
  "uptimeSec": 123
}
```

```http
GET /health/ready
```

```json
{
  "status": "ok",
  "service": "ridy-backend",
  "database": { "status": "ok", "latencyMs": 3 },
  "redis": { "status": "disabled", "message": "Redis client is not configured" }
}
```

### Metrics

```http
GET /metrics
```

```txt
# HELP ridy_process_uptime_seconds Process uptime in seconds
# TYPE ridy_process_uptime_seconds gauge
ridy_process_uptime_seconds 123
```

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE14-A01 | GraphQL health 조회 | service/status/database/redis/uptime/timestamp 반환 |
| BE14-A02 | `/health/live` 조회 | 프로세스 생존 상태 반환 |
| BE14-A03 | `/health/ready` DB 정상 | status `ok`, database `ok` 반환 |
| BE14-A04 | `/metrics` 조회 | Prometheus text format 반환 |
| BE14-A05 | 요청 완료 | JSON request log와 metrics count 증가 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE14-E01 | DB readiness 실패 | ready status `degraded`, database `error` |
| BE14-E02 | metrics request | 로깅 middleware가 민감한 body/header를 출력하지 않음 |
| BE14-E03 | GraphQL operationName 없음 | `operationName=null`로 안전 처리 |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE14-X01 | Redis 미설정 | redis `disabled`, ready 전체는 DB 정상 시 `ok` |
| BE14-X02 | 매우 빠른 요청 | durationMs 0 이상 정수 기록 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE14-A01 | `health.service.ts`, `health.resolver.ts` | GraphQL health | `health.service.spec.ts`, `health.e2e-spec.ts` | `health 쿼리는 의존성 상태를 반환한다` | TODO |
| BE14-A02 | `health.controller.ts` | live endpoint | `health.e2e-spec.ts` | `live endpoint는 생존 상태를 반환한다` | TODO |
| BE14-A03 | `health.controller.ts` | ready endpoint | `health.e2e-spec.ts` | `ready endpoint는 DB 상태를 반환한다` | TODO |
| BE14-A04 | `metrics.controller.ts` | metrics endpoint | `monitoring.e2e-spec.ts` | `metrics endpoint는 Prometheus 포맷을 반환한다` | TODO |
| BE14-A05 | `request-logging.middleware.ts`, `metrics.service.ts` | request observe | `metrics.service.spec.ts` | `요청 메트릭을 기록한다` | TODO |
| BE14-E01 | `health.service.ts` | DB failure | `health.service.spec.ts` | `DB 실패 시 degraded 상태를 반환한다` | TODO |
| BE14-X01 | `health.service.ts` | Redis disabled | `health.service.spec.ts` | `Redis 미설정 상태를 disabled로 반환한다` | TODO |

## 수정/생성할 파일 경로

- `src/graphql/schema.graphql` — `DependencyHealth`, `Health` 확장
- `src/graphql/generated/schema-types.ts` — `npm run codegen` 산출물
- `src/services/health/health.service.ts` — dependency readiness
- `src/services/health/health.resolver.ts` — async health query
- `src/services/health/health.controller.ts` — REST health endpoints
- `src/services/health/dto/health-response.dto.ts` — DTO 확장
- `src/services/health/health.service.spec.ts` — health unit tests
- `src/common/metrics/metrics.service.ts` — in-process metrics registry
- `src/common/metrics/metrics.controller.ts` — `/metrics`
- `src/common/metrics/metrics.module.ts` — metrics module
- `src/common/logging/structured-logger.service.ts` — JSON logger
- `src/common/logging/request-logging.middleware.ts` — request logging middleware
- `src/common/logging/logging.module.ts` — logging module
- `src/app/app.module.ts` — controller/module/middleware 등록
- `test/health.e2e-spec.ts` — REST health E2E 보강
- `test/monitoring.e2e-spec.ts` — metrics E2E

## TDD 순서

1. `health.service.spec.ts`에 DB 정상/실패/Redis disabled 테스트를 작성한다.
2. `schema.graphql`을 확장하고 `npm run codegen`을 실행한다.
3. `HealthService`와 resolver/controller를 구현해 unit tests를 통과시킨다.
4. `MetricsService` unit test를 작성하고 in-process metrics를 구현한다.
5. `/metrics` E2E를 추가하고 controller/module을 연결한다.
6. request logging middleware는 민감 정보 없이 method/path/status/duration/operationName만 기록하도록 구현한다.
7. 전체 검증을 실행한다.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run test:e2e
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- GraphQL `health`가 DB/Redis/uptime/timestamp를 반환한다.
- `/health/live`, `/health/ready`, `/metrics`가 동작한다.
- 요청 메트릭과 JSON 구조화 로그가 동작한다.
- 민감한 body/header를 로그에 남기지 않는다.
- Redis/Sentry/Slack/Docker 미구현 범위가 PR 본문에 후속 작업으로 명시된다.
- 검증 명령이 통과한다.
