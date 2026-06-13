# FE09 정산 화면 구현 계획

## 목표

- `frontend#9` 정산 화면을 구현한다.
- 사용자가 `/payments`에서 정산 대기/완료 내역을 확인하고, 정산 상세 금액을 검토한 뒤 결제를 실행할 수 있게 한다.
- 결제수단 등록/삭제와 차주 수익 요약을 같은 화면에서 확인할 수 있게 한다.

## 관련 이슈

- Frontend: `project-ridy/frontend#9` `[FEAT] 정산 화면`
- Backend 선행: `project-ridy/backend#10` 정산/결제 API
- Backend 선행: `project-ridy/backend#12` 정산/평점 API 테스트

## 관련 문서

- `docs/api/PAYMENT.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `docs/design/SCREENS.md` S10 정산 현황
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `backend/src/graphql/schema.graphql`

## 현재 API 기준

Frontend 구현은 backend 실제 GraphQL SDL을 우선한다.

```graphql
query MySettlements($pagination: PaginationInput) {
  mySettlements(pagination: $pagination) { ... }
}

query SettlementDetail($id: ID!) {
  settlementDetail(id: $id) { ... }
}

query MyPaymentMethods {
  myPaymentMethods { ... }
}

mutation PaySettlement($settlementId: ID!, $idempotencyKey: String!) {
  paySettlement(settlementId: $settlementId, idempotencyKey: $idempotencyKey) { ... }
}

mutation RegisterPaymentMethod($input: RegisterPaymentMethodInput!) {
  registerPaymentMethod(input: $input) { ... }
}

