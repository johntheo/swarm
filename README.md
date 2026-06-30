# swarm

> Run a virtual startup org from inside Claude Code. Founders, engineers, marketing. One invocation, then conversational.

A Claude Code plugin that turns your session into the **Chief of Staff of a virtual startup**. You talk normally; behind the scenes, domain leads (Founder, Eng Lead, Marketing) get consulted as subagents, workers ship code and copy in the background, and gated approvals keep production decisions in your hands.

## What you get

- **One persistent org per venture.** `~/swarm/<venture>/` holds personas, memory, decisions, queues.
- **Conversational front door.** Talk to any Claude session with the plugin installed — it auto-detects which venture you're scoped to.
- **Domain leads as subagents.** Strategic questions go to Founder; tech to Eng Lead; positioning to Marketing. Each has its own memory that accumulates across conversations. Transparent attribution always — you see who's speaking.
- **Background workers.** Code, copy, research, design, QA, review. Spawn in parallel; results surface when ready. The orchestrator keeps you informed without blocking you.
- **Gated approvals.** Anything consequential (deploys, sends, money) lands in a queue you drain when convenient. Ratchet trust over time by editing `policy.json`.
- **Ask-when-blocked.** Workers that hit a decision they can't make queue a question for you. Status line shows the count; orchestrator surfaces them on first message of a session.

## Install

```bash
claude plugin install swarm@<your-marketplace>
```

Or for local development:

```bash
git clone https://github.com/johntheo/swarm.git ~/.claude/plugins/cache/swarm
# add to enabledPlugins in ~/.claude/settings.json
```

## Quickstart

In any Claude Code session:

```
/swarm-new acme
```

The orchestrator will:
1. Ask where the product code should live.
2. Scaffold `~/swarm/acme/` and (optionally) `~/dev/acme/`.
3. Interview you about the venture (10–20 min, grilling-style — one question at a time with recommendations).
4. Synthesize a `venture-brief.md` — the org's anchor doc.
5. Offer to wake the leads for first-week planning, or hold them until you bring work.

After kickoff, just talk:

- _"What's pending?"_ → status, queue counts.
- _"Should we target dentists or therapists?"_ → spawns Founder, returns their take, updates Founder's memory.
- _"Build the onboarding flow."_ → Eng Lead briefs a developer worker; runs in the background; result surfaces when ready.
- _"Approve a-001, reject a-002 with 'go simpler'."_ → batch resolves queue items.
- _"How should we ship this given the launch?"_ → spawns Founder, Eng Lead, and Marketing in parallel; orchestrator synthesizes.

## Org shape (`lean`, the default)

| Role | Lifetime | Purpose |
|---|---|---|
| Chief of Staff (orchestrator) | Active Claude session + skill | Front door, routing, queue drainage |
| Founder | Persistent on disk | Market thesis, ICP, strategic calls |
| Eng Lead | Persistent on disk | Architecture, tech, code quality |
| Marketing Lead | Persistent on disk | Positioning, copy, channels |
| Workers (dev, copy, researcher, designer, QA, reviewer) | Ephemeral background subagents | Execution, in parallel |

Persistent leads accumulate memory in `~/swarm/<venture>/agents/<role>/memory/`. Personas live in the plugin (`templates/personas/`) and are copied at kickoff. The brief carries all per-venture context.

## What's in v1

- `/swarm-new <name>` — bootstrap a venture.
- `swarm` CLI: `new`, `here`, `list`, `switch`, `status`, `answer`, `approve`, `reject`, `ask`, `request-approval`, `archive`.
- Lead consultation (Founder, Eng Lead, Marketing Lead) with transparent attribution + memory write-back.
- Background workers with brief/result file contract + auto-retry-once on crash → escalate.
- Gated approvals queue + questions queue, drained conversationally.
- Project-pinned scoping with global fallback (`.swarm` pointer + `~/.swarm/active.json`).

## What's deferred

- Cron-driven scheduled work / off-keyboard progress.
- Worktrees for parallel coding (workers serialize on the repo in v1).
- MCP integrations (GitHub, Linear, Slack, Stripe).
- AI-tailored personas at kickoff.
- Cross-venture institutional memory.

See [DESIGN.md](./DESIGN.md) for the full spec.

## Filesystem

```
~/swarm/<venture>/         ← the org
  .swarm.json              ← marker (auto-detection)
  shape.json               ← active leads, repo pointer
  policy.json              ← gates, caps, escalation
  venture-brief.md         ← the anchor doc
  agents/<role>/
    persona.md             ← identity
    memory/                ← accumulates over the venture's life
    inbox.md
  questions/pending.json   ← worker-blocked questions
  approvals/pending.json   ← gated actions
  workers/spawned/<id>/    ← per-worker dir: brief.md, result.md, log.txt
  artifacts/               ← decks, research, marketing drafts

~/dev/<venture>/           ← the product (separate lifecycle)
  .swarm                   ← pointer back to swarm home

~/.swarm/active.json       ← last-engaged venture (global fallback)
```

## License

MIT
