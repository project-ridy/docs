# Ridy — 채팅 API (WebSocket + GraphQL)

## 개요

Socket.IO 기반 실시간 채팅. 카풀 매칭 수락 시 채팅방 자동 생성. **같은 회사 사원끼리만** 채팅 가능.

---

## GraphQL 스키마 (이력 조회)

```graphql
type ChatRoom {
  id: ID!
  ride: Ride!
  companyId: String!
  lastMessage: Message
  unreadCount: Int!
  createdAt: String!
  messages: [Message!]!
}

type Message {
  id: ID!
  roomId: ID!
  sender: User!
  content: String!
  type: MessageType!
  createdAt: String!
}

enum MessageType {
  TEXT
  IMAGE
  SYSTEM
}

type Query {
  myChatRooms: [ChatRoom!]!
  chatRoom(id: ID!): ChatRoom!
  messages(roomId: ID!, cursor: String, limit: Int = 30): [Message!]!
}
```

## WebSocket 이벤트

```
연결: wss://api.ridy.dev/chat?token=<access_token>

# 네임스페이스: /chat
# Room: company:<companyId> (회사별 격리)

클라이언트 → 서버:
  join_room        { roomId }
  leave_room       { roomId }
  send_message     { roomId, content, type? }
  mark_read        { roomId, messageIds }

서버 → 클라이언트:
  new_message      { id, roomId, sender, content, type, createdAt }
  room_updated     { roomId, lastMessage, unreadCount }
  user_typing      { roomId, userId }
  message_read     { roomId, messageIds, readBy }
```

---

## 기능: 채팅방 접근

### 설명

카풀 매칭 수락 시 자동 생성. 같은 회사 사원만 접근 가능.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 방 입장 | join_room → 기존 메시지 수신 |
| A2 | 이력 조회 | messages 쿼리로 과거 메시지 30개 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 타회사 채팅방 | 다른 회사 채팅방 접근 | `FORBIDDEN: 접근 권한이 없습니다` |
| E2 | 비참여자 | 매칭되지 않은 사원 | `FORBIDDEN: 해당 채팅방의 참여자가 아닙니다` |
| E3 | 만료된 토큰 | WebSocket 연결 시 | 연결 거부, 4001 코드 |

---

## 기능: 메시지 전송 (`send_message`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 텍스트 메시지 | 방 내 모든 참여자에게 브로드캐스트 |
| A2 | 이미지 메시지 | type=IMAGE, content=URL |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 빈 메시지 | content="" | 무시 (전송 안 됨) |
| E2 | 과도한 길이 | content > 1000자 | `BAD_REQUEST: 메시지는 1000자 이내여야 합니다` |
| E3 | 스팸 방지 | 3초 내 5회 이상 전송 | `TOO_MANY_REQUESTS: 잠시 후 다시 시도해주세요` |
| E4 | 종료된 카풀 | ride status=COMPLETED | 읽기 전용, 메시지 전송 불가 |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 오프라인 참여자 | 상대방 오프라인 | 메시지 저장, 재접속 시 수신 |
| X2 | 동시 전송 | 2명이 동시에 메시지 | 순서 보장 (createdAt 기준) |

---

## 기능: 읽음 처리 (`mark_read`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 읽음 표시 | unreadCount 감소, 상대에게 message_read 이벤트 |

---

## 기능: 채팅방 목록 (`myChatRooms`)

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 전체 목록 | 참여 중인 채팅방 목록 (최신 메시지순) |
| A2 | 안읽은 카운트 | 각 방의 unreadCount 포함 |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 채팅방 없음 | 매칭 전 | 빈 배열 `[]` |
