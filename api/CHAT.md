# Ridy — 채팅 API (WebSocket + GraphQL)

## 개요

Socket.IO 기반 실시간 채팅. 카풀 매칭 수락 시 채팅방 자동 생성.

---

## GraphQL 스키마 (이력 조회)

```graphql
type ChatRoom {
  id: ID!
  ride: Ride!
  lastMessage: Message
  unreadCount: Int!
  participants: [User!]!
  createdAt: String!
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

type ChatRoomListResult {
  rooms: [ChatRoom!]!
  total: Int!
}

type MessageListResult {
  messages: [Message!]!
  nextCursor: String
  hasMore: Boolean!
}

type Query {
  myChatRooms: ChatRoomListResult!
  chatMessages(roomId: ID!, cursor: String, limit: Int): MessageListResult!
}

type Mutation {
  markAsRead(roomId: ID!): Boolean!
}
```

---

## WebSocket 이벤트 (Socket.IO)

### 클라이언트 → 서버

| 이벤트 | 페이로드 | 설명 |
|--------|----------|------|
| `join_room` | `{ roomId }` | 채팅방 입장 |
| `leave_room` | `{ roomId }` | 채팅방 퇴장 |
| `send_message` | `{ roomId, content, type }` | 메시지 전송 |
| `typing` | `{ roomId }` | 타이핑 중 알림 |

### 서버 → 클라이언트

| 이벤트 | 페이로드 | 설명 |
|--------|----------|------|
| `new_message` | `{ id, senderId, content, type, createdAt }` | 새 메시지 |
| `user_typing` | `{ userId, roomId }` | 타이핑 알림 |
| `user_joined` | `{ userId, roomId }` | 참가 알림 |
| `user_left` | `{ userId, roomId }` | 퇴장 알림 |
| `error` | `{ code, message }` | 에러 |

---

## 기능: 채팅방 자동 생성

### 설명

차주가 탑승 요청 수락 시 자동으로 채팅방 생성. 시스템 메시지로 시작.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 첫 탑승자 수락 | ChatRoom 생성 + 시스템 메시지 "박준서님이 김도윤님의 탑승을 수락했습니다" |
| A2 | 추가 탑승자 수락 | 기존 채팅방에 참여자 추가 + 시스템 메시지 |

---

## 기능: 메시지 전송 (`send_message`)

### 설명

채팅방 참여자가 텍스트/이미지 메시지 전송.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 텍스트 메시지 | `{ roomId, content: "안녕하세요!", type: "TEXT" }` | DB 저장 + 방 참여자에게 브로드캐스트 |
| A2 | 이미지 메시지 | `{ roomId, content: "https://...", type: "IMAGE" }` | 이미지 URL 저장 + 브로드캐스트 |
| A3 | 시스템 메시지 | 내부 트리거 (매칭 수락 등) | type=SYSTEM, sender 없음 |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 방 미참여자 전송 | roomId에 속하지 않은 유저 | `FORBIDDEN: 채팅방 참여자만 메시지를 보낼 수 있습니다` |
| E2 | 빈 메시지 | content="" | `BAD_REQUEST: 메시지 내용을 입력해주세요` |
| E3 | 과도한 메시지 길이 | content 1000자 초과 | `BAD_REQUEST: 메시지는 1000자 이내로 입력해주세요` |
| E4 | 존재하지 않는 방 | roomId 무효 | `NOT_FOUND: 채팅방을 찾을 수 없습니다` |
| E5 | 인증 안 됨 | 소켓 연결 시 토큰 없음 | 연결 거부 |

### 엣지 케이스

| # | 케이스 | 설명 | 기대 결과 |
|---|--------|------|-----------|
| X1 | 연결 끊김 중 메시지 | 보내는 중 네트워크 단절 | 재연결 후 채팅 이력에서 확인 가능 |
| X2 | 스팸 방지 | 1초 내 5회 이상 전송 | `TOO_MANY_REQUESTS: 메시지를 너무 빠르게 보내고 있습니다` |
| X3 | 카풀 종료 후 채팅 | COMPLETED 카풀의 채팅방 | 읽기 전용 전환 (메시지 전송 불가, 이력 조회만 가능) |

---

## 기능: 채팅 이력 조회 (`chatMessages`)

### 설명

커서 기반 페이지네이션으로 이전 메시지 로딩.

### 정상 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| A1 | 최초 로딩 | cursor 없음, limit=30 | 최근 30개 메시지 + nextCursor |
| A2 | 추가 로딩 | cursor=마지막 메시지 ID | 다음 30개 메시지 |
| A3 | 메시지 없음 | 빈 채팅방 | `{ messages: [], hasMore: false }` |

### 예외 케이스

| # | 케이스 | 입력 | 기대 결과 |
|---|--------|------|-----------|
| E1 | 방 미참여자 조회 | roomId에 속하지 않은 유저 | `FORBIDDEN` |
| E2 | 존재하지 않는 방 | 무효 roomId | `NOT_FOUND` |

---

## 기능: 채팅방 목록 (`myChatRooms`)

### 설명

유저가 참여 중인 채팅방 목록. 마지막 메시지 + 안읽은 수 포함.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 채팅방 목록 조회 | 참여 중인 방 목록, 최근 메시지 순 정렬 |
| A2 | 채팅방 없음 | `{ rooms: [], total: 0 }` |

---

## 기능: 읽음 처리 (`markAsRead`)

### 설명

채팅방 입장 시 안읽은 메시지 읽음 처리.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 읽음 처리 | unreadCount 0으로 초기화, 상대방에게 읽음 알림 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 방 미참여자 | `FORBIDDEN` |

---

## 기능: 소켓 연결/인증

### 설명

Socket.IO 연결 시 JWT 토큰으로 인증.

### 정상 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| A1 | 유효한 토큰 연결 | 소켓 연결 성공, 유저 정보 바인딩 |

### 예외 케이스

| # | 케이스 | 기대 결과 |
|---|--------|-----------|
| E1 | 토큰 없음 | 연결 거부 |
| E2 | 만료된 토큰 | 연결 거부, 클라이언트에서 토큰 갱신 후 재시도 |
| E3 | 중복 연결 | 같은 유저의 기존 소켓 연결 해제 후 새 연결 허용 |
