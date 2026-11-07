+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: An Immutable Linux Agentic Coding Setup'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

Default Bluefin, one `dev` container, two curl installs. The host stays immutable; the container absorbs toolchain churn; `$HOME` carries the configuration. The highest-leverage piece is a global `AGENTS.md` that tells every agent how to write, delegate, verify, and choose the simplest design.

## The stack

<img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin, OpenCode, and Hex architecture" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

- **Default Bluefin.** `ghcr.io/ublue-os/bluefin-dx:stable` with zero layered `rpm-ostree` packages. The host keeps Wayland, PipeWire, Vulkan, Flatpak GUI apps, and atomic updates.
- **One `dev` Distrobox.** Debian on rootless Podman. `mise` manages `rust`, `go`, `node`, `python`, and `gh`; the container handles mutable dependencies.
- **OpenCode and Hex from curl.** Both install into `$HOME`, so the host and container share the same binaries and config. No host packages, no custom image.

```bash
curl https://mise.run | sh
curl -fsSL https://opencode.ai/install | bash
curl -fsSL https://anoma.ly/hex/install-linux.sh | sh
```

Host shells need one helper:

```bash
eval "$(mise activate bash)"
opencode() { distrobox enter dev -- opencode "$@"; }
```

`$HOME` is the contract: `~/.config/opencode`, `~/.config/mise`, `~/.local/share/hex`, and git worktrees are visible on both sides.

## Global AGENTS.md

<img src="/images/opencode-delegation-flow.svg" alt="OpenCode subagent delegation and verification flow" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

`~/.config/opencode/AGENTS.md` is the centerpiece. It applies to every project and every subagent. Exact contents:

```markdown
# Writing

- Be terse. Cut filler, hedging, and meta-commentary.
- Lead with subject and verb, never `What ...` or `The way ...` openers (`"What X is..."` -> `"X is..."`; `"What this prototype implements"` -> `"This prototype implements"`).
- State reality directly: separate what exists now from production (`"This repo uses placeholder X now; production uses Y"`).

# Delegation

- Delegate hands-on work: `explore` for reading/search, `general` for multi-step research, edits, and commands. Keep the primary agent for planning, decisions, review, and summary.
- Work directly only on trivial single steps. Put your context into the subagent brief instead of doing the work yourself.
- Subagents start fresh: state goal, scope, expected size, gotchas, and definition of done. Tell them to stop and report rather than expand scope.
- Parallelize independent tasks. Never share a checkout, branch, or stateful resource across parallel tasks; use a worktree.
- Check tool output before rerunning when results look missing. Spot-check key subagent claims.

# Verification

- Gate irreversible steps (push, merge, deploy, delete) on passing checks with `&&` or `set -e`, never `;`.
- Report failures immediately, especially after an irreversible step.

# Engineering principles

## Grug brain / Carson Gross
- Cut complexity: say no to unrequested features/abstractions; offer the 80/20 solution.
- Keep behavior local (locality of behavior) rather than split across layers.
- Prefer boring, proven tech and the simplest stack that fits.
- Favor integration tests over brittle unit tests; add a regression test for every bug fix.
- Name intermediate values over clever one-liners; optimize for debuggability.
- Chesterton's fence: understand why code exists before removing it.

## Mitchell Hashimoto
- Make it work, then make it good: ship small, demoable increments, not big-bang rewrites.
- Design from the user out: good defaults, clear errors, and docs are part of the feature.
- Keep the core small; push variation to config or plugins.
- Prefer explicit code over magic.

## Casey Muratori
- Compression-oriented programming: write concrete code first; extract only after repetition appears (some duplication beats the wrong abstraction).
- Don't hide what the machine is doing behind indirection.
- Treat performance as design: avoid needless allocations and layers; measure, don't guess.
- Prefer plain data and functions over deep class hierarchies.
```

Each section targets a specific failure mode:

### Writing

Terse output cuts token spend and review noise. Subject-verb openers force direct statements. Separating prototype from production stops agents from describing intentions as shipped behavior.

### Delegation

The primary agent stays on the frontier model for planning, architecture, decisions, review, and summary. Hands-on work goes to fast subagents: `explore` for read-only search, `general` for multi-step edits and commands.

Every brief carries goal, scope, expected size, gotchas, and definition of done. Parallel tasks get separate git worktrees; no shared checkout, branch, or state.

Pin the subagent models in `~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "agents": {
    "general": {
      "model": "opencode-go/deepseek-v4.1-flash#max"
    },
    "explore": {
      "model": "opencode-go/deepseek-v4.1-flash#max"
    }
  }
}
```

The primary keeps the expensive model; search and edits run cheap and parallel.

### Verification

Irreversible steps wait on passing checks. `&&` or `set -e` gates `push`, `merge`, `deploy`, and `delete`. Failures surface immediately, before damage spreads.

### Engineering principles

The last section encodes three schools: Carson Gross (cut complexity, 80/20, locality of behavior, integration tests, Chesterton's fence), Mitchell Hashimoto (small increments, design from the user out, explicit code), and Casey Muratori (compression-oriented programming, plain data and functions, performance as design). The rules give agents a default against premature abstractions and layered bloat.

## Hex: local voice input

<img src="/images/hex-voice-pipeline.svg" alt="Hex local voice dictation pipeline" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

Hex runs local voice dictation on the Vulkan iGPU: `parakeet-unified-en` for ASR, Silero VAD for speech gating, then spoken formatting and filler-word removal. Press `F8` to dictate subagent briefs and review feedback straight into the terminal or OpenCode. Audio never leaves the machine.

Default host, one container, two curl installs, one global brief. That is most of the setup.
