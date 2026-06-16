# Ridy — 디자인 시스템

## 브랜드 아이덴티티

- **슬로건**: 함께 타는 길
- **키워드**: 안전, 편리, 친환경, 연결
- **디자인 기준**: Socar Frame의 “가장 쏘카다운, 가장 효율적인” 원칙을 Ridy 맥락에 맞게 재해석합니다.
- **적용 원칙**:
  - **Respect the Legacy**: 기존 Ridy 화면 구조와 사용자 흐름을 유지하면서 토큰과 컴포넌트 일관성을 먼저 개선합니다.
  - **Predictability**: 버튼, 입력, 카드, 상태 배지의 크기·상태·색상 의미가 모든 화면에서 예측 가능해야 합니다.
  - **Token first**: 프론트엔드는 문서화된 CSS/Tailwind token을 우선 사용하고, 임의 값은 계획서에 사유가 있을 때만 허용합니다.

## Socar Frame 참고 범위와 채택 정책

| 항목 | 참고 내용 | Ridy 적용 결정 |
|---|---|---|
| Principle | Respect the Legacy, Predictability | 기존 화면을 급격히 바꾸지 않고 token/component 규칙부터 정렬 |
| Foundation package | `@socar-inc/socar-frame-foundation` | Tailwind v3.4.1 기준 문서이므로 Ridy Tailwind v4.3.0에 직접 채택하지 않음 |
| Color | Gray/Blue/Green/Red/Orange 계열의 단계형 palette | Ridy semantic token으로 변환해 사용 |
| Typography | Pretendard 중심 | 기존 Pretendard 유지, role/line-height 세분화 |
| Components | Buttons, ActionButton, Input, Tab, Alert, Badge, Chip, SelectionBox, TopAppBar | shadcn/ui + Ridy custom component 위에 패턴만 반영 |

> **BLOCKED 조건**: Socar Frame npm package를 직접 설치하거나 Tailwind v3 설정을 복제하려면 별도 호환성 검증 계획이 먼저 필요합니다. 현재 구현 계획에서는 token/pattern 기반 적용만 허용합니다.

## 컬러 팔레트

### Foundation palette

Ridy는 Socar Frame의 단계형 색상 체계를 참고하되, 서비스 의미에 맞게 아래 palette를 SSoT로 사용합니다.

| Token | Hex | 용도 |
|---|---|---|
| `white` | `#FFFFFF` | 카드, sheet, dialog 표면 |
| `black` | `#000000` | 고대비 텍스트/오버레이 기준 |
| `gray-50` | `#F9F9FB` | 앱 배경 |
| `gray-100` | `#F3F4F6` | 보조 배경, tab container |
| `gray-200` | `#E5E7EB` | 구분선, disabled border |
| `gray-300` | `#D1D5DB` | 입력 기본 border |
| `gray-500` | `#6B7280` | 보조 텍스트, placeholder |
| `gray-700` | `#374151` | 서브 타이틀 |
| `gray-900` | `#111827` | 메인 텍스트 |
| `gray-1000` | `#141A24` | bottom sheet scrim/dark surface |
| `blue-50` | `#EBF5FF` | primary subtle background |
| `blue-100` | `#D6EAFE` | selected background |
| `blue-500` | `#3B82F6` | hover/focus accent |
| `blue-600` | `#2563EB` | primary CTA |
| `blue-700` | `#1D4ED8` | pressed primary |
| `blue-900` | `#0033A9` | high-emphasis primary text |
| `green-50` | `#E6FEF0` | success/eco subtle background |
| `green-500` | `#10B981` | success, matched, eco |
| `green-700` | `#047857` | success text |
| `green-900` | `#008D7A` | eco high-emphasis |
| `red-50` | `#FFF0F3` | error subtle background |
| `red-500` | `#EF4444` | error/destructive |
| `red-700` | `#B91C1C` | destructive text |
| `red-900` | `#C10027` | critical state |
| `orange-50` | `#FFF7ED` | warning subtle background |
| `orange-500` | `#F59E0B` | warning/pending |
| `orange-700` | `#B45309` | warning text |

### Semantic tokens

