# Frontend 이메일 인증코드 기반 비밀번호 인증 구현 계획

## 목표

OAuth 간편로그인 버튼을 제거하고, S02 로그인/가입 화면을 회사 이메일 + 비밀번호 로그인 및 가입 코드 + 회사 이메일 인증코드 + 비밀번호 설정 가입 플로우로 전환한다.

## 아키텍처 접근

- `/login` 화면은 로그인 탭과 가입 탭을 제공한다.
- 로그인 탭은 `companyEmail`, `password`로 `login` mutation을 호출한다.
- 가입 탭은 `inviteCode`, `companyEmail` 입력 후 `requestCompanyEmailVerification`을 호출해 인증코드를 발송한다.
- 인증코드 발송 성공 후 `verificationCode`, `password`, `passwordConfirm` 입력을 노출한다.
- 가입 완료 시 `completeEmailPasswordSignup` mutation을 호출하고 AuthPayload를 저장한 뒤 `/profile/setup`으로 이동한다.
- GraphQL 타입은 손작성하지 않고 `npm run codegen` generated 타입을 사용한다.

## 관련 이슈

- docs: project-ridy/docs#50
- frontend 구현 이슈: 생성 필요
- frontend 테스트 이슈: 생성 필요

## 관련 docs 문서

- `docs/WORKFLOW.md`
- `docs/api/AUTH.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`

## A/E/X 케이스 분석

### A — 정상 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| A-FE-EMAIL-AUTH-001 | 기존 사용자 로그인 | 회사 이메일 + 비밀번호 | 토큰 저장, 홈 이동 |
| A-FE-EMAIL-AUTH-002 | 인증코드 발송 | 가입 코드 + 회사 이메일 | 인증코드 입력 단계 노출 |
| A-FE-EMAIL-AUTH-003 | 인증코드 + 비밀번호 가입 완료 | challengeId + code + password | 토큰 저장, `/profile/setup` 이동 |
| A-FE-EMAIL-AUTH-004 | 입력 정규화 | 가입 코드 공백/소문자, 이메일 공백 | 요청 변수 정규화 |

### E — 예외 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| E-FE-EMAIL-AUTH-001 | 로그인 이메일 누락 | empty email | inline error, 요청 없음 |
| E-FE-EMAIL-AUTH-002 | 로그인 비밀번호 누락 | empty password | inline error, 요청 없음 |
| E-FE-EMAIL-AUTH-003 | 가입 코드 누락 | empty inviteCode | inline error, 요청 없음 |
| E-FE-EMAIL-AUTH-004 | 회사 이메일 형식 오류 | invalid email | inline error, 요청 없음 |
| E-FE-EMAIL-AUTH-005 | 비밀번호 정책 오류 | weak password | inline error, complete 요청 없음 |
| E-FE-EMAIL-AUTH-006 | 비밀번호 확인 불일치 | password mismatch | inline error, complete 요청 없음 |
| E-FE-EMAIL-AUTH-007 | backend 도메인/코드/로그인 에러 | GraphQL error | inline error 표시 |

