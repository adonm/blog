+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: An Immutable Linux Agentic Coding Setup'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

I keep [Bluefin DX](https://docs.projectbluefin.io/bluefin-dx) stock, run OpenCode and the toolchain in one Debian Distrobox, and dictate with Hex on the host. The part I reuse across projects is a short global `AGENTS.md`: delegate work, check what changed, and commit locally.

## The stack

<img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin, OpenCode, and Hex architecture" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

- **Stock Bluefin DX.** `ghcr.io/ublue-os/bluefin-dx:stable`, zero layered `rpm-ostree` packages. The host handles Wayland, PipeWire, Vulkan, Flatpak GUI apps, and atomic updates.
- **One `dev` [Distrobox](https://distrobox.it/).** Debian 13 on rootless [Podman](https://podman.io/); [mise](https://mise.jdx.dev/) supplies `rust`, `go`, `node`, `python`, and `gh`.
- **[OpenCode](https://opencode.ai/) and [Hex](https://anoma.ly/hex) ([GitHub](https://github.com/anomalyco/hex)) from curl.** Both install into `$HOME`. Hex runs on the host, OpenCode inside `dev`. No host packages, no custom image.

On the host, create and enter `dev`:

```bash
distrobox create --name dev --image docker.io/library/debian:13 --yes
distrobox enter dev
```

Inside `dev`, install the toolchain and [OpenCode V2](https://opencode.ai/docs/). The explicit paths avoid shell startup edits:

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl git
curl -fsSL https://mise.run | sh
"$HOME/.local/bin/mise" use -g rust go node python gh
curl -fsSL https://opencode.ai/v2/install | bash -s -- --no-modify-path
"$HOME/.local/bin/mise" exec -- "$HOME/.opencode/bin/opencode" --version
```

Drop `--version` to launch from `dev`. To launch from a project in a host Bash shell, use this optional function:

```bash
opencode() {
  distrobox enter dev -- "$HOME/.local/bin/mise" exec -- \
    "$HOME/.opencode/bin/opencode" "$@"
}
```

`mise exec` supplies tool paths inside `dev`; no host `mise activate` needed. If a host OpenCode service is already running, [use `opencode --standalone`](https://opencode.ai/docs/cli#background-service) with the function to start a private server in `dev`. Both sides can access projects and config in `$HOME`. That's convenient, not a security sandbox.

## Global AGENTS.md

<img src="/images/opencode-delegation-flow.svg" alt="OpenCode subagent delegation and verification flow" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

[OpenCode V2](https://opencode.ai/docs/instructions) combines `~/.config/opencode/AGENTS.md` with project instructions. I keep personal defaults here and repo-specific commands next to the code. Exact contents, 48 lines:

```markdown
# Writing

- Be terse. Cut filler, hedging, and meta-commentary.
- Lead with subject and verb, never `What ...` or `The way ...` openers (`"What X is..."` -> `"X is..."`; `"What this prototype implements"` -> `"This prototype implements"`).
- State reality directly: separate what exists now from production (`"This repo uses placeholder X now; production uses Y"`).

# Delegation

- Delegate hands-on work: `explore` for reading/search, `general` for multi-step research, edits, and commands. Keep the primary agent for planning, decisions, review, and summary.
- Work directly only on trivial single steps. Put your context into the subagent brief instead of doing the work yourself.
- Subagents start fresh: state goal, scope, expected size, gotchas, and definition of done. Tell them to stop and report rather than expand scope.
- Parallelize independent work. Read-only agents may share a stable checkout; give concurrent writers separate worktrees and isolate other mutable resources.
- Check tool output before rerunning when results look missing. Spot-check key subagent claims.

# Verification

- Exercise changed behavior with the smallest relevant test or runtime check. Report what passed, failed, or wasn't run.
- Turn recurring bugs and agent mistakes into deterministic checks (types, lint rules with clear fix messages, tests) instead of adding prose or comments.
- Require passing checks before an authorized push, merge, deploy, or destructive action. Use `&&` when chaining; report failures immediately.

# Commits & PRs

- Commit locally as you implement, unless asked not to. Stage and commit only your edits for the task. Make small, logical commits in review order; each must build and pass relevant checks.
- Use `<ticket>: <imperative summary>` when the ticket is known; otherwise omit the prefix. Explain why and note non-obvious constraints in the body.
- Lead PR descriptions with who or what is affected, where to start reviewing, and which checks passed, failed, or weren't run.
- Push, open or merge PRs, squash commits, or rewrite pushed history only when explicitly asked.

# Engineering principles

- Keep the core small: cut unrequested features and abstractions; offer the 80/20 solution.
- Keep behavior local (locality of behavior) rather than split across layers.
- Prefer boring, proven tech and the simplest stack that fits.
- Favor integration tests over brittle unit tests; add a regression test for every bug fix.
- Name intermediate values over clever one-liners; optimize for debuggability.
- Chesterton's fence: understand why code exists before removing it.
- Match repo conventions, not repo tech debt: don't copy deprecated patterns or hacks into new code.
- Fix root causes instead of patching symptoms with timers, retries, casts, or lint suppressions; never use comments or docs to justify a workaround (comment only non-obvious invariants and external constraints).
- Make it work, then make it good: ship small, demoable increments, not big-bang rewrites.
- Design from the user out: good defaults, clear errors, and docs are part of the feature.
- Compression-oriented programming: write concrete code first; extract only after repetition appears (some duplication beats the wrong abstraction).
- Prefer explicit code over magic and indirection; don't hide what the machine is doing.
- Treat performance as design: avoid needless allocations and layers; measure, don't guess.
- Prefer plain data and functions over deep class hierarchies.

# Web & reports

- Query `modern-web-guidance` (`search` then `retrieve`) first on HTML, CSS, and client-JS tasks instead of relying on training-weight patterns.
- Use the repo's existing UI stack. Default to [Oat](https://oat.ink) (`@knadh/oat`) for standalone reports and new UIs without an established stack. Prefer semantic HTML and accessible inline SVG.
```

These are preferences, not guarantees; checks in the repo do the enforcing. Keep tool-specific lines only for tools you have: `modern-web-guidance` is an installed skill here.

### Context window hygiene

`grep` output, whole-file reads, and dead ends crowd out the task; disposable subagents absorb that, leaving the primary window for planning, review, and summary.

### Parallelism without collisions

Read-only `explore` agents can share a stable checkout. Concurrent writers get separate [git worktrees](https://git-scm.com/docs/git-worktree) so they don't collide in the working tree or index. Isolate other shared resources, such as ports and databases, separately.

### Cost and latency split

`~/.config/opencode/opencode.jsonc` routes `explore` and `general` to a fast model without changing the primary model. Adapt this example to a model your provider offers in `/models`:

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

The split spends frontier tokens on decisions, not typing. Credentials belong in the provider's login flow.

### Verification as a circuit breaker

Exercise what changed: run a regression test, try the CLI command, or walk the UI flow. Turn recurring mistakes into types, lint rules, or tests instead of adding prose. Lauren Tan's [All Code Is Slop (Except Mine)](https://youtu.be/O_zcc6Xl12g) makes the case for fixing the code or the checks first.

Publishing is separate. The agent commits locally as it works, in small commits that each pass their checks, but does not `push`, open a PR, or deploy until explicitly asked. Here a push to `main` triggers Pages.

### Prior art as compressed judgment

The `# Engineering principles` section names mental models instead of giving generic advice. [The Grug Brained Developer](https://grugbrain.dev/) and [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/) push against indirection. [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) blocks deletion without understanding. [Mitchell Hashimoto](https://mitchellh.com/writing) favors small increments and explicit code. [Casey Muratori](https://caseymuratori.com/)'s [Semantic Compression](https://caseymuratori.com/blog_0015) favors concrete code, plain data, and measured performance.

## Hex: local voice input

<img src="/images/hex-voice-pipeline.svg" alt="Hex local voice dictation pipeline" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

Install [Hex](https://anoma.ly/hex) on the **host** and open it from the app launcher:

```bash
curl -fsSL https://anoma.ly/hex/files/install-linux.sh | sh
```

My setup uses [Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) on the Vulkan iGPU and [Silero VAD](https://github.com/snakers4/silero-vad) to detect speech, with spoken formatting and filler-word removal enabled in [Labs](https://github.com/anomalyco/hex/blob/dev/docs/features/labs.md). I use `F8`; [Hex defaults to `Ctrl+Alt+Space` on GNOME](https://github.com/anomalyco/hex/blob/dev/docs/linux.md#shortcuts). Transcription runs on-device. Optional rewriting in Hex or submitting the dictated text to a remote OpenCode model can send the text off-device.
