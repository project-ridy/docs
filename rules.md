# Rules — Ridy Docs

이 파일은 docs 레포의 문서 작성 규칙과 컨벤션을 정의합니다.

---

## 마크다운

### 구조
- **H1(`#`)은 파일당 1개** — 파일 제목으로만 사용
- **H2(`##`)부터 섹션 분할** — 논리적 계층 유지
- **H3(`###`)는 세부 항목** — H2 아래에만 사용
- **H4 이하는 피하기** — 구조가 깊어지면 파일 분리 검토

### 서식
- **코드 블록에 언어 명시** — ` ```typescript`, ` ```json`, ` ```prisma`, ` ```bash`
- **인라인 코드는 백틱** — `RideStatus`, `prisma.$transaction`
- **강조는 볼드** — **필수**, **금지**, **주의**
- **이모지는 제목에만** — 본문에서는 과도한 이모지 자제
- **링크는 상대 경로** — 같은 레포 내 문서는 `[API 스펙](../api/AUTH.md)`

### 표
- **가독성 우선** — 열이 5개 이상이면 세로 표로 전환
- **헤더 행은 볼드** — `| **항목** | **설명** |`
- **정렬 맞춤** — 파이프라인 정렬로 가독성 확보
- **빈 셀은 `-`** — `| - |` 로 명시적 표시

```markdown
✅ Good — 세로 표 (열이 많을 때)
| 필드 | 타입 | 필수 | 설명 | 기본값 |
|---|---|---|---|---|
| id | string | ✅ | 고유 식별자 | uuid() |

✅ Good — 가로 표 (간단한 비교)
| 항목 | 설명 |
|---|---|
| 인증 | JWT 기반 |
| DB | PostgreSQL |
```

---

## API 문서

### 엔드포인트 형식
모든 엔드포인트는 **HTTP 메서드 + 경로**로 시작:

```markdown
### POST /auth/login

소셜 제공자의 토큰으로 로그인/회원가입 처리

**Request:**
```json
{
  "provider": "kakao" | "google" | "apple",
  "accessToken": "string"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "string",
    "refreshToken": "string",
    "user": { ... }
  }
}
```

**Error Responses:**
| 상태 코드 | 에러 코드 | 설명 |
|---|---|---|
| 401 | INVALID_TOKEN | 유효하지 않은 소셜 토큰 |
| 400 | MISSING_FIELDS | 필수 필드 누락 |
```

### API 문서 규칙
- **HTTP 메서드는 대문자** — `POST`, `GET`, `PUT`, `DELETE`, `PATCH`
- **경로는 kebab-case** — `/ride-requests`, `/matchings/search`
- **경로 파라미터는 camelCase** — `/matchings/:matchingId`
- **쿼리 파라미터는 camelCase** — `?departureTime=...&seats=2`
- **Request/Response는 JSON 코드 블록**
- **필수/선택 필드 명시** — 필수: `✅`, 선택: `optional`
- **에러 응답은 테이블** — 가능한 모든 에러 코드 포함
- **페이지네이션 응답 규격** — `page`, `limit`, `total` 포함

### 공통 응답 형식

```json
// 성공
{
  "success": true,
  "data": { ... }
}

// 페이지네이션
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}

// 에러
{
  "success": false,
  "error": {
    "code": 401,
    "message": "유효하지 않은 토큰입니다"
  }
}
```

---

## 데이터베이스 문서

### 테이블 정의 형식

```markdown
### users

| 컬럼 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | UUID | PK, DEFAULT uuid() | 고유 식별자 |
| email | VARCHAR(255) | UNIQUE, NOT NULL | 이메일 |
| name | VARCHAR(100) | NOT NULL | 이름 |
| created_at | TIMESTAMP | DEFAULT now() | 생성 시간 |
| updated_at | TIMESTAMP | ON UPDATE | 수정 시간 |

**인덱스:**
| 이름 | 컬럼 | 타입 |
|---|---|---|
| idx_users_email | email | UNIQUE |
| idx_users_created_at | created_at | BTREE |
```

### ERD 표기
- **1:N** — `user ||--o{ ride : "운행"`
- **1:1** — `user ||--|| profile : "프로필"`
- **N:M** — 중간 테이블로 표현

### 스키마 변경 시
1. `DATABASE.md`에 테이블/컬럼 정의 추가
2. `ARCHITECTURE.md`의 ERD 다이어그램 업데이트
3. 관련 API 스펙에 필드 추가/변경 반영
4. Developer 에이전트 작업 큐에 마이그레이션 이슈 추가

