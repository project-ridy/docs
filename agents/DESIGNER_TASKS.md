# Ridy — 디자인 에이전트 작업 큐

> Orchestrator가 이 파일에 작업을 추가합니다. Designer 에이전트는 여기서 작업을 가져가 수행합니다.

## 현재 작업

### TASK-D001: 디자인 시스템 컴포넌트 구현
- **상태**: PENDING
- **담당**: designer
- **레포**: frontend
- **우선순위**: HIGH
- **의존성**: 없음
- **설명**: DESIGN_SYSTEM.md에 정의된 컴포넌트를 React 컴포넌트로 구현. Button, Card, Input, MatchingCard, BottomNav 등
- **완료 조건**:
  - Storybook 또는 페이지에서 각 컴포넌트 확인 가능
  - 디자인 토큰 (색상, 타이포, 여백) 준수
  - 반응형 (모바일/태블릿/데스크탑) 대응
- **참고 문서**: docs/design/DESIGN_SYSTEM.md

### TASK-D002: 로그인/온보딩 화면 디자인 구체화
- **상태**: PENDING
- **담당**: designer
- **레포**: frontend
- **우선순위**: HIGH
- **의존성**: TASK-D001
- **설명**: WIREFRAMES.md의 로그인/온보딩 와이어프레임을 실제 UI로 구체화. 스플래시, 소셜 로그인, 프로필 설정 화면
- **완료 조건**:
  - 3개 화면의 상세 UI 코드 (HTML/CSS/React)
  - 모바일/데스크탑 대응
  - 애니메이션/트랜지션 정의
- **참고 문서**: docs/design/WIREFRAMES.md, docs/design/SCREENS.md

### TASK-D003: 홈 화면 디자인 구체화
- **상태**: PENDING
- **담당**: designer
- **레포**: frontend
- **우선순위**: MEDIUM
- **의존성**: TASK-D001
- **설명**: 탑승자/차주 홈 화면 UI 구체화. 출발지/도착지 입력, 정기 카풀 카드, 바텀 내비게이션
- **완료 조건**:
  - 탑승자 홈 화면 UI
  - 차주 홈 화면 UI
  - 빈 상태 / 데이터 있음 상태 모두 구현
- **참고 문서**: docs/design/WIREFRAMES.md, docs/design/SCREENS.md

### TASK-D004: 매칭 결과/상세 화면 디자인
- **상태**: PENDING
- **담당**: designer
- **레포**: frontend
- **우선순위**: MEDIUM
- **의존성**: TASK-D001
- **설명**: 매칭 검색 결과 리스트, 매칭 상세 프로필 화면 UI
- **완료 조건**:
  - 매칭 카드 리스트 UI
  - 매칭 상세 화면 (프로필, 경로 지도, 요청 버튼)
  - 지도 컴포넌트 포함
- **참고 문서**: docs/design/WIREFRAMES.md, docs/design/SCREENS.md

### TASK-D005: 채팅 화면 디자인
- **상태**: PENDING
- **담당**: designer
- **레포**: frontend
- **우선순위**: MEDIUM
- **의존성**: TASK-D001
- **설명**: 채팅방 목록, 실시간 채팅 화면 UI
- **완료 조건**:
  - 채팅방 리스트 UI
  - 채팅 화면 (메시지 버블, 입력창, 타이핑 표시)
  - 시스템 메시지 스타일
- **참고 문서**: docs/design/WIREFRAMES.md, docs/design/SCREENS.md

---

## 완료된 작업

(없음)
