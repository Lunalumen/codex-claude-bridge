# 변경 기록

## 2026-11-17 · 양방향 브리지 업그레이드 (코드 미공개)

> [!IMPORTANT]
> 아래 변경은 개발자 PC에서 시험을 마쳤지만, **코드는 아직 이 저장소에 올라오지 않았습니다.**
> 공개 전 점검 항목은 [#5](https://github.com/Lunalumen/codex-claude-bridge/issues/5)에 있습니다.
> 지금 저장소에 있는 `bridge.py`(Codex → Claude)는 그대로 동작합니다.

### 무엇이 바뀌나

| 항목 | 이전 | 업그레이드 후 |
|---|---|---|
| Claude → Codex 대화 | 질문마다 새 대화 | **작업 이름(`task_id`)이 같으면 이전 대화를 이어서** 답함 |
| 기록 | 남지 않음 | 작업별 폴더(`.bridge/`)에 질문·답변·상태 저장 (Git 제외) |
| 실행 진입점 | 방향마다 따로 | 공통 실행기 `tools/bridge.py` 하나로 Claude·Codex 양쪽 호출 |
| 상태 확인 | 없음 | `codex_bridge_status` — 모델을 부르지 않고 로그인·이어가기 지원만 확인 |
| 기록 조회 | 없음 | `codex_bridge_history` — 작업별 차례와 사용량만 보여 줌 (질문 원문·세션 ID 제외) |

### 안전 장치
- 구독 로그인만 사용: ChatGPT 로그인 방식을 강제하고, API 키 환경변수는 넘기지 않음 (허용된 환경변수만 전달)
- 설정(폴더·권한·모델)이 바뀌면 이전 대화를 몰래 이어가지 않고 새 대화를 명시하도록 요구
- 실패한 차례가 있으면 자동으로 이어가지 않음, 자동 재시도 없음
- 같은 작업의 동시 실행 방지(잠금), 입력·출력 크기와 대기 시간 제한
- 오류 메시지에서 내부 경로·원본 오류 문구를 숨김

### 검증 (개발자 보고 기준)
- 오프라인 자동 시험 41개 통과 (Codex 쪽 25 · Claude 쪽 13 · 공통 실행기 3)
- 실제 구독 계정으로 양방향 각각 2회 왕복, 두 번째 질문에서 같은 대화가 이어지는 것 확인
- 모델 호출 없는 상태 점검(`codex_bridge_status`)에서 로그인·이어가기 지원 정상
- 확인하지 않은 것: Windows 외 환경, 회사 프록시, 여러 프로그램이 동시에 쓰는 경우, 전체 토큰·비용 절감

### 저장소 정리
- `.bridge-backups/`(로컬 백업)를 Git 제외 목록에 추가

## 고칠 과제

"아직 안 되는 기능"이 아니라, 쓰면서 알고 고쳐 나갈 수 있는 과제입니다. 고친 분은 해당 이슈에 남겨 주세요.

| # | 과제 | 영향 | 임시 해결 |
|---|---|---|---|
| [#1](https://github.com/Lunalumen/codex-claude-bridge/issues/1) | Codex 앱 업데이트 시 실행 경로(버전 포함)가 바뀌어 연결이 끊김 | **높음** | `CODEX_EXE`를 현재 경로로 다시 맞추고 다시 연결 |
| [#2](https://github.com/Lunalumen/codex-claude-bridge/issues/2) | Codex가 읽을 수 있는 폴더가 연결 폴더 안으로만 제한됨 | 중간 | 검토할 파일을 연결 폴더 안으로 복사 |
| [#3](https://github.com/Lunalumen/codex-claude-bridge/issues/3) | 질문 대기 시간 최대 180초 | 낮음 | 긴 검토는 여러 질문으로 나누기 |
| [#4](https://github.com/Lunalumen/codex-claude-bridge/issues/4) | (Windows) Claude 데스크톱 앱 안에서 설치한 CLI가 앱 밖에서 안 보임 | 중간 | CLI 설치와 `claude auth login`은 일반 터미널에서 |
| [#5](https://github.com/Lunalumen/codex-claude-bridge/issues/5) | 업그레이드 코드 공개 전 점검 | — | — |
