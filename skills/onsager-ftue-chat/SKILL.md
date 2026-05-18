---
name: onsager-ftue-chat
description: Design a first workflow draft from the workspace-less Onsager `/chat` entry — for users who haven't created a workspace, installed the GitHub App, or picked a repo yet. Triggers include "I'm new to Onsager", "show me what Onsager can do", "help me get started", "design my first workflow", "what's a workflow", "sketch a workflow for me", "draft a workflow", "can you propose a workflow", "set up an automation but I don't have a workspace yet". Drafts live client-side in the dashboard until the binding step promotes them into a real spine workflow; this skill never asks for workspace_id / install_id / repo. For workspace-scoped design (the user already has somewhere to land the workflow), hand off to `onsager-design-workflow`.
allowed_tools:
- propose_workflow_draft
---

# onsager-ftue-chat

A **workflow draft** is the pre-binding shape of an Onsager workflow: a `{name, trigger, stages}` document the user is authoring, with no workspace / install / repo bound to it yet. This skill is the agent loop for the workspace-less `/chat` entry — the surface a brand-new user lands on before they've created a workspace or installed the GitHub App. Drafts persist in the dashboard's local storage; the binding step (a separate dashboard flow) promotes the draft into a real spine workflow via `propose_workflow`.

If the user is workspace-scoped already — they've named a workspace, or the chat surface has one in scope — hand off to `onsager-design-workflow` and use `propose_workflow` directly.

## When this skill triggers

Phrases that should route here:

- "I'm new — show me what Onsager does"
- "design my first workflow"
- "sketch a workflow that runs on a labeled issue" (with no workspace context)
- "propose a workflow draft"
- "draft an automation for me"
- "help me get started, I haven't set anything up yet"
- "customize this template for my project" (from a TemplateGallery card)

If the user names a workspace, asks to *fire* a workflow, asks to *activate* a workflow, or otherwise expects substrate-side effects, hand off:

- workspace-scoped design / edit / schedule → `onsager-design-workflow`
- run an active workflow → `onsager-run-workflow`
- explore artifacts a run produced → `onsager-explore-artifacts`
- diagnose a stuck or failed run → `onsager-triage-run`

## Operating procedure

### Step 1 — do not ask for workspace context

FTUE callers reach this skill from the workspace-less `/chat` entry. They do **not** have a workspace_id, install_id, or repo bound. Do not ask for them. Do not call any workspace-scoped tool (`propose_workflow`, `edit_workflow`, `run_workflow`, `list_workflows`, `schedule_workflow`, …) — they will fail with `Unauthorized` or `InvalidParams`, and the FTUE preamble explicitly forbids them.

If the user has just clicked a template card, the composer pre-fills with `Customize "<template name>" for my project. <intent>`; treat that as the seed and propose a draft refined for what the user describes.

### Step 2 — sketch the shape

The draft document is the same canonical shape as a real workflow, minus the workspace context:

- `name` — short human label ("Triage every labeled issue").
- `trigger` — the GitHub event that fires the workflow. In v1 the FTUE-facing trigger shape is the issue-label webhook:
  - `install_id` — leave as empty string (`""`); the binding step fills it in from the GitHub App install the user picks.
  - `repo_owner`, `repo_name` — also empty strings unless the user named a specific repo in prose. The binding step's repo picker writes these.
  - `label` — the GitHub label the workflow should react to (e.g. `needs-triage`).
