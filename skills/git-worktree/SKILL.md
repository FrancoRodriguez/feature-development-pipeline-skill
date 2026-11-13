---
name: git-worktree
description: >-
  Creates, binds, bootstraps, verifies, and cleans up dedicated git worktrees
  for isolated feature and bugfix development, preventing branch collisions on
  the main checkout and eliminating duplicate repository clones. Covers agent
  workspace binding (Google Antigravity, Cursor, Claude Code), untracked secret
  and dependency bootstrapping, stacked branches, and end-of-run cleanup of
  stale merged worktrees (>7 days).
---

# Git Worktree Skill

A git worktree attaches an isolated working directory to the same `.git` object store on its own branch, in milliseconds and without duplicating disk storage. Use one worktree per in-flight feature or bugfix (`../<repo>-wt-<slug>`) so parallel agent sessions never overwrite each other or dirty the main checkout.

---

## 1. Create the Isolated Worktree

Inspect existing worktrees and fetch the remote base branch first (`origin/main` or `origin/develop` — verify via `git symbolic-ref refs/remotes/origin/HEAD` or repo guidelines):

```sh
git worktree list
git fetch origin
git worktree add ../<repo>-wt-<slug> -b feature/<slug> origin/<base-branch>
```

- Use `fix/<slug>` for bugfixes, or `origin/<predecessor-branch>` when stacking on an open parent PR.
- **Skip creation** only when the session is already inside a dedicated worktree (`*-wt-*`) or dedicated clean branch created specifically for this task.

---

## 2. Bind the Agent Session & Editor Root

`git worktree add` checks out the new folder on disk, **but it does not move a running agent's default workspace**, and a `cd` inside one shell call lasts only for that command. Without rebinding, file tools keep editing the launch checkout while the worktree stays untouched.

| Agent Environment | How to Bind the Root |
| :--- | :--- |
| **Google Antigravity** | Treat the absolute `<WT_PATH>` as the mandatory target root: every `read_file`, `write_file`, and `edit_file` call MUST use `<WT_PATH>/...`, every `run_command` MUST run inside `<WT_PATH>` (`cd <WT_PATH> && ...` or `git -C <WT_PATH>`), and every `invoke_subagent` prompt MUST pass `<WT_PATH>`. Optionally open the editor window (`agy <WT_PATH>`). |
| **Cursor / VS Code** | Open the worktree as a workspace window (`cursor <WT_PATH>` or `code <WT_PATH>`) and continue from that window so the editor's active branch matches the agent's writes. |
| **Claude Code** | Call `EnterWorktree` with `path: ../<repo>-wt-<slug>` (always pass `path`, never `name`, which creates an unnamed tree under `.claude/worktrees/`), and open `<WT_PATH>` in the user's editor. |

---

## 3. Confirm Before the First Write

Verify isolation from a fresh command before creating any stub, test, or source file:

```sh
git -C <WT_PATH> rev-parse --show-toplevel   # Must equal <WT_PATH>, never the main checkout
git -C <WT_PATH> branch --show-current       # Must equal feature/<slug> or fix/<slug>
```

If either output points to the launch checkout, fix the binding immediately before writing files.

---

## 4. Bootstrap What a Fresh Worktree Does NOT Inherit

A fresh worktree starts strictly from tracked git files at the branch point. Untracked secrets, installed packages, and build artifacts must be bootstrapped (and **never staged into git**):

| Stack | Not Inherited | Bootstrap Action inside `<WT_PATH>` |
| :--- | :--- | :--- |
| **Ruby on Rails** | `config/master.key`, `config/credentials/*.key`, `.env*`, test DB | Symlink `.env` / `config/master.key` from main checkout; run `bundle check \|\| bundle install` and `bin/rails db:test:prepare`. |
| **Node / TypeScript / Monorepos** | `.env*`, `node_modules`, workspace `packages/*/dist`, ORM clients | Symlink `.env*`; run `pnpm install` / `npm ci`, build upstream workspace packages (`turbo run build --filter=<app>^...`), or `npx prisma generate`. |
| **Python** | `.venv`, `.env`, local untracked SQLite state | Symlink `.venv` (`ln -s <MAIN_PATH>/.venv .venv`) and `.env`. Do not copy shared local state databases unless testing isolated schema migrations. |
| **Docker & Ports** | Dev server ports (`3000`, `4000`), test Postgres/Redis ports (`5432`/`5433`, `6379`/`6380`) | Two worktrees cannot bind the same host ports or mutate the same integration test DB simultaneously. Run integration suites one worktree at a time or override `PORT` / test DB env vars. |

---

## 5. Parallel & Stacked Branch Safety

- **Never rebase, reset, or force-push a branch checked out in another worktree**: Check `git worktree list` first. Rewriting SHAs underneath another live session causes silent history divergence.
- **Stacked branches**: When `feature/<B>` stacks on unmerged `feature/<A>`, rebase `<B>` inside `<B>`'s own worktree **after** `<A>` pushes or merges, re-run the verification suite inside `<B>`, and push with `--force-with-lease`.

---

## 6. Sweeping Stale Worktrees (> 7 Days + Merged PR)

Merged worktrees left on disk keep their branch names locked. At the end of a pipeline run, silently check for stale worktrees **without interrupting the task**:

- Candidate criteria (**both** must hold):
  1. Last commit is **> 7 days old**.
  2. Branch has a **merged PR** (`gh pr list --head "$br" --state merged`) OR its `HEAD` is already an ancestor of the remote base branch.
- Exclude the main checkout (first entry of `git worktree list`), the active `<WT_PATH>`, locked worktrees, and detached `HEAD`s.
- **Zsh note**: Name the loop variable `wt`, **never `path`** (assigning `path` in `zsh` overwrites `PATH`).

```sh
cutoff=$(( $(date +%s) - 7*86400 ))
main_wt=$(git worktree list --porcelain | awk 'NR==1{print $2}')
here_wt=$(git rev-parse --show-toplevel)
git worktree list --porcelain \
  | awk '/^worktree /{wt=$2; br=""; lk=0} /^branch /{br=$2; sub("refs/heads/","",br)}
         /^locked/{lk=1} /^$/{if (br != "" && !lk) print wt, br; br=""}
         END{if (br != "" && !lk) print wt, br}' \
  | while read -r wt br; do
      [ "$wt" = "$main_wt" ] || [ "$wt" = "$here_wt" ] && continue
      ts=$(git -C "$wt" log -1 --format=%ct 2>/dev/null || echo "$cutoff")
      [ "$ts" -lt "$cutoff" ] || continue
      pr=$(gh pr list --head "$br" --state merged --json number --jq '.[0].number' 2>/dev/null || echo "")
      [ -n "$pr" ] || continue
      dirty=$(git -C "$wt" status --porcelain | wc -l | tr -d ' ')
      echo "$wt | branch=$br | PR#$pr | uncommitted=$dirty"
    done
```

**Ask only after completing the main task**:
- Report nothing if zero candidates match.
- Otherwise, list each candidate folder, branch, merged PR, and `uncommitted` count (warning that uncommitted files are lost on removal), and remove **only** the ones the user explicitly confirms:
  ```sh
  git worktree remove <abs_wt_path>
  git worktree prune
  ```
