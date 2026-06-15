# 매칭 API 개발 기획서

## 개요

- **대상 이슈**: `project-ridy/backend#6` — `[FEAT] 매칭 API 구현`
- **대상 사용자**: 같은 회사 구성원 중 카풀을 등록하는 차주와 카풀을 검색/요청하는 탑승자
- **목적**: 같은 회사 구성원만 대상으로 카풀 등록, 검색, 탑승 요청, 요청 수락/거절을 처리하는 GraphQL API를 구현한다.
- **현재 상태**:
  - `backend/src/graphql/schema.graphql`에는 `Ride`, `RideRequest`, `searchRides`, `myRides`, `createRide`, `requestRide`, `acceptRideRequest`, `rejectRideRequest` 등이 이미 정의되어 있다.
  - `prisma/schema.prisma`에는 `Ride`, `RideRequest`, `RideStatus`, `RequestStatus` 모델/enum이 이미 정의되어 있다.
  - `backend/src/services/matching/matching-service.module.ts`는 빈 모듈 상태다.

## 기능 분해

| 번호 | 하위 기능 | 설명 | 우선순위 |
|---|---|---|---|
| B-01 | MatchingService 생성 | 카풀 등록/검색/요청/처리 비즈니스 로직 구현 | P0 |
| B-02 | MatchingResolver 생성 | GraphQL query/mutation을 서비스에 위임 | P0 |
| B-03 | 거리 기반 검색 | Haversine 거리 계산으로 출발/도착 반경 필터링 | P0 |
| B-04 | 같은 회사 격리 | 모든 조회/변경에서 `companyId` 필터와 소유권 검증 | P0 |
| B-05 | 탑승 요청 | 중복 요청, 본인 카풀 요청, 만석, 취소 상태 예외 처리 | P0 |
| B-06 | 요청 수락/거절 | 차주 소유권 검증, 트랜잭션으로 좌석 감소/상태 변경 | P0 |
| B-07 | 목록 조회 | `myRides`, `myRideRequests`, `ride` 조회 구현 | P1 |
| B-08 | 정기 카풀 처리 | `recurringDays`, `recurringEnd` 저장과 검색 포함 | P1 |

## 코드 구조

### 백엔드

- 모듈: `src/services/matching/matching-service.module.ts`
- 리졸버: `src/services/matching/matching.resolver.ts`
- 서비스: `src/services/matching/matching.service.ts`
- 테스트:
  - `src/services/matching/matching.service.spec.ts`
  - 후속 E2E: `test/matching.e2e-spec.ts`
- 기존 스키마:
  - `src/graphql/schema.graphql`
  - `src/graphql/generated/schema-types.ts`

### 프론트엔드

- 이번 백엔드 작업에서 프론트엔드 코드는 수정하지 않는다.
- 후속으로 `frontend#5`의 `myRides/searchRides` GraphQL 연동을 진행한다.

## 상세 설계

### B-01/B-02: 카풀 등록

#### 정상 흐름
1. 인증 사용자가 `createRide(input)`을 호출한다.
2. 서비스가 사용자 role이 `DRIVER`, `BOTH`, `ADMIN` 중 하나인지 확인한다.
3. 좌석, 시간, 좌표 유효성을 검증한다.
4. 사용자의 `companyId`와 `id`를 기준으로 `Ride`를 생성한다.

#### 함수 시그니처

```typescript
async function createRide(
  currentUser: CurrentUser,
  input: CreateRideInput,
): Promise<Ride>;
```

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 비차주 | `role=PASSENGER` | `ForbiddenException` |
| E-02 | 좌석 0 이하 | `availableSeats < 1` | `BadRequestException` |
| E-03 | 과거 출발 시간 | `departureTime <= now` | `BadRequestException` |
| E-04 | 출발지와 도착지 동일 | 좌표가 동일 | `BadRequestException` |

### B-03: 카풀 검색

#### 정상 흐름
1. 인증 사용자가 `searchRides(input, pagination)`을 호출한다.
2. 같은 회사, `OPEN`, 잔여 좌석, 출발 시간 이후 조건으로 후보를 조회한다.
3. Haversine 거리로 출발지/도착지 반경을 필터링한다.
4. 본인이 등록한 카풀은 제외한다.
5. `RideConnection` 형태로 반환한다.

#### 함수 시그니처

