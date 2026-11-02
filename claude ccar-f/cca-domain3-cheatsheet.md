# CCA Domain 3 Cheat Sheet — Claude Code Configuration & Workflows

Nov 2, 2026 · @Si

## At a glance

Domain 3 is **20%** of the exam. It tests how you set up Claude Code for a team: persistent context in CLAUDE.md, plan mode for risky changes, reusable commands, hooks, CI/CD, and tool permissions.

| # | Task statement | Section below |
| --- | --- | --- |
| 3.1 | Configure persistent project context with CLAUDE.md | CLAUDE.md |
| 3.2 | Use plan mode for high-impact operations | Plan mode |
| 3.3 | Create custom slash commands for repeatable workflows | Slash commands |
| 3.4 | Integrate Claude Code into CI/CD pipelines | CI/CD |
| 3.5 | Apply hooks for deterministic post-action behavior | Hooks |
| 3.6 | Configure tool access with allowedTools and permissionMode | Tool access |

**The five rules that answer most questions**

1. CLAUDE.md is layered. Team rules go at the project root, stack-specific rules in subdirectories, and personal defaults in `~/.claude/CLAUDE.md`.
2. For a large, risky or irreversible change (10+ files, migrations, auth, CI/CD, deletions, major upgrades), use **plan mode first**.
3. A workflow you repeat becomes a **custom slash command**, committed to version control.
4. Anything that must happen **every time** (lint after each write) is a **hook**, not a line in CLAUDE.md.
5. In CI: run `claude -p` with JSON output, specific review criteria, no permission prompts, and path restrictions.

## CLAUDE.md (3.1)

Claude Code reads CLAUDE.md automatically at the start of each session. Think of it as "a tech lead in a file": build commands, architecture rules, standards and checklists that would otherwise live in people's heads.

| Level | Path | Holds |
| --- | --- | --- |
| User (global) | `~/.claude/CLAUDE.md` | Personal defaults: preferred language, test framework, package manager |
| Project root | `./CLAUDE.md` | Team rules: commands, architecture, code style, testing, PR checklist, security |
| Subdirectory | `./frontend/CLAUDE.md` | Stack-specific rules: React patterns, state management, accessibility |

The study guide says "the most specific file wins". See Latest SDK notes for how the official docs describe it.

**Monorepo pattern.** The root file holds shared rules (git conventions, PR format, CI commands). `frontend/` and `backend/` each get their own file for framework rules, such as React and design tokens, or type hints and FastAPI.

**What belongs in the project file**

- **Commands:** dev server, full test suite, and lint plus typecheck that must pass before a PR.
- **Architecture:** e.g. validate every API input with Zod; database access only through `src/repositories/`; services talk through events.
- **Code style:** strict TypeScript, no `any` (use `unknown` with type guards), explicit async error handling, a logger instead of `console.log`.
- **Testing:** unit tests for business logic, integration tests for API routes, an 80% coverage gate.
- **Security:** never log PII, sanitize inputs, follow OWASP Top 10.
- **CI reviewer context:** a list of known-safe patterns, so the reviewer doesn't flag them.

**Auto memory.** Claude Code also saves what it learns as it works: build commands, file locations, patterns and debugging insights. These notes carry across sessions.

## Plan mode (3.2)

In plan mode, Claude shows everything it intends to do before it changes any file. It is the human review checkpoint for high-stakes work.

| Operation | Why plan first |
| --- | --- |
| Refactor across 10+ files | Check scope before anything changes |
| Database schema migration | Irreversible in production |
| Authentication changes | Security-critical, hard to undo |
| CI/CD pipeline changes | Affects every deployment |
| Major dependency upgrade | Can break many call sites |
| Deleting files or directories | Irreversible |

**Flow:** run `/plan <task>`. Claude returns an analysis, an ordered list of files, a summary of each change, and a risk assessment. You approve, modify or cancel. Nothing runs before approval.

