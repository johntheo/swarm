# Swarm Plugin — Locked Design Spec

## North star
A Claude Code plugin that lets a single user run a virtual startup org — founders, engineers, marketing — to take an idea from brainstorm through validation to production-quality output. One invocation, then conversational. Skills, files, memory, gated approvals under the hood; natural language on top.

## The model in one line
**Orchestrator (active Claude + `swarm` skill) holds the conversation; leads (subagents w/ persistent memory on disk) provide domain judgment; workers (background subagents) execute.**

Design rule that fell out and is load-bearing:
- Multi-turn with the user → orchestrator session + skill.
- One-shot artifact production → subagent.

## Org shape (`lean`, default)
| Role | Lifetime | Mechanism | Purpose |
|---|---|---|---|
| Orchestrator (Chief of Staff) | Per-session, active Claude | Skill loaded | Front door, coordination, drains queues, transparent attribution |
| Founder | Persistent on disk (memory accumulates) | Spawned as subagent per consultation | Market thesis, ICP, strategic calls |
| Eng Lead | Same | Same | Architecture, tech choices, code quality |
| Marketing Lead | Same | Same | Positioning, copy direction, channels |
| Workers (developer, copywriter, researcher, designer, QA, reviewer) | Ephemeral | Background subagents | Execution; in parallel, capped at 3 default |

Shape configurable per venture (`design-led`, `b2b-saas`, …). Personas are stable identities shipped in the plugin; **venture-brief.md** carries all per-venture context.

## Filesystem

```
~/swarm/<venture>/
  .swarm.yaml             ← marker file for plugin auto-detection
  shape.yaml              ← active leads, repo pointer, options
  policy.yaml             ← approvals policy, token caps, escalation timing
  venture-brief.md        ← the anchor doc; everyone reads it
  agents/<role>/
    persona.md            ← identity (from plugin template)
    skills/               ← curated allowlist symlinks
    memory/
      MEMORY.md           ← index
      *.md                ← topic files, accumulate
    inbox.md              ← optional in v1; carried for Phase 2
  workers/spawned/<id>/
    brief.md, result.md, log.txt, seen.flag
  workers/queue.md
  questions/pending.md, resolved.md
  approvals/pending.md, resolved.md
  artifacts/              ← decks, research, marketing drafts
  status/, digest/        ← Phase 2 (cron loops)

~/dev/<venture>/          ← actual product repo, lives separately
  .swarm                  ← pointer back to swarm home

~/.swarm/active.yaml      ← last-engaged venture (global fallback)
~/.claude/plugins/swarm/  ← plugin home: bin/swarm, skill, persona templates, worker subagent defs
```

## Key mechanisms

**Scoping (which swarm am I in?):** project-pinned with global fallback + NL override.
1. cwd walks up looking for `.swarm` → venture pinned.
2. Else `~/.swarm/active.yaml` → last-engaged venture.
3. NL always overrides ("how's beta?" forces beta for this turn).
4. Status line shows `swarm: acme · N pending`.

**Authority:** start everything gated. Consequential actions (merges, deploys, money, external comms) land in `approvals/pending.md`. Ratchet trust by editing `policy.yaml`. Secrets per-role under `.secrets/<role>/`; CoS/orchestrator never reads them, only routes gated actions.

**Communication:** inbox files as durable truth + optional cmux ping when a pane is open. Workers/leads write `result.md` with optional `memory_updates:` block; orchestrator is single writer to memory dirs (no races).

**Consultation pattern (tiered, transparent):**

| Type of ask | Who answers |
|---|---|
| Status / approvals / preview | Orchestrator |
| Strategic / market / positioning | Founder subagent |
| Architecture / tech / stack | Eng Lead subagent |
| Positioning / copy / channels | Marketing Lead subagent |
| Cross-functional | Multiple leads in parallel; orchestrator synthesizes |
| Building / drafting / researching | Lead briefs a worker; worker runs in background |

Attribution always visible:
> Let me ask Founder — that's a market call.
> **Founder:** … (Memory updated: `founder/market_thesis.md`)

**Worker contract:**
- Spawn = sync brief file + async `Agent(run_in_background: true)`.
- Three completion paths: success (writes `result.md`), blocked (writes to `questions/pending.md`, exits), crash (auto-retry once with same brief; second crash escalates).
- Concurrency: 3 parallel workers global cap; 1-at-a-time per lead; 15-min wall-clock timeout, tunable.
- Memory write-back: workers/leads emit `memory_updates:` in result; orchestrator applies serially.

**Pending state surfacing:** status line shows live count always; orchestrator surfaces queue on **first message of a session** ("you have N pending — drain now or save?"). Subsequent messages don't re-nag. Items pending >24h escalate to every-turn prepend.

## UX flow

**Install:** `claude plugin install swarm` (one time).

**Bootstrap:** `/swarm-new acme` in any Claude session.

1. Orchestrator asks repo location + shape conversationally.
2. Scaffolds dirs, copies persona templates, writes policy/shape, updates `active.yaml`.
3. Orchestrator enters kickoff (grilling skill loaded) inline — same session, one question at a time, recommendations each round.
4. After interview, orchestrator spawns `swarm-brief-writer` subagent. Subagent synthesizes transcript + seed brief into `venture-brief.md`. Returns one structured artifact.
5. Orchestrator surfaces brief summary; pauses for review. Offers parallel-3-lead first-week planning as opt-in (with token cost stated).

