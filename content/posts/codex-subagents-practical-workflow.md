+++
title = "Codex Subagents: A Practical Workflow for Safer Changes"
date = "2026-09-23"
image = "codex-subagents-practical-workflow.webp"
description = "A practical guide to Codex subagents: configure a scout and reviewer, keep edits sequential, and get focused findings with file references."
tags = ["codex", "ai", "developer-tools", "productivity"]
categories = ["Development"]
+++

One coding agent can become slow, noisy, and difficult to review on a large task. A feature request turns into directory searches, documentation checks, test output, and several competing implementation ideas. By the time there's a diff, the reasoning behind it can be hard to follow.

I want the main conversation to stay useful: what we're changing, why, and what still needs checking. Codex subagents give that work a structure. The trick is choosing assignments that deserve a separate investigation without turning a small patch into a coordination project.

Here's how I'd set up a modest workflow for an ordinary web application, using a recipe-list filter as an invented example.

## Keep one agent responsible for the result

The parent agent is the main conversation. It delegates bounded tasks to subagents, waits for their findings, and combines the results. Subagents do their work in separate agent threads.

For this example, I'd keep the parent responsible for scope and integration. A helper should answer a specific question, such as where filtering happens or which edge cases lack coverage. The parent still needs to decide whether those findings justify changing the implementation.

The [official Codex subagents documentation](https://learn.chatgpt.com/docs/agent-configuration/subagents) covers this local coding workflow. These examples configure Codex custom agents, not an application using the OpenAI Agents API.

I find it useful to write the expected answer before assigning the work. “Find the filtering entry point and its tests” has a clear finish. “Understand the application” doesn't. A focused assignment also makes a weak answer easier to spot.

## Split questions that can stand alone

Independent exploration is a good starting point. For a recipe filter, one investigation can trace the list component, the filter state, and the existing test helpers. It doesn't need a proposed implementation to identify those pieces.

Documentation checks are another useful assignment. I'd ask a helper to check one uncertain behavior against the documentation for the library version in use. The output should include the relevant source and explain its effect on the proposed change. A broad tutorial would just create more reading.

Test-gap analysis can happen alongside that exploration. Give the reviewer the intended behavior and ask what the current tests leave unproven. For example, does clearing the filter restore every recipe? What happens when there are no matches? Does the empty state offer an obvious way back?

Independent review is useful after a patch exists, too. I'd give the reviewer the acceptance criteria and diff, then ask it to trace behavior without relying on the implementer's explanation. That keeps the review centered on evidence in the code.

## Skip delegation when coordination costs more

I wouldn't divide a label correction among three agents. There's little to investigate, and reading several reports could take longer than reviewing the edit.

Dependent work also needs an explicit order. If the filter's state representation is still undecided, a task to implement persistence is premature. Settle the state contract first. Otherwise, the second task is building on a guess that may disappear.

Concurrent edits to the same files are an especially poor starting point. Two individually sensible changes can disagree about naming, behavior, or test setup. Separate filenames don't always mean separate work either: a component and its tests still share assumptions.

My default multi-agent coding workflow is parallel investigation, one writer, then review of a stable diff. Investigation and review can run together when they inspect the same unchanged baseline. A final patch review must wait for the patch.

## Give each role a clear boundary

I'd use three roles, with the parent acting as the implementer initially:

1. **Read-only codebase scout.** Locate relevant files, trace behavior, and report existing conventions. Ask for file references and unresolved questions, without edits.
2. **Workspace-write implementer.** Make the agreed change and run the relevant checks. Keep this role with one writer at a time, whether that's the parent or a later dedicated helper.
3. **Read-only reviewer.** Inspect correctness and test gaps. Report a concrete failure case, its location, and why it matters. Leave fixes to the implementer.

The handoff should be small enough to read in one sitting. I'd ask the scout for entry points, the implementer for changed behavior and validation, and the reviewer for actionable findings ordered by impact. None of them needs to retell every search.

Subagents consume more tokens and inherit the parent session's permissions and sandbox policy. Custom sandbox defaults can narrow access, but live parent overrides are reapplied when spawning. Check effective permissions before relying on a read-only setting.

## Configure two focused helpers

Project-scoped definitions live in `.codex/agents`, one TOML file per agent. Each requires `name`, `description`, and `developer_instructions`. Explicit model and reasoning settings keep these examples deliberate.

In `.codex/config.toml`, I'd start with two concurrent helper threads:

```toml
[agents]
max_concurrent_threads_per_session = 2
```

The limit excludes the primary agent. Two is a conservative starting choice for the scout and reviewer below. I'd raise it only after identifying another independent question worth answering.

Use `.codex/agents/codebase-scout.toml` for the scout:

```toml
name = "codebase-scout"
description = "Map the relevant implementation and tests without edits."
model = "gpt-6-luna"
model_reasoning_effort = "high"
sandbox_mode = "read-only"
developer_instructions = """
Inspect only the area assigned by the parent.
Return entry points, existing conventions, and test locations.
Cite file paths and line numbers. Separate facts from guesses.
Do not edit files or run commands that write to the workspace.
"""
```

Use `.codex/agents/change-reviewer.toml` for the reviewer:

```toml
name = "change-reviewer"
description = "Review behavior and test gaps without changing files."
model = "gpt-6-sol"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
Review the assigned requirements or completed diff.
Prioritize reproducible bugs and missing behavioral coverage.
Give file paths, line numbers, and a concrete failure scenario.
State uncertainty. Do not edit files or run write-producing commands.
"""
```

These instructions define the reports I want. I'd keep stylistic preferences out of the review unless they make the change harder to maintain. “This reset leaves the previous filter active” is useful; a list of alternative variable names usually isn't.

## Give the parent an explicit sequence

Ask Codex directly to delegate. Include the role split, when to wait, and the required output. Here's a prompt for the invented recipe application:

```text
Add a case-insensitive title filter to the recipe list.
An empty query shows all recipes. No matches shows an empty state.
Use existing UI and test conventions; do not add dependencies.

Before editing, delegate these read-only tasks in parallel:
- codebase-scout: locate the recipe list, state handling, and tests.
  Report the smallest plausible change area with file references.
- change-reviewer: inspect current filtering behavior and coverage.
  Identify edge cases and missing tests for these requirements.

Wait for both agents. Reconcile their findings and state your plan.
Then act as the only implementer, using workspace-write permissions
within the session's allowed scope. Make the change and run relevant
checks. Do not delegate concurrent edits.

Once edits stop, ask change-reviewer to inspect the completed diff.
Wait for that review. Address supported findings sequentially and
rerun affected checks. Report any unresolved issues.

Finish with changed behavior, validation results, and a concise
summary of both agents' findings with file paths and line numbers.
Do not commit or push.
```

The initial reviewer checks requirements and existing coverage. The later pass checks the actual implementation. Those are different jobs, and neither should be described as complete before its evidence exists.

## Read the findings before accepting the patch

For Codex code review, I'd read each finding as a claim to verify. Follow the file reference, trace the failure scenario, and compare it with the requirement. A confident report is still a report, not proof.

If the scout and reviewer disagree, ask a narrower follow-up. Does filtering happen before pagination or after it? Which function owns the query? Resolve the disputed behavior before requesting another edit.

I'd also separate checks that ran from checks that were merely suggested. A read-only reviewer may identify missing coverage without executing commands that generate files. The implementer should run those checks and report failures or environment limitations plainly.

Start with two read-only agents: one scout and one reviewer. Keep implementation with the parent until those assignments produce useful, concise findings. Add more orchestration only when you can name the independent work it will handle and explain how you'll review its result.
