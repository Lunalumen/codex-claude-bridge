# 설계: Codex 브리지 허용 폴더 목록 — 보안 수정 설계 (2판)

- 작업ID: `issue-2-allowed-roots`
- 관련 이슈: #2
- 작성일: 2026-11-23
- **보안 수정 설계.** 근거 파일: 같은 폴더의 `security.md`(프로그램 보안, 수정 필요), `inspection.md`(검사, 수정 필요). 기준 설계: `design.md`(1판). 제작 보고: `implementation.md`.
- 이 작업의 보안 재수정은 이번 1회뿐이다. 이 판 이후에도 남는 문제는 "열린 질문"으로 사용자에게 넘긴다.

## 목표

1판 기능은 그대로 두고 다음 세 가지를 막는다. (1) 허용 폴더끼리 겹치는 설정. 이것이 있으면 Codex 턴이 다음 턴의 허용 경계를 옮길 수 있다. (2) 홈 폴더처럼 너무 넓은 폴더를 추가 폴더로 여는 설정. (3) 개인 절대 경로가 든 Codex 실행 부산물이 공개 커밋에 들어가는 것. `STELLA_CODEX_EXTRA_ROOTS`가 없을 때의 동작은 1판과 같이 변경 전과 완전히 같다.

## 범위

바꿀 파일과 함수:

- `tools/codex-bridge/server.py`
  - `parse_extra_roots(value)`: 항목별 검사에 "너무 넓은 폴더" 거부(아래 결정 2)를 더한다. 반환 형식(`resolve()`된 `Path` 목록, 중복 제거)은 그대로 둔다.
  - 새 모듈 함수 `protected_paths()`: 결정 2의 보호 경로 두 묶음을 돌려준다.
  - 새 모듈 함수 `check_root_layout(entries)`: 결정 1의 겹침 검사를 한다. `entries`는 `(입력 절대 표기 Path, resolve된 Path)` 쌍의 목록이다.
  - `Bridge.__init__`: 주 루트의 입력 절대 표기 `self.root_input = Path(os.path.abspath(root or os.environ.get('STELLA_CODEX_ROOT') or ROOT))`를 추가로 저장한다. `self.root` 계산은 바꾸지 않는다.
  - `Bridge.allowed_roots()`: 추가 폴더가 하나라도 있으면 반환 전에 `check_root_layout`을 부른다. 추가 폴더가 없으면 지금과 같이 `[self.root]`만 돌려주고 새 검사는 하지 않는다.
- `tools/codex-bridge/test_server.py`: 아래 완료 기준의 새 시험을 추가한다. 겹침 거부 때문에 깨지는 기존 시험 두 개는 입력 폴더 위치만 고친다(결정 1의 "기존 시험 조정").
- `tools/codex-bridge/README.md`, `tools/BRIDGE_README.md`: 새 거부 규칙 두 개와 경고 문장을 넣는다.
- `.gitignore`: 한 줄 추가(결정 3). 1판의 "`tools/` 밖 파일은 고치지 않는다" 제약에 대한 이 판의 유일한 예외다.

바꾸지 않을 것 (이전 판 유지):

- `handle()`이 호출마다 `Bridge()`를 만드는 구조와 `main()`. 서버 시작 시 1회 해석·고정은 하지 않는다(결정 1의 이유 참고).
- `Bridge.ask()`의 `cwd` 검사 순서, 지문(`fingerprint`) 규칙, `task_dir()`, 상태 파일 형식.
- `CodexClient`, `bounded_process`, `SAFE_ENV`, `codex_launcher`, `TOOLS` 스키마, `tools/bridge.py`, `tools/claude-bridge/`.
- 주 루트(`STELLA_CODEX_ROOT`) 자체의 넓이. 주 루트에는 새 넓이 규칙을 적용하지 않는다. 추가 폴더가 없을 때 동작이 바뀌면 안 되기 때문이다.
- 사용자 결정 2개(추가 폴더에서도 `workspace-write` 허용, 식별자 `cwd_outside_allowed_roots_set_STELLA_CODEX_EXTRA_ROOTS`) — 이전 판 유지.
- 1판의 동작 설계 1~9절 가운데 아래에서 언급하지 않은 부분 — 이전 판 유지.

