# 채팅 API 테스트 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#9` — `[TEST] 채팅 API 테스트`
- **관련 기능 이슈**: `project-ridy/backend#8` — 채팅 WebSocket 서버
- **목적**: 채팅 GraphQL 이력 조회와 Socket.IO 실시간 이벤트가 인증, 권한, 저장, 브로드캐스트, 예외 조건에서 회귀 없이 동작하는지 검증한다.

## 관련 docs 문서

- `docs/planning/implementation/2026-06-12_be08-chat-websocket-server.md`
- `docs/api/CHAT.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/architecture/DATABASE.md`

## 테스트 범위

- 단위 테스트: `ChatService`, `ChatGateway`
- GraphQL E2E: `chatRooms`, `messages`
- Socket.IO E2E: 인증, `chat:join`, `chat:message`, `chat:messageCreated`, `chat:leave`
- 매칭 연계 회귀: `acceptRideRequest` 후 `ChatRoom` 생성

## 범위 제외

- 읽음 처리 영속화: 현재 DB에 read receipt 모델이 없으므로 이슈 본문의 `읽음 처리`는 후속 스키마 설계 전까지 제외한다.
- 실제 멀티 인스턴스 브로드캐스트: Redis adapter 도입 전까지 단일 프로세스 E2E만 검증한다.
- 이미지 업로드: `IMAGE` 타입 URL 저장 검증은 단위 테스트 수준으로 제한한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE09-A01 | 요청 수락 후 채팅방 생성 | `acceptRideRequest` 후 `chat_rooms`에 ride별 방 1개 생성 |
| BE09-A02 | GraphQL 채팅방 목록 조회 | 참여자의 `chatRooms`가 방과 `lastMessage`를 반환 |
| BE09-A03 | GraphQL 메시지 이력 조회 | `messages(roomId)`가 페이지네이션 connection 반환 |
| BE09-A04 | Socket.IO 방 입장/퇴장 | `chat:join`, `chat:leave` ack 성공 |
| BE09-A05 | Socket.IO 메시지 전송/수신 | 저장된 메시지를 같은 room 참여자에게 `chat:messageCreated`로 전달 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE09-E01 | WebSocket 토큰 없음 | 연결 거부 |
| BE09-E02 | GraphQL 미인증 | 인증 에러 |
| BE09-E03 | 비참여자 방 입장 | 권한 에러 ack 또는 exception |
| BE09-E04 | 빈 메시지 전송 | 저장 없이 BadRequest |
| BE09-E05 | 1000자 초과 메시지 | BadRequest |
| BE09-E06 | rate limit 초과 | 429 예외 |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE09-X01 | 채팅방 없음 | `chatRooms` 빈 배열 |
| BE09-X02 | 메시지 없음 | `messages.nodes` 빈 배열 |
| BE09-X03 | 오프라인 상대 | 메시지 저장 후 GraphQL 이력 조회로 확인 가능 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 테스트 파일 | 테스트 이름 | 구현/검증 대상 | 상태 |
|---|---|---|---|---|
| BE09-A01 | `src/services/matching/matching.service.spec.ts` | `차주가 요청을 수락하면 좌석을 감소시킨다` | `chatRoom.upsert` 회귀 | DONE |
| BE09-A02 | `test/chat.e2e-spec.ts` | `참여자의 채팅방 목록을 조회한다` | GraphQL `chatRooms` | TODO |
| BE09-A03 | `test/chat.e2e-spec.ts` | `채팅방 메시지 이력을 조회한다` | GraphQL `messages` | TODO |
| BE09-A04 | `test/chat.e2e-spec.ts` | `참여자가 채팅방에 입장하고 나간다` | Socket.IO join/leave | TODO |
| BE09-A05 | `test/chat.e2e-spec.ts` | `메시지를 저장하고 같은 방 참여자에게 브로드캐스트한다` | Socket.IO messageCreated | TODO |
| BE09-E01 | `src/services/chat/chat.gateway.spec.ts` | `토큰이 없으면 연결을 거부한다` | WebSocket 인증 | DONE |
| BE09-E02 | `test/chat.e2e-spec.ts` | `인증 없이는 채팅방을 조회할 수 없다` | GraphQL auth | TODO |
| BE09-E03 | `test/chat.e2e-spec.ts` | `비참여자는 채팅방에 입장할 수 없다` | 참여자 검증 | TODO |
| BE09-E04 | `src/services/chat/chat.gateway.spec.ts` | `빈 메시지는 저장하지 않는다` | 빈 메시지 검증 | DONE |
| BE09-E05 | `src/services/chat/chat.gateway.spec.ts` | `1000자를 초과하면 거부한다` | 길이 검증 | DONE |
| BE09-E06 | `src/services/chat/chat.gateway.spec.ts` | `3초 내 5회 초과 메시지를 거부한다` | rate limit | DONE |
| BE09-X01 | `src/services/chat/chat.service.spec.ts` | `채팅방이 없으면 빈 배열을 반환한다` | 빈 목록 | DONE |
| BE09-X02 | `src/services/chat/chat.service.spec.ts` | `메시지가 없으면 빈 connection을 반환한다` | 빈 이력 | DONE |
| BE09-X03 | `test/chat.e2e-spec.ts` | `오프라인 상대도 저장된 메시지를 이력으로 조회한다` | 저장 후 이력 조회 | TODO |

## 수정/생성할 파일 경로

- `package.json`, `package-lock.json` — Socket.IO E2E client가 필요하면 `socket.io-client` dev dependency 추가
- `test/chat.e2e-spec.ts` — GraphQL + Socket.IO E2E 테스트
- `src/services/chat/chat.gateway.spec.ts` — 필요 시 ack/error 테스트 보강
- `src/services/chat/chat.service.spec.ts` — 필요 시 GraphQL 매핑 보강
- `src/services/matching/matching.service.spec.ts` — 채팅방 생성 회귀 검증 유지

## TDD 순서

1. `test/chat.e2e-spec.ts`에 GraphQL 인증 실패/채팅방 목록/메시지 조회 테스트를 작성한다.
2. Socket.IO client 기반 join/leave/messageCreated E2E 테스트를 작성한다.
3. 실패 원인이 테스트 인프라 의존성이라면 `socket.io-client`를 dev dependency로 추가한다.
4. 구현 수정이 필요한 경우 최소 변경만 적용한다.
5. `npm run test:e2e`, `npm run test`, `npm run lint`, `npm run build`를 실행한다.

## 실행할 검증 명령

```bash
npm run test:e2e
npm run test
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- 등록표의 TODO E2E 케이스가 모두 PASS로 전환된다.
- `ChatService`/`ChatGateway` 기존 단위 테스트가 회귀 없이 통과한다.
- GraphQL 채팅 query와 Socket.IO 실시간 이벤트가 같은 테스트 데이터에서 동작한다.
- `npm run test:e2e`, `npm run test`, `npm run lint`, `npm run build`가 통과한다.
