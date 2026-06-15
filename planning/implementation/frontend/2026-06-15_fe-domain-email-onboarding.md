# 프론트엔드 회사 이메일 온보딩 구현 계획

> **2026-06-15 기획 변경 이후 상태:** 이 계획서는 OAuth 간편로그인 기반으로 완료된 과거 계획서입니다. 최신 인증 SSoT는 OAuth를 제거하고 회사 이메일 인증코드 + 비밀번호 가입/로그인을 사용합니다. 이후 구현은 `2026-06-15_fe50-email-password-auth.md`를 기준으로 해야 하며, 이 파일을 그대로 재사용하면 `BLOCKED`입니다.

## 목표

`backend#45`에서 `joinWithInviteCode` GraphQL 입력에 `companyEmail`이 필수로 추가되었으므로, 프론트엔드 로그인/가입 화면(S02)이 가입 코드와 회사 이메일을 함께 수집하고 `joinWithInviteCode` 변수에 전달하도록 구현한다.

이 계획은 `docs#44` 이후 정리된 “가입 코드 + 회사 이메일 도메인 검증 + OAuth 가입” SSoT를 프론트엔드 코드에 반영하기 위한 구현 계획서다.

## 아키텍처 접근

- S02 로그인/가입 화면은 가입 코드와 회사 이메일을 입력받은 뒤 소셜 로그인 버튼을 활성화한다.
- 프론트엔드는 회사 도메인의 실제 유효성 판정을 임의로 수행하지 않는다.
  - 최종 판정은 backend `joinWithInviteCode`가 수행한다.
  - 프론트엔드는 빈 값과 기본 이메일 형식만 field-level validation으로 처리한다.
- `companyEmail`은 GraphQL generated 타입에 포함된 `JoinWithInviteCodeInput.companyEmail`을 기준으로 전달한다.
- 직접 API 타입을 손작성하지 않고 `npm run codegen` 결과를 사용한다.
- 기존 OAuth 모킹 토큰 흐름은 유지하되, backend 요청 변수에 `companyEmail`을 추가한다.

## 관련 이슈

- docs: project-ridy/docs#48
- backend 선행 구현: project-ridy/backend#45, project-ridy/backend#46
- frontend 구현 이슈: project-ridy/frontend#35

## 관련 docs 문서

- `docs/WORKFLOW.md`
- `docs/AGENTS.md`
- `agents/protocol/AGENT_PROTOCOL.md`
- `docs/api/AUTH.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/architecture/ARCHITECTURE.md`

## A/E/X 케이스 분석

### A — 정상 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| A-FE-DOMAIN-001 | 가입 코드와 회사 이메일로 카카오 가입 | `ABC123`, `jane@company.com`, kakao | `joinWithInviteCode.input.companyEmail` 포함, 토큰 저장, `/profile/setup` 이동 |
| A-FE-DOMAIN-002 | 가입 코드와 회사 이메일로 구글 가입 | `ABC123`, `jane@company.com`, google | `joinWithInviteCode.input.companyEmail` 포함, 토큰 저장, `/profile/setup` 이동 |
| A-FE-DOMAIN-003 | 가입 코드 대문자 자동 변환 | `abc123`, `jane@company.com` | UI 값과 요청 값이 `ABC123`으로 정규화 |

### E — 예외 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| E-FE-DOMAIN-001 | 가입 코드 누락 | `""`, `jane@company.com` | 소셜 로그인 시 `초대 코드를 입력해주세요.` 표시, 요청 미발생 |
| E-FE-DOMAIN-002 | 회사 이메일 누락 | `ABC123`, `""` | 소셜 로그인 시 `회사 이메일을 입력해주세요.` 표시, 요청 미발생 |
| E-FE-DOMAIN-003 | 회사 이메일 형식 오류 | `ABC123`, `jane-company` | 소셜 로그인 시 `올바른 회사 이메일을 입력해주세요.` 표시, 요청 미발생 |
| E-FE-DOMAIN-004 | backend 도메인 불일치 에러 | `ABC123`, `jane@external.com` | backend 에러 메시지를 inline error로 표시 |

