---
name: review
description: Review the current branch's PR along two axes — Standards (does the code follow this repo's documented coding standards?) and Spec (does the code match what the originating issue/PRD asked for?). Auto-detects the PR via `gh`, runs both reviews in parallel sub-agents, always writes a Mermaid explainer markdown file alongside the review output, and reports the findings. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to "review since X".
---

Two-axis review of the diff for the current branch's PR:

- **Standards** — does the code conform to this repo's documented coding standards?
- **Spec** — does the code faithfully implement the originating issue / PRD / spec?

Both axes run as **parallel sub-agents** so they don't pollute each other's context, then this skill aggregates their findings.

The issue tracker should have been provided to you — run `/setup-matt-pocock-skills` if `docs/agents/issue-tracker.md` is missing.

## Process

### 1. Resolve the PR and pin the fixed point

Detect the PR for the current branch first; only fall back to asking.

1. **`gh` first.** Run `gh pr view --json number,title,baseRefName,url,body`. If it returns a PR, the fixed point is its base branch (`origin/<baseRefName>`). Capture the PR number, title, URL, and body — the body is a spec source (step 2) and feeds the explainer (step 4).
2. **User argument overrides.** If the user explicitly passed a fixed point (commit SHA, branch, tag, `main`, `HEAD~5`, etc.), use that instead.
3. **Fall back to asking.** If `gh` finds no PR and the user gave nothing, ask for the fixed point.

Capture the diff command once: `git diff <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Also note the list of commits via `git log <fixed-point>..HEAD --oneline`.

Before going further, confirm the fixed point resolves (`git rev-parse <fixed-point>`) and the diff is non-empty. A bad ref or empty diff should fail here — not inside two parallel sub-agents.

### 2. Identify the spec source

Look for the originating spec, in this order:

1. The PR body from step 1, plus any issue references it links (`#123`, `Closes #45`, GitLab `!67`, etc.).
2. Issue references in the commit messages — fetch via the workflow in `docs/agents/issue-tracker.md`.
3. A path the user passed as an argument.
4. A PRD/spec file under `docs/`, `specs/`, or `.scratch/` matching the branch name or feature.
5. If nothing is found, ask the user where the spec is. If they say there isn't one, the **Spec** sub-agent will skip and report "no spec available".

### 3. Identify the standards sources

Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below — a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation — and, like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name** — a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code** — the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy** — a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps** — the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession** — a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches** — the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery** — one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change** — one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality** — abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains** — long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man** — a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest** — a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### 4. Spawn the sub-agents in parallel

Send a single message with three `Agent` tool calls. Use the `general-purpose` subagent for all three.

**Standards sub-agent prompt** — include:

- The full diff command and commit list.
- The list of standards-source files you found in step 3, **plus the smell baseline from step 3** pasted in full — the sub-agent has no other access to it.
- The brief: "Report — per file/hunk where relevant — (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls — documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words."

**Spec sub-agent prompt** — include:

- The diff command and commit list.
- The path or fetched contents of the spec.
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words."

If the spec is missing, skip the Spec sub-agent and note this in the final report. The **Explainer** sub-agent always runs.

**Explainer sub-agent prompt** — include:

- The full diff command and commit list, plus the PR number/title/body from step 1.
- The brief: "Write a markdown explainer of what this change does and why, for a reviewer who hasn't seen the code. Include **at least two Mermaid diagrams** — e.g. a `flowchart` of the new/changed control flow and a `sequenceDiagram` of the key interaction, plus an architecture or ER diagram where it helps. Verify every diagram is valid Mermaid (correct fences ```mermaid, balanced brackets, no reserved-word node ids). Sections: Summary, What changed (per area), Diagrams, Risks/edge cases. Return the full markdown document as your output — it will be saved verbatim."

### 5. Write the explainer file

Save the Explainer sub-agent's markdown output verbatim to `/Users/sravanth/Documents/GitHub/Obsidian-Notes/Quaisr/PRs/`, named after the PR number only — e.g. `3695.md`. If that file already exists, overwrite it (this is a fresh review of the same PR — replace the stale content, don't create a suffixed copy). If there's no PR number (fixed point came from a user arg or fallback), use the branch name instead. Report the final path in the summary.

### 6. Aggregate

Present the two review reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do **not** merge or rerank findings — the two axes are deliberately separate (see _Why two axes_).

End with a one-line summary: total findings per axis, the worst issue _within each axis_ (if any), and the path to the Mermaid explainer file. Don't pick a single winner across axes — that's the reranking the separation exists to prevent.

### 7. Append the review to the file

Append the full review to the same `PRs/<number>.md` file written in step 5, after the explainer, under a top-level `# Review` heading. Include the same `## Standards`, `## Spec`, and summary sections you presented in chat, plus a one-line note of the fixed point and diff command so the file is self-contained. This way the file holds both the explainer and the review.

## Why two axes

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.