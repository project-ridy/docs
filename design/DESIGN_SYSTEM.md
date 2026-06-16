# Ridy — 디자인 시스템

## 브랜드 아이덴티티

- **슬로건**: 함께 타는 길
- **키워드**: 안전, 편리, 친환경, 연결
- **디자인 기준**: KT Seamless Flow v1.2의 “모든 경험을 하나로 잇는 흐름”을 Ridy의 폐쇄형 회사 카풀 맥락에 맞게 재해석합니다.
- **적용 원칙**:
  - **Seamless Flow**: 화면 이동, 상태 변화, CTA 우선순위를 끊김 없이 연결합니다.
  - **Gray-first hierarchy**: 화면의 80%는 Gray Scale 기반 정보 구조로 유지하고, Accent/Feedback 색상은 20% 이내의 강조에만 사용합니다.
  - **Token first**: 프론트엔드는 문서화된 CSS/Tailwind token을 우선 사용하고, 임의 값은 구현 계획서에 사유가 있을 때만 허용합니다.
  - **Accessible by default**: 텍스트 명도 대비, 키보드 조작, 44px 이상 터치 타깃, 모션 줄이기 설정을 기본 조건으로 둡니다.

## KT Seamless Flow 참고 범위와 채택 정책

| 항목 | KT Seamless Flow에서 확인한 기준 | Ridy 적용 결정 |
|---|---|---|
| Information architecture | Foundations, Components, Patterns, AI Agent, UX Writing, Resources | Ridy는 Foundations + Components + Patterns + UX Writing을 우선 적용하고 AI Agent는 후속 범위로 둠 |
| Token tier | Primitive Token → Semantic Token → Component Token | Ridy도 `primitive → semantic → component` 3계층으로 문서/코드 매핑 |
| Color usage | Gray Scale 기반, Accent는 브랜드/서비스 강조, Feedback은 성공/안내/긴급 상태 전달 | Ridy 화면은 Gray-first로 정리하고 `primary`, `secondary`, `success`, `warning`, `danger`만 의미 색상으로 노출 |
| Accessibility | 본문 4.5:1 이상, 제목/CTA/icon 3:1 이상, touch target 44x44px 이상 | WCAG AA를 기본 완료 조건으로 지정 |
| Typography | KT Flow(Display), Pretendard(Title & Body), 6단계 Type Scale | 구현은 Pretendard를 유지하고 KT Flow는 브랜드 에셋/히어로용 선택 범위로만 문서화 |
| Spacing | 4pt Grid, 복잡한 컴포넌트에 한해 2의 배수 제한 허용 | Ridy spacing token은 4px 단위를 기본으로 유지 |
| Radius | 기본 8px, 작은 요소 6px, 큰 sheet 16px, pill/circle 제한 사용 | 버튼/입력 8px, tooltip/badge 6px, bottom sheet 16px, chip은 pill 적용 |
| Elevation | Level 0~4, Shadow 1~3, Dim 65% | card/input/dropdown/popup 레이어를 명확히 분리 |

> **BLOCKED 조건**: KT 원본 Figma UI Kit, npm package, 또는 비공개 토큰 export를 직접 가져와야 하는 작업은 별도 라이선스/접근 권한 확인과 호환성 검증 계획이 먼저 필요합니다. 현재 구현 계획에서는 공개 문서에서 확인 가능한 token/pattern 기반 적용만 허용합니다.

## 컬러 팔레트

### Foundation palette

KT Seamless Flow는 Gray Scale을 정보 전달과 시각적 위계의 기반으로 정의하고, `400` 이하 단계는 border/background 같은 구조 요소에, 텍스트는 최소 `500` 이상 단계 사용을 권장합니다. Ridy는 접근성 대비를 보장하기 위해 아래 palette를 SSoT로 사용합니다.

