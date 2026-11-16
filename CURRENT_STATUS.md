# Current status

## In progress

Both tasks run at the same time. Both use `2026-11-16_glm_design-review.md` as a reference, re-check its points skeptically against the current codebase, and are not limited to the contents or the scope of that note.

- Ruby refactoring — owner: Muse — worktree: `/Users/yuuki/github-wiki-to-html_muse` [muse] — refactoring the Ruby code, worked out with Yuuki in chat as it goes; the details are not fixed here. Muse is the only agent changing the main repository in this round, so there is no integrator and no merge step. No report on this board: the result is read from the main repository's HEAD, and any notes belong in the commit message.
- Astro/TS review — owner: Inkling — worktree: `/Users/yuuki/github-wiki-to-html_inkling` [inkling] — review. The review results must be added to this board as a new note file, following the file name rules in `AGENTS.md`. Changes inside its own worktree for the purposes of the review are fine; nothing is merged from that worktree.

## Recently finished

- GLM's design review note committed and pushed (`2026-11-16_glm_design-review.md`).
- Collaboration setup: Muse's brief passed to Inkling; Inkling's reply written and pushed; Inkling collaboration worktree created at Yuuki's direction.
- All of `translations/` tracked in `git` as of `d0fd3c5`; fresh worktrees now build byte-identically.
- `main` advanced to `a6ad5d4` (package metadata / ESM); `inkling` and `Muse` synced accordingly.
