# FE — 홈·검색 통합 및 기록 탭 전환 구현 계획

## 목표

탑승자 홈(`/`)을 지도와 검색이 결합된 단일 탐색 화면으로 정의합니다. 기존 독립 검색 탭은 제거하고 하단 탭의 `검색` 메뉴를 `기록` 메뉴로 변경합니다. `/matchings`는 공유 링크와 기존 딥링크 호환을 위해 홈·검색 UI를 재사용하는 경로로 유지합니다.

## 관련 이슈

- Docs: `project-ridy/docs#67`
- Frontend: 구현 이슈 생성 필요

## 관련 docs 문서

- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/api/MATCHING.md`
- `docs/WORKFLOW.md`
- `docs/planning/implementation/frontend/2026-06-22_fe62-neighborhood-work-carpool.md`

## 아키텍처 접근

1. `app/page.tsx`의 홈 화면은 지도, 상단 검색 입력, 검색 sheet, 카드 rail을 한 화면에 둡니다.
2. `app/matchings/page.tsx`는 별도 검색 리스트를 새로 만들지 않고 홈·검색 화면의 재사용 가능한 단위를 렌더링합니다.
3. `components/ridy/BottomNavigation.tsx`는 `검색` 탭을 제거하고 `기록` 탭을 추가합니다.
4. `기록` 탭은 기존 정산/운행 이력 진입점으로 연결합니다. 전용 운행 기록 화면이 없으면 MVP에서는 `S10 정산 현황(/payments)`로 연결하고, 라벨만 `기록`으로 표시합니다.
5. API/GraphQL 변경은 없습니다. 기존 `nearbyCommuteOffers` operation과 generated 타입을 재사용합니다.

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-FE-HS-001 | A | 탑승자가 홈에 진입 | 상단 검색 입력, 지도, 주변 카풀 카드 rail 표시 |
| A-FE-HS-002 | A | 탑승자가 상단 검색 입력을 선택 | 출발 동네/근무지/시간/반경을 수정하는 검색 sheet 표시 |
| A-FE-HS-003 | A | 검색 조건을 적용 | 같은 화면에서 marker와 카드 rail 갱신 |
| A-FE-HS-004 | A | `/matchings` 딥링크로 진입 | 홈·검색 UI와 동일한 결과 화면 표시 |
| A-FE-HS-005 | A | 하단 탭을 확인 | `홈`, `기록`, `채팅`, `프로필` 순서로 표시 |
| E-FE-HS-001 | E | 검색 결과 없음 | 조건 완화/반경 확대 안내 표시 |
| E-FE-HS-002 | E | 검색 API 실패 | 오류 메시지와 재시도 버튼 표시 |
| X-FE-HS-001 | X | query string 검색 조건이 있는 `/matchings` 진입 | 상단 검색 입력과 결과가 query 조건으로 초기화 |
| X-FE-HS-002 | X | 기록 전용 화면이 아직 없음 | `기록` 탭은 `/payments`로 이동하고 라벨만 `기록` 유지 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-HS-001 | A-FE-HS-001, E-FE-HS-001, E-FE-HS-002 | `app/page.tsx` 또는 `components/ridy/HomeSearchSurface.tsx` — 홈 상단 검색 + 지도 + 카드 rail | `__tests__/Home.test.tsx` — `홈에서 검색 입력과 지도 카풀 결과를 함께 표시한다` | 홈 첫 화면에서 독립 로고 상단 바 없이 검색 입력/지도/카드 표시 |
| FE-HS-002 | A-FE-HS-002, A-FE-HS-003 | `components/ridy/HomeSearchSheet.tsx` 또는 동등 단위 — 검색 조건 수정/적용 | `__tests__/Home.test.tsx` — `검색 조건을 적용하면 같은 화면에서 결과 조건을 갱신한다` | 검색 sheet 접근 가능, 적용 후 `nearbyCommuteOffers` variables 갱신 |
| FE-HS-003 | A-FE-HS-004, X-FE-HS-001 | `app/matchings/page.tsx` — 홈·검색 UI 재사용 및 query 초기화 | `__tests__/Matchings.test.tsx` — `/matchings query 조건으로 홈 검색 UI를 초기화한다` | `/matchings`가 별도 UI가 아니라 같은 검색 surface를 사용 |
| FE-HS-004 | A-FE-HS-005, X-FE-HS-002 | `components/ridy/BottomNavigation.tsx` — `검색` 제거, `기록` 추가 | `__tests__/BottomNavigation.test.tsx` — `하단 탭은 홈 기록 채팅 프로필 순서로 표시한다` | 하단 탭에 검색 없음, 기록 탭은 `/payments`로 이동 |
| FE-HS-005 | A-FE-HS-001, A-FE-HS-005 | Visual smoke evidence | PR screenshots + browser QA 기록 | mobile 390px에서 상단 검색/지도/카드/기록 탭이 겹치지 않음 |

## 수정/생성할 파일 경로

- `frontend/app/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/components/ridy/BottomNavigation.tsx`
- `frontend/components/ridy/HomeSearchSurface.tsx` 또는 기존 홈 컴포넌트 내부 최소 수정
- `frontend/components/ridy/HomeSearchSheet.tsx` 또는 기존 필터 UI 내부 최소 수정
- `frontend/__tests__/Home.test.tsx`
- `frontend/__tests__/Matchings.test.tsx`
- `frontend/__tests__/BottomNavigation.test.tsx`

## TDD 순서

1. FE-HS-004: BottomNavigation 테스트 Red → `기록` 탭 전환 Green
2. FE-HS-001: 홈 통합 화면 테스트 Red → 상단 검색/지도/카드 rail Green
3. FE-HS-002: 검색 sheet 조건 적용 테스트 Red → variables 갱신 Green
4. FE-HS-003: `/matchings` query 딥링크 테스트 Red → 홈 검색 surface 재사용 Green
5. FE-HS-005: mobile browser QA + full verification

## 실행할 검증 명령

```bash
npm run test
npm run lint
npm run build
```

GraphQL operation 변경이 발생한 경우에만 다음을 추가로 실행합니다.

```bash
npm run codegen
```

## 완료 조건

- 홈(`/`)은 검색 입력, 지도, 주변 카풀 카드가 결합된 단일 탐색 화면입니다.
- `/matchings`는 홈·검색 UI를 재사용하며 query 조건을 초기값으로 반영합니다.
- 하단 탭은 `홈`, `기록`, `채팅`, `프로필` 순서이며 `검색` 탭이 없습니다.
- 모든 Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결됩니다.
- PR 본문에 Case ID 확인표와 실제 검증 결과를 포함합니다.

## 범위 제외

- 신규 GraphQL API 또는 DB schema 추가는 제외합니다.
- 전용 운행 기록 화면 신규 구현은 제외합니다. MVP에서는 `기록` 탭을 `/payments`에 연결합니다.
- 알림/프로필 상단 액션 재도입은 제외합니다. 해당 액션은 별도 화면 또는 후속 상단 보조 메뉴 이슈에서 다룹니다.
