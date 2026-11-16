# 연결 상태 · 2026-11-15

## 직접 확인
- Claude 앱에 포함된 공식 CLI 2.1.325를 발견했다. 새 프로그램을 다운로드/설치하지 않았다.
- `auth status`는 기본 제한 환경과 권한 확장 환경 모두 미로그인(authMethod none)을 반환했다.
- `bridge.py` 구현과 오프라인 시험 10개를 실행했고 전부 통과했다. 시험 응답은 가짜 클라이언트의 합성 결과이며 실제 Claude 응답이 아니다.
- 이 버전은 Codex가 터미널로 Claude에게 질문을 보낼 수 있는 진입점이다. 기존 Claude → Codex 브리지와 별개이며 Claude → Codex 쪽 옛 경로 설정은 수정하지 않았다.

## 실제 질문 시험 대기
사용자가 공식 CLI `auth login`을 완료해야 한다. 앱에 로그인돼 있다는 사실만으로 CLI 로그인을 추정하지 않는다. 인증 파일·토큰을 복사하거나 추출하지 않는다.

로그인 후 Codex가 실행할 시험:
1. `bridge.py status`에서 subscription_login=true 확인.
2. `bridge.py ask --task connection-check --question-file questions/connection-check.md`.
3. 같은 작업으로 `questions/resume-check.md` 전송.
4. 실제 첫 답변과 두 번째 답변의 확인 단어를 읽고 session ID가 이어졌는지 검증.

## 남은 경계
- 실제 모델 질문·응답과 resume 시험은 아직 미실행.
- CLI 대화는 별도로 생성된다. 이미 열린 Claude 데스크톱 대화나 현재 Codex 대화를 직접 깨우지 않는다.
- 현재 Codex에서는 터미널로 호출한다. 새로운 MCP 도구가 현재 채팅에 자동 등록됐다고 주장하지 않는다.
- 전체 토큰/요금 절감은 미측정. 구독 사용량이 소비된다.
- 개인 호출 기록은 `.bridge/`에만 남기며 Git에서 제외한다. 폴더 공유와 공개 GitHub 업로드는 다르다. 이번에 커밋/push하지 않았다.

## 실행 경로 보완 · 2026-11-15
사용자 안내에 따라 앱 내부 CLI 자동 탐색을 제거하고 PATH의 claude / npm의 claude.cmd를 우선한다. npm shim은 셸 없이 같은 설치의 node + cli.js로 실행한다. 오프라인 시험 12개 통과. 현재 이 세션과 권한 확장 확인 모두 claude 명령과 AppData/Roaming/npm/claude.cmd 및 cli.js가 발견되지 않았다. 새 설치·로그인 완료는 아직 확인하지 못했으며 실제 모델 왕복은 미실행이다.


## 최신 상태: 실제 연결·이어가기 성공 · 2026-11-15
공식 npm Claude CLI의 구독 로그인(claude.ai)을 확인했다. 연결 프로그램을 현재 bin/claude.exe 형식에 맞춰 보완했고 오프라인 시험 13개 통과. 실제 합성 질문 2회 모두 응답 성공, 같은 session ID 이어가기와 확인 단어 회상을 검증했다. 상세는 VALIDATION_TERMINAL_2026-11-15.md. 위 미설치·미로그인·왕복 미실행 항목은 이전 시점의 기록이다.

