# claude-harness-audit

A **diagnostic-only** Claude Code skill that audits your project's harness (`CLAUDE.md`, `MEMORY.md`, lessons, hooks, skills, docs) against the current Claude Code feature set and reports any conflicts or drift.

> **What it does NOT do**: generate templates, create new files, design an "ideal" project structure, or rewrite anything automatically. It only reads, diagnoses, and reports. All changes remain under your control.

## What is a "harness"?

Following Martin Fowler's / HumanLayer's definition: `Agent = Model + Harness`. The harness is everything wrapping the LLM — `CLAUDE.md` rules, memory files, hooks, skills, subagents, MCP servers, documentation — the whole scaffolding that shapes how a Claude Code session behaves in your project.

Over time, Claude Code itself evolves (2.1.x shipped 38+ patches between 2.1.69 and 2.1.107 alone, many undocumented). Your harness drifts out of sync with reality. This skill finds where.

## Installation

### Personal (available in every project)

```bash
mkdir -p ~/.claude/skills/harness-audit
curl -fsSL https://raw.githubusercontent.com/aminpiano/claude-harness-audit/main/SKILL.md \
  -o ~/.claude/skills/harness-audit/SKILL.md
```

Or clone the whole repo:

```bash
git clone https://github.com/aminpiano/claude-harness-audit.git ~/.claude/skills/harness-audit
```

### Project-local

```bash
mkdir -p .claude/skills/harness-audit
curl -fsSL https://raw.githubusercontent.com/aminpiano/claude-harness-audit/main/SKILL.md \
  -o .claude/skills/harness-audit/SKILL.md
```

Claude Code's live change detection picks up the new skill within the current session — no restart needed.

## Usage

Once installed, Claude Code loads the skill's description automatically at session start. Trigger it by asking naturally:

- "하네스 점검해줘" / "Audit my harness"
- "Is my CLAUDE.md aligned with the current Claude Code features?"
- "Claude Code 업데이트됐는데 내 설정 재배치 필요한지 봐줘"
- "세션 설정 검토"

The skill runs a four-stage audit:

1. **Inventory** — scans `CLAUDE.md`, `MEMORY.md`, `~/.claude/settings.json`, `ai-docs/`, `memory/`, `context/`, `~/.claude/skills/`, project `.claude/` dirs. Counts lines, files, hook registrations.
2. **Feature facts** — checks against the current Claude Code feature set (skill description 1,536-char cap, SessionStart stdout injection, deferred tools, hook frontmatter field, `` !`<cmd>` `` injection, etc.). Optionally re-verifies against official docs via `WebFetch`.
3. **Drift checklist** — walks checklist A–K (session-start rule vs hook alignment, skill description budget pressure, CLAUDE.md top-30-line usage, MEMORY.md length, hook wiring, Single-Source-of-Truth violations, legacy artifacts, machine branching, deferred tool guidance, skill duplication, plugin conflicts).
4. **Report** — outputs a structured markdown report with each drift item labeled `[괴리]` (conflict/drift), a recommendation, and a prioritized action list. **Stops there** — no automatic fixes.

## Design principles

- **Diagnostic, not prescriptive**: reports facts, leaves decisions to you.
- **No hardcoded project paths**: works for any project using `pwd`, `git rev-parse --show-toplevel`, `$HOME`.
- **Factual, not theoretical**: no abstract frameworks (e.g. "N-axis × M-rule × K-principle"). Every check is a concrete boolean backed by official Claude Code documentation.
- **Self-updating facts**: the skill body instructs Claude to re-verify the "feature facts" section against official documentation when needed, so it stays useful across Claude Code version bumps.
- **Bilingual (한국어/English)**: report template and internal prompts use Korean headings; the skill itself works regardless of user language.

## Files

- `SKILL.md` — the skill definition (frontmatter + audit instructions). This is the only file Claude Code reads.
- `README.md` — this file (not loaded by Claude Code).
- `LICENSE` — MIT.

## References

- [Anthropic — Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/harness-engineering.html)
- [What Is an Agent Harness? — Firecrawl](https://www.firecrawl.dev/blog/what-is-an-agent-harness)

## License

MIT — see `LICENSE`.
