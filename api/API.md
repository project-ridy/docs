# Ridy — API 스펙 개요

## 아키텍처

Ridy는 **기업 단위 폐쇄형 카풀 서비스**로, 외부 API는 **GraphQL Gateway + schema-first** 아키텍처를 사용한다.

- **GraphQL Gateway**: `POST https://api.ridy.dev/graphql`
- **Realtime Gateway**: `wss://api.ridy.dev/realtime`
- **스키마 SSoT**: `backend/src/graphql/schema.graphql`
- **Gateway API 계약**: [GRAPHQL_GATEWAY.md](./GRAPHQL_GATEWAY.md)
- **MSA 서비스 경계**: [MSA.md](../architecture/MSA.md)
- **타입 생성**: 양쪽 모두 `npm run codegen`으로 generated 타입 사용

GraphQL은 클라이언트용 API 조합 계층으로 사용한다. 내부 서비스 간 통신은 GraphQL이 아니라 gRPC/internal HTTP와 이벤트 메시지를 사용한다.

## 핵심 개념

### 회사 단위 격리

모든 데이터는 회사(Company) 단위로 격리된다.

- **초대 코드 가입**: 기업 관리자가 발급한 초대 코드로만 가입 가능
- **같은 회사만 매칭**: searchRides 시 자동으로 같은 companyId 필터
- **같은 회사만 채팅**: 채팅방 접근 시 companyId 검증
- **회사별 수수료 정책**: CompanyPlan(FREE/PRO/ENTERPRISE)에 따라 수수료 부담 주체 다름

### 회사 플랜

| 플랜 | 수수료 부담 | 기능 |
|------|------------|------|
| FREE | 사원 전액 부담 | 기본 매칭/채팅/정산 |
| PRO | 회사 50% 부담 | + 관리자 대시보드, ESG 리포트 |
| ENTERPRISE | 회사 100% 부담 | + 맞춤 도메인, 전담 지원 |

## 인증

모든 인증 필요 쿼리/뮤테이션은 Bearer Token (JWT) 필요.

```
Authorization: Bearer <access_token>
```

- 액세스 토큰: 만료 1시간
- 리프레시 토큰: 만료 7일
- 인증 불필요: `joinWithInviteCode`, `login`, `refreshToken`

## GraphQL 에러 처리

GraphQL 표준 에러 형식을 따른다. `extensions.code`로 비즈니스 에러 코드를 구분.

```json
{
  "errors": [
    {
      "message": "유효하지 않은 초대 코드입니다",
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
| `UNAUTHORIZED` | 401 | 권한 없음 (OAuth 토큰 무효 등) |
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
| 인증 | [AUTH.md](./AUTH.md) | 초대 코드 가입, 소셜 로그인, 관리자 초대 코드 관리 | `AuthPayload`, `User`, `Company`, `InviteCode` |
| 매칭 | [MATCHING.md](./MATCHING.md) | 같은 회사 사원 간 카풀 등록/검색/요청/수락 | `Ride`, `RideRequest`, `SearchRidesInput` |
| 채팅 | [CHAT.md](./CHAT.md) | 같은 회사 사원 간 실시간 메시지, Socket.IO | `ChatRoom`, `Message`, WebSocket 이벤트 |
| 정산 | [PAYMENT.md](./PAYMENT.md) | 요금 계산, 정산, 결제, 회사별 수수료 정책 | `Settlement`, `FareCalculation`, `CompanyPlan` |

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
