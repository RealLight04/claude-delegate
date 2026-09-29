---
name: delegate
description: When several independent tasks have piled up, grade each one by risk and complexity, pick the right model (Haiku/Sonnet/Opus) for it, and actually hand it to a subagent running that model. Use when a list of 3+ independent to-dos has just been assembled, when the user says "do all of these" or "work through this list", or when it is unclear which model should handle the work. Do not use for a single one-off instruction — just do that directly.
argument-hint: "[optional: the task list to delegate — empty means use the list built up in this conversation]"
allowed-tools: Read, Grep, Glob, Bash, Agent
---

## When NOT to use this

Delegation is not free. A subagent starts with **zero knowledge of this conversation**.
So in these cases, skip the skill and do the work directly.

- **There are only 1–2 tasks.** Grading and prompt-writing overhead costs more than it saves.
- **The tasks depend on each other.** If one's output is the next one's input, nothing runs in
  parallel. A session that already holds the context will finish sequential work faster.
- **This session already built up the context.** If you have read the files and gone several
  turns with the user, rewriting that context into a prompt eats the entire benefit.
- **The input is review findings.** Use the `review-fix` skill if it is installed. It has
  safeguards built around that specific input shape.
- **It is one deep, indivisible task.** → go to "Recommending a session model" below.

# Grading a task list and delegating by model

## Why this exists

When work piles up, running every item through the same model either wastes an expensive model on
trivia or puts a cheap model on something it will quietly get wrong. This file fixes the criteria
so the call is not made from vibes each time.

**This skill does not just recommend.** It uses the Agent tool's `model` parameter to actually hand
the work to that model. Orchestration — grading, grouping, verification — stays in this session;
the file edits happen in the chosen model's subagent.

## Input

- If the user passed arguments (they arrive on the `ARGUMENTS:` line at the end of this prompt),
  treat them as the task list.
- Otherwise use the list built up in this conversation — to-dos the user enumerated, improvements
  you proposed and they approved, a plan's checklist, to-dos that fell out of research. The source
  does not matter.
- If the list is vague, **write it out and show the user before starting.** Never begin delegating
  from a list that only exists in your head.

## Step 1 — Is it in a shape that can be delegated? (most important)

Review findings are safe to hand off because they are self-contained (file, line, failure
scenario). A general task list is not. **"Fix that thing we talked about" cannot go to an agent
that has no idea what was talked about.**

For each task, ask: **could a brand-new agent with zero context read this instruction and execute
it correctly?**

- If not → **make it concrete first.** Read the files yourself and pin down the target file, the
  location, what changes, and the criteria for judging it — then hand it over.
- If it still cannot be made concrete without the conversation → do not delegate it. Do it here.

**Never delegate something still vague. This is the skill's biggest failure mode.** An agent with
no context does not stop when it gets stuck — it invents something plausible and edits the wrong
thing.

Before starting, capture a **baseline** in every repository the tasks touch (`git -C <repo>` for any
other than the current one): `git -c core.quotePath=false status --porcelain --untracked-files=all` (so each untracked file is
listed on its own, not just its directory), plus `git hash-object <path>` for every file it lists. For a
rename, hash the new path; strip the quotes git adds around paths that contain spaces. A deleted path has no hash; record it as
deleted. The repo may already have unrelated modified or untracked files, and this skill may edit one
of them. Paths alone cannot show that; the hashes can. You need the baseline to separate "what this
skill changed" from "what was already there" when you report at the end.

## Step 2 — Grade each task

**When in doubt, grade up.** Never grade down to save money — the cost of a bad edit exceeds the
cost of a better model.

| Grade | Criteria | Examples | Model |
|---|---|---|---|
| Clerical | Mechanical. Almost no judgment, easy to reverse | Value substitution, removing dead code, typos, formatting, swapping in an already-decided token | `haiku` |
| Standard | Ordinary work requiring an understanding of surrounding logic and existing patterns | Bug fixes, routine refactors, adding a UI component, a new feature following an established pattern | `sonnet` |
| High-risk | Hard to reverse, or wide blast radius if wrong | Auth / payments / personal data, concurrency and transactions, architectural change, DB migrations, behavior changes spanning files, anything where the tradeoff is not clear-cut | `opus` |

Read the target project's CLAUDE.md before grading. The traps written there change the grade — if
it states a pairing rule ("change this file and you must change that one too"), the real scope is
the whole pair, and the grade follows that larger scope.

**Tiers outside this table.** It names `haiku`/`sonnet`/`opus` because those are the delegation
tiers, cheapest to most expensive. A new release inside one of them needs no change here. If the
Agent tool offers another tier (today `fable`, above Opus), do not slot it into this ladder on
assumption. Check how it is positioned first. `fable` is documented for demanding reasoning and
long-horizon agentic work, and it costs more and responds slower than Opus. A short, self-contained
task for a zero-context subagent rarely needs that, so it stays out of the table and appears under
"Recommending a session model" below.