## 동작 설계

### 검사 순서 (`ask()` 안, 1판 4절의 4단계 `roots = self.allowed_roots()`에서)

1. `parse_extra_roots`가 항목별로 검사한다. 1판의 1~4단계(절대 경로, 실제로 있음, 폴더임, 파일시스템 최상위 아님) 뒤에 새 5단계 "너무 넓은 폴더 아님"을 넣고, 중복 제거는 그다음에 한다.
   - 1~4단계 실패: `invalid_STELLA_CODEX_EXTRA_ROOTS_entry` (이전 판 유지)
   - 5단계 실패: `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry` (새 식별자)
2. 항목이 하나 이상 있으면 `check_root_layout`을 부른다. 실패하면 `overlapping_allowed_roots_STELLA_CODEX_EXTRA_ROOTS` (새 식별자).
3. 어느 단계든 실패하면 설정 전체를 거부한다. 거부는 `folder.mkdir`·잠금·`client.ask`보다 먼저 일어난다. `history`·`status`는 1판처럼 영향을 받지 않는다(지연 검사, 이전 판 유지).
4. 새 식별자 두 개는 `[A-Za-z0-9_]+` 형식이다. 따라서 `handle()`이 그대로 내보내고, 경로나 설정값은 넣지 않는다. 예외는 `from None`으로 연결을 끊는다(기존 방식과 같음).

`parse_extra_roots`가 겹침 검사용 입력 표기도 넘겨야 한다. 그래서 내부 보조 함수 하나(이름은 Codex 재량, 예: `_extra_root_entries(value)`)가 `(Path(os.path.abspath(entry)), resolved)` 쌍을 만들게 하고, `parse_extra_roots`는 그 결과에서 resolved만 뽑아 중복을 없앤 목록을 돌려준다. 공개 이름과 반환 형식은 바뀌지 않는다.

---

### 결정 1. 허용 폴더 겹침 (security.md 중간 1번)

**무엇을 막는가.** 상위 허용 폴더에서 `workspace-write`로 도는 Codex 턴이 하위 허용 폴더(주 루트 포함)나 그 경로 중간 폴더를 정션·링크로 바꾸는 경우다. 그러면 다음 호출에서 루트를 다시 해석할 때 허용 경계와 `.bridge/` 위치가 옮겨진다. 설정 항목의 입력 경로가 다른 허용 폴더 안에 있는 링크를 거치는 경우(예: `<A>/link`를 항목으로 넣었는데 `<A>`도 허용 폴더인 경우)도 같은 문제다.

**선택: 겹침 거부를 쓰되, 입력 표기와 실제 경로를 둘 다 비교한다.** 서버 시작 시 고정과 링크 항목 일괄 거부는 쓰지 않는다.

`check_root_layout(entries)` 규칙:

- `entries` = `[(self.root_input, self.root), *(각 추가 항목의 (입력 절대 표기, resolved))]`. 중복을 없애기 **전의** 전체 목록으로 검사한다.
- 서로 다른 두 항목 i, j(순서쌍 전부)에 대해 다음 중 하나라도 참이면 거부한다.
  - `i_resolved != j_resolved` 이고 `i_resolved.is_relative_to(j_resolved)` — 실제 경로끼리 상하위 관계
  - `i_input != j_resolved` 이고 `i_input.is_relative_to(j_resolved)` — 입력 표기가 다른 허용 폴더의 실제 경로 안을 지남(링크·정션이 그 폴더 안에 있다는 뜻)
