# Codex Marketplace

Personal Codex plugin marketplace for `danweinerdev` plugins.

The Codex marketplace manifest lives at `.agents/plugins/marketplace.json`.

## Plugins

- `codex-sdd-planner` - Spec-driven development workflows for Codex, starting with lane-based code review.

## Install

Install from GitHub:

```bash
codex plugin marketplace add danweinerdev/codex-marketplace
codex plugin list --marketplace codex-marketplace --available
codex plugin add codex-sdd-planner@codex-marketplace
```

For local testing from a checkout:

```bash
codex plugin marketplace add <path-to-codex-marketplace>
codex plugin list --marketplace codex-marketplace --available
codex plugin add codex-sdd-planner@codex-marketplace
```
