# Ridy — 화면 정의서

## 화면 목록

| ID | 화면명 | 경로 | 설명 |
|---|---|---|---|
| S00 | 랜딩 페이지 | /landing | 서비스 소개, 시작 CTA, 소셜 프루프 |
| S01 | 스플래시 | / | 앱 로딩, 브랜드 노출 |
| S02 | 로그인/가입 | /login | 가입 코드, 회사 이메일 인증코드, 비밀번호 로그인 |
| S03 | 프로필 설정 | /profile/setup | 신규 유저 정보 입력 |
| S04 | 홈 (탑승자) | / | 집 주변 회사행 카풀 지도 탐색과 선택 |
| S05 | 홈 (차주) | /driver | 동네 출발 위치 등록, 요청 관리 |
| S06 | 주변 카풀 목록 | /matchings | 집 주변 카풀 목록/지도 딥링크 호환 경로 |
| S07 | 매칭 상세 | /matchings/:id | 프로필, 경로, 요청 |
| S08 | 채팅 목록 | /chat | 채팅방 리스트 |
| S09 | 채팅방 | /chat/:id | 실시간 메시지 |
| S10 | 정산 현황 | /payments | 정산 이력, 대기 중 |
| S11 | 마이페이지 | /profile | 프로필, 설정, 차량 정보 |
| S12 | 평점/리뷰 | /reviews/:matchingId | 운행 후 리뷰 |

## 화면 전환 흐름

```
S01 스플래시
  ↓
S02 로그인
  ↓ (가입 코드 + 회사 이메일 인증코드 + 비밀번호 설정)
S03 프로필 설정
  ↓
S04 홈 (탑승자) ←→ S05 홈 (차주)
  ↓                    ↓
S06 주변 카풀 목록   매칭 요청 관리
  ↓
S07 매칭 상세
  ↓ (수락 후)
S09 채팅방
  ↓ (운행 완료)
S12 평점/리뷰
  ↓
S10 정산 현황
```

## 각 화면 상태

### 공통 디자인 시스템 적용 규칙

- 모든 화면은 `DESIGN_SYSTEM.md`의 KT Seamless Flow 참고 기반 token을 우선 사용합니다.
- 화면의 기본 정보 구조는 Gray-first hierarchy를 따르고, Accent/Feedback 색상은 CTA·상태·포커스에만 제한적으로 사용합니다.
- Primary CTA는 화면당 하나를 원칙으로 하며, 보조 액션은 Secondary/Ghost 버튼으로 분리합니다.
- 상태 표현은 Badge 색상과 텍스트를 함께 사용합니다. 색상만으로 `OPEN`, `MATCHED`, `PENDING`, `FAILED`, `CANCELLED`를 구분하지 않습니다.
- 입력 필드는 label 또는 명시적인 `aria-label`을 가져야 하며, placeholder는 label을 대체하지 않습니다.
- Mobile 주요 CTA와 입력은 48px 높이를 기본으로 하고, 모든 터치 타깃은 최소 44px입니다.
- 화면 상단 이동/제목/보조 액션은 TopAppBar 패턴을 사용합니다. Mobile에서는 주요 action 1개 이하만 노출합니다.
- KT Figma UI Kit, 비공개 asset, npm package 직접 설치는 라이선스/호환성 검증 전까지 제외하며, 공개 문서 기반 token/pattern으로만 반영합니다.

### S00 랜딩 페이지
- **히어로**: 슬로건, 서비스 가치, 시작 CTA, 보조 CTA
- **서비스 소개**: 카풀 매칭, 자동 정산, 친환경 임팩트
- **이용 방법**: 가입 코드 → 회사 이메일 인증코드 → 비밀번호 설정 → 매칭 → 탑승
- **소셜 프루프**: 평점, 운행 건수, 절감 CO2 지표
- **하단 CTA**: 로그인/시작 유도와 푸터
- **SEO**: title, description, OG metadata 제공

