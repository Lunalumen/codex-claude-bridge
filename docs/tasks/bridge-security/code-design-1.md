# 1단계 코드 설계

## 요약

이번 답변은 **제작 전 코드 설계**다. 파일 생성·수정, 시험 실행, 다른 AI·모델 호출은 하지 않았다. `.bridge/` 대화 기록도 읽지 않았다.

기준은 `docs/tasks/bridge-security/design.md`의 **A와 H, 1단계 완료 기준**이다.

1단계의 결과는 다음과 같다.

- Codex 호출을 프로젝트 루트 하나와 `read-only`로 고정한다.
- 추가 루트·쓰기 선택 기능을 제거한다.
- MCP 루트와 디스패처 `--root`를 명시적으로 받는다.
- 두 방향의 status·성공 결과에 동일한 방식의 `project`, `root_id`를 제공한다.
- 상태를 만들기 전에 `.bridge/.gitignore`를 준비한다.
- 기존 루트의 읽기 전용 완료 대화는 기존 상태 형식으로 이어간다.

프로세스 종료 장치, 환경 허용 목록, 출력 처리, 공통 안내문 포장은 이후 단계에 둔다. **1단계 완료만으로 전체 보안 개선이나 실제 쓰기 차단 검증이 끝났다고 표시하지 않는다.**

## 파일·함수별 변경

### 1. `tools/codex-bridge/server.py`

**지울 것**

- `_special_path`
- `_read_documents_registry_value`
- `protected_paths`
- `_extra_root_entries`
- `parse_extra_roots`
- `check_root_layout`
- `Bridge.allowed_roots`
- `Bridge.__init__`의 `extra_roots`, `self.extra_roots`, `self.root_input`
- 추가 루트 환경변수를 설정으로 읽는 코드
- `Bridge.ask`의 실행 폴더 선택 및 `cwd_key` 계산
- `workspace-write` 허용
- `CodexClient.ask`의 `sandbox` 선택 인자
- 서버 폴더를 프로젝트 루트로 사용하는 기본값
- 도구 스키마의 `cwd`, 쓰기 선택지

`ROOT` 상수는 사용처를 구분한다. 모듈 위치를 찾는 용도로 필요하면 남기되, 프로젝트 루트나 로그인 점검 cwd의 기본값으로 사용하지 않는다.

**바꿀 것**

| 함수·부분 | 변경 |
|---|---|
| `Bridge.__init__` | 명시 인자 우선, 인자가 `None`일 때만 `STELLA_CODEX_ROOT` 사용. 루트 검사는 실제 기능 호출 시 수행 |
| `Bridge.ask` | 기존 입력 검증과 읽기 전용·고정 cwd 검사를 상태 생성 전에 수행 |
| `Bridge.task_dir` | 공통 검사에서 확정한 프로젝트 루트를 사용. 기존 기록 위치 유지 |
| `CodexClient.checked_status` | 프로젝트 루트를 받아 로그인·도움말 점검의 cwd로 사용 |
| `CodexClient.ask` | sandbox 선택 없이 고정 명령 생성 |
| `handle` | 도구별 허용 인자 검사. 옛 `cwd`만 호환 인자로 허용 |
| `TOOLS` | `sandbox` enum은 `["read-only"]`만 유지 |
| 성공 결과·status | 공통 루트 식별값 추가 |

`initialize`, `ping`, `tools/list`는 `Bridge()` 생성이나 루트 검사 없이 응답한다.

옛 cwd 호환 검사의 핵심은 다음과 같다. **존재 확인이나 링크 해석을 하지 않는다.**

```python
def validate_read_only_request(root, *, cwd=None, sandbox="read-only"):
    if sandbox != "read-only":
        raise ValueError("read_only_bridge_sandbox_not_allowed")

    if cwd is None:
        return
    if not isinstance(cwd, str) or not cwd:
        raise ValueError("cwd_fixed_to_project_root")

    try:
        expected = os.path.normcase(os.path.abspath(str(root)))
        requested = os.path.normcase(
            os.path.abspath(os.path.join(str(root), cwd))
        )
    except (OSError, ValueError, TypeError):
        raise ValueError("cwd_fixed_to_project_root") from None

    if requested != expected:
        raise ValueError("cwd_fixed_to_project_root")
```

