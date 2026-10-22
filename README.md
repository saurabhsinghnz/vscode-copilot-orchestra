# GitHub Copilot Orchestra

A set of four custom agents for **GitHub Copilot Chat in VS Code**. You work with **Conductor**, which coordinates codebase research, an approved plan, focused implementation, and independent review.

This repository contains agent definitions, not a VS Code extension or a standalone application. Install them into the project where you want to use the workflow, or add them as user-level agents.

## Quick start

### 1. Check prerequisites

- Use an up-to-date version of **VS Code** with GitHub Copilot Chat and access to custom agents and subagents.
- Sign in with a GitHub account that has access to the required Copilot features and models.
- Open the project you want to change. A Git repository is recommended for tracking changes and reviewing phase boundaries.

VS Code Insiders is not a blanket requirement. Use a version of VS Code that supports the required capabilities.

### 2. Install all four agents

**Recommended: install in your project.** Copy the four agent files from this repository into `.github/agents/` in your target project:

```text
your-project/
└── .github/
    └── agents/
        ├── conductor.agent.md
        ├── research.agent.md
        ├── implementation.agent.md
        └── code-review.agent.md
```

The files are stored at the root of this repository for distribution. **Putting them at the root of your target project is not the default VS Code discovery location.**

For example, run this from the downloaded Orchestra repository, replacing `/path/to/your-project` with your project's location:

```bash
mkdir -p /path/to/your-project/.github/agents
cp conductor.agent.md research.agent.md implementation.agent.md code-review.agent.md \
  /path/to/your-project/.github/agents/
```

Check for existing agent files before copying; do not overwrite customizations you want to keep. Commit the installed definitions if you want to share them with your team.

**Alternative: install for your user.** In VS Code, run **Chat: New Custom Agent** from the Command Palette, choose the user-level location, and create an agent for each of the four files. Replace each generated file's contents with the corresponding definition from this repository, including its YAML frontmatter. Keep the base names `conductor`, `research`, `implementation`, and `code-review` so the delegation names stay consistent.

