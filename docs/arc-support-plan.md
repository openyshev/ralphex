# Plan: migrate ralphex from Git-only workflows to Arc support

ralphex currently assumes a Git repository and Git command semantics in the runtime backend, startup checks, worktree mode, documentation, and agent prompts. Supporting Arc should be implemented as first-class VCS support rather than by setting `vcs_command = arc`.

## Goals

- Run ralphex from an Arcadia subdirectory.
- Create/switch Arc branches and commit changes with `arc`.
- Review branch changes with Arc-safe commands such as `arc diff -B`.
- Keep ralphex project state (`.ralphex`, `docs/plans`, progress logs) in the project directory, not at the Arcadia root.
- Preserve existing Git behavior.
- Defer Arc worktree support until the non-worktree flow is stable.

## Current blockers

1. `cmd/ralphex/main.go` checks for `.git`, assumes the current directory is the repository root, and opens a Git service.
2. `pkg/git/external.go` shells out to Git-specific commands and output formats:
   - `git rev-parse --show-toplevel`
   - `git symbolic-ref`
   - `git show-ref`
   - `git status --porcelain`
   - `git worktree add/remove/prune`
   - `git diff --numstat base...HEAD`
   - `git ls-files -z --others --exclude-standard`
   - `git hash-object`
3. Default prompts instruct agents to run Git commands directly. These commands do not go through `vcs_command`.
4. Worktree mode is Git-specific. Arc has `arc worktree`, but it is not a drop-in replacement for `git worktree`.
5. In Arcadia, the VCS root is often the monorepo root while the ralphex project is a subdirectory.

## Target design

Introduce explicit VCS support:

```toml
vcs = auto # auto | git | arc
```

Keep VCS operations semantic in Go code and move Git/Arc differences into backend implementations.

Recommended package direction:

- Keep `pkg/git` temporarily for compatibility, or rename to `pkg/vcs` in a dedicated refactor.
- Introduce `gitBackend` and `arcBackend` implementations behind one service API.
- Keep existing Git behavior unchanged while adding Arc as a separate implementation.

## Phase 1: command compatibility spike

Build and verify a command matrix for operations ralphex needs.

| Semantics | Git today | Arc candidate |
| --- | --- | --- |
| repository root | `git rev-parse --show-toplevel` | `arc root` |
| HEAD hash | `git rev-parse HEAD` | `arc rev-parse HEAD` |
| current branch | `git symbolic-ref --short HEAD` | `arc info --json` |
| default branch | `origin/HEAD`, `main`, `master` | `trunk` by default, config override |
| status | `git status --porcelain -uall` | `arc status --short -u all` |
| add file | `git add -- path` | `arc add path` |
| move file | `git mv` | `arc mv` |
| commit | `git commit -m` | `arc commit -m` |
| create branch | `git checkout -b name` | `arc checkout -b name trunk` |
| switch branch | `git checkout name` | `arc checkout name` |
| review diff | `git diff base...HEAD` | `arc diff -B` |
| diff stats | `git diff --numstat base...HEAD` | parse `arc diff -B --stat` or use `arc diff --json` |
| worktree | `git worktree add/remove/prune` | design separately |

Arc rules to encode in code and prompts:

- Default branch is `trunk` unless configured otherwise.
- Use `arc diff -B` for merge-base review diffs.
- Do not use `arc diff trunk...HEAD` or `arc diff trunk..HEAD`.
- Local branches should not be named `users/<login>/...`; Arc adds that namespace on push.
- Prefer full hashes when a hash must be passed to Arc commands.

## Phase 2: split project root from VCS root

Current assumption:

```text
cwd == repo root == project root
```

Required Arc-compatible model:

```text
project root = directory where ralphex was started
vcs root     = `git rev-parse --show-toplevel` or `arc root`
```

Implementation tasks:

1. Add runtime fields for `ProjectRoot` and `VCSRoot`.
2. Resolve `docs/plans`, `.ralphex/config`, `.ralphex/progress`, and dashboard paths relative to `ProjectRoot`.
3. Run VCS commands from `VCSRoot` when needed.
4. Convert absolute paths to VCS-relative paths using `VCSRoot`.
5. Replace the Git-only startup error with VCS-aware validation:
   - Git may keep the current repo-root requirement initially.
   - Arc should allow running from any directory inside an Arc checkout.

## Phase 3: introduce a VCS backend interface

Move the existing Git command assumptions behind a semantic interface. Example shape:

```go
type Backend interface {
    Root() string
    HeadHash() (string, error)
    CurrentBranch() (string, error)
    DefaultBranch() string

    HasCommits() (bool, error)
    HasChanges(path string) (bool, error)
    HasChangesOtherThan(path string) ([]string, error)

    Add(path string) error
    MoveFile(src, dst string) error
    Commit(message string) error
    CommitFiles(message string, paths ...string) error

    BranchExists(name string) bool
    CreateBranch(name, from string) error
    CheckoutBranch(name string) error

    DiffFingerprint() (string, error)
    DiffStats(base string) (DiffStats, error)

    EnsureRuntimeIgnore(projectRoot string) error
}
```

Then implement:

- `gitBackend`: wraps current behavior.
- `arcBackend`: uses Arc-native commands and formats.

Detection order for `vcs = auto`:

