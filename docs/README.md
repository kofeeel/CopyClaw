# CLI AI 하네스 엔지니어링 학습 가이드

> [claw-code](https://github.com/ultraworkers/claw-code-parity) 코드베이스를 기반으로,
> Claude Code와 같은 CLI 기반 AI 도구의 내부 아키텍처를 탑다운 방식으로 설명합니다.
>
> 모든 코드 참조는 실제 레포의 파일 경로, 구조체명, 함수명과 대조 검증되었습니다.

---

## 목차

### 기초 개념

| # | 문서 | 내용 |
|---|------|------|
| 00 | [큰 그림 — CLI AI의 정체](00-big-picture.md) | Harness가 뭔지, LLM과 도구의 관계 |
| 01 | [Conversation Loop — 핵심 동작 원리](01-conversation-loop.md) | tool_use → execute → tool_result 루프, ConversationRuntime 구조체 |

### 핵심 서브시스템 (Level 2)

| # | 문서 | 내용 |
|---|------|------|
| 02 | [API Client — LLM과의 통신](02-api-client.md) | 멀티 프로바이더, SSE 스트리밍, Prompt Cache |
| 03 | [Tool System — AI가 세상과 상호작용하는 방법](03-tool-system.md) | 도구 정의 JSON Schema, tool_use/tool_result 프로토콜 |
| 04 | [Permission System — 안전장치](04-permission-system.md) | 권한 모드, 정책, Rule 기반 제어 |
| 05 | [Hook System — 사용자 확장 포인트](05-hook-system.md) | PreToolUse/PostToolUse, 셸 스크립트 실행, JSON 프로토콜 |
| 06 | [Bootstrap — 시작 시퀀스](06-bootstrap.md) | 단계별 초기화, Fast Path 패턴 |
| 07 | [Session & Compaction — 대화 관리](07-session-compaction.md) | 세션 구조, 컨텍스트 압축 알고리즘 |
| 08 | [MCP — 외부 도구 연결](08-mcp.md) | Model Context Protocol, JSON-RPC, 서버 관리 |

### 심화 학습 (Level 3-4)

| # | 문서 | 내용 |
|---|------|------|
| 09 | [데이터 흐름 — 한 턴의 생애주기](09-data-flow.md) | 11단계 실행 흐름, 실제 예시 |
| 10 | [코드베이스 매핑](10-codebase-map.md) | 디렉토리 구조, 개념-파일 매핑 테이블 |

### 부록

| # | 문서 | 내용 |
|---|------|------|
| 11 | [학습 순서 추천](11-learning-path.md) | Phase 1-4, 15개 핵심 파일 순서 |
| 12 | [용어 정리](12-glossary.md) | 18개 핵심 용어 (한국어 + 영문) |

---

## 추천 학습 순서

```
Phase 1 (핵심 루프)     00 → 01 → 09
Phase 2 (도구와 보안)    03 → 04 → 05
Phase 3 (인프라)        02 → 07 → 06
Phase 4 (확장)          08 → 10 → 11
```

---

## 참고

- **코드 출처**: [claw-code-parity](https://github.com/ultraworkers/claw-code-parity)
- **원본 레포**: [claw-code](https://github.com/ultraworkers/claw-code) (현재 소유권 이전 중)
- 모든 줄 번호와 구조체명은 claw-code-parity main 브랜치 기준으로 검증됨
