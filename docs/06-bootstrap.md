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

