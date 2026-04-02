# CLI AI 하네스 엔지니어링 학습 가이드

> 이 문서는 [claw-code](https://github.com/ultraworkers/claw-code-parity) 코드베이스를 기반으로,
> Claude Code와 같은 CLI 기반 AI 도구의 내부 아키텍처를 탑다운 방식으로 설명합니다.
>
> 모든 코드 참조는 실제 레포의 파일 경로, 구조체명, 함수명과 대조 검증되었습니다.

---

## 목차

1. [Level 0: 큰 그림 — CLI AI의 정체](#level-0-큰-그림--cli-ai의-정체)
2. [Level 1: Conversation Loop — 핵심 동작 원리](#level-1-conversation-loop--핵심-동작-원리)
3. [Level 2: 핵심 서브시스템 상세](#level-2-핵심-서브시스템-상세)
   - [2.1 API Client](#21-api-client--llm과의-통신)
   - [2.2 Tool System](#22-tool-system--ai가-세상과-상호작용하는-방법)
   - [2.3 Permission System](#23-permission-system--안전장치)
   - [2.4 Hook System](#24-hook-system--사용자-확장-포인트)
   - [2.5 Bootstrap](#25-bootstrap--시작-시퀀스)
   - [2.6 Session & Compaction](#26-session--compaction--대화-관리)
   - [2.7 MCP](#27-mcp-model-context-protocol--외부-도구-연결)
4. [Level 3: 데이터 흐름 — 한 턴의 생애주기](#level-3-데이터-흐름--한-턴의-생애주기)
5. [Level 4: 코드베이스 매핑](#level-4-코드베이스-매핑)
6. [학습 순서 추천](#학습-순서-추천)
7. [용어 정리](#용어-정리)

---

## Level 0: 큰 그림 — CLI AI의 정체

Claude Code 같은 CLI AI는 단순히 "채팅하는 AI"가 아닙니다.

LLM(대규모 언어 모델) 자체는 파일을 읽을 수 없고, 코드를 실행할 수 없으며, 인터넷에 접속할 수도 없습니다. LLM은 텍스트를 입력받아 텍스트를 출력하는 함수일 뿐입니다.

**Harness(하네스)** 가 이 격차를 메웁니다. Harness는 LLM을 감싸서 실제 세상(파일시스템, 터미널, 웹)과 연결하는 런타임입니다. Claude Code의 핵심 가치는 Claude 모델 자체가 아니라, 이 Harness의 설계에 있습니다.

```
┌─────────────────────────────────────────────────┐
│                    사용자 (CLI)                    │
│                  "버그 고쳐줘"                      │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              Harness (Runtime)                   │
│                                                 │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ System    │  │ Tool     │  │ Permission   │  │
│  │ Prompt    │  │ Registry │  │ System       │  │
│  │ Builder   │  │          │  │              │  │
│  └───────────┘  └──────────┘  └──────────────┘  │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Session   │  │ Hook     │  │ Conversation │  │
│  │ Manager   │  │ Runner   │  │ Loop         │  │
│  └───────────┘  └──────────┘  └──────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              API Client (SSE Stream)             │
│         Claude API에 메시지 보내고 응답 받기          │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│              Claude 모델 (LLM)                    │
│         "Read 도구로 파일을 읽겠습니다"               │
└─────────────────────────────────────────────────┘
```

### 핵심 통찰

Harness가 하는 일은 다음 세 가지로 요약됩니다:

1. **LLM에게 도구 목록을 알려준다** — "너는 Read, Bash, Edit 같은 도구를 쓸 수 있어"
2. **LLM이 도구를 호출하면 실제로 실행한다** — 파일을 읽고, 명령어를 실행하고, 코드를 수정한다
3. **실행 결과를 다시 LLM에게 보내서 다음 행동을 결정하게 한다** — 이것이 루프

이 세 단계의 반복이 CLI AI의 전부입니다.

---

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

---

## Level 2: 핵심 서브시스템 상세

### 2.1 API Client — LLM과의 통신

**파일**: `rust/crates/api/src/client.rs`

API Client는 LLM 서비스에 메시지를 보내고 응답을 받는 계층입니다.

#### 멀티 프로바이더 설계

```rust
// rust/crates/api/src/client.rs (line 24)
pub enum ProviderClient {
    Anthropic(AnthropicClient),     // Claude API (Anthropic 자체)
    Xai(OpenAiCompatClient),        // Grok API (xAI)
    OpenAi(OpenAiCompatClient),     // OpenAI API (호환 모드)
}
```

모델명으로 프로바이더를 자동 감지합니다:

```rust
// from_model_with_anthropic_auth (line 35)
match providers::detect_provider_kind(&resolved_model) {
    ProviderKind::Anthropic => /* claude-* 모델 */,
    ProviderKind::Xai       => /* grok-* 모델 */,
    ProviderKind::OpenAi    => /* gpt-* 모델 */,
}
```

`resolve_model_alias`는 단축 이름을 정식 모델 ID로 변환합니다 (예: `"opus"` → `"claude-opus-4-6"`, `"grok"` → `"grok-3"`).

#### 메시지 요청 구조

API에 보내는 요청의 형태:

```rust
// rust/crates/api/src/types.rs (line 5)
pub struct MessageRequest {
    pub model: String,                          // "claude-opus-4-6"
    pub max_tokens: u32,                        // 최대 출력 토큰
    pub messages: Vec<InputMessage>,            // 대화 이력
    pub system: Option<String>,                 // 시스템 프롬프트
    pub tools: Option<Vec<ToolDefinition>>,     // 사용 가능한 도구 목록
    pub tool_choice: Option<ToolChoice>,        // 도구 선택 전략
    pub stream: bool,                           // SSE 스트리밍 여부
}
```

#### SSE 스트리밍

응답은 한 번에 오지 않고, SSE(Server-Sent Events)로 토큰 단위로 실시간 전송됩니다:

```rust
// rust/crates/api/src/types.rs (line 238)
pub enum StreamEvent {
    MessageStart(MessageStartEvent),        // 응답 시작
    ContentBlockStart(ContentBlockStartEvent), // 콘텐츠 블록 시작 (텍스트 or tool_use)
    ContentBlockDelta(ContentBlockDeltaEvent),  // 토큰 단위 델타
    ContentBlockStop(ContentBlockStopEvent),    // 콘텐츠 블록 종료
    MessageDelta(MessageDeltaEvent),        // 메시지 레벨 델타 (stop_reason 등)
    MessageStop(MessageStopEvent),          // 응답 완전 종료
}
```

SSE 파싱은 두 레이어에서 이루어집니다:
- **api crate**: `SseParser` (`rust/crates/api/src/sse.rs`) — HTTP 바이트 스트림을 `StreamEvent`로 파싱
- **runtime crate**: `IncrementalSseParser` (`rust/crates/runtime/src/sse.rs`) — 청크 단위 점진적 파싱, `SseEvent` 구조체로 변환

#### Prompt Cache

동일한 시스템 프롬프트를 매번 재전송하면 토큰 낭비입니다. `PromptCache`가 이전 요청의 캐시 상태를 추적하여 중복 전송을 줄입니다:

```rust
// client.rs (line 64)
pub fn with_prompt_cache(self, prompt_cache: PromptCache) -> Self
pub fn prompt_cache_stats(&self) -> Option<PromptCacheStats>
```

### 2.2 Tool System — AI가 세상과 상호작용하는 방법

**파일**: `rust/crates/tools/src/lib.rs`, `src/tools.py`

도구(Tool)란 LLM이 호출을 요청할 수 있는 함수 정의입니다.

#### 도구가 LLM에 전달되는 형태

API 요청의 `tools` 필드에 JSON Schema 형식으로 도구를 정의합니다:

```json
{
  "tools": [
    {
      "name": "Read",
      "description": "파일을 읽습니다",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string", "description": "읽을 파일의 절대 경로" },
          "offset": { "type": "integer", "description": "시작 줄 번호" },
          "limit": { "type": "integer", "description": "읽을 줄 수" }
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "Bash",
      "description": "셸 명령어를 실행합니다",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": { "type": "string" },
          "timeout": { "type": "integer" }
        },
        "required": ["command"]
      }
    }
  ]
}
```

LLM은 이 정의를 보고, 텍스트 대신 도구 호출을 응답할 수 있습니다:

```json
{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "Read",
  "input": { "file_path": "/src/main.py" }
}
```

Harness가 이 JSON을 받아서 **실제로 파일을 읽고**, 결과를 `tool_result`로 대화에 추가합니다:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "content": "1\tfrom __future__ import annotations\n2\timport argparse\n..."
}
```

#### 실제 도구 실행 — file_ops와 bash

Rust crate에서 도구 실행의 실제 구현:

```rust
// rust/crates/runtime/src/file_ops.rs — 파일 관련 도구
pub fn read_file(...) -> ReadFileOutput
pub fn write_file(...) -> WriteFileOutput
pub fn edit_file(...) -> EditFileOutput
pub fn glob_search(...) -> GlobSearchOutput
pub fn grep_search(...) -> GrepSearchOutput

// rust/crates/runtime/src/bash.rs — 셸 실행 도구
pub fn execute_bash(input: BashCommandInput) -> BashCommandOutput
```

#### Python 포팅의 도구 목록

Python 포팅에서는 원본 Claude Code의 도구 목록을 JSON 스냅샷으로 보존합니다:

```python
# src/tools.py (line 11)
SNAPSHOT_PATH = Path(__file__).resolve().parent / 'reference_data' / 'tools_snapshot.json'

# line 37
PORTED_TOOLS = load_tool_snapshot()  # tuple[PortingModule, ...]
```

`PortingModule`은 각 도구의 메타데이터를 담는 데이터 클래스입니다:

```python
# src/models.py (line 14)
@dataclass(frozen=True)
class PortingModule:
    name: str           # 도구 이름 (예: "Read", "Bash")
    responsibility: str  # 역할 설명
    source_hint: str     # 원본 TypeScript 파일 위치
    status: str = 'planned'
```

### 2.3 Permission System — 안전장치

**파일**: `rust/crates/runtime/src/permissions.rs`

AI가 `rm -rf /`를 실행하면 안 되겠죠. Permission System이 모든 도구 호출 전에 권한을 체크합니다.

#### 권한 모드

```rust
// permissions.rs (line 8)
pub enum PermissionMode {
    ReadOnly,           // "read-only"     — 읽기만 가능
    WorkspaceWrite,     // "workspace-write" — 프로젝트 디렉토리 내 쓰기 가능
    DangerFullAccess,   // "danger-full-access" — 모든 것 가능
    Prompt,             // "prompt"        — 매번 사용자에게 물어봄
    Allow,              // "allow"         — 전부 허용 (위험!)
}
```

각 도구에는 **필요한 최소 권한 레벨**이 지정됩니다:
- `Read` → `ReadOnly`면 충분
- `Edit`, `Write` → `WorkspaceWrite` 필요
- `Bash` → `DangerFullAccess` 필요 (임의 명령 실행이므로)

#### 권한 정책(Policy)

```rust
// permissions.rs (line 91)
pub struct PermissionPolicy {
    active_mode: PermissionMode,                      // 현재 활성 모드
    tool_requirements: BTreeMap<String, PermissionMode>, // 도구별 필요 권한
    allow_rules: Vec<PermissionRule>,  // 허용 규칙 (예: bash(git:*))
    deny_rules: Vec<PermissionRule>,   // 거부 규칙 (예: bash(rm -rf:*))
    ask_rules: Vec<PermissionRule>,    // 확인 규칙 (매번 물어봄)
}
```

#### 권한 판단 흐름

`authorize_with_context` 메서드 (line 167)의 판단 순서:

```
1. deny_rules에 매칭? → 즉시 거부
2. hook에서 Deny 오버라이드? → 즉시 거부
3. hook에서 Ask 오버라이드? → 사용자에게 확인
4. ask_rules에 매칭? → 사용자에게 확인
5. allow_rules에 매칭 OR 현재 모드 >= 필요 모드? → 허용
6. 현재 모드가 Prompt이거나, WorkspaceWrite인데 DangerFullAccess 필요? → 사용자에게 확인
7. 그 외 → 거부
```

#### Rule 기반 세밀한 제어

규칙 문법: `도구명(패턴)`

```
bash(git:*)          → git으로 시작하는 bash 명령은 허용
bash(rm -rf:*)       → rm -rf로 시작하는 bash 명령은 거부
Read                 → Read 도구 전체에 대한 규칙
bash(git status)     → 정확히 "git status"만 매칭
```

규칙 매칭 시 도구 입력 JSON에서 `command`, `path`, `file_path` 등의 키를 자동 추출하여 패턴과 비교합니다 (`extract_permission_subject` 함수, line 439).

### 2.4 Hook System — 사용자 확장 포인트

**파일**: `rust/crates/runtime/src/hooks.rs`

Hook은 도구 실행 전후에 자동으로 실행되는 **사용자 정의 셸 스크립트**입니다.

#### 세 가지 Hook 이벤트

```rust
// hooks.rs (line 19)
pub enum HookEvent {
    PreToolUse,         // 도구 실행 "전"
    PostToolUse,        // 도구 실행 "후" (성공 시)
    PostToolUseFailure, // 도구 실행 "후" (실패 시)
}
```

#### Hook 실행 메커니즘

`HookRunner`가 등록된 셸 명령어를 실행합니다 (line 152):

1. **환경변수로 컨텍스트 전달**:
   - `HOOK_EVENT` — 이벤트 종류 ("PreToolUse" 등)
   - `HOOK_TOOL_NAME` — 도구 이름 ("Read", "Bash" 등)
   - `HOOK_TOOL_INPUT` — 도구 입력 JSON
   - `HOOK_TOOL_OUTPUT` — 도구 출력 (PostToolUse만)
   - `HOOK_TOOL_IS_ERROR` — 에러 여부 ("0" 또는 "1")

2. **stdin으로 JSON payload 전달**:
   ```json
   {
     "hook_event_name": "PreToolUse",
     "tool_name": "Bash",
     "tool_input": {"command": "git status"},
     "tool_input_json": "{\"command\":\"git status\"}"
   }
   ```

3. **exit code로 결과 판단**:
   - `0` — 허용 (도구 실행 계속)
   - `2` — 거부 (도구 실행 차단)
   - 기타 — 실패 (hook 자체 오류)

4. **stdout JSON으로 고급 제어**:
   ```json
   {
     "systemMessage": "추가 컨텍스트 메시지",
     "hookSpecificOutput": {
       "permissionDecision": "allow",
       "permissionDecisionReason": "hook이 승인함",
       "updatedInput": {"command": "git status --short"},
       "additionalContext": "LLM에게 전달할 추가 정보"
     }
   }
   ```

#### Hook 실행 결과 구조

```rust
// hooks.rs (line 81)
pub struct HookRunResult {
    denied: bool,                                    // 거부되었는가
    failed: bool,                                    // 실패했는가
    cancelled: bool,                                 // 취소되었는가
    messages: Vec<String>,                           // 메시지들
    permission_override: Option<PermissionOverride>, // 권한 오버라이드
    permission_reason: Option<String>,               // 오버라이드 이유
    updated_input: Option<String>,                   // 수정된 도구 입력
}
```

Hook 시스템 덕분에 사용자는 Harness 코드를 수정하지 않고도 동작을 커스터마이징할 수 있습니다. 예를 들어:
- 모든 Bash 명령 실행 전에 로깅
- 특정 파일 수정을 차단
- 도구 입력을 자동 수정 (예: 경로 정규화)

#### 취소 시그널

장시간 실행되는 hook을 중단하기 위한 메커니즘:

```rust
// hooks.rs (line 59)
pub struct HookAbortSignal {
    aborted: Arc<AtomicBool>,  // 스레드 안전한 취소 플래그
}
```

별도 스레드에서 `abort()`를 호출하면, hook 실행 루프가 20ms 간격으로 폴링하다가 프로세스를 kill합니다 (line 696).

### 2.5 Bootstrap — 시작 시퀀스

**파일**: `rust/crates/runtime/src/bootstrap.rs`

CLI 앱이 시작될 때 모든 서브시스템을 한꺼번에 초기화하면 느립니다. Bootstrap은 단계별 초기화를 통해 빠른 시작을 구현합니다.

```rust
// bootstrap.rs (line 2)
pub enum BootstrapPhase {
    CliEntry,                    // CLI 진입점 파싱
    FastPathVersion,             // --version 같은 빠른 경로 처리
    StartupProfiler,             // 성능 프로파일링 시작
    SystemPromptFastPath,        // 시스템 프롬프트 사전 조립
    ChromeMcpFastPath,           // Chrome MCP 서버 연결
    DaemonWorkerFastPath,        // 데몬 워커 초기화
    BridgeFastPath,              // IDE 브릿지 연결
    DaemonFastPath,              // 데몬 프로세스 초기화
    BackgroundSessionFastPath,   // 백그라운드 세션 복원
    TemplateFastPath,            // 템플릿 처리
    EnvironmentRunnerFastPath,   // 환경 러너 초기화
    MainRuntime,                 // 메인 런타임 루프 시작
}
```

**Fast Path 패턴**: `claude --version`처럼 전체 런타임이 불필요한 명령은 `FastPathVersion` 단계에서 즉시 결과를 반환하고 종료합니다. 이렇게 하면 불필요한 MCP 서버 연결이나 세션 복원을 건너뛸 수 있습니다.

`BootstrapPlan` (line 18)은 실행할 단계의 순서를 정의하며, 중복 단계를 자동으로 제거합니다.

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
```

### 2.7 MCP (Model Context Protocol) — 외부 도구 연결

**파일**: `rust/crates/runtime/src/mcp.rs`, `mcp_client.rs`, `mcp_stdio.rs`

MCP는 **외부 프로그램이 제공하는 도구**를 런타임에 동적으로 추가하는 프로토콜입니다. 예를 들어 GitHub API, 데이터베이스 조회, Slack 메시지 전송 등을 별도 MCP 서버로 구현하면, Claude Code가 이를 도구처럼 사용할 수 있습니다.

#### 통신 방식

```
[Claude Code Runtime] ←JSON-RPC 2.0→ [MCP Server (별도 프로세스)]
```

MCP는 JSON-RPC 2.0 프로토콜을 사용합니다:

```rust
// mcp_stdio.rs (line 37)
pub struct JsonRpcRequest<T> {
    pub jsonrpc: String,    // "2.0"
    pub id: JsonRpcId,      // 요청 ID
    pub method: String,     // "tools/list", "tools/call" 등
    pub params: Option<T>,  // 파라미터
}

// line 66
pub struct JsonRpcResponse<T> {
    pub jsonrpc: String,
    pub id: Option<JsonRpcId>,
    pub result: Option<T>,
    pub error: Option<JsonRpcError>,
}
```

#### MCP 서버 관리

```rust
// mcp_stdio.rs (line 353)
pub struct McpServerManager {
    servers: BTreeMap<String, ManagedMcpServer>,       // 관리 중인 서버들
    unsupported_servers: Vec<UnsupportedMcpServer>,    // 지원하지 않는 서버
    tool_index: BTreeMap<String, ToolRoute>,            // 도구명 → 서버 라우팅
    next_request_id: u64,                               // 다음 JSON-RPC 요청 ID
}
```

#### MCP 도구 구조

```rust
// mcp_stdio.rs (line 113)
pub struct McpTool {
    pub name: String,                       // 도구 이름
    pub description: Option<String>,        // 설명
    pub input_schema: Option<JsonValue>,    // 입력 JSON Schema
    pub annotations: Option<JsonValue>,     // 추가 주석
    pub meta: Option<JsonValue>,            // 메타데이터
}
```

#### 전송 방식 (Transport)

MCP 서버와의 통신은 여러 전송 방식을 지원합니다:

```rust
// mcp_client.rs (line 9)
pub enum McpClientTransport {
    Stdio(McpStdioTransport),              // stdin/stdout으로 로컬 프로세스와 통신
    Sse(McpRemoteTransport),               // SSE HTTP 원격 연결
    Http(McpRemoteTransport),              // HTTP 원격 연결
    WebSocket(McpRemoteTransport),         // WebSocket 원격 연결
    Sdk(McpSdkTransport),                  // SDK 기반 통신
    ManagedProxy(McpManagedProxyTransport), // 프록시를 통한 관리형 통신
}
```

가장 일반적인 방식은 `Stdio` 변형으로, 별도 프로세스를 spawn하고 stdin/stdout으로 JSON-RPC 메시지를 주고받습니다:

```rust
// mcp_stdio.rs
pub fn spawn_mcp_stdio_process(...) -> McpStdioProcess
```

#### MCP 작동 흐름

```
1. 시작 시 MCP 서버 프로세스 spawn
2. "initialize" JSON-RPC 호출로 서버 기능 확인
3. "tools/list" 호출로 서버가 제공하는 도구 목록 가져오기
4. 가져온 도구들을 기존 도구 목록에 추가 (LLM에 전달)
5. LLM이 MCP 도구를 호출하면 "tools/call" JSON-RPC로 서버에 중계
6. 서버 응답을 tool_result로 변환하여 대화에 추가
```

---

## Level 3: 데이터 흐름 — 한 턴의 생애주기

사용자가 "src/main.py의 버그를 고쳐줘"라고 입력했을 때, 시스템 내부에서 일어나는 일을 처음부터 끝까지 추적합니다.

### 단계별 흐름

```
┌──────────────────────────────────────────────────────┐
│ 1. CLI 진입                                           │
│    사용자 입력 → argparse/clap 파싱 → 세션 로드/생성      │
│    (rusty-claude-cli/src/main.rs → app.rs → args.rs)  │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 2. System Prompt 조립                                 │
│    (runtime/src/prompt.rs)                            │
│    - 기본 프롬프트: "You are Claude Code..."           │
│    - CLAUDE.md 내용 주입 (사용자 커스텀 지시사항)         │
│    - 환경 정보: OS, 작업 디렉토리, 모델명               │
│    - 사용 가능한 도구 목록 (tools JSON Schema)          │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 3. API 호출 (conversation.rs:run_turn, line 312)      │
│    ApiRequest { system_prompt, messages }              │
│    → api_client.stream(request)                       │
│    → SSE 스트림으로 실시간 응답 수신                     │
│    → 토큰 단위로 터미널에 출력                           │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 4. 응답 파싱 (conversation.rs, line 335)              │
│    AssistantEvent들을 ConversationMessage로 조립       │
│    tool_use 블록 추출 → pending_tool_uses 리스트        │
│    tool_use가 없으면 → 텍스트만 출력하고 턴 종료         │
└───────────────────────┬──────────────────────────────┘
                        ▼  (tool_use가 있는 경우)
┌──────────────────────────────────────────────────────┐
│ 5. PreToolUse Hook 실행 (conversation.rs, line 361)   │
│    HookRunner.run_pre_tool_use(tool_name, input)      │
│    - 환경변수 + stdin JSON으로 컨텍스트 전달             │
│    - exit code 확인 (0=허용, 2=거부)                   │
│    - stdout JSON에서 입력 수정, 권한 오버라이드 추출      │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 6. Permission Check (conversation.rs, line 392)       │
│    permission_policy.authorize_with_context(           │
│        tool_name, input, hook_context, prompter       │
│    )                                                  │
│    - deny_rules → allow_rules → ask_rules 순서 평가    │
│    - 현재 모드 vs 필요 모드 비교                        │
│    - Deny면 → 에러 메시지를 tool_result로              │
│    - Allow면 → 도구 실행 진행                          │
└───────────────────────┬──────────────────────────────┘
                        ▼  (Allow인 경우)
┌──────────────────────────────────────────────────────┐
│ 7. 도구 실행 (conversation.rs, line 411)               │
│    tool_executor.execute(tool_name, input)             │
│    - Read: file_ops::read_file → 파일 내용 반환        │
│    - Bash: bash::execute_bash → 명령어 실행 결과       │
│    - Edit: file_ops::edit_file → 파일 수정 결과        │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 8. PostToolUse Hook 실행 (conversation.rs, line 417)  │
│    성공: run_post_tool_use_hook(name, input, output)  │
│    실패: run_post_tool_use_failure_hook(...)           │
│    - hook이 추가 피드백을 output에 병합                  │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 9. tool_result를 세션에 추가 (line 454)                │
│    session.push_message(tool_result)                   │
│    → 다시 3번으로 돌아감 (API 재호출)                    │
│    → LLM이 텍스트만 응답할 때까지 반복                   │
└───────────────────────┬──────────────────────────────┘
                        ▼  (루프 종료 후)
┌──────────────────────────────────────────────────────┐
│ 10. 자동 압축 검사 (conversation.rs, line 462)         │
│     maybe_auto_compact()                              │
│     - 입력 토큰이 100,000 초과 시 압축 실행             │
│     - 오래된 메시지를 요약으로 교체                      │
│     - 최근 메시지는 보존                               │
└───────────────────────┬──────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────┐
│ 11. TurnSummary 반환                                  │
│     - assistant_messages: 어시스턴트 응답들             │
│     - tool_results: 도구 실행 결과들                    │
│     - usage: 토큰 사용량                               │
│     - iterations: 루프 반복 횟수                        │
│     → 터미널에 최종 텍스트 렌더링                       │
└──────────────────────────────────────────────────────┘
```

### 실제 예시: 한 턴에 3번의 API 호출

```
[API 호출 1]
  사용자: "src/main.py의 버그 고쳐줘"
  Claude: "먼저 파일을 읽겠습니다" + tool_use(Read, {file_path: "src/main.py"})
  → Read 실행 → 파일 내용을 tool_result로 추가

[API 호출 2]  
  Claude: "23번 줄에 오타가 있네요. 수정하겠습니다" + tool_use(Edit, {file_path: "src/main.py", ...})
  → Edit 실행 → 수정 결과를 tool_result로 추가

[API 호출 3]
  Claude: "버그를 수정했습니다. print를 return으로 변경했습니다." (텍스트만)
  → tool_use 없음 → 루프 종료
```

---

## Level 4: 코드베이스 매핑

### 디렉토리 구조

```
claw-code-parity/
│
├── src/                              # Python 포팅 (아키텍처 학습용)
│   ├── main.py                       # CLI 진입점 — argparse로 서브커맨드 정의
│   ├── runtime.py                    # PortRuntime — 대화 루프의 Python 버전
│   ├── query_engine.py               # QueryEnginePort — 세션/턴 관리
│   ├── tools.py                      # 도구 레지스트리 (JSON 스냅샷에서 로드)
│   ├── commands.py                   # 슬래시 커맨드 레지스트리
│   ├── permissions.py                # ToolPermissionContext — 도구 차단 규칙
│   ├── models.py                     # 핵심 데이터 모델 (PortingModule, UsageSummary 등)
│   ├── context.py                    # 포팅 컨텍스트 (파일 수, 아카이브 상태)
│   ├── transcript.py                 # 대화 기록 저장소
│   ├── session_store.py              # 세션 영속화 (저장/로드)
│   ├── system_init.py                # 시스템 초기화 메시지 생성
│   ├── execution_registry.py         # 도구/커맨드 실행 레지스트리
│   ├── history.py                    # 히스토리 로그
│   └── reference_data/               # 원본 Claude Code의 도구/커맨드 JSON 스냅샷
│
├── rust/                             # Rust 재구현 (실제 동작 코드)
│   ├── Cargo.toml                    # 워크스페이스 정의
│   └── crates/
│       ├── api/                      # API 클라이언트 크레이트
│       │   ├── src/
│       │   │   ├── client.rs         # ProviderClient — Anthropic/xAI/OpenAI 멀티 프로바이더
│       │   │   ├── types.rs          # MessageRequest, MessageResponse, StreamEvent
│       │   │   ├── sse.rs            # SseParser — HTTP SSE 스트림 파싱
│       │   │   ├── prompt_cache.rs   # 프롬프트 캐시 관리
│       │   │   ├── error.rs          # API 에러 타입
│       │   │   └── lib.rs            # 크레이트 루트 (re-exports)
│       │   └── tests/                # 통합 테스트
│       │
│       ├── runtime/                  # 핵심 런타임 크레이트
│       │   └── src/
│       │       ├── conversation.rs   # ★ ConversationRuntime — 메인 대화 루프
│       │       ├── permissions.rs    # ★ PermissionPolicy — 권한 시스템
│       │       ├── hooks.rs          # ★ HookRunner — 훅 시스템
│       │       ├── bootstrap.rs      # BootstrapPlan — 시작 시퀀스
│       │       ├── compact.rs        # CompactionConfig — 컨텍스트 압축
│       │       ├── session.rs        # Session, ConversationMessage, ContentBlock
│       │       ├── config.rs         # RuntimeConfig — 설정 로더
│       │       ├── prompt.rs         # SystemPromptBuilder — 시스템 프롬프트 조립
│       │       ├── bash.rs           # execute_bash — Bash 도구 구현
│       │       ├── file_ops.rs       # read_file, write_file, edit_file 등
│       │       ├── mcp.rs            # MCP 유틸리티 함수
│       │       ├── mcp_client.rs     # MCP 클라이언트 전송 계층
│       │       ├── mcp_stdio.rs      # McpServerManager — MCP stdio 통신
│       │       ├── sandbox.rs        # 샌드박스 격리 (Linux 컨테이너)
│       │       ├── sse.rs            # IncrementalSseParser — 점진적 SSE 파싱
│       │       ├── usage.rs          # UsageTracker — 토큰/비용 추적
│       │       ├── oauth.rs          # OAuth 인증 흐름
│       │       ├── remote.rs         # 원격 세션 컨텍스트
│       │       ├── json.rs           # JSON 유틸리티
│       │       └── lib.rs            # 크레이트 루트 (모든 public 타입 re-export)
│       │
│       ├── commands/                 # 슬래시 커맨드 (/help, /clear 등)
│       ├── tools/                    # 도구 정의
│       ├── plugins/                  # 플러그인 시스템 + hooks
│       ├── rusty-claude-cli/         # CLI 앱 (main, app, args, input, render)
│       ├── compat-harness/           # 호환성 하네스
│       └── telemetry/                # 텔레메트리 수집
│
├── tests/                            # Python 통합 테스트
├── CLAUDE.md                         # 프로젝트별 Claude 지시사항
├── PARITY.md                         # 원본 대비 포팅 현황
└── README.md                         # 프로젝트 설명
```

### 핵심 파일 × 개념 매핑

| 개념 | Rust 파일 | Python 파일 | 핵심 타입 |
|------|-----------|-------------|-----------|
| 대화 루프 | `runtime/conversation.rs` | `query_engine.py` | `ConversationRuntime<C,T>`, `QueryEnginePort` |
| API 통신 | `api/client.rs` | — | `ProviderClient`, `MessageRequest` |
| SSE 파싱 | `api/sse.rs`, `runtime/sse.rs` | — | `SseParser`, `IncrementalSseParser` |
| 도구 정의 | `tools/lib.rs` | `tools.py` | `ToolDefinition`, `PortingModule` |
| 도구 실행 | `runtime/bash.rs`, `file_ops.rs` | — | `BashCommandInput/Output`, `ReadFileOutput` |
| 권한 관리 | `runtime/permissions.rs` | `permissions.py` | `PermissionPolicy`, `ToolPermissionContext` |
| 훅 시스템 | `runtime/hooks.rs` | — | `HookRunner`, `HookRunResult` |
| 부트스트랩 | `runtime/bootstrap.rs` | — | `BootstrapPlan`, `BootstrapPhase` |
| 세션 관리 | `runtime/session.rs` | `session_store.py` | `Session`, `ConversationMessage` |
| 컨텍스트 압축 | `runtime/compact.rs` | `query_engine.py` | `CompactionConfig`, `CompactionResult` |
| MCP 통신 | `runtime/mcp_stdio.rs` | — | `McpServerManager`, `McpTool` |
| 프롬프트 | `runtime/prompt.rs` | `system_init.py` | `SystemPromptBuilder` |
| 비용 추적 | `runtime/usage.rs` | `models.py` | `UsageTracker`, `UsageSummary` |

---

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

---

## 용어 정리

| 용어 | 영문 | 의미 |
|------|------|------|
| 하네스 | Harness | LLM을 감싸서 도구/권한/세션을 관리하는 런타임. CLI AI의 핵심 |
| 도구 | Tool | LLM이 호출을 요청할 수 있는 함수 (Read, Bash, Edit 등) |
| tool_use | tool_use | LLM이 "이 도구를 쓰겠다"고 응답한 JSON 블록 |
| tool_result | tool_result | 도구 실행 결과를 LLM에게 돌려보내는 JSON 블록 |
| 시스템 프롬프트 | System Prompt | LLM에게 역할/규칙/도구를 설명하는 최초 메시지 |
| SSE | Server-Sent Events | 서버가 HTTP 연결을 유지하며 실시간으로 데이터를 밀어주는 프로토콜 |
| MCP | Model Context Protocol | 외부 프로그램의 도구를 JSON-RPC로 동적 연결하는 프로토콜 |
| 훅 | Hook | 도구 실행 전후에 자동 실행되는 사용자 정의 셸 스크립트 |
| 압축 | Compaction | 대화가 길어지면 오래된 메시지를 요약해서 컨텍스트 절약 |
| 컨텍스트 윈도우 | Context Window | LLM이 한 번에 처리할 수 있는 입력 토큰의 최대 크기 |
| 프로바이더 | Provider | LLM API 서비스 제공자 (Anthropic, xAI, OpenAI) |
| JSON-RPC | JSON-RPC | JSON으로 원격 프로시저를 호출하는 프로토콜 (MCP가 사용) |
| 프롬프트 캐시 | Prompt Cache | 동일 시스템 프롬프트의 반복 전송을 줄이는 최적화 |
| 세션 | Session | 하나의 대화에 속하는 메시지들의 모음 |
| 턴 | Turn | 사용자 입력 1개에 대한 전체 처리 단위 (여러 API 호출 포함 가능) |
| 이터레이션 | Iteration | 한 턴 안에서의 API 호출 1회 (tool_use가 있으면 여러 이터레이션) |
| Fast Path | Fast Path | 전체 초기화 없이 빠르게 처리할 수 있는 명령의 단축 경로 |
| 샌드박스 | Sandbox | 도구 실행을 격리된 환경에서 수행하여 시스템 보호 |

---

## 참고 자료

- **이 문서의 코드 출처**: [claw-code-parity](https://github.com/ultraworkers/claw-code-parity) (claw-code의 Rust 포팅 작업 레포)
- **Anthropic Messages API 문서**: Claude API의 tool_use/tool_result 공식 스펙
- **Model Context Protocol 스펙**: MCP의 공식 프로토콜 정의

---

> 이 문서의 모든 코드 참조(파일 경로, 구조체명, 필드명, 줄 번호)는
> claw-code-parity 레포의 main 브랜치 기준으로 검증되었습니다.
