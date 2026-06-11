# Ridy Docs 📚

**함께 타는 길, Ridy** — 카풀 매칭 서비스의 단일 진실 공급원(Single Source of Truth)

이 레포는 Ridy 프로젝트의 모든 설계 문서를 관리합니다. 개발 에이전트(developer, designer)는 이 문서를 기준으로 작업합니다.

## 📂 문서 구조

```
docs/
├── README.md              # 이 파일 — 문서 인덱스
├── planning/              # 기획 문서
│   ├── PLANNING.md        # 프로젝트 기획서 (비전, 타겟, 마일스톤)
│   ├── PERSONAS.md        # 유저 페르소나
│   └── ROADMAP.md         # 상세 로드맵 & 마일스톤
├── api/                   # API 설계
│   ├── API.md             # 전체 API 스펙 개요
│   ├── AUTH.md            # 인증/인가 API
│   ├── MATCHING.md        # 매칭 API
│   ├── CHAT.md            # 채팅 API
│   └── PAYMENT.md         # 정산/결제 API
├── design/                # 디자인 시스템 & 가이드
│   ├── DESIGN_SYSTEM.md   # 디자인 시스템 (색상, 타이포, 컴포넌트)
│   ├── WIREFRAMES.md      # 와이어프레임 설명
│   └── SCREENS.md         # 화면 정의서
├── architecture/          # 기술 아키텍처
│   ├── ARCHITECTURE.md    # 전체 시스템 아키텍처
│   ├── DATABASE.md        # DB 스키마 & ERD
│   └── INFRA.md           # 인프라 구성도
├── plans/                 # 개발 기획서 (Planner 에이전트 산출물)
│   └── README.md          # 기획서 포맷 & 가이드
└── agents/                # 에이전트 관련 문서
    ├── AGENT_SPEC.md      # 에이전트 스펙 (역할, 능력, 제약)
    ├── AGENT_PROTOCOL.md  # 에이전트 협업 프로토콜
    ├── DEVELOPER_TASKS.md # 개발 에이전트 작업 큐
    └── DESIGNER_TASKS.md  # 디자인 에이전트 작업 큐
```

## 🤖 에이전트

| 에이전트 | 역할 | 담당 레포 | 상세 |
|---|---|---|---|
| **Orchestrator** | 전체 관리, 작업 분배, 품질 관리 | docs | [AGENT_SPEC.md](agents/AGENT_SPEC.md) |
| **Planner** | 사용자 요청 분석, 개발 기획서 작성 | docs (plans/) | [AGENT_SPEC.md](agents/AGENT_SPEC.md) |
| **Developer** | 프론트엔드/백엔드 구현 | frontend, backend | [AGENT_SPEC.md](agents/AGENT_SPEC.md) |
| **Designer** | UI/UX 디자인, 접근성 | frontend | [AGENT_SPEC.md](agents/AGENT_SPEC.md) |

### 작업 흐름

**모든 작업은 GitHub Organization Project에서 관리됩니다.**
📍 프로젝트 보드: https://github.com/orgs/project-ridy/projects/1

1. **Orchestrator**가 docs에 설계를 완료하고 Project에 이슈를 생성
2. 서브 에이전트(Developer/Designer)에게 `delegate_task`로 이슈 할당
3. 서브 에이전트가 이슈 내용 + docs 문서를 읽고 TDD 사이클로 작업
4. PR 생성 → 머지 → Project 이슈 Done 처리
5. 기능 작업 후 후속 테스트 이슈 진행

## 🔗 관련 레포

| 레포 | 용도 |
|---|---|
| [project-ridy/frontend](https://github.com/project-ridy/frontend) | 웹/앱 프론트엔드 (Next.js + shadcn/ui) |
| [project-ridy/backend](https://github.com/project-ridy/backend) | API 서버 (NestJS + Prisma) |
| [project-ridy/docs](https://github.com/project-ridy/docs) | 이 레포 |

---

*Project by 손정원*