For supported locations and UI details, see the [VS Code custom agents documentation](https://code.visualstudio.com/docs/agent-customization/custom-agents).

### 3. Check model availability

All four definitions currently specify:

```yaml
model: GPT-6 Luna (copilot)
```

If that model is not available to your account, edit the `model` field in **each installed agent file** to a model offered by your VS Code model picker. The agents do not currently use different models for different roles.

### 4. Start with Conductor

1. Open GitHub Copilot Chat in your target project.
2. Select **conductor** in the agent picker. The label is derived from the file name.
3. Describe the outcome, constraints, and how success should be verified.
4. Review the proposed plan and explicitly approve it before implementation starts.

Example request:

```text
Add pagination to the existing orders endpoint.
Follow the current API response conventions and preserve existing filters.
Add regression tests for page boundaries and invalid input.
Keep database schema changes out of scope.
```

You do not need to switch between agents manually. Conductor delegates the work and brings decisions back to you.

## The four agents

| Agent definition                                     | Role                                                                                                                                                              |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [conductor.agent.md](./conductor.agent.md)           | Coordinates research, creates the plan, delegates implementation and review, and manages approval checkpoints. Does not implement product-code changes itself.    |
| [research.agent.md](./research.agent.md)             | Gathers read-only codebase evidence: relevant code, existing patterns, constraints, and tests. Does not edit files, run commands or tests, or create the plan.    |
| [implementation.agent.md](./implementation.agent.md) | Implements one assigned phase, adds appropriate regression coverage, runs focused validation, and reports evidence and blockers.                                  |
| [code-review.agent.md](./code-review.agent.md)       | Independently reviews changes against acceptance criteria. Does not edit files or execute tests; it inspects code, diagnostics, and supplied validation evidence. |

The research agent has `user-invocable: false`, so it is intentionally hidden from the agent picker. Conductor can still invoke it as a subagent.

## How the workflow works

```text
Your request
  -> Research
  -> Proposed plan
  -> Your approval
  -> Implementation -> Independent review -> Phase report
  -> Your decision to continue
  -> Next phase, or final report
```

### Planning

Conductor inspects the workspace and delegates bounded research. It proposes the fewest cohesive phases needed; a single phase is fine. Each phase has a scope, acceptance criteria, dependencies, and a validation approach.

**Implementation waits for your explicit plan approval.** Material scope changes or unresolved product decisions also come back to you.

### Implementation and validation

The implementation agent works only on the assigned phase and preserves unrelated changes.

For bug fixes and testable behavior changes, the workflow calls for a reproducing failure or failing regression test, followed by the change and a successful rerun. Documentation, configuration, and other work where TDD is unsuitable use appropriate checks instead, with the exception explained.

Validation should use existing project tooling and the smallest checks that establish the acceptance criteria. Missing dependencies, checks that were not run, and unavailable environments must be reported, not treated as passes.

### Independent review

The review agent assesses the actual changes, acceptance criteria, regressions, and validation evidence.

| Review result    | What happens next                                                                                                                     |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `APPROVED`       | Conductor presents the phase report and a proposed commit message. Non-blocking recommendations may remain.                           |
| `NEEDS_REVISION` | Implementation addresses specific blocking findings or evidence gaps, then review reassesses the affected changes.                    |
| `FAILED`         | The review cannot establish correctness because required context, tools, or a decision is missing. Conductor stops for your guidance. |

Conductor stops after two unsuccessful revision attempts on the same blocking finding, or earlier if an attempt makes no progress.

### Phase checkpoints and commits

After each approved phase, Conductor reports what changed, validation results, remaining risks, and commit status, then proposes a commit message.

**You control both commits and continuation:**

- The workflow does not stage, commit, amend, or push without explicit authorization.
- Approving the next phase does **not** authorize a commit.
- Confirming a commit does **not** approve the next phase.
- You may explicitly continue without committing.

For example:

```text
I committed this phase. Proceed to the next phase.
```

Or:

```text
Skip the commit for now and proceed to the next phase.
```

After the final phase checkpoint authorizes completion, Conductor reconciles the results against the approved plan and provides a final report. Reports distinguish verified, partial, and blocked work rather than promising that every task will finish successfully.

## Plans, reports, and resuming work

**Plans and reports stay in chat and task state by default.** You do not need to create a `plans/` directory.

Repository files are created only when you explicitly request or approve them, or when repository instructions require them. To opt in, say:

```text
Save the approved plan and phase/final reports under plans/.
```

When enabled, the agents follow your repository's conventions. Otherwise, they use:

```text
plans/<task-name>-plan.md
plans/<task-name>-phase-<N>-complete.md
plans/<task-name>-complete.md
```

To resume interrupted work, provide the approved plan or saved reports, the last completed phase, and any changes made since then. Conductor should reconcile that information with the current workspace before continuing; previous approval and validation do not automatically cover new changes.

## Customizing the workflow

Edit the **installed copies** of the agent definitions:

- **Models:** Change each agent's `model` field to an available model.
- **Project conventions:** Add or maintain repository instructions for coding standards, validation commands, and scope constraints.
- **Tools:** Keep the frontmatter tool lists aligned with capabilities available in your VS Code environment. Naming a tool in instructions does not make it available.
- **Agent roles:** If you rename agents or add a specialist, update Conductor's delegation instructions to match.

These files provide instructions and tool configurations, not a guarantee of correctness. Review plans, changes, and validation evidence before accepting work.

## Troubleshooting

| Problem                                          | What to check                                                                                                                                                                                                             |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Conductor does not appear in the picker          | Check that the file ends in `.agent.md` and is installed in `.github/agents/` or a supported user-level location. Open **Configure Custom Agents** in the picker to inspect discovery.                                    |
| Research does not appear in the picker           | Expected: it is hidden with `user-invocable: false` and intended for delegation.                                                                                                                                          |
| A configured model is unavailable                | Update the `model` field in all affected installed definitions using a model available to your account.                                                                                                                   |
| Conductor cannot invoke a required agent or tool | Confirm that all four agents are installed and that your VS Code/Copilot setup exposes the required capabilities. Conductor should report the limitation and request an approved alternative, not silently skip the step. |
| The workflow stops after a plan or phase         | Expected: explicit approval is required. State whether you approve the plan, want changes, or authorize continuation.                                                                                                     |
| No plan or completion files are created          | Expected by default. Explicitly request repository artifacts if you want them.                                                                                                                                            |

## License

[MIT](./LICENSE) — Copyright (c) 2026 Saurabh Singh.
