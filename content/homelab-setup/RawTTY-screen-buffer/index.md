---
title: "Fixing RawTTY screen buffer"
weight: 3
draft: false
description: "Fixing RawTTY screen buffer issue on EndeavourOS"
slug: "fixing-raw-tty-endeavourOS"
tags: ["Homelab", "TTY", "EndeavourOS"]
series: ["Homelab Setup"]
series_order: 3
showDate: true
showTableOfContents: true
date: 2026-07-15
---


## 1. The Issue: TTY and the Screen Buffer

When you use a terminal emulator (like the one inside a desktop environment — GNOME Terminal, Konsole, etc.), opening a full-screen program like `vim` or `nano` swaps you into a separate "alternate screen." When you quit the program, your previous terminal content reappears exactly as it was.

**On a raw TTY (a bare console with no desktop, no X, no Wayland — just `Ctrl+Alt+F3` style login), this doesn't work the same way.**

### The symptom

- Opening `vim` or `nano`, editing, then quitting leaves the screen cleared or shows leftover garble instead of restoring your previous shell content.
- This was not an issue on Debian 13 so with the help of Claude [^claude], I devised a plan.

## 2. Fix 1: Custom `nano` and `vim` Commands (Wrapping Them in tmux)

Since `tmux` implements its own alternate-screen handling in software (independent of the console driver), one fix is to make `vim`/`nano` automatically launch inside a temporary `tmux` session.

### Install tmux

```bash
sudo pacman -S tmux
```

### Add wrapper functions

Add this to `~/.bashrc`:

```bash
vim() {
  if [ -z "$TMUX" ]; then
    local quoted=""
    for a in "$@"; do
      quoted+=" $(printf '%q' "$a")"
    done
    tmux new-session "command vim$quoted"
  else
    command vim "$@"
  fi
}

nano() {
  if [ -z "$TMUX" ]; then
    local quoted=""
    for a in "$@"; do
      quoted+=" $(printf '%q' "$a")"
    done
    tmux new-session "command nano$quoted"
  else
    command nano "$@"
  fi
}
```

Reload the shell config:

```bash
source ~/.bashrc
```

### What this does

- If you're **not already inside tmux**, typing `vim file.txt` or `nano file.txt` spawns a brand-new tmux session running just that editor.
- `command vim` / `command nano` (rather than plain `vim`/`nano`) bypasses the function itself, avoiding infinite recursion.
- When you quit the editor, tmux's single window closes, and since it was the only window, the whole tmux session ends automatically — dropping you back at your normal prompt.
- If you're **already inside tmux**, it just runs the editor normally in the current pane (no nested session).

## 3. Problems with Fix 1

- **Only fixes editing, not general TTY use.** Any other full-screen program (`less`, `htop`, etc.) still has the clearing problem, since only `vim`/`nano` are wrapped.
- **The screen clears instead of buffer swap** tmux can restore screens *within itself* correctly (e.g., moving between vim and the shell while tmux keeps running). But when tmux session exits, you get a clear screen and nothing on the terminal is restored. I could have just added `clear` into the `command vim` / `command nano` instead of installing `tmux`
- **Doesn't scale** — if you want this behavior for your whole session (not just editors), wrapping individual commands isn't a maintainable approach.

## 4. Method 2: Replace TTY Login with tmux

A better approach: instead of wrapping individual commands, launch (or attach to) a tmux session automatically the moment you log in. Then every program you run — not just vim/nano — benefits from tmux's internal screen handling, and the `exit` command logs you out.

Add this logic to `~/.bash_profile`

```bash
nano ~/.bashrc
```

Since I also use SSH, there are two common flavors depending on how you want SSH logins to behave.

### 4.1 Flavor 1: Fresh tmux Session for Every SSH Login

Use this if you want your **physical console** login to always reattach to the same persistent session (`main`), while every **SSH** login gets its own brand-new, isolated tmux session that no other SSH login shares.

```bash
if [[ -z "$TMUX" && $- == *i* ]]; then
  if [[ -n "$SSH_CONNECTION" ]]; then
    exec tmux new -s "ssh_$$"
  else
    if tmux has-session -t main 2>/dev/null; then
      exec tmux attach -t main
    else
      exec tmux new -s main
    fi
  fi
fi
```

**How it works:**

- `[ -z "$TMUX" ]` — only runs if you're not already inside a tmux session (avoids nesting).
- `$- == *i*` — only runs for interactive shells (won't interfere with scripts, `scp`, etc.).
- `[ -n "$SSH_CONNECTION" ]` — true only when logging in over SSH.
- **SSH login** → `tmux new -s "ssh_$$"` creates a uniquely named session using the shell's process ID (e.g. `ssh_2001`), so every SSH login is fresh and independent — never shared with another concurrent SSH session.
- **Physical tty login** → attaches to a session named `main` if one exists, or creates it. This one is meant to persist and be reused across tty logins.
- `exec` on every branch means exiting tmux ends the whole login session (logs you out), for both SSH and tty.

**Note:** Since each SSH session's name (`ssh_$$`) is tied to that specific login's process ID, you can't reattach to it later — a new SSH login always gets a brand-new one. This is intentional for "fresh every time" behavior.

### 4.2 Flavor 2: No tmux for SSH Login

Use this if you only want tmux's auto-restore behavior on the **physical console**, and want SSH logins to behave like an ordinary plain shell (no tmux at all).

```bash
if [[ -z "$TMUX" && $- == *i* && -z "$SSH_CONNECTION" ]]; then
  if tmux has-session -t main 2>/dev/null; then
    exec tmux attach -t main
  else
    exec tmux new -s main
  fi
fi
```

**How it works:**

- Same structure as Flavor 1, but `-z "$SSH_CONNECTION"` is added directly to the outer condition — so the entire tmux block only runs when there is **no** SSH connection.
- **Physical tty login** → still auto-attaches/creates `main`; exiting tmux logs you out via `exec`, same as before.
- **SSH login** → the condition is false, so this block is skipped entirely. You just get a completely normal login shell — no tmux, no auto-anything, `exit` behaves as usual.

## Do You Still Need the Method 1 Wrapper Functions?

**No.** Once Method 2 is in place, you're automatically inside tmux the moment you log in on the tty (or, in Flavor 1, on SSH too). Running plain `vim` or `nano` from inside that session already gets the correct alternate-screen restore behavior for free.

You can safely delete the wrapper functions from `~/.bashrc`:

```bash
vim() { ... }
nano() { ... }
```

Just leave `vim` and `nano` as plain commands going forward.

## References

Disclaimer: I had the idea of using `tmux` to circumnavigate an issue during a chat session with [^claude] then I let it write the script and markdown. I did some minor edits on markdown and published it.

[^claude]: [ClaudeAI](claude.ai)
