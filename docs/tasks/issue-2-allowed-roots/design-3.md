# 설계: Codex 브리지 허용 폴더 목록 — 보안 수정 설계 (3판, 마지막 회차)

- 작업ID: `issue-2-allowed-roots`
- 관련 이슈: #2
- 작성일: 2026-11-23
- **보안 수정 설계.** 근거 파일: 같은 폴더의 `inspection-2.md`(검사, 수정 필요, 중간 1건), `security-2.md`(프로그램 보안, 통과, 낮음 3건), `security-codex.md`(Codex 참고 의견, 우려). 기준 설계: `design.md`(1판), `design-2.md`(2판). 제작 보고: `implementation-2.md`.
- **회차 예외.** 이 작업의 보안 재수정은 원래 1회(2판)까지다. 사용자가 2026-11-23에 **이번 한 번만** 추가 회차를 허락했다. 이 3판이 **마지막 회차**다. 이 판 이후에도 남는 문제는 더 고치지 않고 열린 질문과 GitHub 이슈 후보로 사용자에게 넘긴다.
- 이 판은 사용자가 정한 세 결정(특수 경로 표기 거부, 실제 경로만 허용, 홈 판단 실패 시 전체 거부와 보호 목록 보강)을 구체화한다. 다른 방식으로 바꾸지 않는다.

## 목표

`STELLA_CODEX_EXTRA_ROOTS`가 설정된 경우에 다음 세 가지를 막는다. (1) `\\?\`·`//?/`·UNC 표기로 겹침·넓이 검사를 피하는 것(inspection-2 중간). (2) 경로 중간의 링크·정션·8.3 별칭을 거쳐 겹침 검사를 피하는 것(security-2 낮음 1, security-codex 중간 1). (3) 홈을 정할 수 없을 때 보호가 조용히 빠지는 것과, Windows PowerShell 프로필·`CLAUDE_CONFIG_DIR`이 보호 목록에 없는 것(security-2 낮음 2·3, security-codex 중간 2). 추가 폴더가 없을 때의 동작은 1·2판과 같이 변경 전과 완전히 같다.

## 범위

바꿀 파일과 함수 (모두 `tools/codex-bridge/server.py` 안, 실제 이름):

- `_extra_root_entries(value)`: 항목별 검사에 "특수 표기 거부"(결정 1)와 "실제 경로만 허용"(결정 2)을 더한다.
- `protected_paths()`: 홈 판단 실패 시 예외를 내고(결정 3), 보호 목록 B 묶음에 PowerShell 프로필 폴더(홈 기준과 Windows가 알려 주는 실제 문서 폴더 기준)와 `CLAUDE_CONFIG_DIR`을 더한다.
- 새 내부 함수 하나(이름은 Codex 재량, 예: `_read_documents_registry_value()`): Windows 레지스트리에서 문서 폴더 원래 값을 읽어 문자열 또는 `None`을 돌려준다(결정 3). 시험은 이 함수를 `patch.object`로 대체한다.
- `check_root_layout(entries)`: 입력 표기 비교 조건을 지우고 실제 경로끼리의 상하위 비교만 남긴다(결정 2의 정리).
- `Bridge.allowed_roots()`: 추가 폴더가 있을 때 주 루트에도 특수 표기 거부와 실제 경로 규칙을 적용한다(결정 1·2의 주 루트 판단).
- 새 내부 보조 함수 하나(이름은 Codex 재량, 예: `_special_path(path)`): 결정 1의 표기 판정.
- `tools/codex-bridge/test_server.py`: 아래 완료 기준의 새 시험 추가, 기존 시험 한 개의 기대 식별자 변경, 임시 폴더 `.resolve()` 보정(아래 "기존 시험 조정").
- `tools/codex-bridge/README.md`, `tools/BRIDGE_README.md`: 새 규칙과 식별자, 바뀐 한계 문장.

바꾸지 않을 것:

