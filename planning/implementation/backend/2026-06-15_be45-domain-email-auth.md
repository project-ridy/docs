# BE #45: 가입 코드 회사 이메일 도메인 검증 구현 계획서

> **2026-06-15 기획 변경 이후 상태:** 이 계획서는 OAuth 간편로그인 기반으로 완료된 과거 계획서입니다. 최신 인증 SSoT는 OAuth를 제거하고 회사 이메일 인증코드 + 비밀번호 가입/로그인을 사용합니다. 이후 구현은 `2026-06-15_be50-email-password-auth.md`를 기준으로 해야 하며, 이 파일을 그대로 재사용하면 `BLOCKED`입니다.

## 목표

`project-ridy/backend#45`에서 docs#44 이후 최신 인증 SSoT를 backend에 반영한다. 가입 코드 기반 가입(`joinWithInviteCode`)은 가입 코드로 회사를 식별한 뒤 사용자가 입력한 `companyEmail`의 도메인이 `Company.domain`과 일치해야 하며, OAuth 프로필 이메일도 같은 회사 도메인과 일치해야 한다.

## 아키텍처 접근

- GraphQL schema-first 원칙에 따라 `src/graphql/schema.graphql`의 `JoinWithInviteCodeInput`에 `companyEmail: String!`를 먼저 추가한다.
- `npm run codegen`으로 `src/graphql/generated/schema-types.ts`를 갱신한 뒤 resolver/service 타입을 generated 타입 기준으로 맞춘다.
- 도메인 검증은 `AuthService.joinWithInviteCode` 내부에서 가입 코드 검증 이후, 유저 생성 이전에 수행한다.
- 회사 도메인 비교는 대소문자와 공백을 정규화한다.
- 신규 DB 컬럼은 추가하지 않는다. `Company.domain`은 이미 SSoT상 필수지만 현 backend Prisma schema에는 nullable이므로, 이 이슈에서는 런타임 검증으로 null/blank domain을 가입 불가 처리한다. Prisma schema의 `domain` 필수 전환 및 마이그레이션은 별도 DB 정렬 이슈로 분리한다.

## 관련 이슈

- `project-ridy/backend#45` — `[FEAT] 가입 코드 회사 이메일 도메인 검증 구현`
- `project-ridy/docs#46` — backend#45 구현 계획 작성

## 관련 docs 문서

- `docs/api/AUTH.md`
- `docs/api/API.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/architecture/DATABASE.md`
- `docs/architecture/ARCHITECTURE.md`
- `docs/WORKFLOW.md`

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 입력 | 기대 결과 |
|---|---|---|---|
| A1 | 카카오 정상 가입 | 유효한 가입 코드 + `companyEmail=user@acme.co.kr` + OAuth profile email `user@acme.co.kr` | AuthPayload 발급, user.companyId 설정, 가입 코드 사용 횟수 증가 |
| A2 | 구글 정상 가입 | 유효한 가입 코드 + 회사 이메일 + Google OAuth profile email 동일 도메인 | AuthPayload 발급 |
| A3 | 도메인 대소문자 차이 | `companyEmail=User@ACME.CO.KR`, `Company.domain=acme.co.kr` | 정규화 후 가입 성공 |

### E: 예외 케이스

| ID | 케이스 | 입력 | 기대 결과 |
|---|---|---|---|
| E1 | 잘못된 가입 코드 | 존재하지 않는 code | `NotFoundException: 유효하지 않은 가입 코드입니다` |
| E2 | 비활성 가입 코드 | `isActive=false` | `BadRequestException: 비활성화된 가입 코드입니다` |
| E3 | 만료된 가입 코드 | `expiresAt < now` | `BadRequestException: 만료된 가입 코드입니다` |
| E4 | 사용 한도 초과 | `currentUses >= maxUses` | `BadRequestException: 가입 코드 사용 한도가 초과되었습니다` |
| E5 | 회사 정원 초과 | `company._count.users >= maxMembers` | `BadRequestException: 회사 정원이 초과되었습니다` |
| E6 | 기존 가입 이메일 | 동일 email 유저 존재 | `ConflictException: 이미 가입된 계정입니다` |
| E7 | 빈 OAuth 토큰 | `oauthToken=""` | `UnauthorizedException: OAuth 토큰이 필요합니다` |
| E8 | 미지원 provider | `provider="naver"` | `BadRequestException: 지원하지 않는 로그인 방식입니다` |
| E9 | 회사 이메일 누락/blank | `companyEmail=""` 또는 공백 | `BadRequestException: 회사 이메일은 필수입니다` |
| E10 | 회사 이메일 형식 오류 | `companyEmail="not-email"` | `BadRequestException: 유효한 회사 이메일을 입력해주세요` |
| E11 | 회사 도메인 미설정 | `Company.domain=null` 또는 공백 | `BadRequestException: 회사 이메일 도메인이 설정되지 않았습니다` |
| E12 | 회사 이메일 도메인 불일치 | `companyEmail=user@other.com`, `Company.domain=acme.co.kr` | `ForbiddenException: 회사 이메일 도메인이 일치하지 않습니다` |
| E13 | OAuth 이메일 도메인 불일치 | `companyEmail=user@acme.co.kr`, OAuth profile email `user@gmail.com` | `ForbiddenException: OAuth 계정 이메일과 회사 이메일이 일치하지 않습니다` |

