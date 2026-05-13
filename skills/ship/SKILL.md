---
name: ship
description: Use when ready to commit and push completed work — when the user says "ship it", "ship this", "commit and push", or otherwise signals work is done and should land. Verifies the change, stages deliberately, drafts a conventional commit, and pushes safely, with explicit guards against shipping broken code, blanket-staging unrelated work, formatter hooks sweeping the tree, fabricated values, and force-resolved pushes.
---

# Ship

## Core idea

"Ship it" feels routine, but the same failure modes recur whenever it's treated as a one-liner: unverified code lands, an unstaged file gets swept in, a hook reformats files outside the change, a value gets invented, or a rejected push gets force-pushed away. Each is cheap to prevent and expensive to recover from. This skill is a checklist that runs every time, in every kind of repo — build systems are one form of verification, but the discipline applies to scripts, docs, config, and content too.

## Workflow

### 1. Verify the change

Pick the strongest available verifier and run it before staging. The right one depends on what's in the repo:

| Project signal | Verifier |
|---|---|
| `package.json` with a `build` script | `npm run build` / `pnpm build` / `bun run build` |
| `package.json` with no `build` but a typecheck | `npm run typecheck` (or equivalent) |
| `Cargo.toml` | `cargo build` — or `cargo check` for fast feedback |
| `go.mod` | `go build ./...` |
| `pyproject.toml` / `setup.py` | the project's typechecker or test command (e.g. `mypy`, `pytest`) |
| `Package.swift` | `swift build` |
| `*.xcodeproj` / `*.xcworkspace` | `xcodebuild -quiet -scheme <scheme> build` |
| Shell scripts changed | `bash -n <file>` and `shellcheck <file>` if available |
| Markdown / docs / content only | re-read the diff hunk by hunk as if reviewing someone else's PR |
| Config files (JSON / TOML / YAML) | parse with `jq` / `yq` / equivalent, or load with the consuming tool |
| Dotfiles or symlink-only changes | check whether anything sources the changed file and exercise it once |
| Nothing of the above | re-read the diff and walk through it yourself before committing |

If verification fails, **stop**. Fix the underlying issue and re-run. Don't commit broken code.

Pick the *strongest* verifier the project supports, not the easiest. A passing typecheck beats a clean lint; a passing test beats a passing build.

### 2. Sweep for repeats (when fixing)

If this commit fixes a bug, grep the codebase for the same pattern — the same API call, idiom, data shape, or copy-pasted snippet — before declaring done. Many bugs ship in pairs because the original fix didn't sweep. Report what you searched for and what you found (including "no further matches").

### 3. Inspect the diff

Run `git status` and `git diff --stat` in parallel. Report:

- What's already staged
- What's modified but unstaged
- Any untracked files

### 4. Stage by name — never blanket-add

Never run `git add -A`, `git add .`, or `git commit -a`. Stage by name. If there are unstaged modifications you didn't make this session — a file you don't recognise, a regenerated lockfile, an unrelated formatting change — pause and ask: include, revert, or leave alone? Blanket-staging is how unrelated work ends up in a "ship it" commit.

### 5. Watch for hook sweeps

If a pre-commit hook (SwiftFormat, Prettier, ESLint --fix, gofmt, ruff, etc.) modifies files outside the intended scope, **stop**. Show what the hook touched and ask whether to bundle the sweep into this commit, split it into a separate commit, or revert it. Don't quietly accept a 30-file diff because the formatter felt like it.

### 6. Never fabricate values

URLs, IDs, phone numbers, emails, version numbers, dates, ticket references — if you don't have a real value, ask the user or leave a `TODO`. Don't invent. Made-up values are hard to catch in review because they look plausible.

### 7. Draft a commit message

- Imperative mood, present tense.
- Subject under 72 characters.
- Body explains *why*, not *what* — the diff already says what.
- Match the repo's existing style — check `git log -10 --oneline` first. If the repo uses conventional commits (`feat:`, `fix:`, etc.), follow that; if it doesn't, don't impose it.
- No trailing summaries the user didn't ask for.

Pass multi-line messages via HEREDOC.

### 8. Commit and push

Commit, then push to the current branch's upstream. If the branch has no upstream set, ask before setting one. Never force-push, amend a published commit, or rebase without explicit instruction.

#### 8a. If the push is rejected (remote has diverged)

1. **Stop.** Don't retry the push; don't reach for `--force`.
2. `git fetch <remote> <branch>` and inspect what's incoming: `git log --oneline LOCAL..REMOTE` and `git show --stat <each new sha>`.
3. Check for path overlap with the local commit. If the incoming commits touch any of the same paths, surface the conflict risk before integrating.
4. Ask the user how to integrate: **rebase** (preferred when history is linear and there's no overlap), **merge** (introduces a merge commit), or **hold off**.
5. If the working tree has unrelated dirty files, `git stash push -m "..." <paths>` only those paths — don't blanket-stash — integrate, push, then `git stash pop`.
6. After integrating, push and verify the remote moved (`git push` output shows `OLD..NEW <branch> -> <branch>`).

### 9. Report PR status (if applicable)

If on a branch with an open PR, report the PR URL and CI status:

```sh
gh pr view --json url,statusCheckRollup --jq '{url, checks: [.statusCheckRollup[] | {name, status: .conclusion // .status}]}'
```

If `gh` isn't installed or there's no PR, skip this step silently.

## Red flags — STOP and ask

- Verifier failed → don't commit; fix first.
- Unstaged file you didn't author this session.
- Hook touched more than a handful of files outside the change's scope.
- About to invent a value.
- Force-push, `--amend`, or rebase without explicit instruction.
- Pre-commit hook failure → fix the underlying issue and create a NEW commit; never `--no-verify` to bypass it.
- Push rejected → fetch and inspect; never reach for `--force`. Follow 8a.

## Quick reference

| Step | Command |
|---|---|
| Verify (TS/JS) | `npm run build` / `bun run build` / `npm run typecheck` |
| Verify (Rust / Go) | `cargo check` / `go build ./...` |
| Verify (Xcode / Swift) | `xcodebuild -quiet -scheme <S> build` / `swift build` |
| Verify (shell) | `bash -n <file> && shellcheck <file>` |
| Verify (docs / config) | re-read the diff; parse with `jq` / `yq` |
| Status | `git status && git diff --stat` |
| Recent style | `git log -10 --oneline` |
| Commit | `git commit -m "..."` (HEREDOC for multi-line) |
| Push | `git push` (or `git push -u origin <branch>` first time, with confirmation) |
| Push rejected: inspect | `git fetch && git log --oneline LOCAL..REMOTE` |
| Push rejected: stash unrelated | `git stash push -m "..." <paths>` |
| PR status | `gh pr view --json url,statusCheckRollup` |