| Token | Hex | 용도 |
|---|---|---|
| `white` | `#FFFFFF` | 카드, sheet, dialog 표면 |
| `black` | `#000000` | 고대비 텍스트/오버레이 기준 |
| `gray-50` | `#F9FAFB` | 앱 배경, low emphasis surface |
| `gray-100` | `#F3F4F6` | 보조 배경, tab container, disabled fill |
| `gray-150` | `#EEF0F3` | gray background 위 보조 fill |
| `gray-200` | `#E5E7EB` | 구분선, disabled border |
| `gray-300` | `#D1D5DB` | 입력 기본 border |
| `gray-400` | `#9CA3AF` | 구조 아이콘, 비활성 border |
| `gray-450` | `#8B939F` | gray background 위 구조 아이콘/border |
| `gray-500` | `#6B7280` | white background 위 tertiary text 최소 단계 |
| `gray-550` | `#5E6673` | gray background 위 tertiary text 최소 단계 |
| `gray-700` | `#374151` | 서브 타이틀, secondary text |
| `gray-900` | `#111827` | 메인 텍스트 |
| `gray-1000` | `#141414` | dim/scrim 기준 |
| `red-50` | `#FFF1F2` | critical subtle background |
| `red-500` | `#EF4444` | white background 위 critical text/action |
| `red-600` | `#DC2626` | gray background 위 critical text/action |
| `red-700` | `#B91C1C` | destructive text |
| `teal-50` | `#ECFDF5` | eco/success subtle background |
| `teal-500` | `#14B8A6` | eco/accent secondary |
| `teal-700` | `#0F766E` | eco/success text |
| `blue-50` | `#EFF6FF` | info/primary subtle background |
| `blue-500` | `#3B82F6` | focus/info accent |
| `blue-600` | `#2563EB` | primary CTA |
| `blue-700` | `#1D4ED8` | primary pressed |
| `orange-50` | `#FFF7ED` | warning subtle background |
| `orange-500` | `#F59E0B` | warning/pending |
| `orange-700` | `#B45309` | warning text |

### Semantic tokens

| Semantic token | Palette value | 용도 |
|---|---|---|
| `primary` | `blue-600` | 메인 CTA, 활성 탭, 링크, 주요 focus |
| `primary-hover` | `blue-500` | hover/focus accent |
| `primary-pressed` | `blue-700` | pressed |
| `primary-subtle` | `blue-50` | selected/subtle CTA background |
| `secondary` | `teal-500` | 친환경/보조 accent |
| `success` | `teal-500` | 매칭 완료, 성공 |
| `success-subtle` | `teal-50` | 성공 subtle background |
| `warning` | `orange-500` | 대기, 주의 |
| `danger` | `red-500` | 에러, 거절, 삭제 |
| `danger-on-muted` | `red-600` | gray background 위 critical text |
| `surface` | `white` | 기본 카드/폼 표면 |
| `surface-muted` | `gray-50` | 앱 배경 |
| `surface-secondary` | `gray-100` | 보조 컨테이너 |
| `surface-raised` | `white` + `shadow-2` | dropdown, floating card |
| `surface-overlay` | `rgb(20 20 20 / 65%)` | modal/sheet dim |
| `border-default` | `gray-200` | 카드/구분선 |
| `border-input` | `gray-300` | 입력 기본 border |
| `text-primary` | `gray-900` | 본문 핵심 텍스트 |
| `text-secondary` | `gray-700` | 설명/서브타이틀 |
| `text-tertiary` | `gray-500` | 보조 설명, metadata |
| `text-tertiary-on-muted` | `gray-550` | gray background 위 metadata |
| `text-inverse` | `white` | primary CTA 텍스트 |

### 컬러 사용 규칙

- Gray Scale과 Accent/Feedback 색상의 비율은 **8:2**를 권장합니다.
- `400` 이하 컬러는 구조 요소(border, divider, background)에 우선 사용합니다.
- 텍스트는 white background에서 최소 `gray-500`, gray background에서 최소 `gray-550`을 사용합니다.
- background가 어두워지거나 gray surface 위에 올라가는 경우 텍스트/보더/아이콘 token을 1단계 이상 높입니다.
- Feedback color는 상태 의미가 있을 때만 사용하고, 색상만으로 상태를 전달하지 않습니다.

## 타이포그래피

