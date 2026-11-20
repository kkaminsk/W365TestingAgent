# Questions and Recommendations: Expert Review of the Windows 365 Agent Testing Design

**Purpose:** Walk three experts through `w365agentdesign.md` and come out with decisions, not open questions.

**Date:** November 20, 2026

**Design revision reviewed:** `research/w365agentdesign.md`, working tree on branch `work/research-agent` as of November 20, 2026, after the GitHub Copilot CLI revision described in section D.

**Reviewers:**

| Section | Expert | What they are asked to judge |
|---|---|---|
| A | Windows 365 expert | Pools, images, Intune, agent sessions, Entra agent identities, Conditional Access, network. |
| B | Windows developer | What actually runs on the Cloud PC: bootstrap, Windows broker, credential storage, git, Edge automation, hardening. |
| C | GitHub Copilot CLI practitioner | How Copilot CLI is installed, authenticated, governed, and driven unattended. |
| D | All | The changes made when the driver moved from Claude Code CLI to GitHub Copilot CLI. |

**How to use this document:** Each expert gets their section. Questions follow the design document's section order so the expert can be walked through the design top to bottom. Every question states the design section it tests, the question, why the answer matters, and the recommendation the design currently rests on. The expert's job is to confirm or overturn the recommendation. Record the answer under the question, then carry the decision into section 18 of the design document.

Question identifiers are `W-n` (Windows 365), `D-n` (Windows developer), and `C-n` (Copilot CLI).

---

## Section A: Windows 365 expert

### W-1. Pool image and management model

- **Design section:** 1 (topology decision), 8.3 (device provisioning)
- **Question:** Do Windows 365 for Agents pools created through the Agent 365 provisioning policy accept a custom image built from the developer image, with full Intune device management? Or are pools limited to Microsoft gallery images with pool-level management only?
- **Why it matters:** The design bakes heavy tooling into the image and uses Intune for policy. If custom images are not supported, every tool must be installed by Intune after each reset, and time to readiness becomes the dominant cost.
- **Recommendation to confirm:** Custom image plus Intune policy. If custom images are unsupported, the fallback is a winget-based Intune install with readiness measured in Phase 1 before the design is accepted.

### W-2. What a session reset actually does

- **Design section:** 6.5 (provisioning sequence), 8.3 (time to readiness), 9 (Phase 5)
- **Question:** At check-in, is the Cloud PC reprovisioned from the image, restored from a snapshot, or is only the user profile wiped? Does anything written to `C:\ProgramData` or `HKLM` survive?
- **Why it matters:** The design assumes a reset disposes of every fetched secret and all Copilot CLI session state, and that machine-wide policy survives. Both assumptions depend on the reset mechanism.
- **Recommendation to confirm:** Treat the reset as a full reprovision for security purposes (nothing fetched survives) and as a profile wipe for policy purposes (ProgramData and HKLM policy survive) until a test proves otherwise. Verify both in Phase 1.

### W-3. Intune targeting for the agent user

- **Design section:** 8.3 (targeting constraint)
- **Question:** Is the agent user a licensed, groupable identity that Intune can target with user-scoped policy, or must all configuration be device-scoped to the pool?
- **Why it matters:** Copilot CLI managed settings and git configuration are machine-wide in the design, but Edge policy and some credential-helper settings are naturally user-scoped.
- **Recommendation to confirm:** Device-scoped targeting for everything. Move anything user-scoped into the session-start bootstrap.

### W-4. Join type and broker prerequisites

- **Design section:** 8.3 (pool support constraint), 6.4 (Azure Key Vault row), 6.5
- **Question:** Microsoft describes agent Cloud PCs as "domain-joined". Are pool machines Microsoft Entra joined, hybrid joined, or something else? Does the agent user get a Primary Refresh Token and a working Windows broker (WAM) inside the session?
- **Why it matters:** The Key Vault and Azure DevOps credential paths depend on silent broker token acquisition under the agent user. Without a PRT there is no silent path, and the bootstrap fails closed every time.
- **Recommendation to confirm:** Run `dsregcmd /status` and a silent MSAL acquire for the Key Vault audience inside a real agent session in Phase 1. If no PRT exists, the credential model needs a different root (see D-2 and C-1).

### W-5. Is there an interactive logon for the agent user?

- **Design section:** 6.2 (Boundary B), 6.5 (step 2 and 3)
- **Question:** When an agent session starts, is the agent user signed in to an interactive Windows desktop session, or is the browser launched in some other way? Can a process started at session start run as that user with access to the broker?
- **Why it matters:** WAM only works from the interactive user's session. The bootstrap, Copilot CLI, and any browser automation must all run inside that session.
- **Recommendation to confirm:** Interactive logon as the agent user, with the bootstrap triggered at logon as that user. If the session is non-interactive, the Key Vault path changes to a certificate or an external secret delivery step.

### W-6. Conditional Access for agent users