- `stages` — ordered list of `{ id, name, gate_kind, artifact_kind, config }` entries. `id` may be a stable string (`"stage-0"`); the dashboard regenerates ids when it canonicalises the document. `gate_kind` is kebab-case from [`onsager-portal::workflow::GateKind`](https://github.com/onsager-ai/onsager/blob/main/crates/onsager-portal/src/workflow.rs):
  - `agent-session` — dispatches an agent session with a prompt. `config: { "prompt": "…" }`.
  - `external-check` — pass/fail check (e.g. spec-link validation).
  - `governance` — synodic governance review.
  - `manual-approval` — parks for a human click.

  `artifact_kind` is one of `Issue` / `PullRequest` / `Code` / `Document` (or a workspace-defined string in non-FTUE flows). `Issue` is the FTUE default.

### Step 3 — call `propose_workflow_draft`

```json
{
  "name": "Triage every labeled issue",
  "trigger": {
    "install_id": "",
    "repo_owner": "",
    "repo_name": "",
    "label": "needs-triage"
  },
  "stages": [
    {
      "id": "stage-0",
      "name": "Triage agent",
      "gate_kind": "agent-session",
      "artifact_kind": "Issue",
      "config": { "prompt": "Classify this issue and add labels." }
    },
    {
      "id": "stage-1",
      "name": "Governance review",
      "gate_kind": "governance",
      "artifact_kind": "Issue",
      "config": {}
    }
  ]
}
```

The dashboard's tool-call extractor reads the same `{name, trigger, stages}` shape `propose_workflow` accepts; the draft variant just elides workspace context. The right-panel DAG preview re-renders from the tool args. The chat surface persists the draft to local storage keyed by user id (oldest of 50 evicted silently), so the user can close the tab and resume later.

Do **not** pass `workspace_id`, `install_id` as a non-empty value, or any other binding field. Do not pass `active: true`. Activation runs through the binding step in the dashboard, not through this tool.

### Step 4 — frame the result for the user

After the call, tell the user: "Drafted `<name>`. It's saved locally; open **Workflows → bind** when you're ready to point it at a workspace and repo." Use the production-line vocabulary on the first reply per the FTUE framing (trigger as "order intake", stages as "work stations", gates as "QC checkpoints"), then drop back to standard Workflow / Run / Artifact / Stage nouns.

If the user wants to iterate, just call `propose_workflow_draft` again with the revised shape — each call replaces the active draft's document in the right-panel preview.

## Common shapes (copy-paste templates)

### "Triage every labeled issue"

```json
{
  "name": "Triage every labeled issue",
  "trigger": {
    "install_id": "",
    "repo_owner": "",
    "repo_name": "",
    "label": "needs-triage"
  },
  "stages": [
    {
      "id": "stage-0",
      "name": "Triage agent",
      "gate_kind": "agent-session",
      "artifact_kind": "Issue",
      "config": { "prompt": "Classify this issue, add labels, and suggest an owner." }
    }
  ]
}
```

### "Issue → PR for any spec"

```json
{
  "name": "Spec issue to PR",
  "trigger": {
    "install_id": "",
    "repo_owner": "",
    "repo_name": "",
    "label": "spec"
  },
  "stages": [
    {
      "id": "stage-0",
      "name": "Implementer",
      "gate_kind": "agent-session",
      "artifact_kind": "Issue",
      "config": { "prompt": "Implement the spec on a feature branch and open a PR." }
    },
    {
      "id": "stage-1",
      "name": "Manual approval",
      "gate_kind": "manual-approval",
      "artifact_kind": "PullRequest",
      "config": {}
    }
  ]
}
```

### "Empty starter the user will edit"

```json
{
  "name": "Untitled draft",
  "trigger": {
    "install_id": "",
    "repo_owner": "",
    "repo_name": "",
    "label": ""
  },
  "stages": [
    {
      "id": "stage-0",
      "name": "Agent session",
      "gate_kind": "agent-session",
      "artifact_kind": "Issue",
      "config": { "prompt": "" }
    }
  ]
}
```

## Failure modes to watch for

- **The agent reaches for `propose_workflow` instead.** Workspace-less callers will get `InvalidParams` (missing `workspace_id`) or `Unauthorized`. Re-issue the call as `propose_workflow_draft`.
- **The agent asks for `workspace_id`, `install_id`, or a repo.** Those are the binding step's job. Stop asking and call `propose_workflow_draft` with empty trigger identity fields.
- **Schema validation error on `gate_kind`.** `gate_kind` is kebab-case (`agent-session`, not `agent_session`).
- **Draft round-trip lost on reload.** Drafts persist to `localStorage` under `onsager.drafts.<user_id>`; private-mode or quota-exhausted browsers log a warning and drop the persistence. Tell the user to copy the YAML out via the right-panel DAG → YAML toggle if they need to keep it.

## Related skills

- `onsager-design-workflow` — workspace-scoped design, edit, schedule. Use once the user binds the draft or names a workspace.
- `onsager-run-workflow` — fire an active workflow.
- `onsager-explore-artifacts` — inspect what a run produced.
- `onsager-triage-run` — diagnose a stuck or failed run.
