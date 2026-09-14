# Challenges and how the delivery changed

The documentation logic was the part I could work out at my desk. The harder part was
getting it to run somewhere useful and getting other people set up on it. The way it was
delivered changed twice before it settled.

## First attempt: a GitHub Actions workflow

The first version ran as a GitHub Actions workflow in the Actions tab. You supplied the
repository, branch and report name as workflow inputs, the job checked the report
repository out, generated the documentation and opened a pull request.

It worked, but a few things were awkward:

- The feedback loop was slow. If the report name was slightly wrong, you found out after
  the job had run rather than straight away.
- The business wording step needed a model, which meant the workflow needed model
  credentials stored as repository secrets. Nobody wanted to own that.
- The person who asked for the documentation was not involved while it was being written.
  They triggered a job and got a pull request some minutes later.

## Second attempt: CircleCI

The next version moved the same idea to CircleCI, with a pipeline config and a set of
scripts to trigger it, prepare the target repository, run generation and publish the
pull request.

CircleCI gave more control over the pipeline itself, but none of the three problems above
went away. It was still a job that ran away from the developer, it still needed
credentials for the model, and it added its own configuration and scripts to maintain.
The parameters had to be validated before triggering, which was another script, and
another thing to keep in step with the product.

At this point it was clear that the problem was not which CI system we used. The problem
was running it in CI at all.

## Where it ended up: a Cursor Skill

The final version is a Skill invoked in Cursor, in the repository the developer already
has open, using the model they already have configured in their editor.

That removed the credentials question completely. There is no model API key anywhere in
the flow, because the reasoning step runs on the developer's own editor model. It also
put the developer back in the loop at the point where their input is actually useful, and
mistakes like a wrong report name are caught immediately instead of after a pipeline run.

The pipeline's job disappeared with it. Generation, validation and publishing all happen
from the Skill's own bundled runtime.

## Cleaning up the old paths

Once the Skill was the real path, the Actions workflow, the CircleCI config and the CI
scripts were all dead code, along with some committed output folders, sample documents
and old planning notes.

I traced the live path first to confirm nothing in the Skill flow touched any of it, then
removed it in one reviewed pull request, including the tests that only existed to cover
the deleted scripts. Leaving those tests in place would have broken the build and the
cleanup would have been reverted.

The part I got wrong: people had been reporting that CircleCI was still being triggered
during Skill runs, which did not match anything in the code. After the cleanup pull
request, a failing check on my own PR explained it. The message was that no configuration
was found in the project. The CircleCI webhook was still installed on the repository, so
removing the config had stopped the pipeline from doing any work but the integration was
still firing on every push and reporting a failure. Deleting the webhook is what actually
finished the job.

Removing an integration means removing the connection, not just the configuration file
it reads.

## Setup problems on other people's machines

I wrote the setup guide and ran the onboarding session for the team, then spent a while
debugging what happened on real laptops. Almost none of it was the product.

**Homebrew installed but the command was not found.** On macOS the installer finishes and
prints the two lines you need to add it to your shell, and those lines scroll past. The
fix is appending the shell environment line to the shell profile and running it in the
current session, and the path is different on Apple Silicon and Intel. Both versions went
into the guide word for word, because telling someone to add it to their PATH is not
something they can act on.

**A 502 from the install script.** The Homebrew install script returned a 502 from its
CDN once during setup. It looked like a blocked corporate network and it was actually
worth retrying. Worth naming in a guide, because otherwise the next step is raising an IT
ticket.

**Tools not on PATH until the shell restarts.** Especially on Windows, a tool you just
installed is not visible in the terminal that installed it. Every install step in the
guide ends with a version check and a note to reopen the terminal, because a version
number printing is the only proof the step worked.

**Prompts for an admin password.** Some installs ask for a credential that is an IT
credential rather than a developer login. I said so in the guide and pointed at who to
contact, so people stopped guessing.

**More than one GitHub account signed in.** With two accounts authenticated on the same
machine, commands can run as the wrong one. A private repository then reports itself as
not found, which reads like a missing repository rather than a permissions problem. I
added a step to check which account is active and switch if needed. I hit this myself
later when a repository looked like it did not exist.

## MCP setup problems

Connecting the MCP server to Cursor had its own set of issues, and they were all
configuration rather than logic.

**The Python path is different per operating system.** A teammate's server would not
start and the error said the system could not find the path specified. The config had
been copied from a macOS example, so it pointed at a Unix style Python path. On Windows
the virtual environment keeps its interpreter in a different folder with a .exe
extension. Both versions are in the guide now.

**Workspace relative paths do not travel.** A config with a workspace relative Python
path works in the repository it was written for and fails everywhere else. For a setup
that works in any repository, the config belongs at user level with absolute paths. I
added commands that print the exact absolute paths so people can paste them rather than
work them out.

**Nothing reloads on its own.** Changing the MCP config needs a full restart of Cursor,
not a window reload. Obvious once you know, and a dead end if you do not.

**The bundled runtime was trimmed too far.** The Skill's runtime carries what generation
needs. I tested the released bundle rather than assuming it worked, and found a packaging
gap that stopped the MCP server from starting properly, so the documented path gives the
MCP server its own environment while generation stays on the Skill. If I had written the
neater instruction without testing it, everyone following the guide would have ended up
with a server that would not connect.

## Paginated reports

One thing I had to be straight with people about. The documentation is detailed for
interactive Power BI reports because those files carry a lot of structure to read: pages,
visuals, bindings, filters, slicers, measures and lineage. Paginated reports hold much
less of that, so the documentation generated for them is thinner.

That is not something I could fix by writing better prompts, because the information is
not in the files. I explained the reason rather than leaving people to read the
difference as inconsistent quality, and that landed fine.
