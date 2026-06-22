# FE — 동네 기반 직장 카풀 지도 플로우 구현 계획

## 목표

탑승자 홈/매칭 화면을 **동네 주변 회사행 카풀 지도 탐색** 중심으로 전환하고, 차주 홈에는 **지도 기반 출발 위치 등록** 흐름을 추가합니다. API/DB 스펙은 `docs/api/MATCHING.md`, 화면 정의는 `docs/design/SCREENS.md`를 따릅니다.

## 관련 이슈

- Docs: `project-ridy/docs#62`
- Frontend: 구현 이슈 생성 필요

## 관련 docs 문서

- `docs/planning/PLANNING.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/api/MATCHING.md`
- `docs/WORKFLOW.md`

## 아키텍처 접근

1. GraphQL operation을 먼저 작성하고 codegen 타입을 사용합니다.
2. 지도 구현은 Kakao Maps JavaScript API를 사용합니다. 클라이언트 키는 `NEXT_PUBLIC_KAKAO_MAP_APP_KEY`로 주입하고, 키가 없거나 SDK 로드가 실패하면 테스트 가능한 목록 fallback을 표시합니다.
3. Home은 `nearbyCommuteOffers` 결과를 Kakao 지도 marker와 bottom sheet/list로 표현합니다.
4. Driver 화면은 출발 위치 좌표/표시명/근무지/시간/좌석을 입력하는 등록 폼을 제공합니다.
5. 승인 전 정확 주소를 보여주지 않고 `pickupLabel`만 카드/상세에 표시합니다.

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-FE-NW-001 | A | 탑승자가 내 동네 기준 주변 카풀을 조회 | 지도 panel과 제공자 카드 표시 |
| A-FE-NW-002 | A | 탑승자가 marker/card를 선택 | matching detail 또는 bottom sheet 표시 |
| A-FE-NW-003 | A | 차주가 출발 위치와 근무지를 등록 | createRide mutation 호출 |
| E-FE-NW-001 | E | 주변 카풀 없음 | 반경 확대/차주 등록 유도 empty state 표시 |
| E-FE-NW-002 | E | 위치 권한 거부 | 동네 직접 입력 fallback 표시 |
| X-FE-NW-001 | X | 같은 위치 여러 offer | marker cluster/list count 표시 |
| X-FE-NW-002 | X | 승인 전 상세 화면 | 정확 주소 대신 표시용 동네명만 표시 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-NW-001 | A-FE-NW-001, E-FE-NW-001, E-FE-NW-002 | `src/graphql/operations/matching.graphql`, generated types | `__tests__/graphqlOperations.test.ts` — `nearbyCommuteOffers operation이 정의된다` | codegen 통과, 손작성 타입 없음 |
| FE-NW-002 | A-FE-NW-001, X-FE-NW-001 | `app/page.tsx`, Kakao map/list components — 주변 회사행 카풀 지도 panel | `__tests__/Home.test.tsx` — `홈에서 동네 주변 회사행 카풀을 지도와 카드로 표시한다` | Kakao marker/list/empty/fallback state 렌더링 |
| FE-NW-003 | A-FE-NW-002, X-FE-NW-002 | `app/matchings/*` — pickupLabel 중심 카드/상세 | `__tests__/Matchings.test.tsx` — `승인 전 정확 주소 대신 표시용 동네명을 보여준다` | 정확 주소 미노출 |
| FE-NW-004 | A-FE-NW-003 | `app/driver/page.tsx` 또는 driver form component — 출발 위치 등록 폼 | `__tests__/DriverHome.test.tsx` — `차주가 지도 출발 위치와 회사 근무지로 카풀을 등록한다` | createRide variables가 docs와 일치 |
| FE-NW-005 | X-FE-NW-001, X-FE-NW-002 | Visual smoke evidence | PR screenshots + `npm run test/lint/build` | mobile 지도/카드/등록 화면 확인 |

## 수정/생성할 파일 경로

- `frontend/src/graphql/operations/matching.graphql`
- `frontend/app/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/app/matchings/[id]/page.tsx`
- `frontend/app/driver/page.tsx` 또는 관련 component
- `frontend/lib/kakao-map.ts` 또는 관련 Kakao Maps SDK loader
- `frontend/__tests__/Home.test.tsx`
- `frontend/__tests__/Matchings.test.tsx`
- `frontend/__tests__/DriverHome.test.tsx`

## TDD 순서

1. FE-NW-001: GraphQL operation test/codegen Red → operation 추가 Green
2. FE-NW-002: Home Kakao 지도 panel test Red → Kakao marker/list/fallback Green
3. FE-NW-003: matching privacy display test Red → pickupLabel 중심 표시 Green
4. FE-NW-004: driver registration form test Red → createRide form Green
5. FE-NW-005: visual smoke + full verification

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- Kakao Maps API로 동네 기반 탐색 흐름이 동작하고, 키 미설정/SDK 로드 실패 시 목록 fallback이 동작합니다.
- 승인 전 정확 주소를 표시하지 않습니다.
- 모든 Case ID가 구현/테스트에 연결됩니다.
- PR 본문에 Case ID 확인표와 visual evidence를 포함합니다.
