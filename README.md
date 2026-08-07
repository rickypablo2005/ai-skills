# AI Skills

Personal AI skills library and orchestration system.

## Source of truth

The files under `skills/` are the canonical versions of the user's skills. They should be preserved as provided and should not be silently rewritten by the orchestration layer.

## Current skills

- `skills/backend/` — Backend workflows
- `skills/discovery/` — Discovery and project-understanding workflows
- `skills/frontend/` — Frontend workflows
- `skills/quality/` — Quality, debugging, performance, TDD and triage workflows

## System

The `system/` directory contains metadata for discovery, routing, priorities and validation. It orchestrates the skills; it does not replace their source content.

## Usage convention

Preferred top-level triggers:

- `@backend`
- `@discovery`
- `@frontend`
- `@quality`

Specific workflow aliases may be registered in `system/registry.yml`.

## Rule

If a skill references an external file that is not present in this repository, the missing reference must be reported instead of being invented or silently substituted.
