+++
date = '2026-11-07'
title = 'Bluefin, OpenCode, and Hex: An Immutable Linux Agentic Coding Setup'
tags = ["linux", "bluefin", "opencode", "hex", "ai", "wayland"]
+++

Default Bluefin DX plus a disposable Debian container covers this workstation. The host stays immutable. The container absorbs churn. OpenCode and Hex install per-user with curl. A global `AGENTS.md` keeps agent work predictable.

## Architecture

<img src="/images/bluefin-opencode-hex-architecture.svg" alt="Bluefin, OpenCode, and Hex architecture" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

Three layers:

- **Host**: Bluefin DX owns the hardware, Wayland compositor, PipeWire, Vulkan, and GUI apps.
- **Bridge**: one wrapper scrubs AppImage state, then enters `dev` or starts a transient scope.
- **Container**: `dev` Distrobox runs `opencode-bin serve` with `mise` toolchains.

## Bluefin DX and the host-container boundary

`ghcr.io/ublue-os/bluefin-dx:stable` runs unchanged, with zero layered `rpm-ostree` packages. Flatpaks, AppImages, GNOME Wayland, PipeWire, and Vulkan stay on the host. Updates arrive atomically; nothing in `/usr` drifts.

The `dev` Distrobox runs Debian on rootless Podman. `mise` supplies Rust, Go, Node, Python, and `gh`. Rebuild the container from config without touching the host.

OpenCode and Hex install with curl scripts into user directories. OpenCode lands in `~/.opencode`; Hex lands in `~/.local/share/hex` with a `~/.local/bin/hex` symlink. Both sit on `PATH`; neither needs `rpm-ostree`.

## Bridging OpenCode across host and Distrobox

`OpenCode.AppImage` renders the desktop UI on the host. `opencode-bin serve` runs inside `dev`. The AppImage exports `LD_LIBRARY_PATH`, `APPIMAGE`, `APPDIR`, `ARGV0`, `OWD`, and `/tmp/.mount_*` entries in `XDG_DATA_DIRS`. Those paths do not exist inside the container and break the loader.

`~/.opencode/bin/opencode` scrubs the environment, then crosses the boundary:

```bash
#!/usr/bin/env bash
set -euo pipefail

REAL_BIN="$HOME/.opencode/bin/opencode-bin"

# Already inside the container: exec directly.
if [ -n "${CONTAINER_ID:-}" ] || [ -f /run/.containerenv ]; then
  exec "$REAL_BIN" "$@"
fi

unset LD_LIBRARY_PATH APPIMAGE APPDIR ARGV0 OWD

if [ -n "${XDG_DATA_DIRS:-}" ]; then
  XDG_DATA_DIRS="$(printf '%s' "$XDG_DATA_DIRS" \
    | tr ':' '\n' \
    | { grep -v '^/tmp/\.mount_' || true; } \
    | paste -sd: -)"
  export XDG_DATA_DIRS
fi

if [ -t 0 ] && [ -t 1 ]; then
  # Interactive shell: attach the TTY.
  exec distrobox-enter dev -- /bin/bash -lc 'exec "$0" "$@"' "$REAL_BIN" "$@"
else
  # Desktop app or service: transient user scope, no TTY.
  exec systemd-run --user --scope --quiet \
    distrobox-enter -T dev -- /bin/bash -lc 'exec "$0" "$@"' "$REAL_BIN" "$@"
fi
```

A `.bashrc` helper bypasses the AppImage environment for plain host shells:

```bash
export PATH="$HOME/.opencode/bin:$PATH"
if [ -z "${CONTAINER_ID:-}" ]; then
  opencode() { distrobox enter dev -- opencode "$@"; }
fi
```

