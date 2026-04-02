### 2.6 Session & Compaction — 대화 관리

**파일**: `rust/crates/runtime/src/compact.rs`, `rust/crates/runtime/src/session.rs`, `src/query_engine.py`

#### 세션 구조

```rust
// session.rs — 세션은 메시지의 배열
pub struct Session {
    pub messages: Vec<ConversationMessage>,
    // ...
}

pub struct ConversationMessage {
    pub role: MessageRole,         // User, Assistant
    pub blocks: Vec<ContentBlock>, // 텍스트, tool_use, tool_result 등
}

pub enum ContentBlock {
    Text(String),
    ToolUse { id: String, name: String, input: String },
    ToolResult { tool_use_id: String, content: String, is_error: bool },
}
```

#### 컨텍스트 압축 (Compaction)

LLM의 context window(입력 크기 제한)는 유한합니다. 대화가 길어지면 오래된 메시지를 요약해서 토큰을 절약해야 합니다.

```rust
// compact.rs (line 9)
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,  // 최근 N개 메시지는 보존
    pub max_estimated_tokens: usize,      // 이 토큰 수 초과 시 압축 시작
}
```

자동 압축 임계값:

```rust
// conversation.rs (line 18)
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
// 환경변수로 오버라이드 가능:
const AUTO_COMPACTION_THRESHOLD_ENV_VAR: &str = "CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS";
```

압축 결과:

```rust
// compact.rs (line 23)
pub struct CompactionResult {
    pub summary: String,                       // 압축된 요약 텍스트
    pub formatted_summary: String,             // 포맷된 요약
    pub compacted_session: Session,            // 압축 후 세션
    pub removed_message_count: usize,          // 제거된 메시지 수
}
```

`run_turn`의 마지막에서 `maybe_auto_compact()`가 호출되어, 토큰이 임계값을 초과하면 자동으로 오래된 메시지를 요약합니다.

#### Python 포팅의 세션 관리

```python
# src/query_engine.py (line 15)
@dataclass(frozen=True)
class QueryEngineConfig:
    max_turns: int = 8                # 최대 턴 수
    max_budget_tokens: int = 2000     # 토큰 예산
    compact_after_turns: int = 12     # 이 턴 수 초과 시 압축
    structured_output: bool = False   # 구조화된 출력 여부
    structured_retry_limit: int = 2   # 구조화 출력 재시도 한도
