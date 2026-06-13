# FE11 마이페이지 구현 계획

## 목표

- `frontend#11` 마이페이지 화면을 구현한다.
- 사용자가 `/profile`에서 본인 프로필, 평점, 운행 횟수, 회사 정보를 확인하고 프로필을 수정할 수 있게 한다.
- 차주 또는 차주 겸 탑승자가 차량 정보를 등록, 수정, 삭제할 수 있게 한다.
- 알림/앱 설정은 현재 backend 저장 API가 없으므로 로컬 UI 상태로만 동작시킨다.

## 관련 이슈

- Frontend: `project-ridy/frontend#11` `[FEAT] 마이페이지`
- 선행: `project-ridy/frontend#2` 디자인 시스템
- 선행: `project-ridy/frontend#4` 인증
- Backend 선행: `project-ridy/backend#5` 유저 프로필 API

## 관련 문서

- `docs/design/SCREENS.md` S11 마이페이지
- `docs/design/WIREFRAMES.md`
- `docs/design/DESIGN_SYSTEM.md`
- `backend/src/graphql/schema.graphql`
- `frontend/AGENTS.md`

## 현재 API 기준

Frontend 구현은 backend 실제 GraphQL SDL을 우선한다.

```graphql
query Me {
  me { ... }
}

query MyVehicles {
  myVehicles { ... }
}

mutation UpdateProfile($input: UpdateProfileInput!) {
  updateProfile(input: $input) { ... }
}

mutation RegisterVehicle($input: RegisterVehicleInput!) {
  registerVehicle(input: $input) { ... }
}

mutation UpdateVehicle($id: ID!, $input: UpdateVehicleInput!) {
  updateVehicle(id: $id, input: $input) { ... }
}

mutation DeleteVehicle($id: ID!) {
  deleteVehicle(id: $id)
}
```

현재 backend SDL에는 회원탈퇴 mutation, 알림 설정 저장 API, 언어/테마 설정 저장 API가 없다. 이 항목은 화면 내 로컬 상태 또는 placeholder 안내로 제한한다.

## 아키텍처 접근

- GraphQL operation은 기존 `src/graphql/operations/auth.graphql`에 확장한다.
- `npm run codegen`으로 generated 타입과 document를 생성한다.
- API wrapper는 `lib/api/profile-api.ts`에 둔다.
- TanStack Query hook은 `hooks/useProfileQueries.ts`에 둔다.
- 화면은 `app/profile/page.tsx`에 구현한다.
- 로그아웃은 기존 `lib/auth/token-storage.ts`의 `clearAuthTokens`를 사용하고 `/login`으로 이동한다.
- 하단 내비게이션에서 profile 탭 이동 경로는 docs 기준인 `/profile`로 정렬한다.
- 회원탈퇴는 API가 없어 실제 mutation을 호출하지 않고 “준비 중” 안내만 표시한다.

## UX 범위

- `/profile` 상단 프로필 카드
  - 프로필 사진 또는 이니셜 fallback
  - 이름, 이메일, 회사명, 사번, 역할
  - 평점, 운행 횟수
- 프로필 수정 폼
  - 이름
  - 연락처
  - 프로필 이미지 URL
  - 사번
  - 역할
- 차량 정보 관리
  - 차량 목록
  - 차량 등록 폼
  - 차량 수정 폼
  - 차량 삭제 버튼
- 알림 설정
  - 푸시 알림 토글
  - 매칭 알림 토글
  - 정산 알림 토글
- 앱 설정
  - 언어 선택
  - 테마 선택
  - 저장 API 부재 안내
