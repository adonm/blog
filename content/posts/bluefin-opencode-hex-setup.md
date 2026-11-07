+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: An Immutable Linux Agentic Coding Setup'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

Default [Bluefin](https://projectbluefin.io/) keeps the host immutable. One `dev` container absorbs toolchain churn. OpenCode and Hex land in `$HOME`, which carries the configuration. The highest-leverage piece is a global `AGENTS.md` that tells every agent how to write, delegate, verify, and choose the simplest design.

## The stack

<img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin, OpenCode, and Hex architecture" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

- **Default [Bluefin](https://projectbluefin.io/).** `ghcr.io/ublue-os/bluefin-dx:stable` with zero layered `rpm-ostree` packages. The host keeps Wayland, PipeWire, Vulkan, Flatpak GUI apps, and atomic updates.
- **One `dev` [Distrobox](https://distrobox.it/).** Debian on rootless [Podman](https://podman.io/). [mise](https://mise.jdx.dev/) manages `rust`, `go`, `node`, `python`, and `gh`; the container handles mutable dependencies.
- **[OpenCode](https://opencode.ai/) and [Hex](https://anoma.ly/hex) ([GitHub](https://github.com/anomalyco/hex)) from curl.** Both install into `$HOME`, so the host and container share the same binaries and config. No host packages, no custom image.

Bootstrap the whole stack from the host:

```bash
distrobox create --name dev --image debian:13
curl https://mise.run | sh
mise use -g rust go node python gh
curl -fsSL https://opencode.ai/install | bash
curl -fsSL https://anoma.ly/hex/install-linux.sh | sh
```

Host shells need one helper:

```bash
eval "$(mise activate bash)"
opencode() { distrobox enter dev -- opencode "$@"; }
```

`$HOME` is the contract: `~/.config/opencode`, `~/.config/mise`, `~/.local/share/hex`, and [git worktrees](https://git-scm.com/docs/git-worktree) are visible on both sides.

## Global AGENTS.md

<img src="/images/opencode-delegation-flow.svg" alt="OpenCode subagent delegation and verification flow" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

`~/.config/opencode/AGENTS.md` is the centerpiece. [OpenCode](https://opencode.ai/) reads it for every project and every subagent; see the [agents and config docs](https://opencode.ai/docs/). Exact contents:

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

These rules encode mechanics and economics:

### Context window hygiene

Exploration pollutes context fast: `grep` output, whole-file reads, and dead ends crowd out the task. The primary agent keeps the frontier model and spends its window on planning, decisions, review, and summary. Disposable `explore` and `general` subagents absorb search and edits, then return compact reports. Terse writing and briefs keep the main session sharp across long tasks.

### Parallelism without collisions

Parallel subagents on one working tree corrupt each other's index and unstaged files. A separate [git worktree](https://git-scm.com/docs/git-worktree) per task gives each subagent an isolated directory backed by the same `.git` object store. Edits stay parallel; merges stay explicit.

### Cost and latency split

`~/.config/opencode/opencode.jsonc` pins `explore` and `general` to a fast, cheap model while the primary session stays on a frontier model. Search and edits run at low latency and low cost; judgment stays with the expensive model.

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

The split spends frontier tokens on decisions, not typing.

### Verification as a circuit breaker

Irreversible steps carry the highest blast radius. `&&` or `set -e` gates `push`, `merge`, `deploy`, and `delete`; one failed check stops the chain before damage spreads.

### Prior art as compressed judgment

The `# Engineering principles` section names specific mental models instead of generic advice. [The Grug Brained Developer](https://grugbrain.dev/) by Carson Gross and [Locality of Behavior](https://htmx.org/essays/locality-of-behavior/) push against layered indirection. [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) blocks deletion without understanding. [Mitchell Hashimoto's writing](https://mitchellh.com/writing) favors small increments and explicit code. [Casey Muratori](https://caseymuratori.com/)'s [Semantic Compression](https://caseymuratori.com/blog_0015) favors concrete code, plain data, and measured performance. Naming these models steers agents away from enterprise wrapper classes, premature abstractions, and unrequested features.

## Hex: local voice input

<img src="/images/hex-voice-pipeline.svg" alt="Hex local voice dictation pipeline" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

[Hex](https://anoma.ly/hex) runs local voice dictation on the Vulkan iGPU: [Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) for ASR, [Silero VAD](https://github.com/snakers4/silero-vad) for speech gating, then spoken formatting and filler-word removal. Press `F8` to dictate subagent briefs and review feedback straight into the terminal or [OpenCode](https://opencode.ai/). Audio never leaves the machine.

The default host, one container, two app installs, and one global brief cover most of the setup.