### X: 엣지 케이스

| ID | 케이스 | 설명 | 기대 결과 |
|---|---|---|---|
| X1 | 동시 가입 | 같은 가입 코드 `maxUses=1`로 동시 가입 | 기존 currentUses 검증 흐름 유지. 트랜잭션 내 검증/증가 수행 |
| X2 | 이메일 앞뒤 공백 | ` company@acme.co.kr ` | trim 후 처리 |
| X3 | 하위 도메인 | `user@dept.acme.co.kr` vs `acme.co.kr` | SSoT에 허용 규칙 없음. exact domain만 허용 |
| X4 | OAuth 이메일 대소문자 | OAuth profile email `User@ACME.CO.KR` | 정규화 후 도메인 비교 성공 |

## 구현/테스트 Case ID 레지스트리

| Case ID | A/E/X 링크 | 구현 파일/단위 | 테스트 파일/테스트명 | 완료 기준 |
|---|---|---|---|---|
| BE-DOMAIN-AUTH-001 | A1, A2, A3, X2, X4 | `src/graphql/schema.graphql` `JoinWithInviteCodeInput.companyEmail`; `src/services/auth/auth.service.ts` `joinWithInviteCode` 도메인 검증 | `src/services/auth/auth.service.spec.ts` / `가입 코드와 회사 이메일 도메인이 일치하면 가입 성공` | companyEmail 입력 가입 성공, generated 타입 갱신, user 생성 유지 |
| BE-DOMAIN-AUTH-002 | E9, E10 | `src/services/auth/auth.service.ts` `normalizeEmailOrThrow` | `src/services/auth/auth.service.spec.ts` / `회사 이메일이 비어 있거나 이메일 형식이 아니면 거절` | blank/invalid companyEmail이 BadRequestException |
| BE-DOMAIN-AUTH-003 | E11, E12, X3 | `src/services/auth/auth.service.ts` `assertCompanyEmailDomain` | `src/services/auth/auth.service.spec.ts` / `회사 이메일 도메인이 회사 도메인과 다르면 거절` | 회사 domain null/blank 또는 exact mismatch 거절 |
| BE-DOMAIN-AUTH-004 | E13 | `src/services/auth/auth.service.ts` `assertOAuthEmailMatchesCompanyEmail` | `src/services/auth/auth.service.spec.ts` / `OAuth 이메일 도메인이 회사 이메일과 다르면 거절` | OAuth email domain mismatch가 ForbiddenException |
| BE-DOMAIN-AUTH-005 | E1-E8, X1 | 기존 `assertInviteCodeUsable`, 기존 가입/로그인 흐름 | `src/services/auth/auth.service.spec.ts` 기존 가입/로그인 회귀 테스트 | 기존 가입 코드, 중복 가입, 로그인 테스트가 계속 통과 |

## 수정/생성할 파일 경로

- `backend/src/graphql/schema.graphql`
- `backend/src/graphql/generated/schema-types.ts` (`npm run codegen` 산출물)
- `backend/src/services/auth/auth.service.ts`
- `backend/src/services/auth/auth.service.spec.ts`

## TDD 순서

1. 🔴 Red: `auth.service.spec.ts`에 `companyEmail` 필수/도메인 검증 실패/성공 테스트를 먼저 추가한다.
2. 🔴 Red: `schema.graphql`에 `companyEmail`을 추가하기 전 타입 에러 또는 테스트 실패를 확인한다.
3. 🟢 Green: `schema.graphql` 수정 후 `npm run codegen` 실행.
4. 🟢 Green: `AuthService.joinWithInviteCode`에 이메일 정규화와 도메인 검증을 최소 구현한다.
5. 🔵 Refactor: 검증 헬퍼를 작은 private/function 단위로 정리한다.
6. ✅ Verify: `npm run test`, `npm run lint`, `npm run build` 실행.

## 실행할 검증 명령

```bash
npm run codegen
npm run test -- auth.service.spec.ts
npm run test
npm run lint
npm run build
```

## 완료 조건

- `JoinWithInviteCodeInput.companyEmail: String!`가 schema와 generated 타입에 반영된다.
- 모든 `BE-DOMAIN-AUTH-*` Case ID가 구현 파일/단위와 테스트 파일/테스트명에 연결된다.
- 회사 이메일 도메인과 OAuth 이메일 도메인 검증이 user 생성 전에 수행된다.
- docs에 없는 하위 도메인 허용, 회사 계약/B2B 구독, 관리자 대시보드 동작을 추가하지 않는다.
- backend PR 본문에 Case ID 구현/테스트 확인표를 포함한다.

## BLOCKED / 범위 제외

- Prisma `Company.domain String?` → `String` 전환과 마이그레이션은 별도 DB 정렬 이슈로 분리한다.
- `InviteCode` 내부 타입/테이블명을 `JoinCode`로 rename하는 작업은 범위 제외다.
- B2B 구독, 회사 부담 정산, 관리자 대시보드/ESG 리포트는 후속 확장으로 `BLOCKED`다.
