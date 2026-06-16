# Frontend 전체 Socar Frame 기반 UX 고도화 구현 계획

## 목표

- 기존 `FE-SOCAR-DESIGN` 작업으로 정렬된 token/component 기반을 모든 주요 frontend 화면에 확장한다.
- `docs/design/DESIGN_SYSTEM.md`, `SCREENS.md`, `WIREFRAMES.md`에 정의된 화면 구조와 상태 규칙을 유지하면서 Socar Frame의 `Respect the Legacy`, `Predictability`, `Token first` 원칙을 화면 단위로 적용한다.
- API, GraphQL, DB 동작은 변경하지 않고 시각 hierarchy, responsive layout, loading/empty/error/unauthorized/offline 상태, CTA 우선순위, visual QA 범위를 정리한다.

## 아키텍처 접근

- Socar Frame package를 직접 설치하지 않는다.
  - `@socar-inc/socar-frame-foundation` 직접 adoption은 Tailwind v4 호환성 spike 전까지 `BLOCKED`이다.
  - 현재 계획은 `frontend/app/globals.css` token과 기존 shadcn/ui + Ridy custom component 조합만 사용한다.
- 이전 계획 `2026-06-16_fe54-socar-frame-design-refresh.md`의 token/component 변경을 전제로 화면별 조립 품질을 개선한다.
- 화면 구조는 `WIREFRAMES.md`의 정보 순서와 `SCREENS.md`의 상태 정의를 우선한다.
- 공통 page shell, section/card pattern, TopAppBar, empty/error/loading state를 재사용 가능한 작은 단위로 정리하되, 문서에 없는 정보 구조나 API behavior는 추가하지 않는다.
- Authenticated visual QA는 `RIDY_VISUAL_QA_FIXTURE=1` GraphQL fixture를 사용한다.

## 관련 이슈

- docs 계획 이슈: 생성 필요 — `[DOCS] 전체 프론트 Socar UX 고도화 구현 계획`
- frontend 구현 이슈: 생성 필요 — `[FEAT] 전체 프론트 Socar UX 고도화`
- 선행 완료:
  - `project-ridy/docs#54` — Socar Frame 기반 디자인 시스템 기획
  - `project-ridy/frontend#41` / PR `#42` — 디자인 시스템 개선 구현
  - `project-ridy/frontend#43` — 인증 화면 visual QA
  - `project-ridy/frontend#44` / PR `#45` — visual QA GraphQL fixture

## 관련 docs 문서

- `docs/WORKFLOW.md`
- `docs/AGENTS.md`
- `agents/protocol/AGENT_PROTOCOL.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/planning/implementation/frontend/2026-06-16_fe54-socar-frame-design-refresh.md`
- `frontend/AGENTS.md`

## 관련 Socar Frame 참고 문서

- `https://socarframe.socar.kr/development/principle`
- `https://socarframe.socar.kr/development/foundation`
- `https://socarframe.socar.kr/development/foundation/Colors`
- `https://socarframe.socar.kr/development/foundation/Spacing`
- `https://socarframe.socar.kr/development/foundation/Typography`
- `https://socarframe.socar.kr/development/components`
- `https://socarframe.socar.kr/development/components/Buttons`
- `https://socarframe.socar.kr/development/components/Input`

## A/E/X 케이스 분석

### A — 정상 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| A-FE-SOCAR-UX-001 | App/Page shell 정렬 | Mobile/Tablet/Desktop 화면 진입 | 페이지 padding, max-width, safe-area, bottom nav 여백이 `DESIGN_SYSTEM.md` spacing 규칙과 일치한다. |
| A-FE-SOCAR-UX-002 | Public/Auth 화면 고도화 | `/landing`, `/login`, `/profile/setup` | hero/auth card/form hierarchy가 token 기반으로 정리되고 primary CTA가 화면당 하나로 유지된다. |
| A-FE-SOCAR-UX-003 | 홈/차주 화면 고도화 | `/`, `/driver` | 검색/운행 등록/정기 카풀/요청 관리 섹션이 card stack과 TopAppBar 패턴으로 예측 가능하게 표시된다. |
| A-FE-SOCAR-UX-004 | 매칭 검색/상세 화면 고도화 | `/matchings`, `/matchings/:id` | 검색 요약, 필터, MatchingCard, 상세 CTA가 정보 순서와 responsive layout을 유지한다. |
| A-FE-SOCAR-UX-005 | 채팅 화면 고도화 | `/chat`, `/chat/:id` | 채팅 목록/방의 search, unread badge, message bubble, input fixed area가 상태별로 일관된다. |
| A-FE-SOCAR-UX-006 | 정산/프로필/리뷰 화면 고도화 | `/payments`, `/profile`, `/reviews/:matchingId` | 요약 카드, 상태 badge, 위험 동작, rating/review CTA가 문서 규칙을 따른다. |
| A-FE-SOCAR-UX-007 | 공통 상태 UI 적용 | loading/empty/error/unauthorized/offline | skeleton/empty CTA/error retry/offline banner가 색상만이 아닌 텍스트와 action으로 상태를 전달한다. |
| A-FE-SOCAR-UX-008 | Visual QA | 전체 주요 화면 | public/authenticated 화면 스크린샷 또는 browser verification 결과가 PR에 첨부된다. |

