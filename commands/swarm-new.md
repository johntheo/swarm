---
description: Bootstrap a new swarm venture and enter kickoff mode
---

You are about to bootstrap a new swarm venture. The `swarm` skill will be active for the rest of this conversation.

Arguments: `$ARGUMENTS`

Steps:

1. **Parse the arguments.** Expected forms:
   - `<name>` — venture name (kebab-case, required)
   - `<name> --shape <shape>` — optional shape (default: `lean`)
   - `<name> --brief "<one-liner>"` — optional seed brief
   - `<name> --no-repo` — skip product repo creation
   - `<name> --repo <path>` — point at existing repo path

   If no name was passed, ask the user for one in one line and stop.

2. **Confirm repo choice conversationally if not specified.** Ask: "Where should the product code live?"
   - Default: create at `~/dev/<name>` (run with no extra flag)
   - Existing repo: pass `--repo <path>`
   - No repo: pass `--no-repo`

   Default to creating at `~/dev/<name>` if the user says "default" or doesn't specify.

3. **Run the CLI** (Bash):
   ```
   swarm new <name> [--shape ...] [--brief ...] [--repo ... | --no-repo]
   ```

   Show the user the output (it lists what was created).

4. **Enter kickoff mode.** Per the `swarm` skill's "Kickoff mode" section: tell the user you'll interview them to produce `venture-brief.md`, then run the interview one question at a time with recommendations. When complete, spawn the `swarm-brief-writer` subagent to synthesize the brief.

5. After the brief is written, surface a 3-line summary and offer the parallel-3-lead first-week planning step as opt-in.

Important:

- Don't rush through the interview. The brief is load-bearing for every future spawn. 10-20 minutes of good elicitation is the price of a good org.
- Recommendations at every question. Never just "what do you think?" — give your read and the trade-off.
- If the user said `--brief "..."`, lead with "I read the seed. Let me sharpen it with a few questions" rather than starting from zero.
