# FE12 랜딩 페이지 구현 계획

> **docs#44 이후 상태:** 이 계획서는 기존 “초대 코드 가입” 표현을 포함한 과거 계획서입니다. 구현 전 최신 SSoT인 `docs/design/SCREENS.md`, `docs/design/WIREFRAMES.md`, `docs/planning/PLANNING.md` 기준으로 가입 코드 + 회사 이메일 도메인 검증 메시지를 반영해야 합니다. 구식 계획 그대로 구현하면 `BLOCKED`입니다.

## 목표

- `frontend#12` 공식 마케팅 랜딩 페이지를 구현한다.
- 기존 인증 홈(`/`)을 변경하지 않고 public route `/landing`에 랜딩을 추가한다.
- Ridy의 핵심 가치인 같은 회사 카풀 매칭, 자동 정산, 친환경 임팩트를 명확히 전달한다.
- SEO metadata와 반응형 UI를 제공한다.

## 관련 이슈

- Frontend: `project-ridy/frontend#12` `[FEAT] 랜딩 페이지`
- 선행: `project-ridy/frontend#2` 디자인 시스템

## 관련 문서

- `docs/design/SCREENS.md` S00 랜딩 페이지
- `docs/design/WIREFRAMES.md` 0. 랜딩 페이지
- `docs/design/DESIGN_SYSTEM.md`
- `frontend/AGENTS.md`

## 아키텍처 접근

- 화면은 `app/landing/page.tsx`에 static server component로 구현한다.
- SEO metadata는 `app/landing/page.tsx`의 `metadata` export로 설정한다.
- 인증 앱 홈(`/`)은 유지한다. `/`를 랜딩으로 전환하는 라우팅 정책은 별도 제품 결정으로 둔다.
- CTA는 기존 인증 흐름에 맞춰 `/login`으로 연결한다.
- 디자인은 Ridy 디자인 시스템의 Primary Blue, Secondary Emerald, Gray scale, rounded card 패턴을 사용한다.
- 외부 이미지 없이 CSS gradient와 카드 UI로 구현해 LCP/CLS 리스크를 낮춘다.

## UX 범위

- 헤더
  - Ridy 로고 텍스트
  - 서비스 소개 anchor
  - 시작 CTA
- 히어로 섹션
  - 슬로건
  - 핵심 가치 제안
  - primary CTA `/login`
  - secondary CTA `#how-it-works`
- 서비스 소개
  - 같은 회사 카풀 매칭
  - 투명한 자동 정산
  - 친환경 임팩트
- 이용 방법 3단계
  - 초대 코드 가입
  - 출퇴근 경로 매칭
  - 함께 탑승하고 정산
- 소셜 프루프
  - 평균 평점
  - 누적 운행 건수
  - 절감 CO2
- 하단 CTA + 푸터
  - 시작 유도
  - 보안/회사 전용/문의 링크 텍스트

## SEO 범위

- title: `Ridy - 함께 타는 출퇴근 카풀`
- description: 같은 회사 동료와 안전하게 출퇴근 카풀을 찾고 정산하는 서비스 설명
- Open Graph title/description/type/url
- Twitter card metadata
- canonical URL은 환경 도메인이 확정되지 않아 metadataBase 없이 상대 URL만 사용한다.
- JSON-LD는 `Organization` + `SoftwareApplication` schema를 inline script로 제공한다.

## A/E/X 케이스 분석

### A: 정상 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| A1 | 사용자가 `/landing`에 진입 | 히어로, 서비스 소개, 이용 방법, 소셜 프루프, 하단 CTA가 표시된다. |
| A2 | primary CTA 클릭 | `/login` 링크를 제공한다. |
| A3 | secondary CTA 클릭 | 이용 방법 섹션 anchor로 이동한다. |
| A4 | 서비스 소개 확인 | 매칭, 정산, 친환경 임팩트 카드가 표시된다. |
| A5 | 소셜 프루프 확인 | 평점, 운행 건수, CO2 절감 지표가 표시된다. |

### E: 예외 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| E1 | JavaScript가 늦게 로드됨 | server-rendered static content로 핵심 카피와 CTA가 표시된다. |
| E2 | 외부 이미지 로드 실패 | 외부 이미지 의존성이 없어 레이아웃이 깨지지 않는다. |
| E3 | 인증 토큰 없음 | 랜딩은 public route이므로 로그인 리다이렉트가 발생하지 않는다. |

### X: 엣지 케이스

| ID | 케이스 | 기대 결과 |
|---|---|---|
| X1 | 모바일 375px 뷰포트 | CTA와 주요 카피가 한 컬럼으로 표시된다. |
| X2 | 데스크탑 넓은 화면 | 히어로와 카드 섹션이 max-width 컨테이너 안에서 정렬된다. |
| X3 | 긴 한국어 카피 | 줄바꿈과 max-width로 overflow 없이 표시된다. |

## 수정/생성할 파일 경로

- `frontend/app/landing/page.tsx`
- `frontend/__tests__/Landing.test.tsx`

## TDD 순서

1. Red: `__tests__/Landing.test.tsx`에 랜딩 섹션 렌더링 테스트 추가
2. Green: `app/landing/page.tsx` 최소 정적 페이지 구현
3. Red: CTA 링크와 SEO JSON-LD 테스트 추가
4. Green: CTA href, metadata, structured data 구현
5. Red: 서비스 소개/이용 방법/소셜 프루프 테스트 추가
6. Green: 섹션 데이터 기반 렌더링 구현
7. Refactor: section data와 카드 markup 정리
8. Verify: 전체 검증 명령 실행

## 검증 명령

```bash
npm run test
npm run lint
npm run build
```

Lighthouse Performance 90+는 로컬 브라우저 측정 환경이 있을 때 `/landing` production build 기준으로 확인한다. 이번 PR의 최소 대체 검증은 정적 route build 성공과 외부 이미지 미사용이다.

## 완료 조건

- `/landing` public route가 렌더링된다.
- 히어로, 서비스 소개, 이용 방법, 소셜 프루프, 하단 CTA, 푸터가 표시된다.
- CTA가 `/login`과 `#how-it-works`로 연결된다.
- SEO metadata와 structured data가 제공된다.
- 모바일/데스크탑 반응형 레이아웃을 제공한다.
- 테스트가 A/E/X 주요 케이스를 검증한다.
- `npm run test`, `npm run lint`, `npm run build`가 통과한다.

## 범위 제외

- `/`를 랜딩으로 전환하는 라우팅 정책 변경
- 다운로드 앱 링크 실제 스토어 연동
- CMS/관리자 기반 랜딩 카피 편집
- 외부 OG 이미지 파일 생성
- 유료 광고 전환 추적 스크립트