- `Bridge.__init__`의 `self.root`·`self.root_input` 계산, `Bridge.ask()`의 `cwd` 검사 순서와 지문 규칙, `task_dir()`, 상태 파일 형식, `handle()`, `main()` — 이전 판 유지.
- `parse_extra_roots(value)`의 공개 이름과 반환 형식(`resolve()`된 `Path` 목록, 중복 제거) — 이전 판 유지.
- `CodexClient`, `bounded_process`, `SAFE_ENV`, `codex_launcher`, `TOOLS` 스키마, `tools/bridge.py`, `tools/claude-bridge/`, `mcp.example.json`, `.gitignore`.
- 추가 폴더가 없을 때의 주 루트 처리(특수 표기·링크·8.3 경로의 주 루트도 지금처럼 동작). 주 루트의 넓이 규칙 미적용 — 이전 판 유지.
- 2판 결정 3(`codex-last-message.txt` 제외)과 그 완료 기준 — 이전 판 유지.
- `cwd`의 `\\?\` 표기 처리. 지금처럼 허용 폴더와 표기가 다르면 거부된다(막는 쪽 실패). 결정 1 뒤에는 허용 폴더가 모두 일반 표기이므로 특수 표기 `cwd`는 항상 거부된다.

## 동작 설계

### 검사 순서 (`_extra_root_entries`의 항목마다, 위에서부터)

| 단계 | 검사 | 실패 시 식별자 |
|---|---|---|
| 1 | 빈 항목 건너뜀, 절대 경로임 | `invalid_STELLA_CODEX_EXTRA_ROOTS_entry` (이전 판 유지) |
| 2 | **적은 경로가 특수 표기가 아님** (결정 1) | `invalid_STELLA_CODEX_EXTRA_ROOTS_entry` |
| 3 | `resolve(strict=True)` 성공, 폴더임, 파일시스템 최상위 아님, **resolve 결과도 특수 표기 아님** | `invalid_STELLA_CODEX_EXTRA_ROOTS_entry` |
| 4 | **적은 경로(정규화한 절대 경로)와 resolve 결과가 같음** (결정 2) | `extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS` (새) |
| 5 | 보호 목록 계산(첫 유효 항목에서 1회). **홈을 정할 수 없음** (결정 3. 홈에만 적용) | `cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS` (새) |
| 6 | 보호 경로와 겹치지 않음 (2판 규칙 + 결정 3의 보강 목록) | `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry` (이전 판 유지) |

그다음 `Bridge.allowed_roots()`에서 (추가 항목이 하나 이상일 때만):

| 단계 | 검사 | 실패 시 식별자 |
|---|---|---|
| 7 | **주 루트가 특수 표기가 아니고 `self.root_input == self.root`** (결정 1·2의 주 루트 판단) | `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set` (새) |
| 8 | `check_root_layout` — 실제 경로끼리 상하위 관계 없음 | `overlapping_allowed_roots_STELLA_CODEX_EXTRA_ROOTS` (이전 판 유지) |

공통 규칙 (이전 판 유지):

- 어느 단계든 실패하면 설정 전체를 거부한다. 거부는 `folder.mkdir`·잠금·`client.ask`보다 먼저 일어난다. `history`·`status`는 영향을 받지 않는다(지연 검사).
- 새 식별자 세 개는 모두 `[A-Za-z0-9_]+` 형식이라 `handle()`이 그대로 내보낸다. 경로나 환경변수 값은 넣지 않는다. 예외는 `from None`으로 연결을 끊는다.
- 추가 항목이 없으면(`None`·빈 문자열·공백·빈 목록) 1~8단계 모두 돌지 않고 `[self.root]`를 돌려준다. `protected_paths()`도 부르지 않는다.

---

### 결정 1. 특수 경로 표기 거부 (inspection-2 중간 1건)

**무엇을 막는가.** 항목을 `\\?\<경로>`, `//?/<경로>`, `\\localhost\<드라이브>$\<경로>` 같은 표기로 쓰면 `Path.resolve()`가 접두어를 그대로 남긴다. 그래서 일반 표기인 보호 경로·주 루트와의 `is_relative_to` 비교가 항상 거짓이 되어, 가짜 홈·`.ssh`·`APPDATA`·주 루트의 부모가 허용됐다. 같은 표기의 `cwd`로 `.ssh` 안 `workspace-write` 요청이 클라이언트까지 갔다.

**동작.**

