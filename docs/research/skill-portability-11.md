# SKILL.md adapter mechanism — verified per harness

Resolves #11. Checks the assumption behind #1 / #8: "the portable layer is SKILL.md."

Sources: official docs only, fetched 2026-07-31.
- Claude Code: https://code.claude.com/docs/en/skills
- Codex CLI: https://developers.openai.com/codex/skills (build-skills), https://developers.openai.com/codex/sandboxing
- Cursor: https://cursor.com/docs/skills
- Pi: https://pi.dev/docs/latest/skills

## Verdict table

| Harness | Reads SKILL.md? | Directories | Can push to localhost API? | Caveats |
|---|---|---|---|---|
| Claude Code | Yes, native | `~/.claude/skills/`, `.claude/skills/` (project, walks up to repo root, nested subdirs), plugin `skills/` | Yes | Full Bash tool access (curl works), permission-gated like any tool call. Does **not** natively read `~/.agents/skills` — needs the symlink Mo already uses. `disableSkillShellExecution` can block the `!command` shell-preprocessing feature only, not Claude's own Bash tool. |
| Codex CLI | Yes, native | `.agents/skills` (repo, walks up to root), `$HOME/.agents/skills` (user), `/etc/codex/skills` (admin), bundled | Conditional | Reads `~/.agents/skills` directly — Mo's shared skills dir already covers Codex without a `~/.codex/skills` symlink. **Network access is off by default in every sandbox mode**, including `workspace-write`. A localhost curl needs `network_access = true` under `[sandbox_workspace_write]` in `~/.codex/config.toml`, or per-run approval. This is the one real blocker for "push to localhost API" out of the box. |
| Cursor | Yes, native (added ~v2.4) | Project: `.agents/skills/`, `.cursor/skills/`; Global: `~/.agents/skills/`, `~/.cursor/skills/`. Also reads `.claude/skills/` and `.codex/skills/` for compatibility. | Yes | Agent has a terminal/shell tool; skills' `scripts/` directories are explicitly documented as executable. Newest of the four — no long track record. Frontmatter `name` is restricted to lowercase/hyphens; has a `paths` glob-scoping field the others don't use the same way. |
| Pi | Yes, native | Global: `~/.pi/agent/skills/`, `~/.agents/skills/`; Project: `.pi/skills/`, `.agents/skills/` (cwd up to repo root); also `pi.skills` in `package.json` | Yes | General tool-use-loop coding agent; docs state a skill "may include executable code the model invokes." Diverges from the Agent Skills standard: skill name doesn't have to match its parent directory name. |

## Notes

- All four harnesses independently implement the open **Agent Skills standard** (agentskills.io), not just Claude-specific behavior. A skill built for Claude Code is expected to work unmodified in the other three per their own docs.
- `~/.agents/skills` is the one directory that three of the four (Codex, Cursor, Pi) read natively. Claude Code is the outlier — it only reads `~/.claude/skills` / `.claude/skills`, which is why Mo's existing dual-symlink pattern (canonical skill in `~/.agents/skills`, symlinked into both `~/.claude/skills` and `~/.codex/skills`) is the right shape, and could be simplified further: Codex doesn't need its own symlink since it already reads `~/.agents/skills` directly.
- The only hard blocker to "skill → shell → push to localhost API" across all four is Codex's sandboxed network access, which is off by default and needs an explicit config or approval per run.