| Semantic token | Palette value | 용도 |
|---|---|---|
| `primary` | `blue-600` | 메인 CTA, 활성 탭, 링크, 브랜드 포인트 |
| `primary-hover` | `blue-500` | hover |
| `primary-pressed` | `blue-700` | pressed |
| `primary-subtle` | `blue-50` | selected/subtle CTA background |
| `secondary` | `green-500` | 매칭 완료, 친환경, 성공 |
| `warning` | `orange-500` | 대기, 주의 |
| `danger` | `red-500` | 에러, 거절, 삭제 |
| `surface` | `white` | 기본 카드/폼 표면 |
| `surface-muted` | `gray-50` | 앱 배경 |
| `surface-raised` | `white` + shadow | floating card, modal |
| `border-default` | `gray-200` | 카드/구분선 |
| `border-input` | `gray-300` | 입력 기본 border |
| `text-primary` | `gray-900` | 본문 핵심 텍스트 |
| `text-secondary` | `gray-500` | 설명/placeholder |
| `text-inverse` | `white` | primary CTA 텍스트 |

## 타이포그래피

폰트는 Pretendard로 통일합니다. 한 화면에서 제목 hierarchy는 최대 3단계까지만 사용합니다.

| 역할 | 크기 | Line-height | 굵기 | 용도 |
|---|---:|---:|---:|---|
| Display | 32px | 40px | 700 | 랜딩 hero, 핵심 지표 |
| H1 | 28px | 36px | 700 | 화면 제목 |
| H2 | 24px | 32px | 700 | 섹션 제목 |
| H3 | 20px | 28px | 600 | 카드 제목 |
| Body | 16px | 24px | 400 | 기본 본문/입력 |
| Body Strong | 16px | 24px | 600 | 리스트 핵심 값 |
| Caption | 14px | 20px | 400 | 보조 설명, metadata |
| Caption Strong | 14px | 20px | 600 | badge, tab label |
| Small | 12px | 16px | 400 | disclaimer, timestamp |

## 여백 & 간격

| Token | 값 | 용도 |
|---|---:|---|
| `space-1` | 4px | icon/text 미세 간격 |
| `space-2` | 8px | 필드 내부 보조 간격 |
| `space-3` | 12px | tight stack |
| `space-4` | 16px | 기본 section/card padding |
| `space-5` | 20px | form group 간격 |
| `space-6` | 24px | 느슨한 section 간격 |
| `space-8` | 32px | 화면 블록 간격 |
| `page-mobile` | 16px | 모바일 페이지 padding |
| `page-tablet` | 24px | 태블릿 페이지 padding |
| `page-desktop` | 32px | 데스크탑 페이지 padding |

## Radius & elevation

| Token | 값 | 용도 |
|---|---:|---|
| `radius-sm` | 6px | badge, chip |
| `radius-md` | 8px | button, input |
| `radius-lg` | 12px | card |
| `radius-xl` | 16px | auth card, bottom sheet |
| `radius-2xl` | 24px | hero visual, modal |
| `shadow-card` | `0 1px 2px rgb(20 26 36 / 6%)` | 기본 카드 |
| `shadow-raised` | `0 8px 24px rgb(20 26 36 / 10%)` | floating action/sheet |

## 컴포넌트 규칙

### Button

Socar Frame의 Buttons/ActionButton 패턴을 참고해 목적 기반 variant를 사용합니다.

| Variant | 시각 규칙 | 사용처 |
|---|---|---|
| Primary | `primary` 배경, `text-inverse`, 48px mobile height | 가장 중요한 화면 진행 CTA |
| Secondary | `surface`, `primary` border/text | 보조 CTA, 되돌릴 수 있는 액션 |
| Ghost | 투명 배경, `text-primary` 또는 `text-secondary` | TopAppBar, less prominent action |
| Destructive | `red-50` 배경, `red-700` text | 거절, 삭제, 로그아웃 확인 |
| Action | 원형/정사각 icon button, 44px 이상 | 지도/필터/상단 액션 |

- Touch target은 최소 44px, 모바일 주요 CTA는 48px입니다.
- Loading 상태는 label을 유지하고 보조 loading indicator를 추가합니다.
- Disabled는 opacity만으로 의미를 전달하지 않고 helper text 또는 validation message와 함께 사용합니다.

### Input