- 판정 규칙(보조 함수): 경로 문자열에서 `/`를 `\`로 바꾼 뒤 `\\`로 시작하면 특수 표기다. 이 규칙 하나로 `\\?\`, `\\?\UNC\`, `\\.\`, `//?/`, `//./`, UNC(`\\server\share`, `\\localhost\C$`)를 모두 잡는다. OS 분기는 두지 않는다. POSIX에서 `\`로 시작하는 경로는 절대 경로가 아니라 1단계에서 이미 걸리고, `//`로 시작하는 POSIX 경로는 거부되어도 막는 쪽 실패라 문제가 없다.
- 2단계: `str(Path(entry))`에 판정을 적용한다(적은 표기 기준).
- 3단계: `resolve()` 결과에도 같은 판정을 적용한다. 일반 표기로 적었지만 실제 위치가 네트워크 공유인 경우(연결된 네트워크 드라이브, UNC를 가리키는 정션 등)도 여기서 거부된다.
- 식별자: 둘 다 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`(사용자 결정).
- inspection-2에서 `too_broad_...`로 거부되던 `\\.\` + 가짜 홈은 이제 2단계에서 `invalid_..._entry`로 거부된다. 더 일찍 막히는 것이고, 막힌다는 결과는 같다.

**주 루트 판단 (기존 동작 보존 우선).**

- 추가 폴더가 **없으면**: 주 루트에는 아무 검사도 더하지 않는다. 지금 `\\?\` 표기 주 루트로 쓰던 설정도 그대로 동작한다. 이 경우 비교할 다른 허용 폴더가 없어서 우회할 대상이 없다.
- 추가 폴더가 **있으면**: 주 루트의 `self.root_input` 또는 `self.root`가 특수 표기면 7단계에서 `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`으로 거부한다. 주 루트가 `\\?\` 표기면 추가 폴더와의 겹침 비교가 같은 방식으로 무력해지기 때문이다. 추가 폴더 기능은 이 작업에서 새로 생긴 것이라, 이 거부는 기존 사용자 설정을 깨지 않는다.
- 주 루트용 식별자를 따로 둔 이유: `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`는 "추가 항목이 잘못됨"으로 읽혀서, 운영자가 고칠 곳(`STELLA_CODEX_ROOT` 또는 `--root`)을 잘못 찾게 된다.

**남는 위험(문서화만 함).** 네트워크 공유나 UNC 위치의 프로젝트는 추가 폴더로 쓸 수 없다. 의도한 제한이며 README에 적는다.

**완료 기준**

- [ ] `test_special_path_notation_extra_roots_refused` (StoreTests, Windows 전용 `skipUnless(os.name == 'nt')`): 2판 `test_too_broad_extra_roots_refused`와 같은 방식으로 가짜 홈(`.ssh` 포함)과 `APPDATA`·`LOCALAPPDATA` 환경을 만든다. 모든 임시 폴더는 `.resolve()`한 값이다. inspection-2의 우회 6건을 각각 확인한다.
  1. 추가 = `\\?\` + 가짜 홈
  2. 추가 = `\\?\` + 가짜 홈 `.ssh`
  3. 추가 = `//?/` + 가짜 홈(`/` 구분자로 적음)
  4. 추가 = `\\localhost\<드라이브 문자>$\<가짜 홈의 드라이브 뒤 경로>` (UNC 관리 공유)
  5. 추가 = `\\?\` + 주 루트의 부모
  6. 추가 = `\\?\` + 가짜 홈, `ask(..., cwd='\\?\' + 가짜 홈 .ssh, sandbox='workspace-write')`

  여섯 건 모두 `ask`가 정확히 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`로 실패하고, `client.calls == []`, 주 루트 `.bridge` 없음, `history('a')['status'] == 'not_started'`, `handle()` 응답 텍스트가 식별자와 정확히 같고 넣은 경로 문자열을 포함하지 않아야 한다(2판의 `assert_roots_refused` 재사용 가능. 6번은 `cwd`를 준 `ask`로 따로 확인). 4번은 관리 공유에 접근할 수 없는 환경에서도 같은 식별자가 나와야 한다(경로 없음 → 3단계). 1~3번은 같은 값을 `STELLA_CODEX_EXTRA_ROOTS` 환경변수 문자열로 넣은 경우도 한 번 이상 확인한다.
- [ ] 같은 시험에서 추가 = `\\.\` + 가짜 홈도 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`.
- [ ] `test_special_notation_primary_root` (StoreTests, Windows 전용): (가) 주 루트 = `'\\\\?\\' + str(self.root)`, 추가 = 별도 실제 임시 폴더 → `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`, `client.calls == []`. (나) 같은 주 루트, 추가 폴더 없음 → `ask('a', 'x')`가 성공하고 `FakeClient`가 한 번 불린다(기존 동작 보존).
- [ ] 모델 호출 없음: 위 시험은 `FakeClient`만 쓴다.

---

### 결정 2. 실제 경로만 허용 (security-2 낮음 1, security-codex 중간 1, inspection-2 미확인 항목 `subst`)

**무엇을 막는가.** 입력 경로가 허용 폴더 밖의 링크를 지나 다른 허용 폴더 안으로 들어간 뒤 그 안의 링크를 한 번 더 거치는 경우(예: `x/linkA/inner`, `x/linkA` → 허용 폴더 `A`, `A/inner` → `T`). 2판의 입력 표기 비교는 글자 그대로만 비교해서 이를 놓쳤다. `A`에서 `workspace-write`로 도는 턴이 `A/inner`를 바꾸면 다음 호출부터 그 항목(주 루트라면 `.bridge/`까지)의 위치가 옮겨진다. Codex가 지적한 `alias`, `alias/j` 조합도 같은 구멍이다.

**동작 (4단계).**

- 비교 대상: `Path(os.path.abspath(path))`(2판에서 이미 만드는 "입력 절대 표기")와 `resolved`.
- 둘이 다르면 `extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS`로 거부한다. 식별자 이름이 "실제 경로로 적으라"는 뜻을 드러낸다.
- 비교는 `Path` 객체끼리 `!=`로 한다. Windows의 `WindowsPath` 비교는 대소문자를 무시하므로 사용자 결정의 "대소문자 무시 비교"와 같다. POSIX는 대소문자를 구분하지만, POSIX 파일시스템에서는 대소문자가 다르면 보통 다른 경로라 그대로 둔다.
- `os.path.abspath`가 하는 정규화(구분자 통일, `.`·`..` 접기, 끝 구분자 제거, Windows의 끝 마침표·공백 제거)는 통과한다. 따라서 `C:/proj`, `c:\PROJ\`처럼 표기만 다른 실제 경로는 허용된다.
- 이 규칙에 걸리는 것: 경로 중간이나 끝의 심볼릭 링크·정션, 8.3 짧은 이름(`resolve`가 긴 이름으로 풀기 때문), 링크 뒤의 `..`(글자로 접은 결과와 실제 결과가 다름), `subst` 가상 드라이브(실제 드라이브 경로로 풀림). 연결된 네트워크 드라이브는 결정 1의 3단계에서 먼저 걸린다.

**2판 판단을 대신함.** 2판 결정 1은 "링크·정션인 항목을 일괄 거부하지 않는다"고 정했다(macOS `/var` 같은 시스템 링크, 8.3 이름 때문에 정상 설정을 잘못 거부할 수 있다는 이유). 이 판에서는 사용자 결정 "실제 경로만 허용"이 그 판단을 대신한다. 그 결과 macOS에서는 `/var/...` 대신 `/private/var/...`처럼 실제 경로로 적어야 하고, Windows에서는 8.3 이름 대신 긴 이름으로 적어야 한다. README에 적는다.

**주 루트 판단.**

- 추가 폴더가 **없으면**: 주 루트가 링크·8.3 경로여도 지금처럼 동작한다(기존 동작 보존).
- 추가 폴더가 **있으면**: 7단계에서 `self.root_input != self.root`이면 `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`으로 거부한다. security-2가 재현한 "주 루트 = `x/linkA/pj`, 추가 = `[A]`" 경우(`.bridge/` 이동)를 막는 데 필요하다. `self.root_input`은 2판에서 이미 저장하므로 새 속성은 없다.

**`check_root_layout` 정리.**

- 4단계와 7단계가 생기면, 겹침 검사에 들어가는 모든 항목(주 루트 포함)에서 입력 절대 표기와 실제 경로가 같다. 그러면 2판의 조건 (나) "입력 표기가 다른 허용 폴더의 실제 경로 안을 지남"은 조건 (가) "실제 경로끼리 상하위"와 같은 결과를 낸다. 그래서 필요 없다.
- 가장 작은 변경: `check_root_layout`에서 조건 (나) 한 줄만 지운다. 인자 형식(`(입력 절대 표기, resolved)` 쌍 목록)과 호출 위치는 그대로 둔다. 실제 경로끼리 상하위 비교(`resolved != other and resolved.is_relative_to(other)`)와 중복 제거 전 전체 목록으로 검사하는 방식은 유지한다(이전 판 유지).
- 7단계는 `check_root_layout`보다 먼저 돈다. 그래서 주 루트가 링크인 설정은 겹침이 아니라 주 루트 식별자로 거부된다.

**완료 기준**

- [ ] `test_extra_root_must_be_real_path` (StoreTests): 임시 폴더 `outer = Path(folder).resolve()` 아래에 실제 폴더 `primary`, `A`, `T`, `P`를 만든다. 링크는 2판 시험처럼 `os.symlink`를 먼저 시도하고, 실패하면 Windows에서 `_winapi.CreateJunction`, 둘 다 안 되면 `self.skipTest`(이유 문자열 포함). 정리할 때는 링크를 먼저 지운다. 다음 경우가 모두 정확히 `extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS`로 실패하고 `client.calls == []`, 주 루트 `.bridge` 없음이어야 한다.
  1. `outer/'alias'` → `A` 링크. 추가 = `[outer/'alias']` (링크 항목 자체)
  2. `A/'inner'` → `T` 링크. 추가 = `[A, outer/'alias'/'inner']` (security-2 낮음 1의 2단 링크)
  3. 추가 = `[outer/'alias', outer/'alias'/'inner']` (security-codex 중간 1의 별칭 연쇄)
  4. 추가 = `[A/'inner']` (허용 폴더 밖 판단과 관계없이 링크 항목 거부. inspection-2의 "a/j만 → 허용" 결과가 이제 거부로 바뀜)
  5. Windows 전용: 긴 이름 폴더 `outer/'long folder name for short'`를 만들고 `ctypes.windll.kernel32.GetShortPathNameW`로 짧은 이름을 얻는다. 짧은 이름이 긴 이름과 같으면(볼륨에서 8.3 비활성) 이 경우만 `subTest` 안에서 건너뛴다. 추가 = `[짧은 이름 경로]`.
- [ ] 같은 시험의 주 루트 경우: `A/'pj'` → `P` 링크. (가) 주 루트 = `outer/'alias'/'pj'`, 추가 = `[A]` → `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`. (나) 주 루트 = `outer/'alias'`(→ `A`), 추가 = 별도 실제 폴더 → 같은 식별자. 둘 다 `client.calls == []`이고, 링크가 가리키는 실제 폴더 아래에 `.bridge`가 생기지 않는다.
- [ ] 같은 시험의 대조군 (거부되지 않음): (가) 추가 = `[A, T]`(실제 경로. `A/inner`가 `T`를 가리켜도 허용 — inspection-2 "정보" 항목과 같음) → `allowed_roots()`에 `A`, `T` 포함. (나) Windows 전용: 추가 = `[str(A).swapcase()]`, `[str(A) + os.sep]`, `[str(A).replace('\\', '/')]` 각각 허용. (다) 추가 폴더 없이 주 루트 = `outer/'alias'` → `ask('a', 'x')`가 성공하고 상태 파일이 `A/.bridge/codex/tasks/a/state.json`에 생긴다(기존 동작 보존).
- [ ] 2판 `test_extra_root_via_link_inside_other_root_refused`의 기대 식별자 변경(아래 "기존 시험 조정")이 통과한다.
- [ ] `check_root_layout`의 조건 (나) 제거 확인: `grep -n "input_path" tools/codex-bridge/server.py`의 결과에 `check_root_layout` 본문의 `is_relative_to` 비교가 없다(검증 담당이 눈으로 확인). 실제 경로 겹침은 2판 `test_overlapping_roots_refused`가 그대로 통과하는 것으로 확인한다.

---

### 결정 3. 홈 판단 실패 시 전체 거부 + 보호 목록 보강 (security-2 낮음 2·3, security-codex 중간 2)

**무엇을 막는가.** (a) 홈 관련 환경변수가 없으면 `protected_paths()`가 홈과 `.codex`·`.claude`·`.ssh`·`.config` 보호를 조용히 빼서, 넓은 폴더를 추가해도 거부되지 않는다. (b) 상대 경로 값은 브리지 실행 폴더 기준으로 풀려 엉뚱한 곳을 보호한다. (c) `~`가 든 값은 통째로 버려진다. (d) Windows PowerShell 프로필 폴더와 `CLAUDE_CONFIG_DIR`이 보호 목록에 없어, `<홈>\Documents` 같은 흔한 설정에서 sandbox 밖에서 실행되는 프로필에 쓸 수 있다. 문서 폴더가 OneDrive 등으로 옮겨진 PC(실제로 있음)에서는 프로필이 홈 기준 경로가 아닌 곳에 있다.

**사용자 결정 반영 (2026-11-23).** (1) 실제 문서 폴더 위치도 이번에 같이 보호한다. (2) 엄격 거부는 홈에만 적용한다. 선택 환경변수는 지금처럼 비정상이면 그 항목만 생략한다.

**동작 (`protected_paths()`).**

- 홈 값: 지금처럼 `os.path.expanduser('~')`로 얻는다(Windows는 `USERPROFILE`, 없으면 `HOMEDRIVE`+`HOMEPATH`. POSIX는 `HOME`, 없으면 계정 정보).
- 다음 중 하나면 `ValueError('cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS')`를 `from None`으로 낸다. 보호를 조용히 빼지 않는다.
  - 값이 비었음
  - 값에 `~`가 들어 있음(풀리지 않은 `~`. 사용자 결정 그대로 "포함"이면 거부)
  - `os.path.isabs(값)`이 거짓(상대 경로)
  - 결정 1의 특수 표기
- 선택 환경변수 `LOCALAPPDATA`, `APPDATA`, `CODEX_HOME`, `CLAUDE_CONFIG_DIR`: **엄격 거부하지 않는다**(사용자 결정 "홈만 검사"). 없거나 비었거나 비정상(`~` 포함, 상대 경로, 특수 표기)이면 그 보호 항목만 생략하고 나머지 검사는 계속한다. 상대 경로 값을 실행 폴더 기준으로 풀던 동작은 "생략"으로 바뀐다(엉뚱한 곳을 보호하지 않게). `cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS`는 홈에만 쓴다.
- 이 함수는 추가 항목이 있을 때만 불린다(5단계, 첫 유효 항목에서 1회). 추가 폴더가 없으면 홈이 없어도 지금처럼 동작한다.
- `handle()`은 이 식별자를 그대로 내보낸다. 환경변수 이름 외의 값은 넣지 않는다.

**보호 목록 보강 (B 묶음 "안쪽까지 거부"에 추가).**

| 추가 경로 | 조건 |
|---|---|
| `<홈>/Documents/WindowsPowerShell` | 항상 (홈이 정해졌으므로) |
| `<홈>/Documents/PowerShell` | 항상 |
| `CLAUDE_CONFIG_DIR` | 설정돼 있고 정상일 때 |
| `<실제 문서 폴더>/WindowsPowerShell` | Windows이고 레지스트리 값을 정상으로 읽었을 때 |
| `<실제 문서 폴더>/PowerShell` | 같음 |

- 홈 기준 `<홈>/Documents/...` 두 항목은 항상 넣는다. 가짜 홈 환경변수로 시험할 수 있다. 디스크의 폴더 이름은 언어와 관계없이 `Documents`다.
- **실제 문서 폴더 위치(사용자 결정 1).** 표준 라이브러리 `winreg`로 `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders`의 `Personal` 값을 읽는다. 문서 폴더가 OneDrive 등으로 옮겨져 있으면 이 값이 옮겨진 위치다.
  - 읽기 함수(`_read_documents_registry_value()` 등)는 원래 문자열만 돌려준다. `os.name != 'nt'`이면 `winreg`를 가져오지 않고 바로 `None`을 돌려준다(Windows가 아니면 건너뜀). `OSError`(키·값 없음 등)도 `None`. 값 형식이 문자열이 아니면 `None`.
  - `protected_paths()`가 그 값을 `os.path.expandvars`로 풀고(`%USERPROFILE%` 등), 결과가 비었거나, `%`가 남았거나, `~`를 포함하거나, 절대 경로가 아니거나, 결정 1의 특수 표기면 **이 두 항목만 생략**한다. 이 항목 때문에 설정 전체를 거부하지 않는다. 홈 기준 보호는 그대로 남는다.
  - 정상이면 `<값>/WindowsPowerShell`, `<값>/PowerShell`을 `resolve()`(strict=False)해 B 묶음에 넣는다. 홈 기준 항목과 같아도 그대로 둔다(비교만 하므로 중복은 해가 없다).
  - `expandvars`는 브리지 프로세스의 환경을 쓴다. 시험에서 `USERPROFILE`을 가짜 홈으로 바꾸면 `%USERPROFILE%\Documents` 값도 가짜 홈 아래로 풀린다.
- 홈 기준 항목은 OS 분기를 두지 않는다. POSIX에서는 이 경로가 보통 없지만, `resolve()`(strict=False)로 정규화되고 비교만 하므로 해가 없다. POSIX `pwsh`의 프로필 위치(`~/.config/powershell`)는 이미 `.config` 보호에 들어 있다.
- B 묶음 규칙(`p.is_relative_to(r) or r.is_relative_to(p)`)이 그대로 적용된다. 그래서 문서 폴더 자체(프로필 폴더의 상위)와 프로필 폴더 안이 모두 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`로 거부된다. `<문서 폴더>/<프로젝트>`는 허용된다. 문서 폴더 전체를 추가하던 설정은 이제 거부되며, 필요한 프로젝트 폴더만 적으면 된다(README에 적음).
- A 묶음(홈, `LOCALAPPDATA`)과 나머지 B 묶음 항목은 이전 판 유지.

**남는 위험(문서화만 함).**

- `APPDATA`·`LOCALAPPDATA`·`CODEX_HOME`·`CLAUDE_CONFIG_DIR` 값이 없거나 비정상이면 그 항목의 보호만 빠진다(사용자 결정 "홈만 검사"로 받아들인 한계).
- 레지스트리 `Personal` 값을 읽지 못하거나 비정상이면 옮겨진 문서 폴더의 프로필 보호가 빠지고 홈 기준 보호만 남는다.

**시험 격리.** StoreTests `setUp`처럼 `USERPROFILE`·`HOME`을 가짜 홈으로, `APPDATA`·`LOCALAPPDATA`를 가짜 홈 아래로 바꾸고 `CODEX_HOME`·`CLAUDE_CONFIG_DIR`은 지운다. 레지스트리 읽기 함수는 `setUp`에서 `None`을 돌려주도록 대체해 실제 레지스트리 값에 기대지 않게 하고, 필요한 시험에서만 가짜 문서 폴더를 돌려주게 다시 대체한다. 가짜 홈은 `.resolve()`한 임시 폴더 아래에 만든다. 시스템 임시 폴더가 8.3 이름(`~` 포함)인 PC에서도 "`~` 포함 → 거부" 규칙에 걸리지 않게 하기 위해서다.

**완료 기준**

- [ ] `test_cannot_determine_home_refuses_extra_roots` (StoreTests): 유효한 추가 항목 하나(별도 실제 임시 폴더)를 둔 상태에서 다음 경우가 각각 정확히 `cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS`로 실패하고, `client.calls == []`, 주 루트 `.bridge` 없음, `handle()` 응답이 식별자와 정확히 같고 넣은 값을 포함하지 않아야 한다.
  1. Windows 전용: `USERPROFILE`, `HOME`, `HOMEDRIVE`, `HOMEPATH`를 모두 지움
  2. `USERPROFILE`과 `HOME`을 `'relative-home'`으로
  3. `USERPROFILE`과 `HOME`을 `str(<resolve된 임시 폴더> / 'AB~1')`로(`~` 포함)
- [ ] 같은 시험의 대조군 (기본값 불변): 1~3과 같은 환경에서 **추가 폴더가 없으면** `ask('a', 'x')`가 성공한다. 2판의 `check_root_layout` 미호출 단언처럼 `protected_paths`를 `side_effect=AssertionError`로 패치해 불리지 않음을 확인해도 된다.
- [ ] `test_invalid_optional_protection_env_skipped` (StoreTests, 홈만 엄격 검사 확인): 홈은 정상인 가짜 홈으로 두고, `APPDATA='relative-appdata'`, `LOCALAPPDATA=''`, `CODEX_HOME=str(<resolve된 임시 폴더> / 'CO~1')`, `CLAUDE_CONFIG_DIR='relative-claude'`를 각각(또는 함께) 넣는다. 별도 실제 폴더를 추가 항목으로 둔 `ask`가 **성공**한다(`cannot_determine_home_...`이 아님). 같은 환경에서 가짜 홈의 `.ssh/sub`를 추가 항목으로 넣으면 여전히 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`(나머지 보호는 유지).
- [ ] `test_powershell_profile_and_claude_config_protected` (StoreTests): 가짜 홈 아래 `Documents/WindowsPowerShell/sub`, `Documents/PowerShell/Modules`, `Documents/proj`를 만들고, 별도 임시 폴더 `claude-cfg/sub`를 만든다. 레지스트리 읽기 함수는 `None`(setUp 기본값). 다음은 각각 정확히 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`로 실패하고 `client.calls == []`이다.
  - `home/'Documents'`, `home/'Documents'/'WindowsPowerShell'`, `home/'Documents'/'WindowsPowerShell'/'sub'`, `home/'Documents'/'PowerShell'/'Modules'`
  - `CLAUDE_CONFIG_DIR=str(claude-cfg)`로 설정한 뒤 `claude-cfg`와 `claude-cfg/'sub'`
- [ ] 같은 시험의 대조군: `home/'Documents'/'proj'`는 `ask(..., cwd=그 폴더)`가 성공하고 `FakeClient`가 받은 `cwd`가 그 폴더다. `CLAUDE_CONFIG_DIR`을 지운 상태에서는 `claude-cfg`가 허용된다.
- [ ] `test_redirected_documents_profile_protected` (StoreTests): 가짜 홈과 별개인 임시 폴더 아래 가짜 옮겨진 문서 폴더 `redirected/Documents`와 그 안의 `WindowsPowerShell/sub`, `PowerShell`, `proj`를 만든다. `patch.object(module, '<레지스트리 읽기 함수>', return_value=...)`로 대체한다.
  - (가) 반환값 = 가짜 문서 폴더 절대 경로 → `redirected/Documents`, `.../WindowsPowerShell`, `.../WindowsPowerShell/sub`, `.../PowerShell`이 각각 `too_broad_STELLA_CODEX_EXTRA_ROOTS_entry`, `client.calls == []`. `.../proj`는 허용.
  - (나) 반환값 = `'%SYNTHETIC_DOCS%\\Documents'`, 환경변수 `SYNTHETIC_DOCS=str(redirected)` → `expandvars`로 풀려 (가)와 같이 `.../WindowsPowerShell`이 거부된다.
  - (다) 비정상 값 생략: 반환값 `None`, `'relative-docs'`, `'%SYNTHETIC_UNSET%\\Docs'`(풀리지 않음), `'\\\\?\\' + 가짜 문서 폴더`(특수 표기) 각각에서 `redirected/Documents/WindowsPowerShell`은 **허용**되고(그 항목만 생략), 같은 환경에서 `home/'Documents'/'WindowsPowerShell'`은 여전히 `too_broad_...`(홈 기준 보호 유지), 어느 경우도 `cannot_determine_home_...`이 나오지 않는다.- [ ] `test_documents_registry_reader_non_windows_and_errors` (StoreTests): (가) `patch.object(module.os, 'name', 'posix')` 상태에서 읽기 함수가 `winreg`를 가져오지 않고 `None`을 돌려준다(`patch.dict(sys.modules, {'winreg': None})`로 가져오기 시 실패하게 해도 `None`). (나) Windows 전용: `winreg.OpenKey`를 `side_effect=OSError`로 대체하면 `None`. 실제 레지스트리 값은 단언하지 않는다.
- [ ] 2판 `test_too_broad_extra_roots_refused`가 그대로 통과한다(기존 보호 목록 유지).

---

### 기존 시험 조정 (Codex가 고칠 것)

- `test_extra_root_via_link_inside_other_root_refused`: 결정 2 때문에 기대 식별자가 바뀐다. 경우 (1) 추가 `[a, a/'j']`는 `extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS`, 경우 (2) 주 루트 `a/'jp'` + 추가 `[a]`는 `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`. 만드는 폴더·링크와 정리 방식은 그대로 둔다.
- 8.3·macOS 이식성 보정: StoreTests에서 **추가 항목으로 넘기거나, 추가 항목과 함께 쓰는 주 루트**가 되는 임시 폴더와 `setUp`의 가짜 홈은 `.resolve()`한 값으로 만든다(예: `self.root = Path(self.tmp.name).resolve()`, `Path(self.enterContext(tempfile.TemporaryDirectory())).resolve() / 'synthetic-home'`, `Path(other).resolve()`). 시스템 임시 폴더가 8.3 이름이거나 링크(macOS `/var`)인 PC에서 결정 2·3 규칙 때문에 기존 시험이 깨지지 않게 하기 위해서다. 단언 내용은 바꾸지 않는다. 이 PC에서는 임시 폴더가 이미 실제 경로라 결과가 같다.
- 레지스트리 격리: StoreTests `setUp`에서 레지스트리 읽기 함수를 `None`을 돌려주도록 대체한다(`self.enterContext(patch.object(...))` 등). 기존 시험이 이 PC의 실제 문서 폴더 위치에 따라 결과가 달라지지 않게 하기 위해서다.
- 나머지 기존 시험은 고치지 않는다. `test_symlink_escape_from_extra_root_refused`(추가 폴더는 실제 경로, 링크는 `cwd` 쪽)는 지금처럼 `cwd_outside_allowed_roots`를 기대한다.

### README 변경 (두 파일 모두)

- 새 식별자 세 개와 뜻을 적는다: `extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS`(링크·정션·8.3·`subst` 없이 실제 경로로 적을 것), `primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set`(추가 폴더를 쓸 때 주 루트도 일반 표기의 실제 경로일 것), `cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS`(홈 값이 없거나 상대 경로거나 `~`를 포함하거나 특수 표기임. 다른 환경변수에는 적용하지 않음).
- `\\?\`·`//?/`·UNC·네트워크 드라이브 항목은 `invalid_STELLA_CODEX_EXTRA_ROOTS_entry`로 거부된다고 적는다.
- 보호 목록 문장에 `<홈>/Documents/WindowsPowerShell`, `<홈>/Documents/PowerShell`, Windows가 알려 주는 실제 문서 폴더(레지스트리 `Personal`, OneDrive로 옮겨진 경우 포함) 아래 같은 두 폴더, `CLAUDE_CONFIG_DIR`을 더하고, 문서 폴더 전체는 넣을 수 없다고 적는다.
- 한계 문장: `APPDATA`·`LOCALAPPDATA`·`CODEX_HOME`·`CLAUDE_CONFIG_DIR`이나 레지스트리 문서 폴더 값이 없거나 비정상이면 그 항목의 보호만 빠진다.
- 지울 문장: 2판의 "입력 절대 표기도 비교한다(Both resolved paths and input absolute paths are compared…)"와 "8.3 짧은 이름이 입력 표기 겹침 검사를 피할 수 있다"는 한계 문장. 이제 8.3·링크 입력은 거부되므로 맞지 않는다. TOCTOU 한계 문장과 "OS 경계 아님" 문장은 유지한다.

**완료 기준**

- [ ] `grep -c "extra_root_must_be_real_path_STELLA_CODEX_EXTRA_ROOTS\|primary_root_must_be_real_path_when_STELLA_CODEX_EXTRA_ROOTS_set\|cannot_determine_home_STELLA_CODEX_EXTRA_ROOTS" tools/codex-bridge/README.md tools/BRIDGE_README.md` — 두 파일 각각에서 세 식별자가 모두 나온다(검증 담당이 `grep -o`로 식별자별 확인).
- [ ] `grep -n "8.3 short names may evade\|8.3 short names can evade\|input absolute paths are compared" tools/codex-bridge/README.md tools/BRIDGE_README.md` 출력 없음.
- [ ] 두 README에 `WindowsPowerShell`, `CLAUDE_CONFIG_DIR`, `User Shell Folders` 또는 `OneDrive`가 나온다(`grep -n`). 한계 문장은 검증 담당이 눈으로 확인.

## 완료 기준 (검증·검사 담당이 그대로 확인함)

모든 시험은 `FakeClient` 또는 `bounded_process` 대체, 가짜 홈, 임시 폴더만 쓴다. 실제 모델·네트워크 호출은 없다. 정션은 `_winapi.CreateJunction`을 쓸 수 있다.

- [ ] 결정 1·2·3과 "기존 시험 조정", "README 변경"의 완료 기준 전부.
- [ ] inspection-2의 우회 6건이 모두 막힘 — `test_special_path_notation_extra_roots_refused`의 1~6번.
- [ ] security-2 낮음 1(2단 링크, 추가·주 루트 두 경우)과 security-codex 중간 1(별칭 연쇄)이 막힘 — `test_extra_root_must_be_real_path`의 2·3번과 주 루트 (가).
- [ ] security-2 낮음 2(홈 판단 실패)와 낮음 3(PowerShell 프로필, 옮겨진 문서 폴더 포함, `CLAUDE_CONFIG_DIR`)이 막힘 — `test_cannot_determine_home_refuses_extra_roots`, `test_powershell_profile_and_claude_config_protected`, `test_redirected_documents_profile_protected`.
- [ ] 홈만 엄격 검사, 선택 값과 레지스트리 값은 비정상이면 그 항목만 생략 — `test_invalid_optional_protection_env_skipped`, `test_redirected_documents_profile_protected` (다), `test_documents_registry_reader_non_windows_and_errors`.
- [ ] 정상인 실제 경로 설정은 계속 허용 — 2판 `test_non_overlapping_roots_and_duplicates_allowed`, `test_extra_roots_from_environment`, `test_extra_root_allowed_state_stays_in_primary_root`, `test_case_variant_cwd_same_fingerprint`, `test_workspace_write_in_extra_root_passes_only_cwd`와 이 판의 대조군이 통과.
- [ ] 기본값 불변 — `test_default_without_extra_roots_unchanged`, `test_special_notation_primary_root` (나), `test_extra_root_must_be_real_path` 대조군 (다), `test_cannot_determine_home_refuses_extra_roots` 대조군이 통과.
- [ ] 시험 명령 세 개 모두 실패·오류 0: `python -B -X utf8 tools/codex-bridge/test_server.py -v`, `python -B -X utf8 tools/test_bridge_dispatcher.py -v`, `python -B -X utf8 tools/claude-bridge/test_bridge.py -v`. 건너뜀은 링크 생성 불가, 8.3 비활성, 비 Windows 환경 이유만 허용하고 출력에 이유가 있어야 한다. `python` 명령이 없으면 로컬 실행 파일을 직접 지정하고 보고서에는 `<python-executable>`로 적는다.
- [ ] 2판 결정 3(`codex-last-message.txt` 제외)의 완료 기준 — 이전 판 유지(그대로 다시 확인).
- [ ] 개인 정보 없음: 2판과 같은 명령(`git diff -- tools .gitignore`와 미추적 `tools/`, `docs/tasks/issue-2-allowed-roots/`에서 `codex-last-message.txt` 제외)에 `grep -nIiE "users[\\/]|[a-z]:[\\/][^<]|/home/"`. 출력에는 시험의 합성 경로·검색 정규식·공개 URL만 남아야 한다(검사 담당이 눈으로 확인).

## 이번에 고치지 않는 항목 (GitHub 이슈 후보)

이 판에서 고치지 않는다. 2판 판단(받아들일 만한 위험 또는 범위 밖)을 유지한다. 이슈로 만들지는 사용자가 정한다(열린 질문 2).

- 이슈 후보 1 — 검사와 실행 사이 경로 바꿔치기(TOCTOU): `allowed_roots()` 검사 뒤 CLI 실행 전까지 같은 허용 폴더 안에서 경로 구성요소가 바뀔 수 있음(security-2, security-codex 낮음).
- 이슈 후보 2 — 같은 허용 폴더에서 다른 작업ID의 `workspace-write` 턴 동시 실행: 작업별 잠금만 있고 폴더별 쓰기 직렬화가 없음(security-codex 낮음).
- 이슈 후보 3 — 오류 식별자 차이로 경로 존재 여부 노출: `invalid_..._entry`와 다른 식별자로 경로가 있는지 알 수 있음, 경로 원문은 새지 않음(security-2 1차 낮음 판단 유지).
- 이슈 후보 4 — Claude 자식 프로세스 환경에서 `STELLA_CODEX_*` 변수 제거: `tools/claude-bridge/`가 `STELLA_CODEX_EXTRA_ROOTS`를 Claude 자식에게 넘김(security-2 낮음, design-2 열린 질문 1).

## 지켜야 할 제약

- 구독 로그인만 쓰고 API 키 대체 경로는 금지한다. `CodexClient.checked_status()`, `forced_login_method`는 건드리지 않는다(이전 판 유지).
- 자식 환경은 `SAFE_ENV` 허용 목록 그대로 둔다. `CLAUDE_CONFIG_DIR`은 브리지 프로세스 안에서 보호 목록 계산에만 읽고 자식에게 넘기지 않는다.
- 공개 오류는 영문 식별자만 쓴다. 새 식별자 세 개에 경로·환경변수 값을 넣지 않는다.
- 공개 저장소다. 문서·예시·시험에 개인 경로, 사용자 이름, 계정, 세션 ID를 쓰지 않는다. 시험의 가짜 홈·가짜 문서 폴더·8.3 형태 값은 임시 폴더에서 합성한다. 실제 홈·`.codex`·`.claude`·문서 폴더를 읽거나 쓰지 않도록 환경변수와 레지스트리 읽기 함수를 대체한다. 레지스트리는 읽기만 하고 쓰지 않는다. 실제 레지스트리 값을 보고서·시험 출력에 적지 않는다.
- 새 의존성은 넣지 않는다. 레지스트리 읽기는 표준 라이브러리 `winreg`만 쓰고, Windows에서만 함수 안에서 가져온다.
- 실제 기억 DB·개인 자료를 읽지 않는다. 시스템 상태를 바꾸는 시험(`subst`, 네트워크 드라이브 연결, 공유 생성)은 하지 않는다. UNC 관리 공유는 읽기 시도만 하고, 접근할 수 없어도 같은 식별자가 나오는지 확인한다.
- 고칠 수 있는 파일은 `tools/codex-bridge/server.py`, `tools/codex-bridge/test_server.py`, `tools/codex-bridge/README.md`, `tools/BRIDGE_README.md`뿐이다. `codex-last-message.txt`는 지우거나 고치지 않는다.
- 자동 재시도·백그라운드 작업·다른 AI 호출을 넣지 않는다.
- 이것이 마지막 보안 회차다. 제작 중 이 설계로 막히지 않는 새 문제를 찾으면 고치지 말고 `implementation-3.md`의 "확인하지 못한 것과 열린 질문"에 적는다.

## 열린 질문

1. (닫힘) OneDrive 등으로 옮겨진 문서 폴더의 PowerShell 프로필 보호 — 사용자 결정(2026-11-23, "이번에 같이 고침")으로 결정 3에 레지스트리 `Personal` 값 기준 보호를 넣었다.
2. "이번에 고치지 않는 항목"의 이슈 후보 4건을 GitHub 이슈로 게시할지 정해 주세요(게시는 사용자 허락 후에만 합니다).
3. design-2 열린 질문 2(`AGENTS.md`의 기존 로컬 절대 경로 정리)는 이 작업 범위 밖으로 그대로 남습니다.
