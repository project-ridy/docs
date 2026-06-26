# FE — 집 주변 카풀 선택 홈 구현 계획

## 목표

탑승자 홈(`/`)을 검색 화면이 아니라 **집/동네 주변 회사행 카풀을 즉시 보여주고 선택하는 화면**으로 구현합니다. 사용자는 지도 marker 또는 카드 rail에서 카풀을 선택하고, 선택한 카풀의 회사행 상세/요청 화면으로 이동합니다. 홈 하단바는 제거하고, 상단 왼쪽 메뉴 버튼과 상단 오른쪽 프로필 버튼을 제공합니다. 메뉴에는 이전 탑승 기록 진입점을 포함합니다.

## 관련 이슈

- Docs: `project-ridy/docs#69`
- Frontend: `project-ridy/frontend#54` 갱신 필요

## 관련 docs 문서

- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/api/MATCHING.md`
- `docs/WORKFLOW.md`
- `docs/planning/implementation/frontend/2026-06-22_fe62-neighborhood-work-carpool.md`

## 아키텍처 접근

1. `app/page.tsx`의 홈 화면은 위치 안내, 지도, 주변 카풀 카드 rail만 포함합니다. 검색 입력과 검색 sheet는 구현하지 않습니다.
2. `nearbyCommuteOffers` operation을 재사용합니다. 검색용 신규 GraphQL operation, 직접 작성 API 타입, DB 변경은 없습니다.
3. 위치 기준은 현재 위치 opt-in 또는 사용자 동네 fallback입니다. 승인 전 정확한 집 주소는 표시하지 않고 `pickupLabel`만 표시합니다.
4. 현재 위치 또는 fallback 중심점 기준 **반경 1km/2km/5km** 선택 메뉴를 지도 상단에 표시하고, 선택 반경을 지도 테두리와 주변 카풀 조회 `radiusKm`에 반영합니다. 기본값은 5km입니다.
5. `app/matchings/page.tsx`는 기존 딥링크 호환용으로 유지하되, 독립 검색 화면이 아니라 홈과 같은 주변 카풀 UI를 재사용합니다.
6. 홈 화면은 하단 탭을 사용하지 않습니다. 상단 왼쪽 메뉴 버튼에는 이전 탑승 기록(`/payments`) 진입점을 제공하고, 상단 오른쪽 프로필 버튼은 `/profile`로 이동합니다.

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-FE-NH-001 | A | 탑승자가 홈에 진입 | 위치 안내, 지도, 주변 회사행 카풀 카드 rail 표시 |
| A-FE-NH-002 | A | 탑승자가 marker 또는 카드를 선택 | 선택한 카풀의 S07 상세/요청 화면으로 이동 |
| A-FE-NH-003 | A | 홈 상단 액션을 확인 | 왼쪽 메뉴 버튼, 오른쪽 프로필 버튼 표시 |
| A-FE-NH-004 | A | `/matchings` 딥링크로 진입 | 홈과 같은 주변 카풀 UI 표시 |
| E-FE-NH-001 | E | 주변 카풀 없음 | 반경 확대 또는 차주 등록 유도 empty state 표시 |
| E-FE-NH-002 | E | 위치 권한 거부 | 동네 fallback 기준 주변 카풀 표시 |
| E-FE-NH-003 | E | 주변 카풀 API 실패 | 오류 메시지와 재시도 버튼 표시 |
| X-FE-NH-001 | X | 승인 전 위치 표시 | 정확한 집 주소 대신 `pickupLabel`과 대략 위치만 표시 |
| X-FE-NH-002 | X | `/matchings`에 기존 query string이 포함됨 | 검색 UI를 표시하지 않고 주변 카풀 조회로 정규화 |
| X-FE-NH-003 | X | 기록 전용 화면이 아직 없음 | 메뉴의 이전 탑승 기록은 `/payments`로 이동 |
| X-FE-NH-004 | X | 위치 권한 거부 또는 Kakao SDK 실패 | 동네 fallback 중심점의 5km 반경 안내와 목록 fallback 유지 |
| X-FE-NH-005 | X | 사용자가 반경을 바꿈 | 1km/2km/5km 중 선택한 값으로 지도 테두리와 API 조회 반경 동기화 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-NH-001 | A-FE-NH-001, E-FE-NH-001, E-FE-NH-002, E-FE-NH-003 | `app/page.tsx` 또는 `components/ridy/NearbyHomeSurface.tsx` — 위치 안내 + 지도 + 카드 rail | `__tests__/Home.test.tsx` — `홈에서 집 주변 회사행 카풀을 지도와 카드로 표시한다` | 검색 입력 없이 위치 안내/지도/카드/empty/error/fallback 표시 |
| FE-NH-002 | A-FE-NH-002, X-FE-NH-001 | `components/ridy/MatchingCard.tsx`, `app/page.tsx` — marker/card 선택과 상세 이동 | `__tests__/Home.test.tsx` — `주변 카풀 카드를 선택하면 상세 화면으로 이동한다` | 카드/marker 선택 시 `/matchings/:id` 이동, 정확 주소 미노출 |
| FE-NH-003 | A-FE-NH-003, X-FE-NH-003 | `app/page.tsx` — 홈 상단 메뉴/프로필 액션, 이전 탑승 기록 메뉴 항목 | `__tests__/Home.test.tsx` — `FE-NH-003: 홈은 하단바 없이 상단 메뉴와 프로필 버튼을 표시한다` | 하단바 없음, 메뉴에서 이전 탑승 기록은 `/payments`, 프로필 버튼은 `/profile` 이동 |
| FE-NH-004 | A-FE-NH-004, X-FE-NH-002 | `app/matchings/page.tsx` — 주변 카풀 UI 재사용 | `__tests__/Matchings.test.tsx` — `/matchings는 검색 UI 없이 주변 카풀을 표시한다` | `/matchings`에서 검색 입력/sheet 없이 홈과 같은 주변 카풀 UI 표시 |
| FE-NH-005 | A-FE-NH-001, A-FE-NH-003 | Visual smoke evidence | PR screenshots + browser QA 기록 | mobile 390px에서 위치 안내/지도/카드/기록 탭이 겹치지 않음 |
| FE-NH-006 | A-FE-NH-001, A-FE-NH-004, X-FE-NH-004, X-FE-NH-005 | `hooks/useMatchingQueries.ts`, `components/ridy/NeighborhoodCommuteMap.tsx` — 1km/2km/5km 반경 선택과 지도 테두리/API 조회 동기화 | `__tests__/Home.test.tsx` — `FE-NH-006: 현재 위치 반경 선택과 제한 조회를 표시한다` | 기본 5km, 1km/2km/5km 선택 시 지도 안내와 API `radiusKm` 동기화 |

## 수정/생성할 파일 경로

- `frontend/app/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/components/ridy/BottomNavigation.tsx`
- `frontend/components/ridy/NearbyHomeSurface.tsx` 또는 기존 홈 컴포넌트 내부 최소 수정
- `frontend/components/ridy/NeighborhoodCommuteMap.tsx`
- `frontend/hooks/useMatchingQueries.ts`
- `frontend/__tests__/Home.test.tsx`
- `frontend/__tests__/Matchings.test.tsx`
- `frontend/__tests__/BottomNavigation.test.tsx`

## TDD 순서

1. FE-NH-003: 홈 상단 메뉴/프로필 테스트 Red → 하단바 제거와 상단 액션 Green
2. FE-NH-001: 홈 주변 카풀 화면 테스트 Red → 위치 안내/지도/카드 rail Green
3. FE-NH-002: 카드/marker 선택 테스트 Red → 상세 이동 Green
4. FE-NH-004: `/matchings` 딥링크 테스트 Red → 검색 UI 없는 주변 카풀 UI 재사용 Green
5. FE-NH-006: 1km/2km/5km 반경 선택과 `radiusKm` 동기화 테스트 Red → 지도/쿼리 Green
6. FE-NH-005: mobile browser QA + full verification

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

- 홈(`/`)은 검색 입력 없이 위치 안내, 지도, 주변 회사행 카풀 카드가 결합된 선택 화면입니다.
- 사용자는 marker 또는 카드 선택 후 회사까지 가는 S07 상세/요청 화면으로 이동합니다.
- `/matchings`는 검색 UI를 만들지 않고 주변 카풀 UI를 재사용합니다.
- 지도에는 현재 위치 또는 fallback 중심 기준 선택 반경(1km/2km/5km) 테두리를 표시하고, 주변 카풀 조회는 해당 반경으로 제한합니다.
- 홈에는 하단바를 표시하지 않고, 상단 왼쪽 메뉴와 상단 오른쪽 프로필 버튼을 표시합니다.
- 모든 Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결됩니다.
- PR 본문에 Case ID 확인표와 실제 검증 결과를 포함합니다.

## 범위 제외

- 검색 입력, 검색 sheet, 필터 UI 신규 구현은 제외합니다.
- 신규 GraphQL API 또는 DB schema 추가는 제외합니다.
- 전용 운행 기록 화면 신규 구현은 제외합니다. MVP에서는 `기록` 탭을 `/payments`에 연결합니다.
- 알림/프로필 상단 액션 재도입은 제외합니다. 해당 액션은 별도 화면 또는 후속 상단 보조 메뉴 이슈에서 다룹니다.
