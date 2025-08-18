---
layout: post
title: Setup Neovim from Scratch
subtitle: 备忘用
tags: [Tools]
comments: true
---

# Setup

## Download

- [Install Neovim](https://github.com/neovim/neovim/blob/master/INSTALL.md)
- [Blog: 从零配置neovim](https://martinlwx.github.io/zh-cn/config-neovim-from-scratch/)


## Configure

Install [Packer](https://github.com/wbthomason/packer.nvim):

```bash
git clone --depth 1 https://github.com/wbthomason/packer.nvim \
 ~/.local/share/nvim/site/pack/packer/start/packer.nvim
```

Download configuration:

```bash
mkdir -p ~/.config && cd ~/.config
git clone git@github.com:gloriallluo/nvim.git
```

To install nvim plugins, type nvim command `:PackerSync`.