### X — 엣지 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| X-FE-EMAIL-AUTH-001 | 인증코드 재발송 제한 | resendAvailableAt 미래 | 재발송 버튼 disabled 또는 안내 |
| X-FE-EMAIL-AUTH-002 | pending 중 중복 클릭 | 연속 클릭 | 버튼 disabled, 중복 요청 방지 |
| X-FE-EMAIL-AUTH-003 | 인증코드 입력 공백 | ` 123456 ` | trim 후 요청 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| FE-EMAIL-AUTH-001 | A-FE-EMAIL-AUTH-001, E-FE-EMAIL-AUTH-001, E-FE-EMAIL-AUTH-002, E-FE-EMAIL-AUTH-007 | `components/auth/LoginForm.tsx`; `hooks/useAuthMutations.ts` — companyEmail/password login | `__tests__/AuthFlow.test.tsx` — `회사 이메일과 비밀번호로 로그인한다`; `로그인 필수값이 없으면 요청하지 않는다`; `로그인 에러를 표시한다` | 로그인 mutation 변수와 토큰 저장/라우팅 PASS |
| FE-EMAIL-AUTH-002 | A-FE-EMAIL-AUTH-002, A-FE-EMAIL-AUTH-004, E-FE-EMAIL-AUTH-003, E-FE-EMAIL-AUTH-004, X-FE-EMAIL-AUTH-001, X-FE-EMAIL-AUTH-002 | `components/auth/LoginForm.tsx`; `hooks/useAuthMutations.ts` — 인증코드 발송 단계 | `__tests__/AuthFlow.test.tsx` — `가입 코드와 회사 이메일로 인증코드를 요청한다`; `잘못된 가입 입력이면 요청하지 않는다`; `재발송 제한을 표시한다` | request mutation 변수/상태 전환/disabled PASS |
| FE-EMAIL-AUTH-003 | A-FE-EMAIL-AUTH-003, E-FE-EMAIL-AUTH-005, E-FE-EMAIL-AUTH-006, E-FE-EMAIL-AUTH-007, X-FE-EMAIL-AUTH-003 | `components/auth/LoginForm.tsx`; `hooks/useAuthMutations.ts` — 인증코드 검증 + 비밀번호 설정 가입 완료 | `__tests__/AuthFlow.test.tsx` — `인증코드와 비밀번호로 가입을 완료한다`; `약한 비밀번호와 확인 불일치를 막는다`; `인증코드 에러를 표시한다` | complete mutation 변수와 토큰 저장/프로필 이동 PASS |
| FE-EMAIL-AUTH-004 | A-FE-EMAIL-AUTH-001..003 | `src/graphql/operations/auth.graphql`; `src/graphql/generated/graphql.ts` — 새 operations와 generated 타입 | `npm run codegen`; `npm run test`; `npm run lint`; `npm run build` | 손작성 API 타입 없이 generated 타입 기준 PASS |
| FE-EMAIL-AUTH-005 | A-FE-EMAIL-AUTH-001..003 | `components/auth/LoginForm.tsx` — OAuth 버튼 제거 및 안내 문구 변경 | `__tests__/AuthFlow.test.tsx` — `OAuth 버튼 없이 이메일 로그인과 가입 폼을 노출한다` | 카카오/구글 버튼 미노출, 이메일 인증 UI 노출 PASS |

## 수정/생성할 파일 경로

- `components/auth/LoginForm.tsx`
- `hooks/useAuthMutations.ts`
- `lib/auth/auth-api.ts`
- `src/graphql/operations/auth.graphql`
- `src/graphql/generated/graphql.ts` (codegen 산출물)
- `__tests__/AuthFlow.test.tsx`

## TDD 순서

1. Red — OAuth 버튼 미노출, 로그인 탭, 가입 탭 UI 테스트 추가.
2. Red — login mutation 변수/validation 테스트 추가.
3. Green — email/password login UI와 hook 연결.
4. Red — 인증코드 발송 mutation/상태 전환 테스트 추가.
5. Green — request verification 구현.
6. Red — 인증코드 + 비밀번호 가입 완료 테스트 추가.
7. Green — complete signup 구현.
8. Refactor — form state와 validation helper 정리.
9. Verify — 전체 검증 및 visual QA.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- OAuth 간편로그인 버튼이 S02에서 제거된다.
- 기존 사용자는 회사 이메일 + 비밀번호로 로그인할 수 있다.
- 신규 사용자는 가입 코드 + 회사 이메일 인증코드 + 비밀번호 설정으로 가입할 수 있다.
- backend GraphQL 에러가 inline error로 표시된다.
- 모든 `FE-EMAIL-AUTH-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
- UI 변경 후 visual QA 결과가 PR 본문에 기록된다.
