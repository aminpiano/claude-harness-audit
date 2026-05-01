---
name: harness-audit
description: 프로젝트 하네스를 read-only로 측정해 토큰 예산, 세션 복구성, 충돌, provider drift, 비대화, 운영 부담을 계기판처럼 판정한다. 결과는 유지/수정/제거/보류 권고까지이며 파일 자동 수정·생성은 금지. "하네스 점검", "harness audit", "세션 설정 검토", "CLAUDE.md 재배치", "클로드 설정 괴리 확인" 요청 시 사용.
---

# Harness Audit

## 역할

이 skill은 하네스 설계자가 아니라 **측정 장비**다.

프로젝트의 `CLAUDE.md`, `MEMORY.md`, hooks, skills, settings, 세션 로그,
프로젝트 내부 memory/docs가 실제로 AI 행동을 돕는지, 아니면 컨텍스트 비용과
충돌을 늘리는지 read-only로 측정하고 사용자에게 판정한다.

핵심 질문:

1. 시작 컨텍스트를 얼마나 쓰는가?
2. 새 세션이 얼마나 빨리 복구되는가?
3. 서로 다른 하네스 지시가 충돌하는가?
4. Claude Code/provider 변화와 어긋난 낡은 전제가 있는가?
5. 진행 파일이 dashboard가 아니라 changelog로 비대화했는가?
6. 유지 이유가 불명확한 규칙/skill/hook이 쌓였는가?

## 절대 제약

- 파일 수정, 생성, 삭제, 이동 금지.
- hook/script 실행 금지. 존재 여부, 권한, 내용 확인만 한다.
- 긴 원문을 그대로 출력하지 않는다.
- 새 하네스 구조를 강제하지 않는다.
- 프로젝트별 이상 구조를 설계하지 않는다.
- 사용자가 승인하기 전에는 수정 작업으로 넘어가지 않는다.

허용:

- `pwd`, `git rev-parse`, `find`, `rg`, `sed`, `wc`, `stat`, `jq` 같은 읽기 전용 명령.
- 공식 Claude Code 문서 확인. 단, provider drift 판단에 필요할 때만.
- 결과 보고서와 권고 작성.

## 실행 모드

사용자가 별도 지정하지 않으면 **Quick Audit**으로 실행한다.

- **Quick Audit**: 기본. 자동 주입/복구/충돌/비대화/운영 부담만 빠르게 측정한다.
- **Drift Audit**: 사용자가 "업데이트", "최신 Claude Code", "Opus 업데이트 이후"처럼 provider 변화 비교를 요구할 때. 공식 문서를 확인해 stale 전제를 찾는다.
- **Deep Audit**: 사용자가 깊게 보라고 명시할 때. 더 많은 파일을 보되 raw log/transcript는 읽지 않고 `wc`, `rg -m`, `sed` 제한으로 샘플링한다.

## 1단계 - 프로젝트 경계 확인

다음을 확인한다.

- 현재 디렉토리: `pwd`
- git root: `git rev-parse --show-toplevel` 실패 시 `pwd`
- 프로젝트 slug: 경로를 Claude project memory slug 규칙으로 추정
- 현재 날짜
- `CLAUDE.md`, `.claude/`, `memory/`, `context/`, `docs/`, `ai-docs/` 존재 여부

프로젝트가 git repo가 아니어도 실패로 보지 않는다.

## 2단계 - 인벤토리 수집

라인 수와 크기부터 본다. 긴 파일은 먼저 `wc -l -c`로 보고, 필요한 부분만 제한해서 읽는다.

### 자동 주입 후보

- 프로젝트 `CLAUDE.md`
- 글로벌 `~/.claude/CLAUDE.md`
- 프로젝트 `.claude/CLAUDE.md`
- Claude auto-memory: `~/.claude/projects/{slug}/memory/MEMORY.md`
- 글로벌 `~/.claude/settings.json`
- 프로젝트 `.claude/settings.json`, `.claude/settings.local.json`
- SessionStart hook stdout이 주입하는 파일/명령
- 설치된 skill들의 frontmatter `name`, `description`, invocation 관련 필드

### 수동 참조 후보

- `memory/_index.md`, `memory/progress.md`, `memory/topics/`, `memory/ideas/`
- `context/` 최신 세션 로그와 `## 다음 세션 시작 가이드`
- `docs/`, `ai-docs/` 대형 문서
- 프로젝트 `.claude/skills/`
- Claude 관련 `scripts/`, `hooks/`

### 권장 수집 명령

```bash
pwd
git rev-parse --show-toplevel
find . -maxdepth 2 \( -name 'CLAUDE.md' -o -name 'AGENTS.md' -o -name 'settings.json' -o -name 'settings.local.json' \) -print
wc -l -c CLAUDE.md memory/_index.md memory/progress.md 2>/dev/null
find context -maxdepth 1 -type f -name '*.md' -printf '%f\n' 2>/dev/null | sort -V | tail -3
find ~/.claude/skills .claude/skills -maxdepth 2 -name SKILL.md -print 2>/dev/null
```