- **Design section:** 6.3, 6.5 (Conditional Access paragraph)
- **Question:** Which of the preview capabilities are enabled in the tenant: the "Agent execution environments" condition, "All agent users" targeting, agent risk? Is the tenant licensed for them (Microsoft 365 E7, or Agent 365 plus Entra ID P1)? Can a policy scoped to agent users require a compliant device, and do pool Cloud PCs report compliance?
- **Why it matters:** The agent user must reach Key Vault, Azure DevOps, the test application, and GitHub (for Entra-linked EMU sign-in if ever used). A policy that is too narrow blocks the bootstrap; one that is too wide weakens the tenant.
- **Recommendation to confirm:** One Conditional Access policy scoped to the agent user under Assignments > Agents, allowing only the Key Vault, Azure DevOps, and test application audiences, restricted to the pool's execution environment or network. Record the policy result for every denied sign-in in the run record.

### W-7. Azure RBAC on an agent user

- **Design section:** 6.4 (Azure Key Vault row), 6.5 (must be proven), 18 (Agent user Azure RBAC)
- **Question:** Can an agent user be a member of an Entra security group, and does an Azure RBAC assignment on that group appear in the agent user's token for Key Vault? Microsoft documents RBAC for agent identities only.
- **Why it matters:** This is the only path by which a stored secret (the Linear key, and the fallback Copilot PAT) reaches the box without placing it in the image.
- **Recommendation to confirm:** Group-based assignment of Key Vault Secrets User on a vault dedicated to this agent. Prove it in Phase 1 with a real agent-user token. If it fails, the Key Vault access moves to the agent identity through a client-credential flow, and the bootstrap needs a certificate for that identity.

### W-8. Network path and egress allowlist

- **Design section:** 8.1, 8.3 (egress constraint)
- **Question:** Does the pool use the Microsoft hosted network or an Azure network connection? Can the pool reach a private test application? What egress filtering applies, and who maintains the allowlist?
- **Why it matters:** The design now requires egress to the GitHub Copilot service and github.com in addition to Key Vault, the source control host, Linear, and the application. A private application needs an Azure network connection and DNS.
- **Recommendation to confirm:** Azure network connection if the test application is private, Microsoft hosted network otherwise. The endpoint or cloud platform team owns the allowlist. The GitHub Copilot hostnames come from GitHub's network documentation and are confirmed in Phase 2.

### W-9. Session lifecycle limits

- **Design section:** 9 (Phase 5 note), 14 (session expires or resets)
- **Question:** What are the inactivity eviction threshold, maximum session length, and token lifetimes for an agent session? What happens to an in-flight process when the session is evicted? How many concurrent sessions can a pool run?
- **Why it matters:** A fix-and-retest loop can take an hour or more. Evidence written only to the Cloud PC is lost on eviction, and Copilot CLI's own session state is lost with it.
- **Recommendation to confirm:** Upload evidence after every phase of the workflow, not only at the end. Size the pool for the expected number of concurrent runs plus one.

### W-10. Evidence store reachable from the Cloud PC

- **Design section:** 12.2, 18 (Evidence storage)
- **Question:** Which approved store can the agent user write to from the Cloud PC with its Entra token, and what retention and access policy applies?
- **Why it matters:** The design wants no extra secret for evidence upload. Azure Blob Storage with Entra authentication satisfies that; a store that needs a key does not.
- **Recommendation to confirm:** Azure Blob Storage, one container per agent, Storage Blob Data Contributor assigned through the same group as W-7.

### W-11. Computer-Use MCP from inside the Cloud PC

- **Design section:** 1 (integration decision), 6.2 (Boundary A), 7, 18 (On-device browser control)
- **Question:** Is the Windows 365 Computer-Use MCP server intended to be called from an external orchestrator only, or can an agent running inside the Cloud PC use it to drive its own Edge? If the agent is on the box, is a local browser automation tool (for example a Playwright MCP server attached to the installed Edge) the intended pattern?
- **Why it matters:** If local automation is the right pattern, the Agent 365 token flow in section 6.2 disappears from the on-device design, which removes the hardest unproven integration. Sections 4, 5, and 7 still describe the external-orchestrator variant.
- **Recommendation to confirm:** Local Edge automation for the on-device topology. Keep the Computer-Use MCP path documented for an external-orchestrator variant and for session start and end.

### W-12. Consumption and pool monitoring

- **Design section:** 15 (monitoring, cost considerations)
- **Question:** How is Windows 365 for Agents consumption metered and billed, where can pool utilization and abandoned sessions be seen, and can alerts be set on them?
- **Why it matters:** Section 15 asks operations to track session acquisition, time to readiness, and abandoned sessions. Those need a data source.
- **Recommendation to confirm:** Use the Windows 365 reporting surface the expert names, exported to the same workspace as the run records.

---

## Section B: Windows developer

### D-1. How the bootstrap is launched

