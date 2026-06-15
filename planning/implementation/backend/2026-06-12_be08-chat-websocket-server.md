# 채팅 WebSocket 서버 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#8` — `[FEAT] 채팅 WebSocket 서버`
- **목적**: 매칭된 같은 회사 사용자끼리 채팅방에 입장하고, 메시지를 저장/조회/실시간 브로드캐스트할 수 있게 한다.
- **범위**: GraphQL 채팅 이력 조회, Socket.IO 기반 실시간 메시지 송수신, 회사/참여자 접근 제어, 메시지 유효성 검증.
- **범위 제외**: 읽음 영속화 테이블, 첨부 이미지 업로드, 푸시 알림, 운영용 Redis adapter. MVP에서는 단일 프로세스 in-memory connection 기준으로 구현한다.

## 관련 이슈

- `project-ridy/backend#8` — 채팅 WebSocket 서버
- 후속: `project-ridy/backend#9` — 채팅 API 테스트
- 후속: `project-ridy/frontend#7` — 채팅 화면

## 관련 docs 문서

- `docs/api/CHAT.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/architecture/DATABASE.md`
- `docs/architecture/MSA.md`

## 아키텍처 접근

- NestJS `@nestjs/websockets` + Socket.IO gateway를 `src/services/chat/` 아래에 추가한다.
- GraphQL schema-first 원칙에 따라 기존 `ChatRoom`, `Message`, `MessageConnection`, `chatRooms`, `messages` SDL을 유지하고 resolver/service를 구현한다.
- WebSocket 인증은 handshake `auth.token` 또는 `Authorization: Bearer <token>`에서 access token을 받아 검증한다.
- 회사 격리는 `ChatRoom -> Ride -> companyId`와 현재 사용자 `companyId` 비교로 처리한다.
- 참여자 검증은 채팅방의 ride driver 또는 accepted passenger만 입장/메시지 전송 가능하도록 제한한다.
- 메시지 저장은 Prisma `message.create`를 사용하고, 저장 후 같은 Socket.IO room에 `chat:messageCreated`를 브로드캐스트한다.
- MVP rate limit은 gateway 내부 per-socket sliding window로 3초 내 5회 초과를 거부한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE08-A01 | 인증 사용자가 본인 채팅방 목록 조회 | 최신 메시지 기준 채팅방 목록과 `lastMessage`, `unreadCount` 반환 |
| BE08-A02 | 인증 사용자가 채팅방 메시지 조회 | `createdAt` 오름차순 메시지 connection 반환 |
| BE08-A03 | 채팅방 참여자가 `chat:join` | socket이 `chat:<roomId>` room에 join되고 성공 ack 반환 |
| BE08-A04 | 참여자가 `chat:message` 텍스트 전송 | 메시지 DB 저장 후 `chat:messageCreated` 브로드캐스트 |
| BE08-A05 | 참여자가 `chat:leave` | socket이 room에서 leave되고 성공 ack 반환 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE08-E01 | 토큰 없음/만료/위조 | WebSocket 연결 거부, GraphQL은 인증 에러 |
| BE08-E02 | 타회사 채팅방 접근 | `FORBIDDEN: 접근 권한이 없습니다` |
| BE08-E03 | 비참여자 채팅방 접근 | `FORBIDDEN: 해당 채팅방의 참여자가 아닙니다` |
| BE08-E04 | 빈 메시지 | 저장/브로드캐스트 없이 `BAD_REQUEST` ack |
| BE08-E05 | 1000자 초과 메시지 | `BAD_REQUEST: 메시지는 1000자 이내여야 합니다` |
| BE08-E06 | 3초 내 5회 초과 메시지 | `TOO_MANY_REQUESTS: 잠시 후 다시 시도해주세요` |
| BE08-E07 | 완료/취소된 라이드 채팅 전송 | `BAD_REQUEST: 종료된 카풀에는 메시지를 보낼 수 없습니다` |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE08-X01 | 채팅방이 없는 사용자 | `chatRooms` 빈 배열 반환 |
| BE08-X02 | 메시지가 없는 채팅방 | `messages.nodes` 빈 배열 반환 |
| BE08-X03 | 상대방이 오프라인 | 메시지는 저장되고 접속 중인 socket에만 브로드캐스트, 재접속 시 GraphQL 이력 조회로 확인 가능 |
| BE08-X04 | 같은 사용자가 여러 탭 접속 | 모든 socket이 같은 room에 join 가능 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE08-A01 | `src/services/chat/chat.resolver.ts`, `chat.service.ts` | `chatRooms` 조회 | `src/services/chat/chat.service.spec.ts` | `내 채팅방 목록을 최신 메시지 기준으로 조회한다` | TODO |
| BE08-A02 | `src/services/chat/chat.resolver.ts`, `chat.service.ts` | `messages` 조회 | `src/services/chat/chat.service.spec.ts` | `채팅방 메시지를 페이지네이션으로 조회한다` | TODO |
| BE08-A03 | `src/services/chat/chat.gateway.ts` | `chat:join` | `src/services/chat/chat.gateway.spec.ts` | `참여자가 채팅방에 입장한다` | TODO |
| BE08-A04 | `src/services/chat/chat.gateway.ts`, `chat.service.ts` | `chat:message` 저장/브로드캐스트 | `src/services/chat/chat.gateway.spec.ts` | `메시지를 저장하고 room에 브로드캐스트한다` | TODO |
| BE08-A05 | `src/services/chat/chat.gateway.ts` | `chat:leave` | `src/services/chat/chat.gateway.spec.ts` | `참여자가 채팅방을 나간다` | TODO |
| BE08-E01 | `src/services/chat/chat.gateway.ts` | handshake 인증 | `src/services/chat/chat.gateway.spec.ts` | `토큰이 없으면 연결을 거부한다` | TODO |
| BE08-E02 | `src/services/chat/chat.service.ts` | 회사 격리 | `src/services/chat/chat.service.spec.ts` | `타회사 채팅방 접근을 거부한다` | TODO |
| BE08-E03 | `src/services/chat/chat.service.ts` | 참여자 검증 | `src/services/chat/chat.service.spec.ts` | `비참여자 채팅방 접근을 거부한다` | TODO |
| BE08-E04 | `src/services/chat/chat.gateway.ts` | 빈 메시지 검증 | `src/services/chat/chat.gateway.spec.ts` | `빈 메시지는 저장하지 않는다` | TODO |
| BE08-E05 | `src/services/chat/chat.gateway.ts` | 길이 검증 | `src/services/chat/chat.gateway.spec.ts` | `1000자를 초과하면 거부한다` | TODO |
| BE08-E06 | `src/services/chat/chat.gateway.ts` | rate limit | `src/services/chat/chat.gateway.spec.ts` | `3초 내 5회 초과 메시지를 거부한다` | TODO |
| BE08-E07 | `src/services/chat/chat.service.ts` | 종료 라이드 검증 | `src/services/chat/chat.service.spec.ts` | `종료된 카풀에는 메시지를 보낼 수 없다` | TODO |
| BE08-X01 | `src/services/chat/chat.service.ts` | 빈 채팅방 목록 | `src/services/chat/chat.service.spec.ts` | `채팅방이 없으면 빈 배열을 반환한다` | TODO |
| BE08-X02 | `src/services/chat/chat.service.ts` | 빈 메시지 목록 | `src/services/chat/chat.service.spec.ts` | `메시지가 없으면 빈 connection을 반환한다` | TODO |

