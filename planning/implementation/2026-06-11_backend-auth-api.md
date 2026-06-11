# 백엔드 인증 API 구현 계획

## 대상 이슈

- `project-ridy/backend#3` — `[FEAT] 인증 API 구현`

## 구현 범위

- `AuthServiceModule` 구성
- `AuthService` 로그인, 초대 코드 가입, refresh token 갱신
- `JwtTokenService` HS256 access/refresh token 발급 및 검증
- `SocialOAuthVerifier` 추상화 및 개발/테스트용 mock token verifier
- `AuthResolver` GraphQL mutation/query 연결
- `AuthGuard` GraphQL context 인증 보호 기반
- `AuthService` TDD 유닛 테스트

## 범위 제외 및 후속

- 실제 Kakao/Google/Apple API 호출 adapter는 외부 credential 확정 후 교체
- SMS 휴대폰 인증은 후속 이슈로 분리
- refresh token 저장/폐기 목록은 DB 모델 추가 시 별도 구현

## 검증

- `npm test -- --runInBand`
- `npm run lint`
- `npm run build`
- `npm run test:e2e`
