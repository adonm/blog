+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: One Container, One Brief'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

I'm good at turning “install a compiler” into “fiddle with my Linux setup.” I do the fiddling in one Debian container now and leave [Bluefin DX](https://docs.projectbluefin.io/bluefin-dx) stock. Hex stays on the desktop. OpenCode gets a 48-line `AGENTS.md`, which is short for me.

## 1. Keep the host boring

I run `ghcr.io/ublue-os/bluefin-dx:stable` without layering any `rpm-ostree` packages. Bluefin handles the Wayland desktop, PipeWire, Vulkan, Flatpaks, and atomic updates.

I called my [Distrobox](https://distrobox.it/) `dev`. It's Debian 13 on rootless [Podman](https://podman.io/). [OpenCode](https://opencode.ai/) runs there, and [mise](https://mise.jdx.dev/) supplies Rust, Go, Node, Python, and `gh`.

<figure class="bluefin-figure">
  <img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin DX and Hex run on the host, Debian dev runs OpenCode and mise toolchains, and both environments connect to one shared home directory that is not a sandbox." width="560" height="480">
  <figcaption>The host and container share a home directory. Either can change those files.</figcaption>
</figure>

Create `dev` on the host, then enter it:

```bash
distrobox create --name dev --image docker.io/library/debian:13 --yes
distrobox enter dev
```

Install the toolchain and [OpenCode V2](https://opencode.ai/docs/) inside `dev`. I use full paths so I can leave `.bashrc` alone:

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl git
curl -fsSL https://mise.run | sh
"$HOME/.local/bin/mise" use -g rust go node python gh
curl -fsSL https://opencode.ai/v2/install | bash -s -- --no-modify-path
"$HOME/.local/bin/mise" exec -- "$HOME/.opencode/bin/opencode" --version
```

Drop `--version` to launch. On the host, I use this Bash function to start OpenCode in `dev`:

```bash
opencode() {
  distrobox enter dev -- "$HOME/.local/bin/mise" exec -- \
    "$HOME/.opencode/bin/opencode" "$@"
}
```

`mise exec` puts the tools on `PATH` in `dev`. I don't need `mise activate` on the host. If an OpenCode service is already running there, I [pass `--standalone`](https://opencode.ai/docs/cli#background-service) to start a private server in `dev`.

Distrobox shares `$HOME`. I can use the same checkout from either shell. I can wreck it from either shell, too. This isn't a sandbox.

## 2. Keep the brief short

[OpenCode V2](https://opencode.ai/docs/instructions) reads `~/.config/opencode/AGENTS.md` alongside the project's instructions. I put my habits in the global file and leave build commands in the repo that uses them.

<figure class="bluefin-figure">
  <img src="/images/opencode-delegation-flow.svg" alt="Three steps in order: brief and delegate, exercise the change, commit locally. A separate dashed callout holds publishing back until someone asks." width="560" height="544" loading="lazy">
  <figcaption>I still have to say yes to a push.</figcaption>
</figure>

The subagent didn't sit through the conversation that got me here. I write down what to do, what not to touch, and how I'll know it's done. Sometimes that's when I find out I don't know what I want yet.

Then I want a check that could actually fail. Run the test, try the command, or open the page. Tell me what passed, failed, or didn't run. Make small local commits once the checks pass.

No pushing, opening or merging PRs, or deploying until I ask. A push to this blog's `main` publishes it. If I'm going to publish a sentence I regret, I'd like to do it myself.

If I keep catching the same bug, a test has a better shot than line 49 of `AGENTS.md`. A type or lint rule will do, too. Lauren Tan makes the case in [All Code Is Slop (Except Mine)](https://youtu.be/O_zcc6Xl12g).

The `modern-web-guidance` line is for a skill I have installed. Leave it out if you don't.

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

I send `explore` to look things up. I want the answer in the main session, not all the `grep` output. Readers can share a stable checkout. Writers get separate [git worktrees](https://git-scm.com/docs/git-worktree), plus their own ports or databases if they'd otherwise collide.

### Pick a cheaper model for subagents

I don't need my priciest model to find a file. Here's how to give the subagents a cheaper one in the global `opencode.jsonc`. Pick a model and variant you see in `/models`:

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

**Prior art.** I borrowed most of it: [Carson Gross's Grug](https://grugbrain.dev/), [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/), [Chesterton's fence](https://en.wikipedia.org/wiki/G._K._Chesterton#Chesterton's_fence), [Mitchell Hashimoto](https://mitchellh.com/writing) on small increments, and Casey Muratori's [Semantic Compression](https://caseymuratori.com/blog_0015) on starting with concrete code.

## 3. Keep the audio local

<figure class="bluefin-figure">
  <img src="/images/hex-voice-pipeline.svg" alt="Hex transcribes microphone audio on the device. Hex can paste the transcript into the focused app; only text can go to a remote model if you rewrite or submit it." width="560" height="656" loading="lazy">
  <figcaption>Hex does the listening. A remote model only gets text.</figcaption>
</figure>

Install [Hex](https://anoma.ly/hex) on the **host**, then open it from the app launcher:

```bash
curl -fsSL https://anoma.ly/hex/files/install-linux.sh | sh
```

[Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) runs on my Vulkan iGPU. [Silero VAD](https://github.com/snakers4/silero-vad) detects speech, and I use [Labs](https://github.com/anomalyco/hex/blob/dev/docs/features/labs.md) for spoken formatting and filler-word removal. I bound `F8`; [Hex defaults to `Ctrl+Alt+Space` on GNOME](https://github.com/anomalyco/hex/blob/dev/docs/linux.md#shortcuts).

Hex transcribes on-device. The audio stays here. If I use a remote model to rewrite a dictation or submit it to one in OpenCode, that model gets the transcript.

Dictation makes it easier to write a long prompt. I'm trying not to take that as encouragement.