**Daily life:** in any Claude session.

- Status line ambient: `swarm: acme · 2 questions · 1 approval`.
- "What's going on?" → orchestrator reads state, summarizes.
- "Should we target dentists or therapists?" → spawns Founder, returns take, updates Founder memory.
- "Build the onboarding flow" → Eng Lead briefs a developer worker, runs in background, you keep chatting; result surfaces in next turn.
- "Approve a-001, reject q-002 with 'go simpler', hold rest" → batch resolution.

**Optional cmux surfaces (Phase 2):**

- `swarm watch` → cmux layout, one pane per active background worker, tailing output.
- `swarm focus founder` → cmux pane with a persistent Founder session for deep-work conversations.

## What's explicitly deferred / not in v1

- Cron loops / scheduled heartbeats / off-keyboard progress.
- Friday Read / comprehension digest as a system feature (still do it manually).
- Hard writer/verifier doubling on every output (apply only to high-stakes outputs as a worker option).
- Worktrees for parallel code work (workers serialize on the repo in v1).
- MCP connectors (GitHub, Linear, Slack).
- AI-tailored personas at kickoff.
- Cross-venture institutional memory.
- Auto-retry > 1 attempt; verifier-streak auto-pause; per-loop token budgets.
- CoS-as-persistent-pane.

## Phase plan

**Phase 0 — Foundation (build first):**

- `bin/swarm` CLI: `new`, `switch`, `here`, `status`, `answer`, `approve`, `archive`.
- `swarm` skill (the orchestrator's brain) — routing rules, attribution patterns, queue-surfacing protocol, brief loading.
- Persona templates for CoS, Founder, Eng Lead, Marketing Lead.
- Worker subagent definitions: developer, copywriter, researcher, designer, QA, reviewer.
- Default `policy.yaml` (everything gated, 3-worker cap, 24h escalation).
- Filesystem scaffolding logic + scoping resolver.
- Status line hook.

**Phase 1 — Kickoff & consultation:**

- `swarm-brief-writer` subagent.
- Inline grilling skill activation during kickoff.
- Lead consultation pattern with transparent attribution, parallel cross-functional spawns.
- Memory write-back via single-writer orchestrator.

**Phase 2 — Background work + cmux surfaces:**

- Background worker spawn with file-based contract.
- Crash handling: auto-retry once → escalate.
- Completion detection + first-message surfacing.
- `swarm watch`, `swarm focus <lead>` cmux integrations.

**Phase 3 (future) — Autonomy:**

- Scheduled loops (cron-driven daily standup, Friday Read).
- Worktrees for parallel code.
- Per-loop token budgets, anomaly auto-pause.
- Writer/verifier hard separation.

**Phase 4 (future) — Integrations:**

- MCP connectors: GitHub, Linear, Slack, Stripe.
- CI triage loop.
- Real-world action authority ratcheting.

---

# v0.2 addendum — Double Diamond + Delivery

Shipped in v0.2:

## Double Diamond as first-class state

`shape.json` gains a `phase` field: `discover | define | develop | deliver`. Kickoff sets `discover`. Advance with `swarm advance-phase <target>`, gated by an artifact:

| Transition | Required artifact |
|---|---|
| discover → define | `discovery-report.md` |
| define → develop | `venture-brief.md` (updated after define) |
| develop → deliver | `prototype-spec.md` |

Gates can be bypassed with `--force`. Every transition is logged to `shape.json` → `phase_history`.

Orchestrator biases behavior per phase (see SKILL.md). Deliver-phase adds finer approval action types and per-target deploy policy.

## Delivery workers

- **`swarm-instrumentation`**: adds PostHog analytics events + creates feature flags (initial 0% rollout) for new features. Maintains `artifacts/instrumentation/taxonomy.md` as the canonical event schema.
- **`swarm-deployer`**: runs deploys per target (`staging` | `production`) via GitHub Actions. Reads `policy.json` → `delivery.deploy_targets.<target>.gate`. Post-deploy: checks Sentry for new issue signatures.

Both only spawn in `deliver` phase.

## Policy for delivery

`policy.json` gains a `delivery` section:

```json
"delivery": {
  "deploy_targets": {
    "staging":    { "gate": "gated", "auto_approve_after_clean_runs": 5 },
    "production": { "gate": "gated" }
  },
  "feature_flag_rollout": { "auto_approve_at_or_below_percent": 0 },
  "external_comms":       { "auto_approve_at_or_below_recipients": 0 }
}
```

`authority.auto_approve` accepts action-type strings: `["staging-deploy", "internal-comms"]`. Ratchet trust over time by adding entries.

## MCP integrations (opt-in via userConfig)

Three MCPs are configured via `.mcp.json`:

- **GitHub** — hosted at `https://api.githubcopilot.com/mcp/` (HTTP transport). Auth via bearer token.
- **PostHog** — `npx -y @posthog/mcp` (stdio). Requires personal API key + project id.
- **Sentry** — hosted at `https://mcp.sentry.dev/mcp` (HTTP transport). Auth via bearer token.

All tokens are declared in `plugin.json` → `userConfig` as optional + sensitive. Plugin core works with tokens empty; only delivery workers need them.

## What v0.2 doesn't change

- The core model (orchestrator + skill; leads + workers as subagents).
- The scoping resolver.
- The worker contract (brief → result → memory_updates).
- Kickoff mode (inline grilling).