기본 본문과 UI 텍스트는 Pretendard를 사용합니다. KT Flow는 브랜드/히어로용 display asset으로만 선택 적용하며, 일반 UI text에는 사용하지 않습니다.

| 역할 | 크기 | Line-height | 굵기 | 용도 |
|---|---:|---:|---:|---|
| Display 1 | 60px | 130% | 500/700 | PC 최상위 랜딩 headline |
| Display 2 | 40px | 130% | 500/700 | PC main title |
| Display 3 | 36px | 148% | 500/700 | 강조형 headline |
| Heading 1 | 32px | 148% | 500/700 | 페이지 상위 제목 |
| Heading 2 | 24px | 148% | 500/700 | 하위 제목 |
| Heading 3 | 22px | 148% | 500/700 | 소제목, 그룹 제목 |
| Title 1 | 20px | 148% | 500/700 | 콘텐츠 타이틀 |
| Title 2 | 18px | 148% | 500/600/700 | 서브 페이지 제목 |
| Body 1 | 16px | 148% / 170% | 400/500/600/700 | PC 권장 본문, 장문 |
| Body 2 | 15px | 148% / 158% | 400/500/600/700 | 기본 UI 본문 |
| Body 3 | 14px | 148% / 158% | 400/500/600/700 | 소형 본문 |
| Label 1 | 17px | 148% | 500 | PC 버튼 라벨 |
| Label 2 | 15px | 148% | 600 | 버튼, 입력 폼 |
| Label 3 | 13px | 148% | 500/600 | 입력 레이블 |
| Label 4 | 10px | auto | 500/600/700 | 바텀 아이콘 레이블 |
| Caption | 12px | 148% | 500/600 | 툴팁, 보조 설명 |

- 정보성 텍스트는 15px 이상을 기본으로 사용합니다.
- 모바일 최소 폰트는 12px, PC 최소 폰트는 14px입니다.
- 단문은 148%, 장문은 170% line-height를 권장합니다.
- 한 화면의 제목 hierarchy는 최대 3단계까지 사용합니다.

## 여백 & 간격

KT Seamless Flow의 4pt Grid 원칙에 따라 spacing은 4px 단위를 기본으로 정의합니다. 복잡한 컴포넌트 내부에서만 2의 배수 단위를 제한적으로 허용합니다.

| Token | 값 | 용도 |
|---|---:|---|
| `space-1` | 4px | icon/text 미세 간격, radio-label gap |
| `space-2` | 8px | 컴포넌트 요소 간 gap |
| `space-3` | 12px | tight stack, compact card 내부 gap |
| `space-4` | 16px | 컴포넌트 padding, 기본 section/card padding |
| `space-5` | 20px | margin, form group 간격 |
| `space-6` | 24px | 느슨한 section 간격 |
| `space-8` | 32px | 화면 블록 간격 |
| `page-mobile` | 16px | 모바일 페이지 padding |
| `page-tablet` | 24px | 태블릿 페이지 padding |
| `page-desktop` | 32px | 데스크탑 페이지 padding |

## Breakpoint & grid

| 구간 | 너비 | 레이아웃 기준 | 주요 규칙 |
|---|---:|---|---|
| Mobile | 0~639px | 단일 컬럼 | 터치 타깃 44px 이상, page padding 16px |
| Tablet | 640~1023px | 단일 컬럼 + 보조 카드 2열 가능 | page padding 24px, 카드 grid 2열 허용 |
| Desktop | 1024px 이상 | 중앙 컨테이너 + side panel | 콘텐츠 최대 폭 1120px, 리스트/상세 2열 |

## Radius & elevation

### Radius

| Token | 값 | 용도 |
|---|---:|---|
| `radius-sm` | 6px | tooltip, small badge |
| `radius-md` | 8px | button, input, dropdown 기본값 |
| `radius-lg` | 12px | 중첩 card/container |
| `radius-xl` | 16px | bottom sheet, large panel |
| `radius-pill` | `9999px` | chip, segmented indicator |

