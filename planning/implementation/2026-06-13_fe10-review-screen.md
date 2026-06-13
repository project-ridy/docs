# FE10 평점/리뷰 화면 구현 계획

## 목표

- `frontend#10` 평점/리뷰 화면을 구현한다.
- 사용자가 완료된 운행 후 `/reviews/:matchingId`에서 상대방에게 별점과 코멘트를 남길 수 있게 한다.
- 같은 화면에서 운행 리뷰 목록과 받은 리뷰 목록을 확인할 수 있게 한다.

## 관련 이슈

- Frontend: `project-ridy/frontend#10` `[FEAT] 평점/리뷰 화면`
- Backend 선행: `project-ridy/backend#11` 평점/리뷰 API
- Backend 선행: `project-ridy/backend#12` 정산/평점 API 테스트

## 관련 문서

- `docs/planning/implementation/2026-06-12_be11-review-api.md`
- `docs/planning/implementation/2026-06-12_be12-settlement-review-api-tests.md`
- `docs/design/SCREENS.md` S12 평점/리뷰
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `backend/src/graphql/schema.graphql`

## 현재 API 기준

별도 `docs/api/REVIEW.md`는 없다. Frontend 구현은 backend 실제 GraphQL SDL을 우선한다.

```graphql
query RideReviews($rideId: ID!) {
  rideReviews(rideId: $rideId) { ... }
}

query UserReviews($userId: ID!, $pagination: PaginationInput) {
  userReviews(userId: $userId, pagination: $pagination) { ... }
}

mutation CreateReview($input: CreateReviewInput!) {
  createReview(input: $input) { ... }
}
```

`/reviews/:matchingId`의 `matchingId`는 현재 backend `Ride.id`로 취급한다. 별도 `Matching` GraphQL 타입은 없다.

## 아키텍처 접근

- GraphQL operation을 `src/graphql/operations/review.graphql`에 먼저 추가한다.
- `npm run codegen`으로 generated 타입과 document를 생성한다.
- API wrapper는 `lib/api/review-api.ts`에 둔다.
- TanStack Query hook은 `hooks/useReviewQueries.ts`에 둔다.
- 화면은 `app/reviews/[matchingId]/page.tsx`에 구현한다.
- 대상자 선택은 backend 스키마상 `toUserId`가 필수이므로, 화면에서 `상대방 ID` 입력을 제공한다. 추후 ride 상세/요청 데이터에서 자동 결정하는 개선은 후속으로 둔다.
- 빠른 리뷰 태그는 backend 별도 필드가 없으므로 선택한 태그를 코멘트 앞부분에 합쳐 전송한다.
- 코멘트 입력 UI 제한은 이슈 요구에 맞춰 200자로 제한한다. backend는 최대 1000자까지 허용한다.
- 리뷰 제출 성공 후 `rideReviews`, `userReviews` query를 invalidate한다.

## UX 범위

- `/reviews/:matchingId` 상단 안내 카드
  - 운행 ID
  - 리뷰 작성 규칙 안내
- 별점 입력
  - 1~5점 버튼/별 선택
  - 선택 점수 표시
- 빠른 리뷰 태그
  - 시간준수
  - 친절
  - 안전운행
  - 쾌적한 차량
  - 경로 배려
- 코멘트 입력
  - 200자 제한
  - 현재 글자 수 표시
- 리뷰 제출
  - `createReview` mutation 호출
  - 완료 피드백
  - 실패 피드백
- 리뷰 목록
  - 운행 리뷰 목록 `rideReviews`
  - 받은 리뷰 목록 `userReviews`
  - 빈 상태/로딩/에러/재시도 표시

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 사용자가 리뷰 화면에 진입 | 제목, 운행 ID, 별점 입력, 코멘트 입력, 태그가 표시된다. |
| A2 | 별점 1~5 선택 | 선택한 별점이 활성화되고 제출 버튼 조건에 반영된다. |
| A3 | 빠른 태그 다중 선택 | 선택 태그가 활성화되고 코멘트 payload에 포함된다. |
| A4 | 코멘트 입력 | 200자 이하 코멘트와 글자 수가 표시된다. |
| A5 | 리뷰 제출 | `createReview` mutation이 `rideId`, `toUserId`, `rating`, `comment`로 호출된다. |
| A6 | 운행 리뷰 조회 | `rideReviews` 결과가 별점/작성자/코멘트와 함께 표시된다. |
| A7 | 받은 리뷰 조회 | `userReviews` 결과가 별점/작성자/코멘트와 함께 표시된다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | 별점 미선택 | 제출 버튼이 비활성화된다. |
| E2 | 대상자 ID 미입력 | 제출 버튼이 비활성화된다. |
| E3 | 리뷰 생성 실패 | 실패 메시지를 표시하고 입력값을 유지한다. |
| E4 | 운행 리뷰 조회 실패 | 에러 안내와 `다시 시도` 버튼을 표시한다. |
| E5 | 받은 리뷰 조회 실패 | 에러 안내와 `다시 시도` 버튼을 표시한다. |
| E6 | 인증 없음 | 기존 `AuthGuard`가 로그인 흐름으로 보호한다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 리뷰 목록 없음 | 빈 상태를 표시한다. |
| X2 | 코멘트 없이 태그만 선택 | 태그만 포함된 코멘트로 제출한다. |
| X3 | 태그와 코멘트 모두 없음 | `comment=null`로 제출한다. |
| X4 | 코멘트 200자 초과 입력 | UI에서 200자 이상 입력되지 않는다. |
| X5 | route param 없음 | 화면에 잘못된 운행 ID 안내를 표시한다. |

## 수정/생성할 파일 경로

- `frontend/src/graphql/operations/review.graphql`
- `frontend/lib/api/review-api.ts`
- `frontend/hooks/useReviewQueries.ts`
- `frontend/lib/review-format.ts`
- `frontend/app/reviews/[matchingId]/page.tsx`
- `frontend/__tests__/Reviews.test.tsx`

## TDD 순서

1. Red: `__tests__/Reviews.test.tsx`에 별점 선택/제출 테스트 추가
2. Green: `review.graphql`, `review-api.ts`, `useReviewQueries.ts`, `/reviews/[matchingId]` 최소 구현
3. Red: 빠른 태그 다중 선택 테스트 추가
4. Green: 태그 선택과 코멘트 payload 조합 구현
5. Red: 운행 리뷰/받은 리뷰 목록 테스트 추가
6. Green: `rideReviews`, `userReviews` 조회 UI 구현
7. Red: 빈 상태/조회 실패/제출 실패 테스트 추가
8. Green: 로딩/에러/빈 상태 처리
9. Refactor: 별점/리뷰 카드 helper 정리
10. Verify: 전체 검증 명령 실행

## 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- `/reviews/:matchingId` 화면이 인증 사용자 기준으로 렌더링된다.
- 별점 1~5 선택과 코멘트 입력이 동작한다.
- 빠른 리뷰 태그 다중 선택이 동작한다.
- `createReview` mutation이 generated 타입 기반으로 호출된다.
- 운행 리뷰 목록과 받은 리뷰 목록이 표시된다.
- 테스트가 A/E/X 주요 케이스를 검증한다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`가 통과한다.

## 범위 제외

- 리뷰 수정/삭제/신고
- 상대방 자동 판별 로직
- 리뷰 알림
- 리뷰 작성 완료 후 정산 화면 자동 이동
- 별도 리뷰 API 문서 신설
