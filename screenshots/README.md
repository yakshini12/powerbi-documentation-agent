# Screenshots

**This folder is intentionally empty.**

The visuals in this write-up are original diagrams in [`../diagrams/`](../diagrams),
drawn specifically for this repository. They contain no employer or client material.

Screenshots of real generated documentation are deliberately **not** included. Even
cropped, a business HTML page, technical PDF, filter table, or lineage export still shows
report names, measure and table names, company-specific business language, and real
layout — which makes it client work product regardless of how tightly it is framed.

## If you want to add images later

The safe route is to generate them from a report you own:

1. Build a throwaway Power BI report with invented data — something obviously synthetic
   like a coffee shop, library, or sample sales dataset.
2. Commit it to a personal repository.
3. Run the documented flow against it.
4. Screenshot those outputs.

That demonstrates exactly the same capability with no question about what's in frame.

Public tooling UI is also fine on its own: an editor chat with placeholder repository and
report values, a tool-server settings panel showing a connected server, a draft pull
request on a personal dummy repository.

## Redaction checklist

Before adding any image, confirm it contains **none** of the following:

- [ ] Employer or client organisation names, logos, or internal branding
- [ ] Internal repository, registry, or organisation names in URLs, paths, or terminal prompts
- [ ] Real report names, measure names, table names, or column names
- [ ] Any actual metric values, even partial, even blurred
- [ ] Employee names, usernames, email addresses, or profile photos
- [ ] Internal chat channels, ticket identifiers, or meeting invites
- [ ] Tokens, credentials, session identifiers, or authentication output
- [ ] Machine hostnames or user directories that identify an employer environment
- [ ] Browser tabs, bookmarks, notifications, or sidebars showing internal tools

Crop to a single panel. Full-window captures almost always include a sidebar someone
forgot about.

**Rule of thumb:** if you can still tell which company or which real report it is, don't
upload it.
