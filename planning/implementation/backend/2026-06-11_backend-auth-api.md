# 백엔드 인증 API 구현 계획

> **docs#44 이후 상태:** 이 계획서는 기존 “초대 코드 가입” 중심의 과거 계획서입니다. 구현 전 최신 SSoT인 `docs/api/AUTH.md` 기준으로 `companyEmail` 입력, 회사 이메일 도메인 검증, OAuth 이메일 도메인 일치 검증을 포함한 새 구현/테스트 Case ID 레지스트리로 재계획해야 합니다. 구식 계획 그대로 구현하면 `BLOCKED`입니다.

## 대상 이슈

- `project-ridy/backend#3` — `[FEAT] 인증 API 구현`

## 구현 범위

- `AuthServiceModule` 구성
- `AuthService` 로그인, 초대 코드 가입, refresh token 갱신
- `JwtTokenService` HS256 access/refresh token 발급 및 검증
- `SocialOAuthVerifier` 추상화 및 Kakao/Google provider verifier
- `AuthResolver` GraphQL mutation/query 연결
- `AuthGuard` GraphQL context 인증 보호 기반
- `AuthService`/`SocialOAuthVerifier` TDD 유닛 테스트

## 범위 제외 및 후속

- 운영 SMS provider(CoolSMS/NCP 등)의 실제 credential 연결은 배포 환경 secret 확정 후 `SmsSender` adapter만 교체
- refresh token 저장/폐기 목록은 보안 정책 확정 시 별도 DB 모델로 추가

## 검증

- `npm test -- --runInBand`
- `npm run lint`
- `npm run build`
- `npm run test:e2e`