**What to check in a plan:** scope (all affected files?), order (dependencies first?), reversibility (is there a rollback?), tests (are tests updated?), surprises (does it touch files it shouldn't?).

| | Plan mode | `acceptEdits` |
| --- | --- | --- |
| Behavior | Show plan → human approves → execute | Auto-approves file edits |
| Use for | High-impact or irreversible changes | CI/CD, automated batch work |
| Never for | — | Production-impacting changes without review |

## Custom slash commands (3.3)

A slash command packages a multi-step workflow so everyone runs it the same way. Commands live in the repo and are shared through version control.

- `/review-pr`: lint, check test coverage, run a security scan, summarize findings.
- `/add-tests`: explore the structure, find gaps, generate tests, run them.
- `/deploy-staging`: build, test, deploy, smoke test, notify the team.

## Hooks (3.5)

Hooks run shell commands at fixed points in Claude Code's loop. They behave the same every time, which prompts and CLAUDE.md can't promise. The classic example: run the linter or formatter after every file write.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs -r npx prettier --write" }
        ]
      }
    ]
  }
}
```

This goes in `.claude/settings.json`. It uses the official schema; the study guide's version is out of date (see Latest SDK notes).

## Tool access (3.6)

| Agent | `allowed_tools` | `permission_mode` |
| --- | --- | --- |
| Read-only exploration | Read, Glob, Grep | `default` (asks before any write) |
| Development | + Write, Bash | `acceptEdits` (auto-approves file edits) |
| CI review, no shell | Read, Glob, Grep (+ Write for the report) | Bash in `disallowed_tools` |

`disallowed_tools` blocks a tool even if other settings would allow it.

**Path limits for test generation.** In `.claude/settings.json`, allow edits only under the tests folder, deny edits to `src` and `config`, and deny Bash. Then test generation can't modify production code.

## CI/CD (3.4)

```yaml
- run: |
    claude -p "Review this PR against CLAUDE.md: security, missing tests,
      architecture violations, performance, docs. Return JSON." \
      --output-format json --max-turns 20 > review.json
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

- `-p` runs non-interactively. `--output-format json` gives machine-readable output. `--max-turns` caps the run.
- Use `fetch-depth: 0` on checkout so the full git history is available for diff context.
- Ask for a **structured result**, e.g. `{"approved": bool, "blocking_issues": [{file, line, severity, issue, suggestion}]}`.
- Give **specific criteria**, not "review this code". Generic prompts produce generic reviews.
- Reduce false positives by listing known-safe patterns in CLAUDE.md, not by removing security checks.
- Use few-shot examples of a good and a bad review comment (see Domain 4).
- No one can approve prompts in CI. The study guide's answer is `acceptEdits`.

## Anti-patterns and exam triggers

| If the question says… | The answer is… | Trap answer |
| --- | --- | --- |
| Monorepo with React + Python, rules get mixed | Root CLAUDE.md + one per subdirectory | One giant root CLAUDE.md |
| A 40-file refactor or a DB migration | Plan mode first, review, then execute | Let Claude start straight away |
| Team runs the same review steps by hand | Custom slash command in the repo | Paste the steps into each chat |
| "Always run the linter after a file change" | `PostToolUse` hook | A line in CLAUDE.md |
| CI job hangs waiting for approval | Non-interactive permission mode (`acceptEdits` per guide) | `default` mode |
| CI review flags known-safe code | Add exceptions to CLAUDE.md | Remove the security checks |
| Reviews are vague | Specific criteria + good/bad few-shot examples | "Review this code" |
| Test generation edited `src/` | Path-scoped deny rules on `src`, allow on tests | Ask Claude not to touch `src` |
| Downstream tool parses review output | `--output-format json` + a fixed schema | Parse free text |
| Core assumption changed mid-project | Fresh session + structured summary (Domain 1) | Keep the old session going |

## Exam scenarios that use this domain

- **Scenario 2, code generation with Claude Code (D3, D5).** Hierarchical CLAUDE.md for a multi-stack monorepo, plan mode before 40+ file refactors and migrations, resume vs fresh across days, CI review. Traps: skipping plan mode; one root CLAUDE.md for every stack; continuing a session after a core assumption changed.
- **Scenario 4, developer productivity (D1, D2, D3).** Tool profiles by phase. Trap: every tool in every phase.
- **Scenario 5, Claude Code for CI/CD (D3, D4).** Security exceptions in CLAUDE.md, good/bad few-shot review examples, `acceptEdits` for unattended runs, path limits on test generation. Traps: generic prompts; removing security checks to cut false positives; `default` mode in CI.

## Latest SDK notes (official docs vs study guide)

The exam likely follows the study guide's model. These are the places where the current official Claude Code docs differ.

**Corrections**

