---
name: harness-audit
description: 프로젝트 하네스(CLAUDE.md, MEMORY.md, lessons, hook, skill, docs 등) 상태를 점검하고 Claude Code 현재 기능/버전과의 충돌·괴리 지점을 찾아 보고한다. 진단 전용 — 파일 자동 수정 금지. 새 파일 생성 금지. "하네스 점검", "harness audit", "세션 설정 검토", "CLAUDE.md 재배치 필요한지", "클로드 설정 괴리 확인" 등 요청 시 호출.
---

# Harness Audit

## 목적

현재 프로젝트의 **하네스**(Claude Code 세션을 감싸는 모든 문서/설정/hook/skill)가
최신 Claude Code 기능·관행과 얼마나 정합한지 **진단**하고, 충돌·괴리 지점을
사용자에게 **보고**한다.

> **핵심 제약:** 이 skill은 **템플릿 생성기가 아니다**. 새 파일을 만들지
> 않는다. 새 구조를 강제하지 않는다. "이상적인 하네스 아키텍처"를 설계하지
> 않는다. 단지 **현재 상태를 있는 그대로 점검**하고 **괴리만 짚는다**.
>
> 실제 수정은 점검 결과를 본 사용자가 합의한 뒤 별도로 진행한다.
> 이 skill 실행 중에는 파일 수정 금지.

## 작동 원칙

1. **진단만**: 수정·생성·이동·삭제 금지. Read/Glob/Bash(ls, wc, cat) 중심.
2. **팩트 기준**: 추상 프레임워크(N축 × M규칙 같은 이론적 분류)를 **인용하지 마라**. 체크리스트 항목이 곧 판단 근거다.
3. **현재 기능 팩트 최신화**: Claude Code는 빠르게 바뀐다. 체크리스트의 "팩트" 섹션은 skill 실행 시점에 필요하면 `https://code.claude.com/docs/en/skills` 같은 공식 문서를 WebFetch로 재확인하라. 버전 차이가 감지되면 보고서에 명시.
4. **프로젝트 하드코딩 금지**: DCA-Commander든 다른 프로젝트든 동일하게 돌아야 한다. 경로는 `pwd` / `git rev-parse --show-toplevel` / `$HOME` 기반으로 동적으로 파악.
5. **과잉 제안 금지**: 근본 목적은 "현재 Claude Code 기능과 정합화"일 뿐이다. "이상적 프로젝트 구조"를 제안하지 마라.

## 1단계 — 인벤토리 수집

아래 위치를 차례로 스캔하고 **존재 여부 + 라인 수 + 개수**를 표로 정리한다.
존재하지 않으면 "없음"으로 기록. 존재하는데 비어 있으면 "빈 파일"로 기록.

### 자동 주입 경로 (매 세션 시 Claude context에 들어가는 것)
- 프로젝트 루트 `CLAUDE.md`
- 글로벌 `~/.claude/CLAUDE.md`
- 프로젝트 `.claude/CLAUDE.md` (모노레포/하위 디렉토리용)
- `~/.claude/projects/{project-slug}/memory/MEMORY.md` (auto-memory 인덱스)
- `~/.claude/settings.json` (`hooks.SessionStart`, `enabledPlugins`, `env`, `model`, `permissions`)
- 프로젝트 `.claude/settings.json` / `.claude/settings.local.json`

### 수동 참조 경로
- `ai-docs/` 또는 `docs/` 하위 문서 (파일 수 + 총 라인)
- `~/.claude/projects/{slug}/memory/*.md` (feedback_*, project_*, lessons_*, links_* 등 카테고리별 개수)
- `context/` 또는 세션 기록 디렉토리 (파일 수 + 최신 번호)
- `scripts/` 내 Claude 관련 스크립트 (`*session*`, `*boot*`, `*hook*`)

### Skill / Plugin 생태계
- `~/.claude/skills/` 각 디렉토리의 `SKILL.md` name + description 라인만 추출
- 프로젝트 `.claude/skills/` 마찬가지
- `~/.claude/settings.json` `enabledPlugins`
- `~/.claude/skills-unused/` 또는 비슷한 비활성 디렉토리 유무