- 계정 영역
  - 로그아웃
  - 회원탈퇴 준비 중 안내

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 사용자가 마이페이지에 진입 | 프로필 카드, 설정 섹션, 차량 섹션, 로그아웃 버튼이 표시된다. |
| A2 | `me` 조회 성공 | 이름, 이메일, 회사, 평점, 운행 횟수, 역할이 표시된다. |
| A3 | 프로필 수정 제출 | `updateProfile` mutation이 이름, 연락처, 이미지 URL, 사번, 역할로 호출된다. |
| A4 | 차량 목록 조회 성공 | `myVehicles` 결과가 모델, 색상, 번호판, 정원과 함께 표시된다. |
| A5 | 차량 등록 제출 | `registerVehicle` mutation이 모델, 색상, 번호판, 정원으로 호출된다. |
| A6 | 차량 수정 제출 | `updateVehicle` mutation이 선택 차량 ID와 수정값으로 호출된다. |
| A7 | 차량 삭제 클릭 | `deleteVehicle` mutation이 선택 차량 ID로 호출된다. |
| A8 | 알림/앱 설정 변경 | 화면 내 토글/선택 상태가 즉시 반영된다. |
| A9 | 로그아웃 클릭 | localStorage 인증 토큰을 제거하고 `/login`으로 이동한다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | `me` 조회 실패 | 프로필 조회 실패 안내와 재시도 버튼을 표시한다. |
| E2 | 차량 조회 실패 | 차량 조회 실패 안내와 재시도 버튼을 표시한다. |
| E3 | 프로필 수정 실패 | 실패 메시지를 표시하고 입력값을 유지한다. |
| E4 | 차량 등록/수정/삭제 실패 | 실패 메시지를 표시하고 목록을 임의 변경하지 않는다. |
| E5 | 필수 차량 필드 누락 | 등록/수정 버튼을 비활성화하거나 브라우저 required 검증을 사용한다. |
| E6 | 인증 없음 | 기존 `AuthGuard`가 로그인 흐름으로 보호한다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 프로필 이미지 없음 | 이름 이니셜 fallback을 표시한다. |
| X2 | 회사 정보 없음 | 회사명 영역에 기본 안내 문구를 표시한다. |
| X3 | 차량이 없음 | 빈 상태와 차량 등록 CTA를 표시한다. |
| X4 | 차량 색상이 없음 | 색상은 선택 정보로 처리한다. |
| X5 | 사용자가 탑승자 역할만 보유 | 차량 섹션은 “차주로 전환하면 차량을 등록할 수 있습니다” 안내와 함께 표시한다. |
| X6 | 회원탈퇴 클릭 | backend API 부재로 준비 중 안내만 표시한다. |

## 수정/생성할 파일 경로

- `frontend/src/graphql/operations/auth.graphql`
- `frontend/lib/api/profile-api.ts`
- `frontend/hooks/useProfileQueries.ts`
- `frontend/lib/profile-format.ts`
- `frontend/app/profile/page.tsx`
- `frontend/app/page.tsx`
- `frontend/app/matchings/page.tsx`
- `frontend/app/matchings/[id]/page.tsx`
- `frontend/app/chat/page.tsx`
- `frontend/app/chat/[id]/page.tsx`
- `frontend/app/payments/page.tsx`
- `frontend/__tests__/Profile.test.tsx`

## TDD 순서

1. Red: `__tests__/Profile.test.tsx`에 프로필 카드 렌더링 테스트 추가
2. Green: `Me`, `MyVehicles` operation과 `/profile` 최소 렌더링 구현
3. Red: 프로필 수정 mutation 테스트 추가
4. Green: 프로필 수정 폼과 `updateProfile` 연동 구현
5. Red: 차량 등록/수정/삭제 테스트 추가
6. Green: 차량 CRUD operation, API wrapper, hook, UI 구현
7. Red: 알림/앱 설정 토글과 로그아웃 테스트 추가
8. Green: 로컬 설정 상태, 토큰 삭제, `/login` 이동 구현
9. Red: 조회/수정 실패와 빈 상태 테스트 추가
10. Green: 로딩/에러/빈 상태 처리
11. Refactor: format helper와 섹션 컴포넌트 정리
12. Verify: 전체 검증 명령 실행

## 검증 명령

```bash
npm run codegen
npm run test
npm run lint
npm run build
```

## 완료 조건

- `/profile` 화면이 인증 사용자 기준으로 렌더링된다.
- 프로필 정보 표시와 수정이 generated GraphQL 타입 기반으로 동작한다.
- 차량 등록, 수정, 삭제가 generated GraphQL 타입 기반으로 동작한다.
- 알림/앱 설정 토글이 화면 내 로컬 상태로 동작하며 저장 API 부재가 안내된다.
- 로그아웃 시 인증 토큰이 제거되고 `/login`으로 이동한다.
- 하단 내비게이션의 profile 탭 경로가 `/profile`로 정렬된다.
- 테스트가 A/E/X 주요 케이스를 검증한다.
- `npm run codegen`, `npm run test`, `npm run lint`, `npm run build`가 통과한다.

## 범위 제외

- 실제 회원탈퇴 mutation 호출
- 알림 설정 서버 저장
- 언어/테마 설정 서버 저장
- 파일 업로드 기반 프로필 사진 변경
- 사용자 선호 설정의 영구 저장
- backend GraphQL SDL 변경