1. Arc if `arc root` succeeds, `.arc` / `.arcignore` exists, or `arc info` succeeds.
2. Git if `.git` exists or `git rev-parse --show-toplevel` succeeds.
3. Otherwise report a clear unsupported repository error.

## Phase 4: implement minimal Arc backend

MVP should exclude worktree support.

Suggested Arc implementation:

- `Root()` -> `arc root`
- `HeadHash()` -> `arc rev-parse HEAD`
- `CurrentBranch()` -> parse `arc info --json`
- `DefaultBranch()` -> `trunk` unless overridden by config
- `Status()` / dirty file detection -> parse `arc status --short -u all`
- `Add()` -> `arc add path`
- `MoveFile()` -> `arc mv src dst`
- `Commit()` -> `arc commit -m message`
- `CreateBranch()` -> `arc checkout -b name trunk`
- `CheckoutBranch()` -> `arc checkout name`
- `DiffStats()` -> parse `arc diff -B --stat` first; consider `arc diff --json` if it is stable and easier to parse
- `DiffFingerprint()` -> hash `arc diff HEAD` plus untracked file content manually; do not depend on `git hash-object`

For unsupported operations in the MVP:

```text
Arc worktree mode is not supported yet; run without --worktree.
```

## Phase 5: make prompts VCS-aware

Default prompt files and prompt builders currently hardcode Git. Add a prompt dialect driven by the selected backend.

Examples:

| Intent | Git | Arc |
| --- | --- | --- |
| first review diff | `git diff {{DEFAULT_BRANCH}}...HEAD` | `arc diff -B` |
| first review stat | `git diff --stat {{DEFAULT_BRANCH}}...HEAD` | `arc diff -B --stat` |
| commit history | `git log {{DEFAULT_BRANCH}}..HEAD --oneline` | `arc log {{DEFAULT_BRANCH}}..HEAD --oneline` |
| uncommitted fixes | `git diff` | `arc diff` |
| commit fixes | `git commit -m "fix: ..."` | `arc commit -m "fix: ..."` |
| update/rebase | `git fetch origin && git rebase origin/{{DEFAULT_BRANCH}}` | `arc pull {{DEFAULT_BRANCH}} && arc rebase {{DEFAULT_BRANCH}}` |

Files/builders to update:

- `pkg/processor/prompts.go`
- `pkg/config/defaults/prompts/review_first.txt`
- `pkg/config/defaults/prompts/review_second.txt`
- `pkg/config/defaults/prompts/codex.txt`
- `pkg/config/defaults/prompts/custom_eval.txt`
- `pkg/config/defaults/prompts/custom_review.txt`
- `pkg/config/defaults/prompts/finalize.txt`

Avoid constructing Arc diffs with Git-style `...` syntax. For PR-style changes, prefer `arc diff -B`.

## Phase 6: runtime ignore files

`EnsureLocalGitignore` currently creates `.ralphex/.gitignore`.

Make this backend-specific:

- Git: keep `.ralphex/.gitignore`.
- Arc: verify whether nested `.arcignore` works in the intended Arcadia checkout. If yes, create `.ralphex/.arcignore`. If not, document or implement a project-level ignore strategy that does not pollute the Arcadia root.

The desired result is that `.ralphex/progress` and temporary runtime files do not show as untracked Arc changes.

## Phase 7: defer Arc worktree mode

Do not implement Arc worktree support in the first PR. It needs separate design because Arc's worktree lifecycle is not equivalent to Git's `worktree add/remove/prune` flow.

For the first Arc release:

- Disable `--worktree` for Arc with a clear error.
- Keep Git worktree behavior unchanged.

Later design topics:

- how to create isolated Arc working trees;
- where to put them;
- how to remove/unmount them safely;
- how to archive plan files on the feature branch;
- how to handle interrupts and retained worktrees.

## Phase 8: docs and UX

Update README and config docs:

```toml
vcs = arc
default_branch = trunk
use_worktree = false
```

Document examples:

```bash
cd ~/arcadia/path/to/service
ralphex docs/plans/my-feature.md
ralphex --review
```

Text cleanup:

- Replace generic “git repository” wording with “VCS repository” where appropriate.
- Keep Git-specific wording only in Git-specific sections.
- Clarify that Arc support starts without `--worktree`.
- Mention that local Arc branch names should not include `users/<login>/`.

## Suggested PR sequence

1. Split `ProjectRoot` and `VCSRoot`, preserving Git behavior.
2. Introduce backend interface and move current Git behavior into `gitBackend`.
3. Add minimal `arcBackend` without worktree mode.
4. Add VCS-aware prompt generation and Arc prompt dialect.
5. Update README/config/help/error messages.
6. Add integration tests in a real Arc checkout.
7. Design and implement Arc worktree mode separately, if still needed.

## MVP acceptance criteria

In an Arcadia subdirectory, the following should work without Git:

```bash
ralphex docs/plans/some-plan.md
ralphex --review
```

Acceptance checklist:

- ralphex detects Arc or respects `vcs = arc`.
- ralphex creates an Arc branch from `trunk`.
- ralphex commits plan and task changes via `arc commit`.
- reviewers use `arc diff -B`, not Git diff syntax.
- `.ralphex` state remains under the project directory.
- `--worktree` under Arc fails early with a clear unsupported-mode message.
- Existing Git tests and behavior remain unchanged.