### E — 예외 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| E-FE-SOCAR-UX-001 | docs 미정의 화면/동작 발견 | 구현 중 API/DB/design behavior 불명확 | 임의 구현하지 않고 `BLOCKED`로 보고하고 docs/계획서를 먼저 갱신한다. |
| E-FE-SOCAR-UX-002 | Direct Socar package adoption 시도 | Socar Frame npm install/import | Tailwind v4 호환성 검증 계획 없이는 제외하고 PR에 제외 확인을 남긴다. |
| E-FE-SOCAR-UX-003 | Auth 필요 화면 미인증 접근 | `/`, `/matchings`, `/profile` 등 | 기존 AuthGuard redirect behavior를 변경하지 않고 error/redirect visual state만 검증한다. |
| E-FE-SOCAR-UX-004 | 데이터 로드 실패 | GraphQL/API error | error card/banner에 원인, 재시도 CTA, 이전 화면 이동 수단이 제공된다. |

### X — 엣지 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| X-FE-SOCAR-UX-001 | 긴 텍스트 | 긴 경로, 회사명, 사용자명, 마지막 메시지 | 카드/리스트/TopAppBar가 overflow 없이 wrap/truncate 규칙을 적용한다. |
| X-FE-SOCAR-UX-002 | 작은 화면/safe area | 320px mobile, iOS safe-area | bottom nav, sticky CTA, chat input이 콘텐츠를 가리지 않는다. |
| X-FE-SOCAR-UX-003 | 빈 데이터 | 빈 매칭/채팅/정산/차량/리뷰 | 단일 primary CTA와 설명 문구가 표시되고 임의 데이터가 노출되지 않는다. |
| X-FE-SOCAR-UX-004 | 상태 badge 조합 | OPEN/MATCHED/PENDING/FAILED/CANCELLED/REFUNDED | badge 색상과 텍스트가 함께 표시되고 CTA와 충돌하지 않는다. |
| X-FE-SOCAR-UX-005 | Desktop 확장 | 1024px 이상 | list/detail, summary grid, chat split layout이 중앙 컨테이너/2열 규칙을 따른다. |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-SOCAR-UX-001 | A-FE-SOCAR-UX-001, X-FE-SOCAR-UX-002, X-FE-SOCAR-UX-005 | `app/layout.tsx`, `app/globals.css`, 선택 시 `components/ridy/PageShell.tsx` — page container, safe-area, responsive max-width, bottom nav offset | `__tests__/layoutPatterns.test.tsx` — `페이지 shell이 mobile safe-area와 desktop max-width class를 제공한다` | 공통 layout token/class가 적용되고 mobile/desktop layout test PASS |
| FE-SOCAR-UX-002 | A-FE-SOCAR-UX-002, E-FE-SOCAR-UX-003, X-FE-SOCAR-UX-001 | `app/landing/page.tsx`, `app/login/page.tsx`, `app/profile/setup/page.tsx`, `components/auth/LoginForm.tsx` — public/auth/onboarding hierarchy | `__tests__/AuthFlow.test.tsx` — `로그인/가입 화면은 단일 primary CTA와 token 기반 안내를 표시한다`; `__tests__/LandingPage.test.tsx` — `랜딩 hero와 지표 섹션을 예측 가능한 hierarchy로 표시한다` | public/auth 화면 CTA/hierarchy/accessibility PASS |
| FE-SOCAR-UX-003 | A-FE-SOCAR-UX-003, X-FE-SOCAR-UX-003, X-FE-SOCAR-UX-004 | `app/page.tsx`, `app/driver/page.tsx`, `components/ridy/RouteInput.tsx`, `components/ridy/MatchingCard.tsx` — rider/driver home card stack and state sections | `__tests__/HomeScreens.test.tsx` — `탑승자 홈이 검색과 정기 카풀 empty CTA를 표시한다`; `차주 홈이 운행 등록과 요청 상태 badge를 표시한다` | rider/driver home state UI PASS |
| FE-SOCAR-UX-004 | A-FE-SOCAR-UX-004, E-FE-SOCAR-UX-004, X-FE-SOCAR-UX-001, X-FE-SOCAR-UX-004, X-FE-SOCAR-UX-005 | `app/matchings/page.tsx`, `app/matchings/[id]/page.tsx`, `components/ridy/MatchingCard.tsx`, 선택 시 `components/ridy/FilterPanel.tsx` — matching list/detail responsive state | `__tests__/MatchingScreens.test.tsx` — `매칭 결과가 검색 요약/필터/카드 정보를 순서대로 표시한다`; `매칭 상세이 full/cancelled/alreadyRequested CTA 상태를 표시한다` | matching list/detail states and responsive hooks PASS |
| FE-SOCAR-UX-005 | A-FE-SOCAR-UX-005, E-FE-SOCAR-UX-004, X-FE-SOCAR-UX-001, X-FE-SOCAR-UX-002, X-FE-SOCAR-UX-003 | `app/chat/page.tsx`, `app/chat/[id]/page.tsx`, 선택 시 `components/ridy/ChatRoomList.tsx`, `components/ridy/MessageBubble.tsx` — chat list/room layout | `__tests__/ChatScreens.test.tsx` — `채팅 목록이 unread badge와 empty CTA를 표시한다`; `채팅방이 message bubble과 fixed input safe-area를 유지한다` | chat list/room state UI PASS |
| FE-SOCAR-UX-006 | A-FE-SOCAR-UX-006, X-FE-SOCAR-UX-003, X-FE-SOCAR-UX-004, X-FE-SOCAR-UX-005 | `app/payments/page.tsx`, `app/profile/page.tsx`, `app/reviews/[matchingId]/page.tsx`, 선택 시 `components/ridy/SummaryCard.tsx`, `components/ridy/RatingInput.tsx` — payments/profile/review screen hierarchy | `__tests__/AccountScreens.test.tsx` — `정산 화면이 요약 카드와 상태 badge를 표시한다`; `마이페이지가 차량 empty CTA와 위험 동작을 분리한다`; `리뷰 화면이 별점 미선택 시 제출을 disabled 처리한다` | payments/profile/review states PASS |
| FE-SOCAR-UX-007 | A-FE-SOCAR-UX-007, E-FE-SOCAR-UX-004, X-FE-SOCAR-UX-003 | `components/ridy/StateView.tsx`, `components/ridy/AlertBanner.tsx`, affected screen files — loading/empty/error/offline common state | `__tests__/StateViews.test.tsx` — `empty/error/offline 상태가 원인 문구와 다음 행동 CTA를 제공한다`; affected screen tests | 공통 상태 UI가 색상+텍스트+action으로 검증됨 |
| FE-SOCAR-UX-008 | A-FE-SOCAR-UX-008, E-FE-SOCAR-UX-003, X-FE-SOCAR-UX-002, X-FE-SOCAR-UX-005 | Visual QA execution — `/landing`, `/login`, `/profile/setup`, `/`, `/driver`, `/matchings`, `/matchings/:id`, `/chat`, `/chat/:id`, `/payments`, `/profile`, `/reviews/:matchingId` | Visual QA evidence — screenshot paths and/or Chrome verification log in PR | 주요 화면 public/authenticated visual QA 결과가 PR에 첨부됨 |
| FE-SOCAR-UX-009 | E-FE-SOCAR-UX-001, E-FE-SOCAR-UX-002 | Scope guard — package direct adoption and undocumented behavior exclusion | PR Case ID 확인표 — `Socar package direct adoption 없음`; `docs 미정의 동작 추가 없음` | 제외/guard 조건이 PASS로 명시됨 |

