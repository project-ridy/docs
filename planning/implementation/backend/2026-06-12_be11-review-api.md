# 평점/리뷰 API 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#11` — `[FEAT] 평점/리뷰 API`
- **목적**: 완료된 카풀 운행 후 차주와 탑승자가 서로 평점/리뷰를 작성하고, 받은 리뷰 조회와 유저 평균 평점 갱신을 제공한다.
- **범위**: `Review`, `CreateReviewInput`, `createReview`, `rideReviews`, `userReviews`, 중복 리뷰 방지, 평균 평점 업데이트.
- **범위 제외**: 리뷰 수정/삭제, 신고/블라인드, 리뷰 알림, 관리자 리뷰 moderation, 리뷰 E2E 전체 보강. 정산/평점 통합 테스트는 `backend#12`에서 진행한다.

## 관련 이슈

- `project-ridy/backend#11` — 평점/리뷰 API
- 후속: `project-ridy/backend#12` — 정산/평점 API 테스트
- 후속: `project-ridy/frontend#10` — 평점/리뷰 화면

## 관련 docs 문서

- `docs/architecture/DATABASE.md` — `Review` 모델, `User.rating`, `User.rideCount`
- `docs/api/GRAPHQL_GATEWAY.md` — User rating 필드
- `docs/architecture/ARCHITECTURE.md` — Matching Service 범위
- `docs/planning/ROADMAP.md` — Phase 3 정산 + 평점

## 아키텍처 접근

- GraphQL schema-first 방식으로 `src/graphql/schema.graphql`을 먼저 수정한다.
- `npm run codegen`으로 generated 타입을 갱신한다.
- `src/services/review/review.service.ts`에 리뷰 생성/조회/평점 재계산 로직을 둔다.
- `src/services/review/review.resolver.ts`는 GraphQL entrypoint만 담당한다.
- `src/services/review/review-service.module.ts`를 추가하고 `AppModule`에 등록한다.
- 리뷰 작성 가능 조건은 service에서 검증한다.
- 평균 평점 업데이트는 리뷰 생성과 같은 트랜잭션에서 처리한다.

## GraphQL 설계

```graphql
type Review {
  id: ID!
  ride: Ride!
  fromUser: User!
  toUser: User!
  rating: Int!
  comment: String
  createdAt: DateTime!
}

input CreateReviewInput {
  rideId: ID!
  toUserId: ID!
  rating: Int!
  comment: String
}

type Query {
  rideReviews(rideId: ID!): [Review!]! @auth @companyScope
  userReviews(userId: ID!, pagination: PaginationInput): [Review!]! @auth @companyScope
}

type Mutation {
  createReview(input: CreateReviewInput!): Review @auth @companyScope
}
```

## 권한/비즈니스 규칙

