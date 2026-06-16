# Frontend Socar Frame 기반 디자인 시스템 개선 구현 계획

## 목표

- `docs#54`에서 정리한 Socar Frame 참고 원칙을 Ridy frontend 디자인 시스템에 적용한다.
- Ridy의 기존 화면 구조와 GraphQL/API 동작은 변경하지 않고, 색상·타이포그래피·간격·컴포넌트 상태 규칙을 `docs/design/DESIGN_SYSTEM.md`에 맞게 정렬한다.
- Tailwind CSS 4 기반 현재 frontend 구조를 유지하면서 token-first 방식으로 구현한다.

## 아키텍처 접근

- Socar Frame package를 직접 설치하지 않는다.
  - Socar Frame foundation 문서는 Tailwind v3.4.1 기준이다.
  - Ridy frontend는 Tailwind v4.3.0을 사용하므로 direct adoption은 별도 spike 전까지 `BLOCKED`이다.
- `app/globals.css`의 CSS custom properties와 Tailwind v4 `@theme` token을 SSoT에 맞춰 확장한다.
- shadcn/ui + Base UI 기반 기존 컴포넌트를 유지하고, variant/size/state class만 최소 수정한다.
- Ridy custom component는 정보 구조와 접근성 상태를 개선한다.
- UI 변경 후 Playwright/Chrome 기반 visual QA 또는 스크린샷 검증 결과를 PR에 남긴다.

## 관련 이슈

- docs: `project-ridy/docs#54` — `[DOCS] Socar Frame 기반 Ridy 디자인 시스템 개선 기획`
- frontend 구현 이슈: `project-ridy/frontend#41` — `[FEAT] Socar Frame 기반 디자인 시스템 개선`

## 관련 docs 문서

- `docs/WORKFLOW.md`
- `docs/AGENTS.md`
- `agents/protocol/AGENT_PROTOCOL.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `frontend/AGENTS.md`

## 관련 Socar Frame 참고 문서

- `https://socarframe.socar.kr/development/principle`
- `https://socarframe.socar.kr/development/foundation`
- `https://socarframe.socar.kr/development/foundation/Colors`
- `https://socarframe.socar.kr/development/foundation/Spacing`
- `https://socarframe.socar.kr/development/foundation/Typography`
- `https://socarframe.socar.kr/development/components`
- `https://socarframe.socar.kr/development/components/Buttons`
- `https://socarframe.socar.kr/development/components/Buttons/ActionButton`
- `https://socarframe.socar.kr/development/components/Input`

## A/E/X 케이스 분석

### A — 정상 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| A-FE-SOCAR-DESIGN-001 | Token 확장 | 앱 로드 | `globals.css`에 palette/semantic/spacing/radius/elevation token이 존재하고 Tailwind theme로 노출된다. |
| A-FE-SOCAR-DESIGN-002 | Button variant 정렬 | primary/secondary/ghost/destructive/action | 문서화된 목적별 variant와 44~48px touch target을 제공한다. |
| A-FE-SOCAR-DESIGN-003 | Input 상태 정렬 | default/focus/error/disabled | label/aria-label, 48px 높이, focus/error 상태가 token 기반으로 표시된다. |
| A-FE-SOCAR-DESIGN-004 | Badge/Chip 상태 정렬 | OPEN/MATCHED/PENDING/FAILED/CANCELLED | 상태별 subtle background + text 색상과 상태 텍스트가 함께 표시된다. |
| A-FE-SOCAR-DESIGN-005 | MatchingCard 정보 구조 개선 | driver/route/fare/seats/status | `title → route/time → price/seats/status` 순서로 표시되고 클릭 가능 카드 접근성이 유지된다. |
| A-FE-SOCAR-DESIGN-006 | Navigation/Tab 예측 가능성 | active tab/auth tab | active 상태가 token, `aria-current` 또는 `aria-selected`로 표현된다. |
| A-FE-SOCAR-DESIGN-007 | 화면 visual QA | `/landing`, `/login`, `/`, `/matchings`, `/profile` | 주요 화면이 깨짐 없이 token 기반 스타일을 표시한다. |

### E — 예외 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| E-FE-SOCAR-DESIGN-001 | Direct package adoption 시도 | `@socar-inc/socar-frame-foundation` 설치/import | Tailwind v4 호환성 spike 없이는 구현 범위에서 제외하고 `BLOCKED`로 보고한다. |
| E-FE-SOCAR-DESIGN-002 | Disabled CTA | disabled button | opacity만으로 의미를 전달하지 않고 disabled 상태와 helper/error text가 함께 유지된다. |
| E-FE-SOCAR-DESIGN-003 | Error state | form validation/API error | `danger` token과 에러 문구가 함께 표시된다. |
| E-FE-SOCAR-DESIGN-004 | Missing handler | clickable custom component with no handler | button role/tabIndex를 부여하지 않아 keyboard trap이 생기지 않는다. |