- **Design section:** 6.5 (steps 2 to 6)
- **Question:** After the agent user signs in, what launches the bootstrap: a logon-triggered scheduled task running as the logged-on user, a RunOnce entry, an Intune remediation script, or the orchestrator through the Computer-Use MCP? How does the run objective and run ID reach it?
- **Why it matters:** The Windows broker only works from the interactive user's session, so the bootstrap must run as the agent user, not as SYSTEM or an Intune management account.
- **Recommendation to confirm:** A scheduled task triggered at logon, running as the logged-on user, that reads the run objective from the evidence store or a queue and fails closed if it cannot obtain a broker token within a short timeout.

### D-2. Silent token acquisition implementation

- **Design section:** 6.4 (Azure Key Vault row), 6.5 (step 3)
- **Question:** Which library and account hint should the bootstrap use: MSAL.NET with the broker and the operating-system account, or an Azure Identity credential with broker options and the default broker account? PowerShell with the MSAL module, or a small compiled helper? What does the bootstrap do when the broker returns a UI-required result?
- **Why it matters:** A silent attempt can fall back to an interactive prompt that nobody will answer. The design requires fail closed.
- **Recommendation to confirm:** A compiled .NET helper using MSAL with the broker and the operating-system account, with interaction disabled, returning a non-zero exit code on any UI-required result. The bootstrap logs the failure to the evidence store and exits.

### D-3. Where fetched secrets live for the session

- **Design section:** 6.4 (rules), 6.5 (steps 4 and 5)
- **Question:** The agent's shell runs as the agent user and can read anything the agent user can read. Given that, where should the bootstrap put the Copilot token, the Linear key, and the git credential: Windows Credential Manager entries, a credential-helper store, an MCP configuration file, or the process environment of the agent only?
- **Why it matters:** The design rejects environment variables because Copilot CLI lets `COPILOT_GITHUB_TOKEN` override the stored credential and because the model can print the environment.
- **Recommendation to confirm:** Windows Credential Manager for the Copilot token (service name `copilot-cli`) and the git credential; an MCP configuration file for the Linear key with an access control list limited to the agent user. Accept that the model can read what the agent user can read, and compensate with one-hour installation tokens, session-scoped PATs, and Key Vault rotation.

### D-4. Git credential helper, authorship, and signing

- **Design section:** 6.4 (source control rows), 18 (Commit authorship)
- **Question:** Which credential helper serves each host: Git Credential Manager with Entra for Azure DevOps, and what for GitHub installation tokens? Does the agent commit as itself or as the developer? Are commits signed, and may the agent push to protected branches?
- **Why it matters:** The run record must attribute changes, and branch protection decides whether the agent can land a fix without a human.
- **Recommendation to confirm:** Git Credential Manager for Azure DevOps; a small credential helper that calls the broker for GitHub. The agent commits under its own identity, unsigned in Phase 2 and signed in Phase 4, and never pushes to protected branches.

### D-5. Install mechanics on a managed image

- **Design section:** 8.3 (developer image and Intune app rows)
- **Question:** Should Copilot CLI be installed with winget or npm, under which account, and machine-wide or per-user? Does the developer image ship Node.js 22 or later? How is the version pinned, and how is auto-update suppressed for every user on the box?
- **Why it matters:** A per-user npm install under the agent user disappears if the profile is wiped at reset. A machine-wide install survives. Auto-update mid-run changes behavior without a change record.
- **Recommendation to confirm:** Machine-wide winget install of `GitHub.Copilot` with an explicit version, `COPILOT_AUTO_UPDATE=false` set as a machine environment variable, and Node.js only if a chosen MCP server needs it.

### D-6. Configuration precedence and tamper resistance

- **Design section:** 8.3 (device configuration row), 6.5 (must be proven)
- **Question:** Which Copilot CLI configuration is machine-wide (registry policy, `%ProgramFiles%` managed settings, ProgramData policy hooks) and which is per-user (`~/.copilot`)? Can the machine-wide files and keys be access-controlled so the model, running as the agent user, cannot rewrite its own permission policy?
- **Why it matters:** Unattended permission control depends on the model being unable to loosen it. GitHub's documentation disagrees with itself on managed settings precedence.
- **Recommendation to confirm:** Policy in `HKLM\SOFTWARE\Policies\GitHubCopilot` delivered by Intune, hooks in `C:\ProgramData\GitHub\Copilot\policy.d` with write access limited to administrators, and a Phase 2 test that attempts an override from the agent user's profile and confirms it is ignored.

### D-7. Driving Edge on the box