### Hook 구성
- `~/.claude/settings.json` `hooks.*`의 각 이벤트별 등록 hook 나열 (SessionStart, PreToolUse, PostToolUse, Stop, PreCompact 등)
- 등록된 hook 스크립트 경로 + 존재 여부 + 실행 권한 여부

## 2단계 — Claude Code 현재 기능 팩트 (체크리스트 참고용)

아래는 **2.1.x 계열 기준**의 팩트. 하네스가 이것들과 어긋나면 "괴리"다.
**실행 시점에 버전이 더 올라갔으면 공식 문서를 다시 확인**하라.

### Skill 관련
- **Skill name + description은 세션 시작 시 자동으로 시스템 프롬프트에 주입**된다. 본문(SKILL.md body)은 `/skill-name` 호출 또는 Claude 자동 invocation 시에만 로드.
- **description + when_to_use 합계 1,536자 캡**. 넘으면 뒷부분 잘림.
- 전체 skill description budget은 **context window의 1% 또는 8,000자 (기본)**. 많이 설치될수록 뒷순위 description이 잘릴 위험.
- **SKILL.md 본문 500줄 이하 권장**.
- SKILL.md 프론트매터에서 사용 가능한 필드:
  - `disable-model-invocation: true` — Claude 자동 호출 불가 + **description도 context에 안 올라감**.
  - `user-invocable: false` — 사용자 `/` 메뉴에서 숨김 (하지만 Claude 자동 호출은 여전히 가능).
  - `allowed-tools`, `paths`, `context: fork`, `agent`, `hooks`, `effort`, `model`.
- **`` !`<command>` ``** 인라인 쉘 — skill 본문 렌더 전에 실행, 결과 치환.
- **Live change detection** — 세션 도중 skill 파일 수정하면 즉시 반영.
- **skill 비활성화 방법 3가지**: ① `disable-model-invocation: true` ② `~/.claude/skills/`에서 디렉토리 빼기(다른 위치로 이동) ③ `/permissions`에서 `Skill(name)` deny.

### Hook 관련
- **SessionStart hook의 stdout은 system message로 context에 주입**된다 (exit 0 시).
- hook 객체 구조: `hooks.{event}[*].hooks[*]` — 각 이벤트는 배열, 각 원소는 matcher+hooks 객체, hooks 배열에 `{type, command, timeout}` 원소.
- 여러 hook을 같은 이벤트에 등록 가능 — hooks 배열에 여러 원소 추가.

### Deferred tools (2.1.69 이후)
- WebSearch, WebFetch, TaskCreate, TaskUpdate, TaskList, WebFetch, NotebookEdit 등 일부 tool이 **deferred** 상태로 등장.
- 사용하려면 `ToolSearch`로 스키마 먼저 로드해야 함.
- 과거 skill이나 문서가 "WebSearch/WebFetch를 즉시 호출하라"고 지시하면 괴리.

### CLAUDE.md / MEMORY.md
- **CLAUDE.md 200줄 내외 권장**. 길수록 개별 룰의 심리적 가중치 희석.
- **MEMORY.md는 200줄 초과 시 잘림** (시스템 프롬프트 주입 시).
- 프로젝트 `CLAUDE.md`, 글로벌 `~/.claude/CLAUDE.md`, 모노레포 하위 `.claude/CLAUDE.md` 모두 자동 주입됨.

### Plugin / Marketplace
- `~/.claude/settings.json` `enabledPlugins`로 관리.
- 공식 마켓: `claude.com/plugins`, 커뮤니티: `claudemarketplaces.com`, `buildwithclaude.com` 등.

## 3단계 — 충돌/괴리 감지 체크리스트

아래 항목을 순서대로 점검. 해당하면 **[괴리]** 마커로 보고한다.
각 항목은 독립적 — 순서는 편의일 뿐이다.

