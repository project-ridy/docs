# 정산/결제 API 개발 기획서

## 목표

- **대상 이슈**: `project-ridy/backend#10` — `[FEAT] 정산/결제 API`
- **목적**: 거리 기반 요금 계산, 내 정산 조회, 정산 결제 처리, 결제수단 등록/조회/삭제 API를 GraphQL schema-first 방식으로 구현한다.
- **범위**: `calculateFare`, `mySettlements`, `settlementDetail`, `paySettlement`, `myPaymentMethods`, `registerPaymentMethod`, `deletePaymentMethod`, 결제 gateway 추상화, 서비스 단위 테스트.
- **범위 제외**: 실제 Toss Payments 운영 API 호출, 결제 취소/환불, 회사 관리자 정산 요약, 웹훅 처리, 정산 자동 생성 배치. 실제 Toss 연동은 gateway 인터페이스와 mock 구현까지만 포함한다.

## 관련 이슈

- `project-ridy/backend#10` — 정산/결제 API
- 후속: `project-ridy/backend#12` — 정산/평점 API 테스트
- 후속: `project-ridy/frontend#9` — 정산 화면

## 관련 docs 문서

- `docs/api/PAYMENT.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/architecture/DATABASE.md`
- `docs/architecture/MSA.md`

## 현재 스펙 차이와 결정

- `docs/api/PAYMENT.md`는 결제수단과 `settlementDetail`까지 포함한다.
- `docs/api/GRAPHQL_GATEWAY.md`와 현재 backend schema는 MVP로 `calculateFare`, `mySettlements`, `paySettlement`만 포함한다.
- Prisma schema에는 이미 `Settlement.companyFee`, `Settlement.passengerFee`, `PaymentMethod`가 존재한다.
- 이번 구현은 DB/`PAYMENT.md`에 있는 결제수단과 상세 조회를 GraphQL schema에 추가한다.
- `calculateFare`는 현재 backend schema와 `GRAPHQL_GATEWAY.md`에 맞춰 **Query**로 유지한다. 이슈 본문의 “mutation” 표기는 구현 중 schema 변경하지 않는다.
- GraphQL `SettlementStatus.PAID`와 Prisma `SettlementStatus.COMPLETED` 명칭이 다르므로 service mapper에서 `COMPLETED -> PAID`, `REFUNDED -> CANCELLED`로 변환한다. 저장 시 GraphQL `PAID`는 Prisma `COMPLETED`로 기록한다.

## 아키텍처 접근

- `src/graphql/schema.graphql`을 먼저 수정한다.
- `npm run codegen`으로 `src/graphql/generated/schema-types.ts`를 갱신한다.
- `src/services/payment/payment.service.ts`에 요금 계산, 정산 조회, 결제 처리, 결제수단 관리를 둔다.
- `src/services/payment/payment.resolver.ts`는 GraphQL entrypoint만 담당한다.
- `src/services/payment/payment-gateway.ts`에 Toss Payments gateway 인터페이스와 mock/default 구현을 둔다.
- 실제 외부 결제 호출은 하지 않고, `approvePayment`가 성공/실패를 주입 가능한 형태로 둔다.
- 내 정산 조회는 같은 회사이며 현재 사용자가 탑승자이거나 차주인 정산만 반환한다.
- `paySettlement`는 passenger 본인 정산만 결제 가능하게 제한한다.
- 멱등성은 `idempotencyKey` 필수 인자를 검증하고, 이미 완료된 정산에는 같은 mutation이 재실행되지 않도록 `BAD_REQUEST` 처리한다. 별도 idempotency ledger 테이블은 없으므로 영속 멱등성 저장은 범위 제외한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE10-A01 | FREE 플랜 요금 계산 | 거리 기반 `passengerAmount`, `platformFee`, `driverAmount`, `companySubsidy=0` 반환 |
| BE10-A02 | ENTERPRISE 플랜 요금 계산 | `companySubsidy=platformFee`, 승객 부담 수수료 없음 |
| BE10-A03 | 내 정산 목록 조회 | 탑승자/차주로 관련된 같은 회사 정산만 반환 |
| BE10-A04 | 정산 상세 조회 | 권한 있는 정산 상세를 반환 |
| BE10-A05 | 정산 결제 승인 | `PENDING -> PAID`, `paidAt` 기록 |
| BE10-A06 | 결제수단 등록 | 현재 사용자 결제수단 생성, 첫 결제수단은 기본값 |
| BE10-A07 | 결제수단 조회 | 현재 사용자 결제수단 목록 반환 |
| BE10-A08 | 결제수단 삭제 | 본인 결제수단 삭제 후 `true` 반환 |

### E: 예외 케이스

| 케이스 ID | 조건 | 처리 |
|---|---|---|
| BE10-E01 | passengers < 1 | `BAD_REQUEST` |
| BE10-E02 | 출발지=도착지 | `BAD_REQUEST` |
| BE10-E03 | 거리 200km 초과 | `BAD_REQUEST: 카풀 거리는 200km 이내여야 합니다` |
| BE10-E04 | 타회사/무관한 정산 접근 | `NOT_FOUND` 또는 `FORBIDDEN` |
| BE10-E05 | 이미 결제된 정산 | `BAD_REQUEST: 이미 결제 완료된 정산입니다` |
| BE10-E06 | 결제 gateway 실패 | `PAYMENT_FAILED` 성격의 `BadRequestException` |
| BE10-E07 | 결제수단 소유자 불일치 | `NOT_FOUND` |
| BE10-E08 | 빈 idempotencyKey | `BAD_REQUEST` |

### X: 엣지 케이스