따라서 다음과 같이 처리한다.

| 입력 | 결과 |
|---|---|
| sandbox 생략 또는 `read-only` | 허용 |
| sandbox `null`, 쓰기 값, 그 밖의 값 | `read_only_bridge_sandbox_not_allowed` |
| cwd 생략·`null`·`.`·정규화하면 루트인 문자열 | 허용 |
| 하위 폴더·다른 폴더·없는 폴더·비문자열·빈 문자열 | `cwd_fixed_to_project_root` |

실행에는 검사를 통과한 호출값이 아니라 **저장된 프로젝트 루트**를 사용한다.

지문은 새 형식으로 바꾸지 않는다.

```python
fingerprint = {
    "cwd": ".",
    "sandbox": "read-only",
    "model": model,
}
```

모르는 인자는 `handle`에서 도구별로 검사한다.

```python
ASK_ARGUMENTS = {
    "task_id", "prompt", "cwd", "sandbox",
    "model", "timeout_sec", "new_session",
}

if set(args) - ASK_ARGUMENTS:
    raise ValueError("unknown_argument")
```

status는 빈 인자만, history는 `task_id`만 허용한다. 오류에 인자 이름·값을 넣지 않는다. 직접 `Bridge.ask`를 부르는 경로도 같은 오류를 내도록 `**unknown` 검사나 별도 검증 함수를 둔다.

**Codex 고정 명령**

명령 조립은 시험하기 쉬운 `CodexClient.command(...)`로 분리한다.

```python
def command(self, *, root, output, session, model):
    result = self.command_prefix + [
        "exec",
        "--ignore-user-config",
        "--ignore-rules",
        "--skip-git-repo-check",
        "--json",
        "-s", "read-only",
        "-C", str(root),
        "-o", str(output),
        "-c", 'sandbox_mode="read-only"',
        "-c", 'forced_login_method="chatgpt"',
        "-c", 'model_provider="openai"',
    ]
    if model:
        result += ["-m", model]

    if session:
        result += [
            "resume",
            "-c", 'sandbox_mode="read-only"',
            session_uuid(session),
            "-",
        ]
    else:
        result += ["-"]

    return result
```

기존 `self.command`와 메서드 이름이 충돌하므로 실행 파일 접두 배열은 `self.command_prefix`로 바꾼다.

- 준비 점검에 `--ignore-rules` 지원 확인을 추가한다.
- `bounded_process(..., cwd=root)`도 같은 루트를 사용한다.
- `--add-dir`이나 권한 우회 옵션은 추가하지 않는다.
- 이 명령의 **실제 CLI 호환성과 이어가기의 쓰기 차단은 미확인**으로 남긴다.

### 2. 공통 새 파일: `tools/bridge_process.py`

설계의 새 공통 파일 한 개를 사용한다. 1단계에서는 아래 프로젝트 관련 함수만 넣고, 프로세스 실행기는 2단계에서 추가한다.

| 새 함수 | 역할 |
|---|---|
| `validate_project_root(value)` | 절대경로·기존 폴더·너무 넓은 루트 검사 |
| `project_identity(root)` | 두 방향에 동일한 `project`, `root_id` 생성 |
| `ensure_bridge_gitignore(root)` | 상태 생성 전 제외 파일 검사·생성 |

파일 이름은 최종 공통 실행기 이름을 따른다. 별도의 루트 유틸 파일까지 추가하지 않는다.

루트 검사의 주요 코드는 다음과 같다. `root_key`는 Windows에서 대소문자를 정규화하는 내부 비교 함수다.

```python
def root_key(path):
    return os.path.normcase(os.path.normpath(str(path)))


def validate_project_root(value):
    try:
        if value is None or not os.fspath(value):
            raise ValueError
        candidate = Path(value)
        if not candidate.is_absolute():
            raise ValueError
    except (TypeError, ValueError):
        raise ValueError("project_root_required") from None

    try:
        root = candidate.resolve(strict=True)
        if not root.is_dir():
            raise ValueError
    except (OSError, RuntimeError, ValueError):
        raise ValueError("project_root_invalid") from None

    try:
        raw_home = os.path.expanduser("~")
        if not raw_home or "~" in raw_home:
            raise ValueError
        home = Path(raw_home)
        if not home.is_absolute():
            raise ValueError
        home = home.resolve()
    except (OSError, RuntimeError, ValueError):
        raise ValueError("cannot_determine_home") from None

    if (
        root.parent == root
        or root_key(root) == root_key(home)
        or root_key(root) in {root_key(p) for p in home.parents}
    ):
        raise ValueError("project_root_too_broad")

    return root
```

