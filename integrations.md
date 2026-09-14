# Integration work

The extraction and rendering logic was the part I could reason about at a desk. The work
that actually consumed my time was integration: how the thing gets distributed, how it
authenticates, how it behaves on someone else's laptop, and how to retire the pipeline it
replaced. These are the changes I went through, in roughly the order they happened.

---

## 1. From CI-triggered pipeline to editor-native Skill

**Where it started.** Documentation generation ran as a CI pipeline. A developer supplied
parameters, the pipeline materialised the target repository, ran generation, and opened a
PR. It worked, but the model was wrong for the job:

- The feedback loop was a pipeline round-trip. Getting a report name slightly wrong meant
  waiting on a build to find out.
- The business explanation step wanted a language model, which meant CI needed model
  credentials — a configuration and data-handling problem nobody wanted to own.
- The developer was never in the loop at the moment it mattered. They kicked off a job
  and received a PR.

**Where it landed.** The product became a Skill invoked directly in the editor, in the
repository the developer already has open, using the model they already have configured.
The pipeline's role disappeared entirely.

**What this actually bought:** the reasoning step runs on the developer's existing editor
model, so there is no external model API and no key to provision. That single change is
what made the design approvable, and it's the thing I'd lead with if I were pitching the
architecture again.

## 2. Packaging the runtime so setup is one command

An editor Skill is normally instructions. This one needed to run a real Python product,
and "clone the source repository, create a virtual environment, install requirements"
is a setup story that loses people immediately.

So the Skill ships the runtime. A launcher script resolves a suitable interpreter,
creates a private virtual environment inside the Skill directory on first use, installs
a pinned dependency set, and hashes the requirements file so the environment rebuilds
automatically when dependencies change and not otherwise.

The result: install the Skill, invoke it, and the first run prepares its own environment.
No clone, no manual environment, nothing to keep in sync by hand.

## 3. Private registry distribution, and the scoping question

The Skill is distributed from a private organisation registry, so access is
organisation membership plus SSO authorisation, not a public download.

The question I got asked most — by two managers independently — was: *if this is
installed per repository, do we reinstall it for every report repo?* The answer is that
a global install is per **machine**, not per repository, so one install covers every
local repository. Scoping it deliberately also mattered: install **only** this Skill and
**only** for the one editor, rather than pulling the whole registry, so nobody ends up
with unrelated tooling they didn't ask for.

Two operational consequences I had to keep explaining:

- An installed Skill is pinned to a **released bundle**. Changes to the source repository
  don't reach anyone until a release, and a teammate's install is unaffected by source
  repository edits — including deletions.
- Updating is a distinct, explicit action, and the update command for a globally installed
  Skill is not the same as the project-scoped one. People ran the project-scoped version,
  got "nothing to update," and concluded updates were broken.

## 4. Cross-platform onboarding, debugged on real machines

I wrote the setup guide, ran the onboarding session, and then debugged what actually
happened on other people's laptops. Almost none of it was the product. All of it was
environment.

**A package manager that installed but "didn't exist."** On macOS, the package manager
installed successfully and then wasn't found. The installer had printed the two lines
needed to add it to the shell environment, and those lines had scrolled past. The fix
is appending the shell environment line to the shell profile and evaluating it in the
current session — and the path differs between Apple Silicon and Intel. I put both
variants in the guide verbatim, because "add it to your PATH" is not an instruction
someone can act on at 9am before a demo.

**A transient download failure read as a broken machine.** The package manager install
script returned a `502` from its CDN. It looked like a blocked corporate network and was
in fact a retry-and-it-works. Worth naming explicitly in a setup guide, because the
natural next step otherwise is filing an IT ticket.

**Tools installed but not on `PATH` until the shell restarts.** On Windows especially,
newly installed tools aren't visible in the session that installed them. Every install
step in my guide ends with a version check and an instruction to reopen the terminal,
because a version number printing is the only reliable proof a step worked.

**Admin-credential prompts.** Some installs prompt for credentials that are an IT
credential, not a developer login. I documented that explicitly and named who to contact,
so people stopped burning time guessing at passwords.

