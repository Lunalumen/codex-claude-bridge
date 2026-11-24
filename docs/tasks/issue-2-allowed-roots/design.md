# 설계: Codex 브리지 허용 폴더 목록

- 작업ID: `issue-2-allowed-roots`
- 관련 이슈: #2 ([고칠 과제] Codex가 읽을 수 있는 폴더가 연결 폴더 안으로만 제한됨)
- 작성일: 2026-11-23

## 목표

운영자가 MCP 설정(환경변수)으로 `STELLA_CODEX_ROOT` 외의 추가 허용 폴더를 명시하면 `ask_chatgpt`의 `cwd`를 그 폴더 안으로도 지정할 수 있다. 설정이 없으면 지금과 완전히 같게 동작하고, 목록 밖은 계속 거부하며 오류 식별자가 설정 방법을 알려 준다.

## 범위

바꿀 파일과 함수:

- `tools/codex-bridge/server.py`
  - 새 모듈 함수 `parse_extra_roots(value)` 추가.
  - `Bridge.__init__(self, root=None, client=None)` → `Bridge.__init__(self, root=None, client=None, extra_roots=None)`.
  - 새 메서드 `Bridge.allowed_roots()` 추가.
  - `Bridge.ask()`에서 `cwd` 검사(현재 275-279행)와 `fingerprint` 계산 부분만 고친다.
- `tools/codex-bridge/test_server.py`: 아래 완료 기준의 시험을 `StoreTests`·`ProtocolTests`에 추가한다. `test_cwd_outside_root`는 그대로 통과해야 한다(정규식 `cwd_outside`).
- `tools/codex-bridge/mcp.example.json`: `env`에 자리표시자 한 줄 추가.
- `tools/codex-bridge/README.md`, `tools/BRIDGE_README.md`: 설정 방법과 한계 설명 몇 문장.

바꾸지 않을 것:

- `Bridge.task_dir()`, 작업 상태 저장 위치(`self.root/.bridge/codex/tasks/<task>`), 잠금·상태 파일 형식.
- `CodexClient`(명령 조립 `-s`, `-C`, `-o` 포함), `bounded_process`, `SAFE_ENV`, `codex_launcher`.
- MCP 도구 스키마 `TOOLS`. 요청 인자로 허용 폴더를 넓히는 기능은 넣지 않는다(설정은 운영자 환경변수에서만).
- `tools/bridge.py`(디스패처)와 `tools/claude-bridge/`. 디스패처는 `codex.Bridge(root=root)`를 그대로 부르며, 환경변수는 `Bridge`가 직접 읽으므로 수정이 필요 없다. 디스패처의 `--question-file`·`--context` 읽기 범위(`--root` 안)는 바꾸지 않는다.
- 저장소 밖 연결 파일(코드 변경 없음, 아래 "설정 방법"만 문서화).

## 동작 설계

### 1. 설정 입력

- 환경변수 이름: `STELLA_CODEX_EXTRA_ROOTS`.
- 형식: 절대 경로를 `os.pathsep`으로 구분(Windows는 `;`, 그 밖은 `:`). 예: `<other-project-folder-1>;<other-project-folder-2>`.
- `STELLA_CODEX_ROOT`(= `self.root`)는 항상 허용된다. 추가 목록은 "그 외에 더" 허용할 폴더다.
- 변수가 없거나 빈 문자열이거나 공백만 있으면 추가 폴더 없음 → 현재 동작과 동일.
- `Bridge(extra_roots=...)` 인자를 주면 환경변수 대신 그 값을 쓴다(시험용). 값은 문자열(환경변수와 같은 형식) 또는 경로 목록(list/tuple) 둘 다 받는다. `None`이면 `os.environ.get('STELLA_CODEX_EXTRA_ROOTS', '')`.

### 2. `parse_extra_roots(value)` 처리

입력이 문자열이면 `os.pathsep`으로 나누고, 각 항목 앞뒤 공백을 지우고 빈 항목은 건너뛴다. 각 항목마다:

