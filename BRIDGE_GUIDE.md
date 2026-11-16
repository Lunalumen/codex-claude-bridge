# ChatGPT/Codex → Claude 터미널 연결

## 동작
사용자가 요청한 질문을 공식 Claude Code CLI의 `-p --output-format json`으로 보낸다. 기존 구독 로그인을 공식 CLI가 직접 사용한다. API 키 대체·자격증명 복사·다른 사용자 중개는 하지 않는다. 구독 사용량을 소비한다.

Claude에게 도구는 제공하지 않는다. `--safe-mode --restricted`와 빈 도구/MCP 설정을 사용한다. 참고 파일은 사용자가 지정한 이 프로젝트 안의 Markdown/텍스트 최대 4개·합계 20,000자만 읽어 전달한다. 폴더 전체·전체 대화·인증 파일은 읽지 않는다.

같은 `--task`는 저장한 Claude session ID로 `--resume`한다. 실패 시 자동 재시도·다른 대화로 자동 전환하지 않는다. 전송 상태가 불명확하면 기록 확인 후 명시적으로 `--new-session`을 선택한다. 작업별 동시 실행 잠금이 있으며 오래된 잠금을 자동 제거하지 않는다.

## 실행 (이 폴더를 작업 위치로)
아래 python은 설치된 Python 실행 경로로 바꿔도 된다.

Claude 실행 경로는 `claude` 명령(PATH) → `AppData/Roaming/npm/claude.cmd` → `.local/bin/claude.exe` 순서로 찾는다. 앱 내부 버전 경로는 자동 사용하지 않는다. npm의 .cmd/.ps1은 같은 공식 패키지의 bin/claude.exe(현재 설치 형식), 또는 기존 Node.js + cli.js 형식으로 직접 실행하므로 셸 명령 조합을 사용하지 않는다. 새 설치 직후에는 새 터미널에서 PATH를 확인한다.

```powershell
python -B -X utf8 bridge.py status
python -B -X utf8 bridge.py ask --task connection-check --question-file questions/connection-check.md
python -B -X utf8 bridge.py ask --task connection-check --question-file questions/resume-check.md
python -B -X utf8 bridge.py history --task connection-check
```

로그인 안 된 경우 `status`가 보여주는 공식 CLI 경로를 사용해 사용자가 직접 `auth login`한다. 로그인 비밀번호·인증 코드는 채팅에 붙이지 않는다. Claude 앱 로그인과 CLI 로그인은 같다고 가정하지 않는다. `--bare`는 구독 로그인을 읽지 않으므로 사용하지 않는다.

개발 질문은 `questions/<이름>.md`에 저장하고 고유한 `--task`로 보낸다. 필요한 프로젝트 문서를 `--context README.md`처럼 선택한다. 질문과 참고 파일 내용은 Claude 서비스에 전달되므로 승인된 자료만 지정한다.

## 공유 기록 위치
`.bridge/tasks/<task>/state.json`: 현재 대화 ID·진행 상태.
`.bridge/tasks/<task>/<순번>/request.json`: 보낸 질문과 명시적 참고 자료.
같은 폴더의 `answer-claude.md`: 실제 Claude의 보이는 답변.
`result.json`: 성공/실패와 CLI가 반환한 선택 usage. 내부 사고 과정은 보관하지 않는다.

`.bridge`는 로컬 전용으로 Git에서 제외한다. 파일 존재·실제 답변·Claude 데스크톱의 기존 대화 접근은 서로 다르다. 이 도구는 현재 데스크톱 대화를 제어하지 않고 별도의 작업별 Claude CLI 대화를 만든다. 기존 Claude → Codex 브리지는 수정하지 않는다. 현재 Codex는 이 스크립트를 터미널 도구로 실행할 수 있다. 네이티브 MCP 등록이나 자동 상시 왕복은 이 버전에 포함하지 않는다.

## 검증
`python -B -X utf8 test_bridge.py -v`는 가짜 클라이언트로 오프라인 실행한다. 이 테스트의 합성 답변을 실제 연결 결과로 부르지 않는다. 실제 로그인 후 연결 시험·이어가기 시험을 별도로 실행하고 확인한다.

공식 참고: https://code.claude.com/docs/en/cli-reference 및 https://code.claude.com/docs/en/headless .