### X — 엣지 케이스

| Case | 설명 | 입력/상태 | 기대 결과 |
|---|---|---|---|
| X-FE-SOCAR-DESIGN-001 | 긴 경로/이름 | 긴 출발지/도착지/사용자명 | card layout이 overflow 없이 줄바꿈/말줄임 처리된다. |
| X-FE-SOCAR-DESIGN-002 | 잔여석 0/1석 | `availableSeats` 0 또는 1 | 상태 badge/문구가 CTA와 충돌하지 않는다. |
| X-FE-SOCAR-DESIGN-003 | Mobile safe area | iOS safe-area inset | 하단 navigation/CTA가 safe area를 침범하지 않는다. |
| X-FE-SOCAR-DESIGN-004 | Dark class 존재 | `.dark` 적용 | shadcn/ui 기본 token이 깨지지 않고 Ridy token과 충돌하지 않는다. |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-SOCAR-DESIGN-001 | A-FE-SOCAR-DESIGN-001, X-FE-SOCAR-DESIGN-004 | `app/globals.css` — palette, semantic color, typography, spacing, radius, elevation CSS/Tailwind token | `__tests__/designTokens.test.ts` — `Socar Frame 참고 기반 Ridy palette token을 노출한다`; `semantic/radius/elevation token을 노출한다`; `dark token과 충돌하지 않는다` | 문서 token이 CSS custom property와 `@theme`에 연결되고 test PASS |
| FE-SOCAR-DESIGN-002 | A-FE-SOCAR-DESIGN-002, E-FE-SOCAR-DESIGN-002 | `components/ui/button.tsx` — 목적 기반 variant/size/action button state | `__tests__/UiComponents.test.tsx` — `목적별 버튼 variant를 렌더링한다`; `모바일 주요 CTA touch target을 유지한다`; `disabled 버튼은 클릭되지 않는다` | Primary/Secondary/Ghost/Destructive/Action variant와 disabled 상태 PASS |
| FE-SOCAR-DESIGN-003 | A-FE-SOCAR-DESIGN-003, E-FE-SOCAR-DESIGN-003 | `components/ui/input.tsx`; `components/ridy/RouteInput.tsx` — input height/focus/error/leading icon 규칙 | `__tests__/UiComponents.test.tsx` — `입력 필드 상태 class를 노출한다`; `__tests__/RidyComponents.test.tsx` — `RouteInput label과 변경 이벤트를 유지한다` | 48px input, label/aria-label, error/focus token PASS |
| FE-SOCAR-DESIGN-004 | A-FE-SOCAR-DESIGN-004, E-FE-SOCAR-DESIGN-003 | `components/ui/badge.tsx` — ride/payment/review 상태별 semantic variant | `__tests__/UiComponents.test.tsx` — `상태 badge variant를 상태 텍스트와 함께 렌더링한다` | OPEN/MATCHED/PENDING/FAILED/CANCELLED/neutral badge PASS |
| FE-SOCAR-DESIGN-005 | A-FE-SOCAR-DESIGN-005, E-FE-SOCAR-DESIGN-004, X-FE-SOCAR-DESIGN-001, X-FE-SOCAR-DESIGN-002 | `components/ridy/MatchingCard.tsx` — 정보 hierarchy, 상태 badge, keyboard activation, overflow handling | `__tests__/RidyComponents.test.tsx` — `MatchingCard가 상태와 CTA를 예측 가능한 순서로 표시한다`; `클릭 가능한 카드 keyboard activation을 지원한다`; `만석/긴 경로에서도 레이아웃이 유지된다` | 카드 정보 구조/상태/접근성 PASS |
| FE-SOCAR-DESIGN-006 | A-FE-SOCAR-DESIGN-006, X-FE-SOCAR-DESIGN-003 | `components/ridy/BottomNavigation.tsx`; `components/auth/LoginForm.tsx`; 선택 시 `components/ridy/TopAppBar.tsx` 신규 | `__tests__/RidyComponents.test.tsx` — `BottomNavigation active tab을 token과 aria-current로 표시한다`; `__tests__/AuthFlow.test.tsx` — `인증 탭 active 상태를 aria-selected와 token으로 표시한다` | Navigation/Tab/TopAppBar 패턴 PASS |
| FE-SOCAR-DESIGN-007 | A-FE-SOCAR-DESIGN-007 | 주요 화면: `app/landing/page.tsx`, `app/login/page.tsx`, `app/page.tsx`, `app/matchings/page.tsx`, `app/profile/page.tsx`에서 token class 회귀 확인 | Visual QA 기록 — `/landing`, `/login`, `/`, `/matchings`, `/profile` screenshot 또는 browser verification 결과 | UI 깨짐/명백한 token 누락 없음, PR에 visual QA 결과 첨부 |
| FE-SOCAR-DESIGN-008 | E-FE-SOCAR-DESIGN-001 | 구현 범위 관리 — package install/import 금지 | PR Case ID 확인표 — `Socar Frame package direct adoption: BLOCKED/Excluded` 명시 | Tailwind v4 호환성 검증 없는 direct adoption 없음 |

