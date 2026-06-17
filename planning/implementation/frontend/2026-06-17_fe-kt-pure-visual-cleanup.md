# FE — KT Pure Visual Cleanup 구현 계획

## 목표

`frontend#48` 머지 후 확인된 Socar/모빌리티 프로모션 감성 잔재를 제거하고, `docs/design/DESIGN_SYSTEM.md`의 KT Seamless Flow 재해석 원칙(Gray-first hierarchy, Accent 20% 제한, Token first)에 더 엄격히 맞춘다. 변경 범위는 시각 표현과 테스트 보강에 한정하며 API/DB/GraphQL 동작은 변경하지 않는다.

## 관련 이슈

- Frontend: `project-ridy/frontend#50` — KT 디자인 순수화 visual cleanup
- Docs: `project-ridy/docs#60` — KT 디자인 순수화 구현 계획서 추가
- Related: `project-ridy/frontend#48`, `project-ridy/frontend#49`

## 관련 docs 문서

- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/WORKFLOW.md`
- `agents/protocol/AGENT_PROTOCOL.md`

## 아키텍처 접근

1. 기존 route/API/GraphQL operation은 유지한다.
2. Socar처럼 보이는 강한 blue/green gradient, radial glow, 과한 blur/elevation을 제거하거나 Gray-first surface로 약화한다.
3. 문서 SSoT에 없는 `green-*`, `blue-900`, `red-900` 등 legacy/deep color token은 제거하고 `primary`, `secondary`, `success`, `danger` 또는 문서화된 palette 단계로 정리한다.
4. Pill shape는 badge/chip/indicator처럼 의미 있는 요소에만 유지하고, 버튼/카드/패널은 KT radius token 기준으로 정리한다.
5. 테스트는 visual intent를 직접 확인할 수 있는 class/token assertions를 먼저 추가/수정한다.

## 범위

### 포함

- Landing/Auth/Home의 강한 gradient/radial glow 제거 또는 Gray-first surface 전환
- `app/globals.css`의 문서 외 legacy color token 정리
- Button/Badge의 문서화 token 정합성 보정
- Home search card의 top gradient strip 제거
- 테스트 assertions 갱신
- mobile viewport visual smoke evidence

### 제외

- KT Figma UI Kit, npm package, 비공개 asset 직접 도입
- API/DB/GraphQL schema 또는 operation 변경
- 신규 화면/비즈니스 플로우 추가
- pixel-perfect KT clone

## A/E/X 케이스 분석

| Case | 유형 | 설명 | 기대 결과 |
|---|---|---|---|
| A-FE-KT-PURE-001 | A | Landing/Auth/Home이 Gray-first hierarchy를 유지함 | 배경/카드가 gray/white surface 중심이며 accent는 CTA와 상태에 제한 |
| A-FE-KT-PURE-002 | A | 토큰이 문서화된 semantic/foundation palette와 일치함 | `green-*`, `blue-900`, `red-900` 등 문서 외 token 제거 |
| A-FE-KT-PURE-003 | A | 카드/elevation/radius가 KT 규칙에 맞음 | 큰 glow/blur/프로모션 카드 느낌 제거, shadow level 제한 |
| E-FE-KT-PURE-001 | E | 기존 상태/disabled/error 표현은 유지됨 | cleanup 후에도 상태 텍스트와 CTA가 사라지지 않음 |
| X-FE-KT-PURE-001 | X | 모바일 viewport에서 과한 accent가 시야를 지배하지 않음 | hero/search/auth 첫 화면의 accent 면적이 낮아짐 |
| X-FE-KT-PURE-002 | X | 기존 FE-KT 테스트 가시성 token은 깨지지 않음 | tests가 KT 순수화 후에도 통과 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-KT-PURE-001 | A-FE-KT-PURE-002, X-FE-KT-PURE-002 | `app/globals.css` — 문서 외 deep/legacy color token 제거, semantic alias 정리 | `__tests__/designTokens.test.ts` — `KT palette가 문서화된 색상 단계만 노출한다` | 문서 외 `green-*`, `blue-900`, `red-900` 토큰 미노출 |
| FE-KT-PURE-002 | A-FE-KT-PURE-001, A-FE-KT-PURE-003, X-FE-KT-PURE-001 | `app/landing/page.tsx`, `components/auth/LoginForm.tsx` — radial gradient/glow/elevation 약화 | `__tests__/Landing.test.tsx`, `__tests__/AuthFlow.test.tsx` — `랜딩/인증 화면이 강한 radial gradient 없이 Gray-first surface를 사용한다` | hero/auth first view가 gray/white 중심 |
| FE-KT-PURE-003 | A-FE-KT-PURE-001, A-FE-KT-PURE-003, E-FE-KT-PURE-001 | `app/page.tsx` — home search card top gradient strip 제거, card elevation 낮춤 | `__tests__/Home.test.tsx` — `홈 검색 카드가 gradient strip 없이 KT surface hierarchy를 사용한다` | 홈 핵심 카드가 프로모션 배너처럼 보이지 않음 |
| FE-KT-PURE-004 | A-FE-KT-PURE-002, E-FE-KT-PURE-001 | `components/ui/badge.tsx`, `components/ui/button.tsx` — 문서화 token 기반 variant 정리 | `__tests__/UiComponents.test.tsx`, `__tests__/uiPrimitives.test.tsx` — `badge/button이 문서화 token을 사용한다` | 상태/액션 token이 docs palette와 일치 |
| FE-KT-PURE-005 | X-FE-KT-PURE-001, X-FE-KT-PURE-002 | Visual smoke evidence | PR screenshot evidence + `npm run test`, `npm run lint`, `npm run build` | mobile viewport에서 KT/Socar 혼합 인상 제거 확인 |

## 수정/생성할 파일 경로

- `frontend/app/globals.css`
- `frontend/app/landing/page.tsx`
- `frontend/components/auth/LoginForm.tsx`
- `frontend/app/page.tsx`
- `frontend/components/ui/badge.tsx`
- `frontend/components/ui/button.tsx`
- `frontend/__tests__/designTokens.test.ts`
- `frontend/__tests__/Landing.test.tsx`
- `frontend/__tests__/AuthFlow.test.tsx`
- `frontend/__tests__/Home.test.tsx`
- `frontend/__tests__/UiComponents.test.tsx`
- `frontend/__tests__/uiPrimitives.test.tsx`

## TDD 순서

1. FE-KT-PURE-001: token test Red → `globals.css` cleanup Green → legacy alias 제거 Refactor
2. FE-KT-PURE-002: landing/auth gradient 부재 assertions Red → 화면 cleanup Green → elevation/radius Refactor
3. FE-KT-PURE-003: home gradient strip 부재 assertion Red → home card cleanup Green
4. FE-KT-PURE-004: badge/button token assertion Red → variant cleanup Green
5. FE-KT-PURE-005: visual smoke + full verification

## 실행할 검증 명령

```bash
npm run test -- --run __tests__/designTokens.test.ts __tests__/Landing.test.tsx __tests__/AuthFlow.test.tsx __tests__/Home.test.tsx __tests__/UiComponents.test.tsx __tests__/uiPrimitives.test.tsx
npm run test
npm run lint
npm run build
```

## 완료 조건

- 모든 `FE-KT-PURE-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결됨
- 강한 blue/green radial gradient와 home gradient strip 제거
- 문서 외 legacy/deep color token 제거
- API/DB/GraphQL 동작 미변경
- `npm run test`, `npm run lint`, `npm run build` 통과
- PR 본문에 Case ID 구현/테스트 확인표와 visual evidence 포함
