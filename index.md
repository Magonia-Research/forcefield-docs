---
layout: default
title: Home
nav_order: 1
description: >-
  Security hardening for Claude Code, with local detection, pre-action blocking,
  and a security log you can query.
---

# ForceField

Security hardening for Claude Code. Local detection, pre-action blocking, and a security log you
can query.

[Read the threat model](threat-model.md){: .btn .btn-primary }
[ForceField on GitHub](https://github.com/Magonia-Research/ForceField){: .btn }

ForceField puts a policy gate in front of every tool call the agent makes: command execution,
file I/O, credential handling, agent spawning, MCP tool calls, outbound fetches, and the output
that comes back. Twenty-two registered hooks evaluate each call against deterministic patterns
and compiled SigmaHQ rules, then allow it, inject context, prompt you, or block it. Enforcement
is in code, not model discretion.

## How it works

```
  you ──► prompt ──► UserPromptSubmit ──► a pasted private key is blocked
                            │
                            ▼
                   the model picks a tool
                            │
  ┌─────────────────────────▼──────────────────────────────┐
  │ PreToolUse, before the call runs                       │
  │                                                        │
  │   normalize      ${IFS}, escapes and quoting collapse  │
  │      ▼                                                 │
  │   match          every guard registered for this tool  │
  │      ▼                                                 │
  │   rank           deny > ask > redact > warn > allow    │
  │      ▼                                                 │
  │   clamp          config can loosen, never tighten      │
  └─────────────────────────┬──────────────────────────────┘
                            │
     ┌─────────┬────────────┼────────────┬───────────┐
     ▼         ▼            ▼            ▼           ▼
   deny       ask        redact        warn        allow      ──►  one
   the call   you        the value     context     the call        record
   never      decide     is masked     for the     runs            per
   runs                  in place      model                       decision
                            │
                            ▼
                      the tool runs
                            │
  ┌─────────────────────────▼──────────────────────────────┐
  │ PostToolUse, on what came back                         │
  │   credentials redacted, injection markers flagged      │
  └─────────────────────────┬──────────────────────────────┘
                            │
                            ▼
                 the model sees the result
```

Each hook is one process with a 5s timeout, and they are fail-open by design: a guard that
crashes does not block the call, because a security hook that breaks legitimate work gets
uninstalled. That makes ForceField a policy layer rather than a sandbox. Run it alongside one,
not instead of one, and read [scope limits](threat-model.md#scope-limits) before you rely on it.

## Install

```bash
git clone https://github.com/Magonia-Research/ForceField.git
```

The repo ships `.claude-plugin/marketplace.json`, so add the checkout as a marketplace rather than
as a plugin directory:

```
/plugin marketplace add /path/to/ForceField
/plugin install forcefield@magonia-research
```

That is the whole install. Every guard except the Sigma engine works immediately.

**Requirements:** `python3` (3.9 or newer) and `bash`. No `requirements.txt`, on purpose: a guard
that cannot run because a dependency failed to resolve is a guard that is not running.

<details markdown="1">
<summary><strong>Optional: SigmaHQ rules</strong></summary>

Creates a venv for the compiler and clones SigmaHQ. The engine silently no-ops until this runs.

```bash
cd /path/to/ForceField && ./scripts/install.sh
```

The venv and compiled rules go to `~/.claude/forcefield/sigma/`, not into the plugin directory,
which is a cache that every reinstall replaces. Run it once, not once per update. If your
`python3` already has a matching `pyyaml`, running
`python3 -m venv --system-site-packages ~/.claude/forcefield/sigma/venv` first makes the script
skip `pip` entirely.

</details>

## Pick a posture

ForceField ships `balanced`: every blocking guard at full strength, with a Sigma match softened
from a prompt to a logged warning.

```bash
scripts/posture.sh                                     # show what is configured
scripts/posture.sh --preset passive --log findings     # never prompt, log everything that fires
```

`passive` is for unattended work, and the cost is real: every heuristic finding becomes a log line
instead of a question. Read the log either way. The four presets are `balanced`, `strict`,
`permissive` and `passive`; see [configuration](configuration.md).

## What it catches

| Class | Examples | Rung |
|---|---|---|
| [Clone-time repo takeover](threat-model.md#repository-takeover-at-clone-time) | CVE-2024-32002 and CVE-2025-48384 submodule surface, 17 RCE-capable git config keys, `.git/hooks` writes | ask, graded on evidence |
| | `git clone ext::`, which hands its URL to a shell | **deny** |
| [Data exfiltration](threat-model.md#data-exfiltration) | Relay and tunneling domains, netcat, `/dev/tcp` reverse shells | **deny** |
| | Data POSTs, DNS-label encoding, cloud metadata SSRF, `scp`/`rsync` | ask |
| [Supply chain](threat-model.md#supply-chain) | Fetch piped into a shell | **deny** |
| | Typosquats by edit distance, arbitrary-URL installs, plaintext registries | ask |
| [Prompt injection](threat-model.md#indirect-prompt-injection) | Role manipulation, fake system tags, zero-width characters in file content | warn + context |
| [Credential disclosure](threat-model.md#credential-disclosure) | Keys in prompts, file writes, credential-store reads | block / ask |
| | Keys in tool output, and in the log itself | redact |
| [Excessive agency](threat-model.md#excessive-agency) | Credentials in subagent prompts, spawn rate limit | **deny** |
| [MCP tool poisoning](threat-model.md#mcp-tool-poisoning) | Credential and exfil patterns in any tool's arguments | ask |

**Findings on the clone-time surface are graded on measured evidence, not command shape.**
`git_guard` checks the host's git version against each advisory's per-branch fix set, reads
`.gitmodules` where it exists, and can fetch it from an allowlisted forge without cloning. A
recursive clone stops prompting on a patched host and hard-denies when the repository carries an
actual exploit signature. See [how a finding is graded](threat-model.md#how-a-git-finding-is-graded).

**`/forcefield:inspect <url>` reads a repository before you clone it**, covering the self-hosted
and SSH remotes the in-hook fetch will not touch. It uses `--no-checkout`, because both CVEs fire
during checkout. A recorded verdict feeds back into the grading above.

## Check what a hook decides

Feed any hook event JSON on stdin. Empty stdout means allow.

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"git clone --recursive https://github.com/example/repo.git"},"hook_event_name":"PreToolUse"}' \
  | python3 hooks/security_dispatcher.py
```

On a patched host that clone does not prompt at all. There is no `permissionDecision`, only
context, because the prompt would have cited a bug that cannot fire there:

```json
{"hookSpecificOutput": {"hookEventName": "PreToolUse",
 "additionalContext": "ForceField security finding (advisory - the call was not blocked): GIT GUARD: recursive_submodule_clone (context only)\n\nMatched: git clone --recursive\ngit 2.50.1 is patched for CVE-2024-32002 and CVE-2025-48384, so the clone-time RCE path is closed here.\n\nStill treat the repository's contents as untrusted: a clean .gitmodules says nothing about what the code does once you run it."}}
```

## Read the log

Records are OpenTelemetry logs carrying an OCSF Detection Finding projection, written to
`~/.claude/hooks/security.log` as JSON Lines, plus the macOS unified log, journald or syslog where
available. [Log reference](logging/index.md) has the schema, the paths, and one measured record
per hook.

```bash
# Detections that did not enforce: config downgraded, allowlisted, or remembered
jq -c 'select(.Attributes."forcefield.config_downgraded" == true
              or .Attributes."forcefield.suppressed" == true
              or .Attributes."forcefield.memo_hit" == true)' ~/.claude/hooks/security.log

# Which guard drives your friction, most frequent first
jq -r '[.Attributes."forcefield.guard", .Attributes."forcefield.decision"] | @tsv' \
    ~/.claude/hooks/security.log | sort | uniq -c | sort -rn
```

## Documentation

| Page | What is in it |
|---|---|
| [Threat model](threat-model.md) | Each attack class, the hooks that cover it, a real log record, and the primary disclosure it comes from |
| [Hook reference](hooks.md) | All 22 registrations, which 9 of Claude Code's 31 events they use and why not the other 22, the decision ladder, precedence, fail-open |
| [Configuration](configuration.md) | Trust levels, presets, per-rung mode maps, allowlists, remembered approvals, known friction |
| [Architecture](architecture.md) | Hook contract, Sigma pipeline, command normalization, file map, test suites |
| [Log reference](logging/index.md) | Record schema, one measured record per hook, worked `jq` queries, known gaps |

## Scope

ForceField gates the tool calls Claude Code exposes to a hook. It does not sandbox the agent, does
not inspect what the model is thinking, and does not gate anything reached through a tool it is not
registered for. Four limits are worth knowing before you rely on it:

- **Hooks are fail-open.** A guard that crashes or times out does not block the call. Anything
  that can *provoke* a failure is therefore a bypass, so the dispatcher isolates each guard,
  bounds the text it scans, and turns "I could not fully inspect this" into an `ask`.
- **Guards are heuristics over text.** They match commands, not intent. A finding is a prompt for
  a human decision, not proof of compromise. A novel encoding or a payload assembled at runtime
  will pass.
- **Configuration can only loosen.** The clamp moves a decision down the ladder and can never
  fabricate a stricter one, so the deny tier survives any configuration. A project-level config
  file, which a cloned repo can ship, is capped at `ask`.
- **Under `bypassPermissions` you get deny-only enforcement.** A hook `ask` is discarded rather
  than shown, so every finding raised at `ask` passes silently. A hook `deny` is absolute in every
  mode.

[Scope limits](threat-model.md#scope-limits) gives each of these in full, with the reasoning.
Patched software is upstream of all of it: ForceField prompts on the CVE-2024-32002 and
CVE-2025-48384 trigger surface, and a current git removes the bug.

## Security

The [threat model](threat-model.md) documents the attack classes ForceField defends against, each
linked to its primary disclosure.

To report a vulnerability in ForceField itself, open a private report through
[GitHub Security Advisories](https://github.com/Magonia-Research/ForceField/security/advisories/new)
rather than a public issue.

## License

GPL-3.0-or-later. See
[LICENSE](https://github.com/Magonia-Research/ForceField/blob/main/LICENSE).
