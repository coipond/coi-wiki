# coi wiki (source of truth)

This repository holds the documentation that appears on the
**[coipond/coi Wiki](https://github.com/coipond/coi/wiki)**.

Edit the docs **here** — through normal pull requests against `master`. On every
push to `master`, the [`Sync to coi wiki`](.github/workflows/sync-wiki.yml)
workflow publishes the Markdown pages into `coipond/coi`'s Wiki tab.

> The Wiki tab is a **read-only mirror**. Edits made directly on the Wiki tab
> will be overwritten by the next sync — always change pages in this repo.

## How it works

- Each top-level `*.md` file is one wiki page (`Home.md` is the landing page;
  `_Sidebar.md` / `_Footer.md` are the wiki chrome). `README.md` (this file) is
  repo-only and is **not** published.
- The sync mirrors the full page set, so renames and deletions propagate too.

## One-time setup

The sync needs a repository secret named **`WIKI_SYNC_TOKEN`** with write access
to `coipond/coi` (the wiki shares that repo's access — the built-in
`GITHUB_TOKEN` can't reach another repo's wiki):

1. Create a **fine-grained PAT** scoped to `coipond/coi` with
   **Contents: Read and write** (a classic PAT with `repo` scope also works).
2. Add it under **Settings → Secrets and variables → Actions → New repository
   secret**, name `WIKI_SYNC_TOKEN`.
3. Run the workflow once (**Actions → Sync to coi wiki → Run workflow**) to
   confirm it publishes.
