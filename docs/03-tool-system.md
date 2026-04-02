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