`rg` 결과가 길어질 수 있으면 `-m`, `--max-count`, `head`, `wc`로 제한한다.

## 3단계 - 계기판 측정

각 항목은 0-5점으로 판정한다. 점수는 정밀 수치가 아니라 운영 판단용 신호다.

- **5점**: 가볍고 명확하다. 당장 유지.
- **4점**: 작은 정리 후보가 있지만 정상.
- **3점**: 노란불. 가까운 시점에 정리 필요.
- **2점**: 실제 행동 오류나 컨텍스트 낭비 가능성이 높다.
- **1점**: 반복 사고/충돌/복구 실패를 만들 가능성이 크다.
- **0점**: 보안, 데이터 손상, 실행 실패 같은 즉시 위험.

### A. Autoload Budget

시작 시 자동/준자동으로 들어가는 규칙과 메모리의 양을 본다.

측정:

- 자동 주입 후보 파일 라인 수/바이트 수
- skill description 총량과 과도하게 긴 description
- SessionStart hook stdout이 unbounded 출력인지
- 시작 지침이 "많이 읽어라" 형태로 늘어나는지

노란불:

- `CLAUDE.md`/`MEMORY.md`/hook 출력 중 하나라도 긴 운영 로그처럼 변함
- skill description이 많아져 실제 관련 skill이 묻힐 가능성
- 세션 시작마다 읽는 파일이 계속 늘어남

빨간불:

- hook이 context/log/transcript 원문을 제한 없이 출력
- 자동 주입 파일에 장기 changelog, 대형 체크리스트, 대형 원문이 들어감

### B. Recovery Quality

컨텍스트 압축/새 세션 후 바로 이어갈 수 있는지 본다.

측정:

- 최신 `context/*.md` 존재와 최신성
- `## 다음 세션 시작 가이드`가 짧고 실행 가능한지
- `memory/_index.md`가 주요 메모리를 가리키는지
- `memory/progress.md`가 현재 상태판 역할을 하는지
- 다음 행동/대기 항목/블로커가 구분되는지

노란불:

- 최신 세션 로그는 있지만 시작 가이드가 너무 길거나 산만함
- progress가 현재 상태와 과거 기록을 섞음

빨간불:

- 최신 세션 복구 포인터가 없음
- progress/context가 raw transcript, JSONL, 긴 커밋 로그로 대체됨

### C. Rule Conflict

같은 행동에 대해 서로 다른 파일이 반대 지시를 주는지 본다.

측정:

- `CLAUDE.md`, `AGENTS.md`, skill, hook, memory 간 중복/충돌
- "반드시", "절대", "항상", "금지" 같은 강한 지시의 위치와 일관성
- Claude 전용 지시가 Codex/OpenAI 지침에 그대로 섞였는지
- hook이 자동 수행하는 일을 문서가 수동 수행하라고 반복하는지

노란불:

- 같은 파일 목록이 여러 곳에 중복됨
- 프로젝트별 예외가 문서마다 다르게 표현됨

빨간불:

- 한 곳은 수정 금지, 다른 곳은 자동 수정 지시
- 하위에이전트/웹검색/메모리 수정 같은 고비용 행동이 상충함

### D. Provider Drift

Claude Code 또는 provider 기본 하네스 변화와 어긋난 전제를 찾는다.

측정:

- 버전/기능을 단정한 문구가 최신인지
- skill/hook/command/plugin 관련 설명이 공식 문서와 맞는지
- "이 도구는 항상 직접 호출 가능" 같은 stale 지시가 있는지
- provider가 기본 제공하는 기능을 개인 하네스가 중복 구현하는지

Drift Audit일 때만 공식 문서를 확인한다. 공식 문서를 확인했다면 보고서에 URL과 확인일을 남긴다.

노란불:

- 버전 숫자와 기능 팩트가 문서 안에 박혀 있지만 출처/확인일이 없음

빨간불:

- 현재 런타임에서 사라졌거나 deferred/permission 처리되는 기능을 즉시 사용하라고 지시
- provider 기본 정책과 반대로 작동하는 hook/skill 전제

### E. Bloat / 비대화

하네스 파일이 원래 역할을 잃고 누적 기록장이 되었는지 본다.

측정:

- `memory/progress.md` 줄 수, 긴 줄, 섹션 수
- `CLAUDE.md`가 운영 규칙 대신 역사/설명/사례를 품고 있는지
- `README`, `docs`, `context`가 자동 주입 경로로 잘못 들어왔는지
- 오래된 실험 산출물(`draft`, `v1`, `old`, `backup`, `tmp`)이 active 경로에 남았는지

노란불:

- 사람/AI가 읽기 부담스러운 긴 bullet, 긴 JSON-like 블록
- 완료된 작업이 progress의 절반 이상을 차지함

빨간불:

- progress가 대형 changelog로 변해 매 세션 읽기 부담
- 자동 주입 파일이 1회성 조사 원문을 포함

### F. Safety / Permission Surface

