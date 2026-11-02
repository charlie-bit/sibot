# Contributing to sibot

sibot is an open-source project. Everything committed here is public forever — treat
every commit, file, and message as if it will be read by strangers a decade from now.

This document is the contract for contributions. Read it before your first commit.

---

## 1. Project status

sibot is at day zero: the scope is not fixed yet (see [`.doc/README.md`](.doc/README.md)).
For anything larger than a typo:

1. Open an issue describing the problem first.
2. Wait for agreement on the approach.
3. Then send the pull request.

A large PR with no prior discussion may be closed without review.

---

## 2. What must never be committed

These are hard rules, not preferences. A violation must be fixed by rewriting history —
deleting the file in a later commit is **not** enough, because the content stays in the
public git history.

### Never commit

| Category | Examples |
|---|---|
| **Secrets** | API keys, tokens, PATs, private keys, certificates, passwords, session cookies, `.env` files, database dumps with credentials |
| **Personal data** | Customer or user records, emails, phone numbers, wallet addresses tied to a person, account IDs, IP logs, screenshots containing real user info |
| **Employer / internal material** | Internal hostnames or URLs, internal ticket or repo IDs, unpublished roadmaps, internal Slack threads, non-public vendor contracts |
| **Licensed third-party code** | Copied source that is not license-compatible with this project, or copied without its required attribution |
| **Generated output** | Build artifacts, compiled binaries, `node_modules`/`vendor` trees, coverage reports, local caches, editor state files |
| **Large binaries** | Images, videos, datasets, model weights — host them elsewhere and link them, unless the project explicitly decides otherwise |
| **Anything you cannot explain** | If you do not understand why a line is there, do not commit it |

### Also avoid

- Debug prints, commented-out code, `TODO`s without an issue reference.
- Real product names of competing services in code identifiers — say "competitor A" / "competitor B".
- Placeholder content that looks real (fake credentials, fake customer records, fake receipts) —
  scanners, and readers, cannot tell it apart from the real thing.

### If you leak something

1. **Rotate the secret first.** Assume it is already public.
2. Then remove it from history and force-push the branch.
3. If it reached `main`, notify the maintainers immediately.

---

## 3. Before you commit

```bash
git status              # know exactly what you are staging
git diff                # read your own change first
git diff --cached       # read what will actually be committed
```

Stage paths explicitly. `git add -A` and `git commit -a` are how leaked keys and stray
scratch files get published.

---

## 4. Branches

- `main` is the release branch. Non-trivial work goes through a branch and a pull request.
- Branch name: `<type>/<short-slug>` — e.g. `feat/slack-ingest`, `fix/retry-loop`, `docs/contributing`.
- `<type>` uses the same vocabulary as commit types (§5).
- Keep branches short-lived. Rebase on `main` rather than merging `main` back in.
- External contributors: fork, then PR from the fork.

---

## 5. Commit messages

sibot follows [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).

```
<type>(<scope>): <subject>

<body>

<footers>
```

### Subject line

- Imperative mood, present tense: `add`, not `added` / `adds`.
- Lowercase after the colon, no trailing period.
- ≤ 72 characters. One line, no wrapping.
- English. (Docs inside `.doc/` may be Chinese; commit messages are English.)
- Describe the change, not the process: `fix(retry): cap backoff at 60s`, never `fix bug` / `update` / `wip`.

### Types

| Type | Use for |
|---|---|
| `feat` | A user-visible capability |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `refactor` | Behaviour-preserving restructuring |
| `perf` | Performance work |
| `test` | Tests only |
| `build` | Build system, dependencies, packaging |
| `ci` | CI configuration |
| `chore` | Maintenance that fits nothing above |
| `revert` | Reverting a previous commit |

### Scope

Optional but encouraged: the module, package, or area touched — `feat(api):`, `fix(parser):`.

### Body

- Explain **why**, not what — the diff already shows what.
- Wrap at 72 characters.
- Include the constraint that made the obvious approach wrong, if there was one.

### Footers

- `BREAKING CHANGE: <what breaks and how to migrate>` — for any incompatible change.
- `Refs: #123` / `Closes: #123` — link the issue.
- `Co-Authored-By: <name> <email>` — co-authors, including AI assistance (§6).

### Never

- `wip`, `fix`, `update`, `asdf`, `fix again`, `address comments` as the only content.
- Mixing unrelated changes in one commit. One logical change per commit.
- Commit messages that reference private context the reader cannot see.

---

## 6. AI-assisted contributions

AI-assisted code is welcome. The rules:

1. **Disclose it** in the commit trailer:
   `Co-Authored-By: Claude Code <noreply@anthropic.com>` (or the tool actually used).
2. **You own it.** You are the author of record and are responsible for correctness,
   licensing, and security — "the model wrote it" is not a defence.
3. **Read every line.** Do not commit code you cannot explain or maintain.
4. **Check the license.** Generated code that reproduces licensed source must carry that
   license. When in doubt, rewrite it.
5. **No secrets in prompts, no secrets in output.** See §2.
6. Add the `ai-assisted` label to the pull request.

---

## 7. Pull requests

- Title follows the commit convention: `feat(scope): summary`.
- Body answers: what changed, why, how it was tested, what could break.
- Fill in the PR template; delete sections that do not apply rather than leaving them empty.
- One concern per PR. If the description needs "and also", split it.
- CI must be green. Behaviour changes need tests.
- Self-review the diff before requesting review.
- Do not merge your own PR without review, except for `docs` / `chore` changes by a maintainer.

---

## 8. Code expectations

- Match the surrounding style; do not reformat unrelated code in a feature commit.
- No dead code, no commented-out blocks, no leftover debugging.
- Validate untrusted input at the boundary; never build SQL, shell commands, paths, or HTML
  from unsanitized input.
- No `eval`, unsafe deserialization, disabled TLS verification, or permissive CORS.
- Keep dependencies minimal and explained — every new dependency is a supply-chain decision.

---

## 9. Licensing of contributions

By submitting a pull request you agree that your contribution is licensed under the
project's license (see `LICENSE`) and that you have the right to submit it.

Do not add code you did not write unless its license is compatible and its attribution is
preserved.

---

## 10. Security issues

Do not open a public issue for a vulnerability. Follow [`SECURITY.md`](SECURITY.md).
