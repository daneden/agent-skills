# agent-skills

A small, opinionated collection of [Claude Code](https://claude.com/claude-code) skills (also compatible with the Claude Agent SDK). Each top-level directory contains a single skill defined by a `SKILL.md` file with YAML frontmatter.

## What's here

| Skill | What it does |
| --- | --- |
| [`iterative-design`](./iterative-design) | For creative or aesthetic output where you can't pre-specify what you want — produces multiple low-fidelity variations early and converges on the user's reactions. |
| [`reference-interview`](./reference-interview) | Librarian-style clarification before answering vague or open-ended requests. Surfaces the real underlying need rather than the surface question. |

## Install

### Claude Code

Clone anywhere, then symlink each skill into `~/.claude/skills/`:

```sh
git clone https://github.com/daneden/agent-skills ~/agent-skills
mkdir -p ~/.claude/skills
for d in ~/agent-skills/*/; do
  name=$(basename "$d")
  [ -f "$d/SKILL.md" ] || continue
  ln -sf "$d" "$HOME/.claude/skills/$name"
done
```

To update later: `git -C ~/agent-skills pull`.

### Claude Agent SDK

Point your agent at the cloned directory; each subdirectory is a skill.

## Contributing

These skills are tuned to how I work, but PRs to sharpen wording, fix bugs, or generalize triggers are welcome. Skills that are too personal or workplace-specific live in [my dotfiles](https://github.com/daneden/dotfiles) instead.

## License

[MIT](./LICENSE)
