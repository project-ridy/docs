# Ridy — API 스펙 개요

## 아키텍처

Ridy는 **GraphQL schema-first** 아키텍처를 사용한다.

- **GraphQL 엔드포인트**: `POST https://api.ridy.dev/graphql`
- **WebSocket (채팅)**: `wss://api.ridy.dev/chat?token=<access_token>`
- **스키마 SSoT**: `backend/src/graphql/schema.graphql`
- **타입 생성**: 양쪽 모두 `npm run codegen`으로 generated 타입 사용

## 인증

모든 인증 필요 쿼리/뮤테이션은 Bearer Token (JWT) 필요.

```
Authorization: Bearer <access_token>
```

- 액세스 토큰: 만료 1시간
- 리프레시 토큰: 만료 7일
- 인증 불필요: `login`, `refreshToken`

## GraphQL 에러 처리

GraphQL 표준 에러 형식을 따른다. `extensions.code`로 비즈니스 에러 코드를 구분.

```json
{
  "errors": [
    {
      "message": "OAuth 토큰이 필요합니다",
      "extensions": {
        "code": "UNAUTHORIZED"
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
| `FORBIDDEN` | 403 | 리소스 접근 권한 없음 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `BAD_REQUEST` | 400 | 입력값 오류 / 비즈니스 룰 위반 |
| `CONFLICT` | 409 | 중복 요청 / 상태 충돌 |
| `PAYMENT_FAILED` | 402 | 결제 거절 |
| `TOO_MANY_REQUESTS` | 429 | 요청 과다 (스팸 방지) |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

## API 그룹

| 그룹 | 문서 | 설명 | 주요 타입 |
|------|------|------|-----------|
| 인증 | [AUTH.md](./AUTH.md) | 소셜 로그인, 토큰 관리, 프로필 | `AuthPayload`, `User`, `LoginInput` |
| 매칭 | [MATCHING.md](./MATCHING.md) | 카풀 등록/검색/요청/수락 | `Ride`, `RideRequest`, `SearchRidesInput` |
| 채팅 | [CHAT.md](./CHAT.md) | 실시간 메시지, Socket.IO | `ChatRoom`, `Message`, WebSocket 이벤트 |
| 정산 | [PAYMENT.md](./PAYMENT.md) | 요금 계산, 정산, 결제 | `Settlement`, `FareCalculation`, `PaymentMethod` |

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
