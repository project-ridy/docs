# FE — KT Seamless Flow 디자인시스템 전환 구현 계획

## 목표

`docs/design/DESIGN_SYSTEM.md`에 정의된 KT Seamless Flow 참고 기반 토큰과 컴포넌트 규칙을 프론트엔드에 적용해 기존 Socar UX refresh 방향을 대체한다. 변경 범위는 UI token/pattern migration에 한정하며, API/DB/GraphQL 동작은 변경하지 않는다.

## 관련 이슈

- Docs: `project-ridy/docs#58` — KT 디자인시스템 전환 SSoT 및 구현 계획 작성
- Frontend: 새 기능 이슈 생성 필요 — 이 계획서의 Case ID 표를 이슈 본문에 복사한다.
- Superseded: `project-ridy/frontend#46`, `project-ridy/frontend#47` — Socar UX 방향으로 작성된 작업은 닫거나 superseded 상태로 관리한다.

## 관련 docs 문서

- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/WORKFLOW.md`
- `agents/protocol/AGENT_PROTOCOL.md`

## 아키텍처 접근

1. `frontend/app/globals.css`에서 KT 기반 primitive/semantic token을 Tailwind v4 `@theme`와 CSS custom properties로 매핑한다.
2. `components/ui/*`와 `components/ridy/*`는 token name을 사용하도록 정리한다. 임의 hex, 임의 spacing, 과도한 gradient를 제거한다.
3. 화면별 구조는 기존 route/API/GraphQL operation을 유지하고, visual hierarchy·state feedback·responsive shell만 변경한다.
4. 새 UI 컴포넌트는 먼저 테스트를 작성한 뒤 최소 구현한다.
5. Visual QA는 mobile 390px, tablet 768px, desktop 1280px 기준으로 핵심 경로를 캡처한다.

## 범위

### 포함

- KT Seamless Flow 기준 색상, 타이포, spacing, radius, elevation token 적용
- Button/Input/Card/Badge/Tab/Alert/TopAppBar/BottomNavigation/MatchingCard 패턴 정리
- 랜딩, 인증, 홈, 차주 홈, 매칭, 채팅, 정산, 프로필, 리뷰 화면의 visual hierarchy migration
- 접근성 회귀 테스트 및 visual QA evidence

### 제외

- KT Figma UI Kit, Zeroheight private asset, npm package 직접 도입
- KT Flow font 실사용 적용
- API/DB/GraphQL schema 변경
- 신규 비즈니스 플로우 추가
- pixel-perfect clone

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-FE-KT-001 | A | 앱 전역 token이 KT 기반 semantic token으로 매핑됨 | `globals.css`와 component class가 문서 token을 사용 |
| A-FE-KT-002 | A | 공통 UI primitive가 KT radius/elevation/accessibility 규칙을 따름 | button/input/card/badge/tab/alert가 44px touch, focus, 상태 텍스트를 제공 |
| A-FE-KT-003 | A | Public/Auth 화면이 Gray-first hierarchy와 단일 primary CTA를 사용 | 랜딩/로그인 화면에서 핵심 CTA가 하나로 식별됨 |
| A-FE-KT-004 | A | Core app 화면이 KT 기반 surface/card/list hierarchy를 사용 | 홈/매칭/채팅/정산/프로필/리뷰가 동일한 visual language를 공유 |
| A-FE-KT-005 | A | Responsive shell이 mobile/tablet/desktop 기준에 맞음 | safe-area, max-width, 2열 전환, bottom nav offset이 유지됨 |
| E-FE-KT-001 | E | API error/loading/empty/offline 상태 | 색상만이 아니라 원인·다음 행동 텍스트와 CTA를 제공 |
| E-FE-KT-002 | E | disabled/action unavailable 상태 | opacity만으로 전달하지 않고 helper text를 제공 |
| X-FE-KT-001 | X | gray surface 위 텍스트/critical state | `gray-550`, `red-600` 등 대비 강화 token을 사용 |
| X-FE-KT-002 | X | reduced motion 환경 | 과도한 transition/animation을 제거하거나 축소 |
| X-FE-KT-003 | X | 긴 한글/조직명/경로명 | card/list layout이 깨지지 않고 truncate/wrap 규칙 적용 |
| X-FE-KT-004 | X | mobile one-hand usage | 주요 CTA와 bottom nav touch target이 44px 이상 |
| X-FE-KT-005 | X | desktop wide layout | 1120px max-width와 side/list-detail split 유지 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-KT-001 | A-FE-KT-001, X-FE-KT-001 | `app/globals.css` — KT primitive/semantic color, typography, spacing, radius, shadow tokens | `__tests__/designTokens.test.ts` — `KT semantic token이 CSS theme 변수로 노출된다` | 문서 token과 CSS token 이름/용도가 일치 |
| FE-KT-002 | A-FE-KT-002, E-FE-KT-002, X-FE-KT-004 | `components/ui/button.tsx`, `components/ui/input.tsx`, `components/ui/card.tsx` | `__tests__/uiPrimitives.test.tsx` — `버튼과 입력이 KT touch target과 focus 스타일을 제공한다` | touch target 44px 이상, focus visible, disabled helper 가능 |
| FE-KT-003 | A-FE-KT-002, E-FE-KT-001 | `components/ridy/StateView.tsx`, `components/ridy/AlertBanner.tsx` | `__tests__/StateViews.test.tsx` — `상태 컴포넌트가 원인과 다음 행동을 제공한다` | empty/error/offline/loading 상태가 텍스트+CTA로 전달 |
| FE-KT-004 | A-FE-KT-003, X-FE-KT-003 | `app/landing/page.tsx`, `components/auth/LoginForm.tsx` | `__tests__/LandingPage.test.tsx`, `__tests__/AuthFlow.test.tsx` — `랜딩/인증 화면이 KT hierarchy와 단일 primary CTA를 사용한다` | Public/Auth 화면이 Gray-first + primary CTA 1개 원칙 준수 |
| FE-KT-005 | A-FE-KT-004, X-FE-KT-004, X-FE-KT-005 | `app/page.tsx`, `app/driver/page.tsx`, `components/ridy/PageShell.tsx`, `components/ridy/BottomNavigation.tsx` | `__tests__/HomeScreens.test.tsx`, `__tests__/layoutPatterns.test.tsx` — `홈 화면과 PageShell이 KT responsive 기준을 따른다` | mobile safe-area, desktop max-width, driver/passenger hierarchy 적용 |
| FE-KT-006 | A-FE-KT-004, E-FE-KT-002, X-FE-KT-003 | `components/ridy/MatchingCard.tsx`, `app/matchings/page.tsx`, `app/matchings/[id]/page.tsx` | `__tests__/Matchings.test.tsx` — `매칭 카드가 KT badge/CTA 상태 규칙을 따른다` | 상태 badge, disabled 사유, route hierarchy 유지 |
| FE-KT-007 | A-FE-KT-004, E-FE-KT-001, X-FE-KT-004 | `app/chat/page.tsx`, `app/chat/[id]/page.tsx` | `__tests__/Chat.test.tsx` — `채팅 목록과 대화방이 KT surface와 fixed input safe-area를 사용한다` | message bubble, empty state, input bar 접근성 유지 |
| FE-KT-008 | A-FE-KT-004, X-FE-KT-001, X-FE-KT-003 | `app/payments/page.tsx`, `app/profile/page.tsx`, `app/reviews/[matchingId]/page.tsx` | `__tests__/Payments.test.tsx`, `__tests__/Profile.test.tsx`, `__tests__/Reviews.test.tsx` — `정산/프로필/리뷰 화면이 KT feedback hierarchy를 따른다` | 정산 요약, vehicle empty CTA, review rating guide가 token 기반으로 표시 |
| FE-KT-009 | A-FE-KT-005, X-FE-KT-002, X-FE-KT-005 | Visual QA fixture and screenshots | Visual QA evidence paths in PR — mobile/tablet/desktop 핵심 화면 캡처 | screenshot evidence와 reduced-motion 확인 결과를 PR에 첨부 |
| FE-KT-010 | E-FE-KT-001, E-FE-KT-002 | Scope guard — API/DB/GraphQL 미변경 확인 | PR 확인표 + `npm run test`, `npm run lint`, `npm run build` | schema/operation/generated 변경이 없거나 필요한 경우 codegen 결과만 포함 |

## 수정/생성할 파일 경로

### Frontend implementation

- `frontend/app/globals.css`
- `frontend/components/ui/button.tsx`
- `frontend/components/ui/input.tsx`
- `frontend/components/ui/card.tsx`
- `frontend/components/ridy/PageShell.tsx`
- `frontend/components/ridy/StateView.tsx`
- `frontend/components/ridy/AlertBanner.tsx`
- `frontend/components/ridy/BottomNavigation.tsx`
- `frontend/components/ridy/MatchingCard.tsx`
- `frontend/app/landing/page.tsx`
- `frontend/components/auth/LoginForm.tsx`
- `frontend/app/page.tsx`
- `frontend/app/driver/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/app/matchings/[id]/page.tsx`
- `frontend/app/chat/page.tsx`
- `frontend/app/chat/[id]/page.tsx`
- `frontend/app/payments/page.tsx`
- `frontend/app/profile/page.tsx`
- `frontend/app/reviews/[matchingId]/page.tsx`

### Frontend tests

- `frontend/__tests__/designTokens.test.ts`
- `frontend/__tests__/uiPrimitives.test.tsx`
- `frontend/__tests__/layoutPatterns.test.tsx`
- `frontend/__tests__/LandingPage.test.tsx`
- `frontend/__tests__/AuthFlow.test.tsx`
- `frontend/__tests__/HomeScreens.test.tsx`
- `frontend/__tests__/Matchings.test.tsx`
- `frontend/__tests__/Chat.test.tsx`
- `frontend/__tests__/Payments.test.tsx`
- `frontend/__tests__/Profile.test.tsx`
- `frontend/__tests__/Reviews.test.tsx`
- `frontend/__tests__/StateViews.test.tsx`

## TDD 순서

1. FE-KT-001: `designTokens.test.ts` Red → `globals.css` token Green → 불필요한 임의 값 제거 Refactor
2. FE-KT-002~003: primitive/state component tests Red → UI primitive/state component Green → shared class 정리
3. FE-KT-004: landing/auth tests Red → 화면 hierarchy Green → CTA/typography Refactor
4. FE-KT-005~008: 화면군별 tests Red → 최소 화면 변경 Green → PageShell/MatchingCard 중심 중복 제거
5. FE-KT-009: visual QA fixture 실행 → screenshot evidence 첨부
6. FE-KT-010: scope guard와 전체 verification 실행

## 실행할 검증 명령

```bash
npm run test -- --run __tests__/designTokens.test.ts __tests__/uiPrimitives.test.tsx __tests__/layoutPatterns.test.tsx
npm run test
npm run lint
npm run build
```

GraphQL operation 변경이 생긴 경우에만 다음을 추가 실행합니다.

```bash
npm run codegen
```

## 완료 조건

- 모든 `FE-KT-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결됨
- PR 본문에 Case ID 구현/테스트 확인표가 포함됨
- KT Figma UI Kit/npm package/비공개 asset을 직접 도입하지 않음
- API/DB/GraphQL 동작을 변경하지 않음
- `npm run test`, `npm run lint`, `npm run build` 통과
- Visual QA screenshot evidence와 reduced-motion 확인 결과를 PR에 첨부