1. `Path(entry).is_absolute()`가 아니면 거부.
2. `Path(entry).resolve(strict=True)`. 없는 경로(OSError)는 거부.
3. `is_dir()`가 아니면 거부.
4. 파일시스템 최상위(드라이브 루트 `X:\`, `/`, UNC 공유 최상위 — 즉 `resolved.parent == resolved`)면 거부. 디스크 전체를 여는 설정을 막기 위함.
5. 이미 목록에 있는 경로(Windows 비교 규칙상 같은 경로)는 중복 제거.

어느 항목이든 거부되면 목록 전체를 버리고 `ValueError('invalid_STELLA_CODEX_EXTRA_ROOTS_entry')`를 낸다(일부만 적용하지 않음 — 실패 시 닫힘). 메시지에 경로 값을 넣지 않는다. 결과는 `resolve()`된 `Path` 목록.

### 3. `Bridge.allowed_roots()`와 지연 검사

- `__init__`은 원시 설정값만 저장하고 파싱하지 않는다. 그래서 설정이 잘못돼도 `codex_bridge_history`·`codex_bridge_status`는 지금처럼 동작한다(`handle()`은 도구 종류와 무관하게 먼저 `Bridge()`를 만들기 때문).
- `allowed_roots()`는 `[self.root, *parse_extra_roots(...)]`(self.root와 같은 항목은 제거)를 돌려준다. `ask()`에서만 부른다.

### 4. `Bridge.ask()`의 `cwd` 검사

기존 순서(작업ID·프롬프트·timeout·new_session·sandbox·model 검사)는 그대로 두고, 그 뒤:

1. `directory = Path(cwd) if cwd else self.root`
2. 상대 경로는 지금처럼 `self.root` 기준으로 붙인다(추가 폴더 기준이 아님). 추가 폴더는 절대 경로로 지정해야 한다.
3. `directory = directory.resolve(strict=True)` — `..`, 심볼릭 링크, Windows 정션, 8.3 짧은 이름이 실제 경로로 풀린다. 없는 경로는 지금처럼 OSError → 공개 응답 `bridge_request_failed_no_retry`(동작 변경 없음).
4. `roots = self.allowed_roots()` (설정 오류면 여기서 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`, 클라이언트 호출·상태 파일 생성 전).
5. `directory.is_dir()`이고 `any(directory.is_relative_to(r) for r in roots)`여야 한다. 아니면 `ValueError('cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS')`.
   - 비교는 둘 다 `resolve()`된 경로끼리 한다. Python 3.13의 `WindowsPath.is_relative_to`는 대소문자를 구분하지 않으므로 대소문자·드라이브 문자 표기 차이로 거부되거나 우회되지 않는다. 다른 드라이브는 포함 관계가 성립하지 않는다.
   - 링크로 허용 폴더 밖을 가리키면 3단계에서 실제 대상으로 풀린 뒤 검사되므로 거부된다. 허용 폴더 밖의 링크가 허용 폴더 안을 가리키면 대상 기준으로 허용된다(의도된 동작).
6. 검사 순서상 이 모든 것은 `folder.mkdir`·잠금 생성·`client.ask` 이전에 끝난다(현재와 같은 위치).

### 5. 대화 이어가기 지문(`fingerprint['cwd']`)

