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

