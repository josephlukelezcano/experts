# experts

An Adobe CX Enterprise (Experience Cloud) consulting team, built as Claude Code subagents.

- `docs/adobe-platform-snapshot.md`: product knowledge base (Nov 2026)
- `docs/adobe-consulting-team.md`: firm roles and team sizing
- `.claude/agents/`: 11 specialist agents (see `CLAUDE.md` for the roster)

## Using the agents
Open this repo in Claude Code, then either:
- describe the work, and Claude routes it to the right specialist automatically
- name one directly: "Use the ajo-journey-architect to design a cart abandonment journey"
- @-mention one: `@aep-data-architect review this identity design`
- run a session as one agent: `claude --agent adobe-solution-architect`

Deliverables are written to `deliverables/<client-or-topic>/`.
