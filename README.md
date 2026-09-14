# Power BI Documentation Agent — Project Showcase

A write-up of a system I designed and shipped that turns Power BI reports stored in Git
into reviewable business and technical documentation, delivered as a pull request.

This repository contains **no employer source code and no client data**. It is my own
description of the problem, the architecture I arrived at, the integrations I worked
through, and what I learned. Code samples, where they appear, are illustrative snippets
I wrote for this write-up.

---

## The problem

A BI team maintained a large estate of Power BI reports in Git. Those reports were the
organisation's source of truth for its core operational and financial reporting, but the
documentation around them was not:

- **Business users** opened a report and could not tell what a KPI meant, when to trust
  it, or which page answered their question.
- **Engineers** inheriting a report had no map of its filters, slicers, measures, or
  upstream data lineage, so every change started with an archaeology exercise.
- **Documentation went stale immediately.** Anything hand-written drifted from the report
  within a sprint or two, so people stopped trusting it, which made them stop writing it.

The team did not need "more docs." They needed documentation that regenerated from the
report itself and arrived through the same review process as code.

## What I built

A documentation agent that reads a report package already committed to Git, extracts
structured evidence from it, produces two audiences' worth of documentation, and opens a
**draft pull request** for a human to review.

| Output | Audience | Purpose |
|---|---|---|
| Business Markdown + HTML | Analysts, stakeholders | What the report is for, what each KPI means, when to use it |
| Technical Markdown + PDF + CSVs | Engineers | Pages, visuals, filters, slicers, measures, dependency and lineage maps |
| In-report HTML handoff file | Report developers | A self-contained file to embed on a Documentation page inside the report |

Two design decisions mattered more than anything else:

**1. Evidence first, prose second.** The pipeline never asks a language model "what does
this report do?" against raw files. A deterministic extraction step produces a structured
evidence payload from the report package. Only that payload is handed to the reasoning
step. If a fact is not in the payload, the model is instructed to leave it out rather
than fill the gap.

**2. Nothing is published without a human.** The agent's terminal action is a *draft* PR.
It does not merge, it does not push to the BI service, and the mutating cache/extract
operation cannot be invoked automatically — a person has to approve it.

## How it runs

The developer stays in their editor, in the report repository, and invokes the agent as
an editor Skill:

```text
/powerbi-documentation

Repository: owner/report-repository
Branch:     main
Report:     <exact report name>
```

Everything below happens behind that one invocation:

![End-to-end flow from Skill invocation to draft pull request](diagrams/01-end-to-end-flow.svg)

The reasoning step runs on the model the developer already has configured in their
editor. There is no external model API and no personal API key anywhere in the flow —
which is what made the whole thing approvable from a data-handling standpoint.

## The information boundary

The single most important design decision: decide what the model may *not* see, and make
everything else deterministic.

![What the reasoning step may and may not read](diagrams/04-information-boundary.svg)

Once the explanation JSON exists and its payload hash verifies, generation involves no
model at all. The same inputs always produce the same documents.

## How it's distributed

Three repositories, three completely different jobs. Getting this clear in people's heads
was half the support burden — and it's also why cleaning up the source repository was
safe to do.

![The source repository, the Skill registry, and the report repositories](diagrams/02-distribution-model.svg)

## Technical Q&A over MCP

A thousand-line technical document is a reference, not an answer. So the same evidence is
exposed to the editor as narrow, read-only tools an engineer can simply ask — "which
filters apply on this page", "is this measure used anywhere", "what breaks if I change
this table".

**[Read the full write-up in `mcp.md`](mcp.md)** — the tool design, why the tools are
narrow rather than one catch-all, the safety rules including why the one mutating tool
can never be auto-invoked, and the cross-platform configuration problems I debugged.

## Architecture

See [`architecture.md`](architecture.md) for the component breakdown, the pipeline
stages, and the full trust-boundary table.

## Integrations I worked through

See [`integrations.md`](integrations.md). Short version: this project's hardest problems
were not the extraction logic. They were distribution, authentication, cross-platform
onboarding, and retiring a CI pipeline that everyone assumed was load-bearing.

## What I owned

- **The delivery model.** Moved the product from a CI-triggered pipeline to an
  editor-native Skill that developers invoke directly, and packaged the runtime so a
  single install command is the entire setup.
- **The reasoning boundary.** Specified the hard information boundary for the business
  explanation step (read only the payload and manifest; classify every statement as
  fact, inference, or unknown; never invent) and the fact that generation is
  deterministic once the explanation is approved.
- **The technical documentation surface.** Pages, visuals, filters, slicers, measure
  dependencies, usage, impact analysis, and lineage — including being explicit about
  confidence and about gaps, rather than presenting inferred lineage as confirmed.
- **The Q&A integration (MCP).** Designed and shipped the read-only tool server that
  exposes the evidence model to the editor, so engineers ask targeted questions instead
  of reading a 3,000-line technical document — including the tool decomposition, the
  bounded-lineage defaults, the human-approval gate on the one mutating operation, and
  the cross-platform configuration. Written up in [`mcp.md`](mcp.md).
- **Team enablement.** Wrote the cross-platform setup guide, ran the onboarding session,
  and debugged the first wave of real installs on both macOS and Windows.
- **Repository hygiene.** Audited the repository, identified the retired CI pipeline,
  dead scripts, generated output folders and stale planning notes, and removed them in a
  reviewed PR after confirming nothing on the live path touched them.

## What I'd do differently

- **Bundle the Q&A server dependency from the start.** The distributed Skill shipped a
  trimmed runtime that was perfect for generation but missing one dependency the tool
  server needed, so Q&A had to be set up from the source checkout. Splitting "generation
  runtime" and "full runtime" earlier would have avoided a two-path setup story.
- **Treat the CI webhook as part of the migration.** Deleting the pipeline config stopped
  the pipeline from doing work but left the integration still firing and posting a failed
  check. Decommissioning means removing the hook, not just the config.
- **Write the onboarding guide against a clean machine sooner.** Almost every real
  onboarding failure was environment-level — a missing package manager, a shell that
  hadn't picked up a new `PATH`, an OS-specific interpreter path, the wrong signed-in
  account. Those are the steps worth over-documenting.

## Results

- Documentation for a report went from a multi-hour manual write-up to a single
  invocation producing a reviewable draft PR.
- Business and technical readers each got a document written for them, from the same
  extracted evidence.
- Onboarding for a new team member went from "clone this, build that, configure these"
  to one install command, with an optional second setup for Q&A.
- The legacy CI pipeline, its scripts, its tests and its webhook were fully retired, so
  the live path is the one developers actually use.

---

## A note on scope

Everything here is my own description of work I did. This repository deliberately
excludes employer source code, report packages, extracted evidence, generated client
documentation, internal repository and registry names, and any credentials or internal
contact details. Screenshots, if present, are either of publicly available tooling UI or
of sample data I created for this write-up.
