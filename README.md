# agent-skills

Skills for [Claude Code](https://claude.com/claude-code).

| Skill | Description |
|---|---|
| [go-review](skills/go-review/SKILL.md) | Review Go changes against an opinionated checklist (arpc handlers, pgctx, schema, asynq, project layout). Also `/go-review setup godoclive` to wire OpenAPI doc generation. |

## Install

```sh
git clone https://github.com/xkamail/agent-skills.git
cp -R agent-skills/skills/go-review ~/.claude/skills/
```