| Study guide says | Official docs now |
| --- | --- |
| "The most specific CLAUDE.md wins" | Files are **concatenated, not overridden**. They load from the filesystem root down to the working directory, so closer files are read last. If two instructions conflict, Claude **may pick either**. Remove contradictions instead of relying on precedence. |
| Three levels | More layers: **managed policy** (organization), `~/.claude/CLAUDE.md`, `./CLAUDE.md` or `./.claude/CLAUDE.md`, **`CLAUDE.local.md`** (personal, gitignored), and **`.claude/rules/*.md`** (can be scoped to file paths). Files above the working directory load at launch; **subdirectory files load on demand** when Claude works there. `AGENTS.md` is read if there's no CLAUDE.md. |
| (not covered) | **`@path` imports** pull other files in (max 4 hops). Keep each CLAUDE.md **under ~200 lines**. `/init` generates a starter file. CLAUDE.md is context, not enforcement: to block an action, use a `PreToolUse` hook. |
| Auto memory lives in the project's `.claude/` | It lives in `~/.claude/projects/<project>/memory/` with a `MEMORY.md` index. The first 200 lines or 25 KB load each session. Toggle it in `/memory`. |
| Enter plan mode with `/plan` | Three ways: **Shift+Tab** cycles default → acceptEdits → plan; `claude --permission-mode plan`; or a `/plan` prefix on one prompt. Claude reads and explores but doesn't edit source until you approve. Approval options: continue in auto mode, approve each edit manually, or keep planning. **Ctrl+G** opens the plan in your editor. |
| Commands in `.claude/commands/` | Commands are now part of **skills**: `.claude/skills/<name>/SKILL.md` is recommended. Old command files still work, and a skill wins on a name clash. Frontmatter includes `description`, `argument-hint`, `allowed-tools`, `disable-model-invocation: true` (only you can run it, good for `/deploy`). Arguments: `$ARGUMENTS`, `$0`, `$1`. |
| Hook `"matcher": {"tool_name": "Write"}` and `${file}` | The matcher is a **string**: `"Write\|Edit"`, or a regex such as `"mcp__memory__.*"`. The hook receives **JSON on stdin**, e.g. `tool_input.file_path`. **Exit 2 blocks** on events that can block; exit 1 does **not**. `PostToolUse` can't undo the tool call. Hook types: `command`, `http`, `mcp_tool`, `prompt`, `agent`. |
| `acceptEdits` for CI; `default` stalls CI | Six modes: `default` (shown as Manual), `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`. The official CI recommendation is **`dontAsk` with an explicit allowlist**, or `auto`. In a `claude -p` run, calls that need approval are **denied rather than left waiting**. `acceptEdits` only auto-approves edits and common filesystem commands; other shell commands still need allow rules. For the exam, `acceptEdits` is still the expected answer. |
| CLAUDE.md always loads in CI | True for plain `claude -p`. But **`--bare`**, which is recommended for CI and will become the default, skips CLAUDE.md, hooks, skills and MCP servers. Pass context with `--append-system-prompt` or `--settings`. |
| A hand-written workflow that installs the CLI | The official integration is **`anthropics/claude-code-action@v1`**. It responds to `@claude` mentions, or runs automatically when you give it a `prompt` input. Pass CLI flags through `claude_args` (e.g. `--max-turns 5 --allowedTools ...`). Quick setup: `/install-github-app`. |
| `--output-format json` for structure | Also **`--json-schema '<schema>'`**: the validated result appears in a `structured_output` field. Check that the field exists, since it's missing if Claude declines. Output formats are `text`, `json` and `stream-json`. |

**Also useful**

- Deny rules apply in every permission mode, even `bypassPermissions`.
- Writes to protected paths such as `.git` and `.claude` are never auto-approved (except in `bypassPermissions`).
- `--allowedTools` uses permission-rule syntax: `Bash(git diff *)` allows any command starting with `git diff`.
- `--continue` resumes the latest session; `--resume <id>` resumes a specific one.

## Sources

- [CCA Prep — Domain 3 overview](https://claudecertprep.com/study-guide/domain3/overview)
- [CCA Prep — CLAUDE.md](https://claudecertprep.com/study-guide/domain3/claude-md)
- [CCA Prep — Plan mode](https://claudecertprep.com/study-guide/domain3/plan-mode)
- [CCA Prep — CI/CD](https://claudecertprep.com/study-guide/domain3/cicd)
- [CCA Prep — Anti-patterns](https://claudecertprep.com/study-guide/anti-patterns)
- [CCA Prep — Exam scenarios](https://claudecertprep.com/study-guide/exam-scenarios)
- [Claude Code — Memory and CLAUDE.md (official)](https://code.claude.com/docs/en/memory)
- [Claude Code — Permission modes (official)](https://code.claude.com/docs/en/permission-modes)
- [Claude Code — Skills (official)](https://code.claude.com/docs/en/skills)
- [Claude Code — Hooks (official)](https://code.claude.com/docs/en/hooks)
- [Claude Code — Run programmatically (official)](https://code.claude.com/docs/en/headless)
- [Claude Code — GitHub Actions (official)](https://code.claude.com/docs/en/github-actions)

CCA Prep is a community resource, not affiliated with Anthropic.