mutation DeletePaymentMethod($id: ID!) {
  deletePaymentMethod(id: $id)
}
```

`docs/api/PAYMENT.md`에는 `PaySettlementInput`, `setDefaultPaymentMethod`, 기간 필터 기반 `mySettlements` 등이 남아 있으나, 현재 backend SDL에는 없다. 이번 FE 구현은 backend schema와 codegen 산출 타입에 맞춘다.

## 아키텍처 접근

- GraphQL operation을 `src/graphql/operations/payment.graphql`에 먼저 추가한다.
- `npm run codegen`으로 generated 타입과 document를 생성한다.
- API wrapper는 `lib/api/payment-api.ts`에 둔다.
- TanStack Query hook은 `hooks/usePaymentQueries.ts`에 둔다.
- `/payments`는 `app/payments/page.tsx`에서 구현한다.
- 컴포넌트 안 직접 fetch를 금지하고 API wrapper와 hook을 통해 접근한다.
- 결제 완료 후 `mySettlements`, `settlementDetail` query를 invalidate한다.
- 결제수단 등록/삭제 후 `myPaymentMethods` query를 invalidate한다.
- 실제 Toss Payments 운영 위젯 로딩은 이번 범위에서 제외한다. backend가 mockable `PaymentGateway` 기반이므로 frontend는 결제수단과 `paySettlement` mutation 호출 플로우를 구현한다.

## UX 범위

- `/payments` 상단 요약 카드
  - 대기 정산 금액
  - 완료 정산 금액
  - 차주 수령 예정/누적 금액
- 상태 필터
  - 전체
  - 대기
  - 완료
- 정산 카드 리스트
  - 경로
  - 운행 시간
  - 탑승자/차주
  - 결제 금액
  - 플랫폼 수수료
  - 회사 부담 수수료
  - 사원 부담 수수료
  - 상태 배지
- 정산 상세 패널
  - 운행 정보
  - 금액 breakdown
  - 결제 버튼
- 결제수단 관리
  - 카드/카카오페이/토스페이 등록
  - 결제수단 삭제
  - 기본 결제수단 표시는 backend `isDefault`만 사용
- 빈 상태/로딩/에러/재시도 상태를 제공한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 사용자가 정산 목록에 진입 | 대기/완료 정산이 상태 배지와 금액으로 표시된다. |
| A2 | 대기 필터 선택 | `PENDING` 정산만 표시된다. |
| A3 | 완료 필터 선택 | `PAID` 정산만 표시된다. |
| A4 | 정산 카드 선택 | 상세 패널에 경로, 수수료 breakdown, 결제 버튼이 표시된다. |
| A5 | 결제 실행 | `paySettlement` mutation 호출 후 완료 안내와 갱신된 목록이 표시된다. |
| A6 | 결제수단 등록 | `registerPaymentMethod` mutation 호출 후 결제수단 목록이 갱신된다. |
| A7 | 결제수단 삭제 | `deletePaymentMethod` mutation 호출 후 해당 수단이 목록에서 제거된다. |
| A8 | 차주 수익 데이터 존재 | `driverAmount` 합산으로 예정/누적 수익을 표시한다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | 정산 목록 조회 실패 | 에러 안내와 `다시 시도` 버튼을 표시한다. |
| E2 | 상세 조회 실패 | 상세 패널 대신 에러 안내를 표시한다. |
| E3 | 결제 실패 | 결제 버튼 하단에 실패 메시지를 표시하고 목록 상태는 유지한다. |
| E4 | 결제수단 등록 실패 | 등록 실패 메시지를 표시한다. |
| E5 | 결제수단 삭제 실패 | 삭제 실패 메시지를 표시한다. |
| E6 | 인증 없음 | 기존 `AuthGuard`가 로그인 흐름으로 보호한다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 정산 내역 없음 | 빈 상태와 홈/매칭 탐색 안내를 표시한다. |
| X2 | 결제수단 없음 | 결제 버튼은 가능하되 결제수단 등록 CTA를 함께 보여준다. |
| X3 | 이미 완료된 정산 선택 | 결제 버튼을 숨기고 결제 완료 시각을 표시한다. |
| X4 | `dueDate` 없음 | 기한 문구를 생략하거나 `기한 없음`으로 표시한다. |
| X5 | 금액이 0원 | 자동 결제/회사 부담 케이스로 결제 버튼 문구를 `정산 완료 처리`로 표시한다. |
| X6 | `FAILED`/`CANCELLED` 상태 | 결제 버튼을 숨기고 상태별 설명을 표시한다. |

## 수정/생성할 파일 경로

- `frontend/src/graphql/operations/payment.graphql`
- `frontend/lib/api/payment-api.ts`
- `frontend/hooks/usePaymentQueries.ts`
- `frontend/lib/payment-format.ts`
- `frontend/app/payments/page.tsx`
- `frontend/__tests__/Payments.test.tsx`

## TDD 순서

1. Red: `__tests__/Payments.test.tsx`에 정산 목록 렌더링 테스트 추가
2. Green: `payment.graphql`, `payment-api.ts`, `usePaymentQueries.ts`, `/payments` 최소 구현
3. Red: 상태 필터 테스트 추가
4. Green: 클라이언트 필터 구현
5. Red: 상세 패널과 결제 mutation 테스트 추가
6. Green: 상세 조회, `paySettlement`, query invalidation 구현
7. Red: 결제수단 등록/삭제 테스트 추가
8. Green: payment method mutation과 UI 구현
9. Red: 빈 상태/조회 실패/결제 실패 테스트 추가
10. Green: 로딩/에러/빈 상태 처리
11. Refactor: formatting helper와 작은 UI 섹션 정리
12. Verify: 전체 검증 명령 실행

## 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- `/payments` 화면이 인증 사용자 기준으로 정산 목록과 요약을 표시한다.
- 대기/완료 상태 필터가 동작한다.
- 정산 상세 금액 breakdown이 표시된다.
- 대기 정산 결제 mutation이 호출되고 성공/실패 상태가 표시된다.
- 결제수단 등록/삭제 mutation이 호출되고 성공/실패 상태가 표시된다.
- 테스트가 A/E/X 주요 케이스를 검증한다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`가 통과한다.

## 범위 제외

- Toss Payments 운영 위젯 script 로딩 및 실제 카드 인증 UI
- 기본 결제수단 변경 mutation (`setDefaultPaymentMethod`) 구현
- 기간 필터 API 연동 (`mySettlements(from, to, role)`) 구현
- 회사 관리자 정산 통계 화면
- 영수증 PDF/다운로드
