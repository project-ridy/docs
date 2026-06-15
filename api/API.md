# Ridy — API 스펙 개요

## 아키텍처

Ridy는 **회사 이메일 도메인 기반 폐쇄형 카풀 서비스**로, 외부 API는 **GraphQL Gateway + schema-first** 아키텍처를 사용한다.

- **GraphQL Gateway**: `POST https://api.ridy.dev/graphql`
- **Realtime Gateway**: `wss://api.ridy.dev/realtime`
- **스키마 SSoT**: `backend/src/graphql/schema.graphql`
- **Gateway API 계약**: [GRAPHQL_GATEWAY.md](./GRAPHQL_GATEWAY.md)
- **MSA 서비스 경계**: [MSA.md](../architecture/MSA.md)
- **타입 생성**: 양쪽 모두 `npm run codegen`으로 generated 타입 사용

GraphQL은 클라이언트용 API 조합 계층으로 사용한다. 내부 서비스 간 통신은 GraphQL이 아니라 gRPC/internal HTTP와 이벤트 메시지를 사용한다.

## 핵심 개념

### 회사 이메일 도메인 단위 격리

모든 데이터는 회사(Company)와 회사 이메일 도메인 단위로 격리된다.

- **가입 코드 + 회사 이메일 인증 가입**: 가입 코드 입력 후 회사 이메일로 인증코드를 발송하고, 인증 성공 시 비밀번호를 설정해 가입
- **같은 회사 도메인만 매칭**: `searchRides` 시 자동으로 같은 `companyId`/`domain` 필터
- **같은 회사 도메인만 채팅**: 채팅방 접근 시 `companyId`와 도메인 검증
- **MVP 수수료 정책**: 탑승자가 플랫폼 수수료를 부담한다. 회사 부담 정산은 후속 B2B 확장 전까지 `BLOCKED`

### 회사 플랜 — 후속 확장

`CompanyPlan(FREE/PRO/ENTERPRISE)`은 후속 B2B 계약/구독 확장을 위한 예약 개념이다. MVP 구현에서는 플랜별 기능 차등이나 회사 부담 정산을 사용하지 않는다.

| 플랜 | 수수료 부담 | 기능 |
|------|------------|------|
| FREE | 미정 | 기본 매칭/채팅/정산 후보 |
| PRO | BLOCKED | 후속 관리자 대시보드, ESG 리포트 후보 |
| ENTERPRISE | BLOCKED | 맞춤 웹 도메인, 전담 지원 후보 |

## 인증

모든 인증 필요 쿼리/뮤테이션은 Bearer Token (JWT) 필요.

```
Authorization: Bearer <access_token>
```

- 액세스 토큰: 만료 1시간
- 리프레시 토큰: 만료 7일
- 인증 불필요: `requestCompanyEmailVerification`, `completeEmailPasswordSignup`, `login`, `refreshToken`

## GraphQL 에러 처리

GraphQL 표준 에러 형식을 따른다. `extensions.code`로 비즈니스 에러 코드를 구분.

```json
{
  "errors": [
    {
      "message": "유효하지 않은 가입 코드입니다",
      "extensions": {
        "code": "NOT_FOUND"
      }
    }
  ],
  "data": null
}
```

### 에러 코드

| 코드 | HTTP 유사 | 설명 |
|------|-----------|------|
| `UNAUTHENTICATED` | 401 | 인증 필요 / 토큰 만료 |
| `UNAUTHORIZED` | 401 | 권한 없음 (비밀번호 불일치 등) |
| `FORBIDDEN` | 403 | 리소스 접근 권한 없음 (타회사 접근 등) |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `BAD_REQUEST` | 400 | 입력값 오류 / 비즈니스 룰 위반 |
| `CONFLICT` | 409 | 중복 요청 / 상태 충돌 |
| `PAYMENT_FAILED` | 402 | 결제 거절 |
| `TOO_MANY_REQUESTS` | 429 | 요청 과다 (스팸 방지) |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

## API 그룹

| 그룹 | 문서 | 설명 | 주요 타입 |
|------|------|------|-----------|
| Gateway | [GRAPHQL_GATEWAY.md](./GRAPHQL_GATEWAY.md) | GraphQL Gateway 공통 계약, context, 공통 SDL, 보안/성능 제한 | `Query`, `Mutation`, `PageInfo`, directives |
| 인증 | [AUTH.md](./AUTH.md) | 가입 코드, 회사 이메일 인증코드, 비밀번호 로그인, 운영자 가입 코드 관리 | `AuthPayload`, `User`, `Company`, `InviteCode`, `EmailVerificationChallenge` |
| 매칭 | [MATCHING.md](./MATCHING.md) | 같은 회사 이메일 도메인 구성원 간 카풀 등록/검색/요청/수락 | `Ride`, `RideRequest`, `SearchRidesInput` |
| 채팅 | [CHAT.md](./CHAT.md) | 같은 회사 이메일 도메인 구성원 간 실시간 메시지, Socket.IO | `ChatRoom`, `Message`, WebSocket 이벤트 |
| 정산 | [PAYMENT.md](./PAYMENT.md) | 요금 계산, 정산, 결제, MVP 탑승자 수수료 정책 | `Settlement`, `FareCalculation` |

## 케이스 네이밍 컨벤션

각 API 문서의 테스트 케이스는 다음 규칙으로 분류:

| 접두어 | 의미 | 설명 |
|--------|------|------|
| **A** | 정상 (Accept) | happy path, 기대 동작 |
| **E** | 예외 (Error) | 입력 오류, 인증 실패, 비즈니스 룰 위반 |
| **X** | 엣지 (eXtreme) | 동시성, 경계값, 특수 상황 |

> 예: `E3` = 3번째 예외 케이스, `X1` = 1번째 엣지 케이스

## Codegen 워크플로우

스키마 변경 시 양쪽 모두 codegen을 실행해야 한다.

```bash
# 1. 스키마 수정 (SSoT)
vim backend/src/graphql/schema.graphql

# 2. 백엔드 코드젠 (typescript-resolvers)
cd backend && npm run codegen

# 3. 프론트엔드 코드젠 (client-preset)
cd frontend && npm run codegen

# 4. 생성된 타입 사용
# 백엔드: import { Resolvers } from './graphql/generated/schema-types'
# 프론트엔드: import { useHealthQuery } from './graphql/generated'
```

**규칙:**
- `src/graphql/generated/` 폴더는 수동 수정 금지 (codegen 산출물)
- 손작성 API 타입 금지 — 반드시 generated 타입 사용
- generated 폴더는 lint/TS 검사에서 제외
