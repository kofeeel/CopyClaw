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

