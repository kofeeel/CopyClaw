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