| 상태 | 시각 규칙 |
|---|---|
| Default | 48px height, `border-input`, `radius-md`, label 필수 |
| Focus | `primary` ring/border |
| Error | `danger` border, error text `red-700` |
| Disabled | `gray-100` background, `gray-500` text |

- Placeholder는 예시이며 label을 대체하지 않습니다.
- 검색/경로 입력은 leading icon을 사용할 수 있습니다.

### Card

- `surface` 배경, `border-default`, `radius-lg`, `shadow-card`를 기본으로 합니다.
- 클릭 가능한 카드는 `role="button"`, keyboard activation, hover/pressed 상태를 제공합니다.
- 리스트 카드는 내부 정보 hierarchy를 `title → route/time → price/seats/status` 순서로 유지합니다.

### Badge & Chip

| 상태 | 색상 |
|---|---|
| OPEN / available | `blue-50` background, `blue-900` text |
| MATCHED / COMPLETED / success | `green-50` background, `green-700` text |
| PENDING / waiting | `orange-50` background, `orange-700` text |
| FAILED / CANCELLED / rejected | `red-50` background, `red-700` text |
| Neutral metadata | `gray-100` background, `gray-700` text |

### Tab

- Container는 `gray-100` 배경 + 4px padding을 사용합니다.
- Active tab은 `surface` 배경, `shadow-card`, `text-primary`를 사용합니다.
- 접근성: `role="tablist"`, `role="tab"`, `aria-selected`를 유지합니다.

### Alert

- Error/Warning/Success/Info variant를 제공합니다.
- 문장 첫 줄은 원인, 둘째 줄 또는 CTA는 다음 행동을 설명합니다.
- Offline alert는 상단 fixed banner로 표시하되 콘텐츠 읽기를 막지 않습니다.

### TopAppBar

- Mobile: 56px height, 뒤로가기/제목/주요 action 1개 이하.
- Desktop: sticky header 또는 side navigation과 충돌하지 않게 사용합니다.

### MatchingCard (핵심 컴포넌트)

```text
┌──────────────────────────────┐
│ 박준서 · 개발팀        08:30 │
│ 강남역  ─────────▶  수원역   │
│ 5,000원/인       2석 남음    │
│ [OPEN]                  [요청]│
└──────────────────────────────┘
```

- Driver identity, route, fare/seats, ride status를 한 카드 안에서 일관된 순서로 표시합니다.
- CTA가 카드 전체 클릭과 중복될 경우 primary action은 하나만 유지합니다.
- 만석/취소/이미 요청 상태는 CTA 대신 상태 badge와 사유 문구를 표시합니다.

## 아이콘

- 아이콘 세트: Lucide Icons
- 기본 크기: 20px
- 내비게이션/TopAppBar: 24px
- 입력 leading icon: 18px
- 장식 아이콘은 `aria-hidden="true"`를 사용합니다.

## 접근성 기준

- 색상만으로 상태를 전달하지 않습니다. 상태 텍스트, badge label, helper text를 함께 제공합니다.
- 모든 interactive element는 keyboard focus와 visible focus ring을 가져야 합니다.
- Mobile touch target은 최소 44px입니다.
- Form field는 label 또는 명시적인 `aria-label`을 가져야 합니다.
- 모달/sheet/dialog는 focus trap과 닫기 수단을 제공합니다.

## 프론트엔드 구현 매핑

| 문서 token | Frontend target |
|---|---|
| palette/semantic color | `frontend/app/globals.css` CSS custom properties + Tailwind v4 `@theme` |
| typography roles | `frontend/app/globals.css` `--text-*` theme tokens |
| button/input/card radius | `components/ui/button.tsx`, `components/ui/input.tsx`, `components/ui/card.tsx` |
| matching card pattern | `components/ridy/MatchingCard.tsx` |
| bottom navigation state | `components/ridy/BottomNavigation.tsx` |
| tab/auth segmented control | `components/auth/LoginForm.tsx` and future `Tabs` composition |

## 구현 제외 및 후속 검증

- Socar Frame package direct adoption은 현재 제외합니다.
- Tailwind v3 설정으로 downgrade하지 않습니다.
- Figma parity나 pixel-perfect clone은 목표가 아닙니다.
- Direct adoption을 검토하려면 별도 spike에서 Tailwind v4, PostCSS, token import, bundle 영향, visual QA를 검증해야 합니다.
