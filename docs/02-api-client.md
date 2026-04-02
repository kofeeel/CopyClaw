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
