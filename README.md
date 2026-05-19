# agent-skills

A small, opinionated collection of [Claude Code](https://claude.com/claude-code) skills (also compatible with the Claude Agent SDK). Skills live under [`skills/`](./skills); each subdirectory is one skill, defined by a `SKILL.md` file with YAML frontmatter.

## What's here

| Skill | What it does |
| --- | --- |
| [`iterative-design`](./skills/iterative-design) | For creative or aesthetic output where you can't pre-specify what you want — produces multiple low-fidelity variations early and converges on the user's reactions. |
| [`reference-interview`](./skills/reference-interview) | Librarian-style clarification before answering vague or open-ended requests. Surfaces the real underlying need rather than the surface question. |
| [`ship`](./skills/ship) | End-to-end commit-and-push workflow with guards against shipping broken code, blanket-staging, formatter sweeps, fabricated values, and force-resolved pushes. Works in any repo — build systems, scripts, docs, or config. |
| [`swiftui-gotchas`](./skills/swiftui-gotchas) | Reference catalog of SwiftUI, Swift Concurrency, SwiftData, WidgetKit, and App Intents antipatterns under Swift 6 / MainActor-default. Distilled from real corrections across multiple iOS codebases — animation scope, framework-first defaults, widget timeline constraints, `Sendable` services, and bisection over theorizing when stuck. |

## Install

### Claude Code

Clone anywhere, then symlink each skill into `~/.claude/skills/`:

```sh
git clone https://github.com/daneden/agent-skills ~/agent-skills
mkdir -p ~/.claude/skills
for d in ~/agent-skills/skills/*/; do
  name=$(basename "$d")
  [ -f "$d/SKILL.md" ] || continue
  ln -sf "$d" "$HOME/.claude/skills/$name"
done
```

To update later: `git -C ~/agent-skills pull`.

### Claude Agent SDK

Point your agent at the `skills/` directory; each subdirectory is a skill.

## Contributing

These skills are tuned to how I work, but PRs to sharpen wording, fix bugs, or generalize triggers are welcome. Skills that are too personal or workplace-specific live in [my dotfiles](https://github.com/daneden/dotfiles) instead.

## License

[MIT](./LICENSE)
