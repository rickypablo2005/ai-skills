# AI Skills — Claude Code

Use this repository as the canonical skill registry.

Before executing a task:

1. Read `system/registry.yml`.
2. Resolve explicit `@...` aliases using the registry.
3. If no explicit alias exists, use `system/routing.yml` to infer the most specific workflow from the task.
4. Load the canonical skill file under `skills/`.
5. Apply `system/priority.yml` when multiple skills interact.
6. Validate using `system/validation.yml` before declaring completion.

The canonical skill text must not be silently rewritten. Missing referenced files must be reported rather than invented.

Preferred triggers include:

- `@backend`
- `@discovery`
- `@frontend`
- `@quality`
- `@debug`
- `@performance`
- `@tdd`
- `@triage`
- `@api`
- `@supabase-postgres`
- `@lighthouse`
- `@redesign`
- `@animations`
- `@design`