- `directory`가 `self.root` 안이면 지금과 똑같이 `directory.relative_to(self.root).as_posix()` (루트 자체는 `'.'`). 기존 `state.json`의 `configuration`과 호환되어 업그레이드 후에도 이어가기가 깨지지 않는다. 추가 폴더가 `self.root` 안에 겹쳐 있어도 이 규칙이 우선한다.
- 그 밖(추가 폴더 안)이면 `os.path.normcase(str(directory))` — 절대 경로 문자열. Windows에서는 소문자·역슬래시로 정규화되어 같은 폴더를 다른 대소문자로 지정해도 같은 지문이 된다.
- 상대 표기(`'.'`, `'sub/x'`)는 드라이브 문자나 `/`·`\\`로 시작하지 않으므로 절대 표기와 겹치지 않는다. 따라서 루트가 달라도 같은 하위 폴더 이름이 서로 다른 지문이 되고, 같은 작업ID로 다른 루트로 옮기면 기존 규칙대로 `session_configuration_changed_explicit_new_session_required`가 난다.
- `fingerprint` 키 구성(`cwd`, `sandbox`, `model`)은 바꾸지 않는다.
- 절대 경로가 `state.json`·`request.json`에 남는다. 이 파일들은 이미 프롬프트 원문을 평문으로 담는 비공개 `.bridge/` 아래이며 `history()` 출력에는 `configuration`이 포함되지 않는다(현재 코드 그대로). 공개 응답에 경로가 나가지 않는다.

### 6. 작업 상태 위치

`task_dir()`는 변경 없음. `cwd`가 추가 폴더여도 상태·잠금·답변 파일은 항상 `self.root/.bridge/codex/tasks/<task>/`에만 생긴다. 추가 폴더 안에는 브리지가 `.bridge/`를 만들지 않는다.

### 7. sandbox

- `sandbox` 허용값은 지금처럼 `read-only`, `workspace-write` 두 가지. 추가 폴더에서도 같은 규칙.
- `CodexClient.ask()`는 변경 없이 `-C <directory>`와 `-s <sandbox>`만 넘기고 `--add-dir` 등 쓰기 범위를 넓히는 인자는 넣지 않는다. 따라서 `workspace-write`의 쓰기 범위는 검사를 통과한 `directory`(허용 폴더 안의 한 폴더)로 한정되고 허용 폴더 전체나 다른 추가 폴더로 넓어지지 않는다. Codex CLI 자체의 기본 임시 폴더 쓰기 허용은 현재와 같다(변경 없음).
- 한계(문서에 그대로 유지): 허용 목록은 `cwd`와 쓰기 범위의 경계이지 OS 읽기 경계가 아니다. `read-only`에서도 Codex CLI가 다른 경로를 읽을 수 있는 것은 현재와 같다.

### 8. 오류 식별자 (공개 응답)

`handle()`의 기존 규칙(`[A-Za-z0-9_]+`인 `ValueError`만 그대로, 나머지는 일반 식별자)을 그대로 쓴다. 새 식별자 두 개 모두 이 형식이다.

| 상황 | 공개 응답 텍스트 |
|---|---|
| 허용 폴더 밖 `cwd` | `cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS` |
| 추가 폴더 설정 항목이 잘못됨 | `invalid_STELLA_CODEX_EXTRA_ROOTS_entry` |
| 없는 `cwd` | `bridge_request_failed_no_retry` (현재와 동일) |

경로·환경변수 값은 어떤 오류에도 들어가지 않는다. 기존 식별자 `cwd_outside_operator_root`는 새 식별자로 바뀐다(정규식 `cwd_outside`로 확인하는 기존 시험은 계속 통과).

### 9. 설정 방법 (문서에만 적을 내용)

- MCP 설정 `env`: `"STELLA_CODEX_EXTRA_ROOTS": "<other-project-folder-1>;<other-project-folder-2>"` (POSIX는 `:` 구분).
- 저장소 밖 연결 파일을 쓰는 경우: 그 파일에서 다른 값처럼 `os.environ.setdefault('STELLA_CODEX_EXTRA_ROOTS', '<other-project-folder-1>;<other-project-folder-2>')`를 `server.py`를 불러오기 전에 넣는다.
- 설정을 바꾼 뒤에는 MCP 호스트를 다시 연결해야 실행 중인 서버에 반영된다.
- 디스패처: `python tools/bridge.py --root <project-folder> codex ask --task <task> --question-file question.txt --cwd <other-project-folder-1>` (같은 환경변수가 셸에 설정되어 있어야 함).

## 완료 기준 (검증 담당이 그대로 확인함)