하네스가 위험한 행동을 조용히 자동화하고 있는지 본다.

측정:

- SessionStart, PreToolUse, PostToolUse, Stop, PreCompact hook의 명령
- hook 스크립트 실행 권한과 위치
- destructive 명령 패턴(`rm -rf`, `git reset`, `pkill -f`, broad chmod/chown)
- 외부 네트워크/credential 접근
- 사용자 승인 없이 쓰기/삭제/커밋/푸시를 유도하는 문구

노란불:

- hook 경로가 유지보수 불명확
- 권한/네트워크 접근 이유가 문서화되지 않음

빨간불:

- 자동 hook이 파일 수정/삭제/네트워크 전송을 수행
- 광범위 destructive 명령이 보호장치 없이 문서화됨

### G. Maintainability

사용자가 하네스를 직접 읽지 않아도 운영 가능한지 본다.

측정:

- 각 하네스 파일의 존재 이유가 한 줄로 설명되는지
- 유지/제거 기준이 있는지
- 중복 source of truth가 있는지
- 설치 위치와 원본 repo가 추적 가능한지
- 사람이 읽을 문서와 AI가 읽을 문서가 구분되는지

노란불:

- "왜 있는지"는 알겠지만 제거 조건이 없음

빨간불:

- 어느 파일이 원본인지 알 수 없음
- 수정해야 할 위치가 여러 곳이라 항상 동기화 실패 가능

## 4단계 - 판정 규칙

각 항목 점수 뒤에 다음 중 하나를 붙인다.

- **유지**: 지금 상태가 비용보다 효용이 큼.
- **수정**: 목적은 맞지만 크기/위치/표현을 줄여야 함.
- **제거**: provider 기본 기능 또는 다른 하네스와 중복되며 효용이 낮음.
- **격리**: 자동 주입 경로에서 빼고 필요할 때만 읽게 해야 함.
- **보류**: 정보가 부족해 지금 건드리면 위험.

전체 판정:

- 평균 4점 이상, 빨간불 없음: 정상 유지.
- 3점대 또는 노란불 3개 이상: 정리 권장.
- 2점대 또는 빨간불 1개 이상: 우선순위 정리 필요.
- 0-1점 항목 존재: 즉시 사용자 확인 후 수정 작업 분리.

## 5단계 - 보고 형식

보고서는 짧게 쓴다. 기본 출력은 70줄을 넘기지 않는다.

```markdown
# 하네스 계기판

점검일: YYYY-MM-DD
프로젝트: /path/to/project
모드: Quick | Drift | Deep
전체 판정: 정상 유지 | 정리 권장 | 우선순위 정리 필요 | 즉시 확인 필요

## 점수판

| 항목 | 점수 | 판정 | 핵심 근거 |
|---|---:|---|---|
| Autoload Budget | 4/5 | 유지 | ... |
| Recovery Quality | 3/5 | 수정 | ... |
| Rule Conflict | 5/5 | 유지 | ... |
| Provider Drift | 보류 | 보류 | 공식문서 확인 안 함 |
| Bloat / 비대화 | 2/5 | 수정 | ... |
| Safety / Permission | 4/5 | 유지 | ... |
| Maintainability | 3/5 | 수정 | ... |

## 주요 발견

1. **High** - ...
   - 근거: `file:line` 또는 명령 결과 요약
   - 영향: ...
   - 권고: 유지/수정/제거/격리/보류

## 바로 할 일

1. ...
2. ...
3. ...

## 보류/불확실

- ...
```

주요 발견은 최대 7개만 출력한다. 더 많으면 "추가 후보"로 한 줄만 남긴다.

## 증거 규칙

모든 지적에는 최소 하나의 증거를 붙인다.

- 파일이면 `path:line` 또는 `path` + 라인 수.
- 설정이면 key path. 예: `~/.claude/settings.json hooks.SessionStart`.
- hook이면 command path + 존재/권한.
- 공식문서면 URL + 확인일.
- 추정이면 "추정"이라고 표시한다.

확신 없는 provider 기능 팩트는 단정하지 않는다. "공식문서 확인 필요"로 보류한다.

## 금지 출력

- 긴 원문 인용
- 전체 파일 덤프
- 추상 프레임워크 설명
- 사용자가 바로 실행해야 하는 대형 작업 목록
- "완벽한 하네스 아키텍처" 설계
- 수정 패치 제안서

## 사용 예시

사용자: "하네스 점검해줘"

동작:

1. Quick Audit로 인벤토리 수집.
2. 7개 계기판 항목 점수화.
3. 주요 발견 최대 7개와 바로 할 일 3개 출력.
4. 사용자가 "1번 수정하자"라고 하면 skill은 종료되고 일반 수정 작업으로 전환.

사용자: "Opus 업데이트 이후 하네스 괴리 봐줘"

동작:

1. Drift Audit로 실행.
2. Claude Code 공식 문서/릴리즈 노트를 확인.
3. stale provider 전제만 따로 표시.
4. 수정은 하지 않고 판정만 출력.
