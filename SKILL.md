---
name: agent-constitution-framework
description: A pre-action planning protocol for AI agents. FAZA 0-4 forces the agent to think before executing, preventing reactive mistakes and scope creep.
version: 1.0.0
author: nerudek
compatible-with: claude-code, hermes-agent, kimi-code, openclaw
tags: [agent-framework, planning, constitution, decision-protocol, ai-safety]
---

# Agent Constitution Framework — FAZA 0-4 Pre-Action Planning Protocol

## Problem

**AI agents act first, think later. Then they break things.**

An agent receives a task: "deploy the new configs." It immediately starts moving files, overwriting existing configs, restarting services. Three minutes later, the user notices that staging configs were deployed to production, the old backup was overwritten, and a critical service is down.

This isn't malicious. It's the default behavior of LLM agents: react to the prompt, execute the most obvious action, move on. There is no built-in "stop and think" mechanism. The agent's context window is a firehose of tokens, and the urge to produce output overrides the need to verify assumptions.

**The pattern repeats across every agent and every task:**

- **Assumption blindness:** "Deploy configs" assumes the current directory is the right one. It assumes the target machine is reachable. It assumes the configs are valid. None of these are checked.
- **Scope creep:** "Fix the banner editor" becomes "rewrite the entire codebase." The agent finds one bug, then another, then another, and never stops to ask "is this still the original task?"
- **Silent environment changes:** The agent modifies a config file, a symlink, a launchd service. It doesn't tell the user what changed. The next agent starts fresh and has no idea the environment is different.
- **Missing verification:** "Done" means "I ran the command." It doesn't mean "the command worked." The agent doesn't check output, doesn't verify state, doesn't confirm the result matches the intent.

**The specific failure:** Claude was asked to "update the HARNESS with communication rules." It wrote a 20-line addition to the file. But it also reformatted the entire file (400 lines), changed section numbering, and removed a security section it deemed "redundant." The communication rules were correct. Everything else was silently wrong. Rolling back took 30 minutes and required diffing the original from git.

**Why prompting doesn't fix this:**
"Think before you act" in the system prompt doesn't work. LLMs process prompts sequentially — they generate the response token by token. By the time they "think," they've already started acting. What's needed is a STRUCTURAL barrier: a protocol that forces the agent to externalize its plan before it can execute.

## Solution

**FAZA 0-4: A mandatory pre-action planning protocol.**

Before ANY action that modifies state (files, configs, services, git), the agent MUST complete five phases:

```
FAZA 0: UNDERSTAND — "What am I being asked to do?"
FAZA 1: SCOPE — "What exactly will I change?"
FAZA 2: VERIFY — "What assumptions am I making?"
FAZA 3: PLAN — "What are the exact steps?"
FAZA 4: EXECUTE — "Go." (only after FAZA 0-3 are written down)
```

The key insight: **FAZA 0-3 must be WRITTEN, not thought.** The agent externalizes its plan into text that the user can review. This creates a natural checkpoint — if the plan is wrong, the user catches it before execution.

### The Protocol

**FAZA 0 — UNDERSTAND** (30 seconds max)
- Restate the task in your own words
- Identify the GOAL (what should be true after)
- Identify CONSTRAINTS (what must NOT change)
- If anything is ambiguous, ASK before proceeding

**FAZA 1 — SCOPE** (60 seconds max)
- List every file, service, or config that will be touched
- Mark each as READ, WRITE, or DELETE
- Identify DEPENDENCIES (what must exist before)
- Set a LIMIT: "I will touch at most N files. If I need more, I'll stop and ask."

**FAZA 2 — VERIFY** (60 seconds max)
- List every assumption: "I assume X is true"
- For each assumption, write the CHECK: "I will verify X by running Y"
- If any check fails — STOP, report, ask
- Identify what could go WRONG and what the ROLLBACK is

**FAZA 3 — PLAN** (90 seconds max)
- Write the exact commands/steps in order
- For each step: what is the expected output?
- For each step: what is the rollback if it fails?
- Present the plan to the user for GO/NO-GO

**FAZA 4 — EXECUTE**
- Execute steps in order
- After EACH step: verify output matches expected
- If any step fails: execute rollback, STOP, report
- After ALL steps: verify the GOAL from FAZA 0 is achieved
- Write HANDOFF: what was done, what changed, what to watch

### What FAZA 0-4 Prevents

