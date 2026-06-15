# FE13 친환경 임팩트 대시보드 구현 계획

## 목표

- `frontend#13` 친환경 임팩트 대시보드를 구현한다.
- 사용자의 개인 카풀 참여로 절감한 CO₂ 총량, 트리 환산, 월별 추이, 운행별 절감 내역, 뱃지/레벨을 표시한다.
- GraphQL schema-first 흐름을 따라 `myCarbonImpact`, `carbonHistory` operation과 generated 타입을 사용한다.

## 관련 이슈

- Frontend: `project-ridy/frontend#13` `[FEAT] 친환경 임팩트 대시보드`
- Docs: `project-ridy/docs#38` `[DOCS] 친환경 임팩트 대시보드 API/기획서 정리`
- 선행: `project-ridy/frontend#2` 디자인 시스템
- 선행 backend follow-up: `myCarbonImpact`, `carbonHistory` schema/resolver 구현 PR

## 관련 문서

- `docs/api/GRAPHQL_GATEWAY.md` Payment / Analytics SDL
- `docs/architecture/ARCHITECTURE.md` 개인 친환경 임팩트 조회
- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `frontend/AGENTS.md`

## 아키텍처 접근

- route는 사원 앱의 인증 영역에 `frontend/app/eco-impact/page.tsx`로 추가한다.
- GraphQL operation은 `frontend/graphql/eco-impact.graphql`에 작성하고 `npm run codegen`으로 generated 타입을 생성한다.
- API 호출은 기존 GraphQL client/TanStack Query 패턴에 맞춰 `features/eco-impact/api` 또는 기존 feature 구조에 둔다.
- 차트는 MVP에서 외부 차트 라이브러리를 새로 추가하지 않고 CSS 기반 막대 차트로 구현한다. 라이브러리 추가가 필요하면 별도 의존성 검토 이슈로 분리한다.
- SNS 공유는 MVP에서 Web Share API를 우선 사용한다. Web Share API 미지원 브라우저는 공유 문구를 clipboard에 복사한다.
- 절감 성과 이미지 생성은 이번 범위에서 DOM-to-image 의존성을 추가하지 않는다. `navigator.share` payload는 텍스트/URL 기반으로 제공한다.
- 서버가 반환하는 `level`, `badges`, `treeEquivalent`를 표시한다. 프론트에서 같은 기준을 중복 계산하지 않는다.

## GraphQL operation

