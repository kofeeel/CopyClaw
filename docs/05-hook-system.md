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

