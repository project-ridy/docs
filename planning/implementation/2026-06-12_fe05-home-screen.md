# 홈 화면 개발 기획서

## 개요

- **대상 이슈**: `project-ridy/frontend#5` — `[FEAT] 홈 화면 (탑승자/차주)`
- **대상 사용자**: 초대 코드 가입과 프로필 설정을 완료한 회사 구성원
- **목적**: 로그인/온보딩 이후 사용자가 출발지, 도착지, 출발 시간을 입력해 매칭 탐색으로 진입하고, 정기 카풀/내 카풀 상태를 확인할 수 있는 홈 진입점을 제공한다.
- **현재 판단**: `project-ridy/backend#6` 매칭 API 구현이 아직 완료되지 않았으므로, 이번 1차 구현은 프론트 홈 화면 구조와 라우팅, 테스트를 우선 제공한다. GraphQL `myRides`, `searchRides` 실 연동과 차주 홈 데이터화는 backend#6 이후 진행한다.

## 기능 분해

| 번호 | 하위 기능 | 설명 | 우선순위 |
|---|---|---|---|
| F-01 | 인증 보호 | `/` 홈 진입 시 토큰이 없으면 `/login`으로 이동 | P0 |
| F-02 | 탑승자 경로 검색 | 출발지, 도착지, 출발 시간 입력과 매칭 찾기 CTA 제공 | P0 |
| F-03 | 매칭 결과 라우팅 | 검색 입력값을 `/matchings` 쿼리 파라미터로 전달 | P0 |
| F-04 | 내 정기 카풀 섹션 | 정기 카풀 카드 리스트와 빈 상태를 표시 | P1 |
| F-05 | 하단 내비게이션 | 홈, 검색, 채팅, 내 정보 탭 이동 | P1 |
| F-06 | 차주 홈 | 등록 카풀과 탑승 요청 상태 표시 | P2 |
| F-07 | GraphQL 연동 | `myRides`, `searchRides` 기반 서버 상태 연결 | P2 |

## 코드 구조

### 백엔드

- 이번 1차 프론트 작업에서 백엔드 코드는 수정하지 않는다.
- 선행/후속 의존성:
  - `project-ridy/backend#6` — 매칭 알고리즘 & API
  - `backend/src/graphql/schema.graphql`
    - `searchRides(input: SearchRidesInput!, pagination: PaginationInput): RideConnection`
    - `myRides(status: RideStatus, pagination: PaginationInput): RideConnection`
- 현재 확인 결과 `backend/src/services/matching/matching-service.module.ts`는 빈 모듈 상태이므로, 프론트 실 API 연동 완료 조건은 backend#6 이후로 둔다.

### 프론트엔드

- 페이지: `app/page.tsx`
  - `AuthGuard`로 인증 보호
  - `RouteInput`, `MatchingCard`, `BottomNavigation` 조합
  - 출발 시간 `input[type="time"]`
  - `useRouter`로 `/matchings` 라우팅
- 테스트: `__tests__/Home.test.tsx`
  - 인증 리다이렉트
  - 홈 화면 렌더링
  - 검색 파라미터 라우팅
  - 하단 내비게이션 이동
- 후속 GraphQL 연동 시 추가 예정:
  - `src/graphql/operations/matching.graphql`
  - `lib/api/matching-api.ts`
  - `hooks/useMatchingQueries.ts`

## 상세 설계

### F-01: 인증 보호

#### 정상 흐름
1. 사용자가 `/`에 접근한다.
2. `AuthGuard`가 `ridy.accessToken` 존재 여부를 확인한다.
3. 토큰이 있으면 홈 화면을 렌더링한다.

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | access token 없음 | `/login`으로 이동 |
| E-02 | 브라우저 스토리지 접근 불가 | 토큰 없음과 동일하게 처리 |

### F-02: 탑승자 경로 검색

#### 정상 흐름
1. 사용자가 출발지와 도착지를 입력한다.
2. 출발 시간을 선택한다.
3. `매칭 찾기` 버튼을 누른다.

#### 함수 시그니처

```typescript
function handleSearch(): void;
```

#### 예외 처리

