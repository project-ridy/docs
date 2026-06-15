# 정산/평점 API 테스트 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#12` — `[TEST] 정산/평점 API 테스트`
- **목적**: `backend#10` 정산/결제 API와 `backend#11` 평점/리뷰 API를 GraphQL E2E 관점에서 검증하고 회귀를 방지한다.
- **범위**: payment/review GraphQL E2E 테스트 추가, 기존 unit test 보강, 인증/권한/예외 응답 확인.
- **범위 제외**: 실제 Toss API 호출, 결제 취소/환불 API, 프론트엔드 화면 테스트, 운영 DB 기반 커버리지 리포트 강제.

## 관련 이슈

- `project-ridy/backend#10` — 정산/결제 API
- `project-ridy/backend#11` — 평점/리뷰 API
- `project-ridy/backend#12` — 정산/평점 API 테스트

## 관련 docs 문서

- `docs/planning/implementation/backend/2026-06-12_be10-settlement-payment-api.md`
- `docs/planning/implementation/backend/2026-06-12_be11-review-api.md`
- `docs/api/PAYMENT.md`
- `docs/architecture/DATABASE.md`

## 아키텍처 접근

- `test/payment-review.e2e-spec.ts`를 추가한다.
- 기존 E2E 패턴과 동일하게 `AppModule`, `PrismaService` override, Supertest `POST /graphql` 방식을 사용한다.
- 인증은 기존 E2E의 JWT helper 또는 mock token 패턴을 재사용한다.
- Payment gateway는 module provider를 override하거나 mock gateway provider를 사용한다.
- DB는 실제 연결 없이 Prisma mock으로 동작을 검증한다.
- 기존 service unit tests가 이미 일부 비즈니스 규칙을 검증하므로 E2E는 GraphQL schema, resolver wiring, auth/permission, error shape를 중심으로 한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE12-A01 | `calculateFare` query | 거리 기반 요금 반환 |
| BE12-A02 | `mySettlements` query | 내 정산 목록 반환 |
| BE12-A03 | `paySettlement` mutation | 정산 상태 `PAID` 반환 |
| BE12-A04 | `registerPaymentMethod` mutation | 결제수단 등록 반환 |
| BE12-A05 | `myPaymentMethods` query | 결제수단 목록 반환 |
| BE12-A06 | `deletePaymentMethod` mutation | `true` 반환 |
| BE12-A07 | `createReview` mutation | 리뷰 생성 반환 |
| BE12-A08 | `rideReviews` query | 운행 리뷰 목록 반환 |
| BE12-A09 | `userReviews` query | 유저 받은 리뷰 목록 반환 |

### E: 예외 케이스

| 케이스 ID | 조건 | 기대 결과 |
|---|---|---|
| BE12-E01 | 인증 없이 payment query | GraphQL error |
| BE12-E02 | `calculateFare` passengers=0 | GraphQL error |
| BE12-E03 | 이미 결제된 정산 결제 | GraphQL error |
| BE12-E04 | 리뷰 평점 6 | GraphQL error |
| BE12-E05 | 중복 리뷰 | GraphQL error |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE12-X01 | payment/review schema field selection | 새 GraphQL field가 runtime에서 resolve됨 |
| BE12-X02 | 기존 auth/matching/chat E2E 회귀 | 기존 E2E 전체 pass |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE12-A01 | `payment.resolver.ts` | `calculateFare` | `test/payment-review.e2e-spec.ts` | `요금을 계산한다` | TODO |
| BE12-A02 | `payment.resolver.ts` | `mySettlements` | `payment-review.e2e-spec.ts` | `내 정산 목록을 조회한다` | TODO |
| BE12-A03 | `payment.resolver.ts` | `paySettlement` | `payment-review.e2e-spec.ts` | `정산 결제를 승인한다` | TODO |
| BE12-A04 | `payment.resolver.ts` | `registerPaymentMethod` | `payment-review.e2e-spec.ts` | `결제수단을 등록한다` | TODO |
| BE12-A05 | `payment.resolver.ts` | `myPaymentMethods` | `payment-review.e2e-spec.ts` | `결제수단 목록을 조회한다` | TODO |
| BE12-A06 | `payment.resolver.ts` | `deletePaymentMethod` | `payment-review.e2e-spec.ts` | `결제수단을 삭제한다` | TODO |
| BE12-A07 | `review.resolver.ts` | `createReview` | `payment-review.e2e-spec.ts` | `리뷰를 생성한다` | TODO |
| BE12-A08 | `review.resolver.ts` | `rideReviews` | `payment-review.e2e-spec.ts` | `운행 리뷰 목록을 조회한다` | TODO |
| BE12-A09 | `review.resolver.ts` | `userReviews` | `payment-review.e2e-spec.ts` | `유저 받은 리뷰 목록을 조회한다` | TODO |
| BE12-E01 | auth guard | auth error | `payment-review.e2e-spec.ts` | `인증 없이는 정산을 조회할 수 없다` | TODO |
| BE12-E02 | `PaymentService` | fare validation | `payment-review.e2e-spec.ts` | `잘못된 요금 계산 입력을 거부한다` | TODO |
| BE12-E03 | `PaymentService` | payment status validation | `payment-review.e2e-spec.ts` | `이미 결제된 정산 결제를 거부한다` | TODO |
| BE12-E04 | `ReviewService` | rating validation | `payment-review.e2e-spec.ts` | `잘못된 평점을 거부한다` | TODO |
| BE12-E05 | `ReviewService` | duplicate validation | `payment-review.e2e-spec.ts` | `중복 리뷰를 거부한다` | TODO |

## 수정/생성할 파일 경로

- `test/payment-review.e2e-spec.ts` — 정산/평점 GraphQL E2E 테스트
- 필요 시 `src/services/payment/payment.service.spec.ts` — payment unit 보강
- 필요 시 `src/services/review/review.service.spec.ts` — review unit 보강

## TDD 순서

1. `payment-review.e2e-spec.ts`에 인증 없는 payment query 실패 테스트를 먼저 작성하고 실패를 확인한다.
2. AppModule 테스트 wiring과 Prisma mock을 구성해 Green으로 만든다.
3. payment 정상/예외 E2E를 추가한다.
4. review 정상/예외 E2E를 추가한다.
5. 필요하면 unit test를 보강한다.
6. 전체 검증을 실행한다.

## 실행할 검증 명령

```bash
npm run test
npm run test:e2e
npm run lint
npm run build
npx prisma validate
npx prisma generate
```

## 완료 조건

- 정산/결제 GraphQL E2E 정상/예외 테스트가 추가된다.
- 평점/리뷰 GraphQL E2E 정상/예외 테스트가 추가된다.
- 기존 auth/matching/chat E2E가 회귀 없이 통과한다.
- unit tests와 build/lint/Prisma 검증이 통과한다.