- 기본 값은 8px입니다.
- 지나치게 둥근 형태는 chip/indicator/pill처럼 의미가 있는 요소에만 사용합니다.
- 중첩 컴포넌트는 외부 radius보다 내부 radius를 작게 유지합니다.

### Elevation

| Level | 주요 구성 요소 | Semantic 적용 |
|---|---|---|
| Level 0 | Background | `surface-muted` |
| Level 1 | Chip, Button, Text Field, Card | `surface` / `shadow-1` |
| Level 2 | Notice, Canvas, Side Panel | `surface-raised` / `shadow-2` |
| Level 3 | Dropdown, floating tag/process indicator | `surface-raised` / `shadow-2` |
| Level 4 | Popup, Bottom Sheet, Navigation Bar | `surface-raised` / `shadow-3` + dim |

| Shadow token | 값 | 용도 |
|---|---|---|
| `shadow-1` | `0 2px 4px rgb(20 20 20 / 10%), 0 0 2px rgb(20 20 20 / 5%)` | input/select focused, 기본 interactive |
| `shadow-2` | `0 4px 8px rgb(20 20 20 / 10%), 0 0 4px rgb(20 20 20 / 5%)` | dropdown, floating card |
| `shadow-3` | `0 8px 12px rgb(20 20 20 / 20%), 0 0 8px rgb(20 20 20 / 5%)` | popup, modal, strong separation |
| `dim-65` | `rgb(20 20 20 / 65%)` | modal/sheet overlay |

## 컴포넌트 규칙

### Button

| Variant | 시각 규칙 | 사용처 |
|---|---|---|
| Primary | `primary` background, `text-inverse`, 48px mobile height | 가장 중요한 화면 진행 CTA |
| Secondary | `surface`, `primary` border/text | 보조 CTA, 되돌릴 수 있는 액션 |
| Tertiary/Ghost | 투명 배경, `text-secondary` | TopAppBar, 낮은 우선순위 액션 |
| Destructive | `red-50` background, `red-700` text | 거절, 삭제, 로그아웃 확인 |
| Icon Action | 44px 이상 정사각 touch target | 지도/필터/상단 액션 |

- Touch target은 최소 44px, 모바일 주요 CTA는 48px입니다.
- Loading 상태는 label을 유지하고 보조 loading indicator를 추가합니다.
- Disabled는 opacity만으로 의미를 전달하지 않고 helper text 또는 validation message와 함께 사용합니다.
- 화면당 Primary CTA는 하나만 둡니다.

### Input

| 상태 | 시각 규칙 |
|---|---|
| Default | 48px height, `border-input`, `radius-md`, label 필수 |
| Focus | `primary` ring/border + `shadow-1` |
| Error | `danger` border, error text `red-700` 또는 gray surface에서는 `danger-on-muted` |
| Disabled | `surface-secondary` background, `text-tertiary` |

- Placeholder는 예시이며 label을 대체하지 않습니다.
- 검색/경로 입력은 leading icon을 사용할 수 있습니다.

### Card & surface

- `surface` 배경, `border-default`, `radius-lg`, `shadow-1`을 기본으로 합니다.
- 클릭 가능한 카드는 `role="button"`, keyboard activation, hover/pressed 상태를 제공합니다.
- 리스트 카드는 내부 정보 hierarchy를 `title → route/time → price/seats/status` 순서로 유지합니다.
- 복잡한 정보가 있는 카드는 Gray-first 구조를 유지하고, 상태/CTA에만 accent를 사용합니다.

### Badge & Chip

| 상태 | 색상 |
|---|---|
| OPEN / available | `blue-50` background, `blue-700` text |
| MATCHED / COMPLETED / success | `teal-50` background, `teal-700` text |
| PENDING / waiting | `orange-50` background, `orange-700` text |
| FAILED / CANCELLED / rejected | `red-50` background, `red-700` text |
| Neutral metadata | `gray-100` background, `gray-700` text |

### Tab & navigation

- Container는 `surface-secondary` 배경 + 4px padding을 사용합니다.
- Active tab은 `surface` background, `shadow-1`, `text-primary`를 사용합니다.
- 접근성: `role="tablist"`, `role="tab"`, `aria-selected`를 유지합니다.
- Bottom navigation은 Level 4로 간주하고 mobile safe-area와 충돌하지 않게 합니다.

