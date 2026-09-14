# What I took away from this project

Personal reflection, written for my own future reference and for anyone curious about how
I work.

---

## Designing around what a language model should *not* do

The instinct with a documentation problem is to point a model at the files and ask for
docs. I'm glad I didn't, because the failure mode is invisible: a model handed a report
package will produce confident, fluent, plausible definitions for metrics it has no
information about, and a reviewer cannot tell those apart from the correct ones.

What worked was inverting it — decide what the model is *not* allowed to see, and make
everything else deterministic. The model gets a curated payload and nothing else. Prose
is its job; facts are extraction's job. Generation after the prose is approved involves
no model at all, so the same inputs always produce the same documents.

The payload hash check is the detail I'm most pleased with. It's a few lines of work and
it makes an entire class of mistake structurally impossible: an explanation authored
against a different report simply cannot be used.

I'd carry this into any model-assisted feature now. The interesting design question isn't
"what can the model do here," it's "what must remain deterministic, and how do I make the
boundary enforceable rather than aspirational."

## Human-in-the-loop as a design constraint, not a disclaimer

The agent's last step is a draft PR, under the developer's own credentials, in their own
review process. It doesn't merge, it doesn't publish to the BI service, and the one
operation that writes to disk can't be triggered automatically.

That wasn't caution for its own sake. It's what made the tool adoptable. An agent that
opens a reviewable draft is something a team will try on a real report; an agent with
write access to the reporting estate is something a team will block. Constraining the
blast radius was a feature, and framing it that way in conversations went much better
than framing it as a limitation.

## Distribution is the product

I spent a long time on extraction quality. Then I watched onboarding sessions and learned
that the thing standing between my work and its users was a package manager that hadn't
been added to a shell profile.

Bundling the runtime into the Skill so a single install command does everything was worth
more than any individual feature I shipped. Same for scoping the install to one Skill and
one editor rather than a whole registry. A tool with an eight-step setup gets tried once;
a tool with a one-step setup gets used.

I also learned to be precise about **where state lives**. Most of the confusion I fielded
came from one misconception: that editing the source repository changes what a teammate's
editor runs. It doesn't — installs are pinned to released bundles. Once I could explain
that crisply, cleanup stopped being scary and update questions stopped recurring.

## Verify against the artifact people actually run

Twice, checking the real artifact instead of the source of truth I assumed changed my
answer.

The Q&A runtime is the clearest case. I could have documented the elegant "the Skill
already ships the server, just point the editor at it" setup. Instead I read the released
bundle's dependency list and ran its launcher, and found the dependency the server needs
isn't in the trimmed runtime — the server falls back rather than starting properly. The
elegant instruction would have produced a red connection indicator for every person who
followed it, and I'd have been debugging it remotely after leaving.

The CI webhook was the same shape. The repository said the pipeline was gone. The
repository was right and the integration was still live, because the connection existed
outside the files I was reading.

Read the artifact, not the description of the artifact. Especially when writing
instructions other people will follow without you in the room.

## Writing instructions for someone who can't ask you a follow-up question

The setup guide went through several rewrites, and each one was driven by watching
someone get stuck.

What I ended up believing:

- **Complete command sequences beat prose.** "Add it to your PATH" is not actionable.
  The two exact lines are.
- **Every install step needs a verification command.** A printed version number is the
  only trustworthy evidence a step worked.
- **Platform differences go inline, not in a footnote.** The Windows interpreter path
  problem happened because someone copied a macOS example. Both belong side by side.
- **Troubleshooting tables should be keyed on exact error text.** People search for the
  string they see, not for a description of their situation.
- **Name the transient failures.** A `502` from a CDN looks identical to a blocked
  corporate network, and one deserves a retry while the other deserves a ticket.

The guide became the highest-leverage artifact of the project. It's also what I'd want
someone to inherit.

## Decommissioning is its own skill

I'd have said removing a retired pipeline meant deleting its config and scripts. It
doesn't. Deleting the config stopped the pipeline from doing work, but its webhook kept
firing on every push and posting a failing check — which was the source of a
"the old pipeline still triggers" report I couldn't explain for a while.

The sequence I'd follow next time: trace the live path and confirm what it touches;
remove code and the tests that covered only that code, together, so the build stays
green; then remove the *connection* — webhooks, integrations, credentials, scheduled
triggers — and verify by pushing and watching what fires.

Also: put cleanup in its own reviewed PR against the current main branch, not mixed into
a feature branch. It was a 90-file deletion. It deserved to be reviewable as exactly that
and nothing else.

## Saying the honest thing about quality

The agent produces rich documentation for interactive report packages and noticeably
thinner documentation for paginated ones. That's not a bug — the metadata simply isn't
there to extract.

My instinct was to hedge. What worked better was explaining the cause plainly: here's
what we extract from each format, here's why one yields more narrative, here's what you
can expect. Stakeholders were fine with a real limitation clearly explained. They would
not have been fine with discovering it themselves after being told the output was uniform.

## Where I'd go next

If I picked this up again:

- Split the runtime into a generation variant and a full variant, so Q&A doesn't need a
  separate setup path.
- Strengthen paginated report extraction, since the documentation is thin because the
  extraction is thin, and that's addressable.
- Add a smoke test that runs the released bundle end to end on a sample report, so a
  trimmed-dependency problem surfaces at release time rather than during someone's
  onboarding.
