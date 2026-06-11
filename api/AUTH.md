# Ridy — 인증 API

## 소셜 로그인

### POST /auth/login

소셜 제공자의 토큰으로 로그인/회원가입 처리

**Request:**
```json
{
  "provider": "kakao" | "google" | "apple",
  "accessToken": "string"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "user": {
      "id": "uuid",
      "name": "김도윤",
      "email": "user@example.com",
      "profileImage": "https://...",
      "phoneVerified": false,
      "isNewUser": true
    }
  }
}
```

## 토큰 갱신

### POST /auth/refresh

**Request:**
```json
{
  "refreshToken": "eyJ..."
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

## 로그아웃

### POST /auth/logout

refreshToken 무효화

**Response:**
```json
{
  "success": true,
  "data": null
}
```

## 휴대폰 인증

### POST /auth/phone/send

인증번호 발송

**Request:**
```json
{
  "phone": "01012345678"
}
```

### POST /auth/phone/verify

인증번호 확인

**Request:**
```json
{
  "phone": "01012345678",
  "code": "123456"
}
```