### Alert & feedback

- Error/Warning/Success/Info variant를 제공합니다.
- 문장 첫 줄은 원인, 둘째 줄 또는 CTA는 다음 행동을 설명합니다.
- Offline alert는 상단 fixed banner로 표시하되 콘텐츠 읽기를 막지 않습니다.
- Feedback color는 항상 상태 텍스트와 함께 사용합니다.

### TopAppBar

- Mobile: 56px height, 뒤로가기/제목/주요 action 1개 이하.
- Desktop: sticky header 또는 side navigation과 충돌하지 않게 사용합니다.
- Navigation bar는 Level 4 elevation을 갖되 shadow는 scroll 상태에서만 강화합니다.

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
- 카드 내부의 decorative color 사용을 줄이고 상태/CTA에만 accent를 사용합니다.

## 모션

- 화면 전환과 상태 변화는 사용자의 현재 위치와 다음 단계를 이해시키는 데 필요한 경우에만 사용합니다.
- 운영체제/브라우저의 `prefers-reduced-motion`을 감지해 간소화된 전환을 제공합니다.
- 과도한 속도, 반짝임, 갑작스러운 움직임을 지양합니다.
- 모션은 포커스된 요소 또는 방금 변경된 요소에 한정합니다.

## 아이콘

- 아이콘 세트: Lucide Icons
- 기본 크기: 20px
- 내비게이션/TopAppBar: 24px
- 입력 leading icon: 18px
- 장식 아이콘은 `aria-hidden="true"`를 사용합니다.
- 아이콘은 배경 대비 3:1 이상을 충족해야 합니다.

## 접근성 기준

- 본문 텍스트는 배경 대비 최소 4.5:1을 충족합니다.
- 18px Bold 이상 제목, CTA 버튼, 아이콘은 배경 대비 최소 3:1을 충족합니다.
- 색상만으로 상태를 전달하지 않습니다. 상태 텍스트, badge label, helper text를 함께 제공합니다.
- 모든 interactive element는 keyboard focus와 visible focus ring을 가져야 합니다.
- Mobile touch target은 최소 44px입니다.
- Form field는 label 또는 명시적인 `aria-label`을 가져야 합니다.
- 읽기/포커스 순서는 실제 시각 배치와 일치해야 합니다.
- 모달/sheet/dialog는 focus trap과 닫기 수단을 제공합니다.
- `prefers-reduced-motion` 환경에서는 과도한 transition을 제거합니다.

## 프론트엔드 구현 매핑

| 문서 token | Frontend target |
|---|---|
| primitive/semantic color | `frontend/app/globals.css` CSS custom properties + Tailwind v4 `@theme` |
| typography roles | `frontend/app/globals.css` `--text-*` theme tokens |
| spacing/radius/elevation | `frontend/app/globals.css`, `components/ui/*`, `components/ridy/*` |
| button/input/card pattern | `components/ui/button.tsx`, `components/ui/input.tsx`, `components/ui/card.tsx` |
| matching card pattern | `components/ridy/MatchingCard.tsx` |
| bottom navigation state | `components/ridy/BottomNavigation.tsx` |
| tab/auth segmented control | `components/auth/LoginForm.tsx` and future `Tabs` composition |
| common state/feedback | `components/ridy/StateView.tsx`, `components/ridy/AlertBanner.tsx` |

## 구현 제외 및 후속 검증

- KT Figma UI Kit, Zeroheight private assets, npm package direct adoption은 현재 제외합니다.
- KT Flow font는 라이선스와 asset 제공 경로가 확인되기 전까지 일반 UI에 적용하지 않습니다.
- Pixel-perfect clone은 목표가 아닙니다. 공개 가이드 기반 token/pattern migration이 목표입니다.
- Direct adoption을 검토하려면 별도 spike에서 라이선스, Tailwind v4 token import, bundle 영향, visual QA를 검증해야 합니다.