- 실제 경로와 입력 표기가 모두 같은 항목(단순 중복, 추가 항목이 주 루트와 같은 경우)은 거부하지 않는다. 지금처럼 중복 제거로 합친다(이전 판 유지).
- 비교는 `Path` 객체로 한다. Windows에서는 `WindowsPath`의 대소문자 무시 비교를 그대로 쓴다(1판 4-5와 같은 근거).

**왜 이 조합인가.**

- 겹침이 없으면 어떤 허용 폴더도 다른 허용 폴더의 실제 경로 구성요소를 포함하지 않는다. 그러므로 한 폴더에서 `workspace-write`로 도는 턴은 다른 허용 폴더의 경로를 바꿀 수 없다. 입력 표기 비교는 남는 구멍, 즉 "항목이 다른 허용 폴더 안의 링크를 거쳐 지정된 경우"를 막는다.
- 서버 시작 시 1회 해석·고정은 하지 않는다. 겹침이 없으면 호출마다 다시 해석해도 같은 결과가 나온다. 고정하려면 `handle()`/`main()` 구조와 1판의 "지연 검사" 방식을 바꿔야 해서 가장 작은 변경이 아니다. 오히려 호출마다 다시 검사하므로, 실행 중에 누군가 설정 폴더를 바꿔 겹침이 생기면 다음 `ask`가 실패한다(실패 시 닫힘).
- 링크·정션인 항목을 일괄 거부하지 않는다. 링크가 어떤 허용 폴더 밖에 있으면 브리지가 띄운 Codex가 그 링크를 바꿀 수 없으므로 위험이 아니다. 일괄 거부는 macOS의 `/var`→`/private/var` 같은 시스템 링크나 8.3 짧은 이름 때문에 정상 설정을 잘못 거부할 수 있다. 위험한 경우(링크가 허용 폴더 안에 있음)는 입력 표기 비교가 막는다.
- 주 루트의 입력 표기도 같은 규칙에 넣는다. 주 루트가 추가 폴더 안의 링크를 거쳐 지정되면 `.bridge/` 위치가 옮겨질 수 있기 때문이다. 이 검사는 추가 폴더가 있을 때만 돈다. 추가 폴더가 없을 때 동작은 이전 판과 같다.

**남는 위험(문서화만 함).** 입력 표기에 8.3 짧은 이름을 쓰고 그 경로가 다른 허용 폴더 안의 링크를 거치면 입력 표기 비교가 놓칠 수 있다. 실제 경로 비교는 계속 동작한다. 또 Codex CLI의 `workspace-write`는 임시 폴더 쓰기도 허용하므로, 허용 폴더나 그 링크를 시스템 임시 폴더 안에 두지 말라고 README에 적는다.

