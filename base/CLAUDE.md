# Project Instructions

## Code Rules

- Follow the existing architecture and coding style.
- Make the smallest change that satisfies the task.
- No unrelated refactoring. No modifying unrelated files.
- Prefer existing utilities over new abstractions.
- Do not modify generated files unless explicitly requested.
- If you edit a file for an experiment, restore it and confirm the restore.

## Agent Workflow

Implementation tasks run as: Main -> `implementer` -> `reviewer` -> fix loop -> done.

1. Main analyzes the task and defines the acceptance criterion.
2. Main delegates the implementation to `implementer`.
3. Main delegates the result to `reviewer`.
4. Main decides what to do with the findings.
   - Critical or Major -> send back to `implementer`, then re-run `reviewer`.
   - Minor -> Main decides. Unfixed items go in the final summary.
5. Stop after 2 fix rounds. If the reviewer still reports Critical or Major,
   stop and report the open issue to the user instead of looping further.

## Delegation

Subagents do not see Main's conversation. Each delegation message must be
self-contained.

To `implementer`: the task, the acceptance criterion, the relevant files if known.

To `reviewer`: what changed, the acceptance criterion, what `implementer`
already verified, and what is out of scope. Give the reviewer enough to be
adversarial without rediscovering context.

## Main Responsibilities

Main owns: requirements, task division, delegation, interpreting subagent
reports, deciding which findings require changes, deciding when the task is
complete, and the final summary.

Main does not write the implementation itself. Subagents do not decide the
workflow.