- **Design section:** 7, 18 (On-device browser control)
- **Question:** If the agent drives the installed Edge locally, which mechanism: a Playwright MCP server using the `msedge` channel and the agent user's existing Entra single sign-on, Copilot CLI's built-in computer use, or the Windows 365 Computer-Use MCP? Do Intune Edge policies (remote debugging disabled, extension blocks, profile restrictions) interfere? How are screenshots captured and uploaded?
- **Why it matters:** This decides whether the Agent 365 token flow is needed at all and whether Edge policy must be relaxed for the pool.
- **Recommendation to confirm:** Playwright MCP against the installed Edge with a persistent profile for the agent user so Entra SSO applies. Screenshots go to the local run folder and are uploaded by the bootstrap after each phase.

### D-8. The orchestration entry point

- **Design section:** 9 (Phase 1 and Phase 2)
- **Question:** What starts a run: the Windows 365 session start itself, the scheduled task from D-1, or an external caller that reaches the box through the Computer-Use MCP? How is the Copilot CLI invocation built from the run objective, and what reads its JSON output?
- **Why it matters:** The run record needs a run ID before the first tool call, and something must parse the result and classify the outcome.
- **Recommendation to confirm:** The bootstrap builds the invocation in non-interactive prompt mode with JSON output, writes the run ID into the prompt and the evidence folder name, and a small wrapper parses the JSON stream into the run record.

### D-9. Evidence capture before reset

- **Design section:** 12.2
- **Question:** What captures screenshots, Copilot CLI logs, browser diagnostics, and the run record, and what uploads them before the Cloud PC is released? What happens if upload fails?
- **Why it matters:** Local disk is not durable. An upload failure must be recorded as inconclusive, not silently lost.
- **Recommendation to confirm:** The bootstrap uploads after every phase with the agent user's Entra token (see W-10) and writes an upload manifest. A failed final upload marks the run inconclusive.

### D-10. Hardening the agent user and the box

- **Design section:** 13 (controls)
- **Question:** Is the agent user a standard user, not a local administrator? Should AppLocker or Windows Defender Application Control restrict what the agent can execute? Are secrets kept out of environment variables visible to child processes? Is `--yolo` blocked by managed settings on the device?
- **Why it matters:** The model has the same file and shell access as the agent user. Reducing that access is the only control that does not depend on the model behaving.
- **Recommendation to confirm:** Standard user, application control allowing the developer toolchain only, no secrets in the environment, `permissions.disableBypassPermissionsMode` set to disable, and an explicit deny list for destructive shell patterns.

### D-11. Signalling failure from the box

- **Design section:** 14
- **Question:** When the session is evicted mid-run or Copilot CLI exits with a non-zero code, how does the box tell the outside world what happened? Copilot CLI's exit codes are undocumented beyond "non-zero if interrupted".
- **Why it matters:** The run record needs a failure classification even when the agent did not finish.
- **Recommendation to confirm:** The wrapper treats any non-zero exit as blocked, uploads whatever evidence exists with the exit code and the last JSON event, and the orchestrator decides whether to retry.

---

## Section C: GitHub Copilot CLI practitioner

### C-1. Which GitHub credential the CLI should use

- **Design section:** 1 (driver decision), 6.1 (GitHub identity row), 6.4 (Copilot CLI model access row), 18 (Copilot credential)
- **Question:** The design prefers a GitHub App installation token carrying "Copilot Requests: read", minted by the same broker that mints source-control tokens. GitHub documents this token type for the Copilot SDK and for Actions, not for the standalone CLI. Have you seen the standalone CLI accept an installation token through `COPILOT_GITHUB_TOKEN`? Does your organization allow Copilot requests from App installations billed to the organization? If not, is a fine-grained PAT from a dedicated Enterprise Managed User account acceptable?
- **Why it matters:** The App token gives a one-hour credential with no human seat. The PAT is documented but long-lived and tied to a person-shaped account. The answer decides whether a stored secret exists at all.
- **Recommendation to confirm:** Prove the App token in Phase 2. Fall back to the PAT in Key Vault only if the CLI rejects the installation token.

### C-2. Who pays and how spend is capped

- **Design section:** 6.4, 13 (controls), 15 (cost considerations)
- **Question:** With an App token, requests are billed to the organization as AI credits. With a PAT, they draw on that account's seat. Which does your organization prefer, what is a sensible `--max-ai-credits` per run, and have you seen runaway spend from autopilot loops?
- **Why it matters:** The fix-and-retest loop is open-ended. Without a cap, a stuck run can consume credits until the session is evicted.
- **Recommendation to confirm:** Organization billing through the App token, a per-run credit cap set from Phase 2 measurements, and no autopilot until the loop is understood.

### C-3. Token placement on a shared box

- **Design section:** 6.4 (rules), 6.5 (step 5)
- **Question:** The design stores the token in Windows Credential Manager under the `copilot-cli` service name and avoids `COPILOT_GITHUB_TOKEN` because an environment variable overrides the stored credential and the model can print it. Is that how you handle tokens on machines where the agent runs unattended? Does the CLI pick up a Credential Manager entry written by another process?
- **Why it matters:** If the CLI only reads credentials it wrote itself, the bootstrap must use the `--with-token` sign-in instead.
- **Recommendation to confirm:** Bootstrap writes the token with `copilot login --with-token` from a pipe, never from a file or an environment variable that outlives the call.

