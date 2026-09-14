# Screenshots

Drop images in this folder and reference them from the main write-up, e.g.:

```markdown
![Skill invocation in the editor](screenshots/01-skill-invocation.png)
```

## What's worth capturing

Each of these can be produced without exposing anything confidential:

| Suggested file | What to show |
|---|---|
| `01-skill-invocation.png` | The editor chat invoking the Skill with placeholder repository/branch/report values |
| `02-skill-installed.png` | Terminal output of the global Skill list, showing the Skill installed |
| `03-mcp-connected.png` | The editor's tool-server settings panel showing the server connected |
| `04-business-doc.png` | A rendered business document — **use the sample report below, not a client report** |
| `05-technical-doc.png` | A page of technical output: pages/visuals/filters table, or a lineage section |
| `06-draft-pr.png` | A draft PR showing the generated artifacts as changed files |
| `07-architecture.png` | Your own diagram of the flow, if you'd rather show it as an image than ASCII |

## Redaction checklist

Before adding any image, confirm it contains **none** of the following:

- [ ] Employer or client organisation names, logos, or internal branding
- [ ] Internal repository, registry, or organisation names in URLs, paths, or terminal prompts
- [ ] Real report names, measure names, table names, or column names from client data
- [ ] Any actual metric values, even partial, even blurred
- [ ] Employee names, usernames, email addresses, or profile photos
- [ ] Internal chat channels, ticket IDs, or meeting invites
- [ ] Tokens, credentials, session identifiers, or authentication output
- [ ] Machine hostnames or user directories that identify an employer environment
- [ ] Browser tabs, bookmarks, notifications, or sidebars showing internal tools

Crop tightly. A screenshot of one panel is safer than a full-window capture, and full
screen captures tend to include a sidebar someone forgot about.

## Producing safe screenshots

The clean way to get demonstrative images without touching anything confidential is to
build a small fake report of your own:

1. Create a throwaway Power BI report with invented pages, measures and dimensions —
   something obviously synthetic like a coffee shop or library dataset.
2. Commit it to a personal repository.
3. Run the documented flow against it.
4. Screenshot those outputs.

The result demonstrates exactly the same capability, and there is no question about what
is in the frame.

If you'd rather not reproduce the environment at all, the architecture diagram and the
write-up stand on their own — a portfolio that explains a system clearly is more
convincing than one with screenshots of unclear provenance.