```graphql
query EcoImpactDashboard($period: String!, $pagination: PaginationInput) {
  myCarbonImpact(period: $period) {
    period
    totalRides
    totalDistanceKm
    co2SavedKg
    treeEquivalent
    level
    badges {
      id
      label
      description
      achievedAt
    }
  }
  carbonHistory(period: $period, pagination: $pagination) {
    monthly {
      period
      totalRides
      totalDistanceKm
      co2SavedKg
    }
    rides {
      rideId
      completedAt
      departureAddr
      arrivalAddr
      distanceKm
      co2SavedKg
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

## UX 범위

- 헤더
  - 화면 제목 `친환경 임팩트`
  - 기간 필터: 기본 최근 12개월 (`P12M` 또는 backend 합의 period string)
- 요약 카드
  - 총 CO₂ 절감량 kg
  - 트리 환산 수치
  - 완료 카풀 수
  - 총 공유 거리 km
- 월별 절감량 차트
  - `carbonHistory.monthly` 기반 막대 차트
  - 데이터가 없을 때 empty state 표시
- 운행별 절감 내역
  - 완료 일시, 출발/도착 주소, 거리, 절감 CO₂
  - `pageInfo.hasNextPage`가 true이면 더 보기 버튼 표시
- 뱃지/레벨
  - 서버에서 받은 `level`, `badges` 표시
  - 아직 달성한 뱃지가 없으면 안내 empty state 표시
- SNS 공유
  - Web Share API 지원 시 공유 sheet 호출
  - 미지원 시 clipboard 복사와 toast/inline feedback 제공

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 사용자가 `/eco-impact` 진입 | 총 CO₂, 트리 환산, 완료 카풀 수, 공유 거리 요약 카드가 표시된다. |
| A2 | 월별 데이터가 존재 | 월별 절감량 막대 차트가 `period`, `co2SavedKg` 기준으로 표시된다. |
| A3 | 운행별 데이터가 존재 | 각 운행의 완료일, 경로, 거리, CO₂ 절감량이 목록에 표시된다. |
| A4 | 뱃지가 존재 | 레벨과 달성 뱃지 카드가 표시된다. |
| A5 | 공유 버튼 클릭 | Web Share API 또는 clipboard fallback이 실행되고 성공 feedback이 표시된다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | GraphQL 요청이 실패 | 에러 메시지와 재시도 버튼이 표시된다. |
| E2 | 인증 토큰이 없음 | 기존 인증 가드/GraphQL client 정책에 따라 로그인 흐름으로 이동하거나 인증 에러 UI가 표시된다. |
| E3 | Web Share API와 clipboard 모두 실패 | 공유 실패 안내가 표시되고 앱이 crash되지 않는다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 임팩트 데이터가 0 | 빈 차트 대신 첫 카풀을 유도하는 empty state가 표시된다. |
| X2 | `departureAddr`/`arrivalAddr`가 null | `주소 정보 없음` fallback을 표시한다. |
| X3 | 매우 큰 CO₂ 수치 | 숫자 포맷팅으로 overflow 없이 표시된다. |
| X4 | 모바일 375px 화면 | 요약 카드와 차트가 한 컬럼으로 표시된다. |
| X5 | `pageInfo.hasNextPage`가 false | 더 보기 버튼을 표시하지 않는다. |

## 구현/테스트 케이스 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트 이름 | 완료 기준 |
|---|---|---|---|---|
| FE13-C01 | A1, X3, X4 | `app/eco-impact/page.tsx`, `features/eco-impact/components/ImpactSummaryCards.tsx` | `__tests__/eco-impact/EcoImpactPage.test.tsx` — `renders carbon summary cards` | 요약 카드 4개가 generated mock 데이터로 렌더링된다. |
| FE13-C02 | A2, X1, X4 | `features/eco-impact/components/CarbonMonthlyChart.tsx` | `__tests__/eco-impact/CarbonMonthlyChart.test.tsx` — `renders monthly carbon bars and empty state` | 월별 막대와 empty state를 모두 검증한다. |
| FE13-C03 | A3, X2, X5 | `features/eco-impact/components/CarbonRideList.tsx` | `__tests__/eco-impact/CarbonRideList.test.tsx` — `renders ride savings with address fallback and pagination state` | 운행 목록, null 주소 fallback, 더 보기 버튼 조건을 검증한다. |
| FE13-C04 | A4, X1 | `features/eco-impact/components/CarbonBadges.tsx` | `__tests__/eco-impact/CarbonBadges.test.tsx` — `renders level badges and empty badge state` | 레벨/뱃지와 뱃지 없음 상태를 검증한다. |
| FE13-C05 | A5, E3 | `features/eco-impact/components/ShareImpactButton.tsx` | `__tests__/eco-impact/ShareImpactButton.test.tsx` — `uses web share or clipboard fallback` | share/clipboard 성공 및 실패 feedback을 검증한다. |
| FE13-C06 | E1, E2 | `features/eco-impact/api/use-eco-impact-dashboard.ts`, page error state | `__tests__/eco-impact/EcoImpactPage.test.tsx` — `renders retryable api error state` | GraphQL/MSW 에러 시 재시도 가능한 에러 UI를 검증한다. |

## 수정/생성할 파일 경로

- `frontend/graphql/eco-impact.graphql`
- `frontend/app/eco-impact/page.tsx`
- `frontend/features/eco-impact/api/use-eco-impact-dashboard.ts`
- `frontend/features/eco-impact/components/ImpactSummaryCards.tsx`
- `frontend/features/eco-impact/components/CarbonMonthlyChart.tsx`
- `frontend/features/eco-impact/components/CarbonRideList.tsx`
- `frontend/features/eco-impact/components/CarbonBadges.tsx`
- `frontend/features/eco-impact/components/ShareImpactButton.tsx`
- `frontend/__tests__/eco-impact/EcoImpactPage.test.tsx`
- `frontend/__tests__/eco-impact/CarbonMonthlyChart.test.tsx`
- `frontend/__tests__/eco-impact/CarbonRideList.test.tsx`
- `frontend/__tests__/eco-impact/CarbonBadges.test.tsx`
- `frontend/__tests__/eco-impact/ShareImpactButton.test.tsx`
- 필요 시 `frontend/mocks/handlers/eco-impact.ts`

## TDD 순서

1. Red: `graphql/eco-impact.graphql` 작성 후 `npm run codegen`을 실행한다. backend schema에 query가 없으면 codegen 실패를 확인하고 backend follow-up으로 BLOCKED 처리한다.
2. Red: `EcoImpactPage.test.tsx`에 FE13-C01/E1 테스트를 작성하고 실패를 확인한다.
3. Green: page, hook, MSW mock, summary card 최소 구현.
4. Red: FE13-C02 차트 테스트 작성.
5. Green: CSS 기반 chart 구현과 empty state 구현.
6. Red: FE13-C03 운행 목록 테스트 작성.
7. Green: list, fallback, pagination state 구현.
8. Red: FE13-C04 뱃지 테스트 작성.
9. Green: level/badge UI 구현.
10. Red: FE13-C05 공유 버튼 테스트 작성.
11. Green: Web Share API + clipboard fallback 구현.
12. Refactor: feature barrel/export 정리, 중복 formatter 분리.
13. Verify: 전체 검증 명령 실행.

## 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- `/eco-impact` 인증 화면이 렌더링된다.
- `myCarbonImpact`, `carbonHistory` generated 타입을 사용한다.
- 총 CO₂ 절감량, 트리 환산, 월별 차트, 운행별 내역, 레벨/뱃지, 공유 버튼이 표시된다.
- FE13-C01~FE13-C06의 구현/테스트가 모두 존재한다.
- PR 본문에 Case ID 확인 표를 포함한다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build` 결과를 PR에 기록한다.

## 범위 제외

- backend resolver/service 구현
- DB migration 추가
- 이미지 파일 기반 SNS 공유 카드 생성
- 외부 차트 라이브러리 추가
- 관리자 ESG 리포트 화면

## Backend follow-up 필요 사항

Frontend 구현 전 또는 병행으로 backend 이슈/PR에서 다음을 완료해야 한다.

- `backend/src/graphql/schema.graphql`에 `CarbonImpact`, `CarbonHistory`, `myCarbonImpact`, `carbonHistory` 추가
- `npm run codegen`으로 backend generated 타입 갱신
- `Ride`, `RideRequest`, `User` 기반 개인 CO₂ 계산 service/resolver 구현
- `myCarbonImpact`, `carbonHistory` GraphQL E2E 테스트 작성
- docs의 `docs/api/GRAPHQL_GATEWAY.md`, `docs/architecture/ARCHITECTURE.md`와 구현 일치 확인