**기존 시험 조정 (Codex가 고칠 것).** 아래 두 시험은 추가 폴더를 주 루트 안(`self.root / 'a'` 등)에 만들기 때문에 새 규칙에서 겹침으로 거부된다. 시험이 보려던 내용(환경변수 파싱, 잘못된 항목 거부)은 그대로 두고, 추가 폴더를 주 루트와 별개인 `tempfile.TemporaryDirectory()` 안으로 옮긴다.
- `test_extra_roots_from_environment`: `a`, `b`를 별도 임시 폴더 아래로 옮긴다. `extra_roots=(a, a, self.root)`의 중복 제거 확인(`[root, a]`)은 유지한다.
- `test_invalid_extra_roots_refused_without_leaking`: `valid`를 별도 임시 폴더로 옮긴다. `file.txt`, `missing`, `relative`, `Path(self.root.anchor)` 경우는 그대로 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`를 기대한다(항목별 검사가 겹침 검사보다 먼저).

**완료 기준**

- [ ] `test_overlapping_roots_refused` (StoreTests): 임시 폴더 `outer` 아래에 `primary = outer/'primary'`, 형제 `outer/'a'`를 만들고 `Bridge(primary, FakeClient(), extra_roots=...)`로 다음 세 경우를 각각 확인한다. (1) `[outer]`(주 루트를 포함함) (2) `[primary/'inner']`(주 루트 안) (3) `[outer/'a', outer/'a'/'b']`(추가 폴더끼리 상하위). 세 경우 모두 `ask`가 정확히 `overlapping_allowed_roots_STELLA_CODEX_EXTRA_ROOTS`로 실패하고, `client.calls == []`이며 `primary/.bridge`가 없어야 한다. `history('a')['status'] == 'not_started'`도 확인한다. `handle()` 응답 텍스트는 식별자와 정확히 같고, 넣은 경로 문자열을 포함하지 않는다.
- [ ] `test_extra_root_via_link_inside_other_root_refused` (StoreTests): `outer/'a'` 안에 `outer/'t'`를 가리키는 디렉터리 링크 `outer/'a'/'j'`를 만든다. 링크는 `os.symlink`로 먼저 시도하고, 실패하면 Windows에서 `_winapi.CreateJunction(str(target), str(link))`로 만든다. 둘 다 안 되면 `self.skipTest`. 두 경우를 확인한다. (1) 추가 항목 `[outer/'a', outer/'a'/'j']` (2) 주 루트를 `outer/'a'/'jp'`(→ 별도 폴더 `outer/'p'`를 가리키는 링크)로, 추가 항목을 `[outer/'a']`로 둔 경우. 둘 다 `overlapping_allowed_roots_STELLA_CODEX_EXTRA_ROOTS`로 실패하고 `client.calls == []`이어야 한다. 정리할 때는 링크를 먼저 지운다.
- [ ] `test_non_overlapping_roots_and_duplicates_allowed` (StoreTests): 서로 겹치지 않는 형제 폴더 두 개, 같은 항목을 두 번 넣은 경우, 주 루트와 같은 항목을 넣은 경우는 거부되지 않는다. `allowed_roots()`가 `[root, a, b]`(resolve 기준)이다.
- [ ] 기본값 불변 — 1판의 `test_default_without_extra_roots_unchanged`가 그대로 통과한다(추가 폴더가 없으면 `check_root_layout`을 부르지 않음).
- [ ] 명령: `python -B -X utf8 tools/codex-bridge/test_server.py -v` 실패·오류 0. `python` 명령이 없는 환경에서는 로컬 Python 실행 파일을 직접 지정하고 보고서에는 `<python-executable>`로 적는다.

---

### 결정 2. 너무 넓은 추가 폴더 (security.md 중간 2번)

**무엇을 막는가.** 홈 폴더나 그 상위 같은 넓은 폴더를 추가 폴더로 등록하면, AI 요청 하나(`sandbox='workspace-write'`, `cwd=<그 폴더>`)만으로 셸 프로필, 시작 프로그램 폴더, `.codex`·`.claude` 설정·인증 파일, 브리지 코드 자체에 쓰기가 열린다. 사용자 결정 "추가 폴더에서도 `workspace-write` 허용"은 유지한다(이전 판 유지). 그 대신 등록할 수 있는 폴더의 넓이를 제한한다.

**선택: `parse_extra_roots`의 항목별 5단계에서 보호 경로와 겹치는 항목을 거부한다.**

`protected_paths()`는 호출할 때마다 환경에서 읽어 두 묶음을 돌려준다. 값이 비었거나 `~`가 풀리지 않은 항목은 건너뛴다. 각 경로는 `Path(...).resolve()`(strict=False)로 정규화한다.

| 묶음 | 경로 | 거부 조건 (항목 `r`, 보호 경로 `p`) |
|---|---|---|
| A. 넓은 폴더 | 홈 `os.path.expanduser('~')`, `LOCALAPPDATA` | `p.is_relative_to(r)` — 항목이 보호 경로 자신이거나 그 상위(사용자 프로필 루트·드라이브 바로 아래 포함) |
| B. 설정·실행 코드 | `CODEX_HOME`(있으면), `<홈>/.codex`, `<홈>/.claude`, `<홈>/.config`, `<홈>/.ssh`, `APPDATA`, 브리지 코드 폴더 `ROOT.parent`(= `tools/`, Codex·Claude 브리지와 디스패처 포함) | `p.is_relative_to(r) or r.is_relative_to(p)` — 항목이 보호 경로이거나 그 상위이거나 그 안 |

거부하면 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`를 낸다(경로 없음).

