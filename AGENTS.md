# Project Instructions

## Reference repositories

The `pnpm bootstrap` command materializes these read-only references in `.repos/`.
Run `./scripts/sync-reference-repos.sh` to refresh them directly.

| Repository                                                    | Path              | Useful for                                                                                   |
| ------------------------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------------- |
| [`Effect-TS/effect`](https://github.com/Effect-TS/effect)     | `.repos/effect`   | Effect APIs, runtime behavior, schemas, and TypeScript implementation patterns.              |
| [`cordiverse/cordis`](https://github.com/cordiverse/cordis)   | `.repos/cordis`   | Context, service injection, plugin lifecycle, effect scopes, and disposal patterns.          |
| [`anomalyco/opencode`](https://github.com/anomalyco/opencode) | `.repos/opencode` | Coding-agent sessions, providers, tools, permissions, and terminal application architecture. |
| [`earendil-works/pi`](https://github.com/earendil-works/pi)   | `.repos/pi`       | Agent loops, model providers, sessions, tools, extensions, and terminal UI architecture.     |

## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues. See `docs/agents/issue-tracker.md`.

### Triage labels

Use the five default canonical triage labels. See `docs/agents/triage-labels.md`.

### Domain docs

This repo uses a multi-context domain-doc layout. See `docs/agents/domain.md`.
