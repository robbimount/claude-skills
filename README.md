# claude-skills

Claude Code skills:

- **check-yo-self**: a multi-lens self-review of your changes before a PR ("check yo self").
- **yeet-yo-yap**: write and review code comments to be short, accurate and useful ("yeet yo yap").

## Install

Clone once, then symlink each skill into your personal skills directory:

```bash
git clone https://github.com/robbimount/claude-skills.git ~/.claude/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/.claude/claude-skills/check-yo-self ~/.claude/skills/check-yo-self
ln -s ~/.claude/claude-skills/yeet-yo-yap ~/.claude/skills/yeet-yo-yap
```

## Updates

Each skill tells Claude to `git pull --ff-only` this clone when it's used and the last pull was
over a day ago, so a symlinked install stays current. To update by hand:
`git -C ~/.claude/claude-skills pull`.
