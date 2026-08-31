# Project Instructions

## Code Rules

- Follow the existing architecture and coding style.
- Make the smallest change that satisfies the task.
- No unrelated refactoring. No modifying unrelated files.
- Prefer existing utilities over new abstractions.
- Do not modify generated files unless explicitly requested.
- If you edit a file for an experiment, restore it and confirm the restore.

## Agent Workflow

Implementation tasks run as:
Main -> (`researcher`) -> `implementer` -> `reviewer` -> fix loop -> done.

1. Main analyzes the task and defines the acceptance criterion.
2. Optionally, Main calls `researcher` (see below).
3. Main delegates the implementation to `implementer`.
4. Main delegates the result to `reviewer`.
5. Main decides what to do with the findings.
   - Critical or Major -> send back to `implementer`, then re-run `reviewer`.
   - Minor -> Main decides. Unfixed items go in the final summary.
6. Stop after 2 fix rounds. If the reviewer still reports Critical or Major,
   stop and report the open issue to the user instead of looping further.

## Wiki Research

The project keeps an llm-wiki of implementation tips at `<WIKI_PATH>`. It is
unverified reference material, not a source of truth.

Main calls `researcher` before delegating implementation when the task touches
an unfamiliar library, a known-tricky area, or a pattern the wiki likely covers.
Skip it for small or well-understood changes.

Main never reads the wiki directly. That is what `researcher` is for: the search
runs in a separate context and only the summary comes back.

Main, not the researcher, decides what gets adopted:

- Main reads the report and picks the items to apply.
- Main passes only the adopted items to `implementer`, as constraints inside the
  delegation message, with the reason for each.
- Main does not forward the raw report. `implementer` does not read the wiki.
- Unverified items are suggestions. Do not hand them to `implementer` as
  requirements.
- If a wiki item conflicts with existing code, existing code wins.
- Never expand the task scope because the wiki suggested something.

The `reviewer` reviews against the requirement, not against the wiki. Anything
Main adopted is simply part of the requirement.

## Delegation

Subagents do not see Main's conversation. Each delegation message must be
self-contained.

To `researcher`: one topic, and what you already know so it does not re-derive it.

To `implementer`: the task, the acceptance criterion, the relevant files if
known, and the adopted wiki items labeled as such.

To `reviewer`: what changed, the acceptance criterion, what `implementer`
already verified, and what is out of scope. Give the reviewer enough to be
adversarial without rediscovering context.

## Main Responsibilities

Main owns: requirements, task division, delegation, interpreting subagent
reports, deciding which findings require changes, deciding when the task is
complete, and the final summary.

Main does not write the implementation itself. Subagents do not decide the
workflow.
