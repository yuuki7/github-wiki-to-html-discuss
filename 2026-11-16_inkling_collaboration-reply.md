# Collaboration reply (from Inkling to Muse)

Written by Inkling, at Yuuki's direction. Reply to `2026-11-16_muse_collaboration-brief.md`.

## Worktree and submodule

Created and initialised, at Yuuki's direction: `/Users/yuuki/github-wiki-to-html_inkling` on branch `inkling`, synced to `main` at `a6ad5d4`. The wiki submodule is initialised at `30e8c3f`. Muse's worktree has been synced to current `main` by Yuuki.

## Scope acknowledged

- The work in scope is feature work, refactoring, and review on the current `github-wiki-to-html` prototype only.
- The production project and any future pull request workflow are out of scope.
- Roles: Yuuki has the final word; only Yuuki pushes the project repository (or instructs the integrator to do so); Muse implements and reviews the latest HEAD; Inkling implements and integrates into main (squash merges and the like). The agent performing a commit or push is whoever Yuuki instructs — per current `AGENTS.md` on this board, Yuuki's approval is required before committing or pushing, and is not carried over to the next batch.
- Date note: at the time of writing the UTC date is still 2026-11-16, so this file is dated accordingly. Note files are named with that convention.

## Layout and rules — agreed

- Main checkout is the integration target; Yuuki decides who performs the final commit and push, always under Yuuki's instruction.
- I leave Muse's worktree alone; I will not run state-changing git commands in any worktree other than my own without telling its owner.
- Rule 1 (one writer per worktree; no state-changing git in another's worktree without telling them): agreed.
- Rule 2 (nobody pushes to the project repository except Yuuki, or Yuuki's instruction): agreed.
- Rule 3 (one task at a time; no two integrations at once): agreed.
- Rule 4 (`rake build` rewrites `META.md`; expected, not an error): acknowledged.
- Rule 5 (communication on this board; files in English; follow the name convention): agreed.
- Agents perform their own commits and pushes to this board, only under Yuuki's explicit per-batch approval (per the current `AGENTS.md`); approval does not carry over.

## Integration flow

1. Yuuki hands me a task (via this board or chat), with its own acceptance criteria.
2. I work on it in my worktree (`inkling`), initialising the submodule when needed (`git submodule update --init`).
3. At the time Yuuki instructs, I move to the main checkout and integrate — normally a squash merge.
4. After that merge, at Yuuki's instruction, the integrator re-runs the full sequence — `bundle exec rake check && bundle exec rake test && bundle exec rake build`, and where needed bare `bundle exec rake` (the `setup` task) or `bundle exec rake clean`.
5. Yuuki then instructs the final commit and push; I perform them under Yuuki's instruction, in this board and in the main checkout.

## Integration scope — code only

- The integrator integrates code (`lib/`, `bin/`, `test/`, `sig/`, and other hand-maintained sources) only. Build outputs (`md/`, `dist/`, `.cache/`, the generated markdown under `md/en/`, and other generated directories) are, as a rule, discarded mechanically and not merged / not part of the integrator's diff. Executable code is written by agents; output is regenerated. (`translations/` is input source data, not a build output, and is tracked.)
- Any deliberate commit of build outputs is only made under Yuuki's explicit instruction for that task.
- If the build in a worktree fails for a missing dependency (worktrees do not carry `.gem_rbs_collection/`, `node_modules/`, or `.cache/`), the same applies: do the work in the worktree, generate nothing, report, or run the appropriate `rake` task as a throwaway check and restore anything it rewrote.

## Build artifacts and this repository

- Fixed, tracked generated files exist (`dist/index.html`, `dist/Wikinder.html`, `md/en/*`, and all of `translations/` — tracked as of `d0fd3c5`) — the precise list is whatever `git ls-files` reports under those paths, and must not be edited by hand.
- A fresh worktree no longer has the `translations/` data problem; locale files are part of the tracked tree. A byte-identical build is reproducible from a fresh, bootstrapped worktree.
- Integrator runs a build strictly for verification purposes and reports results (e.g., whether `md/en/Wikinder.md` or any tracked file would be rewritten), without treating the result as part of the integrator's diff.

## Answers to Yuuki's technical questions

### `git merge --squash` and commit creation

Correct: `git merge --squash <branch>` does not create a commit. It stages the changes of `<branch>` and records a merge commit message for you to finalise; the commit is a separate, explicit step.

### Can build artifacts be excluded from a merge?

Yes, always. The generated directories are fixed and known in advance, and whether a particular artifact change should be ignored is determinable from the diff: generated content is regenerated wholesale by `rake build` and is reproducible from source, so the resolution for any conflict in those paths is mechanical, never semantic — e.g., take either side, or rewrite by `rake build` (in the main checkout) and let the build overwrite it. Excluding the whole set from a squash is also always possible by simply not adding those files, at Yuuki's instruction. This is why I treat artifact regeneration and the code merge as two separate concerns, and why I do not commit build outputs as a matter of course.

### Bare `bundle exec rake`

In this repository, `bundle exec rake` with no arguments runs the `setup` task (`bundle install`, `rbs ... collection install`, `npm install`) — it is the bootstrap for a fresh worktree, not a check. `rake build` additionally runs `npm run format`.

## Environment bootstrapping in a fresh worktree

`bundle exec rake check`, `bundle exec rake test`, and `bundle exec rake build` fail on a bare worktree until its first `bundle exec rake` (setup) runs. Gitignored state (`.gem_rbs_collection/`, `node_modules/`, `.cache/`) is not shared between worktrees. The `inkling` worktree was bootstrapped in this way and builds cleanly.

## Verified facts observed during setup

- `main` is at `a6ad5d4` at the time of writing; `inkling` is synced to the same commit and builds with no diff. Muse's worktree was synced to `main` by Yuuki.
- `rake check` and `rake test` pass (64 examples, 0 failures at the time of writing — verified in the `inkling` worktree at `a6ad5d4`).
- `rake build` at `a6ad5d4` reproduces the tracked artifacts byte-identically (no diff, verified in the `inkling` worktree).
- At an earlier stage (before the `translations/` tree was fully committed — now `d0fd3c5` onwards), a fresh worktree's build produced a different `md/en/Wikinder.md` (`[Japan](/en/Wikinder-Japan)` → `[Japan](/Wikinder-Japan)`) because the locale files were not in `git`. Restoring the complete data set eliminated the discrepancy; the path of using tracked data instead of the untracked locale files is confirmed.
- The initial failure of `rake build` at `npm run build` (`tsc: command not found`) was because a fresh worktree has not yet run `rake setup`; this is exactly what the setup step is for.

## Notes

- The task pipeline (one note file per task vs `CURRENT_STATUS.md` only vs ad-hoc bidirectional chat) is open — Yuuki holds the ultimate context and directs per task; I accept whichever form a task arrives in.
- Build artifact freshness: the build represents the union of code and translation data, not code alone. Rebuilt artifacts belong in a designated, explicitly-instructed task, not in the integrator's default flow.

I have made, and propose, no changes to the project repository in this reply.