**판단 근거.**

- 홈과 그 상위를 막으면 셸 프로필(`~/.bashrc` 등)과 홈 바로 아래 설정 파일(`~/.claude.json` 등)을 포함하는 폴더를 모두 막게 된다. "사용자 프로필 루트"(홈의 부모)도 같은 조건에 걸린다.
- `LOCALAPPDATA`는 A 묶음(상위만 거부)에 둔다. 시스템 임시 폴더가 보통 그 안에 있어서, 안쪽까지 막으면 합성 시험과 정상 사용이 깨진다. `<홈>/AppData` 같은 상위는 이 조건으로 막힌다.
- `APPDATA`는 B 묶음(안쪽도 거부)에 둔다. Windows 시작 프로그램 폴더와 npm 전역 실행 파일(Codex CLI 실행 진입점 포함)이 그 안에 있어 sandbox 밖에서 실행되는 코드를 심을 수 있기 때문이다.
- `.codex`·`.claude`·`CODEX_HOME`은 인증·과금·로그인 설정이고, `.ssh`는 인증 키, `.config`는 POSIX 자동 시작·셸 설정이다. 브리지 코드 폴더는 다음 실행 때 sandbox 밖에서 도는 코드다. 이것들은 안쪽까지 막는다.
- 주 루트는 이 규칙에서 제외한다(이전 판 유지). 다만 추가 폴더가 주 루트를 포함하거나 그 안에 있는 경우는 결정 1이 막는다.
- `C:\Windows`, `Program Files`, `/etc`, `/usr`처럼 일반 사용자 권한으로 쓸 수 없는 시스템 폴더는 목록에 넣지 않는다. 브리지가 일반 사용자 권한으로 돌기 때문에 OS가 이미 막는다.
- "추가 폴더는 `read-only`만, 쓰기는 폴더별 운영자 표시가 있을 때만" 안은 쓰지 않는다. 사용자 결정 1과 충돌하고, 설정 형식이 바뀌기 때문이다.
- README에 경고 문장 하나를 넣는다. "추가 폴더 전체가 AI 요청만으로 쓰기 가능해지므로 필요한 프로젝트 폴더만 넣는다. 시스템 임시 폴더 안은 피한다."

**시험 격리.** 시험은 `patch.dict(os.environ, {...})`로 `USERPROFILE`·`HOME`을 임시 가짜 홈으로, `APPDATA`·`LOCALAPPDATA`를 가짜 홈 아래 경로로 바꾸고, `CODEX_HOME`은 지운다. Windows의 `expanduser`는 `USERPROFILE`을 먼저 읽으므로 가짜 홈이 적용된다. 기존 시험의 추가 폴더(시스템 임시 폴더 아래)는 보호 경로의 상위가 아니므로 영향을 받지 않는다.

**완료 기준**

