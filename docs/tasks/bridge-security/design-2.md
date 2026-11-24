# 설계 2판: 1단계 보정
- 작업ID: `bridge-security` / 작성일: 2026-11-24 / 근거: `code-design-1.md`의 이견과 대조 결과
- 아래에 적지 않은 것은 `design.md`를 유지한다. 1단계 제작은 `design.md`의 A·H, `code-design-1.md`, 이 문서를 함께 따르며, 서로 다르면 이 문서가 우선한다.

1. **지우는 것의 단계**: `design.md`의 지우는 표는 전체 목록이다. 1단계에서는 A·H에 해당하는 것만 지운다. `stop_tree`의 `taskkill`, `MAX_LINE`, Claude의 직접 `subprocess.run` 2곳은 2·3단계, `NO_API_ENV`와 Codex 전용 안내문 조립은 3단계에서 바꾼다.
2. **폴더 오류**: 절대 경로이지만 없거나, 폴더가 아니거나, 읽을 수 없는 루트는 `project_root_invalid`. 값 없음·빈 값·상대 경로는 `project_root_required`.
3. **`.bridge/.gitignore` 비교**: 새로 만들 때는 `*`와 LF 한 줄. 이미 있으면 UTF-8(BOM 허용)로 읽어 줄바꿈 종류와 앞뒤 공백·빈 줄을 뺀 내용이 `*` 하나뿐일 때 같은 것으로 본다. 그 밖(주석, 다른 줄, 읽기 실패)은 `bridge_gitignore_must_ignore_all`. 기존 파일은 바꾸지 않는다. 다른 쪽이 막 만들어 아직 빈 파일이면 실패해도 되지만 파일을 덮지 않는다.
4. **홈 판단의 `~`**: 펼친 결과가 비었거나, `~`로 **시작**하거나(펼치지 못함), 상대 경로이거나, 해석에 실패할 때만 `cannot_determine_home`. 이름 중간의 `~`(예: 8.3 이름, 사용자 이름)는 허용한다.
5. **이어가기 조건**: 기존 루트·읽기 전용 완료 대화는 루트 검사와 3번 검사를 통과할 때 그대로 이어진다. README에 적는다.

추가 완료 기준:
- [ ] `test_project_root_home_unavailable`: 펼치지 못한 `~`·빈 값·상대 홈은 거부, 이름 중간에 `~`가 있는 가짜 홈은 허용
- [ ] `test_project_root_invalid`: 없는 폴더·파일 루트는 `project_root_invalid`, 경로 미노출
- [ ] `test_bridge_state_git_ignored`에 CRLF·BOM·앞뒤 빈 줄 허용, 주석·추가 줄 거부 경우 추가
- [ ] 1단계 뒤에도 기존 시간 제한·출력·환경 시험이 그대로 통과(2·3단계 대상 코드를 미리 지우지 않음)
