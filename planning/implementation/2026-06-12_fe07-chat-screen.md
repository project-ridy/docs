# 채팅 화면 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/frontend#7` — `[FEAT] 채팅 화면`
- **목적**: 매칭 수락 후 생성된 채팅방 목록과 채팅방 메시지를 조회하고, 참여자가 실시간으로 메시지를 주고받을 수 있는 모바일 화면을 구현한다.
- **범위**: `/chat` 채팅 목록, `/chat/[id]` 채팅방, GraphQL 채팅방/메시지 조회, Socket.IO 메시지 송수신, 로딩/빈/에러 상태, MSW 기반 화면 테스트.
- **범위 제외**: 읽음 영속화, 타이핑 표시, 이미지 업로드 UI, 푸시 알림, 오프라인 재시도 큐, 무한 스크롤. 현재 백엔드 MVP가 제공하지 않는 `mark_read`/read receipt는 구현하지 않는다.

## 관련 이슈

- `project-ridy/frontend#7` — 채팅 화면
- 선행 완료: `project-ridy/backend#8` — 채팅 WebSocket 서버
- 선행 완료: `project-ridy/backend#9` — 채팅 API 테스트
- 후속: `project-ridy/frontend#8` — 홈/매칭/채팅 통합 테스트

## 관련 docs 문서

