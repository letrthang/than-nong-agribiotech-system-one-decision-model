> [!IMPORTANT]
> **Do NOT auto-commit or auto-push unless the developer explicitly asks the bot to do so.**
> The bot must always ask the developer for confirmation before committing and pushing code.

> [!IMPORTANT]
> **The bot creates a new release only when the developer asks for it.** Merging into `develop` or `main` is not a
> release: no `release` branch change, no tag and no GitHub release unless the developer explicitly asks for one. See
> [Git flow](#git-flow).

> [!WARNING]
> **This repository is PUBLIC.** Every commit, branch, PR and issue can be read by anyone and stays in git history.
> See [What must never be committed](#what-must-never-be-committed).

# GitHub Commit and Push Rules for Bot

When the bot is asked to commit or push changes to this repository, it must follow these rules in order (step 1 to step 7).
The repo follows [git flow](#git-flow): everyday work goes to **`develop`**, not `main` and never `release`.

1. Try to use GitHub MCP tools first to access and operate on the remote GitHub repository.
2. Work in a **git worktree** (or a separate clone), never switch branches in the developer's main checkout. Fetch the latest
   `develop` from the remote. Stop and report to the developer if remote changes conflict with local work in a way the bot
   cannot safely resolve.
3. From `develop`, create a new branch using `feat/*`, `fix/*` or `docs/*`, and commit only on that branch.
4. Create a pull request (PR) to merge the new branch into `develop`.
5. Merge the PR into `develop` (squash). No need to wait for approval from other developers, but stop and report if there are
   conflicts the bot cannot safely resolve.
6. After the merge, delete the feature branch (local and remote).
7. Check the merged `develop` (the docs render, links work; once code exists, it builds and its tests pass). When the check
   passes, create a PR `develop` → `main` and merge it with a **merge commit** — see [Daily work](#daily-work).

## Git flow

Three long-lived branches, in this order: **`develop` → `main` → `release`** — the same flow as the
[RAG demo project](https://github.com/letrthang/spring-ai-with-hybrid-search-and-system-one-typesafe-jev/releases) and
[AIMTAS System One — Laya](https://huggingface.co/spaces/letrthang/aimtas-system-one-laya-slm).

| Branch | What it holds | Gets its changes from |
|---|---|---|
| `develop` | **Daily work.** | PRs from `feat/*`, `fix/*`, `docs/*` (squash) |
| `main` | **Checked work**: what passed the check on `develop`. The default branch people see. | PR `develop` → `main` (merge commit) |
| `release` | **Released versions**, tagged `vX.Y.Z`. Created from `main` at the first release. When the models and the `/v1/systemone` server exist, what they publish (Hugging Face model repos, the Space) comes from here. | PR `main` → `release` (merge commit) — **only when the developer asks for a new release** |
| `feat/*`, `fix/*`, `docs/*` | One task each, from `develop`. Deleted after the merge. | — |

- Never commit directly to `develop`, `main` or `release`: every change arrives by PR.
- Use a **merge commit** (not squash) for `develop` → `main` and `main` → `release`, so the three branches share their history
  and the next PR between them carries only the new commits.
- **Never create `release/*` branches.** Git cannot hold `release` and `release/…` together.

### Daily work

1. Steps 1–7 above: a branch from `develop`, a PR into `develop`, check the merged `develop`, then a PR `develop` → `main`.
2. That is all — no tag, no release until the developer asks for one.

### A new release — only when the developer asks

1. **CHANGELOG:** PR into `develop` that adds a `## [X.Y.Z] — YYYY-MM-DD` section to `CHANGELOG.md` (create the file at the
   first release; move the *[Unreleased]* items into the new section). Versions follow
   [Semantic Versioning](https://semver.org/): a fix → patch, a new feature → minor.
2. **Check:** PR `develop` → `main` (merge commit); check `main` (once code exists: build and tests).
3. **Release:** PR **`main` → `release`** (merge commit). At the first release, create `release` from `main` instead. When
   something is published from `release` (a model, a server image, a Space), check that it is live and works.
4. **Tag + GitHub release** on the head of `release`:
   ```bash
   gh release create vX.Y.Z --target release --title "vX.Y.Z — YYYY.MM.DD" --notes-file notes.md
   ```
   - No assets unless the developer asks for them, not a draft.
   - Notes in the format of the RAG demo's releases: `### Features` / `### Fixes` / `### Documentation` as they apply; each
     bullet starts with a **bold lead sentence**, then why and how in plain words, and a "Verified: …" note where something
     was checked.
5. **Next version:** PR into `develop` opening a new `## [Unreleased]` section in `CHANGELOG.md`, then `develop` → `main`.

### Urgent fix on a release (hotfix) — only when the developer asks

1. `fix/*` from `release`, the fix and its check → PR into `release` (merge commit).
2. Then PR `release` → `main` and `main` → `develop`, so the fix is not lost. A tag and GitHub release only if the developer
   asks for them.

## What must never be committed

This repository is public, and git history is permanent: deleting a file later does not un-publish it.

| Never commit | Why |
|---|---|
| tokens, keys, passwords, `.env` / `.env.*`, `.mcp.json` | anyone could use them; a leaked secret must be **rotated**, not just deleted |
| real farm, patient or animal-owner data; real farm / zone / device IDs; names, phone numbers, addresses | personal and business data — use synthetic or anonymised examples |
| data without the right to publish it | training and eval data keep their own licences (see the data card) |
| model weights, checkpoints, large binaries (`*.gguf`, `*.safetensors`, `*.onnx`, datasets) | models live in the Hugging Face model repos, not in git |
| IDE and OS files (`.idea/`, `.DS_Store`), lockfiles or settings changed as a side effect | noise that hides a real change |

## What the Bot May Stage — NEVER `git add -A`

> [!CAUTION]
> **Stage explicit paths. Never `git add -A`, `git add .` or `git add -u`.** Never `git stash`.
> A bot knows which files it edited; a wildcard stages the ones it did not, and those are exactly the files nobody reviews in the
> diff — in a public repo, they are published the moment they are pushed.

```bash
# WRONG — sweeps whatever else the working tree happens to hold
git add -A && git commit -m "..."

# RIGHT — the files this change actually touched
git add README.md architecture.md
git commit -m "..."
```

### Before every commit

1. `git status --short` — read it. Anything you did not edit does not get staged.
2. Stage by path, then `git diff --cached --stat` and confirm the file list is only your change.
3. Read the staged diff for anything from [the list above](#what-must-never-be-committed).
4. If a secret-bearing file is already tracked, untrack it (`git rm --cached <file>`) and tell the developer the secret must be
   rotated.

## Commit messages and PRs

- Commit subject: `<type>: <what changed>` — `feat`, `fix`, `docs`, `chore`, `refactor`, `test`. The body says why, and how it
  was checked.
- PR description: what, why, how it was checked, and what happens after the merge (e.g. "no release").
- Shell: never put backticks or `$(` inside an unquoted heredoc — use `<<'EOF'` or write the text to a file first.

## Issues

All issues for this project go in **this repo**: <https://github.com/letrthang/than-nong-agribiotech-system-one-slm/issues>.
Title format `[<type>] <short description>` — `[bug]`, `[feat]`, `[docs]`, `[refactor]`, `[release]`,
`[need-admin-approve]`. Apply a label (`bug`, `enhancement`, `documentation`). Issues are public too: never put a secret or
personal data in one.

## Notes

- Do not commit directly to `develop`, `main` or `release`. Feature branches → `develop`, then `develop` → `main` after the
  check, and `main` → `release` only for a new release.
- Create a new release (`release` branch, tag, GitHub release) **only when the developer asks**.
- Fetch and rebase before every push.
- Always delete the feature branch after its PR is merged (never `develop`, `main` or `release`).
- After a merge, pull the branch the PR went into.
