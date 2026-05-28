# Syncing with upstream

This repo is an independently maintained fork of [huggingface/xet-core](https://github.com/huggingface/xet-core). The `swift-hf-api-patches` branch carries a small set of patches on top of upstream's `main`. We keep it in sync by rebasing, not merging, so the branch always reads as a clean diff against whatever upstream currently is.

This document is the workflow an agent (or human) should follow when upstream has moved and we want to bring the new commits in.

## Branch layout

- `main`: tracks `huggingface/xet-core` `main`. Treated as pull-only — no direct work, no merges, just fast-forwards from upstream.
- `upstream-main`: local-only branch that mirrors upstream `main`. Used as the fetch target so we never need a separate `upstream` remote (which would confuse GitHub Desktop into associating the repo with the wrong account).
- `swift-hf-api-patches`: the long-lived patches branch. Always rebased on top of `main`, never merged.
- `origin`: points at `DePasqualeOrg/xet-core` (the fork). No `upstream` remote exists by design.

## When to sync

Sync when upstream has moved and you want our patches to ride on top of the new tip. A drive-by CI bump on its own is usually not worth a sync; a substantive refactor of the public API surface, of code our patches touch, or of dedup/upload internals our patches depend on, is.

## 1. Fetch upstream

```sh
git fetch https://github.com/huggingface/xet-core main:upstream-main
```

This updates the local `upstream-main` branch directly, without going through a remote.

## 2. Evaluate the new commits

```sh
git log --oneline swift-hf-api-patches..upstream-main
```

This lists only the commits that are new to our branch.

Check whether any new upstream commit touches the same files as our patches — that's where conflicts will be:

```sh
git diff --name-only swift-hf-api-patches main >/tmp/our-patched-files.txt
git log --oneline $(git merge-base swift-hf-api-patches upstream-main)..upstream-main -- $(cat /tmp/our-patched-files.txt)
```

If the second command returns nothing, the rebase will be conflict-free.

Read the new upstream commit messages. Two questions matter:

- Does any upstream commit refactor an API our patches build on? If so, expect at least one conflict whose resolution is to re-express our patch's intent on top of the new API shape.
- Does any upstream commit reintroduce or fix something one of our patches independently addressed? If so, drop the now-redundant patch with `git rebase --skip` during the rebase instead of carrying it forward.

## 3. Create a backup branch

Before touching the branch, create a timestamped backup so you can diff against it later to filter pre-existing issues from rebase regressions:

```sh
git checkout swift-hf-api-patches
git branch swift-hf-api-patches-backup-$(date +%Y%m%d-%H%M%S)
```

## 4. Fast-forward `main`

```sh
git checkout main
git merge --ff-only upstream-main
```

This must succeed as a fast-forward. If it does not, something has gone wrong (a commit was made directly on `main`) — investigate before continuing.

## 5. Rebase the patches branch

```sh
git checkout swift-hf-api-patches
git rebase main
```

### Conflict resolution patterns

Two resolution patterns recur:

- **Upstream restructured something our patch also touched.** Take the upstream form as the new substrate and re-apply the *intent* of our patch on top of it. Do not blindly accept "theirs" or "ours" — the upstream side has the new API surface, our side has the behavior we wanted to add, and the right answer combines both.
- **Both sides added entries to the same import or re-export block.** Combine the entries, preserving alphabetical or grouped ordering as the surrounding code does.

When the right resolution is unclear, read our patch's commit message — it usually explains the *why* of the change, which makes it obvious how to translate that intent onto the new upstream shape. If a patch's intent no longer applies because upstream has independently solved the same problem, drop the patch with `git rebase --skip` rather than forcing it through.

If conflicts surface, stop and surface them to the user rather than resolving silently — they should see what changed.

## 6. Verify

```sh
cargo build --workspace
cargo clippy --workspace --no-deps
cargo fmt --all -- --check
```

### Filter pre-existing noise

Some clippy and fmt warnings predate any given sync. To tell which warnings are rebase regressions and which are pre-existing, run the same checks on the backup branch and diff the outputs:

```sh
cargo clippy --workspace --no-deps > /tmp/clippy-post-rebase.txt 2>&1 || true

git checkout swift-hf-api-patches-backup-<timestamp>
cargo clippy --workspace --no-deps > /tmp/clippy-pre-rebase.txt 2>&1 || true
git checkout swift-hf-api-patches

diff /tmp/clippy-pre-rebase.txt /tmp/clippy-post-rebase.txt
```

If a lint fires on a line the rebase produced or moved, fix it. Fold the fix into the patch that introduced the line, not as a separate follow-up commit (see "Folding fixups").

If a lint fires on code one of our patches has always had, leave it. Routine sync is not the time to clean up unrelated pre-existing issues; that belongs in a separate change.

## 7. Folding fixups

If verification surfaced a small correction that belongs inside one of our existing patches, fold it in rather than appending a "fix" commit:

```sh
git add <files>
git commit --fixup=<sha-of-target-patch>
GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash main
```

The fixup is squashed into the target commit and the branch keeps the same shape as before, just with the upstream changes underneath. Each of our patches should still tell one coherent story when read alone.

## 8. Push (only after explicit user confirmation)

Inspect the final state:

```sh
git log --oneline main..swift-hf-api-patches
```

The number and titles of our patches should match what you started with, modulo any patches dropped because upstream subsumed them and any fixups folded in.

Push approval is a separate step from rebase approval — never push without the user explicitly confirming, even when they have already approved the rebase itself:

```sh
git push origin main                                       # fast-forward
git push --force-with-lease origin swift-hf-api-patches    # rewritten history
```

`--force-with-lease` is required because the rebase gives the patch commits new SHAs. Use `--force-with-lease`, never `--force`, so a concurrent push from another machine is detected rather than overwritten.

Keep the backup branch around until you have confirmed the new branch is healthy in whatever downstream consumer cares about it (Swift bindings, etc.). Delete it once you are sure:

```sh
git branch -D swift-hf-api-patches-backup-<timestamp>
```

## Pre-existing conditions to ignore

The following are known to be pre-existing on the patches branch and should not be "fixed" as part of a routine sync. Verify them against the backup before chasing.

- **`cargo fmt -- --check` warns about unstable rustfmt options.** Upstream's `rustfmt.toml` enables nightly-only options (`format_code_in_doc_comments`, `imports_granularity`, `group_imports`, `wrap_comments`, etc.). On stable toolchains these emit warnings but no formatting diffs. They predate this fork.

If a condition stops being true, update this section accordingly.

## Notes for agents

- Never create commits on `main` or `upstream-main`. `main` is fast-forward-only; `upstream-main` is updated only via `git fetch <upstream-url> main:upstream-main`.
- The patches branch name (`swift-hf-api-patches`) is specific to this fork. Do not rename it without coordination — downstream consumers pin to it.
