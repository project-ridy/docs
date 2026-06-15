# Backend 이메일 인증코드 기반 비밀번호 인증 구현 계획

## 목표

OAuth 간편로그인을 제거하고, 회사 이메일 인증코드 + 비밀번호 기반 가입/로그인을 backend GraphQL API로 구현한다. 최신 SSoT는 `docs/api/AUTH.md`와 `docs/architecture/DATABASE.md`다.

## 아키텍처 접근

- GraphQL schema-first 순서로 `src/graphql/schema.graphql`에 다음 mutation/input/type을 추가하고 기존 OAuth 기반 `joinWithInviteCode`/provider login 입력을 제거한다.
  - `requestCompanyEmailVerification(input: RequestCompanyEmailVerificationInput!): EmailVerificationChallenge!`
  - `completeEmailPasswordSignup(input: CompleteEmailPasswordSignupInput!): AuthPayload!`
  - `login(input: LoginInput!): AuthPayload!` (`companyEmail`, `password`)
- Prisma schema에 `User.passwordHash`, `User.emailVerifiedAt`, `EmailVerificationChallenge`를 추가한다.
- 인증코드는 원문 저장 금지. hash 저장 후 검증한다.
- 비밀번호는 강한 단방향 해시(argon2/bcrypt 등 repo 표준으로 결정)로 저장한다.
- 가입 완료 시 User 생성, invite code 사용 횟수 증가, challenge usedAt 설정을 transaction으로 처리한다.
- 이메일 발송은 `EmailSender` adapter로 분리하고 테스트에서는 fake adapter를 사용한다.

## 관련 이슈

- docs: project-ridy/docs#50
- backend 구현 이슈: project-ridy/backend#47
- backend 테스트 이슈: 생성 필요

## 관련 docs 문서

- `docs/WORKFLOW.md`
- `docs/api/AUTH.md`
- `docs/api/API.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/architecture/DATABASE.md`
- `docs/architecture/ARCHITECTURE.md`

## A/E/X 케이스 분석

### A — 정상 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| A-BE-EMAIL-AUTH-001 | 회사 이메일 인증코드 발송 | 유효한 inviteCode + `jane@company.com` | challenge 생성, email sender 호출 |
| A-BE-EMAIL-AUTH-002 | 인증코드 검증 + 가입 완료 | challengeId + valid code + strong password | User 생성, passwordHash 저장, emailVerifiedAt 설정, AuthPayload 발급 |
| A-BE-EMAIL-AUTH-003 | 이메일/비밀번호 로그인 | `jane@company.com` + password | AuthPayload 발급 |
| A-BE-EMAIL-AUTH-004 | 가입 코드/이메일 정규화 | ` abc123 `, `Jane@Company.COM` | 코드 uppercase, 이메일 lowercase 후 처리 |

### E — 예외 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| E-BE-EMAIL-AUTH-001 | 잘못된 가입 코드 | unknown code | NOT_FOUND |
| E-BE-EMAIL-AUTH-002 | 회사 이메일 도메인 불일치 | external email | FORBIDDEN |
| E-BE-EMAIL-AUTH-003 | 인증코드 재발송 제한 | 60초 이내 재요청 | TOO_MANY_REQUESTS |
| E-BE-EMAIL-AUTH-004 | 만료된 인증코드 | expired challenge | BAD_REQUEST |
| E-BE-EMAIL-AUTH-005 | 인증코드 불일치 | wrong code | BAD_REQUEST |
| E-BE-EMAIL-AUTH-006 | 약한 비밀번호 | policy fail | BAD_REQUEST |
| E-BE-EMAIL-AUTH-007 | 비밀번호 로그인 실패 | wrong password | UNAUTHORIZED |
| E-BE-EMAIL-AUTH-008 | 이미 가입된 이메일 | existing user | CONFLICT |

### X — 엣지 케이스

