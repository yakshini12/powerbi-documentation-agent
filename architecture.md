# Architecture

My own description of how the documentation agent is put together and why the pieces sit
where they do. No employer source code is reproduced here.

---

## The three repositories problem

The first thing that confused everyone — including, briefly, me — was that this system
lives across three different repositories that play completely different roles. Getting
this separation clear in people's heads was half the support burden.

```
┌───────────────────────────────┐
│  Source repository            │   Python product: extraction, rendering,
│  (engineering-owned)          │   validation, publishing, tool server.
└───────────────┬───────────────┘   Where development and review happen.
                │
                │  vendored + released as a versioned bundle
                ▼
┌───────────────────────────────┐
│  Skill registry repository    │   The distributed artifact. What a developer
│  (private, org-wide)          │   actually installs into their editor.
└───────────────┬───────────────┘
                │
                │  installed once per machine
                ▼
┌───────────────────────────────┐
│  Report repositories          │   Power BI report packages committed to Git.
│  (BI-owned, many of them)     │   The agent runs *here* and opens PRs *here*.
└───────────────────────────────┘
```

The consequence that mattered operationally: **editing the source repository does not
change what anyone's editor is running.** An installed Skill is pinned to a released
bundle. A fix only reaches the team when it is released into the registry and they
update. I had to explain this repeatedly, and it's also what made repository cleanup
safe to do — deleting retired files from the source repository could not break an
already-installed Skill.

## Pipeline stages

### 1. Discovery

Given a repository, a branch and a report name, resolve exactly one report package.
Zero matches or multiple matches is a hard stop, not a guess — silently documenting the
wrong report is worse than failing.

### 2. Deterministic extraction

Parse the report package into a structured evidence model: pages, visuals and their
field bindings, filters (distinguishing report/page/visual scope), slicers and their
applied state, measures, and the dependency and lineage graph.

Two properties I cared about here:

- **Provenance on every fact.** Each extracted item carries where it came from and a
  confidence level. Inferred lineage is labelled inferred; unresolved hops are labelled
  as gaps. The documentation then says so, instead of presenting a guess as a fact.
- **Evidence is the only interface.** Everything downstream — business prose, technical
  docs, PDFs, the Q&A server — reads this model rather than re-parsing report files.
  One extraction, many renderers.

### 3. The bounded reasoning step

This is the part I'd defend most in a design review.

Business prose is the one place a language model genuinely helps: turning structured
facts into "here's what this report is for, here's what this KPI means, here's when to
be careful with it." But it's also where a model will happily invent a plausible
definition for a metric it knows nothing about.

So the exchange is deliberately narrow:

1. A prepare step writes an **explain payload** — a curated selection of evidence — plus
   a **manifest** carrying a hash of that payload.
2. The reasoning step may read **only** those two files. Not the report package, not the
   evidence JSON, not previously generated docs, not the source code, not the Q&A tool
   server. Every statement must cite payload references.
3. Facts absent from the payload stay absent. The instruction is to preserve the
   ambiguity, not resolve it.
4. The result is written back as a JSON object matching a fixed contract.
5. The Python side re-checks the payload hash before using the result, so a response
   authored against a different payload is rejected.

```
evidence ──> [prepare] ──> explain-payload.json + manifest.json (hashed)
                                      │
                                      ▼
                          reasoning step (editor's model)
                             reads ONLY those two files
                                      │
                                      ▼
                            explain-result.json
                                      │
                          hash verified ──> deterministic render
```

Once that JSON exists, **generation involves no model at all**. Same payload plus same
result produces the same documents. That reproducibility is what made the output
trustworthy enough to put in a PR.

### 4. Rendering

From approved evidence and the approved explanation:

- business Markdown and styled HTML;
- technical Markdown, a paginated PDF, and CSV inventories (pages, visuals, filters,
  slicers, measures, dependencies, lineage, impact);
- a self-contained HTML file written beside the report package, for a developer to embed
  on a Documentation page inside the report itself.

That last one is a **handoff artifact**, not an automatic publish. Binding it into the
report is a deliberate manual step, and being precise about that in the docs prevented a
lot of false expectations.

### 5. Validation

Before anything is published: evidence integrity checks, the payload hash check, and a
quality report. A generation failure stops the run before the publish stage — no branch,
no commit, no PR, nothing half-written pushed anywhere.

### 6. Publish

Create a branch, commit the artifacts, open a **draft** PR against the target branch
using the developer's own authenticated CLI session. Their identity, their permissions,
their review process. No service account and no bot with write access to the whole
estate.

## The Q&A tool server

A 3,000-line technical document is a reference, not an answer. So the same evidence model
is exposed to the editor as a read-only tool server over stdio, with narrow tools —
report structure, report/page/visual filters, slicer state, measure dependencies,
measure usage, source dependencies, impact analysis, lineage gaps — so a question like
"what filters are applied on this page" doesn't require dumping an entire lineage graph
into the conversation.

Safety rules I built into how it's used:

- It is **not** used during business explanation. That boundary stays sealed.
- Lineage queries are targeted by default, with bounded depth and edge caps, instead of
  returning the full graph.
- The one mutating operation (extract and cache to disk) **must never be auto-invoked**.
  A human approves it explicitly.
- Unrelated tool servers are never accepted as evidence about a report.

## Trust boundaries, summarised

| Boundary | Rule |
|---|---|
| Business prose | Payload + manifest only. No package files, no evidence JSON, no tool server. |
| Generation | Deterministic. No model involvement after the explanation is approved. |
| Model | The developer's own editor model. No external API, no personal key. |
| Writes to a report repo | Draft PR only, under the developer's own credentials. |
| Cache/extract mutation | Human approval required, never automatic. |
| Publishing to the BI service | Out of scope. Happens through the team's normal sync after merge. |
