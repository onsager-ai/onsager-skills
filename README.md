# onsager-skills

Public **skills bundle** for [Onsager](https://github.com/onsager-ai/onsager) — the operating-procedures knowledge layer that pairs with portal's MCP server (the action layer). Together they are clause 1 of [ADR 0007](https://github.com/onsager-ai/onsager/blob/main/docs/adr/0007-tools-and-skills-as-the-public-contract.md): the **two-layer public contract** Onsager exposes to AI runtimes.

> **Cross-repo dev-process skills moved to [`onsager-ai/dev-skills`](https://github.com/onsager-ai/dev-skills).**
> `plan-dag`, `issue-spec`, `railway`, `ci-triage`, `web-testing`, and other
> engineering-methodology skills used across Onsager, Duhem, and lean-spec
> now live in a sibling bundle and install globally:
> `npx skills add -g onsager-ai/dev-skills --skill '*' -a claude-code`.
> This bundle is now exclusively the **user-facing Onsager MCP loop**.

Tools are *what* you can call. Skills are *when to call which tool*, *how to sequence them*, and *what shapes the arguments expect*. Without the skills, an LLM staring at 11 tool descriptions has to guess the workflow; with them, the LLM sees a trigger phrase → tool sequence → example shape, and ships the right call on the first try.

## Install

In a Claude Code session (project-scope, drops symlinks into `./.claude/skills/`):

```bash
npx skills add onsager-ai/onsager-skills
```

To install only a subset:

```bash
npx skills add onsager-ai/onsager-skills --skill onsager-design-workflow
npx skills add onsager-ai/onsager-skills --skill 'onsager-*'
```

User-global install (`~/.claude/skills/`):

```bash
npx skills add -g onsager-ai/onsager-skills
```

### Prerequisites

The Onsager MCP skills call the portal MCP server. You need:

- An Onsager portal URL (default `https://your-org.onsager.dev`; for local dev, `http://localhost:3002`).
- A Personal Access Token with workspace access. Create one in the dashboard at **Settings → Tokens**.
- Your MCP client configured to point at `<portal-url>/mcp/messages` with the PAT in the `Authorization: Bearer <token>` header.

## Skills

### Onsager MCP loop (paired with portal's MCP server)

Each skill is a `SKILL.md` with YAML frontmatter — `name`, `description`, trigger-phrase list, and `allowed_tools`. The body is the operating procedure: when the trigger fires, here is the sequence of MCP tool calls to make, the argument shapes to use, and the failure modes to watch for.

| Skill | Triggers (sample) | Tools it grants |
| --- | --- | --- |
| [`onsager-ftue-chat`](skills/onsager-ftue-chat/SKILL.md) | "I'm new to Onsager", "design my first workflow", "sketch a workflow for me", "draft an automation" | `propose_workflow_draft` |
| [`onsager-design-workflow`](skills/onsager-design-workflow/SKILL.md) | "design a workflow", "create an automation", "build a pipeline" | `propose_workflow`, `edit_workflow`, `list_workflows`, `schedule_workflow` |
| [`onsager-run-workflow`](skills/onsager-run-workflow/SKILL.md) | "run this workflow", "execute the pipeline", "trigger a run" | `run_workflow`, `list_workflows`, `list_runs`, `inspect_run` |
| [`onsager-triage-run`](skills/onsager-triage-run/SKILL.md) | "the run failed", "diagnose this", "why did it fail" | `inspect_run`, `get_stage_logs`, `propose_remediation`, `cancel_run` |
| [`onsager-explore-artifacts`](skills/onsager-explore-artifacts/SKILL.md) | "show me the artifacts", "what did this run produce" | `get_artifact`, `list_runs` |

### How the Onsager skills compose

The Onsager skills cover one product loop, with `onsager-ftue-chat` as the workspace-less on-ramp:

0. **First touch** — sketch a workflow draft before there's a workspace to land it in (`onsager-ftue-chat`). The dashboard's binding step promotes the draft into a real workflow.
1. **Design** a workflow against a workspace (`onsager-design-workflow`).
2. **Run** it manually or wait for its trigger to fire (`onsager-run-workflow`).
3. **Explore** the artifacts a run produced (`onsager-explore-artifacts`).
4. **Triage** a run that failed or got stuck (`onsager-triage-run`).

You can hold the whole loop in one chat — the trigger phrases are designed not to collide, so "design me a workflow, run it once, and show me what it produced" cleanly chains three skills back-to-back.

## Cross-repo contract (Onsager MCP skills)

The MCP tool registry is the single source of truth: [`crates/onsager-portal/src/mcp/registry.rs`](https://github.com/onsager-ai/onsager/blob/main/crates/onsager-portal/src/mcp/registry.rs) in the main repo. Two invariants are mechanically enforced by `xtask check-tools-and-skills`:

1. Every tool name listed in any skill's `allowed_tools` is a real, registered MCP tool.
2. Every registered MCP tool appears in `allowed_tools` of at least one skill — no orphan tools, no orphan grants.

That cross-check is exactly the [half-wired drift pattern](https://github.com/onsager-ai/onsager/blob/main/CLAUDE.md#architectural-drift-patterns-to-watch) caught for tools/skills, the way `check-events` catches producer-without-consumer for spine events.

To run the cross-check locally, with both repos checked out side-by-side:

```bash
cd onsager
ONSAGER_SKILLS_DIR=../onsager-skills cargo run -p xtask -- check-tools-and-skills
```

> **Note on the in-flight staging copy.** A duplicate of the four Onsager MCP skills also lives at `onsager/public-skills/` inside the main repo, where it has been the source of truth for `xtask check-tools-and-skills` (`ONSAGER_SKILLS_DIR=public-skills`). That staging copy is being retired in a follow-up: once CI is cut over to clone this repo (`onsager-ai/onsager-skills`) instead, `onsager/public-skills/` will be removed and this repo becomes the canonical entry point. Until then, the two copies must stay in sync.

## License

[MIT](LICENSE)
