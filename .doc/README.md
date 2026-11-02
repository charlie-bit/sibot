# sibot documentation

> Project scope is still undecided (TODO). This folder holds design and decision records.

## Repository rules (read first)

| File | Purpose |
|------|---------|
| [`CONTRIBUTING.md`](../CONTRIBUTING.md) | Forbidden content, language policy, branches, commit format, PR flow, AI disclosure |
| [`SECURITY.md`](../SECURITY.md) | Private vulnerability reporting |
| [`LICENSE`](../LICENSE) | MIT |
| [`.github/pull_request_template.md`](../.github/pull_request_template.md) | PR template (applied automatically) |
| [`.gitignore`](../.gitignore) | First line of defence against committing secrets and artifacts |

Short version: the history of a public repository cannot be unpublished, so secrets,
personal data, employer material, and anything belonging to another project are not
welcome at all. Commits are `type(scope): subject` in English, short, one logical change
each; AI assistance is disclosed in the trailer and labelled on the pull request; all code
and comments are English.

## Layout

| File | Content |
|------|---------|
| `README.md` | This index |
| `requirements.md` | Requirements, user stories, scope (TBD) |
| `design.md` | Architecture and key decisions (TBD) |
| `decisions/` | ADRs, one per decision: `NNNN-<slug>.md` (TBD) |
| `local/` | Local scratch notes, gitignored — the only place non-English text is allowed |

## Open questions

- [ ] What problem does sibot solve, and for whom?
- [ ] What are its inputs and outputs (CLI, long-running service, Slack/Jira integration, agent)?
- [ ] Tech stack and deployment shape
