+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: One Container, One Brief'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

I keep [Bluefin DX](https://docs.projectbluefin.io/bluefin-dx) stock so changing toolchains doesn't touch the system image. OpenCode runs in one Debian container; Hex takes dictation on the desktop. A 48-line `AGENTS.md` sets the defaults for the work itself: check the change, commit locally, and publish only when asked.

## 1. Keep the host boring

The host runs `ghcr.io/ublue-os/bluefin-dx:stable` with no layered `rpm-ostree` packages. Bluefin handles the Wayland desktop, PipeWire, Vulkan, Flatpak apps, and atomic updates. Hex runs there, next to the apps it types into.

One [Distrobox](https://distrobox.it/) named `dev` runs Debian 13 on rootless [Podman](https://podman.io/). I install [OpenCode](https://opencode.ai/) and [mise](https://mise.jdx.dev/) there; mise supplies Rust, Go, Node, Python, and `gh`.

<figure class="bluefin-figure">
  <img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin DX and Hex run on the host, Debian dev runs OpenCode and mise toolchains, and both environments connect to one shared home directory that is not a sandbox." width="560" height="480">
  <figcaption>Stock host, mutable container, one shared home. The arrows show convenience, not isolation.</figcaption>
</figure>

Create and enter it from the host:

```bash
distrobox create --name dev --image docker.io/library/debian:13 --yes
distrobox enter dev
```

Inside `dev`, install the toolchain and [OpenCode V2](https://opencode.ai/docs/). Explicit paths keep shell startup files untouched:

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl git
curl -fsSL https://mise.run | sh
"$HOME/.local/bin/mise" use -g rust go node python gh
curl -fsSL https://opencode.ai/v2/install | bash -s -- --no-modify-path
"$HOME/.local/bin/mise" exec -- "$HOME/.opencode/bin/opencode" --version
```

Drop `--version` to launch. To start from a project in a host Bash shell, add a function:

```bash
opencode() {
  distrobox enter dev -- "$HOME/.local/bin/mise" exec -- \
    "$HOME/.opencode/bin/opencode" "$@"
}
```

`mise exec` supplies tool paths inside `dev`; no host `mise activate` needed. If a host OpenCode service is already running, [pass `--standalone`](https://opencode.ai/docs/cli#background-service) for a private server in `dev`.

Both sides see the same `$HOME`, so projects and config don't need copying. That isn't a sandbox: `dev` can still write to your files there.

## 2. Keep the brief short

[OpenCode V2](https://opencode.ai/docs/instructions) reads my global `~/.config/opencode/AGENTS.md` alongside project instructions. I keep personal defaults in the global file and repo-specific commands next to the code.

<figure class="bluefin-figure">
  <img src="/images/opencode-delegation-flow.svg" alt="Three steps in order: brief and delegate, exercise the change, commit locally. A separate dashed callout holds publishing back until someone asks." width="560" height="544" loading="lazy">
  <figcaption>Passing checks is a condition for committing, not permission to publish.</figcaption>
</figure>

- **Brief before delegating.** Give subagents the goal, scope, gotchas, and definition of done. They don't inherit the primary session's history.
- **Exercise the change.** Run the smallest real check: a regression test, the CLI command, the UI path. Report what passed, failed, or did not run.
- **Commit locally by default.** Small commits in review order, each passing its checks.
- **Publish only on request.** Pushing, opening or merging PRs, and deploying need an explicit yes. A push to this blog's `main` publishes the site.

If a mistake repeats, add a type constraint, lint rule, or regression test instead of another line of prose. Lauren Tan's [All Code Is Slop (Except Mine)](https://youtu.be/O_zcc6Xl12g) makes the case for fixing the code or checks first. Prose sets expectations; checks enforce them.

Keep tool-specific rules only for tools you use. `modern-web-guidance`, for example, is an installed skill here.

<details class="bluefin-details">
<summary>Read the exact 48-line <code>AGENTS.md</code></summary>

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

</details>

### Keep the primary session focused

Subagents absorb search output and dead ends so the primary session has room for planning and review. Read-only `explore` agents can share a stable checkout. Give concurrent writers separate [git worktrees](https://git-scm.com/docs/git-worktree), and give them separate ports or databases when they need them.

### Pick a cheaper model for subagents

This `opencode.jsonc` example sets a fast model for `explore` and `general` without changing the primary model. Put it in the global config directory and choose a model and variant available in `/models`:

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

**Prior art.** [Grug](https://grugbrain.dev/) and [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/) favor simplicity and nearby code. [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) says to understand a rule before removing it. [Mitchell Hashimoto](https://mitchellh.com/writing) favors small increments, and Casey Muratori's [Semantic Compression](https://caseymuratori.com/blog_0015) starts with concrete code.

## 3. Keep the audio local

<figure class="bluefin-figure">
  <img src="/images/hex-voice-pipeline.svg" alt="Hex transcribes microphone audio on the device. Hex can paste the transcript into the focused app; only text can go to a remote model if you rewrite or submit it." width="560" height="656" loading="lazy">
  <figcaption>Local dictation and remote model use are separate: the microphone feeds Hex, not a model provider.</figcaption>
</figure>

Install [Hex](https://anoma.ly/hex) on the **host**, then open it from the app launcher:

```bash
curl -fsSL https://anoma.ly/hex/files/install-linux.sh | sh
```

My setup runs [Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) on the Vulkan iGPU and uses [Silero VAD](https://github.com/snakers4/silero-vad) to detect speech. I turn on spoken formatting and filler-word removal in [Labs](https://github.com/anomalyco/hex/blob/dev/docs/features/labs.md). My shortcut is `F8`; [Hex defaults to `Ctrl+Alt+Space` on GNOME](https://github.com/anomalyco/hex/blob/dev/docs/linux.md#shortcuts).

Hex transcribes on-device. Rewriting a dictation with a remote model, or submitting it to a remote model in OpenCode, can send the transcript, not the audio. I treat a dictated prompt the same way I treat a typed one: I decide when to send it.
