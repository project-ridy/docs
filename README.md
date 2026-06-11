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
└── agents/                # 에이전트 작업 지시서
    ├── AGENT_PROTOCOL.md  # 에이전트 협업 프로토콜
    ├── DEVELOPER_TASKS.md # 개발 에이전트 작업 큐
    └── DESIGNER_TASKS.md  # 디자인 에이전트 작업 큐
```

## 🤖 에이전트 작업 흐름

1. **Orchestrator**가 docs의 설계 문서를 기준으로 작업을 판단
2. `agents/` 폴더의 TASK 파일에 작업을 기록
3. Developer/Designer 에이전트가 각자 TASK 파일을 읽고 작업 수행
4. 완료 후 TASK 파일 업데이트

## 🔗 관련 레포

| 레포 | 용도 |
|---|---|
| [project-ridy/frontend](https://github.com/project-ridy/frontend) | 웹/앱 프론트엔드 |
| [project-ridy/backend](https://github.com/project-ridy/backend) | API 서버 |
| [project-ridy/docs](https://github.com/project-ridy/docs) | 이 레포 |

---

*Project by 손정원*