## 수정/생성할 파일 경로

### 필수 후보 구현 파일

- `frontend/app/layout.tsx`
- `frontend/app/globals.css`
- `frontend/app/landing/page.tsx`
- `frontend/app/login/page.tsx`
- `frontend/app/profile/setup/page.tsx`
- `frontend/app/page.tsx`
- `frontend/app/driver/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/app/matchings/[id]/page.tsx`
- `frontend/app/chat/page.tsx`
- `frontend/app/chat/[id]/page.tsx`
- `frontend/app/payments/page.tsx`
- `frontend/app/profile/page.tsx`
- `frontend/app/reviews/[matchingId]/page.tsx`
- `frontend/components/auth/LoginForm.tsx`
- `frontend/components/ridy/BottomNavigation.tsx`
- `frontend/components/ridy/MatchingCard.tsx`
- `frontend/components/ridy/RouteInput.tsx`

### 조건부 신규 공통 컴포넌트

- `frontend/components/ridy/PageShell.tsx`
- `frontend/components/ridy/TopAppBar.tsx`
- `frontend/components/ridy/StateView.tsx`
- `frontend/components/ridy/AlertBanner.tsx`
- `frontend/components/ridy/FilterPanel.tsx`
- `frontend/components/ridy/SummaryCard.tsx`
- `frontend/components/ridy/ChatRoomList.tsx`
- `frontend/components/ridy/MessageBubble.tsx`
- `frontend/components/ridy/RatingInput.tsx`

### 테스트 파일