`project_root_invalid`は設計に未記載のため、**追加する固定エラー識別子の提案**だ。存在しないルート、ファイル、参照できないルートを同じ識別子で拒否する。

식별값은 한 함수에서만 만든다.

```python
def project_identity(root):
    normalized = root_key(root).encode("utf-8")
    return {
        "project": root.name,
        "root_id": hashlib.sha256(normalized).hexdigest()[:12],
    }
```

이 값은 status·성공 결과에만 추가하고 모델에 보내는 글에는 넣지 않는다. 절대경로도 결과에 넣지 않는다.

제외 파일은 기존 파일을 덮어쓰지 않도록 배타적으로 생성한다.

```python
def ensure_bridge_gitignore(root):
    folder = root / ".bridge"
    folder.mkdir(exist_ok=True)
    path = folder / ".gitignore"

    try:
        fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    except FileExistsError:
        if path.read_bytes() != b"*\n":
            raise ValueError("bridge_gitignore_must_ignore_all")
    else:
        with os.fdopen(fd, "wb") as stream:
            stream.write(b"*\n")
```

실제 구현에서는 읽기·쓰기 실패도 경로 없는 고정 오류로 처리한다. 동시에 양쪽이 처음 시작해도 기존 파일을 덮지 않는다.

호출 위치는 다음과 같다.

- Codex: 요청·루트 검증 후, 작업 폴더와 잠금 생성 전.
- Claude: 기존 구독 로그인 실패 시 상태를 만들지 않는 동작을 보존하고, 로그인 점검 통과 후 작업 폴더 생성 전.
- status·history: 제외 파일을 생성하지 않음.

공통 파일은 각 브리지의 `__file__`에서 계산한 위치를 `importlib.util.spec_from_file_location`으로 읽는다. 현재 작업 폴더나 전역 `sys.path` 검색에 기대지 않는다.

### 3. `tools/bridge.py`

**지울 것**

- `--root`의 `TOOLS.parent` 기본값
- Codex ask의 `--cwd`, `--sandbox`
- 해당 값을 `Bridge.ask`에 전달하는 코드

**바꿀 것**

```python
parser.add_argument("--root", type=Path, required=True)
```

- 입력값을 먼저 `.resolve()`해서 상대경로를 숨기지 않는다. 공통 검사에 원래 값을 전달한다.
- 두 방향의 status에도 확정한 루트를 전달한다.
- Codex ask에는 `model`, `timeout`, `new_session`만 전달한다.
- 모르는 옵션을 모듈 로딩·파일 읽기 전에 argparse가 거부하게 유지한다.

Codex 전용 안내문 조립은 **3단계 F에서 옮긴다**. 1단계에서 없애면 MCP 공통 포장을 만들기 전에 현재 안내가 사라진다.

### 4. `tools/claude-bridge/bridge.py`

**1단계에서는 삭제할 기능 없음.**

| 함수·부분 | 변경 |
|---|---|
| `Bridge.__init__` | 단독 실행 기본 루트는 보존. 명시 루트에 공통 검사 적용 |
| `Bridge.ask` | 루트 검사 → 기존 로그인 점검 → 제외 파일 준비 → 기존 상태 처리 |
| `ClaudeClient.status` | 프로젝트 루트를 받아 `subprocess.run(..., cwd=root)` 사용 |
| 성공 결과·status | 공통 `project`, `root_id` 추가 |
| `main` | 단독 status도 기존 기본 루트를 검사한 뒤 사용 |

주입용 `FakeClient.status`도 루트를 받도록 바꾼다.

`NO_API_ENV`, 직접 `subprocess.run` 두 곳은 각각 3단계·2단계에서 교체한다. 참고 파일 개수·확장자·크기도 1단계에서 바꾸지 않는다.

### 5. 문서·설정 예제

