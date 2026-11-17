+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: An Immutable Linux Agentic Coding Setup'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

Default [Bluefin](https://projectbluefin.io/) keeps the host immutable. One `dev` container absorbs toolchain churn. OpenCode and Hex install into `$HOME`, which carries the config. The highest-leverage piece is a short global `AGENTS.md`: terse writing, delegated work, the smallest useful check, small local commits. It biases behavior; it does not enforce it.

## The stack

<img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin, OpenCode, and Hex architecture" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

- **Default [Bluefin](https://projectbluefin.io/).** `ghcr.io/ublue-os/bluefin-dx:stable`, zero layered `rpm-ostree` packages. The host keeps Wayland, PipeWire, Vulkan, Flatpak GUI apps, and atomic updates.
- **One `dev` [Distrobox](https://distrobox.it/).** Debian 13 on rootless [Podman](https://podman.io/); [mise](https://mise.jdx.dev/) supplies `rust`, `go`, `node`, `python`, and `gh`.
- **[OpenCode](https://opencode.ai/) and [Hex](https://anoma.ly/hex) ([GitHub](https://github.com/anomalyco/hex)) from curl.** Both install into `$HOME`. Hex runs on the host, OpenCode inside `dev`. No host packages, no custom image.

Bootstrap from the host; `distrobox` needs no layering on Bluefin DX:

```bash
distrobox create --name dev --image docker.io/library/debian:13
distrobox enter dev -- sudo apt-get update
distrobox enter dev -- sudo apt-get install -y curl ca-certificates git
distrobox enter dev -- sh -c 'curl https://mise.run | sh'
distrobox enter dev -- "$HOME/.local/bin/mise" use -g rust go node python gh
curl -fsSL https://opencode.ai/v2/install | bash
curl -fsSL https://anoma.ly/hex/files/install-linux.sh | sh
```

`distrobox create` prompts before pulling an absent image; `--yes` makes that unattended. OpenCode lands in `$HOME/.opencode/bin/opencode`; add `bash -s -- --no-modify-path` to skip rc edits.

For host one-offs, skip the guest login shell. This helper pins both paths, and assumes `~/.local/bin` is on `PATH`:

```bash
mkdir -p "$HOME/.local/bin"
cat > "$HOME/.local/bin/oc" <<'EOF'
#!/usr/bin/env bash
exec distrobox enter dev -- "$HOME/.local/bin/mise" exec -- "$HOME/.opencode/bin/opencode" "$@"
EOF
chmod +x "$HOME/.local/bin/oc"
oc --version
```

Distrobox forwards the caller's `PATH` but noninteractive shells still miss guest paths, so the helper pins both binaries: no host `mise activate`, no profile edits. `$HOME` is the contract: `~/.config/opencode`, `~/.config/mise`, `~/.local/share/hex`, and [git worktrees](https://git-scm.com/docs/git-worktree) work on both sides. Shared home is convenient, not a sandbox.

## Global AGENTS.md

<img src="/images/opencode-delegation-flow.svg" alt="OpenCode subagent delegation and verification flow" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

`~/.config/opencode/AGENTS.md` is the centerpiece. [OpenCode V2](https://opencode.ai/docs/instructions) layers global instructions before project ones, so this sets defaults everywhere and a repo's own `AGENTS.md` refines them. Exact contents, 48 lines:

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

`explore` never writes, so read-only agents can share a stable checkout. Writers cannot: concurrent edits corrupt each other's index and unstaged files. Give each writer its own [git worktree](https://git-scm.com/docs/git-worktree) over the same object store, and isolate other shared resources like ports or databases.

### Cost and latency split

`~/.config/opencode/opencode.jsonc` sends `explore` and `general` to a fast model while the primary keeps a frontier one. This `agents` key is an example, not a settled catalog; confirm IDs with `/models` and your provider.

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

Run the smallest check that exercises the change: one test file, one type check, one `--version`. Recurring mistakes become deterministic checks, not paragraphs: a type, a lint rule with a clear message, a regression test. [All Code Is Slop (Except Mine)](https://youtu.be/O_zcc6Xl12g) argues taste does not scale; a CI check does.

Publishing is separate. The agent commits locally as it works, in small commits that each pass their checks, but does not `push`, open a PR, or deploy until explicitly asked. Here a push to `main` triggers Pages.

### Prior art as compressed judgment

The `# Engineering principles` section names mental models instead of giving generic advice. [The Grug Brained Developer](https://grugbrain.dev/) and [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/) push against indirection. [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence) blocks deletion without understanding. [Mitchell Hashimoto](https://mitchellh.com/writing) favors small increments and explicit code. [Casey Muratori](https://caseymuratori.com/)'s [Semantic Compression](https://caseymuratori.com/blog_0015) favors concrete code, plain data, and measured performance.

## Hex: local voice input

<img src="/images/hex-voice-pipeline.svg" alt="Hex local voice dictation pipeline" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

[Hex](https://anoma.ly/hex) runs local voice dictation on the Vulkan iGPU: [Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) for ASR, [Silero VAD](https://github.com/snakers4/silero-vad) for speech gating, then optional spoken formatting and filler-word removal via the [Labs](https://github.com/anomalyco/hex/blob/dev/docs/features/labs.md) toggles, which I enabled. `F8` is my binding; GNOME's is `Ctrl+Alt+Space`. Audio is transcribed on-device, but text can still leave: optional Output/OpenCode rewriting sends transcripts to its model, and prompted text goes to whichever provider OpenCode uses. Both are explicit choices. See the [Linux docs](https://github.com/anomalyco/hex/blob/dev/docs/linux.md).

If a shared background service runs with different config and you must bypass it, the [CLI docs](https://opencode.ai/docs/cli#background-service) cover `opencode --standalone`.

The default host, one container, two app installs, and one short global brief cover most of the setup.