### A. 세션 시작 규칙과 hook의 일치
- [ ] CLAUDE.md가 "세션 시작 시 X 파일 읽기"를 **텍스트로만** 지시하는가?
- [ ] 해당 지시를 자동화하는 SessionStart hook이 등록돼 있는가?
- [ ] 지시와 hook이 **같은 파일 목록**을 언급하는가? (불일치 = 유지보수 함정)
  → **[괴리]** 텍스트 지시는 Claude가 무시 가능. hook 자동화 + CLAUDE.md에 "hook이 처리함" 표시만 남기는 패턴 권장.

### B. Skill description 잘림 위험
- [ ] `~/.claude/skills/` + 프로젝트 `.claude/skills/` 합계 개수?
- [ ] 각 SKILL.md 프론트매터 `description` 라인 합계(대략)가 8,000자에 근접하는가?
- [ ] 현재 프로젝트 주제와 **무관한** skill이 몇 개인가? (예: React 프로젝트에 Swift 스킬)
- [ ] 무관한 skill에 `disable-model-invocation: true`가 설정돼 있는가, 아니면 그냥 description을 소비하고 있는가?
  → **[괴리]** 무관 skill 이동(`skills-unused/`) 또는 `disable-model-invocation: true` 권장.

### C. CLAUDE.md 최상단 30줄
- [ ] 최상단에 "1번 위반 = 즉시 사고"급 절대 금지 규칙이 있는가?
- [ ] 아니면 일반 설명·환경 전제·역할 기술이 자리를 차지하는가?
- [ ] 절대 금지 규칙이 본문 중간에 흩어져 있는가?
  → **[괴리]** 최상단은 가장 비싼 자리. Critical만 고정 권장.

### D. MEMORY.md 길이 + 내용 성격
- [ ] MEMORY.md가 200줄 초과? (자동 주입 잘림 위험)
- [ ] MEMORY.md 안에 "교훈/함정/과거 사고 사례"(본래 `lessons_*.md`로 분리돼야 할 것)가 직접 들어가 있는가?
- [ ] 이미 해소됐는데 남아있는 이슈가 있는가? (작성 날짜 확인)
- [ ] CLAUDE.md 규칙이 MEMORY.md에 **중복** 기록돼 있는가?
  → **[괴리]** 분리/정리/중복 제거 권장.

### E. Hook 구성 완전성
- [ ] SessionStart hook이 존재하는가?
- [ ] 프로젝트별 컨텍스트(필독 파일 cat 등)를 주입하고 있는가, 아니면 공용 hook만인가?
- [ ] hook 스크립트 경로가 유지보수 가능한 위치인가? (`~/.local/bin/`, 프로젝트 `scripts/` 등)
- [ ] 스크립트에 실행 권한(`chmod +x`)이 있는가?
- [ ] hook 내부에 하드코딩된 절대경로가 하네스 이전 시 깨질 소지가 있는가?

### F. Single Source of Truth 위반
- [ ] CLAUDE.md, hook 스크립트, memory 파일이 **같은 경로 리스트를 이중으로** 유지하는가?
- [ ] 한 곳 수정 시 다른 곳도 동기화가 필요한 구조인가?
  → **[괴리]** 한 곳으로 집중 권장. 경로/목록 관리는 스크립트 한 곳이 이상적.

### G. Legacy / 실험 산출물
- [ ] 과거 세션에서 만든 실험 문서/skill이 **active 상태**로 남아 있는가? (특히 `harness`, `bootstrap`, `v1`, `wip`, `draft` 이름)
- [ ] `docs/`, `ai-docs/` 하위에 큰 미사용 파일이 있는가? (1,000줄+)
- [ ] 실험 산출물에 대한 참조가 MEMORY.md "미해결 이슈" 등에 남아 있는가?
  → **[괴리]** `archive/` 이동 또는 삭제 권장. MEMORY.md 참조도 함께 정리.

### H. 프로젝트 특화 환경 분기
- [ ] 프로젝트가 2머신 이상(로컬/프로덕션, Mac/Linux 등) 워크플로우인가?
- [ ] CLAUDE.md가 머신별 분기를 명시하는가?
- [ ] 머신마다 hook/skill 구성이 독립적으로 정합한가?

