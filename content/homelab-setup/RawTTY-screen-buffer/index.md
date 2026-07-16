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


## The Issue: TTY and the Screen Buffer

When I switched from Debian 13 to no desktop environment on EndeavourOS (Titan Neo) - don't ask why I switched to a rolling distribution, I just wanted to try it on a friend's suggestion - I noticed a peculiar problem.

### The symptom

- Opening `vim` or `nano`, editing, then quitting shows leftover garble instead of restoring your previous shell content.

This was not an issue on Debian 13. I learned about terminal emulators (Kitty/Alacritty)but all these need a desktop environment. Whereas my no-desktop environment used RawTTY. I don't know how Debian 13 did but I needed a fix. After a chat session with the help of Claude [^claude], I thought what if I ran my session with `tmux`, which natively supports screen buffer?

## Fix 0: Best fix

EndeavourOS already has a lot of colors installed, even in no desktop mode. Just pick one of the colors using

```bash
toe
```

I chose `xfce` for example. You can change the color scheme or update in `~/.bash_profile`

```bash
export TERM=xfce
```

`xfce` supports screen buffer nicely and there was no need of installing `tmux`. If not for Claude leading me on a tangent, saying there was no solution, I wouldn't have needed `tmux`. The rest of the article is just me having fun trying to implement a solution via `tmux`.

## Fix 1: Custom `nano` and `vim` Commands (Wrapping Them in tmux)

Since `tmux` implements its own alternate-screen handling in software (independent of the console driver), one fix is to make `vim`/`nano` automatically launch inside a temporary `tmux` session.

### Install tmux

```bash
sudo pacman -S tmux
```

### Add wrapper functions

Adding this to `~/.bashrc`:

```bash
nano ~/.bashrc
```

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
- `command vim` / `command nano` (rather than plain `vim`/`nano`) bypasses the function itself, avoiding infinite recursion - as per Claude [^claude]
- When you quit the editor, tmux's single window closes, and since it was the only window, the whole tmux session ends automatically — dropping you back at your normal prompt.
- If you're **already inside tmux**, it just runs the editor normally in the current pane (no nested session).

## Problems with Fix 1

- **The screen clears instead of buffer swap** `tmux` can restore screens within its sessions correctly. But when tmux session exits, you get a clear screen and nothing on the RawTTY session is restored. I could have just added `clear` into the `command vim` / `command nano` instead of installing `tmux`
- **Doesn't scale** — if you want this behavior for every full-screen program, wrapping individual commands isn't ideal.

## Fix 2: Replace TTY Login with tmux

What if the tmux session started automatically the moment you log in? Then every program you run — not just vim/nano — benefits from tmux's internal screen handling, and the `exit` command logs you out.

Adding this logic to `~/.bash_profile`

```bash
nano ~/.bashrc
```

Since I also use SSH, there are two common flavors depending on how you want SSH logins to behave.

### Flavor 1: Fresh tmux Session for Every SSH Login

Use this if you want your **host device** login to always reattach to the same persistent session (`main`), while every **SSH** login gets its own brand-new, isolated tmux session that no other SSH login shares.

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
- **SSH login** → `tmux new -s "ssh_$$"` creates a unique named session using the shell's process ID (e.g. `ssh_2001`), so every SSH login is fresh and independent — never shared with another concurrent SSH session.
- **Physical tty login** → attaches to a session named `main` if one exists, or creates it. This one is meant to persist and be reused across tty logins.
- `exec` on every branch means exiting tmux ends the whole login session (logs you out), for both SSH and tty.

**Note:** Since each SSH session's name (`ssh_$$`) is tied to that specific login's process ID, you can't reattach to it later (unless you get hold of the session name when it is active) — a new SSH login always gets a brand-new one. This is intentional for "fresh every time" behavior.

### Flavor 2: No tmux for SSH Login

Use this if you only want `tmux` on the **host device**, and want SSH logins to behave like an ordinary plain shell (no tmux at all).

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
- **On SSH login** → the condition is false, so this block is skipped entirely. You just get a completely normal login shell — no tmux.

## Do You Still Need the Fix 1 Wrapper Functions?

**No.** Once Fix 2 is in place, you're automatically inside tmux when you log in on the tty (or, in Flavor 1, on SSH too). Running `command vim` or `command nano` is no different from plain versions so you can safely delete the wrapper functions from `~/.bashrc`:

## References

Disclaimer: I had the idea of using `tmux` to circumnavigate an issue during a chat session with [^claude] then I let it write the script and markdown. I did some minor edits on markdown and published it.

Edit: I edited the writeup myself after first publish because it looked so fake! I promise myself to not use AI writeup directly.

[^claude]: [ClaudeAI](claude.ai)