| 케이스 ID | 설명 | 기대 결과 |
|---|---|---|
| BE10-X01 | 정산 없음 | 빈 배열 반환 |
| BE10-X02 | 첫 결제수단 등록 | `isDefault=true` |
| BE10-X03 | 기본 결제수단 삭제 | 삭제 성공, 후속 기본 재지정은 범위 제외 |
| BE10-X04 | `Settlement.companyId` null legacy row | ride.companyId로 회사 범위 보완 |

## 구현/테스트 케이스 등록표

| 케이스 ID | 구현 파일 | 구현 단위 | 테스트 파일 | 테스트 이름 | 상태 |
|---|---|---|---|---|---|
| BE10-A01 | `src/services/payment/payment.service.ts` | `calculateFare` FREE | `src/services/payment/payment.service.spec.ts` | `FREE 플랜 요금을 계산한다` | TODO |
| BE10-A02 | `src/services/payment/payment.service.ts` | `calculateFare` ENTERPRISE | `src/services/payment/payment.service.spec.ts` | `ENTERPRISE 플랜은 플랫폼 수수료를 회사 보조금으로 처리한다` | TODO |
| BE10-A03 | `payment.service.ts`, `payment.resolver.ts` | `mySettlements` | `payment.service.spec.ts` | `내 정산 목록을 조회한다` | TODO |
| BE10-A04 | `payment.service.ts`, `payment.resolver.ts` | `settlementDetail` | `payment.service.spec.ts` | `정산 상세를 조회한다` | TODO |
| BE10-A05 | `payment.service.ts`, `payment-gateway.ts` | `paySettlement` | `payment.service.spec.ts` | `정산 결제를 승인한다` | TODO |
| BE10-A06 | `payment.service.ts` | `registerPaymentMethod` | `payment.service.spec.ts` | `첫 결제수단을 기본 결제수단으로 등록한다` | TODO |
| BE10-A07 | `payment.service.ts` | `myPaymentMethods` | `payment.service.spec.ts` | `내 결제수단 목록을 조회한다` | TODO |
| BE10-A08 | `payment.service.ts` | `deletePaymentMethod` | `payment.service.spec.ts` | `내 결제수단을 삭제한다` | TODO |
| BE10-E01 | `payment.service.ts` | passengers 검증 | `payment.service.spec.ts` | `탑승 인원이 1보다 작으면 거부한다` | TODO |
| BE10-E02 | `payment.service.ts` | 좌표 검증 | `payment.service.spec.ts` | `출발지와 도착지가 같으면 거부한다` | TODO |
| BE10-E03 | `payment.service.ts` | 거리 제한 | `payment.service.spec.ts` | `200km 초과 카풀 거리를 거부한다` | TODO |
| BE10-E04 | `payment.service.ts` | 권한 검증 | `payment.service.spec.ts` | `무관한 정산 상세 접근을 거부한다` | TODO |
| BE10-E05 | `payment.service.ts` | 상태 검증 | `payment.service.spec.ts` | `이미 결제된 정산 결제를 거부한다` | TODO |
| BE10-E06 | `payment-gateway.ts`, `payment.service.ts` | 결제 실패 | `payment.service.spec.ts` | `결제 gateway 실패를 결제 실패로 변환한다` | TODO |
| BE10-E08 | `payment.service.ts` | idempotencyKey 검증 | `payment.service.spec.ts` | `멱등키가 없으면 결제를 거부한다` | TODO |
| BE10-X01 | `payment.service.ts` | 빈 목록 | `payment.service.spec.ts` | `정산이 없으면 빈 배열을 반환한다` | TODO |

## 수정/생성할 파일 경로

- `src/graphql/schema.graphql` — `PaymentMethod`, `PaymentType`, `RegisterPaymentMethodInput`, `settlementDetail`, 결제수단 query/mutation 추가
- `src/graphql/generated/schema-types.ts` — `npm run codegen` 산출물, 수동 수정 금지
- `src/services/payment/payment-service.module.ts` — resolver/service/gateway provider 등록
- `src/services/payment/payment.resolver.ts` — GraphQL resolver
- `src/services/payment/payment.service.ts` — 정산/요금/결제수단 비즈니스 로직
- `src/services/payment/payment-gateway.ts` — 결제 gateway 인터페이스와 기본 mock 구현
- `src/services/payment/payment.service.spec.ts` — TDD 단위 테스트

## TDD 순서

1. `payment.service.spec.ts`에 요금 계산 정상/예외 테스트를 먼저 작성한다.
2. `src/graphql/schema.graphql`에 필요한 결제수단/상세 필드를 추가하고 `npm run codegen`을 실행한다.
3. `PaymentService.calculateFare`를 최소 구현해 요금 계산 테스트를 통과시킨다.
4. 정산 목록/상세 권한 테스트를 작성하고 조회 로직을 구현한다.
5. 결제 gateway 인터페이스와 결제 승인/실패 테스트를 작성한 뒤 `paySettlement`를 구현한다.
6. 결제수단 등록/조회/삭제 테스트를 작성한 뒤 구현한다.
7. `PaymentResolver`, `PaymentServiceModule`을 연결한다.
8. `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`, `npx prisma validate`, `npx prisma generate`를 실행한다.

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

- 거리 기반 요금 계산이 플랜별 수수료/보조금 규칙을 반영한다.
- 내 정산 목록/상세 조회가 회사/참여자 권한을 지킨다.
- 정산 결제 승인/실패/이미 결제됨/멱등키 누락이 테스트된다.
- 결제수단 등록/조회/삭제가 본인 범위에서 동작한다.
- generated 타입을 사용하고 GraphQL schema-first 순서를 지킨다.
- 등록표의 TODO 케이스가 구현/테스트와 연결된다.
- 검증 명령이 통과한다.
