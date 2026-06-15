# 매칭 API 테스트 기획서

## 개요

- **대상 이슈**: `project-ridy/backend#7` — `[TEST] 매칭 API 테스트`
- **관련 기능 이슈**: `project-ridy/backend#6` — `[FEAT] 매칭 API 구현`
- **목적**: 매칭 API의 서비스 단위 테스트를 보강하고 GraphQL E2E 테스트를 추가해 카풀 등록, 검색, 요청, 수락/거절 흐름을 검증한다.
- **대상 사용자**: 같은 회사 안에서 카풀을 등록하는 차주와 탑승 요청을 보내는 탑승자

## 기능 분해

| 번호 | 하위 기능 | 설명 | 우선순위 |
|---|---|---|---|
| T-01 | 서비스 테스트 보강 | 좌석 초과, 본인 카풀 요청, 상태 예외, 목록/취소 케이스 추가 | P0 |
| T-02 | GraphQL 인증 컨텍스트 검증 | Authorization 헤더가 `currentUser`로 연결되는지 E2E로 검증 | P0 |
| T-03 | GraphQL 생성/검색 E2E | `createRide`, `searchRides` query/mutation 응답 검증 | P0 |
| T-04 | GraphQL 요청 처리 E2E | `requestRide` → `acceptRideRequest` 흐름 검증 | P0 |
| T-05 | 회귀 검증 | 기존 테스트, lint, build 전체 통과 확인 | P0 |

## 코드 구조

### 백엔드

- 기존 서비스 테스트 보강: `src/services/matching/matching.service.spec.ts`
- 신규 GraphQL E2E 테스트: `test/matching.e2e-spec.ts`
- 필요 시 인증 컨텍스트 보정: `src/gateway/graphql/graphql-gateway.module.ts`

## 상세 설계

### T-01: 서비스 테스트 보강

#### 정상/예외 흐름
1. `createRide`는 차주 권한, 미래 시간, 좌석 수 조건을 검증한다.
2. `requestRide`는 같은 회사, 열린 카풀, 잔여 좌석, 중복 요청을 검증한다.
3. `acceptRideRequest`는 차주 소유권과 잔여 좌석을 검증하고 트랜잭션으로 처리한다.
4. `cancelRideRequest`는 요청자 본인의 대기 요청만 취소한다.

#### 예외 처리

| 케이스 | 조건 | 기대 결과 |
|---|---|---|
| E-01 | 좌석 0 이하 생성 | `BadRequestException` |
| E-02 | 본인 카풀 요청 | `BadRequestException` |
| E-03 | 만석 카풀 요청 | `BadRequestException` |
| E-04 | 다른 회사 카풀 요청 | `NotFoundException` |
| E-05 | 수락 시 남은 좌석 없음 | `BadRequestException` |
| E-06 | 다른 사용자의 요청 취소 | `NotFoundException` |

### T-02/T-03/T-04: GraphQL E2E

#### 정상 흐름
1. 테스트가 JWT access token을 발급해 `Authorization: Bearer <token>` 헤더를 보낸다.
2. GraphQL context가 토큰 payload를 `currentUser`로 변환한다.
3. `createRide` mutation이 차주 사용자로 카풀을 생성한다.
4. `searchRides` query가 탑승자 사용자로 같은 회사/반경 조건 결과를 반환한다.
5. `requestRide` mutation으로 요청을 만들고 `acceptRideRequest` mutation으로 수락한다.

#### 엣지 케이스

- 인증 헤더가 없으면 매칭 API는 `UnauthorizedException`을 반환한다.
- 다른 회사 카풀은 GraphQL 응답에서 노출되지 않는다.
- E2E는 실제 DB 대신 `PrismaService` mock을 사용해 GraphQL resolver/service 연결을 검증한다.

## 테스트 시나리오

### 유닛 테스트

| 테스트 ID | 대상 | 설명 |
|---|---|---|
| BE-UT-01 | `createRide` | 차주가 정상 입력으로 카풀을 생성한다 |
| BE-UT-02 | `createRide` | 탑승자 role은 카풀을 생성할 수 없다 |
| BE-UT-03 | `createRide` | 좌석 0 이하는 실패한다 |
| BE-UT-04 | `searchRides` | 같은 회사, 반경, 좌석 조건에 맞는 카풀만 반환한다 |
| BE-UT-05 | `searchRides` | 탑승 인원 1 미만은 실패한다 |
| BE-UT-06 | `requestRide` | 정상 탑승 요청을 생성한다 |
| BE-UT-07 | `requestRide` | 중복 요청은 실패한다 |
| BE-UT-08 | `requestRide` | 본인 카풀 요청과 만석 요청은 실패한다 |
| BE-UT-09 | `acceptRideRequest` | 요청 수락 시 좌석을 감소시키고 요청 상태를 변경한다 |
| BE-UT-10 | `acceptRideRequest` | 남은 좌석이 없으면 실패한다 |
| BE-UT-11 | `rejectRideRequest` | 요청 거절 시 요청 상태만 변경한다 |
| BE-UT-12 | `cancelRideRequest` | 본인의 대기 요청만 취소한다 |

### E2E 테스트

| 테스트 ID | 대상 | 설명 |
|---|---|---|
| BE-E2E-01 | GraphQL auth context | Authorization 헤더로 매칭 mutation이 인증 사용자를 인식한다 |
| BE-E2E-02 | GraphQL createRide | 차주가 카풀을 생성하고 좌표/좌석 응답을 받는다 |
| BE-E2E-03 | GraphQL searchRides | 탑승자가 반경 내 열린 카풀을 검색한다 |
| BE-E2E-04 | GraphQL request/accept | 탑승 요청 생성 후 차주가 수락한다 |

## 의존성

- 선행 완료:
  - `project-ridy/backend#6` 매칭 API 구현
- 관련 문서:
  - `docs/api/MATCHING.md`
  - `docs/architecture/DATABASE.md`
  - `docs/planning/implementation/backend/2026-06-12_be06-matching-api.md`

## 검증 명령

```bash
npm run codegen
npm run test -- matching.service.spec.ts
npm run test:e2e -- matching.e2e-spec.ts
npm run test
npm run lint
npm run build
```

## 완료 조건

- 서비스 테스트가 이슈 #7의 HIGH 우선순위 케이스를 모두 포함한다.
- GraphQL E2E 테스트가 매칭 resolver/service 연결과 인증 context를 검증한다.
- 기존 테스트 회귀가 없다.
- `npm run test`, `npm run lint`, `npm run build`가 모두 통과한다.
