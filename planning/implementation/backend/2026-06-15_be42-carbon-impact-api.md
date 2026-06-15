# BE42 개인 친환경 임팩트 GraphQL API 구현 계획

## 목표

- **대상 이슈**: `project-ridy/backend#42` — `[FEAT] 개인 친환경 임팩트 GraphQL API`
- **목적**: `frontend#13` 친환경 임팩트 대시보드가 사용할 `myCarbonImpact`, `carbonHistory` GraphQL API를 schema-first 방식으로 구현한다.
- **범위**: 개인 누적 CO₂ 절감량, 트리 환산, 레벨, 뱃지, 월별 추이, 운행별 절감 내역 조회.
- **범위 제외**: 관리자 ESG 리포트 PDF, 신규 DB 테이블/migration, 이미지 기반 SNS 공유 카드, 배치 집계 최적화.

## 관련 이슈

- Backend: `project-ridy/backend#42` — 개인 친환경 임팩트 GraphQL API
- Docs: `project-ridy/docs#38`, `project-ridy/docs#39`
- Frontend: `project-ridy/frontend#13` — 친환경 임팩트 대시보드

## 관련 docs 문서

- `docs/api/GRAPHQL_GATEWAY.md` — Payment / Analytics SDL의 carbon impact query/type
- `docs/architecture/ARCHITECTURE.md` — 개인 친환경 임팩트 조회, 산출 기준, 레벨/뱃지 기준
- `docs/planning/implementation/frontend/2026-06-15_fe13-eco-impact-dashboard.md`
- `backend/AGENTS.md`

## 현재 스펙 차이와 결정

- 현재 backend schema에는 관리자용 `companyEsgReport(period: String!)`만 있다.
- `frontend#13`은 `myCarbonImpact`, `carbonHistory`를 요구하므로 backend schema/resolver 구현이 선행되어야 한다.
- MVP에서는 개인 임팩트를 위해 신규 Prisma 모델을 추가하지 않는다.
- 개인 임팩트는 `Ride`, `RideRequest`, `User`를 읽어 요청 시 계산한다.
- 완료 운행만 포함한다: `Ride.status === COMPLETED`.
- 사용자가 운전자이거나 `ACCEPTED` 탑승 요청자인 운행만 포함한다.
- 모든 조회는 `viewer.companyId`로 제한하며 클라이언트 입력에서 `companyId`를 받지 않는다.

## 아키텍처 접근

- `src/graphql/schema.graphql`을 먼저 수정한다.
- `npm run codegen`으로 `src/graphql/generated/schema-types.ts`를 갱신한다.
- 기존 analytics/payment 관련 module이 있으면 그 구조를 따른다. 없다면 `src/services/eco-impact/eco-impact.module.ts`, `eco-impact.resolver.ts`, `eco-impact.service.ts`를 추가한다.
- Resolver는 GraphQL entrypoint만 담당하고, 계산/조회 로직은 service에 둔다.
- CO₂ 계산 상수와 레벨/뱃지 판정은 service 내부 private method 또는 `eco-impact.constants.ts`로 분리한다.
- 거리 데이터는 `Ride`의 좌표 기반 haversine 계산 또는 이미 저장된 정산/거리 값이 있으면 기존 값 우선 사용한다. 현재 Prisma `Ride`에는 거리 필드가 없으므로 MVP 기본은 좌표 기반 계산이다.
- 금액/정산 상태와 무관하게 완료된 카풀 참여 사실을 기준으로 계산한다.

## GraphQL SDL 추가

```graphql
type CarbonBadge {
  id: ID!
  label: String!
  description: String!
  achievedAt: DateTime
}

type CarbonImpact {
  period: String!
  totalRides: Int!
  totalDistanceKm: Float!
  co2SavedKg: Float!
  treeEquivalent: Float!
  level: String!
  badges: [CarbonBadge!]!
}

type CarbonHistoryPoint {
  period: String!
  totalRides: Int!
  totalDistanceKm: Float!
  co2SavedKg: Float!
}

type CarbonRideSaving {
  rideId: ID!
  completedAt: DateTime!
  departureAddr: String
  arrivalAddr: String
  distanceKm: Float!
  co2SavedKg: Float!
}

type CarbonHistory {
  monthly: [CarbonHistoryPoint!]!
  rides: [CarbonRideSaving!]!
  pageInfo: PageInfo!
}

extend type Query {
  myCarbonImpact(period: String): CarbonImpact! @auth @companyScope
  carbonHistory(period: String!, pagination: PaginationInput): CarbonHistory! @auth @companyScope
}
```

