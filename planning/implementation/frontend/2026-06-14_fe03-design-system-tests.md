# FE03 디자인 시스템 테스트 구현 계획

## 목표

- `frontend#3` 디자인 시스템 테스트를 보강한다.
- Ridy 커스텀 컴포넌트와 shadcn/ui 기반 기본 컴포넌트가 에러 없이 렌더링되고 접근성/이벤트/토큰 규칙을 지키는지 검증한다.
- 기존 화면 테스트 회귀 없이 디자인 시스템의 기본 안정성을 확보한다.

## 관련 이슈

- Frontend: `project-ridy/frontend#3` `[TEST] 디자인 시스템 테스트`
- 선행: `project-ridy/frontend#2` 디자인 시스템

## 관련 문서

- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/WIREFRAMES.md`
- `frontend/AGENTS.md`

## 현재 테스트 기준

- `frontend/__tests__/designTokens.test.ts`
  - 브랜드 컬러 CSS 변수
  - shadcn/ui CSS 변수
  - Pretendard font token
- `frontend/__tests__/RidyComponents.test.tsx`
  - `MatchingCard`
  - `RouteInput`
  - `BottomNavigation`

## 아키텍처 접근

- 기존 `RidyComponents.test.tsx`를 확장해 Ridy 커스텀 컴포넌트의 접근성/반응형 클래스/이벤트 케이스를 보강한다.
- 새 `UiComponents.test.tsx`를 추가해 기본 UI 컴포넌트 렌더링과 주요 상태를 검증한다.
- `designTokens.test.ts`는 DESIGN_SYSTEM.md의 spacing/radius/input/button token 연결 검증을 추가한다.
- 테스트는 Vitest + Testing Library 기준으로 작성한다.
- 구현 컴포넌트 변경은 테스트를 통과시키는 데 필요한 최소 수정만 허용한다.

## 테스트 범위

- Ridy 커스텀 컴포넌트
  - `MatchingCard`: button role, 클릭 이벤트, 좌석/요금 표시, hover/rounded/shadow class
  - `BottomNavigation`: nav landmark, active tab aria-current, tab click payload
  - `RouteInput`: 출발지/도착지 label, controlled value, 각각의 change handler
- shadcn/ui 기본 컴포넌트
  - `Button`: variant/disabled/click
  - `Input`: label 연결과 입력 이벤트
  - `Card`: card/card-content 렌더링
  - `Badge`: 텍스트와 variant 렌더링
  - `Avatar`: fallback 렌더링
  - `Tabs`: tab list/trigger/content 기본 렌더링
- 디자인 토큰
  - page padding token
  - card radius token
  - input height token
  - button radius token

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 커스텀 컴포넌트 렌더링 | `MatchingCard`, `RouteInput`, `BottomNavigation`이 에러 없이 표시된다. |
| A2 | 기본 UI 컴포넌트 렌더링 | `Button`, `Input`, `Card`, `Badge`, `Avatar`, `Tabs`가 에러 없이 표시된다. |
| A3 | 버튼/탭 클릭 | 전달된 이벤트 핸들러가 정확한 payload로 호출된다. |
| A4 | controlled input 입력 | 입력값 변경 핸들러가 호출된다. |
| A5 | active navigation tab | 활성 탭에 `aria-current="page"`가 설정된다. |
| A6 | 토큰 검증 | DESIGN_SYSTEM.md 기반 CSS token이 `globals.css`에 존재한다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | disabled button 클릭 | 클릭 핸들러가 호출되지 않는다. |
| E2 | `MatchingCard` onClick 미전달 | 버튼 렌더링은 유지되고 클릭해도 에러가 발생하지 않는다. |
| E3 | `RouteInput` 핸들러 미전달 | 입력 필드 렌더링은 유지된다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 잔여석 1석 | `1석`으로 표시된다. |
| X2 | 긴 출발지/도착지 | 텍스트가 렌더링되고 컴포넌트가 깨지지 않는다. |
| X3 | Avatar image 없음 | fallback 텍스트가 표시된다. |

## 수정/생성할 파일 경로

- `frontend/__tests__/RidyComponents.test.tsx`
- `frontend/__tests__/UiComponents.test.tsx`
- `frontend/__tests__/designTokens.test.ts`

## TDD 순서

1. Red: `UiComponents.test.tsx`에 기본 UI 컴포넌트 테스트 추가
2. Green: 기존 컴포넌트가 통과하는지 확인하고 필요한 최소 수정
3. Red: `RidyComponents.test.tsx`에 접근성/이벤트/엣지 케이스 추가
4. Green: 필요한 최소 수정
5. Red: `designTokens.test.ts`에 spacing/radius/input/button token 검증 추가
6. Green: token 정의와 테스트 통과 확인
7. Verify: 전체 검증 명령 실행

## 검증 명령

```bash
npm run test
npm run lint
npm run build
```

## 완료 조건

- 디자인 시스템 관련 테스트 케이스가 추가된다.
- 커스텀 Ridy 컴포넌트와 기본 UI 컴포넌트 테스트가 모두 통과한다.
- 기존 화면 테스트 회귀가 없다.
- `npm run test`, `npm run lint`, `npm run build`가 통과한다.

## 범위 제외

- 시각적 회귀 테스트 도입
- Storybook 도입
- 커버리지 임계값 CI 강제
- 컴포넌트 API 대규모 리팩토링
