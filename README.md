# AI Agent Skills

Personal collection of reusable [Hermes agent](https://hermes.nousresearch.com) skills.

## Install a skill

```bash
hermes skills install github:DPStudioGR/ai-agent-skills/skills/<category>/<name>
```

## Skills

| Name | Category | Description |
|------|----------|-------------|
| [chrome-cdp-connect](./skills/system/chrome-cdp-connect/SKILL.md) | system | Auto-launches Chrome with CDP debugging enabled so the agent can use the `browser_cdp` tool without manual intervention |

## Structure

```
skills/
  <category>/
    <skill-name>/
      SKILL.md        # Skill instructions (required)
      scripts/        # Helper scripts (optional)
      references/     # Reference docs (optional)
```
