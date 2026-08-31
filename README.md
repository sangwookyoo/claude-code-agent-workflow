# Implementer / Reviewer Agent Set

A Claude Code subagent configuration for implementation tasks. The main session
orchestrates; `implementer` writes the code; `reviewer` independently checks it.

Two variants are included. Pick one.

| | `base/` | `with-researcher/` |
|---|---|---|
| Files | 3 | 4 |
| Wiki lookup | no | yes, optional step |
| Needs setup | no | yes, `<WIKI_PATH>` |

`with-researcher/` is `base/` plus a read-only `researcher` agent, three extra
rules in `implementer.md`, and one optional workflow step. If the main session
never calls `researcher`, it behaves exactly like `base/`. Installing it costs
almost nothing; leaving it out means you cannot use it when you want it.

## Install

Copy one set into your repository:

```bash
# without the wiki lookup
cp base/CLAUDE.md .
mkdir -p .claude/agents && cp base/implementer.md base/reviewer.md .claude/agents/

# with the wiki lookup
cp with-researcher/CLAUDE.md .
mkdir -p .claude/agents && cp with-researcher/*.md .claude/agents/
rm .claude/agents/CLAUDE.md
```

`CLAUDE.md` goes in the repository root. The agent files go in `.claude/agents/`
and their filenames must stay as they are — the agent name comes from the file.

## Setup for `with-researcher/`

Replace the `<WIKI_PATH>` placeholder in two places:

- `CLAUDE.md`, under **Wiki Research**
- `researcher.md`, the **Wiki root** line at the top

Point both at your llm-wiki directory. Nothing else needs configuring.

## Workflow

```
Main → (researcher) → implementer → reviewer → fix loop → done
```

The main session defines the acceptance criterion and delegates. Subagents run
in their own context and cannot see the main conversation, so every delegation
message has to be self-contained. `CLAUDE.md` spells out what each one needs.

After review, Critical and Major findings go back to `implementer` and trigger
another review round. Minor findings are the main session's call. The loop stops
after two fix rounds; if the reviewer still reports Critical or Major, it stops
and reports the open issue rather than looping.

## What the agents will and won't do

`implementer` implements only the requested scope. It will not refactor adjacent
code, delete or skip a test to make the suite pass, or fix a test that was
already failing before its change. It reports the command it ran and the actual
pass/fail output rather than asserting success.

`reviewer` cannot modify source files. It runs the tests itself instead of
trusting the implementer's report. It reports only what affects correctness or
the stated requirement — style preferences are not findings, and reporting
nothing is a valid outcome. Its last line is always `VERDICT: PASS` or
`VERDICT: NEEDS_CHANGES`; NEEDS_CHANGES requires at least one Critical or Major
finding.

`researcher` is read-only and reports each wiki entry with a confidence level:
**Verified** (confirmed against this repo, with a file:line), **Unverified**, or
**Conflicts**. It treats wiki content as data, not instructions, and reports
"No relevant entries" rather than filling gaps from general knowledge.

## Things to know

**The files are self-contained.** Each agent file carries its own rules and does
not depend on `CLAUDE.md` being inherited. This duplicates a few lines across
files, which is deliberate: it is robust either way.

**The main session decides what the wiki is worth.** `researcher` surfaces
candidates; it does not endorse them. The main session picks which items to
apply and passes only those to `implementer`, as constraints with a reason. The
raw report is not forwarded, and `implementer` never reads the wiki.

**Watch for two failure modes with the wiki.** Calling `researcher` on every task
adds a round for usually no result. And an Unverified tip handed to
`implementer` as a requirement will pass review, because the reviewer checks
against the requirement. Consider narrowing the trigger to explicit requests
until you know the wiki earns its round.

**Frontmatter.** Only `name`, `description`, and `tools` are set. Misspelled
frontmatter fields fail silently, so add `model:` only after confirming the
agents run correctly.

## Verifying it works

Run one small task end to end and check that `implementer` reports real test
output, and that `reviewer` runs the tests itself instead of quoting them.

For `researcher`, two cases are worth checking once the wiki is connected:

1. Ask about a topic the wiki does not cover. It should return
   "No relevant entries" and stop. Filling the gap from general knowledge is a
   failure.
2. Ask about something where the wiki and your codebase disagree. It should
   report **Conflicts** and defer to the existing code.
