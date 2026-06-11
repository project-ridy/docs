# Ridy Docs 📚

**함께 타는 출퇴근, Ridy** — 기업 단위 폐쇄형 카풀 매칭 서비스의 설계 문서 저장소

같은 회사 사원끼리만 출퇴근 방향이 맞으면 카풀할 수 있도록 연결합니다. 초대 코드 기반 가입으로 외부인을 차단하고, 사내 검증된 환경에서 안전한 카풀을 제공합니다.

이 레포는 Ridy 프로젝트의 제품 기획, API 스펙, 디자인 시스템, 아키텍처 등 **설계 문서**만 관리합니다. 에이전트 스펙/프로토콜/작업 큐/개발 기획서는 [`project-ridy/agents`](https://github.com/project-ridy/agents) 레포에서 별도로 관리합니다.

## 📂 문서 구조

```
docs/
├── README.md              # 이 파일 — 문서 인덱스
├── planning/              # 제품 기획 문서
│   ├── PLANNING.md        # 프로젝트 기획서 (비전, 타겟, 마일스톤)
│   ├── PERSONAS.md        # 유저 페르소나
│   └── ROADMAP.md         # 상세 로드맵 & 마일스톤
├── api/                   # API 설계
│   ├── API.md             # 전체 API 스펙 개요
│   ├── AUTH.md            # 인증/인가 API (초대 코드 포함)
│   ├── MATCHING.md        # 매칭 API (같은 회사 사원만)
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
└── assets/                # 로고, 이미지, 파일 에셋
```

## 🤖 에이전트 문서

에이전트 관련 문서는 `agents` 레포로 분리했습니다.

| 문서 | 위치 |
|---|---|
| 에이전트 스펙 | [project-ridy/agents/spec/AGENT_SPEC.md](https://github.com/project-ridy/agents/blob/main/spec/AGENT_SPEC.md) |
| 에이전트 프로토콜 | [project-ridy/agents/protocol/AGENT_PROTOCOL.md](https://github.com/project-ridy/agents/blob/main/protocol/AGENT_PROTOCOL.md) |
| 개발 기획서 | [project-ridy/agents/plans/](https://github.com/project-ridy/agents/tree/main/plans) |
| 작업 큐 | [project-ridy/agents/tasks/](https://github.com/project-ridy/agents/tree/main/tasks) |

## 작업 흐름

**모든 작업은 GitHub Organization Project에서 관리됩니다.**
📍 프로젝트 보드: https://github.com/orgs/project-ridy/projects/1

1. 사용자 요청 → `agents` 레포의 Planner가 개발 기획서 작성
2. Orchestrator가 이 설계 문서(`docs`)를 갱신하거나 확인
3. 기획서 + 설계 문서를 기준으로 Frontend Developer/Backend Developer/Designer가 구현
4. PR 생성 → 머지 → Project 이슈 Done 처리

## 🔗 관련 레포

| 레포 | 용도 |
|---|---|
| [project-ridy/agents](https://github.com/project-ridy/agents) | 에이전트 스펙, 프로토콜, 기획서, 작업 큐 |
| [project-ridy/frontend](https://github.com/project-ridy/frontend) | 웹/앱 프론트엔드 (Next.js + shadcn/ui) |
| [project-ridy/backend](https://github.com/project-ridy/backend) | API 서버 (NestJS + Prisma) |
| [project-ridy/docs](https://github.com/project-ridy/docs) | 설계 문서 (이 레포) |

---

*Project by 손정원*
