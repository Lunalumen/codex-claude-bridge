# AI 연결 기술 개발

사용자 지정일: 2026-11-15.
작업 위치: `E:/스텔라 ai/모음 파일/clude와 gpt에 연결 기술/`.

이 폴더는 AI 연결 기술을 개발하는 프로젝트다. 사용자 정정(2026-11-16)에 따라 기술 개발 담당은 ChatGPT/Codex이며 Claude는 개발 담당에서 제외한다. Claude를 지원하는 연결 기능 자체는 유지한다.

## 현재 확인한 위치
- Claude 작업 폴더: `E:/스텔라 ai/claude/`.
- 공통 Stella 스킬: `E:/스텔라 ai/claude/skills/`.
- 기존 Claude → 새 Codex 브리지: `E:/스텔라 ai/claude/프로젝트 파일/codex-mcp/server.py`.
- 기존 협의 절차: `E:/스텔라 ai/claude/collaboration/WORKFLOW.md`.
- Stella Memory 협의: `E:/스텔라 ai/claude/collaboration/tasks/stella-memory-001/`.
- 사용자 ChatGPT 영역: `E:/스텔라 ai/chatgpt/` (내용은 이번에 읽지 않음).

예전 `E:/claude/`는 현재 존재하지 않는다. 이전 보고서의 경로·전역 연결 설정이 자동으로 바뀌었다고 가정하지 않는다.

## 개발 목표와 현재 경계
1. 프로젝트 안내에서 현재 작업·최신 보고서·다음 담당을 바로 찾는다.
2. 작업별 요청·답변·결정·미해결 사항을 공유한다.
3. 기존 브리지의 실제 새 경로와 초기 연결을 확인한다.
4. 대화 ID 보관과 이어가기, Codex → Claude 역방향 연결은 각각 구현 가능 조건을 확인한 뒤 개발한다.

앞선 옛 경로에서 확인한 브리지는 Claude가 새 Codex 실행에 질문하는 단방향 구조였다. 현재 채팅을 깨우는 기능·작업별 대화 이어가기·역방향 호출은 검증되지 않았다. 폴더 이전 이후 모델 질문 왕복도 아직 재시험하지 않았다.

이번 작업은 폴더와 개발 기준을 마련한 것이다. 기존 코드 이동·전역 설정 변경·모델 호출은 하지 않았다.

## Codex → Claude 연결 프로그램 · 2026-11-15
터미널 진입점은 bridge.py, 사용법은 BRIDGE_GUIDE.md, 현재 확인 범위와 로그인 대기는 CONNECTION_STATUS.md를 따른다. bridge.py status / ask / history로 사용한다. 실제 모델 왕복 성공과 오프라인 합성 시험은 구분한다.


## 실제 연결 확인 · 2026-11-15
Codex → Claude 질문과 같은 작업 대화 이어가기의 실제 왕복을 확인했다. VALIDATION_TERMINAL_2026-11-15.md와 bridge.py history --task connection-check를 참조한다. 기존 데스크톱 대화를 제어하는 기능은 아니다.