## 수정/생성할 파일 경로

### 필수 구현 파일

- `frontend/app/globals.css`
- `frontend/components/ui/button.tsx`
- `frontend/components/ui/input.tsx`
- `frontend/components/ui/badge.tsx`
- `frontend/components/ridy/MatchingCard.tsx`
- `frontend/components/ridy/RouteInput.tsx`
- `frontend/components/ridy/BottomNavigation.tsx`
- `frontend/components/auth/LoginForm.tsx`

### 조건부 구현 파일

- `frontend/components/ridy/TopAppBar.tsx` — 화면 상단 패턴을 공통화할 때만 생성
- `frontend/app/landing/page.tsx`
- `frontend/app/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/app/profile/page.tsx`

### 테스트 파일

- `frontend/__tests__/designTokens.test.ts`
- `frontend/__tests__/UiComponents.test.tsx`
- `frontend/__tests__/RidyComponents.test.tsx`
- `frontend/__tests__/AuthFlow.test.tsx`
- visual QA evidence: PR 본문 screenshot/로그 또는 별도 첨부

## TDD 순서

1. Red — `designTokens.test.ts`에 palette/semantic/radius/elevation token 실패 테스트 추가.
2. Green — `app/globals.css` token을 최소 추가해 통과.
3. Red — `UiComponents.test.tsx`에 Button/Input/Badge variant와 disabled/error 상태 테스트 추가.
4. Green — `components/ui/*` variant/size/state class 최소 수정.
5. Red — `RidyComponents.test.tsx`에 MatchingCard/RouteInput/BottomNavigation 접근성·상태 테스트 추가.
6. Green — Ridy custom component를 문서 규칙에 맞게 최소 수정.
7. Red — `AuthFlow.test.tsx`에 auth tab active token/aria 테스트 추가.
8. Green — `LoginForm` segmented tab styling을 token 기반으로 정렬.
9. Refactor — 반복 class를 `cn`/variant로 정리하되 public API 변경은 최소화.
10. Verify — 전체 검증 명령 실행.
11. Visual QA — `/landing`, `/login`, `/`, `/matchings`, `/profile`에서 스크린샷/브라우저 확인 결과를 PR에 기록.

## 실행할 검증 명령

```bash
npm run test
npm run lint
npm run build
```

GraphQL operation이 변경되지 않는 계획이므로 `npm run codegen`은 `npm run test`와 `npm run build` 내부 실행으로 충분합니다. GraphQL 파일을 수정하면 별도로 `npm run codegen`을 먼저 실행해야 합니다.

## PR Case ID 확인표 요구사항

Frontend PR에는 아래 형식의 확인표를 포함해야 합니다.

| Case ID | 구현 파일/단위 | 테스트 파일/테스트명 | 상태 |
|---|---|---|---|
| FE-SOCAR-DESIGN-001 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-002 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-003 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-004 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-005 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-006 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-007 |  |  | PASS/BLOCKED |
| FE-SOCAR-DESIGN-008 |  |  | PASS/BLOCKED |

`BLOCKED`가 남아 있으면 PR은 리뷰/머지 대상이 아닙니다. 단, `FE-SOCAR-DESIGN-008`은 direct package adoption 제외 확인으로 `PASS` 처리해야 합니다.

## 완료 조건

- `DESIGN_SYSTEM.md`의 token/component 규칙이 frontend token 및 component class와 연결된다.
- 모든 `FE-SOCAR-DESIGN-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
- `npm run test`, `npm run lint`, `npm run build`가 통과한다.
- UI 변경 후 visual QA 결과가 PR 본문에 기록된다.
- Socar Frame package direct adoption을 하지 않았음을 PR에 명시한다.

## 범위 제외

- API/GraphQL/DB 동작 변경
- Socar Frame npm package 직접 설치
- Tailwind v3 downgrade 또는 config 복제
- Storybook 도입
- 전체 화면 재설계 또는 IA 변경
- pixel-perfect Socar Frame clone
