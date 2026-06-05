---
name: ios-version-preflight
description: Use at the START of iOS feature work — before writing code or branching — or when the user says "version preflight", "/ios-version-preflight", "am I about to work on an already-shipped version?", or "check the release status before I start". Compares the source MARKETING_VERSION against the app's live and in-flight App Store Connect versions via asc, and — only if the source number is already claimed — offers to bump the marketing version (patch/minor/major) and land that bump on the default branch, a standalone PR, or the new feature branch. The mirror image of verify-ios: that one gates "done", this one gates "start".
---

# iOS version preflight

## Core idea

New feature work should not pile onto a marketing version that the App Store has
already shipped or accepted. If `MARKETING_VERSION` in the project still equals the
number that's live (or already in review), the next build inherits a version string
that's spoken-for — TestFlight uploads collide, and the work has no clean version to
ship under. This skill catches that *before* the first line of feature code, and
offers to bump + place the bump where the user wants it.

It does **not** touch the build number (`CURRENT_PROJECT_VERSION`) — that's managed
by the archive/CI flow, not here.

## When to run

- The user is starting a new feature / branch and wants a pre-flight.
- Explicitly invoked (`/ios-version-preflight`) or matched by the description above.

This is a *manual* pre-flight. A skill cannot auto-fire "before every change" — if
the user wants it enforced automatically, that's a `settings.json` hook layered on
top, not this skill.

## Steps

### 1. Read the source marketing version

From `*.xcodeproj/project.pbxproj` (NOT Info.plist — that uses `$(MARKETING_VERSION)`):

```bash
grep -m1 "MARKETING_VERSION" *.xcodeproj/project.pbxproj | sed 's/.*= //; s/;//'
```

Note there is usually one `MARKETING_VERSION` line per build configuration (Debug,
Release, per target) — they should all share the same value. Record the value, e.g.
`1.0.2`, and the count of lines (you'll rewrite all of them on a bump).

### 2. Resolve the App Store Connect app ID

In order of preference:
1. `ASC_APP_ID` env var, if set.
2. Bundle ID → app: read `PRODUCT_BUNDLE_IDENTIFIER` from the pbxproj, then
   `asc apps list --bundle-id "<bundle-id>" --output json`.
3. Fall back to the `asc-id-resolver` skill if either is ambiguous.

### 3. Query live + in-flight versions

```bash
asc versions list --app "<APP_ID>" --platform IOS --output json
```

Each entry has a `versionString` and an `appStoreState`. Classify each state:

- **Claimed** (matching the source version here means BUMP):
  `READY_FOR_SALE`, `PROCESSING_FOR_APP_STORE`, `PENDING_DEVELOPER_RELEASE`,
  `PENDING_APPLE_RELEASE`, `IN_REVIEW`, `WAITING_FOR_REVIEW`, `READY_FOR_REVIEW`,
  `ACCEPTED` — i.e. live, or submitted/in-flight.
- **Draft (informational only, does NOT trigger):** `PREPARE_FOR_SUBMISSION`,
  `DEVELOPER_REJECTED`, `REJECTED`, `INVALID_BINARY`, `METADATA_REJECTED`. Mention a
  matching draft so the user isn't surprised, but don't force a bump on it.

The relevant comparison number is the **highest** `versionString` among the claimed
states (call it `live`).

### 4. Compare and branch

Parse both as semver `x.y.z` and compare numerically (not string compare — `1.0.10`
> `1.0.9`):

- **`source == live`** → the live/in-flight number is already claimed. Go to step 5.
- **`source > live`** → already bumped ahead of the store. Report it and stop —
  nothing to do. The user is clear to start work.
- **`source < live`** → anomaly: App Store Connect is ahead of the source (e.g. a
  version was created directly in ASC). Do **not** auto-bump. Surface the mismatch,
  show both numbers, and ask the user how to proceed.

### 5. Offer the bump (only when source == live)

Ask the user, in two parts:

**a. Magnitude** — present the three concrete results, let them pick:
- patch → `1.0.2` → `1.0.3`
- minor → `1.0.2` → `1.1.0`
- major → `1.0.2` → `2.0.0`

**b. Where the bump lands:**
- **Default branch** — commit the version bump straight onto `main` (or the repo's
  default). Verify the working tree is clean first; do not sweep unrelated changes
  into the bump commit.
- **Standalone PR** — create a dedicated branch (e.g. `chore/bump-1.0.3`), commit
  only the bump, push, open a PR with `gh`. Honors this project's "don't push
  without an explicit go" rule — confirm before pushing.
- **New feature branch** — create the feature branch the user is about to work on
  and make the version bump its first commit. Ask for the branch name if not given.

### 6. Apply the bump

Rewrite **every** `MARKETING_VERSION = <old>;` line in the pbxproj to the new value
(use an exact, repo-wide replace — the count must match what step 1 found). Leave
`CURRENT_PROJECT_VERSION` alone.

```bash
# verify before/after counts match
grep -c "MARKETING_VERSION = <old>;" *.xcodeproj/project.pbxproj
```

Prefer a precise string replace over `agvtool` — `agvtool new-marketing-version`
can behave inconsistently when the version lives in build settings rather than
Info.plist. The `asc-xcode-build` skill is the canonical place for version/build
number mechanics if you need more than a flat replace.

Commit with a conventional message (e.g. `chore: bump marketing version to 1.0.3`),
following the landing-target choice from step 5. Do not push unless the user has
said to (this repo gates pushes behind an explicit "ship").

## Output

End with a one-line status the user can act on:
- "Clear to start — source `1.0.3` is ahead of live `1.0.2`." or
- "Bumped `1.0.2` → `1.0.3` on `feat/foo` (not pushed)." or
- "⚠️ Source `1.0.1` is *behind* live `1.0.2` — need your call before proceeding."

## Notes / edge cases

- **No `asc` / not authenticated** → say so plainly and let the user run the source
  check alone; don't guess the live version.
- **Multiple platforms** (universal iOS+macOS apps): this skill compares the IOS
  version. If the app also ships macOS, mention that its version may differ.
- **Pre-1.0 / non-semver strings** (`1.0`, `2`): pad missing components to compare.