### S04 홈 (탑승자)
- **주변 카풀 탐색**: 집/동네 또는 현재 위치 중심으로 주변 회사행 카풀 marker와 카드를 표시합니다.
- **빈 상태**: 주변 등록 카풀 없음 → "동네를 넓혀 다시 찾아보세요"
- **카풀 선택**: marker 또는 bottom sheet 카드 선택 후 회사까지 가는 S07 상세/요청으로 이동
- **알림**: 매칭 요청 수락/거절 푸시
- **하단 탭**: 홈, 기록, 채팅, 프로필 순서로 표시합니다. `기록`은 사용자의 지난/예정 운행 기록 또는 정산 기록 진입점입니다.

### S05 홈 (차주)
- **빈 상태**: 등록된 운행 없음 → "카풀을 등록해보세요"
- **출발 위치 등록**: 지도에서 동네 출발 위치, 표시용 동네명, 회사 근무지, 출발 시간을 입력
- **모집 중**: OPEN 상태 회사행 카풀 카드
- **요청 대기**: 새 탑승 요청 배지

---

## 공통 반응형 브레이크포인트

| 구간 | 너비 | 레이아웃 기준 | 내비게이션 | 주요 규칙 |
|---|---:|---|---|---|
| Mobile | 0~639px | 단일 컬럼, 최대 폭 100% | 하단 탭 바 | 터치 타깃 최소 44px, 페이지 패딩 16px |
| Tablet | 640~1023px | 단일 컬럼 + 보조 카드 2열 가능 | 하단 탭 또는 상단 보조 메뉴 | 페이지 패딩 24px, 카드 그리드 2열 허용 |
| Desktop | 1024px 이상 | 중앙 컨테이너 + 사이드 패널 | 상단/좌측 내비게이션 | 콘텐츠 최대 폭 1120px, 주요 리스트/상세 2열 |

### 공통 상태 규칙

- **Loading**: 스켈레톤 UI를 우선 사용합니다. 전체 화면 스피너는 초기 앱 부팅(S01)에만 사용합니다.
- **Empty**: 다음 행동을 유도하는 단일 Primary CTA를 포함합니다.
- **Error**: 원인 메시지, 재시도 버튼, 이전 화면 이동 수단을 제공합니다.
- **Unauthorized**: 로그인 화면(S02)으로 이동하되, 가입 코드와 회사 이메일이 필요한 플로우에서는 입력값을 유지합니다.
- **Offline**: 네트워크 오류 배너를 상단에 고정하고 읽기 가능한 캐시 데이터는 유지합니다.

---

## 화면별 상세 UI 스펙

### S00 랜딩 페이지 (`/landing`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 헤더 | Ridy 로고, 서비스 보기 anchor, 시작하기 CTA | Desktop은 sticky header, Mobile은 로고 + 시작하기만 노출 |
| 히어로 | 슬로건, 보조 설명, 가입 코드 CTA, 서비스 보기 CTA | Primary CTA는 `/login`으로 이동, 보조 CTA는 기능 섹션으로 스크롤 |
| 지표 | 평점, 운행 횟수, CO₂ 절감량 | 데이터 미연동 시 문서화된 예시 값 사용 |
| 기능 | 카풀 매칭, 자동 정산, 친환경 임팩트 카드 | Mobile 1열, Tablet 3열, Desktop 3열 |
| 이용 방법 | 가입 코드 → 회사 이메일 인증코드 → 비밀번호 설정 → 매칭 → 탑승 | 단계 번호와 아이콘 표시 |
| 푸터 | 서비스, 보안, 문의 링크 | 외부 링크는 새 탭 |

### S01 스플래시 (`/` 초기 진입)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 브랜드 | 로고, 서비스 슬로건 | 1초 이상 표시하지 않습니다 |
| 로딩 | 앱 초기화 indicator | 세션 확인 중 표시 |
| 라우팅 | 인증 상태 분기 | 인증됨: S04/S05, 미인증: S02 |

상태: `loading`, `authenticated`, `unauthenticated`, `error`.