- [ ] `test_too_broad_extra_roots_refused` (StoreTests): 임시 폴더 `base` 아래에 가짜 홈 `base/'home'`과 그 안의 `.codex`, `.claude`, `AppData/Roaming`, `AppData/Local`, `projects/p`를 만든다. 위처럼 환경변수를 패치한 뒤, 다음 항목을 하나씩(각각 유효한 형제 항목과 섞어서) 넣으면 `ask`가 정확히 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`로 실패하고 `client.calls == []`, 주 루트 `.bridge`가 없어야 한다.
  - 가짜 홈 자체, 가짜 홈의 부모(`base`)
  - `home/'.codex'`, `home/'.codex'/'sub'`, `home/'.claude'`
  - `home/'AppData'`(LOCALAPPDATA의 상위), `home/'AppData'/'Roaming'/'x'`(APPDATA 안)
  - 별도 임시 폴더를 `CODEX_HOME`으로 지정하고 그 폴더를 항목으로 넣은 경우
  - `Path(module.__file__).resolve().parent.parent`(브리지 코드 폴더)와 그 부모(저장소 루트)
- [ ] 같은 시험의 대조군: `home/'projects'/'p'`와 `home/'AppData'/'Local'/'proj'`(LOCALAPPDATA 안, 허용)는 `ask`가 성공하고 `FakeClient`가 받은 `cwd`가 그 폴더다.
- [ ] `handle()` 응답 텍스트가 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`와 정확히 같고 넣은 경로 문자열을 포함하지 않음 — 같은 시험 안에서 한 경우 이상 확인.
- [ ] `history('a')`는 이 설정에서도 `not_started`를 돌려준다(지연 검사 유지).
- [ ] 문서: `grep -n "too_broad_STELLA_CODEX_EXTRA_ROOTS_entry\|overlapping_allowed_roots_STELLA_CODEX_EXTRA_ROOTS" tools/codex-bridge/README.md tools/BRIDGE_README.md`가 두 파일에서 각각 두 식별자를 모두 찾는다. 두 README에 경고 문장(넓은 폴더 전체가 쓰기 가능해짐, 임시 폴더 피하기)이 있다 — 검증 담당이 눈으로 확인.

---

### 결정 3. `codex-last-message.txt`의 개인 경로 (inspection.md 중간)

**무엇을 막는가.** Codex를 실행할 때 마지막 메시지를 저장한 파일 `docs/tasks/<작업ID>/codex-last-message.txt`에 드라이브 문자로 시작하는 개인 절대 경로가 들어간다. 이 파일이 공개 저장소에 커밋되는 것을 막는다.

**선택: `.gitignore`에 `docs/tasks/*/codex-last-message.txt` 한 줄을 추가한다.** 파일 내용은 고치지 않고 지우지도 않는다. 로컬 사용자 파일을 보존하기 위해서다. 이 파일은 실행 방식에서 생기는 부산물이고, 공식 산출물은 `implementation.md`다(AGENTS.md 규칙). 그러므로 공개 저장소에 둘 이유가 없다. 경로를 지우거나 자리표시자로 바꾸는 방법은 실행할 때마다 다시 생기므로 쓰지 않는다.

**완료 기준**

- [ ] `git check-ignore -v docs/tasks/issue-2-allowed-roots/codex-last-message.txt` 종료 코드 0, 일치 규칙이 `.gitignore`의 새 줄.
- [ ] `git status --porcelain --untracked-files=all docs/tasks | grep codex-last-message` 출력 없음.
- [ ] `git check-ignore docs/tasks/issue-2-allowed-roots/design-2.md docs/tasks/issue-2-allowed-roots/implementation.md` 종료 코드 1(공식 산출물은 무시되지 않음).

---

### 낮음 항목 판단 (이번에 고치지 않음)

