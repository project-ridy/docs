# Ridy — 개발 에이전트 작업 큐

> Orchestrator가 이 파일에 작업을 추가합니다. Developer 에이전트는 여기서 작업을 가져가 수행합니다.

## 현재 작업

### TASK-001: 백엔드 프로젝트 셋업
- **상태**: PENDING
- **담당**: developer
- **레포**: backend
- **우선순위**: HIGH
- **의존성**: 없음
- **설명**: NestJS 프로젝트 초기 셋업. Prisma, ESLint, Prettier, 환경변수 설정 포함
- **완료 조건**: 
  - `npm run start:dev` 실행 시 서버 정상 구동
  - Prisma 스키마 초기 파일 존재
  - health check 엔드포인트 (`GET /health`) 정상 응답
- **참고 문서**: docs/architecture/ARCHITECTURE.md

### TASK-002: 프론트엔드 프로젝트 셋업
- **상태**: PENDING
- **담당**: developer
- **레포**: frontend
- **우선순위**: HIGH
- **의존성**: 없음
- **설명**: Next.js 15 (App Router) 프로젝트 초기 셋업. Tailwind CSS, Pretendard 폰트, 디자인 토큰 설정
- **완료 조건**:
  - `npm run dev` 실행 시 개발 서버 구동
  - Tailwind CSS 정상 동작
  - Pretendard 폰트 적용
  - 디자인 시스템 토큰 (색상, 타이포) Tailwind config에 정의
- **참고 문서**: docs/design/DESIGN_SYSTEM.md

### TASK-003: 인증 API 구현
- **상태**: PENDING
- **담당**: developer
- **레포**: backend
- **우선순위**: HIGH
- **의존성**: TASK-001
- **설명**: 소셜 로그인 (카카오/구글/Apple) 인증 API 구현. JWT 발급/갱신, 휴대폰 인증
- **완료 조건**:
  - POST /auth/login 정상 동작
  - POST /auth/refresh 정상 동작
  - JWT 검증 미들웨어 동작
  - 유닛 테스트 통과
- **참고 문서**: docs/api/AUTH.md, docs/architecture/DATABASE.md

### TASK-004: DB 스키마 구현
- **상태**: PENDING
- **담당**: developer
- **레포**: backend
- **우선순위**: HIGH
- **의존성**: TASK-001
- **설명**: Prisma 스키마 작성 및 마이그레이션. users, rides, ride_requests, chat_rooms, messages, settlements, reviews, vehicles, payment_methods 테이블
- **완료 조건**:
  - `npx prisma migrate dev` 성공
  - `npx prisma generate` 성공
  - 스키마가 DATABASE.md와 일치
- **참고 문서**: docs/architecture/DATABASE.md

### TASK-005: 로그인 화면 구현
- **상태**: PENDING
- **담당**: developer
- **레포**: frontend
- **우선순위**: MEDIUM
- **의존성**: TASK-002
- **설명**: 소셜 로그인 화면 UI 구현. 카카오/구글/Apple 로그인 버튼, 스플래시
- **완료 조건**:
  - 로그인 화면 렌더링
  - 소셜 로그인 버튼 클릭 시 인증 플로우 진입
  - 디자인 시스템 준수
- **참고 문서**: docs/design/WIREFRAMES.md, docs/design/DESIGN_SYSTEM.md

---

## 완료된 작업

(없음)
