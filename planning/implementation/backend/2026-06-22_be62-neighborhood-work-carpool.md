# BE — 동네 기반 직장 카풀 도메인 전환 구현 계획

## 목표

매칭 도메인을 기존 임의 출발/도착 검색에서 **동네 출발지 → 회사 근무지** 전용 흐름으로 전환합니다. 차주는 지도 기반 출발 위치와 표시용 동네명을 등록하고, 탑승자는 주변 회사행 카풀 제공자를 조회해 요청합니다.

## 관련 이슈

- Docs: `project-ridy/docs#62`
- Backend: 구현 이슈 생성 필요

## 관련 docs 문서

- `docs/planning/PLANNING.md`
- `docs/api/MATCHING.md`
- `docs/api/API.md`
- `docs/architecture/DATABASE.md`
- `docs/WORKFLOW.md`

## 아키텍처 접근

1. GraphQL schema-first로 `Workplace`, `PickupPrivacy`, `nearbyCommuteOffers`를 먼저 추가합니다.
2. Prisma schema에 `Workplace`와 `Ride.workplaceId`, `Ride.pickupLabel`, `Ride.pickupPrivacy`를 추가합니다.
3. 기존 `searchRides`는 하위 호환용으로 유지하되, 신규 지도 화면은 `nearbyCommuteOffers`를 사용합니다.
4. 모든 조회/요청은 `companyId`로 격리하고, `workplace.companyId === user.companyId`를 강제합니다.
5. 승인 전 응답에는 정확한 `departureAddr`를 노출하지 않습니다.

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-BE-NW-001 | A | 차주가 같은 회사 근무지로 회사행 카풀을 생성 | `Ride.status=OPEN`, `workplaceId`, `pickupLabel` 저장 |
| A-BE-NW-002 | A | 탑승자가 주변 회사행 카풀 조회 | 같은 회사/반경/시간대/잔여석 조건에 맞는 offer 반환 |
| A-BE-NW-003 | A | 탑승자가 지도에서 선택한 offer에 요청 | `RideRequest.status=PENDING` 생성 |
| E-BE-NW-001 | E | 타회사 근무지로 카풀 생성 | `FORBIDDEN` |
| E-BE-NW-002 | E | 반경이 정책 한도 초과 | `BAD_REQUEST` |
| E-BE-NW-003 | E | 회사행 카풀이 아닌 legacy/잘못된 목적지 요청 | `BAD_REQUEST` |
| X-BE-NW-001 | X | 같은 위치에 여러 offer 존재 | 모두 반환하되 클라이언트 cluster 가능 필드 제공 |
| X-BE-NW-002 | X | 승인 전 상세 조회 | 정확 주소 대신 `pickupLabel` 중심 응답 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| BE-NW-001 | A-BE-NW-001, E-BE-NW-001 | `prisma/schema.prisma`, migration — `Workplace`, `Ride.workplaceId/pickupLabel/pickupPrivacy` | `test/prisma-schema.spec.ts` 또는 migration validation — `동네 회사행 카풀 스키마가 유효하다` | `npx prisma validate`, `npx prisma generate` 통과 |
| BE-NW-002 | A-BE-NW-001, E-BE-NW-001, E-BE-NW-003 | `src/graphql/schema.graphql`, matching resolver/service — `createRide` 회사 근무지 제약 | `test/matching/create-ride.e2e-spec.ts` — `차주가 같은 회사 근무지로 동네 출발 카풀을 생성한다` | 타회사 근무지/임의 목적지 차단 |
| BE-NW-003 | A-BE-NW-002, E-BE-NW-002, X-BE-NW-001 | matching resolver/service — `nearbyCommuteOffers` | `test/matching/nearby-commute-offers.e2e-spec.ts` — `탑승자가 주변 회사행 카풀을 조회한다` | 회사/반경/시간/좌석 필터 적용 |
| BE-NW-004 | A-BE-NW-003, X-BE-NW-002 | matching resolver/service — `requestRide` privacy-safe request flow | `test/matching/request-ride.e2e-spec.ts` — `탑승자가 지도 선택 offer에 요청한다` | 중복/만석/본인 offer 차단, 승인 전 주소 비노출 |

## 수정/생성할 파일 경로

- `backend/prisma/schema.prisma`
- `backend/src/graphql/schema.graphql`
- `backend/src/matching/*`
- `backend/test/matching/*`

## TDD 순서

1. BE-NW-001: Prisma/schema validation Red → schema/migration Green
2. BE-NW-002: createRide E2E Red → resolver/service Green
3. BE-NW-003: nearbyCommuteOffers E2E Red → geo/company filter Green
4. BE-NW-004: requestRide privacy/edge E2E Red → request flow Green

## 실행할 검증 명령

```bash
npm run codegen
npx prisma validate
npx prisma generate
npm run test
npm run lint
npm run build
```

## 완료 조건

- GraphQL/Prisma schema가 docs와 일치합니다.
- `nearbyCommuteOffers`가 회사/반경/시간/좌석 필터를 적용합니다.
- 승인 전 정확 주소 비노출 원칙을 테스트합니다.
- 모든 Case ID가 구현/테스트에 연결됩니다.
