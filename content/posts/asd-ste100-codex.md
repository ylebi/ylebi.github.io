+++
title = "Using ASD-STE100 Principles to Write Clearer Codex Prompts"
date = "2026-10-11"
description = "Use consistent terms, explicit actions, and short instructions to make coding-agent prompts easier to interpret."
image = "/images/asd-ste100-codex.webp"
tags = ["codex", "prompts", "agent-instructions", "asd-ste100"]
categories = ["Development"]
+++

I've used an ASD-STE100-inspired skill in Codex to improve my prompts. A few of its habits fit coding-agent instructions well: use one name for one thing, make actions explicit, and keep instructions short.

[ASD-STE100](https://www.asd-ste100.org/STE_faq.html) is a controlled language for technical documentation. It has writing rules and a controlled dictionary. I'm borrowing those habits, not claiming that my prompts meet the standard. You can request a copy from the [official downloads page](https://www.asd-ste100.org/STE_downloads.html).

## Stop making the agent guess

Say a prompt calls the same value a “query,” a “filter,” and a “search term.” Does Codex treat those as one thing or three? Call it a `query` throughout. “Add a test for an empty query” gives Codex an action. “Address empty-query coverage” leaves more to interpret.

Here is a **fictional** recipe app task. The first prompt sounds reasonable until you try to implement it:

```text
Make the recipe search better.
```

“Better” could mean almost anything. Once the human chooses the behavior, the prompt can say what to build:

```text
Update the recipe list search.
Match recipe titles without case sensitivity.
Show all recipes when the query is empty.
Show an empty-state message when no titles match.
Use the existing UI conventions.
Do not add dependencies.
Add tests for case matching, empty queries, and no matches.
Run the recipe search tests. Report the results.
```

None of those requirements came from the vague line. The human had to choose the search scope, matching rule, empty-query behavior, no-match behavior, dependency limit, and tests. A tidy rewrite of “make search better” would still be missing those decisions.

## Try it on your own prompts

If you installed `$asd-ste100` in your Codex setup, invoke it by name. [Codex supports explicit skill invocation](https://learn.chatgpt.com/docs/build-skills). This example requires a skill with that name in your setup.

```text
$asd-ste100 Rewrite these Codex instructions for clarity.
Preserve their meaning, conditions, and uncertainty.
Use consistent terms and explicit actions.
Do not add requirements or facts.
Ask me when the meaning is unclear.

Instructions:
[paste your instructions]
```

If you don't use that skill, paste this instead:

```text
Rewrite these instructions for a coding agent.
Use active voice.
Use consistent names for the same things and actions.
Put one action in each instruction.
Keep instructions short.
Preserve meaning, conditions, and uncertainty.
Do not add requirements.
Ask me when the meaning is unclear.

Instructions:
[paste your instructions]
```

Read the rewrite before using it. If “may” becomes “will,” the meaning changed. If the task still lacks a requirement, decide it yourself or ask a question. The missing behavior remains your call.
