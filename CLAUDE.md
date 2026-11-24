# Adobe Experts — Consulting Team Repository

This repo is an **Adobe CX Enterprise (formerly Experience Cloud) consulting practice** built as Claude Code subagents. The owner is an engineer who consults on Adobe products: AEP, Data Collection, RT-CDP, AJO, CJA, Adobe Analytics, Target and AEM.

## Knowledge base (read before answering Adobe questions)
- `docs/adobe-platform-snapshot.md`: current state of every product, 2026 changes, deprecations, sources. Snapshot date Nov 24, 2026.
- `docs/adobe-consulting-team.md`: firm roles, team sizing, engagement pods, hiring plan.

Adobe ships monthly. When a dated or numeric claim matters, verify it on experienceleague.adobe.com and update the snapshot if it's stale.

## The team (`.claude/agents/`)
Route work to the specialist whose description fits. For anything that spans several products, start with **adobe-solution-architect**. It coordinates the others.

| Agent | Use for |
|---|---|
| `adobe-solution-architect` | End-to-end, multi-product designs, discovery, final synthesis (lead) |
| `aep-data-architect` | XDM, identity graphs, profile, sources, Query Service, governance, licensing |
| `data-collection-engineer` | Web SDK, Mobile SDK, Tags, datastreams, Event Forwarding, consent, migrations |
| `rtcdp-consultant` | Audiences, segmentation, destinations, activation use cases |
| `ajo-journey-architect` | Journeys, campaigns, channels, deliverability, Experience Decisioning |
| `cja-analytics-consultant` | CJA, Adobe Analytics, AA→CJA / GA4→CJA migration, reporting |
| `target-personalization-consultant` | Experimentation, Target activities, Recommendations, edge personalization |
| `aem-architect` | AEM Sites / Edge Delivery Services, Assets, Cloud Manager, Dispatcher |
| `ai-agentic-strategist` | Coworker, Agent Orchestrator, MCP, Brand Visibility/GEO, Brand Concierge |
| `implementation-qa-engineer` | Audits, validation, test automation, reviewing other agents' output |
| `engagement-manager` | SOWs, estimates, staffing, timelines, RACI, proposals |

## Conventions
- Client deliverables go in `deliverables/<client-or-topic>/` as Markdown, plus JSON/SQL/JS where relevant.
- Never commit real client PII, credentials, API keys or org IDs. Use placeholders (`<ORG_ID>@AdobeOrg`, `<DATASTREAM_ID>`).
- Use current Adobe names (CX Enterprise, Coworker, Optimize activity, Experience Decisioning, Brand Visibility), and mention the old name when a client may still use it.
- Label feature availability as GA, LA, beta or announced.
- Agents keep project memory in `.claude/agent-memory/<agent>/`. Commit useful learnings, never client secrets.