### C-4. The exact unattended invocation

- **Design section:** 6.5 (step 6), 9
- **Question:** Write down the command line you would use: `-p` with the objective, `--output-format json`, `--no-ask-user`, specific `--allow-tool` and `--deny-tool` patterns, `--allow-url` scope, `--max-ai-credits`, and whether `--autopilot` with `--max-autopilot-continues` has a place. How do you detect completion versus interruption given exit codes are undocumented?
- **Why it matters:** The wrapper in D-8 is built around this command and its output.
- **Recommendation to confirm:** No autopilot initially. JSON output parsed by the wrapper. Any non-zero exit is blocked. Tool allow patterns limited to `shell(git:*)`, the build and test commands of the target repository, and the named MCP servers.

### C-5. Where MCP servers are defined and how remote OAuth works

- **Design section:** 6.5 (step 1), 7 (Option A), 8.3
- **Question:** For machine-wide MCP definitions (Playwright, Linear, optionally the Windows 365 Computer-Use server), would you use `--additional-mcp-config` from the bootstrap, the user `mcp-config.json`, or `.github/mcp.json` in the repository? Have you used the CLI's OAuth settings (client ID, scopes, client-credentials grant) against a remote MCP server, and did token refresh work across a long session?
- **Why it matters:** Option A's viability depends on the CLI completing the Agent 365 OAuth flow. The config location decides whether the repository can inject servers.
- **Recommendation to confirm:** `--additional-mcp-config` pointing at a file in ProgramData so the repository cannot add servers, with the enterprise MCP allowlist as the backstop.

### C-6. Install and pinning behaviour

- **Design section:** 8.3
- **Question:** On a managed image, winget or npm? Have you seen the behaviour where a newer cached version runs instead of the installed one, and does `COPILOT_AUTO_UPDATE=false` stop it? Does the CLI need Node.js when installed through winget?
- **Why it matters:** A version change between the image build and the run breaks reproducibility of the run record.
- **Recommendation to confirm:** winget with an explicit version, `COPILOT_AUTO_UPDATE=false` machine-wide, version recorded in every run record.

### C-7. Managed settings: which source wins

- **Design section:** 6.5 (must be proven), 8.3 (device configuration row), 18 (Copilot governance gaps)
- **Question:** GitHub's deployment guide says MDM policy beats server-managed settings, which beat the `%ProgramFiles%` file, which beats user settings. The configuration directory reference says users can override most managed keys. Which have you observed? Is `permissions.disableBypassPermissionsMode` honoured when the user passes `--yolo`? Does the MCP allowlist apply to the CLI in your organization, given the About page says the CLI does not support the organization MCP policies and the administration page says it does?
- **Why it matters:** Unattended permission control depends on managed settings winning.
- **Recommendation to confirm:** Registry policy from Intune as the single source, tested in Phase 2 by attempting overrides from the agent user's profile and the command line.

### C-8. Hooks that fail open

- **Design section:** 8.3 (hooks fail open), 13 (controls), 18 (Hook fail-open)
- **Question:** Policy hooks in `C:\ProgramData\GitHub\Copilot\policy.d` are treated as allowed when they time out. Do you rely on hooks for enforcement, or only for logging and shaping? Which `preToolUse` deny patterns would you ship? Is the CLI reading `.claude/settings.json` hooks and `CLAUDE.md` from a checked-out repository a help for migration or a hazard?
- **Why it matters:** A repository can carry hooks and instructions that the agent will honour. Enforcement must not depend on a hook completing in time.
- **Recommendation to confirm:** Managed `permissions.deny` rules for hard limits, hooks for audit logging to the evidence store only, and a repository review rule that instruction and hook files are part of the reviewed code.

### C-9. Instructions, agents, and skills

- **Design section:** 8.3, 13 (controls)
- **Question:** Where should the run objective and the guardrails live: `AGENTS.md` in the repository, `.github/copilot-instructions.md`, the user-level instructions file, or the prompt itself? Would you define custom agents for distinct roles (test runner, fixer, reporter)? Where do skills live on the image?
- **Why it matters:** Instructions in the repository are reviewable but can be changed by a pull request; instructions on the image are controlled but invisible to developers.
- **Recommendation to confirm:** Guardrails in a user-level instructions file delivered by Intune, run objective in the prompt, repository `AGENTS.md` for build and test commands only.

### C-10. Resuming after eviction

- **Design section:** 9 (Phase 5 note), 14 (session expires or resets)
- **Question:** Copilot CLI's `--resume` and `--continue` depend on session state under `~/.copilot`, which the Cloud PC reset discards. Sessions can also be synced to GitHub. Would you resume from synced sessions, or restart from the evidence store? Is session sync acceptable under your data residency rules?
- **Why it matters:** Resume from a synced session keeps context but puts prompts and tool output in GitHub's storage, where local deletion does not remove them.
- **Recommendation to confirm:** Restart from the evidence store. Disable session sync if managed settings allow it, or document it as part of the data-handling review.

