# W365TestingAgent

Playground for making Windows 365 an application testbed.

The project explores an AI coding agent that runs on a Windows 365 for Agents Cloud PC. The agent signs in as a dedicated Microsoft Entra agent user, drives Microsoft Edge to test an Entra-protected web application, fixes defects in the repository, redeploys, and retests. It produces a run record that ties each result to a source revision and a deployed build.

**Status:** research and design only. There is no code yet. Everything in `research/` is a proposal that still has to be proven in a real tenant.

## Contents

| File | What it is |
|---|---|
| [`research/w365agentdesign.md`](research/w365agentdesign.md) | Reference architecture: topology, identity and credential model, MCP integration options, device provisioning, workflow, testing strategy, security, roadmap, open decisions. |
| [`research/questionsandrecommendations.md`](research/questionsandrecommendations.md) | Review guide for three experts (Windows 365, Windows developer, GitHub Copilot CLI practitioner), a change log from the original Claude Code design, and the results of the Copilot CLI fact-check. |

## Current design decisions

- **Agent:** GitHub Copilot CLI runs on the Cloud PC itself. It replaced Claude Code CLI on November 20, 2026.
- **Device:** the Cloud PC pool is built from the Windows 365 developer image. Intune installs software and delivers configuration, never secrets.
- **Credentials:** the agent user's Entra sign-in roots access to Azure Key Vault and Azure DevOps. Model access needs a GitHub token. A short-lived GitHub App installation token is preferred and a fine-grained PAT held in Key Vault is the fallback.
- **Browser control:** undecided. The agent either drives Edge through the Windows 365 Computer-Use MCP server or through a local automation tool.

## What is not proven

These need a real tenant and are listed as Phase 2 proof items in the design:

1. The standalone Copilot CLI accepts a GitHub App installation token. GitHub documents it for the Copilot SDK only.
2. Managed settings enforce the limits the design needs. GitHub's own documentation conflicts on precedence.
3. Server-managed settings, including the MCP allowlist, reach an agent that has no signed-in GitHub account.
4. Azure RBAC and silent Windows broker sign-in work for an Entra agent user inside an agent session.

## Working with this repository

Work happens on branches named `work/<topic>`. Documentation is plain Markdown. The Copilot CLI claims were fact-checked on November 20, 2026 against GitHub documentation, and section E of the questions document records each result.

## License

See [`LICENSE`](LICENSE).