| Case | 설명 | 입력 | 기대 결과 |
|---|---|---|---|
| X-BE-EMAIL-AUTH-001 | 같은 challenge 동시 가입 완료 | parallel complete requests | 하나만 성공 |
| X-BE-EMAIL-AUTH-002 | maxUses=1 race | 두 유저 동시 가입 완료 | transaction으로 하나만 성공 |
| X-BE-EMAIL-AUTH-003 | 이전 challenge 재요청 | active challenge exists | 이전 challenge 무효화 또는 최신 challenge만 유효 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| BE-EMAIL-AUTH-001 | A-BE-EMAIL-AUTH-001, A-BE-EMAIL-AUTH-004, E-BE-EMAIL-AUTH-001, E-BE-EMAIL-AUTH-002, E-BE-EMAIL-AUTH-003 | `src/graphql/schema.graphql`; `src/services/auth/auth.resolver.ts`; `src/services/auth/auth.service.ts` — `requestCompanyEmailVerification` | `src/services/auth/auth.service.spec.ts` — `가입 코드와 회사 이메일이 유효하면 인증코드를 발송한다`; `도메인이 다르면 거절한다`; `재발송 제한을 적용한다` | challenge 생성, 이메일 발송 adapter 호출, 예외 코드 PASS |
| BE-EMAIL-AUTH-002 | A-BE-EMAIL-AUTH-002, E-BE-EMAIL-AUTH-004, E-BE-EMAIL-AUTH-005, E-BE-EMAIL-AUTH-006, E-BE-EMAIL-AUTH-008, X-BE-EMAIL-AUTH-001, X-BE-EMAIL-AUTH-002 | `src/services/auth/auth.service.ts`; `prisma/schema.prisma` — challenge 검증, passwordHash 저장, transaction 가입 | `src/services/auth/auth.service.spec.ts` — `인증코드와 강한 비밀번호로 가입을 완료한다`; `만료/불일치/약한 비밀번호를 거절한다`; `동시 완료는 하나만 성공한다` | User 생성, emailVerifiedAt, invite code count, usedAt transaction PASS |
| BE-EMAIL-AUTH-003 | A-BE-EMAIL-AUTH-003, E-BE-EMAIL-AUTH-007 | `src/services/auth/auth.service.ts`; `src/services/auth/auth.resolver.ts` — email/password login | `src/services/auth/auth.service.spec.ts` — `회사 이메일과 비밀번호로 로그인한다`; `비밀번호가 틀리면 거절한다` | AuthPayload 발급 및 실패 예외 PASS |
| BE-EMAIL-AUTH-004 | A-BE-EMAIL-AUTH-001, A-BE-EMAIL-AUTH-002 | `src/services/auth/email-sender.ts` 또는 adapter 위치 — 인증 이메일 발송 추상화 | `src/services/auth/email-sender.spec.ts` 또는 service fake test — `인증코드 이메일 payload를 구성한다` | 실제 provider 의존 없이 테스트 가능 |
| BE-EMAIL-AUTH-005 | A-BE-EMAIL-AUTH-001..003 | `src/graphql/generated/schema-types.ts` — codegen 산출물 | `npm run codegen`; `npm run test`; `npm run lint`; `npm run build`; `npx prisma validate`; `npx prisma generate` | generated 타입/Prisma 검증 PASS |

## 수정/생성할 파일 경로

- `src/graphql/schema.graphql`
- `src/services/auth/auth.resolver.ts`
- `src/services/auth/auth.service.ts`
- `src/services/auth/*.spec.ts`
- `src/services/auth/email-sender.ts` 또는 기존 adapter 위치
- `prisma/schema.prisma`
- `src/graphql/generated/schema-types.ts` (codegen 산출물)

## TDD 순서

1. Red — schema/service spec에 인증코드 발송 정상/도메인 불일치/재발송 제한 테스트 추가.
2. Green — schema/codegen 후 request mutation 최소 구현.
3. Red — 인증코드 검증 + 비밀번호 설정 가입 테스트 추가.
4. Green — challenge 검증, password hash, transaction 가입 구현.
5. Red — email/password login 정상/실패 테스트 추가.
6. Green — login 구현.
7. Refactor — email sender/password hasher/challenge repository 경계 정리.
8. Verify — 전체 검증 명령 실행.

## 실행할 검증 명령

```bash
npm run codegen
npx prisma validate
npx prisma generate
npm run test
npm run lint
npm run build
```

## 완료 조건

- OAuth provider/oauthToken 기반 가입/로그인이 schema와 구현에서 제거된다.
- 회사 이메일 인증코드 발송/검증/비밀번호 설정 가입이 동작한다.
- 회사 이메일 + 비밀번호 로그인이 동작한다.
- 인증코드 원문 및 비밀번호 원문이 저장되지 않는다.
- 모든 `BE-EMAIL-AUTH-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
