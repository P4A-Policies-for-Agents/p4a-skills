# p4a-skills

A curated collection of [Claude Code](https://claude.com/claude-code) **skills** for working with
the **P4A (Policies 4 Agents)** platform as an external **policy author**.

Where [`omni-gateway-pdk-skills`](https://github.com/P4A-Policies-for-Agents/omni-gateway-pdk-skills)
teaches you how to *write PDK/Rust policy code*, this repo teaches you the **P4A platform surface
around that code** — how to discover and deploy policies through the P4A MCP server, how to shape a
GitHub repo so it is submittable, and how to self-check a submission against the platform's minimum
requirements before you submit it.

Each skill is an installable guide that coaches an AI coding agent (or a human reading along)
through one slice of the P4A author workflow. The content references **stable public concepts** —
the P4A documentation site, the OpenAPI contract, and the P4A MCP tool set — never the private
portal's internal file paths.

## What's inside

Skills live under [`skills/`](skills/), one directory per skill (`p4a-<topic>/SKILL.md`):

- **`p4a-mcp-usage`** — connect an agent to the P4A MCP server (PAT auth + scopes) and drive its
  tools to search the catalog, inspect a policy, get an install command, and deploy.
- **`p4a-build-policy`** — assemble a GitHub-hosted policy repo that P4A can accept: required files,
  the `pdk` dependency floor, and the metadata the catalog expects.
- **`p4a-verify-requirements`** — self-check a submission's repo shape and metadata against the
  platform's minimum requirements *before* submitting, so the first submission passes validation.

## Using these skills

Clone or copy the `skills/` directory into a location your agent loads skills from (for Claude
Code, a plugin or your `~/.claude/skills` directory). Once installed, the agent invokes the
relevant `p4a-*` skill automatically when you work against the P4A platform.

## License

[Apache License 2.0](LICENSE). Copyright (c) 2026 Salesforce, Inc.