| Without FAZA 0-4 | With FAZA 0-4 |
|---|---|
| Agent deploys to wrong directory | FAZA 2 catches: "I assume pwd is /prod" — verified false |
| Agent reformats entire file for a 1-line change | FAZA 1 limits: "I will touch exactly lines 50-51" |
| Agent says "done" but config is broken | FAZA 4 verifies: "Config parses without errors? Checked." |
| Agent changes 10 files, user finds out later | FAZA 3 shows plan: "I will modify X, Y, Z. GO?" |
| Next agent has no idea what happened | FAZA 4 writes HANDOFF with exact changes |

### Integration with Existing Systems

FAZA 0-4 is a CONSTITUTION (decision protocol), not a HARNESS (rules engine). They work together:

- **HARNESS.md** says: "Use ACP bridge for communication, never delete without backup, always write HANDOFF."
- **Constitution** says: "Before any action, complete FAZA 0-3. Write it down. Get approval."

The Constitution is embedded in `~/agentos/constitution/HERMES_CONSTITUTION.md` and applies to all agents that modify state.

## FAQ

**Q1: Doesn't this slow everything down?**
Yes. That's the point. A 2-minute planning phase prevents 30-minute rollbacks. For trivial tasks ("echo hello"), skip FAZA 0-3. For anything that modifies files, configs, or services, the overhead is worth it.

**Q2: How do agents know when to use FAZA 0-4?**
If the task involves WRITE, DELETE, or EXECUTE operations, use FAZA 0-4. If it's READ-ONLY (searching, reading, analyzing), skip it. The constitution document defines the trigger conditions.

**Q3: What if the user says "just do it quickly"?**
The agent should respond: "FAZA 1: I will touch X and Y. FAZA 2: I assume Z. Confirm?" — a 10-second condensed version. The user can override, but the agent must externalize the scope.

**Q4: How is this different from chain-of-thought prompting?**
Chain-of-thought is internal reasoning. FAZA 0-4 is externalized planning that the user can REVIEW. The distinction is critical: internal reasoning is invisible and unverifiable. Externalized plans create accountability.

**Q5: Can FAZA 0-4 be automated?**
The phases are structural, not automated. The agent must generate the content. But future versions could have templates: "FAZA 1: You are about to modify [FILE]. List all other files that depend on it."

**Q6: What happens when FAZA 2 assumptions fail?**
The agent STOPS and reports. Example: "FAZA 2 FAIL: I assumed NATS was running. `nats ping` returned timeout. Please start NATS or confirm I should proceed without it."

**Q7: Does this work with sub-agents?**
Yes. When Vox delegates to Goose, Goose runs its own FAZA 0-3 before executing. Vox sees the plan and can approve or redirect. This prevents delegated tasks from going off-rails.

**Q8: How do you handle "I'll know it when I see it" tasks?**
These are the DANGER ZONE. FAZA 1 forces the agent to define a stopping condition: "I will explore until I find X, then stop and report." Without this, exploration becomes infinite loop.

**Q9: What's the relationship between Constitution and HARNESS?**
HARNESS = rules (always do X, never do Y). Constitution = process (before doing Z, complete phases 0-3). HARNESS is what you must/mustn't do. Constitution is how you decide what to do.

**Q10: Can this be applied to non-AI processes?**
Yes. The framework is human-readable and human-executable. A developer can use FAZA 0-4 before a production deploy. The same structure works for humans and agents alike.

**Q11: How do you measure compliance?**
After each task: did the agent write FAZA 0-3 before executing? Check the session transcript. Non-compliance = the agent skipped straight to action. This is tracked in session reviews.

**Q12: What about emergency hotfixes?**
FAZA 0-3 can be condensed to 30 seconds: "FAZA 1: restarting nginx. FAZA 2: assuming config is valid. FAZA 3: systemctl restart nginx." The phases still happen, just faster.

**Q13: Does this work with voice/talking agents?**
Yes. The agent can speak the phases: "I understand you want X. I'll need to change Y. I'm assuming Z is true. My plan is A, B, C. Shall I proceed?"

**Q14: How do you prevent FAZA from becoming boilerplate?**
The agent must write SPECIFIC content for each phase. "FAZA 2: I assume environment is correct" is boilerplate and rejected. "FAZA 2: I assume /etc/hosts has entry for kubuntu (verified with grep)" is specific and valid.

**Q15: Where is this documented for agents to find?**
`~/agentos/constitution/HERMES_CONSTITUTION.md` on M4. Referenced from HARNESS.md §0. Every agent reads HARNESS on startup, which points to the Constitution.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