## 산출 기준

```txt
co2SavedKg = distanceKm * 0.21
treeEquivalent = co2SavedKg / 22
```

레벨 기준:

| 레벨 | 조건 |
|---|---|
| `SEED` | 0kg 이상 |
| `SPROUT` | 10kg 이상 |
| `TREE` | 50kg 이상 |
| `FOREST` | 100kg 이상 |

뱃지 기준:

| 뱃지 ID | 조건 |
|---|---|
| `FIRST_SHARE` | 완료된 카풀 1회 이상 |
| `TEN_KG_SAVER` | CO₂ 10kg 이상 절감 |
| `MONTHLY_STREAK` | 같은 달 완료 카풀 4회 이상 |

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE42-A01 | 완료된 운전자 운행 조회 | `myCarbonImpact`에 totalRides, totalDistanceKm, co2SavedKg 반영 |
| BE42-A02 | 승인된 탑승자 운행 조회 | 탑승자로 참여한 완료 운행도 개인 임팩트에 포함 |
| BE42-A03 | 월별 히스토리 조회 | `carbonHistory.monthly`가 월 단위로 합산되어 반환 |
| BE42-A04 | 운행별 절감 내역 조회 | `carbonHistory.rides`가 최신 완료 운행순으로 반환 |
| BE42-A05 | 레벨/뱃지 산출 | CO₂ 절감량과 완료 횟수에 맞는 level/badges 반환 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE42-E01 | 인증 없음 | `UNAUTHENTICATED` |
| BE42-E02 | 다른 회사 운행 데이터 존재 | 결과에 포함하지 않음 |
| BE42-E03 | 잘못된 period 형식 | `BAD_REQUEST` |
| BE42-E04 | pagination.first가 허용 범위 초과 | 최대값으로 clamp하거나 `BAD_REQUEST` |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE42-X01 | 완료 운행 없음 | 0 값과 빈 badges/monthly/rides 반환 |
| BE42-X02 | 주소 필드 null | null 그대로 반환하고 프론트 fallback에 맡김 |
| BE42-X03 | 같은 사용자가 운전자이면서 요청자 데이터가 중복 존재 | 같은 rideId는 1회만 집계 |
| BE42-X04 | 기간 경계에 걸친 운행 | `departureTime` 또는 완료 기준으로 period 필터가 일관되게 적용 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE42-A01 | `eco-impact.service.ts` | 운전자 완료 운행 집계 | `eco-impact.service.spec.ts` | `운전자로 완료한 운행의 탄소 절감량을 집계한다` | TODO |
| BE42-A02 | `eco-impact.service.ts` | 탑승자 완료 운행 집계 | `eco-impact.service.spec.ts` | `승인된 탑승자로 참여한 운행을 집계한다` | TODO |
| BE42-A03 | `eco-impact.service.ts` | 월별 그룹핑 | `eco-impact.service.spec.ts` | `월별 탄소 절감 히스토리를 반환한다` | TODO |
| BE42-A04 | `eco-impact.service.ts`, `eco-impact.resolver.ts` | 운행별 목록/pagination | `eco-impact.e2e-spec.ts` | `carbonHistory가 운행별 절감 내역을 반환한다` | TODO |
| BE42-A05 | `eco-impact.service.ts` | 레벨/뱃지 판정 | `eco-impact.service.spec.ts` | `절감량과 운행 횟수에 맞는 레벨과 뱃지를 반환한다` | TODO |
| BE42-E01 | `eco-impact.resolver.ts` | auth guard 적용 | `eco-impact.e2e-spec.ts` | `인증 없이는 개인 임팩트를 조회할 수 없다` | TODO |
| BE42-E02 | `eco-impact.service.ts` | company scope 필터 | `eco-impact.service.spec.ts` | `다른 회사 운행을 집계하지 않는다` | TODO |
| BE42-E03 | `eco-impact.service.ts` | period 검증 | `eco-impact.service.spec.ts` | `잘못된 period 형식을 거부한다` | TODO |
| BE42-E04 | `eco-impact.service.ts` | pagination 검증 | `eco-impact.service.spec.ts` | `pagination first가 허용 범위를 넘으면 거부한다` | TODO |
| BE42-X01 | `eco-impact.service.ts` | 빈 결과 | `eco-impact.service.spec.ts` | `완료 운행이 없으면 0 값과 빈 목록을 반환한다` | TODO |
| BE42-X03 | `eco-impact.service.ts` | rideId 중복 제거 | `eco-impact.service.spec.ts` | `같은 rideId는 한 번만 집계한다` | TODO |