## Step 3 — Group, then delegate

**Tasks touching the same file — or files bound together by a CLAUDE.md pairing rule — go in the
same Agent call.** Sent separately they overwrite each other or land half-applied. Where files do
not overlap, send them in parallel even if the grades differ.

**If a group contains mixed grades, send the whole group at the highest grade in it.** Do not
downgrade a group to clerical just because a clerical task shares a file with a high-risk one.

**Once grouped, if there is only one group, re-examine whether delegating is worth it at all.**
Whether the list had 3 items or 10, if they all converge on one file (very common — UI polish
items all landing in a single stylesheet), nothing runs in parallel, so the only remaining benefit
is saving main-session context. If this session has already read that file, even that is gone — do
the work directly. **The threshold is the number of groups after grouping, not the number of items.**

Each Agent call must state:

- Exactly what to do (file, location, what changes, criteria), for every task in the group
- That it should do **only these tasks** — no fixing other problems it happens to notice
- The relevant CLAUDE.md rules, including "you must also change this file" if a pairing rule applies
- To delete any scratch or cache files its own testing creates (temp scripts, `__pycache__`)
- Not to commit

### If the target is outside the current project, pass its CLAUDE.md through by hand

A subagent is given the CLAUDE.md of the **current working directory**. If the files to change live
in a different project folder, that project's CLAUDE.md **is not included** — the agent works
without knowing that project's rules.

→ When the target is outside the current project, **read that project's CLAUDE.md yourself and
quote the relevant rules into the prompt.** Use absolute paths throughout.

## Step 4 — Verify

**Never take "done" at face value.** The weight of verification varies by grade; skipping it does not.

- Clerical — look at the result yourself (read the file, or check `git status`).
- Standard — the above, plus run the smoke test or test script if one exists. For a UI change, a
  passing build is not the check: if a browser automation tool (e.g. Playwright) is available,
  exercise the behavior the task names.
- High-risk — the above, plus read the diff yourself and confirm it stayed inside scope.

When verification does not match expectations, do not let it slide:

1. Retry with the same model, stating what failed and the exact scope more explicitly.
2. If that fails, retry once, one grade up (clerical → standard, standard → high-risk). Never
   escalate to a model outside the Step 2 table.
3. If that retry also fails, or there is no grade above in that table, stop. Report what you tried and the current
   state to the user. Do not retry forever, and do not leave it quietly broken.

## Step 5 — Report

A table per task: task / grade chosen / model actually used / why (one line) / outcome (done,
skipped, failed after retry).

Anything you did yourself instead of delegating, or left out as too vague, goes in a **separate**
group with the reason. Do not fold it into "all done."

Compare the final `git -c core.quotePath=false status --porcelain --untracked-files=all` and hashes, in each repository the
baseline covered, against the Step 1 baseline and show **only what this skill changed.** New paths
are this skill's. A baseline path whose hash or status changed (for example, modified to deleted), or
that has vanished from the final status, was edited here too; label it "already modified before,
also edited by this skill." Only baseline paths with an unchanged hash are called pre-existing and
unrelated.

**Do not commit or push unless the user asks.**

## Recommending a session model (when delegation is not the answer)

One deep, indivisible task — an architectural decision, a design whose tradeoff is not clear-cut, a
judgment that needs the whole conversation — should not be forced through delegation. Split it up
and each piece works blind to the whole, and stitching them back together costs more than was saved.

For that case, say it in one line: **"This is better done directly on a stronger model than
delegated — consider switching with `/model opus`."** Opus is the default suggestion. Name a tier
above it (today `fable`) only when the task is long-horizon and demanding enough that Opus at higher
effort would likely still fall short.

You cannot change your own model. Recommend, and let the user decide. If they decline, proceed
directly on the current model — never stall the work waiting on the recommendation.

## Checklist

- [ ] There are 3+ tasks and they are independent (otherwise do not use this skill)
- [ ] After grouping by file there are still 2+ groups, or the one group is a file this session has
      not read (otherwise delegation buys nothing)
- [ ] Captured a baseline in every repo touched (`git -c core.quotePath=false status --porcelain --untracked-files=all` plus
      file hashes) before starting
- [ ] Every task is concrete enough for a zero-context agent to execute
- [ ] Read the target project's CLAUDE.md
- [ ] If the target is outside the current project, its CLAUDE.md is in the prompt and paths are absolute
- [ ] Same-file and paired-file tasks are in one call
- [ ] Mixed-grade groups were sent at the highest grade
- [ ] If the Agent tool offers a tier not in the Step 2 table, checked how it is positioned before
      using it, rather than assuming it fits the delegation ladder (and never used it as a retry
      escalation)
- [ ] Did not trust "done" — verified per grade
- [ ] Reported separately whatever was not delegated or was left out
- [ ] Reported only changes made by this skill, against the baseline
- [ ] Did not commit unless the user asked
