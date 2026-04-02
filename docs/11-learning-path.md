## 학습 순서 추천

### Phase 1: 핵심 루프 이해 (1-2일)

| 순서 | 파일 | 목표 |
|------|------|------|
| 1 | `rust/crates/runtime/src/conversation.rs` | `run_turn` 메서드를 읽고 tool_use 루프 이해 |
| 2 | `rust/crates/api/src/types.rs` | `MessageRequest`와 `StreamEvent`로 API 형태 이해 |
| 3 | `src/query_engine.py` | Python으로 된 단순화된 버전으로 개념 복습 |

### Phase 2: 도구와 보안 (2-3일)

| 순서 | 파일 | 목표 |
|------|------|------|
| 4 | `rust/crates/runtime/src/bash.rs` | 실제 Bash 도구가 어떻게 실행되는지 |
| 5 | `rust/crates/runtime/src/file_ops.rs` | Read/Write/Edit/Glob/Grep 구현 |
| 6 | `rust/crates/runtime/src/permissions.rs` | 권한 모드, 규칙 매칭, 판단 흐름 |
| 7 | `rust/crates/runtime/src/hooks.rs` | 훅 실행 메커니즘, JSON 프로토콜 |

### Phase 3: 인프라 계층 (3-4일)

| 순서 | 파일 | 목표 |
|------|------|------|
| 8 | `rust/crates/api/src/client.rs` | 멀티 프로바이더 설계, SSE 스트리밍 |
| 9 | `rust/crates/runtime/src/prompt.rs` | 시스템 프롬프트 조립 로직 |
| 10 | `rust/crates/runtime/src/compact.rs` | 컨텍스트 압축 알고리즘 |
| 11 | `rust/crates/runtime/src/session.rs` | 세션 모델, 메시지 구조 |

### Phase 4: 확장 시스템 (4-5일)

| 순서 | 파일 | 목표 |
|------|------|------|
| 12 | `rust/crates/runtime/src/mcp_stdio.rs` | MCP 프로토콜, JSON-RPC, 서버 관리 |
| 13 | `rust/crates/runtime/src/mcp_client.rs` | 전송 계층 (stdio, remote, SDK) |
| 14 | `rust/crates/runtime/src/bootstrap.rs` | 시작 시퀀스, Fast Path 패턴 |
| 15 | `rust/crates/runtime/src/sandbox.rs` | 샌드박스 격리 |

### 학습 팁

- **Rust 코드를 먼저, Python을 보충으로**: Rust 코드가 실제 동작하는 구현이고, Python은 구조만 재현한 포팅입니다
- **테스트 코드를 함께 읽기**: 각 `.rs` 파일 하단의 `#[cfg(test)] mod tests`가 사용법의 최고의 문서입니다
- **`lib.rs`의 re-export 목록**: 각 크레이트의 `lib.rs`가 public API의 전체 목록입니다
- **trait부터 구현체로**: `ApiClient`, `ToolExecutor`, `PermissionPrompter` 같은 trait을 먼저 이해하면 전체 설계가 보입니다

