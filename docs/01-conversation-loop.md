## Level 1: Conversation Loop — 핵심 동작 원리

CLI AI에서 가장 중요한 개념은 **대화 루프(Conversation Loop)** 입니다.

### 해당 코드

- **Rust**: `rust/crates/runtime/src/conversation.rs` — `ConversationRuntime::run_turn` 메서드 (line 286)
- **Python**: `src/query_engine.py` — `QueryEnginePort.submit_message` 메서드 (line 61)

### 루프의 구조

```
사용자 입력: "src/main.py의 버그 고쳐줘"
       │
       ▼
  ┌─────────────┐
  │ System      │  ← "너는 소프트웨어 엔지니어를 돕는 AI야.
  │ Prompt 조립  │     다음 도구를 쓸 수 있어: Read, Bash, Edit..."
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │ API 호출     │  ← system_prompt + messages[] + tools[] 를 API에 전송
  │ (Streaming)  │     SSE(Server-Sent Events)로 실시간 응답 수신
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────┐
  │ 응답에 tool_use가    │──── 있음 ──▶ 도구 실행 ──▶ 결과를 messages에 추가
  │ 포함되어 있는가?      │                              │
  └─────────────────────┘                              ▼
         │                                      다시 API 호출 (루프!)
         │
         └──── 없음 (텍스트만) ──▶ 사용자에게 출력 ──▶ 턴 종료
```

### Rust 코드로 보는 핵심 구조

`ConversationRuntime`은 대화 루프를 구동하는 핵심 구조체입니다:

```rust
// rust/crates/runtime/src/conversation.rs (line 116)
pub struct ConversationRuntime<C, T> {
    session: Session,                    // 대화 이력 (메시지 배열)
    api_client: C,                       // LLM API 호출 (trait ApiClient)
    tool_executor: T,                    // 도구 실행 (trait ToolExecutor)
    permission_policy: PermissionPolicy, // 권한 관리
    system_prompt: Vec<String>,          // 시스템 프롬프트
    max_iterations: usize,              // 최대 반복 횟수 (무한루프 방지)
    usage_tracker: UsageTracker,         // 토큰 사용량 추적
    hook_runner: HookRunner,             // 훅 실행기
    auto_compaction_input_tokens_threshold: u32, // 자동 압축 임계값
    hook_abort_signal: HookAbortSignal,  // 훅 취소 시그널
    hook_progress_reporter: Option<Box<dyn HookProgressReporter>>,
    session_tracer: Option<SessionTracer>,
}
```

**제네릭 `<C, T>`의 설계 의도**: API 클라이언트(`C`)와 도구 실행기(`T`)를 trait(인터페이스)으로 분리합니다. 이렇게 하면 테스트 시 실제 API를 호출하지 않고 mock 객체를 넣을 수 있습니다.

```rust
// line 49 — API 클라이언트 인터페이스
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

// line 53 — 도구 실행기 인터페이스
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}
```

### run_turn 메서드의 실제 흐름

`run_turn` (line 286)의 핵심 로직을 단순화하면:

```rust
pub fn run_turn(&mut self, user_input: String) -> Result<TurnSummary, RuntimeError> {
    // 1. 사용자 메시지를 세션에 추가
    self.session.push_user_text(user_input);

    loop {
        // 2. API 호출 — 시스템 프롬프트 + 전체 대화 이력 전송
        let request = ApiRequest {
            system_prompt: self.system_prompt.clone(),
            messages: self.session.messages.clone(),
        };
        let events = self.api_client.stream(request)?;

        // 3. 응답에서 tool_use 블록 추출
        let assistant_message = build_assistant_message(events)?;
        let pending_tool_uses = /* assistant_message에서 ToolUse 블록만 필터 */;

        // 4. 어시스턴트 메시지를 세션에 추가
        self.session.push_message(assistant_message);

        // 5. tool_use가 없으면 루프 종료 (텍스트만 응답한 것)
        if pending_tool_uses.is_empty() {
            break;
        }

        // 6. 각 도구 호출을 순서대로 실행
        for (tool_use_id, tool_name, input) in pending_tool_uses {
            // 6a. PreToolUse 훅 실행
            let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);

            // 6b. 권한 체크
            let permission = self.permission_policy.authorize_with_context(...);

            // 6c. 권한이 Allow면 도구 실행, Deny면 거부 메시지
            let result = match permission {
                Allow => {
                    let output = self.tool_executor.execute(&tool_name, &input);
                    // 6d. PostToolUse 훅 실행
                    self.run_post_tool_use_hook(&tool_name, &input, &output);
                    ConversationMessage::tool_result(tool_use_id, output)
                }
                Deny { reason } => {
                    ConversationMessage::tool_result(tool_use_id, reason, is_error: true)
                }
            };

            // 6e. 도구 실행 결과를 세션에 추가
            self.session.push_message(result);
        }
        // → 루프 처음으로 돌아가서 다시 API 호출
    }

    // 7. 토큰이 많이 쌓였으면 자동 압축
    self.maybe_auto_compact();

    Ok(TurnSummary { ... })
}
```

### LLM 응답의 두 가지 형태

LLM은 항상 다음 두 가지 중 하나로 응답합니다:

**1. 텍스트 응답** — 사용자에게 보여줄 메시지
```
AssistantEvent::TextDelta("버그를 수정했습니다. 변경 사항은...")
```

**2. 도구 호출(tool_use)** — Harness에게 실행을 요청
```
AssistantEvent::ToolUse {
    id: "toolu_abc123",
    name: "Read",
    input: "{\"file_path\": \"src/main.py\"}"
}
```

Harness는 도구 호출을 실행하고, 그 결과를 `tool_result`로 대화 이력에 추가한 뒤, 다시 API를 호출합니다. LLM이 텍스트만 응답할 때까지 이 루프가 반복됩니다.

### Python 포팅에서의 동일 패턴

```python
# src/query_engine.py (line 61)
def submit_message(self, prompt, matched_commands, matched_tools, denied_tools) -> TurnResult:
    if len(self.mutable_messages) >= self.config.max_turns:
        return TurnResult(..., stop_reason='max_turns_reached')

    # 메시지 처리 → 세션에 추가 → 결과 반환
    self.mutable_messages.append(prompt)
    self.transcript_store.append(prompt)
    self.compact_messages_if_needed()
    return TurnResult(..., stop_reason='completed')
```

Python 포팅은 실제 API 호출 없이 구조만 재현한 것이므로, 대화 루프의 **골격**을 이해하는 데 적합합니다.

