# 매칭 결과/상세 화면 개발 기획서

## 개요

- **대상 이슈**: `project-ridy/frontend#6` — `[FEAT] 매칭 결과/상세 화면`
- **대상 사용자**: 홈에서 출발지/도착지/시간을 입력한 탑승자
- **목적**: 같은 회사 동료의 카풀 검색 결과를 비교하고, 상세 화면에서 차주 정보와 경로를 확인한 뒤 탑승 요청을 보낼 수 있게 한다.
- **선행 완료**:
  - `project-ridy/backend#6` 매칭 API
  - `project-ridy/backend#7` 매칭 API 테스트
  - `project-ridy/frontend#5` 홈 화면 GraphQL 연동

## 기능 분해

| 번호 | 하위 기능 | 설명 | 우선순위 |
|---|---|---|---|
| M-01 | 검색 결과 페이지 | `/matchings`에서 `searchRides` 결과를 카드 리스트로 표시 | P0 |
| M-02 | 정렬 | 출발시간순, 요금순, 평점순 정렬 | P0 |
| M-03 | 빈/로딩/에러 상태 | 검색 결과 없음, 로딩, 요청 실패 상태 표시 | P0 |
| M-04 | 상세 페이지 | `/matchings/:id`에서 `ride` 상세 조회 | P0 |
| M-05 | 탑승 요청 | `requestRide` mutation과 메시지 입력 모달 연결 | P0 |
| M-06 | 하단 내비게이션 | 검색 탭 active 상태 유지 | P1 |

## 코드 구조

### 프론트엔드

- GraphQL operation: `src/graphql/operations/matching.graphql`
  - `RideDetail`
  - `RequestRide`
- API: `lib/api/matching-api.ts`
  - `fetchRideDetail`
  - `requestRide`
- Hook: `hooks/useMatchingQueries.ts`
  - `useRideDetailQuery`
  - `useRequestRideMutation`
- Pages:
  - `app/matchings/page.tsx`
  - `app/matchings/[id]/page.tsx`
- Tests:
  - `__tests__/Matchings.test.tsx`

## 상세 설계

### M-01: 검색 결과 페이지

#### 정상 흐름
1. 홈에서 전달된 `departure`, `destination`, `departureTime` query를 읽는다.
2. 현재 백엔드 검색은 좌표 기반이므로, 1차 화면에서는 기본 좌표를 사용해 `searchRides`를 호출한다.
3. 결과를 `MatchingCard` 리스트로 표시한다.
4. 카드를 누르면 `/matchings/:id`로 이동한다.

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 검색 결과 없음 | 빈 상태 표시 |
| E-02 | GraphQL 실패 | 에러 문구와 재시도 버튼 표시 |
| E-03 | 주소 query 없음 | 제목은 기본 “매칭 결과”로 표시 |

### M-02: 정렬

#### 정상 흐름
1. 사용자가 정렬 segmented control을 선택한다.
2. 클라이언트에서 현재 결과 배열을 정렬한다.
3. 카드 리스트가 즉시 재정렬된다.

| 정렬 | 기준 |
|---|---|
| 출발시간순 | `departureTime` 오름차순 |
| 요금순 | `fare` 오름차순 |
| 평점순 | `driver.rating` 내림차순 |

### M-04/M-05: 상세와 탑승 요청

#### 정상 흐름
1. 상세 페이지가 `ride(id)`를 조회한다.
2. 차주 이름, 평점, 운행 횟수, 경로, 시간, 요금, 좌석을 표시한다.
3. 사용자가 탑승 요청 버튼을 누르면 메시지 입력 dialog를 연다.
4. 제출 시 `requestRide(input)` mutation을 호출한다.
5. 성공 시 요청 완료 상태를 표시하고 버튼을 비활성화한다.

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 상세 조회 실패 | 에러 상태 표시 |
| E-02 | 메시지 없이 요청 | 빈 메시지로 요청 가능 |
| E-03 | 요청 mutation 실패 | 에러 문구 표시 |

## 테스트 시나리오

| 테스트 ID | 대상 | 설명 |
|---|---|---|
| FE-UT-01 | `/matchings` | 검색 결과 카드 리스트를 표시한다 |
| FE-UT-02 | `/matchings` | 빈 결과 상태를 표시한다 |
| FE-UT-03 | `/matchings` | 요금순/평점순/출발시간순 정렬이 동작한다 |
| FE-UT-04 | `/matchings` | 카드 클릭 시 상세 화면으로 이동한다 |
| FE-UT-05 | `/matchings/:id` | 상세 화면에서 차주/경로/요금/좌석 정보를 표시한다 |
| FE-UT-06 | `/matchings/:id` | 메시지 입력 후 탑승 요청 mutation을 호출하고 완료 상태를 표시한다 |
| FE-UT-07 | `/matchings/:id` | 요청 실패 시 에러 상태를 표시한다 |

## 검증 명령

```bash
npm run codegen
npm test
npm run lint
npm run build
```

## 완료 조건

- `/matchings` 결과 화면이 검색 결과/정렬/빈/에러 상태를 제공한다.
- `/matchings/:id` 상세 화면이 차주 정보와 탑승 요청 dialog를 제공한다.
- `searchRides`, `ride`, `requestRide` GraphQL 연동이 generated 타입 기반으로 동작한다.
- `npm test`, `npm run lint`, `npm run build`가 통과한다.