## 수정/생성할 파일 경로

- `package.json` — WebSocket/Socket.IO 의존성 추가
- `src/app/app.module.ts` — WebSocket module import 필요 시 반영
- `src/services/chat/chat-service.module.ts` — resolver/service/gateway provider 등록
- `src/services/chat/chat.service.ts` — 채팅방/메시지 조회, 접근 검증, 메시지 저장
- `src/services/chat/chat.resolver.ts` — GraphQL `chatRooms`, `messages`
- `src/services/chat/chat.gateway.ts` — Socket.IO 이벤트 처리
- `src/services/chat/chat.gateway.spec.ts` — gateway 단위 테스트
- `src/services/chat/chat.service.spec.ts` — service 단위 테스트
- `src/graphql/generated/schema-types.ts` — `npm run codegen` 산출물, 수동 수정 금지

## TDD 순서

1. `chat.service.spec.ts`에 채팅방 목록/메시지 조회/접근 제어 실패 테스트를 먼저 작성한다.
2. `chat.gateway.spec.ts`에 join/message/leave와 인증/검증 실패 테스트를 먼저 작성한다.
3. 필요한 WebSocket 의존성을 추가한다.
4. `ChatService`를 최소 구현해 service 테스트를 통과시킨다.
5. `ChatGateway`를 최소 구현해 gateway 테스트를 통과시킨다.
6. `ChatResolver`를 추가하고 generated 타입 기준으로 GraphQL entrypoint를 연결한다.
7. `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`를 실행한다.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- `chatRooms`, `messages` GraphQL query가 generated 타입 기반으로 동작한다.
- `chat:join`, `chat:leave`, `chat:message`, `chat:messageCreated` WebSocket 이벤트가 동작한다.
- 타회사/비참여자 접근이 차단된다.
- 빈 메시지, 1000자 초과, rate limit, 종료된 라이드 메시지 전송이 거부된다.
- 메시지 저장 후 온라인 참여자에게 브로드캐스트되고, 오프라인 참여자는 GraphQL 이력 조회로 확인 가능하다.
- 등록표의 모든 케이스가 구현/테스트와 연결된다.
- 검증 명령이 통과한다.
