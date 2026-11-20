# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this repository is

A research and design project for running an AI coding agent on a Windows 365 for Agents Cloud PC to test and fix an Entra-protected web application. There is no application code, build, or test suite. The deliverables are Markdown documents in `research/`.

- `research/w365agentdesign.md` is the reference architecture. Its section numbers (1 to 19) are cited by other documents, so keep them stable.
- `research/questionsandrecommendations.md` is the expert review guide. Sections A to C hold numbered questions (`W-n`, `D-n`, `C-n`), section D is the change log from the original Claude Code design, and section E records the Copilot CLI fact-check.

## Decisions already made

Do not reopen these without the user asking.

- GitHub Copilot CLI is the agent. It replaced Claude Code CLI on November 20, 2026. The design therefore describes Copilot CLI, not Claude Code.
- The agent runs on the Cloud PC, not on a developer workstation. The pool is built from the Windows 365 developer image, and Intune installs software and configuration.
- Intune never delivers secrets. Secrets and tokens are fetched at session start by a bootstrap that runs as the agent user.
- The agent user's Entra sign-in roots Azure Key Vault and Azure DevOps access. Model access uses a GitHub token: a one-hour App installation token is preferred, and a fine-grained PAT in Key Vault is the fallback.
- No Key Vault VM extension and no Azure Arc on Cloud PCs.

## Accuracy rules for this project

The design depends on documentation for preview features that conflict or are silent. Treat accuracy as the main quality bar.

- Separate documented, inferred, and unproven. Say "undocumented" or "must be proven in the target tenant" instead of asserting.
- Do not state Copilot CLI, Windows 365 for Agents, or Entra Agent ID behavior from memory. Check the current official documentation or changelog and record the source and date.
- Most docs.github.com pages are undated. Cite the page and say "retrieved" with the date. Dated changelog entries are stronger evidence.
- Name the product precisely: "GitHub Copilot CLI" (`copilot`, `@github/copilot`). It is not the retired `gh copilot` extension and not Copilot in Windows Terminal.
- Known conflicts and gaps are tracked in section E of the questions document and the Phase 2 proof list in design sections 6.5 and 18. Update them when a claim is proven or refuted.

## Editing conventions

- Plain Markdown with single blank lines. Do not add backslash escapes before headings, bold markers, or list dashes. An earlier version had them and did not render.
- Keep the mermaid diagram in design section 4 unescaped.
- When a design change affects a cited section, update the matching question and the change log in section D of the questions document.
- Write dates in full, for example November 20, 2026.
- Write for a reader who knows the domain but did not watch the work: short sentences, no invented shorthand names.

## Git

- Work on a `work/<topic>` branch. The current one is `work/research-agent`. The main branch is `main`.
- Commit and push only when the user asks.
- End commit messages with the attribution line given in the session's system reminder.
- The Windows working tree converts line endings, so Git warns about LF and CRLF. This is expected.

## Tooling notes

- A very long shell heredoc fails on Windows with an argument-length error. Write large files with the file tools or put scripts in a file first.
- Apostrophes inside a bash heredoc passed through the tool can break parsing. Put such scripts in a file.
- Use `research/.claude/` only for local settings. Do not commit secrets or tokens anywhere in this repository.