### C-11. What the CLI logs and where it goes

- **Design section:** 12.2, 13 (trust boundary 1)
- **Question:** What ends up in `~/.copilot/logs` and the per-session `events.jsonl`: prompts, tool output, secrets? Would you export through the OpenTelemetry settings to the evidence store? What does `--disable-client-telemetry` stop, given GitHub says it does not affect server-side telemetry?
- **Why it matters:** The run record, not the CLI's own history, should be the system of record, and local logs may contain the very content the design keeps out of reports.
- **Recommendation to confirm:** OpenTelemetry export to the evidence workspace through managed settings, local logs uploaded with the evidence and then discarded with the reset, client telemetry disabled.

### C-12. CLI or SDK

- **Design section:** 7 (Option C)
- **Question:** The GitHub Copilot SDK is generally available, uses the same engine, and documents App installation token authentication. Would you build the on-device orchestrator on the SDK instead of wrapping the CLI? What can the SDK do unattended that the CLI cannot?
- **Why it matters:** The wrapper in D-8 re-implements control flow the SDK provides natively. The SDK also closes the "App token with the standalone CLI" gap in C-1 because the SDK documents it.
- **Recommendation to confirm:** CLI for Phase 2 to stay close to the developer experience. Move to the SDK in Phase 4 if the wrapper grows beyond invocation and output parsing.

### C-13. Model pinning

- **Design section:** 6.5 (step 5), 15 (cost)
- **Question:** Pin `COPILOT_MODEL` or let `auto` choose? Which models has the enterprise enabled, and does the managed `model` key override the user's choice?
- **Why it matters:** A model change between runs is a variable the run record must capture, and model pricing differs.
- **Recommendation to confirm:** Pinned through the managed `model` key, recorded in every run record, changed only with a design note.

### C-14. Phase 2 exit criteria you would add

- **Design section:** 16 (Phase 2)
- **Question:** The design's Phase 2 proves the credential type, managed settings precedence, the MCP connection, and a small browser journey. What would you add before calling Copilot CLI integration proven?
- **Why it matters:** The practitioner knows the failure modes the design authors have not met.
- **Recommendation to confirm:** Add at least: a run that exhausts `--max-ai-credits` and stops cleanly, a run where the repository carries a hostile `AGENTS.md`, and a run where the token expires mid-session.

---

## Section D: Changes made for GitHub Copilot CLI

The original design, written and fact-checked on November 18, 2026, assumed Claude Code CLI as the agent running on the Cloud PC. On November 20, 2026 the owner decided the agent will be GitHub Copilot CLI. The design document was revised in place. This section lists every change so reviewers can see the delta from the original.

**What did not change:** the on-device topology (Cloud PC from the developer image, Intune for configuration), the Windows 365 for Agents session model, Entra-rooted access to Azure Key Vault and Azure DevOps through the Windows broker, the Linear key fetched from Key Vault at session start, the two authentication boundaries, Conditional Access guidance, the test strategy, the evidence model, failure handling, and the acceptance criteria.

**The one material consequence:** model access is no longer reached with the agent user's Entra sign-in. Copilot CLI authenticates to GitHub, so a GitHub credential must be delivered at session start. The design prefers a one-hour GitHub App installation token so that no long-lived secret exists, and keeps a fine-grained PAT in Key Vault as the fallback.