### S02 로그인 (`/login`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 로그인 탭 | 회사 이메일, 비밀번호, 로그인 버튼 | 기존 가입자는 회사 이메일 + 비밀번호로 로그인 |
| 가입 코드 | 6자리 입력 필드, 검증 메시지 | 대문자 자동 변환, 잘못된 코드는 inline error |
| 회사 이메일 | 회사 이메일 입력 필드, 인증코드 발송 버튼 | 가입 코드가 가리키는 `Company.domain`과 일치해야 인증코드 발송 가능 |
| 인증코드 | 6자리 인증코드 입력, 재발송 안내 | 이메일 발송 후 노출, 만료/불일치 inline error |
| 비밀번호 설정 | 비밀번호, 비밀번호 확인 입력 | 최소 10자, 영문/숫자 포함, 확인값 불일치 inline error |
| 안내 | “가입 코드와 회사 이메일 인증으로만 가입 가능” 설명 | 회사 이메일 도메인 기반 폐쇄형 서비스 원칙 명시 |

상태: `loginReady`, `loginPending`, `loginFailed`, `signupEmpty`, `requestingEmailCode`, `emailCodeSent`, `invalidInviteCode`, `invalidCompanyEmailDomain`, `verifyingEmailCode`, `invalidEmailCode`, `passwordPolicyError`, `signupPending`, `signupFailed`.

### S03 프로필 설정 (`/profile/setup`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 기본 정보 | 프로필 사진, 이름, 사번, 연락처 | 이름 필수, 사번/연락처는 회사 도메인 운영 정책에 따라 optional |
| 역할 선택 | 차주로 시작 토글 | 활성화 시 차량 정보 영역 표시 |
| 차량 정보 | 모델, 색상, 번호판, 좌석 수 | 차주 토글 활성 시 필수 |
| 완료 CTA | 시작하기 | 유효성 통과 전 disabled |

상태: `newUser`, `driverEnabled`, `validationError`, `submitting`, `submitFailed`.

### S04 홈 — 탑승자 (`/`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 상단 안내 | “집 주변 회사행 카풀” 제목, 현재 동네/위치 사용 CTA | 검색 입력을 제공하지 않습니다. 위치 CTA는 권한 요청 또는 동네 fallback만 담당합니다. |
| 지도 | 집/동네 또는 현재 위치 중심 지도, 5km 반경 테두리, 주변 회사행 카풀 marker | 승인 전 marker는 대략 위치와 `pickupLabel`만 표시. 결과는 중심점 기준 5km 이내로 제한 |
| 제공자 카드 | 차주 이름, 평점, 표시용 출발 위치, 출발 시간, 잔여 좌석 | 카드 선택 시 S07 이동 |
| 빈 상태 | 주변 카풀 없음 | 반경 확대 또는 차주 등록 유도 |
| 하단 탭 | 홈, 기록, 채팅, 프로필 | Mobile 고정, Desktop은 사이드 내비로 전환. `기록`은 검색이 아니라 운행/정산 이력으로 이동 |

상태: `emptyNearby`, `hasNearby`, `locationReady`, `locationDenied`, `loading`, `loadError`, `notificationReceived`.

### S05 홈 — 차주 (`/driver`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 운행 등록 | 지도 출발 위치, 표시용 동네명, 회사 근무지, 시간/좌석/요금 입력 | 등록 성공 시 OPEN 카드 생성 |
| 내 운행 | OPEN/MATCHED/IN_PROGRESS 카드 | 상태별 badge 색상 사용 |
| 요청 관리 | 탑승 요청 리스트, 수락/거절 버튼 | 수락 시 좌석 수 감소, 거절 시 요청 상태 갱신 |
| 빈 상태 | 운행 등록 유도 CTA | 등록 폼으로 focus 이동 |

상태: `noRide`, `openRide`, `requestPending`, `matched`, `inProgress`, `cancelled`, `createFailed`.