**Multiple signed-in accounts.** More than one authenticated account on one machine means
commands can silently run as the wrong identity — including a private repository
returning "not found" when the account simply lacks access. I added an explicit "check
which account is active, switch if needed" step, and later hit exactly this myself:
tooling reported a repository as nonexistent purely because the active account wasn't
the one with access.

## 5. Editor tool-server (MCP) integration

Connecting the Q&A server to the editor was its own integration, and the failure modes
were all configuration rather than logic.

**The OS-specific interpreter path.** A teammate's server wouldn't start: *the system
cannot find the path specified*. The configuration had been copied from a macOS example,
so it pointed at a Unix-style interpreter path that does not exist on Windows, where the
virtual environment puts its interpreter in a different subdirectory with a `.exe`
extension. Both variants now appear explicitly in the guide.

**User-level versus project-level configuration.** A tool-server config with a
workspace-relative interpreter path works in the repository it was written for and breaks
everywhere else. For a "works in any repository" setup, the configuration belongs at the
user level with **absolute** paths — and I added commands that print the exact absolute
paths to paste in, rather than asking people to construct them.

**The catalogue doesn't hot-reload.** Configuration changes need a full editor restart,
not a window reload. Obvious once you know; a dead end if you don't.

**A trimmed runtime that was too trimmed.** The distributed Skill's runtime carries only
what generation needs. The tool server needs an additional dependency that isn't in that
set, so the documented, working path for Q&A is a source-repository environment while
generation stays on the Skill. I verified this rather than assuming it — checked the
released bundle's dependency list, ran its launcher, and confirmed the server falls back
instead of starting a real editor tool server. It's the clearest "split the runtime into
two variants earlier" lesson I took from the project.

## 6. Retiring the old CI pipeline properly

Cleanup was the last thing I did, and it taught me something about decommissioning.

I audited the repository and separated what the live path touches from what it doesn't.
Retired: the CI pipeline config, its trigger/publish/materialise scripts, an older
workflow from an earlier iteration, the integration contract docs describing the retired
flow, generated output and evaluation folders committed during development, sample
documents at the repository root, stale planning notes, and the tests that only existed
to cover the deleted scripts. Kept: the product package, its dependencies, the policy
check and its enforcement workflow, the bundle build script, the tool-server config, and
the architecture and contract documentation.

Sequencing that mattered: confirm the live path first, delete second. I traced the actual
Skill invocation end to end and verified nothing on it referenced the CI scripts before
removing them. Then I removed the tests that covered only deleted code — leaving them
would have broken the build, and "the cleanup PR broke CI" is how cleanup PRs get
reverted.

**The part I got wrong first.** People had reported the CI system still triggering during
Skill runs, which made no sense given the Skill never calls it. After the cleanup PR, the
answer showed up as a failing check on my own PR: *no configuration found in your
project.* The integration webhook was still installed on the repository. Deleting the
config stopped the pipeline from **doing** anything, but the webhook kept firing on every
push and reporting a failure. Removing the webhook is what actually decommissioned it.

Decommissioning an integration means removing the *connection*, not just the
configuration it reads. I'd check for the webhook first next time.

## 7. Documentation as the deliverable

Two audiences, and I underestimated the second one at first.

The product's output is documentation, but the *project's* output included the setup
guide, and that guide was what determined whether anyone could use any of this. The
version I ended on separates the two working paths (global Skill install for generation,
source environment for Q&A), gives complete command sequences per operating system
instead of prose, ends every install step with a verification command, and has a
troubleshooting table keyed on the **exact error text** people see — because that's what
someone actually searches for when they're stuck.

The same applied to being honest about capability. The agent produces genuinely rich
documentation for interactive report packages, because those carry structured metadata:
pages, visuals, bindings, filters, slicers, measures, lineage. Paginated reports are
thinner by nature — the extractable narrative is much smaller, so the documentation is
correspondingly thinner. I explained that difference plainly to stakeholders rather than
letting them read uneven output as inconsistent quality. Setting that expectation
directly was more useful than any hedging.