### X — 엣지 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| X-FE-DOMAIN-001 | 이메일 앞뒤 공백 | `ABC123`, ` jane@company.com ` | 요청 값은 trim된 `jane@company.com` |
| X-FE-DOMAIN-002 | 가입 코드 앞뒤 공백 + 소문자 | ` abc123 `, `jane@company.com` | 요청 값은 `ABC123` |
| X-FE-DOMAIN-003 | pending 중 중복 클릭 | 유효 입력 후 버튼 반복 클릭 | 버튼 disabled, 중복 요청 방지 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-DOMAIN-ONBOARD-001 | A-FE-DOMAIN-001, A-FE-DOMAIN-002 | `components/auth/LoginForm.tsx` — 회사 이메일 입력 필드 추가 및 provider submit 전달 | `__tests__/AuthFlow.test.tsx` — `로그인 화면은 초대 코드, 회사 이메일과 카카오/구글 로그인만 노출하고 Apple 로그인은 제거한다` | 회사 이메일 input이 접근 가능한 label로 노출되고 소셜 버튼은 유지 |
| FE-DOMAIN-ONBOARD-002 | E-FE-DOMAIN-001, E-FE-DOMAIN-002, E-FE-DOMAIN-003 | `components/auth/LoginForm.tsx` — 가입 코드/회사 이메일 field-level validation | `__tests__/AuthFlow.test.tsx` — `초대 코드 또는 회사 이메일 없이 소셜 로그인을 누르면 유효성 메시지를 보여준다`; `회사 이메일 형식이 올바르지 않으면 요청하지 않고 유효성 메시지를 보여준다` | 잘못된 입력에서는 GraphQL 요청 없이 inline error 표시 |
| FE-DOMAIN-ONBOARD-003 | A-FE-DOMAIN-001, A-FE-DOMAIN-003, X-FE-DOMAIN-001, X-FE-DOMAIN-002 | `hooks/useAuthMutations.ts` — `SocialJoinInput.companyEmail`; `joinWithInviteCode` 변수 전달 | `__tests__/AuthFlow.test.tsx` — `초대 코드와 회사 이메일로 카카오 로그인이 성공하면 companyEmail을 전달하고 프로필 설정으로 이동한다` | MSW에서 `input.companyEmail`과 `input.inviteCode` 확인 후 토큰 저장/라우팅 PASS |
| FE-DOMAIN-ONBOARD-004 | E-FE-DOMAIN-004 | `components/auth/LoginForm.tsx` — mutation error 메시지 표시 | `__tests__/AuthFlow.test.tsx` — `도메인 불일치 응답이면 backend 에러 메시지를 표시한다` | backend GraphQL 에러 메시지가 사용자에게 표시됨 |
| FE-DOMAIN-ONBOARD-005 | A-FE-DOMAIN-001, A-FE-DOMAIN-002 | `src/graphql/generated/graphql.ts` — codegen 산출물에 `companyEmail` 반영 | `npm run codegen`, `npm run test`, `npm run lint`, `npm run build` | 손작성 타입 없이 generated 타입 기준으로 빌드 PASS |

## 수정/생성할 파일 경로

### frontend

- `components/auth/LoginForm.tsx`
- `hooks/useAuthMutations.ts`
- `src/graphql/generated/graphql.ts` (codegen 산출물)
- `__tests__/AuthFlow.test.tsx`

### 변경하지 않는 파일/범위 제외

- backend resolver/service 재수정은 범위 제외 — 이미 backend#45에서 완료.
- `Company.domain` Prisma nullability 변경은 범위 제외 — 별도 DB 정렬 이슈에서 처리.
- 실제 OAuth SDK 연동은 범위 제외 — 기존 mock token 흐름 유지.
- 회사 도메인 사전 조회 API 또는 invite-code prevalidation API는 docs에 없으므로 추가하지 않는다.
- Apple 로그인 추가/복원은 범위 제외.

## TDD 순서

1. Red — `__tests__/AuthFlow.test.tsx`에 회사 이메일 필드 노출 및 validation 테스트를 먼저 추가한다.
2. Red — 성공 플로우 테스트에서 MSW handler가 `variables.input.companyEmail`을 검증하도록 변경한다.
3. Green — `LoginForm`에 회사 이메일 state/input/validation을 추가한다.
4. Green — `SocialJoinInput`과 `useSocialJoinMutation`에 `companyEmail` 전달을 추가한다.
5. Green — backend 최신 schema에 맞춰 `npm run codegen`을 실행해 generated 타입을 갱신한다.
6. Refactor — validation helper가 필요하면 `LoginForm` 내부 private function 수준으로만 정리한다.
7. Verify — 전체 검증 명령을 실행한다.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- S02 로그인 화면에 가입 코드와 회사 이메일 입력이 모두 존재한다.
- 가입 코드/회사 이메일 누락 및 이메일 형식 오류가 요청 전에 inline error로 표시된다.
- `joinWithInviteCode` 요청 변수에 `companyEmail`이 포함된다.
- backend 도메인 불일치 등 GraphQL 에러 메시지가 inline error로 표시된다.
- 모든 `FE-DOMAIN-ONBOARD-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build` 결과를 PR 본문에 기록한다.