모든 시험은 `FakeClient` 또는 `bounded_process` 대체만 쓰고 실제 모델·네트워크를 호출하지 않는다. 각 시험은 `patch.dict(os.environ, ...)`로 `STELLA_CODEX_EXTRA_ROOTS`를 통제한다(검증 환경에 값이 있어도 결과가 같도록 기본 시험에서는 이 키를 제거).

- [ ] 기존 시험 전부 통과 — `python -B -X utf8 tools/codex-bridge/test_server.py -v`, `python -B -X utf8 tools/test_bridge_dispatcher.py -v` 둘 다 실패 0.
- [ ] 기본값 동일 — `test_default_without_extra_roots_unchanged`: 환경변수 없음 상태에서 루트 안 `cwd`는 성공하고 `state.json`의 `configuration['cwd']`가 `'.'`(루트) / `'sub'`(하위)로 기존 형식과 같으며, 루트 밖은 `cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS`로 거부되고 `client.calls`가 비어 있음.
- [ ] 기존 상태와 호환 — `test_existing_state_resumes_after_upgrade`: 이전 형식(`{'cwd': '.', 'sandbox': 'read-only', 'model': None}`)의 `state.json`을 직접 만든 뒤 같은 작업으로 `ask`하면 `resumed`가 True.
- [ ] 추가 폴더 허용·상태 위치 — `test_extra_root_allowed_state_stays_in_primary_root`: 임시 폴더 두 개(주 루트, 추가 폴더)로 `Bridge(root, client, extra_roots=[extra])`; 추가 폴더 하위 `cwd`가 성공하고, `FakeClient`가 받은 `cwd`가 그 폴더의 `resolve()` 결과이며, `root/.bridge/codex/tasks/<task>/state.json`이 있고 `extra/.bridge`는 존재하지 않음.
- [ ] 환경변수 파싱 — `test_extra_roots_from_environment`: `os.pathsep.join([str(a), '', ' ' + str(b) + ' '])`를 환경변수로 넣고 `Bridge(root, client)`의 `allowed_roots()`가 `[root, a, b]`(resolve 기준)임. 빈 문자열·공백만이면 `[root]`.
- [ ] 잘못된 설정은 전체 거부 — `test_invalid_extra_roots_refused_without_leaking`: 상대 경로, 없는 경로, 파일, 파일시스템 최상위(`Path(root.anchor)`) 각각을 유효한 항목과 섞어 넣으면 `ask`가 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`로 실패하고 `client.calls`가 비어 있으며 `root/.bridge`가 생기지 않음. 같은 설정으로 `history('a')`는 정상(`not_started`) 반환. `handle()`로 부른 응답 텍스트에 넣은 경로 문자열이 포함되지 않음.
- [ ] 다른 루트 지문 구분 — `test_same_subfolder_name_in_two_roots_has_distinct_fingerprint`: 주 루트와 추가 폴더에 둘 다 `sub`를 만들고, 작업 `a`를 `cwd=root/'sub'`로 한 번 묻은 뒤 `cwd=extra/'sub'`로 다시 물으면 `configuration_changed`로 거부, `new_session=True`면 성공. 추가 폴더 쪽 `configuration['cwd']`는 `os.path.normcase(str((extra/'sub').resolve()))`와 같음.
- [ ] `..` 우회 거부 — `test_dotdot_escape_from_extra_root_refused`: 추가 폴더의 형제 폴더를 `extra/'..'/'sibling'`으로 지정하면 `cwd_outside_allowed_roots`로 거부, `client.calls` 비어 있음.
- [ ] 링크 우회 거부 — `test_symlink_escape_from_extra_root_refused`: 추가 폴더 안에 바깥 폴더를 가리키는 디렉터리 심볼릭 링크를 만들고 그 링크를 `cwd`로 주면 거부. 링크 생성이 권한 문제(OSError)로 안 되면 `self.skipTest`.
- [ ] 대소문자·드라이브 문자 — `test_case_variant_cwd_same_fingerprint` (Windows 전용, 그 외 `skipUnless(os.name == 'nt')`): 추가 폴더 경로 문자열을 `swapcase()`해서 `cwd`로 주면 허용되고, 원래 표기로 한 번 묻은 작업을 바꾼 표기로 다시 물으면 `resumed`가 True.
- [ ] sandbox 범위 — `test_workspace_write_in_extra_root_passes_only_cwd`: `module.bounded_process`를 대체해 `CodexClient.ask()`가 만든 명령을 잡고(`checked_status`도 대체), 추가 폴더 하위에서 `workspace-write`로 부를 때 명령에 `-s workspace-write`, `-C <그 하위 폴더>`가 있고 `--add-dir`가 없음, `bounded_process`의 `cwd` 인자도 같은 폴더.
- [ ] 공개 오류 형식 — `test_outside_roots_error_is_identifier_only`(ProtocolTests): `handle()`로 루트 밖 `cwd`를 요청하면 `isError`가 True이고 텍스트가 정확히 `cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS`이며 요청 경로 문자열을 포함하지 않음.
- [ ] 자식 환경 비전달 — `test_allowlisted_environment`에 `STELLA_CODEX_EXTRA_ROOTS`를 넣어도 `client.env()`에 없음(기존 시험에 키 하나 추가).
- [ ] 문서·예시 — `tools/codex-bridge/mcp.example.json`이 `python -c "import json,sys;json.load(open(sys.argv[1],encoding='utf-8'))" tools/codex-bridge/mcp.example.json`로 읽히고 `STELLA_CODEX_EXTRA_ROOTS` 값이 자리표시자임. `tools/codex-bridge/README.md`와 `tools/BRIDGE_README.md`에 변수 이름, 구분자, 재연결 필요, "OS 읽기 경계 아님" 문장이 있음 — `grep -n "STELLA_CODEX_EXTRA_ROOTS" tools/codex-bridge/README.md tools/BRIDGE_README.md tools/codex-bridge/mcp.example.json`.
- [ ] 개인 정보 없음 — 바뀐 파일과 이 설계 문서에 실제 사용자 폴더·계정명이 없음: `git diff -- tools docs/tasks/issue-2-allowed-roots | grep -inE "users[\\/]|[a-z]:[\\/][^<]"` 출력이 시험 코드의 임시 폴더 처리 외에 없음(검증 담당이 눈으로 확인).

## 지켜야 할 제약

- 구독 로그인만 사용, API 키 대체 경로 금지: `CodexClient.checked_status()`·`forced_login_method` 부분은 건드리지 않는다.
- 자식 프로세스 환경은 `SAFE_ENV` 허용 목록 그대로. 새 환경변수를 Codex에 넘기지 않는다.
- 공개 오류는 영문 식별자만. 경로·예외 원문·환경변수 값을 응답에 넣지 않는다.
- 실제 모델 호출 없는 합성 시험만 쓴다. 시험은 임시 폴더만 만들고 사용자 자료를 읽지 않는다.
- 공개 저장소: 문서·예시·시험에 개인 경로·사용자 이름·계정·세션 ID를 쓰지 않는다. 예시는 `<project-folder>`, `<other-project-folder-1>` 같은 자리표시자.
- `.bridge/` 원시 상태는 계속 비공개·로컬 평문이며 공개하지 않는다.
- 자동 재시도·백그라운드 작업·다른 AI 호출을 추가하지 않는다.
- 미커밋 사용자 파일을 보존하고, `tools/` 밖 파일(디스패처 제외 범위 포함)은 고치지 않는다.

## 결정된 질문 (사용자, 2026-11-23)

1. 추가 폴더의 `workspace-write`: **허용한다.** 주 루트와 같은 규칙(요청마다 명시하면 허용)이다. `workspace_write_only_in_STELLA_CODEX_ROOT` 식별자는 만들지 않는다.
2. 오류 식별자: **새 이름 `cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS`로 바꾼다.** 기존 `cwd_outside_operator_root`는 쓰지 않는다.

## 열린 질문

없음.