| 케이스 | 조건 | 처리 |
|---|---|---|
| E-01 | 출발지/도착지 미입력 | 현재는 빈 쿼리로 `/matchings` 진입, 검색 화면에서 상세 검증 예정 |
| E-02 | 시간 미입력 | 시간 파라미터 생략 |

### F-03: 매칭 결과 라우팅

#### 정상 흐름
1. 입력된 검색 조건을 `URLSearchParams`로 변환한다.
2. `/matchings?departure=...&destination=...&departureTime=...`로 이동한다.

#### 함수 시그니처

```typescript
function handleTabChange(tabId: string): void;
```

### F-04: 내 정기 카풀 섹션

#### 정상 흐름
1. 정기 카풀 섹션 제목을 표시한다.
2. 카드 리스트가 있으면 `MatchingCard`로 표시한다.
3. 빈 상태 문구로 첫 카풀 탐색을 유도한다.

#### 후속 데이터화

```typescript
function useMyRidesQuery(status?: RideStatus) {
  return useQuery({ queryKey: ['myRides', status], queryFn: ... });
}
```

### F-06/F-07: 차주 홈과 GraphQL 연동

#### BLOCKED 사유

- `backend#6` 매칭 API 구현이 완료되어야 `myRides`, `searchRides`, 요청 대기 상태를 실제 데이터로 연결할 수 있다.
- 현재 프론트에서는 임의 API 타입을 만들지 않고 generated GraphQL 타입을 사용해야 하므로, schema/codegen/백엔드 resolver 구현 완료 후 진행한다.

## 테스트 시나리오

### 컴포넌트/페이지 테스트

| 테스트 | 대상 | 설명 |
|---|---|---|
| FE-UT-01 | `Home` | 토큰이 없으면 `/login` 이동 |
| FE-UT-02 | `Home` | 인증 상태에서 홈 헤더, 경로 검색, 정기 카풀, 빈 상태 표시 |
| FE-UT-03 | `Home.handleSearch` | 출발지/도착지/시간 입력 후 `/matchings` 쿼리로 이동 |
| FE-UT-04 | `BottomNavigation` | 검색 탭 클릭 시 `/matchings` 이동 |

### 통합 테스트

| 테스트 | 대상 | 설명 |
|---|---|---|
| FE-IT-01 | 홈 → 매칭 | 검색 조건 입력 후 매칭 결과 화면으로 라우팅 |
| FE-IT-02 | 홈 인증 보호 | 온보딩 완료 전 홈 직접 접근 시 로그인으로 리다이렉트 |

### 후속 테스트

| 테스트 | 선행 조건 | 설명 |
|---|---|---|
| FE-IT-03 | backend#6 완료 | `myRides` MSW 모킹으로 정기 카풀 로딩/빈/데이터 상태 검증 |
| FE-IT-04 | backend#6 완료 | 차주 role일 때 등록 카풀과 탑승 요청 배지 표시 |

## 의존성

- 선행 완료: `project-ridy/frontend#4` 로그인/온보딩 화면
- 선행 필요: `project-ridy/frontend#2` 디자인 시스템 컴포넌트
- 후속/차단: `project-ridy/backend#6` 매칭 알고리즘 & API
- 관련 문서:
  - `docs/design/SCREENS.md`
  - `docs/design/WIREFRAMES.md`
  - `docs/design/DESIGN_SYSTEM.md`
  - `docs/api/MATCHING.md`

## 완료 조건

### 1차 완료 조건

- `/` 접근 시 인증 가드 적용
- 탑승자 홈에서 출발지/도착지/출발 시간 입력 가능
- `매칭 찾기` 클릭 시 `/matchings`로 검색 조건 전달
- 내 정기 카풀 섹션과 빈 상태 표시
- 하단 내비게이션 4개 탭 표시 및 검색 탭 이동
- `npm run test`, `npm run lint`, `npm run build` 통과

### 후속 완료 조건

- `myRides` GraphQL query 연동
- 역할 기반 탑승자/차주 홈 분기
- 차주 홈 등록 카풀/탑승 요청 배지
- 로딩/빈/에러 상태 스켈레톤 또는 상태 UI
