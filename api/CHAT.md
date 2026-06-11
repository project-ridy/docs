# Ridy — 채팅 API

WebSocket 기반 실시간 채팅

## 연결

```
wss://api.ridy.dev/v1/chat/ws?token=<access_token>
```

## 이벤트

### 클라이언트 → 서버

| 이벤트 | 페이로드 | 설명 |
|---|---|---|
| `join_room` | `{ matchingId }` | 채팅방 입장 |
| `send_message` | `{ matchingId, content, type }` | 메시지 전송 |
| `typing` | `{ matchingId }` | 타이핑 중 |

### 서버 → 클라이언트

| 이벤트 | 페이로드 | 설명 |
|---|---|---|
| `new_message` | `{ id, senderId, content, type, createdAt }` | 새 메시지 |
| `user_typing` | `{ userId, matchingId }` | 타이핑 알림 |
| `user_joined` | `{ userId, matchingId }` | 참가 알림 |
| `user_left` | `{ userId, matchingId }` | 퇴장 알림 |

## 메시지 타입

| type | 설명 |
|---|---|
| text | 일반 텍스트 |
| image | 이미지 |
| location | 위치 공유 |
| system | 시스템 메시지 (매칭 수락 등) |

## 채팅 이력 조회

### GET /chat/rooms/:matchingId/messages?cursor=&limit=

**Response:**
```json
{
  "success": true,
  "data": {
    "messages": [
      {
        "id": "uuid",
        "senderId": "uuid",
        "senderName": "김도윤",
        "senderImage": "https://...",
        "content": "안녕하세요!",
        "type": "text",
        "createdAt": "2026-06-15T08:00:00Z"
      }
    ],
    "nextCursor": "uuid",
    "hasMore": true
  }
}
```

## 채팅방 목록

### GET /chat/rooms

**Response:**
```json
{
  "success": true,
  "data": {
    "rooms": [
      {
        "matchingId": "uuid",
        "lastMessage": { "content": "내일 뵙겠습니다!", "createdAt": "..." },
        "unreadCount": 2,
        "participants": [ { "id", "name", "profileImage" } ]
      }
    ]
  }
}
```
