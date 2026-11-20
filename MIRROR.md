# Mirror notice

`veilops-protocol/veilops-docs` is a **read-only mirror**. During Phase 01 the single source of truth is [`veilops-protocol/veilops`](https://github.com/veilops-protocol/veilops), path `docs/`.

- Open issues and pull requests on `veilops-protocol/veilops`, not here. Pull requests on this repository are closed with a pointer to the monorepo.
- History here is generated with `git subtree split`, so commits touching `docs/` keep their authors and messages. A final commit overlays this repository's `README.md`, `LICENSE` and `MIRROR.md`.
- The branch `main` is **force-pushed** on every sync. Do not base work on it.

## How the mirror is published

Run from a clean checkout of the monorepo, with the satellite files available at `../veilops-github/veilops-docs/`:

```bash
# 1. Extract the history of docs/ into a local branch (deterministic: same input, same commits)
git subtree split --prefix=docs -b mirror/veilops-docs

# 2. Overlay the mirror's own README, LICENSE and MIRROR.md on top of the split
git worktree add ../_mirror-veilops-docs mirror/veilops-docs
cp ../veilops-github/veilops-docs/README.md ../veilops-github/veilops-docs/LICENSE ../veilops-github/veilops-docs/MIRROR.md ../_mirror-veilops-docs/
git -C ../_mirror-veilops-docs add README.md LICENSE MIRROR.md
git -C ../_mirror-veilops-docs commit -m "mirror: add veilops-docs README, LICENSE and MIRROR notice"

# 3. Publish
git push --force git@github.com:veilops-protocol/veilops-docs.git mirror/veilops-docs:main

# 4. Clean up
git worktree remove ../_mirror-veilops-docs
git branch -D mirror/veilops-docs
```

The mirror is updated manually by a maintainer in Phase 01. Automating it (a workflow in the monorepo with a deploy key scoped to this repository) is planned.
