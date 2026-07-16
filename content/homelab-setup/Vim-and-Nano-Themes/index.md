---
title: "[Optional] Vim and Nano Themes"
weight: 4
draft: false
description: "Setting up editor themes for vim and nano themes"
slug: "setting-up-vim-nano-themes"
tags: ["Homelab", "Themes"]
series: ["Homelab Setup"]
series_order: 4
showDate: true
showTableOfContents: true
date: 2026-07-15
---

## Introduction

Trying to add some themes for both `nano` and `vim` to improve readability. This is an optional article and can be skipped.

## Setup themes for `vim`

I chose [embark theme](https://github.com/embark-theme/vim) for `vim`. Asked Claude [^claude] for instructions.

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

Then in `~/.vimrc`

```ini
call plug#begin()
Plug 'embark-theme/vim', { 'as': 'embark', 'branch': 'main' }
call plug#end()
```

Open `vim` and run `:PlugInstall`. Type `:q` after finishing the installation. Then you can change the colorscheme by

```vim
:colorscheme embark
```

To make it persistent, in `~/.vimrc`, add

```ini
colorscheme embark
```

and restart `vim`

## Install nano themes

There is an interesting resource which I haven't tried but here is an interesting link [^nano-colors]. Honestly, I decided to learn `vim` commands as I saw that every version could change the config parameter labels and detecting `sh` based files is probably a pain for later.

## Resource

[^claude]: Used for code generation [ClaudeAI](claude.ai)
[^nano-colors]: [Reference to change text editor colors](https://www.linux.org/threads/how-to-change-the-colors-of-nano.40658/)
