---
name: code-review
description: 'Review the changes since a fixed point (commit, branch, tag, or merge-base)
  along two axes: Standards (does the code follow this repo''s documented coding standards?)
  and Spec (does the code match what the originating issue/spec asked for?). Runs
  independent Pi workers for both axes and reports them side by side. Use when the
  user wants to review a branch, a PR, work-in-progress changes, or asks to "review
  since X".'
license: MIT
metadata:
  author: Matt Pocock
  adapted_by: Igor Guariroba
  version: 1.0.0
---


Two-axis review of the diff between `HEAD` and a fixed point the user supplies:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Both axes run in independent **Pi worker** processes so they don't pollute each other's context, then this skill aggregates their findings. Read `references/pi-workers.md` before dispatching; use its background execution, waiting and optional Herdr path.

For remote issues, use `docs/agents/issue-tracker.md` when present, otherwise the relevant installed tracker skill. A local spec needs no tracker setup; ask only for missing information required to fetch the source.

## Process

### 1. Pin the fixed point

Whatever the user said is the fixed point (a commit SHA, branch name, tag, `main`, `HEAD~5`, etc.). If they didn't specify one, ask for it.

Resolve the fixed point and HEAD to commit SHAs (`git rev-parse --verify '<ref>^{commit}'`), then compute their merge-base. Capture `git diff <merge-base-sha> <head-sha>` and `git log <base-sha>..<head-sha> --oneline` into files outside the repo. Workers read the same frozen artifacts, not moving branch names.

For work-in-progress reviews, also capture staged/unstaged changes with `git diff <head-sha>` and list untracked files with `git ls-files --others --exclude-standard`. Supply the relevant untracked files as snapshots. Record which pre-existing changes are outside the requested scope. A committed-only review explicitly excludes local changes.

Confirm refs resolve and at least one in-scope committed or local change exists before dispatch. Keep the reviewed tree stable while workers run; if it changes, refresh the artifacts and repeat affected review work. Record status before/after to detect unintended worker edits.

### 2. Identify the spec source

Look for the originating spec, in this order:

1. Issue references in the commit messages (`#123`, `Closes #45`, GitLab `!67`, etc.), fetched via the workflow in `docs/agents/issue-tracker.md`.
2. A path the user passed as an argument.
3. A spec file under `docs/`, `specs/`, or `.scratch/` matching the branch name or feature.
4. If nothing is found, ask the user where the spec is. If they say there isn't one, the **Spec** worker will skip and report "no spec available".

### 3. Identify the standards sources

Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below: a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### 4. Start independent Pi workers

Create one prompt per axis and launch both using the shared worker guide. Reviewers inspect code without editing, committing or dispatching further reviewers; their final answer goes to stdout for capture. Completion requires a report with findings or an explicit pass, plus coverage and blockers. Process failure, timeout or missing output is an incomplete review, never a pass.

**Standards worker prompt** should include:

- Absolute paths to the frozen diff(s), commit list, any local snapshots, and the repo cwd.
- The list of standards-source files you found in step 3 and an absolute pointer to this skill's section 3, requiring the worker to read its smell baseline and override rules.
- The brief: "Report, per file/hunk where relevant, (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls: documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words."

**Spec worker prompt** should include:

- Absolute paths to the same frozen diff(s), commit list, any local snapshots, and the repo cwd.
- The path or fetched contents of the spec.
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words."

If the user confirms there is no spec, skip the Spec worker and note this in the final report. If workers cannot run independently, perform separate sequential passes and disclose the loss of context isolation.

### 5. Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do **not** merge or rerank findings, because the two axes are deliberately separate (see _Why two axes_).

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Don't pick a single winner across axes: that's the reranking the separation exists to prevent.

## Why two axes

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.