| 파일 | 1단계 변경 |
|---|---|
| `tools/codex-bridge/README.md` | 기본 루트·추가 루트·쓰기 선택 안내 삭제. 필수 루트와 옛 호출 거부 동작 설명 |
| `tools/BRIDGE_README.md` | 필수 `--root`, 같은 루트·같은 작업 ID로 이어가기, 루트 비교 방법 |
| `tools/claude-bridge/README.md` | 프로젝트 작업은 디스패처 사용. 단독 실행 기본값은 호환용임을 명시 |
| `tools/codex-bridge/mcp.example.json` | 추가 루트 환경변수 줄 삭제. 필수 루트 자리표시 유지 |

문서에는 다음을 명시한다.

- 기존 기록은 원래 루트에 남으며 자동 이동하지 않는다.
- `.bridge/.gitignore`는 이미 추적된 파일·압축·배포·동기화를 보호하지 않는다.
- `read-only`는 쓰기 제한이며 다른 경로 읽기를 막는 경계가 아니다.
- 현재 단계의 확인은 합성 시험 범위다.
- 프로젝트별 local 등록 문법·호스트 적용은 확인 전까지 미확인으로 표시한다.

`docs/BRIDGE_WORKFLOW.md`의 왕복 규칙·공통 자료 안내는 3단계에서 고친다.

## 작업 순서와 시험

아래 명령은 **제작 지시를 받은 뒤 실행할 계획**이다. 이번에는 실행하지 않았다.

### 1. `tools/` 백업과 해시 일치 확인

첫 작업은 `.bridge-backups/`의 **새 폴더**에 현재 `tools/`를 복사하는 것이다. Git 복구만 믿지 않는다.

검증 순서:

1. 백업 위치의 Git 제외 여부 확인.
2. 원본의 상대 파일명·SHA-256 목록 산출.
3. 숨김 파일을 포함해 복사.
4. 백업 해시 목록과 원본 재산출 목록을 비교.
5. 파일 수·상대 파일명·해시가 모두 같아야 다음 순서 진행.

확인 명령:

```powershell
git check-ignore .bridge-backups/bridge-security-stage1/
```

해시 목록은 다음 형태로 원본·백업 각각 산출한다.

```powershell
Get-ChildItem -LiteralPath tools -Recurse -File -Force |
    Get-FileHash -Algorithm SHA256
```

비교할 때 절대경로 대신 각 트리 기준 상대경로를 사용한다. 원본이 복사 중 바뀌어도 실패로 처리한다. 백업 내용·해시 목록은 공개 문서에 싣지 않는다.

이후 기존 합성 시험으로 기준 상태를 기록한다.

```text
python -B -X utf8 tools/codex-bridge/test_server.py -v
python -B -X utf8 tools/test_bridge_dispatcher.py -v
python -B -X utf8 tools/claude-bridge/test_bridge.py -v
```

### 2. 공통 루트 검사·식별값·Git 제외 함수 작성

두 브리지의 import 경로와 status cwd 전달을 함께 고친다.

뒤에 돌릴 시험:

```text
python -B -X utf8 tools/codex-bridge/test_server.py StoreTests.test_project_root_required StoreTests.test_project_root_too_broad StoreTests.test_bridge_state_git_ignored -v
python -B -X utf8 tools/claude-bridge/test_bridge.py BridgeTests.test_project_root_too_broad BridgeTests.test_bridge_state_git_ignored -v
```

### 3. 추가 루트 제거·읽기 전용 고정·기존 지문 호환

옛 기능 시험을 지우고 새 정책 시험으로 교체한다.

뒤에 돌릴 시험:

```text
python -B -X utf8 tools/codex-bridge/test_server.py StoreTests ProtocolTests -v
```

### 4. Codex 고정 argv·프로젝트 cwd 적용

`CodexClient`와 직접 호출 시험·가짜 도움말을 함께 수정한다.

뒤에 돌릴 시험:

```text
python -B -X utf8 tools/codex-bridge/test_server.py ClientTests -v
```

### 5. 디스패처 옵션 제거·양쪽 status 경로 연결

뒤에 돌릴 시험:

```text
python -B -X utf8 tools/test_bridge_dispatcher.py -v
python -B -X utf8 tools/claude-bridge/test_bridge.py -v
```

### 6. 문서 정리와 최종 회귀 확인