- `docs/design/DESIGN_SYSTEM.md`
- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/api/CHAT.md`
- `docs/planning/implementation/2026-06-12_be08-chat-websocket-server.md`
- `docs/planning/implementation/2026-06-12_be09-chat-api-tests.md`

## 아키텍처 접근

- GraphQL operation을 `src/graphql/operations/chat.graphql`에 먼저 추가하고 `npm run codegen`으로 generated 타입을 생성한다.
- API wrapper는 `lib/api/chat-api.ts`에 분리하고, 컴포넌트는 직접 `fetch`하지 않는다.
- TanStack Query hook은 `hooks/useChatQueries.ts`에 둔다.
- Socket.IO client 연결은 `hooks/useChatSocket.ts` 또는 채팅방 페이지 내부 effect로 최소 구현한다. 재사용성이 필요한 경우에만 훅으로 분리한다.
- 인증 토큰은 기존 `token-storage`에서 읽어 Socket.IO handshake `auth.token`으로 전달한다.
- 실시간 이벤트는 백엔드 구현 기준인 `/chat` namespace와 `chat:join`, `chat:leave`, `chat:message`, `chat:messageCreated` 이벤트를 사용한다.
- `docs/api/CHAT.md`에는 구버전 이벤트명(`join_room`, `send_message`)과 구버전 query명(`myChatRooms`, `chatRoom`)이 남아 있으므로, 구현 기준은 `docs/api/GRAPHQL_GATEWAY.md`와 `backend/src/graphql/schema.graphql`로 둔다.
- 화면은 기존 `app/matchings/*`와 동일하게 `AuthGuard`, `BottomNavigation`, shadcn/ui 조합, 디자인 토큰, 모바일 `max-w-md` 레이아웃을 유지한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| FE07-A01 | 채팅방 목록 조회 | `/chat`에서 최신 메시지, 경로, 상대/차주 정보, 안읽은 수를 표시한다. |
| FE07-A02 | 채팅방 없는 사용자 | 빈 상태와 매칭 탐색 CTA를 표시한다. |
| FE07-A03 | 채팅방 메시지 조회 | `/chat/[id]`에서 메시지 이력을 시간순으로 표시한다. |
| FE07-A04 | 채팅방 입장 | 화면 진입 시 `chat:join`을 보내고 나갈 때 `chat:leave`를 보낸다. |
| FE07-A05 | 메시지 전송 | 입력값을 trim해 `chat:message`로 전송하고 입력창을 비운다. |
| FE07-A06 | 새 메시지 수신 | `chat:messageCreated` 수신 시 메시지 목록에 append한다. |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| FE07-E01 | 채팅방 목록 GraphQL 실패 | 에러 카드와 다시 시도 버튼을 표시한다. |
| FE07-E02 | 메시지 이력 GraphQL 실패 | 에러 카드와 다시 시도 버튼을 표시한다. |
| FE07-E03 | 빈 메시지 전송 | 전송 버튼을 비활성화하고 WebSocket 이벤트를 보내지 않는다. |
| FE07-E04 | 1000자 초과 메시지 | 입력 안내/에러를 표시하고 전송하지 않는다. |
| FE07-E05 | WebSocket 연결 실패 | 상단에 연결 불안정 안내를 표시하고 GraphQL 이력 조회 화면은 유지한다. |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| FE07-X01 | 메시지 없는 채팅방 | 빈 대화 안내와 입력창을 표시한다. |
| FE07-X02 | `lastMessage`가 null인 방 | 마지막 메시지 대신 `아직 메시지가 없습니다`를 표시한다. |
| FE07-X03 | 빠른 중복 제출 | 전송 중인 입력은 중복 전송하지 않는다. |
| FE07-X04 | 직접 URL 진입 | 인증 후 `roomId` 기준으로 메시지를 조회하고 접근 실패 시 에러 상태를 표시한다. |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| FE07-A01 | `app/chat/page.tsx`, `hooks/useChatQueries.ts`, `lib/api/chat-api.ts`, `src/graphql/operations/chat.graphql` | 채팅방 목록 | `__tests__/Chat.test.tsx` | `채팅방 목록을 최신 메시지와 함께 표시한다` | TODO |
| FE07-A02 | `app/chat/page.tsx` | 빈 상태 | `__tests__/Chat.test.tsx` | `채팅방이 없으면 빈 상태를 표시한다` | TODO |
| FE07-A03 | `app/chat/[id]/page.tsx`, `hooks/useChatQueries.ts`, `lib/api/chat-api.ts` | 메시지 이력 | `__tests__/Chat.test.tsx` | `채팅방 메시지 이력을 표시한다` | TODO |
| FE07-A04 | `app/chat/[id]/page.tsx`, `hooks/useChatSocket.ts` | 입장/퇴장 이벤트 | `__tests__/Chat.test.tsx` | `채팅방 입장과 이탈 이벤트를 보낸다` | TODO |
| FE07-A05 | `app/chat/[id]/page.tsx`, `hooks/useChatSocket.ts` | 메시지 전송 | `__tests__/Chat.test.tsx` | `메시지를 입력하고 전송한다` | TODO |
| FE07-A06 | `app/chat/[id]/page.tsx`, `hooks/useChatSocket.ts` | 실시간 수신 append | `__tests__/Chat.test.tsx` | `새 메시지를 수신하면 대화에 추가한다` | TODO |
| FE07-E01 | `app/chat/page.tsx` | 목록 에러 | `__tests__/Chat.test.tsx` | `채팅방 목록 조회 실패 시 재시도 상태를 표시한다` | TODO |
| FE07-E02 | `app/chat/[id]/page.tsx` | 메시지 에러 | `__tests__/Chat.test.tsx` | `메시지 조회 실패 시 재시도 상태를 표시한다` | TODO |
| FE07-E03 | `app/chat/[id]/page.tsx` | 빈 메시지 방지 | `__tests__/Chat.test.tsx` | `빈 메시지는 전송하지 않는다` | TODO |
| FE07-E04 | `app/chat/[id]/page.tsx` | 길이 제한 | `__tests__/Chat.test.tsx` | `1000자를 초과하면 전송하지 않는다` | TODO |
| FE07-E05 | `hooks/useChatSocket.ts`, `app/chat/[id]/page.tsx` | 연결 실패 안내 | `__tests__/Chat.test.tsx` | `소켓 연결 실패 안내를 표시한다` | TODO |
| FE07-X01 | `app/chat/[id]/page.tsx` | 빈 대화 | `__tests__/Chat.test.tsx` | `메시지가 없으면 빈 대화 안내를 표시한다` | TODO |
| FE07-X02 | `app/chat/page.tsx` | null lastMessage | `__tests__/Chat.test.tsx` | `마지막 메시지가 없으면 기본 문구를 표시한다` | TODO |

## 수정/생성할 파일 경로

- `package.json`, `package-lock.json` — Socket.IO client 의존성(`socket.io-client`) 추가가 필요한 경우 반영
- `src/graphql/operations/chat.graphql` — `ChatRooms`, `Messages` operation 추가
- `src/graphql/generated/` — `npm run codegen` 산출물, 수동 수정 금지
- `lib/api/chat-api.ts` — GraphQL API wrapper
- `hooks/useChatQueries.ts` — 채팅방/메시지 query hook
- `hooks/useChatSocket.ts` — Socket.IO 연결, join/leave/message 이벤트 처리
- `app/chat/page.tsx` — 채팅 목록 화면
- `app/chat/[id]/page.tsx` — 채팅방 화면
- `__tests__/Chat.test.tsx` — 채팅 화면 테스트

## TDD 순서

1. `src/graphql/operations/chat.graphql`을 추가하고 `npm run codegen`을 실행해 타입 생성을 확인한다.
2. `__tests__/Chat.test.tsx`에 채팅 목록 정상/빈/에러 테스트를 먼저 작성하고 실패를 확인한다.
3. `lib/api/chat-api.ts`, `hooks/useChatQueries.ts`, `app/chat/page.tsx`를 최소 구현해 목록 테스트를 통과시킨다.
4. 메시지 이력 정상/빈/에러 테스트를 작성하고 실패를 확인한다.
5. `app/chat/[id]/page.tsx`를 최소 구현해 메시지 이력 테스트를 통과시킨다.
6. Socket.IO mock 기반 join/leave/send/receive/connection-error 테스트를 작성하고 실패를 확인한다.
7. `hooks/useChatSocket.ts` 또는 페이지 내부 effect를 구현해 실시간 이벤트 테스트를 통과시킨다.
8. 중복/빈/1000자 초과 전송 방지 테스트를 추가하고 통과시킨다.
9. `npm run test`, `npm run lint`, `npm run build`를 실행한다.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- `/chat`에서 사용자의 채팅방 목록, 빈 상태, 에러/재시도 상태가 표시된다.
- `/chat/[id]`에서 메시지 이력, 빈 대화, 에러/재시도 상태가 표시된다.
- 채팅방 진입/이탈 시 `chat:join`, `chat:leave`가 호출된다.
- 메시지 전송 시 `chat:message`가 호출되고, `chat:messageCreated` 수신 시 화면에 반영된다.
- 빈 메시지/1000자 초과/중복 제출을 클라이언트에서 방지한다.
- generated GraphQL 타입만 사용하고 손작성 API 타입을 만들지 않는다.
- 등록표의 모든 케이스가 구현/테스트와 연결된다.
- 검증 명령이 통과한다.