## 수정/생성할 파일 경로

- `src/graphql/schema.graphql` — carbon impact SDL/query 추가
- `src/graphql/generated/schema-types.ts` — `npm run codegen` 산출물, 수동 수정 금지
- `src/services/eco-impact/eco-impact.module.ts` — module 등록
- `src/services/eco-impact/eco-impact.resolver.ts` — GraphQL resolver
- `src/services/eco-impact/eco-impact.service.ts` — 개인 임팩트 조회/계산 로직
- `src/services/eco-impact/eco-impact.constants.ts` — 배출계수, 트리 환산, 레벨/뱃지 기준
- `src/services/eco-impact/eco-impact.service.spec.ts` — service 단위 테스트
- `test/eco-impact.e2e-spec.ts` 또는 기존 GraphQL E2E 위치 — resolver/auth/company scope E2E
- `src/app.module.ts` 또는 서비스 모듈 index — module wiring 필요 시

## TDD 순서

1. `eco-impact.service.spec.ts`에 BE42-X01 빈 결과 테스트를 작성해 실패를 확인한다.
2. `src/graphql/schema.graphql`에 carbon SDL/query를 추가하고 `npm run codegen`을 실행한다.
3. `EcoImpactService.getMyCarbonImpact` 최소 구현으로 빈 결과 테스트를 통과시킨다.
4. BE42-A01/A02 운전자/탑승자 집계 테스트를 작성하고 Prisma mock 기반 조회/계산을 구현한다.
5. BE42-E02/X03 company scope와 중복 제거 테스트를 작성하고 필터링을 구현한다.
6. BE42-A03/A04 월별/운행별 히스토리와 pagination 테스트를 작성하고 구현한다.
7. BE42-A05 레벨/뱃지 테스트를 작성하고 constants/helper를 분리한다.
8. `eco-impact.resolver.ts`와 module wiring을 추가한다.
9. GraphQL E2E에서 BE42-E01/A04를 검증한다.
10. Refactor: 좌표 거리 계산, period parsing, badge 판정을 작은 함수로 정리한다.
11. Verify: 전체 검증 명령 실행.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- `myCarbonImpact(period)`가 개인 누적 CO₂ 절감량, 트리 환산, 레벨, 뱃지를 반환한다.
- `carbonHistory(period, pagination)`가 월별 추이와 운행별 절감 내역을 반환한다.
- 인증/회사 범위/기간/pagination 예외가 테스트된다.
- 신규 DB 테이블 없이 현재 Prisma 모델 기반으로 계산한다.
- GraphQL generated 타입을 사용한다.
- 등록표의 TODO 케이스가 구현/테스트와 연결된다.
- PR 본문에 Case ID 확인 표를 포함한다.
- 검증 명령이 통과한다.

## Frontend unblock 조건

- backend PR merge 후 frontend에서 `npm run codegen`이 `EcoImpactDashboard` operation을 생성할 수 있어야 한다.
- frontend#13은 `docs/planning/implementation/frontend/2026-06-15_fe13-eco-impact-dashboard.md`의 FE13-C01~FE13-C06에 맞춰 진행한다.