| Design section | Original (Claude Code) | Revised (GitHub Copilot CLI) | Reason |
|---|---|---|---|
| Title, status block | "Claude Code Testing with Windows 365 for Agents", dated November 18, 2026 | "GitHub Copilot CLI Testing with Windows 365 for Agents", dated November 20, 2026 | Driver changed. |
| 1 Executive summary | Claude Code CLI investigates and fixes; Claude Code reaches Microsoft Foundry and Key Vault with the agent user's Entra sign-in | Copilot CLI investigates and fixes; the agent reaches Key Vault and Azure DevOps with the Entra sign-in; new "Driver decision (November 20, 2026)" paragraph; product naming clarified (not the retired `gh copilot` extension, not Windows Terminal Chat) | Driver changed; Foundry path no longer applies. |
| 1 Integration decision | "Claude Code supports MCP, but direct compatibility ... must be proven" | Copilot CLI supports stdio, HTTP, and SSE MCP servers and documents OAuth including the client-credentials grant, so direct connection is more plausible but still unproven | Copilot CLI documents more of the needed OAuth surface. |
| 2 Goals | "Give Claude Code enough browser access", "Enable Claude Code to inspect" | Copilot CLI | Word swap. |
| 3 Design principles | Claude Code runs where it can access source code; identities listed without a GitHub identity | Copilot CLI; "GitHub identity" added to the explicit-identities row | New identity in the model. |
| 4 Logical architecture | `CLI[Claude Code CLI]`; no model service node | `CLI[GitHub Copilot CLI]`; new `Copilot[GitHub Copilot service]` node reached from the CLI; mermaid block and all headings de-escaped | Model inference now leaves the tenant; formatting fix. |
| 5 Component responsibilities | Claude Code CLI row; MCP client and bridge rows refer to Claude Code | GitHub Copilot CLI row notes that prompts and tool output go to the GitHub Copilot service; new "GitHub Copilot service" row | New external dependency. |
| 6.1 Identity inventory | Developer identity "runs Claude Code" | "runs Copilot CLI interactively"; new row "GitHub identity for the agent" (App installation preferred, or dedicated Enterprise Managed User with a Copilot seat) | Model access needs a GitHub identity. |
| 6.2 Boundary A | "Claude Code to Windows 365 computer-use tools" | "Copilot CLI to Windows 365 computer-use tools" | Word swap. |
| 6.4 Credential model, intro | Entra sign-in is the root credential for everything the agent does | Entra sign-in is the root credential for Azure-side services; model inference is the exception and needs a GitHub credential | Copilot CLI cannot use an Entra token headlessly. |
| 6.4 Model access row | "Claude Code model access": Microsoft Foundry with Entra ID, Azure default credential chain, Foundry User role, no stored secret, traffic stays inside the tenant | "Copilot CLI model access": GitHub token; primary is a one-hour App installation token with "Copilot Requests: read" minted by the existing broker (undocumented for the standalone CLI, must be proven); fallback is a fine-grained PAT of a dedicated EMU account held in Key Vault (documented, long-lived); watch item is the Entra sign-in preview of November 17, 2026 (interactive only); BYOK to Foundry rejected because it needs an API key and is outside enterprise policy; traffic leaves the tenant | Driver changed; Foundry path replaced. |
| 6.4 GitHub source control row | Separate broker mints installation tokens for git | Same broker and App serve both model access and git | One mechanism instead of two. |
| 6.4 Rules | "exposing tokens to Claude Code as tool output" | "to the agent"; added that `COPILOT_GITHUB_TOKEN` in the environment overrides the stored credential and is readable by the model, so the bootstrap stores the token in Windows Credential Manager under service `copilot-cli` | Copilot CLI credential-resolution behaviour. |
| 6.5 Provisioning step 1 | Intune delivers Claude Code managed settings and managed MCP configuration | Intune delivers Copilot CLI managed settings (registry policy under `HKLM\SOFTWARE\Policies\GitHubCopilot` or `%ProgramFiles%\GitHubCopilot\managed-settings.json`), policy hooks under `C:\ProgramData\GitHub\Copilot\policy.d`, and machine-wide MCP configuration | Copilot CLI's managed-settings mechanism. |
| 6.5 Provisioning step 5 | Set Foundry environment variables without an API key so Claude Code falls through to the Azure default credential chain | Obtain the GitHub token from the broker or Key Vault, store it in Credential Manager, set `COPILOT_AUTO_UPDATE=false`, pin `COPILOT_MODEL`, do not export the token | Foundry path replaced. |
| 6.5 Provisioning step 6 | "Claude Code starts" | "Copilot CLI starts in non-interactive prompt mode" | Word swap plus mode. |
| 6.5 Conditional Access line | Must permit Key Vault and Foundry audiences | Must permit Key Vault and Azure DevOps audiences | Foundry removed. |
| 6.5 Must be proven | Three items: RBAC on agent user, silent broker sign-in, Azure default credential chain finds the sign-in | Five items: RBAC on agent user (Key Vault only), silent broker sign-in, standalone CLI accepts an App installation token, Copilot CLI client enabled and App requests allowed for the organization, managed settings precedence behaves as documented | Credential chain item removed; Copilot items added. |
| 7 Option A | "Direct Claude Code connection" | "Direct Copilot CLI connection"; notes the documented MCP transports and OAuth fields; adds the enterprise MCP allowlist to validation | Copilot CLI MCP client facts. |
| 7 Option B | Claude Code connects to a bridge | Copilot CLI connects to a bridge | Word swap. |
| 7 Option C | "Custom Claude Agent SDK orchestrator" with Microsoft's Claude Agent SDK quickstart link | "Custom GitHub Copilot SDK orchestrator": SDK GA June 2, 2026, six languages, App token auth, BYOK including Foundry; distinguished from the Copilot cloud agent | Driver changed. |
| 8.1 | "The Claude Code environment also needs access to ..." | Copilot CLI environment; adds the GitHub Copilot service | New egress dependency. |
| 8.3 Developer image row | Verify image contents | Adds: npm install needs Node.js 22 or later, winget does not | Copilot CLI install prerequisite. |
| 8.3 Intune apps row | Claude Code, pin versions, disable the Claude Code autoupdater by environment variable | Copilot CLI, machine-wide pinned install (winget `GitHub.Copilot` or npm `@github/copilot@<version>`), `COPILOT_AUTO_UPDATE=false` | Copilot CLI update control. |
| 8.3 Device configuration row | Claude Code managed settings file in ProgramData (permission allowlist, hooks, managed MCP servers) | Copilot CLI managed settings as registry policy (permission allow, ask, deny; `disableBypassPermissionsMode`; `model`; telemetry), policy hooks in ProgramData, enterprise MCP allowlist reference | Copilot CLI's managed-settings mechanism. |
| 8.3 Constraints | Egress lists the Microsoft Foundry resource | New "Hooks fail open" constraint; egress lists the GitHub Copilot service and github.com instead of Foundry | Copilot CLI hook semantics; Foundry removed. |
| 9 Phases, Phase 5 note | Claude Code in Phases 1 and 2; session reset note | Copilot CLI; adds that Copilot CLI session state is discarded by the reset, so resume starts from the evidence store | Copilot CLI session-state location. |
| 10 Testing strategy | "Claude Code with Windows 365 computer-use tools" | Copilot CLI | Word swap. |
| 12.2 Evidence handling | No agent-specific logging note | Adds Copilot CLI logs and session files under `.copilot`, session sync to GitHub, and OpenTelemetry export through managed settings | Copilot CLI logging and sync behaviour. |
| 13 Trust boundary 1 | "Developer environment → Claude service ... approved Claude configuration" | "Cloud PC → GitHub Copilot service": prompts, code, and tool output leave the tenant; enterprise Copilot policy; client telemetry disable does not affect server-side telemetry | Driver changed; topology is on-device. |
| 13 Trust boundary 2 | Restrict tools and bridge access | Adds the enterprise MCP allowlist and managed permission rules | Copilot CLI controls. |
| 13 Controls | Generic shell restriction | Adds: repository instruction files (`AGENTS.md`, `.github/copilot-instructions.md`, `CLAUDE.md`, `.claude/rules`) and `.claude/settings.json` hooks are part of reviewed code; managed `permissions.deny` and `disableBypassPermissionsMode` block `--yolo`; `--max-ai-credits` cap | Copilot CLI reads these files; spend control. |
| 15 Ownership | "Claude Code configuration and repository access" owned by platform or application team | "Copilot CLI managed settings, GitHub App or seat administration, and repository access" owned by the platform team with the GitHub organization administrator | New administrative dependency. |
| 15 Monitoring and cost | "Claude usage" | "Copilot AI credit consumption per run"; cost text explains organization billing for App requests or a seat for the fallback account | Copilot billing model. |
| 16 Phase 0 | "approved Claude data-handling configuration" | "approved Copilot data-handling configuration"; confirm the Copilot CLI client is enabled for the organization | Driver changed. |
| 16 Phase 2 | "Prove Claude Code integration" | "Prove Copilot CLI integration"; adds proving the credential type and that managed settings, hooks, and the MCP allowlist are enforced with `--yolo` refused | New must-prove items. |
| 17 Acceptance criterion 1 | Claude Code acquires and releases a session | Copilot CLI | Word swap. |
| 18 Open decisions | "Silent sign-in in session" asked whether the Azure default credential chain picks up the sign-in; "Deployment initiation" said "Claude-invoked" | "Silent sign-in in session" limited to Key Vault and Azure DevOps audiences; "agent-invoked"; new rows "Copilot credential", "Copilot governance gaps", "Hook fail-open"; "Source control host" notes the broker is shared with model access | Credential chain removed; Copilot decisions added. |
| 19 References | Included Claude Code browser integration, Claude Code on Microsoft Foundry, Microsoft Claude Agent SDK quickstart | Those three removed; fifteen GitHub Copilot CLI and SDK references added (documentation pages and dated changelog entries) | Driver changed. |
| Whole file | 185 lines began with literal backslash escapes (`\#`, `\*\*`, `\-`, `\|`) and the mermaid block was unreadable; every paragraph was separated by two blank lines | Plain markdown, single blank lines, readable mermaid | Formatting fix only; no content change. |

**Claims in the revised design that rest on inference rather than documentation, and must be proven in Phase 2:**

1. The standalone Copilot CLI accepts a GitHub App installation token that carries "Copilot Requests: read". GitHub documents this for the SDK and Actions only.
2. Managed settings delivered by registry policy take precedence over user settings on the device. GitHub's two documentation pages disagree.
3. The enterprise MCP allowlist applies to the CLI. GitHub's two documentation pages disagree.
4. A Credential Manager entry written by the bootstrap is picked up by the CLI without an interactive sign-in.

**Memory and process notes:** the earlier credential-model decision recorded on November 18, 2026 ("Claude Code uses Microsoft Foundry with Entra ID auth") is superseded by this revision. Everything else in that decision stands.
