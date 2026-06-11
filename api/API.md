# Ridy — API 스펙 개요

Base URL: `https://api.ridy.dev/v1`

## 인증

모든 API는 Bearer Token 인증 (JWT)

```
Authorization: Bearer <access_token>
```

## 공통 응답 형식

### 성공
```json
{
  "success": true,
  "data": { ... }
}
```

### 에러
```json
{
  "success": false,
  "error": {
    "code": "MATCH_NOT_FOUND",
    "message": "조건에 맞는 매칭을 찾을 수 없습니다."
  }
}
```

## API 그룹

| 그룹 | 문서 | 설명 |
|---|---|---|
| 인증 | [AUTH.md](./AUTH.md) | 소셜 로그인, 토큰 관리 |
| 매칭 | [MATCHING.md](./MATCHING.md) | 카풀 매칭 생성/조회/수락 |
| 채팅 | [CHAT.md](./CHAT.md) | 실시간 메시지, WebSocket |
| 정산 | [PAYMENT.md](./PAYMENT.md) | 비용 계산, 정산, 결제 |

## HTTP 상태 코드

| 코드 | 의미 |
|---|---|
| 200 | 성공 |
| 201 | 생성됨 |
| 400 | 잘못된 요청 |
| 401 | 인증 필요 |
| 403 | 권한 없음 |
| 404 | 리소스 없음 |
| 409 | 충돌 (중복 등) |
| 500 | 서버 에러 |
