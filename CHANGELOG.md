# Changelog

Versions follow [Semantic Versioning](https://semver.org/). Each release tag `vX.Y.Z` matches
`version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.

## [1.0.0] - 2026-10-02

- Package all 46 existing markdown-only skills as the `devops-skills` Claude Code plugin.
- Add a marketplace for installation as `devops-skills@devops-skills`.
- Document plugin and public GitHub installation, updates, and the maintainer's symlink setup.
- Validate every skill with `agentskills-validate` in CI and check plugin version consistency,
  including release tags.

The plugin includes no hooks, bundled agents, MCP servers, or executable installers.