| 항목 (근거) | 판단 | 이유 |
|---|---|---|
| 검사와 실행 사이 바꿔치기 (security.md 낮음 1, inspection.md 낮음) | 고치지 않음. README 한계 문장에 추가 | 결정 1 이후에는 같은 허용 폴더 안에서 다른 작업ID로 동시에 도는 `workspace-write` 턴이 있을 때만 가능하다. 실행 직전에 다시 검사해도 시간 차는 남아서 완전히 막을 수 없다. 1판 7절의 "OS 경계 아님" 한계 안에 든다. |
| 오류로 경로 존재 여부 노출 (security.md 낮음 3, inspection.md 낮음) | 고치지 않음 | 변경 전부터 있던 동작이다. 호출자인 AI는 `read-only` Codex 턴으로도 경로가 있는지 알 수 있다("OS 읽기 경계 아님", 1판 7절). 그러므로 순서를 바꿔도 막히는 정보가 없다. 경로 원문은 새지 않는다. |
| Claude 자식 환경으로 `STELLA_CODEX_EXTRA_ROOTS` 전달 (security.md 낮음 2) | 이 작업 범위 밖 | 고치려면 `tools/claude-bridge/`를 바꿔야 하는데, 1판에서 범위 밖으로 정했다. 별도 과제로 할지는 열린 질문 1. |
| `AGENTS.md`의 기존 개인 경로 (inspection.md 낮음) | 이 작업 범위 밖 | HEAD에 이미 있는 내용이고 이번 변경으로 늘지 않았다. 열린 질문 2. |
| `\\?\` 접두어가 붙은 허용 폴더 안 경로 거부 (inspection.md 정보) | 지금처럼 거부 유지 | 막는 쪽으로 실패하므로 보안 문제가 아니다. 지원하려면 정규화를 더해야 해서 요청 범위 밖이다. |

## 완료 기준 (검증·검사 담당이 그대로 확인함)

모든 시험은 `FakeClient` 또는 `bounded_process` 대체와 임시 폴더만 쓴다. 실제 모델·네트워크 호출은 없다.

- [ ] 결정 1·2·3의 완료 기준 전부.
- [ ] 기존 시험 전부 통과: `python -B -X utf8 tools/codex-bridge/test_server.py -v`, `python -B -X utf8 tools/test_bridge_dispatcher.py -v`, `python -B -X utf8 tools/claude-bridge/test_bridge.py -v` 모두 실패·오류 0. 링크 관련 건너뜀만 허용하고, 건너뛴 이유가 출력에 있어야 한다.
- [ ] 1판 완료 기준 가운데 이 판에서 바꾸지 않은 것 — 이전 판 유지(그대로 다시 확인).
- [ ] 개인 정보 없음: `git diff -- tools .gitignore` 결과와 새 미추적 파일(`tools/`, `docs/tasks/issue-2-allowed-roots/`에서 `codex-last-message.txt` 제외)에 대해 `grep -nIiE "users[\\/]|[a-z]:[\\/][^<]|/home/"`를 돌린다. 출력에는 시험의 합성 경로만 남아야 한다(검사 담당이 눈으로 확인).

## 지켜야 할 제약

- 구독 로그인만 사용하고 API 키 대체 경로는 금지한다. `CodexClient.checked_status()`, `forced_login_method`는 건드리지 않는다(이전 판 유지).
- 자식 환경은 `SAFE_ENV` 허용 목록 그대로 둔다. `protected_paths()`는 브리지 프로세스 안에서 환경을 읽기만 하고, 자식에게 새 변수를 넘기지 않는다.
- 공개 오류는 영문 식별자만 쓴다. 새 식별자 두 개에 경로나 설정값을 넣지 않는다.
- 공개 저장소다. 문서·예시·시험에 개인 경로, 사용자 이름, 계정, 세션 ID를 쓰지 않는다. 시험의 가짜 홈은 임시 폴더로 만든다.
- 실제 기억 DB·개인 자료를 읽지 않는다. 시험이 실제 홈·`.codex`·`.claude`를 읽거나 쓰지 않도록 환경변수를 패치한다.
- 고칠 수 있는 파일은 범위에 적은 것뿐이다(`tools/codex-bridge/` 네 파일, `tools/BRIDGE_README.md`, `.gitignore`). `codex-last-message.txt`는 지우거나 고치지 않는다.
- 자동 재시도·백그라운드 작업·다른 AI 호출을 넣지 않는다.

## 열린 질문

1. Claude 자식 프로세스에서 `STELLA_CODEX_*` 변수를 빼는 일(security.md 낮음 2)을 별도 GitHub 이슈로 만들지 정해 주세요. 이 작업에서는 고치지 않습니다.
2. `AGENTS.md` 5~6행의 기존 로컬 절대 경로를 자리표시자로 바꾸는 정리를 별도로 할지 정해 주세요. 이 작업에서는 고치지 않습니다.
