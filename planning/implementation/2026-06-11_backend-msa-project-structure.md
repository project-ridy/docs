# 백엔드 MSA 프로젝트 구조 전환 구현 계획

## 개요

Ridy 백엔드를 GraphQL Gateway 중심 MSA 전환이 가능한 모듈형 모놀리스 구조로 재배치한다.

## 참고 문서

- `docs/architecture/MSA.md`
- `docs/api/GRAPHQL_GATEWAY.md`
- `backend/.hermes/plans/2026-06-11_msa-project-structure.md`

## 구현 범위

- `src/app` 루트 조립 계층 추가
- `src/gateway/graphql` GraphQL Gateway 계층 추가
- `src/common` 공통 context/event/health 계층 추가
- `src/services/*` bounded context 계층 추가
- 기존 `invite-code` 기능을 Company bounded context로 이동
- GraphQL SDL을 Gateway 공통 스펙 기준으로 확장

## 검증

- 구조 테스트 추가 후 Red 확인
- `npm run codegen`
- `npm run test`
- `npm run build`
