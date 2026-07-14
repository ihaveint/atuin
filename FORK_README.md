# This fork: "Copy as LLM prompt" for Atuin history search

This is a personal fork of [atuin](https://github.com/atuinsh/atuin) that adds one
extra action to the `Ctrl+R` history search TUI: instead of only being able to
copy a bare command to your clipboard, you can copy the command **plus the
directory it ran in, plus its captured output**, pre-formatted as a prompt
ready to paste into an LLM/coding agent (Devin, Claude Code, Cursor, etc).

Example of what lands on your clipboard:

    I ran this command in `/Users/me/project`:

    ```
    npm run build
    ```

    and got this output:

    ```
    Error TS2345: Argument of type 'string' is not assignable...
    ```

    Can you investigate?

## What changed vs. upstream

- New keybinding **`Alt+Y`** (alongside the existing `Ctrl+Y` = "copy command
  only") in the search TUI, bound to a new `copy-prompt` action.
- On press, it looks up the selected history entry's captured output from the
  Atuin daemon (via the existing `SemanticClient::command_output` RPC that
  Atuin AI already uses internally) and formats it into a ready-to-paste
  prompt, then copies that to your system clipboard.
- If no output was captured for that command (e.g. it ran before you enabled
  capture, or the daemon isn't running), it falls back to just the command +
  cwd with a note that output wasn't available.

Diff is small and self-contained:

- `crates/atuin/src/command/client/search/keybindings/actions.rs` — new
  `Action::CopyPrompt` action + string (de)serialization.
- `crates/atuin/src/command/client/search/keybindings/defaults.rs` — default
  binding: `alt-y` -> `copy-prompt`.
- `crates/atuin/src/command/client/search/interactive.rs` — `InputAction::CopyPrompt`,
  `build_command_prompt()`, `fetch_command_output()`.

## Setup

Output capture requires two things Atuin ships but doesn't enable by default:
the **daemon** (to store recent output in memory) and **pty-proxy** (to
capture it from your terminal via OSC 133 sequences). Without both, `Alt+Y`
still works but will just omit the "output" section of the prompt.

### 1. Build and install this fork

```sh
git clone https://github.com/ihaveint/atuin.git
cd atuin
cargo build --release -p atuin --bin atuin
cp target/release/atuin ~/.atuin/bin/atuin   # or wherever your atuin binary lives
```

On macOS, a locally-built binary may need an ad-hoc code signature to run
without being killed by Gatekeeper:

```sh
codesign --sign - --force ~/.atuin/bin/atuin
```

> **Heads up:** Atuin has a built-in self-updater. If it runs, it will
> overwrite this binary with a stock upstream release, and `Alt+Y` will stop
> being bound. If it disappears, just rebuild/reinstall from this repo. You
> may want to set `update_check = false` in your config to avoid this.

### 2. Enable the daemon

In `~/.config/atuin/config.toml`:

```toml
[daemon]
enabled = true
autostart = true
```

### 3. Enable pty-proxy

Add this **before** your existing `atuin init <shell>` line in your shell's
init file (`~/.zshrc`, `~/.bashrc`, etc):

```sh
eval "$(atuin pty-proxy init zsh)"   # or bash / fish / nu — see below
eval "$(atuin init zsh)"
```

- fish: `atuin pty-proxy init fish | source` (before your `atuin init fish |
  source` line)
- nu: see `atuin pty-proxy init nu`'s own instructions (needs a file on disk,
  since nu can't `source` a dynamic string)

Restart your shell (open a new terminal) for this to take effect. Output is
only captured for commands run **after** this is active — nothing retroactive.

### 4. (macOS only, if relevant) Fix an inherited root-owned `$TMPDIR`

Some environments start shells with `$TMPDIR` pointing at a directory owned
by `root` instead of the current user. `atuin pty-proxy` binds a local control
socket under `$TMPDIR` and will fail with `Permission denied` if it isn't
writable. If you hit that, add this near the top of your shell init file,
before the `pty-proxy` line:

```sh
if [[ ! -w "${TMPDIR:-/tmp}" ]]; then
  export TMPDIR="$(getconf DARWIN_USER_TEMP_DIR 2>/dev/null || echo /tmp)"
fi
```

## Usage

1. Run some command.
2. Open history search (`Ctrl+R`).
3. Find/select the command.
4. Press **`Alt+Y`**. The formatted prompt is now on your clipboard — paste
   it into your coding agent of choice.

`Ctrl+Y` still works as before (copies just the bare command).

## Known limitations

- Output capture isn't retroactive — only commands run while pty-proxy was
  active in that shell session are captured.
- Clipboard is local-machine only (`arboard`) — no OSC 52 fallback yet, so
  this won't reach your local machine's clipboard over SSH. Same limitation
  as upstream's existing `Ctrl+Y`.
- If you run an unconventional "shell" (e.g. a custom login script instead of
  a real shell) as your terminal's default command, `atuin pty-proxy init`'s
  detection of "the zsh binary path" can get confused and wrap the wrong
  thing. If your script relies on `$SHELL` internally (e.g. `exec $SHELL -l`),
  hardcode the real shell path instead, since pty-proxy intentionally
  rewrites `$SHELL` to match whatever it's wrapping.
