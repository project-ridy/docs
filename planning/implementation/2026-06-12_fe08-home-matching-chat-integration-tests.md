# 홈/매칭/채팅 통합 테스트 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/frontend#8` — `[TEST] 홈/매칭/채팅 통합 테스트`
- **관련 기능 이슈**: `project-ridy/frontend#5`, `project-ridy/frontend#6`, `project-ridy/frontend#7`
- **목적**: 홈, 매칭 결과/상세, 채팅 목록/방 사이의 핵심 사용자 흐름이 MSW 기반 GraphQL 응답과 Next navigation mock에서 회귀 없이 이어지는지 검증한다.
- **범위**: 인증 후 홈 진입, 홈 검색 → 매칭 결과, 매칭 카드 → 상세 → 탑승 요청, 홈 하단 채팅 탭 → 채팅 목록 → 채팅방 → 메시지 전송, 바텀 내비게이션, 인증 가드, 주요 API 에러 상태.
- **범위 제외**: 실제 브라우저 Playwright E2E, 실제 Socket.IO 서버 연결, 차주 전용 홈 UI 구현. 현재 프론트에는 `/driver` 차주 홈 화면이 없으므로 `role=driver` 분기는 후속 구현 이슈 전까지 테스트 불가로 기록한다.

## 관련 docs 문서

