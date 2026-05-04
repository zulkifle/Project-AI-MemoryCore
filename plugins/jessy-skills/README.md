# jessy-skills Plugin

Jessy AI companion skill plugins for Claude Code.

## Active Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `springboot-restful` | Auto — any Spring Boot / Java REST task | Enforces layered architecture, OOP design patterns, best practices |
| `laravel-best-practices` | Auto — any Laravel PHP code: controllers, models, migrations, queries, security, routing, architecture | Full Laravel best practices with 20 rule files covering DB performance, security, caching, eloquent, validation, queues, scheduling, and more |
| `pest-testing` | Auto — any Pest test writing, editing, fixing, or refactoring | Pest v4 testing: test()/it(), datasets, mocking, browser testing, smoke testing, arch() tests |
| `tailwindcss-development` | Auto — any Tailwind styling, responsive layouts, UI components, dark mode | Tailwind v4: CSS-first config, replaced utilities, flex/grid layouts, dark mode |
| `laravel-php-skills` | Auto — combined trigger for all Laravel/Pest/Tailwind tasks | Combined full-stack Laravel skill (backend + testing + UI) — single activation covers all three domains |
| `save-memory` | "save", "save memory", "save progress" | Persists conversation insights to memory files |

## Adding New Skills

1. Create folder: `skills/[skill-name]/`
2. Create `SKILL.md` with YAML frontmatter + protocol
3. Done — auto-activates based on description field

Reference `skill-format.md` for the standard structure.