- 완료된 운행(`RideStatus.COMPLETED`)에 대해서만 리뷰를 작성할 수 있다.
- 리뷰 작성자는 운행의 차주 또는 `ACCEPTED` 탑승자여야 한다.
- 리뷰 대상자는 같은 운행의 상대 참여자여야 한다.
- 자기 자신에게 리뷰를 작성할 수 없다.
- 같은 운행에서 같은 작성자가 같은 대상자에게 리뷰를 중복 작성할 수 없다.
- 평점은 1~5 정수만 허용한다.
- 코멘트는 선택값이며 trim 후 빈 문자열이면 null로 저장한다.
- 코멘트 길이는 1000자 이내로 제한한다.
- 리뷰 생성 후 대상 유저의 받은 리뷰 평균을 `User.rating`에 소수점 1자리로 반영한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE11-A01 | 탑승자가 완료 운행의 차주를 리뷰 | Review 생성, 차주 평균 평점 갱신 |
| BE11-A02 | 차주가 완료 운행의 탑승자를 리뷰 | Review 생성, 탑승자 평균 평점 갱신 |
| BE11-A03 | 운행별 리뷰 조회 | 같은 회사의 해당 운행 리뷰 목록 반환 |
| BE11-A04 | 유저 받은 리뷰 조회 | 같은 회사 유저가 받은 리뷰 목록 반환 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE11-E01 | 평점 1~5 범위 밖 | `BAD_REQUEST` |
| BE11-E02 | 운행 미완료 | `BAD_REQUEST` |
| BE11-E03 | 리뷰 작성자가 운행 참여자가 아님 | `FORBIDDEN` |
| BE11-E04 | 리뷰 대상자가 운행 참여자가 아님 | `BAD_REQUEST` |
| BE11-E05 | 자기 자신 리뷰 | `BAD_REQUEST` |
| BE11-E06 | 중복 리뷰 | `CONFLICT` |
| BE11-E07 | 타회사 운행/유저 조회 | `NOT_FOUND` 또는 빈 목록 |
| BE11-E08 | 코멘트 1000자 초과 | `BAD_REQUEST` |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE11-X01 | 코멘트 공백 문자열 | `comment=null` 저장 |
| BE11-X02 | 받은 리뷰가 1개뿐인 유저 | 평균 평점이 해당 rating으로 갱신 |
| BE11-X03 | 여러 리뷰 평균이 소수점 발생 | 소수점 1자리 반올림 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE11-A01 | `src/services/review/review.service.ts` | `createReview` passenger→driver | `src/services/review/review.service.spec.ts` | `탑승자가 완료 운행의 차주를 리뷰한다` | TODO |
| BE11-A02 | `review.service.ts` | `createReview` driver→passenger | `review.service.spec.ts` | `차주가 완료 운행의 탑승자를 리뷰한다` | TODO |
| BE11-A03 | `review.service.ts`, `review.resolver.ts` | `rideReviews` | `review.service.spec.ts` | `운행 리뷰 목록을 조회한다` | TODO |
| BE11-A04 | `review.service.ts`, `review.resolver.ts` | `userReviews` | `review.service.spec.ts` | `유저가 받은 리뷰 목록을 조회한다` | TODO |
| BE11-E01 | `review.service.ts` | rating 검증 | `review.service.spec.ts` | `평점 범위를 검증한다` | TODO |
| BE11-E02 | `review.service.ts` | 완료 운행 검증 | `review.service.spec.ts` | `완료되지 않은 운행 리뷰를 거부한다` | TODO |
| BE11-E03 | `review.service.ts` | 작성자 참여 검증 | `review.service.spec.ts` | `비참여자 리뷰 작성을 거부한다` | TODO |
| BE11-E04 | `review.service.ts` | 대상자 참여 검증 | `review.service.spec.ts` | `운행 참여자가 아닌 대상 리뷰를 거부한다` | TODO |
| BE11-E05 | `review.service.ts` | 자기 리뷰 검증 | `review.service.spec.ts` | `자기 자신 리뷰를 거부한다` | TODO |
| BE11-E06 | `review.service.ts` | 중복 검증 | `review.service.spec.ts` | `중복 리뷰를 거부한다` | TODO |
| BE11-E08 | `review.service.ts` | comment 길이 검증 | `review.service.spec.ts` | `1000자 초과 코멘트를 거부한다` | TODO |
| BE11-X01 | `review.service.ts` | comment normalize | `review.service.spec.ts` | `공백 코멘트를 null로 저장한다` | TODO |
| BE11-X03 | `review.service.ts` | 평균 반올림 | `review.service.spec.ts` | `받은 리뷰 평균을 소수점 1자리로 갱신한다` | TODO |

## 수정/생성할 파일 경로

- `src/graphql/schema.graphql` — Review SDL, input, query, mutation 추가
- `src/graphql/generated/schema-types.ts` — `npm run codegen` 산출물, 수동 수정 금지
- `src/services/review/review-service.module.ts` — Review module
- `src/services/review/review.resolver.ts` — Review resolver
- `src/services/review/review.service.ts` — Review business logic
- `src/services/review/review.service.spec.ts` — TDD 단위 테스트
- `src/app/app.module.ts` — Review module 등록

## TDD 순서

1. `src/graphql/schema.graphql`에 Review SDL을 추가한다.
2. `npm run codegen`으로 generated 타입을 갱신한다.
3. `review.service.spec.ts`에 `BE11-A01` 실패 테스트를 먼저 작성한다.
4. `ReviewService.createReview` 최소 구현으로 Green을 만든다.
5. `BE11-E01`~`BE11-E06` 검증 테스트를 순차 추가하고 통과시킨다.
6. 조회 테스트(`BE11-A03`, `BE11-A04`)를 추가하고 service/resolver를 연결한다.
7. 평균 평점 반올림 테스트(`BE11-X03`)를 추가하고 트랜잭션 로직을 정리한다.
8. 전체 검증을 실행한다.

## 실행할 검증 명령

```bash
npm run codegen
npm run test
npm run test:e2e
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- 리뷰 생성 mutation이 완료된 운행 참여자 간에서만 동작한다.
- 중복 리뷰, 자기 리뷰, 비참여자 리뷰, 미완료 운행 리뷰가 거부된다.
- 운행별/유저별 리뷰 조회가 같은 회사 범위를 지킨다.
- 리뷰 생성 시 대상 유저 평균 평점이 소수점 1자리로 갱신된다.
- generated 타입을 사용하고 GraphQL schema-first 순서를 지킨다.
- 등록표의 TODO 케이스가 구현/테스트와 연결된다.
- 검증 명령이 통과한다.