### S06 주변 카풀 목록 (`/matchings`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 호환 라우팅 | 위치/동네 기준 조회 | 공유 링크 또는 기존 `/matchings` 진입 시 S04와 동일한 주변 카풀 UI를 표시합니다. |
| 결과 UI | S04 지도/카드 rail 재사용 | 별도 독립 검색 화면을 새로 만들지 않습니다. |
| 빈 상태 | 주변 카풀 없음 | 반경 확대 또는 차주 등록 유도 |

상태: `loading`, `hasNearby`, `emptyNearby`, `requesting`, `requestSuccess`, `requestFailed`.

### S07 매칭 상세 (`/matchings/:id`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 차주 프로필 | 이름, 팀/부서, 평점, 운행 횟수 | 같은 회사 이메일 도메인 구성원 정보만 표시 |
| 경로 상세 | 표시용 동네명, 회사 근무지, 예상 시간, Kakao Maps API 지도 | 승인 전 정확한 집 주소 비노출 |
| 운행 조건 | 요금, 좌석, 선호 조건 | 흡연/반려동물/짐 정보 badge |
| 요청 CTA | 탑승 요청, 메시지 입력 optional | 이미 요청한 경우 상태 badge 표시 |

상태: `loading`, `available`, `alreadyRequested`, `full`, `cancelled`, `requestFailed`.

### S08 채팅 목록 (`/chat`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 검색/필터 | 상대 이름/경로 검색 | Mobile은 상단 search input |
| 채팅방 리스트 | 경로, 상대, 마지막 메시지, unread badge | 최신 메시지순 정렬 |
| 빈 상태 | “아직 채팅이 없습니다” | 매칭 찾기 CTA 제공 |

상태: `loading`, `hasRooms`, `emptyRooms`, `offline`, `loadFailed`.

### S09 채팅방 (`/chat/:id`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 상단 바 | 뒤로가기, 경로, 상대 프로필 | Desktop은 우측 ride summary 패널 표시 |
| 메시지 리스트 | 상대/내 메시지 bubble, system message | 날짜 구분선 표시 |
| 입력 | 메시지 입력, 첨부, 전송 | 빈 입력 전송 disabled |
| 운행 정보 | 픽업 위치/시간 요약 | Mobile은 collapsible banner |

상태: `connecting`, `connected`, `sending`, `sendFailed`, `offline`, `roomClosed`.

### S10 정산 현황 (`/payments`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 요약 카드 | 이번 달 결제/수령/대기 금액 | 역할별로 탑승자/차주 문구 조정 |
| 정산 리스트 | 날짜, 경로, 금액, 상태 | 상태 badge: PENDING/COMPLETED/FAILED/REFUNDED |
| 필터 | 월, 상태 | Desktop은 inline, Mobile은 sheet |
| 빈 상태 | 정산 이력 없음 | 홈 이동 CTA |

상태: `loading`, `hasSettlements`, `empty`, `filterEmpty`, `loadFailed`.

### S11 마이페이지 (`/profile`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 프로필 요약 | 사진, 이름, 기업, 역할, 평점 | 역할 변경 진입점 제공 |
| 차량 정보 | 등록 차량, 수정/삭제 | 차량 없으면 등록 CTA |
| 계정 설정 | 알림, 결제수단, 로그아웃 | 위험 동작은 confirm dialog |
| 회사 정보 | 회사명, 회사 이메일 도메인, 사번, 가입 코드 정책 안내 | 운영자면 도메인 운영 화면 진입 CTA |

상태: `loading`, `profileReady`, `noVehicle`, `editing`, `saveFailed`, `logoutConfirm`.

### S12 평점/리뷰 (`/reviews/:matchingId`)

| 영역 | 구성 요소 | 상태/동작 |
|---|---|---|
| 운행 요약 | 경로, 날짜, 상대 사용자 | 완료된 운행만 접근 가능 |
| 별점 입력 | 1~5 rating control | 미선택 시 제출 disabled |
| 코멘트 | optional textarea | 최대 길이 안내 |
| 제출 CTA | 리뷰 제출 | 중복 제출 방지 |

상태: `ready`, `ratingMissing`, `submitting`, `submitted`, `alreadyReviewed`, `submitFailed`.