---

## 디자인 문서

### 디자인 시스템 토큰 형식

```markdown
| 이름 | Hex | 용도 | WCAG 대비 |
|---|---|---|---|
| Primary | `#2563EB` | 메인 CTA | AA ✅ (흰 텍스트) |
| Primary Light | `#3B82F6` | 호버 | AA ✅ |
```

- **WCAG 대비율 명시** — 접근성 검증 필수
- **Tailwind 클래스 매핑** — `bg-blue-600`, `text-blue-500` 등
- **다크모드 변형** — 있는 경우 별도 행으로 정의

### 화면 정의서 형식

```markdown
### S01: 스플래시

- **경로**: `/`
- **인증**: 불필요
- **레이아웃**: 전체 화면
- **구성 요소**:
  - 로고 (중앙)
  - 로딩 인디케이터
- **이동**: 자동 → S02 로그인
```

### 와이어프레임 형식
- **ASCII 아트** — 박스 다이어그램으로 레이아웃 표현
- **화면 ID 연결** — `[S04]` 형식으로 화면 정의서와 연결
- **상태 표시** — 기본/빈/에러/로딩 상태별 와이어프레임

---

## 아키텍처 문서

### 다이어그램
- **ASCII 아트** — 텍스트 기반 다이어그램으로 버전 관리 용이
- **Mermaid** — 복잡한 흐름도는 Mermaid 문법 사용
- **컴포넌트명은 PascalCase** — `AuthService`, `MatchingGateway`
- **화살표 방향** — 의존성 방향 (A → B: A가 B에 의존)

### 아키텍처 결정 기록 (ADR)

중요한 기술 결정은 다음 형식으로 기록:

```markdown
### ADR-001: WebSocket 라이브러리 선택

- **상태**: 수락됨
- **결정**: Socket.IO 사용
- **이유**: 자동 재연결, 룸 기능, 네임스페이스 지원
- **대안**: 원시 WebSocket, Server-Sent Events
- **결과**: 클라이언트 호환성 좋음, 서버 메모리 사용량 증가
```

---

## 문서 간 동기화

### 변경 전 체크리스트
API 스펙을 변경할 때:
- [ ] `DATABASE.md`에 해당 필드/테이블이 있는지 확인
- [ ] `ARCHITECTURE.md`에 영향이 없는지 확인
- [ ] `SCREENS.md`에 해당 화면이 있는지 확인
- [ ] 구현 작업 큐가 필요한 경우 `project-ridy/agents` 레포에 반영했는지 확인

디자인을 변경할 때:
- [ ] `DESIGN_SYSTEM.md` 토큰이 업데이트되었는지 확인
- [ ] `WIREFRAMES.md`에 반영되었는지 확인
- [ ] `SCREENS.md` 화면 정의가 일치하는지 확인
- [ ] 프론트엔드 CONTRIBUTING.md에 영향이 없는지 확인

DB 스키마를 변경할 때:
- [ ] `DATABASE.md` 테이블 정의 업데이트
- [ ] 관련 API 스펙(`api/`)에 필드 반영
- [ ] `ARCHITECTURE.md` ERD 업데이트
- [ ] 백엔드 CONTRIBUTING.md Prisma 규칙에 영향 없는지 확인

---

## 파일 네이밍

| 대상 | 규칙 | 예시 |
|---|---|---|
| 기획 문서 | UPPER_SNAKE.md | `PLANNING.md`, `ROADMAP.md` |
| API 스펙 | UPPER_SNAKE.md | `AUTH.md`, `MATCHING.md` |
| 디자인 가이드 | UPPER_SNAKE.md | `DESIGN_SYSTEM.md`, `WIREFRAMES.md` |
| 아키텍처 | UPPER_SNAKE.md | `ARCHITECTURE.md`, `DATABASE.md` |
| 에셋 | kebab-case.ext | `ridy-logo.png`, `ridy-logo.svg` |

---

## 커밋 컨벤션

```
docs(<scope>): <subject>

Closes #<이슈번호>
```

| scope | 설명 |
|---|---|
| planning | 기획 문서 |
| api | API 스펙 |
| design | 디자인 가이드 |
| arch | 아키텍처 |
| assets | 이미지/파일 에셋 |

**예시:**
```
docs(api): 매칭 API 응답에 detourMinutes 필드 추가
docs(design): Primary 컬러 WCAG 대비율 검증 결과 추가
docs(arch): Socket.IO 게이트웨이 아키텍처 다이어그램 추가
```