- `frontend/__tests__/layoutPatterns.test.tsx`
- `frontend/__tests__/LandingPage.test.tsx`
- `frontend/__tests__/AuthFlow.test.tsx`
- `frontend/__tests__/HomeScreens.test.tsx`
- `frontend/__tests__/MatchingScreens.test.tsx`
- `frontend/__tests__/ChatScreens.test.tsx`
- `frontend/__tests__/AccountScreens.test.tsx`
- `frontend/__tests__/StateViews.test.tsx`
- Visual QA evidence: PR 본문 screenshot path/log

## TDD 순서

1. Red — `layoutPatterns.test.tsx`에 PageShell/safe-area/responsive container 실패 테스트 작성.
2. Green — 공통 layout token/class 또는 `PageShell` 최소 구현.
3. Red — public/auth 화면 테스트(`LandingPage.test.tsx`, `AuthFlow.test.tsx`)에 CTA hierarchy와 token state 테스트 작성.
4. Green — `/landing`, `/login`, `/profile/setup` 최소 UI 고도화.
5. Red — `HomeScreens.test.tsx`에 rider/driver home empty/has data/request state 테스트 작성.
6. Green — `/`, `/driver` 및 관련 Ridy components 최소 UI 고도화.
7. Red — `MatchingScreens.test.tsx`에 list/detail/filter/CTA state 테스트 작성.
8. Green — `/matchings`, `/matchings/[id]` 최소 UI 고도화.
9. Red — `ChatScreens.test.tsx`에 chat empty/unread/message/input safe-area 테스트 작성.
10. Green — `/chat`, `/chat/[id]` 최소 UI 고도화.
11. Red — `AccountScreens.test.tsx`에 payments/profile/reviews 상태 테스트 작성.
12. Green — `/payments`, `/profile`, `/reviews/[matchingId]` 최소 UI 고도화.
13. Red — `StateViews.test.tsx`에 loading/empty/error/offline 공통 상태 테스트 작성.
14. Green — 공통 `StateView`/`AlertBanner` 또는 화면별 상태 UI 최소 구현.
15. Refactor — 반복 class/component를 공통화하되 route/API behavior 변경 금지.
16. Verify — 전체 검증 명령 실행.
17. Visual QA — public/authenticated 주요 화면 screenshot 또는 browser verification 결과를 PR에 기록.

## 실행할 검증 명령

```bash
npm run test
npm run lint
npm run build
```

GraphQL operation 변경이 없는 계획이므로 `npm run codegen`은 `npm run test`와 `npm run build` 내부 실행으로 충분합니다. GraphQL operation이나 generated artifact에 영향이 생기면 별도 `npm run codegen`을 먼저 실행해야 합니다.

## PR Case ID 확인표 요구사항

Frontend PR에는 아래 형식의 확인표를 포함해야 합니다.

| Case ID | 구현 파일/단위 | 테스트 파일/테스트명 | 상태 |
|---|---|---|---|
| FE-SOCAR-UX-001 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-002 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-003 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-004 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-005 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-006 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-007 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-008 |  |  | PASS/BLOCKED |
| FE-SOCAR-UX-009 |  |  | PASS/BLOCKED |

`BLOCKED`가 남아 있으면 PR은 리뷰/머지 대상이 아닙니다. 단, `FE-SOCAR-UX-009`는 제외/guard 조건 확인으로 `PASS` 처리해야 합니다.

## 완료 조건

- 모든 주요 frontend 화면이 `DESIGN_SYSTEM.md` token/component 규칙과 `SCREENS.md` 상태 규칙을 따른다.
- 모든 `FE-SOCAR-UX-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
- `npm run test`, `npm run lint`, `npm run build`가 통과한다.
- public/authenticated visual QA 결과가 PR 본문에 기록된다.
- Socar Frame package direct adoption을 하지 않았음을 PR에 명시한다.
- docs에 없는 API/DB/design behavior를 추가하지 않았음을 PR에 명시한다.

## 범위 제외

- API/GraphQL/DB 동작 변경
- Socar Frame npm package 직접 설치
- Tailwind v3 downgrade 또는 Socar Frame config 복제
- IA/route 변경
- Pixel-perfect Socar Frame clone
- Figma parity 보장
- Storybook 도입
- 신규 business logic 또는 undocumented copy/flow 추가

## BLOCKED 조건

- 대상 route가 frontend에 존재하지 않거나 현재 스펙과 충돌하는 경우
- `SCREENS.md`/`WIREFRAMES.md`에 없는 상태, CTA, API 응답, DB 필드가 필요한 경우
- Case ID별 구현/테스트 연결표를 작성할 수 없는 경우
- Socar Frame package 직접 설치가 필요하다고 판단되는 경우
- visual QA fixture로 authenticated 화면을 재현할 수 없는 경우