- `docs/design/SCREENS.md`
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/planning/implementation/2026-06-12_fe05-home-graphql-integration.md`
- `docs/planning/implementation/2026-06-12_fe06-matchings-pages.md`
- `docs/planning/implementation/2026-06-12_fe07-chat-screen.md`

## 아키텍처 접근

- 새 테스트 파일 `__tests__/IntegrationFlow.test.tsx`를 추가한다.
- 테스트는 기존 `AuthFlow.test.tsx`, `Home.test.tsx`, `Matchings.test.tsx`, `Chat.test.tsx` 패턴을 따른다.
- 각 테스트는 `TestProviders`, `saveAuthTokens`, MSW `graphql.query/mutation`, `next/navigation` mock을 사용한다.
- 실제 라우터를 구성하지 않고, 화면 간 이동은 `router.push` 호출값 검증 후 해당 페이지 컴포넌트를 렌더링하는 방식으로 경로 커버리지를 확보한다.
- Socket.IO 서버는 연결하지 않고 `socket.io-client` mock으로 `chat:message` 호출까지만 검증한다. 실서버 왕복은 backend E2E와 후속 브라우저 E2E 범위다.
- 이미 개별 화면 테스트가 상세 상태를 검증하므로, 이 이슈는 중복 assertion을 최소화하고 화면 간 연결 지점과 핵심 CTA를 중심으로 검증한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| FE08-A01 | 인증 후 홈 화면 진입 | 토큰이 있으면 홈 헤더와 검색 UI가 표시된다. |
| FE08-A02 | 홈 → 매칭 검색 | 출발지/도착지/시간 입력 후 `/matchings?...`로 이동한다. |
| FE08-A03 | 매칭 결과 → 상세 → 탑승 요청 | 카드 선택 후 상세 경로로 이동하고 요청 메시지를 보낼 수 있다. |
| FE08-A04 | 홈 → 채팅 탭 → 채팅방 → 메시지 | 채팅 탭 이동, 채팅방 진입, 메시지 전송 이벤트가 이어진다. |
| FE08-A05 | 바텀 내비게이션 | 홈/검색/채팅/내 정보 탭이 각 경로로 이동한다. |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| FE08-E01 | 미인증 홈 접근 | `/login`으로 리다이렉트한다. |
| FE08-E02 | 매칭 검색 API 실패 | 매칭 결과 화면에서 재시도 에러 UI를 표시한다. |
| FE08-E03 | 채팅방 목록 API 실패 | 채팅 목록 화면에서 재시도 에러 UI를 표시한다. |

### X: 엣지/제약 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| FE08-X01 | 차주 홈 분기 | 현재 `/driver` 화면이 없어 BLOCKED로 기록하고 후속 구현 시 테스트한다. |
| FE08-X02 | 채팅방이 없는 사용자 | 채팅 목록 빈 상태를 표시한다. |

## 구현/테스트 케이스 등록표

| 케이스 ID | 테스트 파일 | 테스트 이름 | 구현/검증 대상 | 상태 |
|---|---|---|---|---|
| FE08-A01 | `__tests__/IntegrationFlow.test.tsx` | `인증 후 홈 화면에 진입한다` | `app/page.tsx`, `AuthGuard` | TODO |
| FE08-A02 | `__tests__/IntegrationFlow.test.tsx` | `홈에서 검색 조건을 입력해 매칭 결과로 이동한다` | `app/page.tsx` → `/matchings` push | TODO |
| FE08-A03 | `__tests__/IntegrationFlow.test.tsx` | `매칭 결과에서 상세로 이동해 탑승 요청을 보낸다` | `app/matchings/page.tsx`, `app/matchings/[id]/page.tsx` | TODO |
| FE08-A04 | `__tests__/IntegrationFlow.test.tsx` | `홈 채팅 탭에서 채팅방으로 이동해 메시지를 전송한다` | `app/page.tsx`, `app/chat/page.tsx`, `app/chat/[id]/page.tsx` | TODO |
| FE08-A05 | `__tests__/IntegrationFlow.test.tsx` | `바텀 내비게이션 4탭 이동 경로를 검증한다` | `BottomNavigation` 사용 화면 | TODO |
| FE08-E01 | `__tests__/IntegrationFlow.test.tsx` | `토큰이 없으면 홈 접근 시 로그인으로 이동한다` | `AuthGuard` | TODO |
| FE08-E02 | `__tests__/IntegrationFlow.test.tsx` | `매칭 검색 실패 시 에러 UI를 표시한다` | `app/matchings/page.tsx` | TODO |
| FE08-E03 | `__tests__/IntegrationFlow.test.tsx` | `채팅방 목록 실패 시 에러 UI를 표시한다` | `app/chat/page.tsx` | TODO |
| FE08-X01 | 없음 | 차주 홈 분기 | `/driver` 화면 미구현 | BLOCKED |
| FE08-X02 | `__tests__/IntegrationFlow.test.tsx` | `채팅방이 없으면 빈 상태를 표시한다` | `app/chat/page.tsx` | TODO |

## 수정/생성할 파일 경로

- `__tests__/IntegrationFlow.test.tsx` — 홈/매칭/채팅 통합 흐름 테스트
- 필요 시 기존 테스트 helper 중복 제거는 하지 않는다. 이번 이슈는 테스트 추가가 목적이며 대규모 테스트 유틸 리팩터링은 범위 제외한다.

## TDD 순서

1. `__tests__/IntegrationFlow.test.tsx`에 인증 후 홈 진입 테스트를 작성하고 실패/통과를 확인한다.
2. 홈 검색 → 매칭 결과 이동 테스트를 작성한다.
3. 매칭 결과 → 상세 → 탑승 요청 테스트를 작성한다.
4. 홈 채팅 탭 → 채팅 목록 → 채팅방 → 메시지 전송 테스트를 작성한다.
5. 바텀 내비게이션 경로 테스트를 작성한다.
6. 인증 가드와 API 에러 상태 테스트를 작성한다.
7. 차주 홈 분기는 `/driver` 구현 부재를 테스트 파일 주석이 아니라 계획서와 PR 본문에 BLOCKED로 기록한다.
8. `npm run test`, `npm run lint`, `npm run build`를 실행한다.

## 실행할 검증 명령

```bash
npm run test
npm run lint
npm run build
```

## 완료 조건

- `__tests__/IntegrationFlow.test.tsx`가 이슈 본문의 HIGH 우선순위 흐름을 모두 커버한다.
- 홈/매칭/채팅 개별 테스트와 신규 통합 테스트가 모두 PASS한다.
- 차주 홈 분기 미구현은 BLOCKED/후속으로 명확히 남긴다.
- 검증 명령이 통과한다.
