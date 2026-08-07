# AI Skills — Agent Instructions

This repository is the source of truth for the user's personal AI skills.

## Before acting

1. Read `system/registry.yml` to discover available skills and aliases.
2. Read `system/routing.yml` to resolve explicit triggers and task intent.
3. Read `system/priority.yml` when multiple skills apply or instructions conflict.
4. Read `system/validation.yml` before declaring work complete.
5. Load the relevant file under `skills/` before applying a skill.

## Source preservation

- Treat files under `skills/` as canonical user-provided skill content.
- Do not silently rewrite, summarize away, or replace skill instructions.
- If a referenced file is missing, report it instead of inventing its contents.

## Trigger convention

Top-level triggers:

- `@backend`
- `@discovery`
- `@frontend`
- `@quality`

Specific aliases are defined in `system/registry.yml`, including `@debug`, `@performance`, `@tdd`, `@api`, `@supabase-postgres`, `@lighthouse`, `@redesign`, `@animations`, and `@design`.

## Execution posture

Detect → load → compose → execute → validate → report.

Do not claim work was performed unless it was actually performed and verified.
