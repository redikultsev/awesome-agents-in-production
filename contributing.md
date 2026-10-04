# Contribution Guidelines

Thanks for helping. This list grows from incidents and controls that someone can check, so every entry has to be traceable to where it came from.

## What fits

- An **incident**: a public, dated case where an agent or an LLM system in production failed, or where the way it was checked turned out to be broken.
- A **control**: a mechanism that still holds when the model is wrong. Papers, specs, advisories, open-source tools and engineering write-ups all count.
- Not in scope: model releases, prompt collections, frameworks listed for their own sake, and pages that sell a product without describing how it works.

## How to add an entry

- Link the primary source: the company's own postmortem, a security advisory, a paper, a court decision or official docs. Use press coverage only when nothing primary is reachable, and say so in the pull request.
- Incidents start with the date of the event itself, not of the article: `- [Name](link) - YYYY-MM. What happened.`
- Controls: `- [Name](link) - How it works.`
- One or two sentences, plain and concrete, starting with a capital letter and ending with a period. Numbers only if they are in the source.
- Put the entry in the section where someone would look for it, at the end of the Incidents or Controls list.
- One pull request per entry, with a title like `Add [Name]`.

## Before you open the pull request

- Run `npx awesome-lint` and fix what it reports.
- Check that the link opens and that the source says what your line says.

If you're not sure whether something fits, open an issue first.
