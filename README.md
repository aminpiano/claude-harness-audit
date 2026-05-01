# claude-harness-audit

A **read-only harness measurement** Claude Code skill.

It does not redesign your harness. It measures whether the current harness is
still helping: context budget, session recovery, rule conflicts, provider drift,
bloat, permission risk, and maintainability.

## What is a harness?

`Agent = Model + Harness`.

The harness is everything around the model that shapes behavior:

- `CLAUDE.md`
- auto-memory / project memory
- hooks
- skills
- plugins
- session logs
- project docs
- permissions and runtime conventions

As Claude Code changes, a personal/project harness can drift. The useful answer
is not always "add more rules"; often it is "keep", "trim", "remove", or
"move this out of autoload".

## What this skill does

The skill produces a compact dashboard:

- **Autoload Budget** — how much context the harness consumes at startup
- **Recovery Quality** — whether a fresh session can resume quickly
- **Rule Conflict** — duplicated or contradictory instructions
- **Provider Drift** — stale assumptions after Claude Code/provider updates
- **Bloat / 비대화** — progress/docs becoming changelogs or raw logs
- **Safety / Permission Surface** — risky hooks or automated actions
- **Maintainability** — whether each harness piece has a clear reason to exist

Each area gets a 0-5 operational score plus one of:

- `유지` — keep
- `수정` — tune
- `제거` — remove
- `격리` — move out of autoload / read only when needed
- `보류` — not enough evidence

## What it does not do

- No file creation
- No file edits
- No deletes or moves
- No hook/script execution
- No template generation
- No "ideal harness architecture" redesign

It stops at measurement and recommendation. Actual changes happen only after
the user chooses a specific item.

## Installation

### Personal

```bash
git clone https://github.com/aminpiano/claude-harness-audit.git ~/.claude/skills/harness-audit
```

Or install only the skill file:

```bash
mkdir -p ~/.claude/skills/harness-audit
curl -fsSL https://raw.githubusercontent.com/aminpiano/claude-harness-audit/main/SKILL.md \
  -o ~/.claude/skills/harness-audit/SKILL.md
```

### Project-local

```bash
mkdir -p .claude/skills/harness-audit
curl -fsSL https://raw.githubusercontent.com/aminpiano/claude-harness-audit/main/SKILL.md \
  -o .claude/skills/harness-audit/SKILL.md
```

## Usage

Ask naturally:

- "하네스 점검해줘"
- "세션 설정 검토"
- "CLAUDE.md 재배치 필요한지 봐줘"
- "Claude Code 업데이트 이후 내 하네스 괴리 봐줘"

## Modes

- **Quick Audit**: default. Local inventory and dashboard scoring.
- **Drift Audit**: used when provider/Claude Code updates are part of the
  request. The skill verifies official docs before making drift claims.
- **Deep Audit**: only when explicitly requested. It still avoids raw
  transcript/log dumps.

## Output Shape

```markdown
# 하네스 계기판

점검일: YYYY-MM-DD
프로젝트: /path/to/project
모드: Quick
전체 판정: 정리 권장

## 점수판

| 항목 | 점수 | 판정 | 핵심 근거 |
|---|---:|---|---|
| Autoload Budget | 4/5 | 유지 | ... |
| Recovery Quality | 3/5 | 수정 | ... |
| Rule Conflict | 5/5 | 유지 | ... |

## 주요 발견

1. **High** - ...
   - 근거: `file:line`
   - 영향: ...
   - 권고: 수정

## 바로 할 일

1. ...
2. ...
3. ...
```

## Files

- `SKILL.md` — skill definition loaded by Claude Code
- `README.md` — repository documentation
- `LICENSE` — MIT

## References

- [Anthropic — Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/harness-engineering.html)
- [What Is an Agent Harness? — Firecrawl](https://www.firecrawl.dev/blog/what-is-an-agent-harness)

## License

MIT — see `LICENSE`.
