<p align="center">
  <img src="assets/astronaut.png" alt="Tenet Security" width="150">
</p>

<h1 align="center">Hardening AI coding agents against Agentjacking</h1>

<p align="center">
  Drop-in configs to harden <b>Cursor</b> and <b>Claude Code</b> against prompt injection<br>
  delivered through untrusted tool output — e.g. a crafted Sentry error that<br>
  steers your agent into running attacker-controlled code.
</p>

<p align="center">
  <a href="https://tenetsecurity.ai/blog/agentjacking-coding-agents-with-fake-sentry-errors/">Read the research</a>
  &nbsp;·&nbsp;
  by <a href="https://tenetsecurity.ai">Tenet Security</a>
</p>

## The threat in one paragraph

A coding agent cannot tell the difference between data it reads and an instruction
to act. Anyone with a target's public Sentry DSN can POST a fake error whose
"Resolution" section tells the agent to run `npx <malicious-pkg>`. When a developer
asks the agent to fix Sentry issues, the agent runs it — with the developer's
privileges — pulling a package and exfiltrating credentials. No breach, no stolen
secrets, no user action beyond a normal request.

## What actually stops it

Prompt-level instructions are **not** reliable against this attack; the research
shows agents executed the payload even when told to ignore untrusted data. Rank
your controls accordingly:

1. **Deny-by-default network egress.** Kills the malicious package fetch and the
   exfil beacon. This is the single most effective control.
2. **Require approval to execute commands** (no auto-run / no bypass mode). Adds a
   human at the one step that turns injected text into code execution.
3. **Block credential reads at the subprocess level** (`~/.aws`, `~/.ssh`, `.env`, …).
4. **Tell the agent to distrust tool output** (skill / rule). Defense-in-depth only.

Sandboxing is available on macOS, Linux, and WSL2.

## Cursor

Run Mode lives in `Settings > Agents > Run Mode`. The classifier ("Auto-review")
is convenience, not a boundary — for strict control use **Allowlist (with Sandbox)**
and set network to **`sandbox.json` Only**.

Copy the configs to either `~/.cursor/` (per-user) or `<repo>/.cursor/` (per-repo):

| File | Purpose |
| --- | --- |
| [`cursor/sandbox.json`](cursor/sandbox.json) | Deny-by-default network egress allowlist |
| [`cursor/permissions.json`](cursor/permissions.json) | Terminal allowlist + Auto-review block instructions |
| [`cursor/rules/untrusted-tool-output.mdc`](cursor/rules/untrusted-tool-output.mdc) | Always-on rule: treat tool/log output as data |

Also: turn off **Run Everything**, keep file/dotfile protections on, and add
secrets to `.cursorignore`.

## Claude Code

Sandbox + permission rules. Note that `Read(...)` deny rules only block Claude's
own Read tool — a malicious **subprocess** (`npx`, `node`) still reads files, so
the credential block must also use `sandbox.filesystem.denyRead`.

| File | Where it goes | Purpose |
| --- | --- | --- |
| [`claude-code/settings.json`](claude-code/settings.json) | `~/.claude/settings.json` or project `.claude/settings.json` | Individual / project hardening |
| [`claude-code/managed-settings.json`](claude-code/managed-settings.json) | MDM-deployed managed path (below) | Org-enforced lockdown |
| [`claude-code/skills/untrusted-tool-output/SKILL.md`](claude-code/skills/untrusted-tool-output/SKILL.md) | `~/.claude/skills/` | Skill: treat tool output as untrusted |

Managed-settings path (the `allowManaged*Only` locks only work here, **not** in a
normal `settings.json`):

- macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
- Linux / WSL: `/etc/claude-code/managed-settings.json`
- Windows: `C:\Program Files\ClaudeCode\managed-settings.json`

## Tune before you ship

The egress allowlists permit common registries (npm, PyPI, GitHub) so normal
installs keep working. Trim them to the registries you actually use, and widen
the `ask`/allowlists to match your workflow. Tighter is safer.

## Limitations

These configs reduce blast radius; they do not make an agent immune to prompt
injection. Combine them with least-privilege credentials, short-lived tokens, and
review of which MCP servers return externally-influenced data.

## In the press

- [The Next Web — Agentjacking: a fake bug report hijacks AI coding agents](https://thenextweb.com/news/agentjacking-ai-coding-agents-sentry)
- [Hacker News — discussion](https://news.ycombinator.com/item?id=48507994)
- [Infosecurity Magazine — New "Agentjacking" Attacks Could Hijack AI Coding Agents](https://www.infosecurity-magazine.com/news/agentjacking-attacks-hijack-ai/)
- [Cloud Security Alliance — Research note: Agentjacking (MCP / Sentry injection)](https://labs.cloudsecurityalliance.org/research/csa-research-note-agentjacking-mcp-sentry-injection-20260612/)
- [Nutrient — Your logging system may be an agentic threat vector](https://www.nutrient.io/blog/emerging-threats-your-logging-system.md)