`~/.config/opencode/opencode.jsonc` routes subagents to the fast model and leaves compaction at defaults:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  // Compaction is left at OpenCode defaults. The previous 666k input cap
  // and 15k keep window forced Cursor sessions to compact far too often.
  "plugins": [
    "cursor-opencode-provider"
  ],
  "agents": {
    "general": {
      "model": "opencode-go/deepseek-v4.1-flash#max",
    },
    "explore": {
      "model": "opencode-go/deepseek-v4.1-flash#max",
    },
  },
}
```

`~/.config/opencode/cli.json` holds TUI-only preferences:

```json
{
  "debug": {
    "turn_tokens": false,
    "devtools": false
  },
  "session": {
    "thinking": "show",
    "sidebar": "hide",
    "new_location": "launch",
    "permissions": "autoaccept",
    "verbosity": "high"
  },
  "prompt": {
    "paste": "compact"
  },
  "terminal": {
    "copy": "select"
  },
  "theme": {
    "name": "opencode"
  },
  "animations": true,
  "mouse": true
}
```

## Global AGENTS.md

<img src="/images/opencode-delegation-flow.svg" alt="OpenCode subagent delegation and verification flow" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

`~/.config/opencode/AGENTS.md` applies to every project. Exact contents:

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

Four pillars hold the workflow together:

- **Writing**: terse output, subject-verb openers, explicit separation of prototype and production.
- **Delegation**: the primary agent plans and reviews. `explore` searches and reads. `general` handles multi-step edits and commands in isolated git worktrees. Briefs carry goal, scope, expected size, gotchas, and done criteria.
- **Verification**: irreversible commands wait on checks behind `&&` or `set -e`.
- **Engineering principles**: Carson Gross cuts complexity, Mitchell Hashimoto ships small increments, and Casey Muratori keeps the machine visible.

The primary agent spends frontier-model tokens on judgment. Fast subagents burn cycles on search and edits. The verification gate decides what reaches `main`.

## Hex: local GPU voice dictation on Wayland

<img src="/images/hex-voice-pipeline.svg" alt="Hex local voice dictation pipeline" style="width: 100%; max-width: 100%; margin: 1.5rem 0;">

Hex (`anomaly.hex`) installs entirely under `~/.local/share/hex`. The launcher is a symlink:

```text
~/.local/share/hex/current -> versions/3.3.0
~/.local/bin/hex -> ~/.local/share/hex/current/hex
```

No `rpm-ostree` layering. Updates replace the version directory.

`~/.config/systemd/user/hex.service` supervises the background service. Paths use `%h` for the home directory:

```ini
[Unit]
Description=Hex background dictation
PartOf=graphical-session.target
After=graphical-session.target
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=notify
NotifyAccess=main
ExecStart=%h/.local/bin/hex service
Environment="HEX_APPLICATION_SUPPORT_DIR=%h/.local/share/hex"
EnvironmentFile=-%h/.local/share/hex/session.env
Restart=on-failure
RestartSec=3
TimeoutStopSec=30
UMask=0077

[Install]
WantedBy=graphical-session.target
```

Hex also ships an XDG autostart entry as a fallback:

```ini
[Desktop Entry]
Type=Application
Name=Hex
X-XDP-Autostart=anomaly.hex
Exec=sh -c "exec \"$HOME/.local/share/hex/current/hex\" start"
```

The model stack runs on the Vulkan iGPU:

- `parakeet-unified-en-0.6b-Q8_0.gguf` for streaming ASR.
- `silero-vad-v6.2-F32.gguf` for voice activity detection.
- Audio ducking to 20% while recording.
- Labs features: `live-text`, `spoken-formatting`, `filler-words`, `spoken-symbols`, `standalone-special-keys`, `voice-filter`.

Dictation binds through `org.freedesktop.portal.GlobalShortcuts`: `F8` dictate, `F9` annotate, `Ctrl+Alt+V` paste last. Portal bindings can drop after suspend or a compositor restart. `hex dictate` drives the same pipeline from a terminal and avoids the portal.

Default Bluefin, one container, two curl installs, one global brief. The host stays clean and the agent work stays reviewable.
