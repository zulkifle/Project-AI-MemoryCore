# jessy-skills Plugin

Jessy AI companion skill plugins for Claude Code.

## Active Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `springboot-restful` | Auto — any Spring Boot / Java REST task | Enforces layered architecture, OOP design patterns, best practices |
| `save-memory` | "save", "save memory", "save progress" | Persists conversation insights to memory files |

## Adding New Skills

1. Create folder: `skills/[skill-name]/`
2. Create `SKILL.md` with YAML frontmatter + protocol
3. Done — auto-activates based on description field

Reference `skill-format.md` for the standard structure.