```text
rg -n "EXTRA_ROOTS|extra_roots|allowed_roots|protected_paths|_special_path|winreg|workspace-write" tools/codex-bridge/server.py tools/bridge.py tools/codex-bridge/mcp.example.json
rg -n "EXTRA_ROOTS|workspace-write" tools/BRIDGE_README.md tools/codex-bridge/README.md
python -B -X utf8 tools/codex-bridge/test_server.py -v
python -B -X utf8 tools/test_bridge_dispatcher.py -v
python -B -X utf8 tools/claude-bridge/test_bridge.py -v
```

最初の `rg` は一致なしを成功とするため、終了コードは通常 `1` になる。試験失敗と混同しない。

最後にバックアップとの差分で変更ファイルを確認し、既存の未コミット変更を保ったことを記録する。制作後の `implementation.md` には実行結果・スキップ理由・未確認事項を書く。次段階は開始しない。

## 完了基準と試験の対応

### 追加・拡張する試験

| 完了基準 | 試験名と確認内容 |
|---|---|
| 追加ルート・書き込み選択コードなし | 上記のコード対象 `rg` が一致なし |
| README に単一ルート設定法がある | 文書対象 `rg` と例の目視確認 |
| 拒否を状態生成前に実施 | `test_read_only_enforced_before_state`：sandbox・cwd・未知引数ごとに識別子、状態なし、準備点検・ask 呼び出し 0 |
| フォルダの有無で拒否が変わらない | 同試験で存在するサブフォルダ・存在しないフォルダを対比 |
| 旧 read-only 呼び出し許容 | `test_legacy_read_only_root_request_accepted`：sandbox 省略/read-only × cwd 省略/null/`.`/ルート絶対文字列 |
| 既存の完了会話を継続 | `test_existing_state_resumes_after_upgrade` をそのまま維持 |
| 旧書き込み・別 cwd 状態を拒否 | `test_legacy_write_or_other_cwd_state_requires_new_session`：拒否前後の `state.json` が同一、明示的新規のみ許容 |
| 新規・継続とも固定コマンド | `test_codex_command_read_only_new_and_resume`：オプション位置、ルート、`--add-dir` 不在 |
| スキーマに書き込み選択なし | `test_tool_schema_has_no_write_choice`：cwd なし、sandbox enum 固定 |
| 旧ディスパッチャオプション拒否 | `test_removed_codex_options_rejected`：`--sandbox workspace-write`、`--cwd x` が各 rc 2、モジュール読み込み前に終了 |
| ルート必須 | `test_project_root_required`：未設定・空・相対を拒否。cwd・他環境変数から推測しない |
| プロトコル初期化はルート不要 | `test_initialize_and_list_without_model`：ルート環境変数を消し、Bridge 構築禁止の mock でも成功 |
| 広すぎるルート拒否 | 両側 `test_project_root_too_broad`：最上位・ホーム・ホーム上位拒否、通常プロジェクト許容。状態・CLI 呼び出し 0 |
| ホーム不明・不正ルート | `test_project_root_home_unavailable`、`test_project_root_invalid`：固定識別子、内部パス非公開 |
| 両側のルート識別が一致 | `test_root_identity_matches`：同一は一致、別ルートは異なる、絶対パスなし |
| Git に記録が出ない | 両側 `test_bridge_state_git_ignored`：一時 Git リポジトリで `git status --porcelain --untracked-files=all` を確認 |
| 異なる既存除外ファイルを保護 | 同試験で異なる内容を拒否し、ファイルと既存状態のバイト不変を確認 |
| 旧状態に除外ファイルだけ追加 | 同試験で合成旧状態を用意し、既存ファイルを変更せず `.gitignore` のみ追加 |
| プロジェクト間分離 | `test_projects_isolated`：2 ルート × 両 provider に同じ task を使用、各ルート内でのみ継続 |
| 点検もプロジェクト cwd | `test_status_uses_project_root`：両側の子プロセス呼び出し引数を確認 |
| 任意の読み込み方に対応 | `test_shared_module_loads_without_cwd_dependency`：別 cwd・importlib 読み込みで共通ファイルを正しく発見 |

Git 試験は Git が使用できなければ理由付きスキップとし、この完了基準は未確認扱いにする。

### 削除する試験

