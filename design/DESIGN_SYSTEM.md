# Ridy — 디자인 시스템

## 브랜드 아이덴티티

- **슬로건**: 함께 타는 길
- **키워드**: 안전, 편리, 친환경, 연결

## 컬러 팔레트

| 이름 | Hex | 용도 |
|---|---|---|
| Primary | `#2563EB` (Blue 600) | 메인 CTA, 링크, 브랜드 |
| Primary Light | `#3B82F6` (Blue 500) | 호버, 액티브 |
| Primary Dark | `#1D4ED8` (Blue 700) | 프레스 |
| Secondary | `#10B981` (Emerald 500) | 성공, 매칭 완료, 친환경 |
| Warning | `#F59E0B` (Amber 500) | 주의, 대기 |
| Danger | `#EF4444` (Red 500) | 에러, 거절, 삭제 |
| Gray 50 | `#F9FAFB` | 배경 |
| Gray 100 | `#F3F4F6` | 카드 배경 |
| Gray 500 | `#6B7280` | 보조 텍스트 |
| Gray 900 | `#111827` | 메인 텍스트 |

## 타이포그래피

| 역할 | 크기 | 굵기 | 폰트 |
|---|---|---|---|
| H1 | 28px | Bold | Pretendard |
| H2 | 24px | Bold | Pretendard |
| H3 | 20px | SemiBold | Pretendard |
| Body | 16px | Regular | Pretendard |
| Caption | 14px | Regular | Pretendard |
| Small | 12px | Regular | Pretendard |

## 컴포넌트

### 버튼
- **Primary**: 파란색 배경, 흰색 텍스트, rounded-lg (8px)
- **Secondary**: 흰색 배경, 파란색 보더
- **Ghost**: 배경 없음, 텍스트만
- 높이: 48px (모바일), 40px (데스크탑)

### 카드
- rounded-xl (12px), shadow-sm
- 패딩: 16px
- 배경: white, 보더: gray-100

### 입력 필드
- 높이: 48px
- 보더: gray-300 → focus 시 primary
- rounded-lg

### 매칭 카드 (핵심 컴포넌트)
```
┌──────────────────────────────┐
│ 🚗 박준서 ⭐ 4.8  (42회)     │
│                              │
│ 강남역 08:30  →  수원역 09:20│
│                              │
│ 💰 5,000원/인  🪑 2/3석      │
│                              │
│ [요청하기]                    │
└──────────────────────────────┘
```

## 아이콘

- 아이콘 세트: Lucide Icons
- 크기: 20px (기본), 24px (내비게이션)

## 여백 & 간격

- 페이지 패딩: 16px (모바일), 24px (태블릿), 32px (데스크탑)
- 컴포넌트 간격: 12px (tight), 16px (normal), 24px (loose)
