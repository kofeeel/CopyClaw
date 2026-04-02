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