### I. Deferred tool 사용 지시
- [ ] 하네스 또는 skill이 WebSearch / WebFetch / TaskCreate 등을 **즉시 호출**하라고 지시하는가?
- [ ] 현재 Claude Code 버전에서 해당 tool이 deferred인가?
  → **[괴리]** ToolSearch 선행 사용하도록 지시 변경 권장.

### J. Skill 중복 기능
- [ ] `~/.claude/skills/` 내 skill들이 **같은 목적**을 다른 이름으로 중복 구현하는가?
- [ ] 과거 작성한 DCA/프로젝트 전용 skill이 현재 구조에서 무의미해졌는가?

### K. Plugin 중복 / 충돌
- [ ] `enabledPlugins`에 활성화된 plugin이 이미 built-in bundled skill과 겹치는가?
- [ ] plugin 이름공간(`plugin:skill`)과 로컬 skill 이름이 충돌하는가?

## 4단계 — 보고서 작성 형식

점검이 끝나면 아래 형식으로 사용자에게 출력한다.

```markdown
# 하네스 점검 결과

> 점검 일시: {date}
> 프로젝트: {project-root}
> Claude Code 기준 버전 팩트: 2.1.x (skill description 1,536자 캡, SessionStart hook stdout 주입, deferred tools)

## 1. 현 인벤토리

### 자동 주입
| 위치 | 존재 | 라인/개수 | 비고 |
|---|---|---|---|
...

### 수동 참조
| 영역 | 파일 수 | 총 라인 | 비고 |
|---|---|---|---|
...

### Skill (총 N개)
| 카테고리 | 수 | 이름 |
|---|---|---|
...

### Hook 등록 현황
| 이벤트 | 스크립트 | 존재/권한 |
|---|---|---|
...

## 2. 괴리 발견 N건

### [괴리 1] {짧은 제목}
- **현 상태**: ...
- **팩트**: ... (버전 출처: ...)
- **영향**: ...
- **권고**: ...

(A~K 중 해당하는 항목만 반복)

## 3. 권고 우선순위

1. **(Critical)** ... — 즉시 수정 권장
2. **(High)** ...
3. **(Medium)** ...
4. **(Low)** ...

## 4. 다음 단계

이 중 어느 것부터 실행할지 사용자가 선택.
이 skill은 진단 전용이므로 실행은 별도 작업으로 진행됨.
```

## 금지 사항 재확인

이 skill은 아래 행동을 **절대** 하지 않는다:

- ❌ 파일 자동 수정 / 생성 / 삭제 / 이동
- ❌ "이상적 프로젝트 구조" 설계 제안
- ❌ 템플릿 파일 생성 (MAIN_HARNESS.md, _HARNESS_GUIDE.md 등)
- ❌ 추상 프레임워크 인용 (N축 × M규칙 × K원칙 같은 과거 실험 개념)
- ❌ 사용자 승인 없이 실행 단계로 넘어감
- ❌ 특정 프로젝트(DCA, Web, Mobile 등)에 하드코딩된 경로/규칙 가정

**허용되는 행동**:
- ✅ Read, Glob, Bash(ls/wc/cat/chmod -v 없이 읽기), Grep
- ✅ WebFetch로 공식 문서 재확인 (skill/hook 팩트 검증용)
- ✅ 사용자에게 질문 (어떤 영역을 중점 점검할지 등)
- ✅ 보고서 형태로 제안 출력

## 사용 예시

```
사용자: 하네스 점검해줘
→ skill 자동 호출
→ 인벤토리 수집 → 체크리스트 순회 → 보고서 출력
→ 사용자가 "A 항목부터 수정하자" 하면 skill 종료, 일반 작업으로 전환
```

```
사용자: Claude Code 업데이트됐는데 기존 CLAUDE.md가 아직 최신 기능 반영했는지 모르겠어
→ skill 자동 호출
→ 2단계 "현재 기능 팩트"를 공식 문서로 업데이트 확인
→ 3단계 체크리스트 I(deferred tools)와 B(skill description)부터 중점 점검
→ 괴리 지점만 보고
```
