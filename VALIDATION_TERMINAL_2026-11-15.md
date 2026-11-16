# Codex → Claude 실제 연결·이어가기 검증

날짜: 2026-11-15. 작성: ChatGPT/Codex.

## 결과
공식 npm 설치의 Claude Code CLI를 통해 합성 질문 2회를 보내고 실제 답변을 받았다. 같은 작업 대화를 이어가는 것도 확인했다. 가짜 클라이언트의 시험 답변을 실제 결과로 표시한 것이 아니다.

## 설치·로그인 확인
- 현재 `claude` 명령과 `C:/Users/<사용자>/AppData/Roaming/npm/claude.cmd`를 찾았다.
- 설치 패키지 2.1.325의 bin 항목은 `bin/claude.exe`다. npm 명령과 같은 바이너리를 셸 없이 직접 실행하도록 연결 프로그램을 보완했다. 앱 내부 바이너리는 자동 사용하지 않는다.
- 공식 auth status의 결과는 loggedIn=true / authMethod=claude.ai. 프로그램은 인증 값을 직접 읽거나 복사하지 않았다.
- 오프라인 가짜 클라이언트 시험: 13개 통과. 실제 연결 시험과 별개다.

## 실제 실행

```text
python -B -X utf8 bridge.py ask --task connection-check --question-file questions/connection-check.md
python -B -X utf8 bridge.py ask --task connection-check --question-file questions/resume-check.md
```

첫 요청은 새 전용 대화로 시작했다. 실제 Claude 답변 첫 줄은 `CLAUDE-BRIDGE-OK`였고, 공유 파일에 작업자·일시·변경 범위·변경 이유·검증 여부를 남기라는 의견을 반환했다.

두 번째 요청은 resumed=true. 요청 본문에 확인 단어를 다시 넣지 않고 직전 확인 단어가 무엇인지 물었다. 실제 답변은 **별빛731**이었다.

두 요청과 현재 state의 session_id가 모두 같고, 두 번째 prompt에 별빛731이 없으며 답변에는 그 단어가 있음을 파일을 재읽어 확인했다. 최종 상태 completed / turn=2.

## 기록
- `.bridge/tasks/connection-check/state.json`: 현재 전용 대화 ID와 상태.
- `0001/request.json`, `0001/answer-claude.md`, `0001/result.json`.
- `0002/request.json`, `0002/answer-claude.md`, `0002/result.json`.

질문·보이는 답변·선택 usage만 로컬 기록으로 남겼다. 내부 사고 과정과 인증 값은 기록하지 않는다. `.bridge/`는 Git 제외 대상이다. 이번에 GitHub 커밋/push하지 않았다.

## 확인 범위
- 현재 Codex가 터미널 도구를 통해 Claude에게 질문 가능.
- 같은 task의 CLI 대화 ID 재사용과 질문/답변 파일 공유 가능.
- 이미 열린 Claude 데스크톱 대화나 현재 Codex 채팅을 직접 깨우는 기능은 아니다.
- 기존 Claude → Codex 브리지의 새 경로/대화 이어가기는 이번에 수정·재시험하지 않았다.
- 네이티브 MCP 도구를 현재 채팅에 등록하지 않았고 상시 감시·무한 왕복을 만들지 않았다.
- 실제 기억 저장소 E:/스텔라 ai/claude/memory 또는 옛 E:/claude/memory는 생성하지 않았다.
- 기존 Claude 구독의 사용량을 소비한다. 별도 API 키 경로로 전환하지 않았으며 전체 토큰/비용 절감은 측정하지 않았다.

다음 개발 요청은 `questions/<이름>.md`에 저장하고 `bridge.py ask --task <작업ID> --question-file questions/<이름>.md`로 보낼 수 있다. 동일 task는 기존 전용 대화를 이어간다. 기존 connection-check는 합성 연결 시험용으로만 유지한다.
