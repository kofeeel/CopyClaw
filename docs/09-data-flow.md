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

