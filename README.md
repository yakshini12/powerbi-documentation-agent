# Power BI Documentation Agent

I built this at ABC Fitness, working as a Software Engineer (AI) on the reporting team.

It takes a Power BI report that is already committed to Git, pulls the structure out of
the report files, writes business and technical documentation from what it found, and
opens a draft pull request so someone can review it before anything is merged.

## The problem

The team kept a lot of Power BI reports in Git. The reports were maintained. The
documentation around them was not.

For business users, a report would show a set of KPIs and pages with no explanation of
what a given measure actually meant or which page answered their question. For
engineers, picking up a report someone else built meant working out for yourself which
filters were applied, what the slicers were set to, which measures fed which visuals,
and where the data came from. None of that was written down anywhere.

The other half of the problem was that anything written by hand went out of date as soon
as the report changed, so people stopped trusting it. I wanted documentation that came
out of the report itself and arrived through the normal pull request review, the same way
code does.

## What it produces

| Output | Who it is for |
|---|---|
| Business Markdown and HTML | Analysts and stakeholders. What the report covers, what the KPIs mean, when to use it |
| Technical Markdown, PDF and CSV files | Engineers. Pages, visuals, filters, slicers, measures, dependencies and lineage |
| A standalone HTML file next to the report package | Report developers, to embed on a documentation page inside the report itself |

## How it works

The developer works in the report repository and runs the Skill in Cursor:

```text
/powerbi-documentation

Repository: owner/report-repository
Branch:     main
Report:     <exact report name>
```

Everything after that happens in stages.

![The stages behind one invocation, from discovery to draft pull request](diagrams/flow.svg)

The part I spent the most time getting right is the split between facts and wording.

A script parses the report package and builds a structured record of what is in it:
pages, visuals and their field bindings, filters at report, page and visual level,
slicers and their state, measures, and a dependency and lineage graph. Each item carries
where it came from and whether it is confirmed, inferred or unknown.

Only a prepared slice of that record goes to the model, along with a manifest holding a
hash of it. The model writes the business wording from those two files and nothing else.
If something is not in the payload, it stays out rather than being guessed at. The reply
comes back as JSON in a fixed shape, and the hash is checked again before it is used, so
a reply written against a different report will not be accepted.

Once that JSON exists, the rest of the run does not use a model at all. The same inputs
produce the same documents.

![What the model step can and cannot read](diagrams/boundary.svg)

The last step opens a draft pull request using the developer's own GitHub login. It does
not merge anything and it does not publish to the Power BI service. Keeping the output as
a draft mattered for getting people to try it, because reviewing a draft PR is a normal
thing to do, while giving a tool write access across the reporting repositories is not.

## Technical questions over MCP

The technical document for a large report runs to a few thousand lines. That is useful as
a reference but it is not a good way to answer a single question like which filters apply
on a page, or whether a measure is used anywhere.

So the same extracted record is exposed to the editor as an MCP server with a set of
tools that only read. The developer asks the question in chat instead of searching the
document.

![The MCP server, its tool groups and the rules around them](diagrams/mcp_server.svg)

Details are in [mcp.md](mcp.md), including why the tools are split up by question
instead of being one general purpose tool, and the approval rule on the one tool that
writes anything.

## How it is delivered

The product lives in one repository, ships as a versioned bundle in a private Skill
registry, and runs inside the many repositories that hold the reports.

![The source repository, the Skill registry and the report repositories](diagrams/distribution.svg)

The thing that caused the most confusion was that changing the source repository does not
change what anyone's editor is running, because an installed Skill stays on the released
bundle until the person updates it. A global install covers every repository on that
machine, so it is not something you repeat per project.

To keep setup to one command, the Skill carries its own Python runtime. A launcher script
finds a suitable Python, builds a private environment the first time it runs, and hashes
the requirements file so the environment is rebuilt when dependencies change and left
alone when they have not.

## Challenges

The delivery model changed twice before it worked. It started as a GitHub Actions
workflow, moved to CircleCI, and ended up as a Cursor Skill. Most of the real work was
in that journey and in the setup problems that showed up on other people's machines.

That is written up in [challenges.md](challenges.md).

## What I would do differently

- Ship two runtime variants from the start, one for generating documentation and one
  with everything the MCP server needs, instead of one trimmed runtime.
- Treat the CI webhook as part of the migration rather than something to clean up later.
  Removing a pipeline config stops the pipeline from doing work, but the integration
  keeps firing until the webhook itself is removed.
- Add a check that runs the published bundle against a sample report at release time, so
  a packaging problem shows up then rather than during someone's setup.
- Write the setup guide against a clean machine earlier. Nearly every problem people hit
  was environment setup, not the tool.

## More detail

| Document | Contents |
|---|---|
| [architecture.md](architecture.md) | How the pieces fit together and what each stage does |
| [mcp.md](mcp.md) | The MCP server, its tools and the rules around them |
| [challenges.md](challenges.md) | The delivery journey and the problems I ran into |

## Note on scope

This is my own write up of work I did. It does not contain company source code, report
files, extracted data or generated client documentation. The diagrams are my own, drawn
for this repository.