```typescript
async function searchRides(
  currentUser: CurrentUser,
  input: SearchRidesInput,
  pagination?: PaginationInput | null,
): Promise<RideConnection>;
```

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 탑승 인원 0 이하 | `passengers < 1` | `BadRequestException` |
| E-02 | 과거 출발 시간 | `departureTime <= now` | `BadRequestException` |
| E-03 | 다른 회사 카풀 | 후보에 존재 | 결과에서 제외 |
| E-04 | 본인 카풀 | `driverId=currentUser.id` | 결과에서 제외 |

### B-05: 탑승 요청

#### 정상 흐름
1. 탑승자가 `requestRide(input)`을 호출한다.
2. 서비스가 같은 회사 카풀인지 확인한다.
3. 만석, 본인 카풀, 취소된 카풀, 중복 요청 여부를 검증한다.
4. `RideRequest`를 `PENDING` 상태로 생성한다.

#### 함수 시그니처

```typescript
async function requestRide(
  currentUser: CurrentUser,
  input: RequestRideInput,
): Promise<RideRequest>;
```

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 카풀 없음/타회사 | `ride.companyId !== currentUser.companyId` | `NotFoundException` |
| E-02 | 본인 카풀 | `ride.driverId === currentUser.id` | `BadRequestException` |
| E-03 | 만석 | `availableSeats <= 0` | `BadRequestException` |
| E-04 | 취소/종료 | `status !== OPEN` | `BadRequestException` |
| E-05 | 중복 요청 | 기존 PENDING/ACCEPTED 요청 존재 | `ConflictException` |

### B-06: 요청 수락/거절

#### 정상 흐름
1. 차주가 `acceptRideRequest(id)` 또는 `rejectRideRequest(id)`를 호출한다.
2. 요청의 카풀이 본인 소유이면서 같은 회사인지 확인한다.
3. 이미 처리된 요청은 거부한다.
4. 수락은 트랜잭션으로 요청 상태 변경과 좌석 감소를 함께 수행한다.
5. 좌석이 0이 되면 카풀 상태를 `MATCHED`로 변경한다.

#### 함수 시그니처

```typescript
async function acceptRideRequest(
  currentUser: CurrentUser,
  id: string,
): Promise<RideRequest>;

async function rejectRideRequest(
  currentUser: CurrentUser,
  id: string,
): Promise<RideRequest>;
```

## 테스트 시나리오

### 유닛 테스트

| 테스트 | 대상 | 설명 |
|---|---|---|
| BE-UT-01 | `createRide` | 차주가 정상 입력으로 카풀을 생성한다 |
| BE-UT-02 | `createRide` | 탑승자 role은 카풀을 생성할 수 없다 |
| BE-UT-03 | `searchRides` | 같은 회사, 반경, 좌석 조건에 맞는 카풀만 반환한다 |
| BE-UT-04 | `searchRides` | 타회사/본인/만석 카풀은 제외한다 |
| BE-UT-05 | `requestRide` | 정상 탑승 요청을 생성한다 |
| BE-UT-06 | `requestRide` | 중복 요청은 `ConflictException` |
| BE-UT-07 | `acceptRideRequest` | 요청 수락 시 좌석을 감소시키고 요청 상태를 변경한다 |
| BE-UT-08 | `rejectRideRequest` | 요청 거절 시 요청 상태만 변경한다 |

### E2E 테스트

| 테스트 | 대상 | 설명 |
|---|---|---|
| BE-E2E-01 | GraphQL | `createRide` mutation 동작 |
| BE-E2E-02 | GraphQL | `searchRides` query 동작 |
| BE-E2E-03 | GraphQL | `requestRide` → `acceptRideRequest` 흐름 |

## 의존성

- 선행 완료 필요:
  - `project-ridy/backend#2` Prisma 스키마 + 마이그레이션
  - `project-ridy/backend#3` 인증 API
- 후속:
  - `project-ridy/backend#7` 매칭 API 테스트
  - `project-ridy/frontend#5` 홈 화면 GraphQL 연동
  - `project-ridy/frontend#6` 매칭 결과/상세 화면
- 관련 문서:
  - `docs/api/MATCHING.md`
  - `docs/architecture/DATABASE.md`
  - `docs/architecture/ARCHITECTURE.md`

## 완료 조건

- `MatchingServiceModule`이 resolver/service를 제공한다.
- `createRide`, `searchRides`, `ride`, `myRides`, `myRideRequests`, `requestRide`, `acceptRideRequest`, `rejectRideRequest`, `cancelRideRequest`가 동작한다.
- 같은 회사 격리와 소유권 검증이 적용된다.
- Haversine 기반 반경 검색이 적용된다.
- 서비스 유닛 테스트가 핵심 정상/예외 케이스를 검증한다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`를 통과한다.
