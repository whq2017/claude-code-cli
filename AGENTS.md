# Repository Guidelines

## Project Structure & Module Organization

This repository is a source-analysis checkout of Claude Code CLI, not a full packaged app. The code lives at the repository root rather than under `src/`. Start with `main.tsx`, `commands.ts`, `tools.ts`, and `context.ts` to trace application flow. Core areas:

- `commands/`: slash-command implementations
- `tools/`: tool definitions and execution logic
- `services/`: API, MCP, analytics, sync, and other backend-facing services
- `components/` and `ink/`: terminal UI layers
- `hooks/`, `utils/`, `types/`, `schemas/`: shared logic and contracts
- `tasks/`, `skills/`, `plugins/`, `migrations/`: background work, extension points, and data migrations

## Build, Test, and Development Commands

The README states this snapshot does not include the full build configuration, package manifest, or dependency lockfiles, so there is no canonical local build or test command in-tree. Use repo-inspection commands while contributing:

- `rg --files commands tools services components`: map the relevant module area
- `rg -n "FileEditTool|buildTool|use[A-Z]" tools/ hooks/`: find implementations by symbol or pattern
- `git status` and `git diff --stat`: review your working tree before and after edits

If you restore missing tooling in a fuller checkout, keep Bun as the default runtime and package manager.

## Coding Style & Naming Conventions

Follow the existing TypeScript/TSX style: 2-space indentation, single quotes, and minimal semicolon use. Preserve relative import specifiers with `.js` suffixes inside TypeScript files. Match current naming patterns:

- `PascalCase` directories for feature bundles such as `tools/FileEditTool`
- `useX.ts` or `useX.tsx` for hooks
- descriptive utility names in `utils/`

Keep `biome-ignore` and `eslint-disable` comments only when required; do not reorder imports around guarded blocks.

## Testing Guidelines

No test runner or test directories are included in this snapshot. For now, validate changes by limiting scope, checking adjacent call sites, and documenting what could not be executed. If you add tests in a fuller environment, place them near the touched feature and use `*.test.ts` or `*.test.tsx`.

## Commit & Pull Request Guidelines

Recent history uses short, imperative commit subjects such as `Create README.md`. Keep commits narrowly scoped and describe the affected area first. Pull requests should include:

- a brief problem statement and change summary
- linked issue or context when available
- screenshots or terminal captures for `components/`, `ink/`, or other UI changes
- explicit notes on validation performed and any missing build/test steps