`tools/codex-bridge/test_server.py` から次を削除する。追加ルートの旧契約を試すため、新契約では維持しない。

```text
test_default_without_extra_roots_unchanged
test_extra_root_allowed_state_stays_in_primary_root
test_extra_roots_from_environment
test_invalid_extra_roots_refused_without_leaking
assert_roots_refused
test_overlapping_roots_refused
test_extra_root_via_link_inside_other_root_refused
test_non_overlapping_roots_and_duplicates_allowed
test_too_broad_extra_roots_refused
test_special_path_notation_extra_roots_refused
test_special_notation_primary_root
test_extra_root_must_be_real_path
test_cannot_determine_home_refuses_extra_roots
test_invalid_optional_protection_env_skipped
test_powershell_profile_and_claude_config_protected
test_redirected_documents_profile_protected
test_documents_registry_reader_non_windows_and_errors
test_same_subfolder_name_in_two_roots_has_distinct_fingerprint
test_dotdot_escape_from_extra_root_refused
test_symlink_escape_from_extra_root_refused
test_case_variant_cwd_same_fingerprint
test_workspace_write_in_extra_root_passes_only_cwd
test_outside_roots_error_is_identifier_only
```

`StoreTests.setUp` のレジストリ mock と追加ルート用準備も削除する。ホーム検査用に必要な最小の合成ホームは新たに用意する。

### 修正して残す試験

- `test_configuration_change_refused_then_explicit_new`：モデル変更を対象にする。新しい会話でも書き込みは許容しない。
- `test_cwd_outside_root`：期待値を `cwd_fixed_to_project_root` にする。
- `test_allowlisted_environment`：既存の環境フィルター回帰を維持する。
- `test_bridge_checks_status_once_per_turn`：ルート引数と新しいクライアントシグネチャに合わせる。
- `test_real_subprocess_fake_events_and_resume_flags`：固定コマンドと新シグネチャに合わせる。
- `test_dispatcher_passes_task_and_context_to_codex`：sandbox の受け渡し確認を、cwd/sandbox 引数を渡さない確認に変更する。
- `test_preflight_failure_keeps_previous_conversation`：新しいヘルプ文字列に対応し、状態不変の検証を維持する。

実行ファイル関連試験は残す。ルート内実行ファイル拒否への変更は 3 段階で行う。

次の主要な既存試験も維持する。

- `test_exact_resume_and_redacted_history`
- `test_tasks_do_not_share_sessions`
- `test_failure_no_retry_and_explicit_reset`
- `test_lock`
- `test_failed_new_session_preserves_previous_state`
- `test_orphan_turn_folder_is_skipped_and_kept`
- `test_mcp_resume_uses_same_store`
- `test_omitted_task_id_gets_unique_one_off_session`
- Claude 側 `test_record_and_resume`
- Claude 側 `test_separate_tasks_and_explicit_new`
- Claude 側 `test_no_api_fallback`
- Claude 側 `test_lock_prevents_second_call`
- Claude 側 `test_timeout_no_retry_or_implicit_new_session`

削除対象に挙げていない既存試験は維持し、インターフェース変更に必要な修正だけ行う。

## 既存会話の互換性と復旧

**会話の保存位置・状態スキーマ・provider ごとの分離は変更しない。**

| 既存状態 | 更新後の動作 |
|---|---|
| 同じルート、完了、cwd `.`、read-only、同じ model | 同じ保存セッションで継続 |
| 書き込み sandbox・サブフォルダ・追加フォルダの指紋 | 暗黙の継続を拒否。既存状態を変更しない |
| `failed` / `inflight` | 現在どおり明示的新規が必要 |
| `turn.lock` あり | 新規指定でも拒否。自動解除しない |
| 別プロジェクトルート | 独立した新しい会話 |
| Claude の既存完了状態 | 同じルート・task で既存の継続処理を使用 |

旧書き込み状態の試験は、新コードで書き込みセッションを生成せず、**合成の旧状態ファイルを直接用意**する。

復旧は次のように行う。

1. 旧サーバーを正常終了し、動作中の呼び出しがない状態で更新する。
2. 途中で止まったら再起動前に変更ファイルを確認する。異なる版のファイルを混在させない。
3. コードを戻す場合はバックアップのファイル単位で復元し、バックアップとハッシュを照合する。
4. `.bridge/` はコード復元の対象に含めない。更新後に増えた会話を旧バックアップで上書きしない。
5. 1 段階で追加した `.bridge/.gitignore` は会話とは別の除外設定として残せる。
6. 呼び出し途中の異常終了で残ったロックを、コードを戻しただけで解除しない。子プロセスの停止確認が先になる。

**1 段階には安全な自動 unlock がない。** その機能はプロセス寿命の改善と一緒に 2 段階で実装する。

## 設計に対する異論

1. **削除表と段階分けが一致していない。**  
   全体の削除表には `stop_tree` の taskkill、`MAX_LINE`、Claude の `NO_API_ENV`、直接実行、案内文の移動まである。しかし段階表ではそれぞれ 2・3 段階だ。現在の `bounded_process` は `MAX_LINE` を参照し、`finally` で `stop_tree` を呼ぶ。1 段階で先に消すと動かなくなるため、段階別削除と明記すべきだ。

2. **不正ルートのエラー識別子が不足している。**  
   H は「存在するフォルダ」を要求するが、不在・ファイル・参照失敗時の識別子がない。`project_root_invalid` への統一を提案する。元の OS 例外は公開しない。

3. **Git 除外の既存内容比較が曖昧だ。**  
   `*` と LF、CRLF、BOM 付きのどれを同じとするか未定義だ。上記案では新規は `b"*\n"`、既存もバイト一致のみとした。安全だが、意味的に同じ CRLF ファイルでも拒否する。許容するなら契約と試験に明記すべきだ。

4. **既存会話の継続には条件が付く。**  
   H-7 に従うと、既存の `.bridge/.gitignore` が異なるだけでも新しいターンを拒否する。したがって「既存会話はそのまま継続」は、ルート検査・除外設定検査も通る場合に限る。既存ファイルを自動修正して互換性を保つ案には反対する。

5. **共通フォルダを指定しても、両者がそのコードを直接読めるわけではない。**  
   現在の Claude コマンドは `--tools ''` と、渡した文章だけを使う指示になっている。Claude は共通ルートを cwd にしてもソースを直接読めない。1 段階の表示は「ルートと保存先が一致」であり、「同じコードを両者が閲覧済み」にはできない。

6. **固定 argv の合成試験では実際の書き込み制限を証明できない。**  
   `--ignore-rules` と resume 後の設定が実際に受理・適用されるかは今回確認していない。1 段階は命令生成と拒否処理の確認までとし、4 段階の実試験を省略しない。

## 除外する二つの意見

**転送前の秘密検査：今回の必須機能から外すことに賛成する。**

文字列パターン検査は、普通のコードを誤って止める一方、任意の形式の秘密や分割した値を見逃す。秘密を含む通常の `.md` が選ばれれば、そのまま送信され得る問題は残るが、検査器を通ったことを安全の証明にはできない。

今回は明示的に選んだ最小限の資料を送る設計と、秘密の読み取り境界ではないという説明を採る。内容検査の追加は別設計とする。

**メッセージごとの発信者・返信 ID：現在の同期処理では除外に賛成する。**

現在は provider ごとに保存先が分かれ、一つの task をロックし、ターン番号と保存セッションで継続している。返信を別の依頼に対応付ける非同期ルーターもないため、追加 ID の必要性は低い。

ただし task ID は返信の重複防止を保証しない。例えば、応答が呼び出し元に届かなかった後に同じ task で手動再送すると、次のターンとして再実行され得る。今回は自動再試行をしない契約で扱い、将来非同期・並列返信を導入する際に再検討する。

## 確認できなかったこと

- 新コードと新試験は未作成で、今回の試験実行は 0 回。
- バックアップの作成・ハッシュ照合は未実施。
- `.bridge/` の実際の記録・ロック・除外ファイルは未読。
- 実際の CLI ヘルプ、ログイン、固定オプションの受理・適用は未確認。
- 実モデルでの継続と、新規・継続双方の書き込み遮断は未確認。
- プロジェクト別 local 登録の構文・ホストの設定優先順位・再接続後の適用は未確認。
- Windows Job Object、残存子プロセス、POSIX 動作は今回の設計対象外。
- ファイル生成・変更、他 AI 呼び出し、登録変更、コミット・公開は行っていない。