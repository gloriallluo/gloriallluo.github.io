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


Install `lua`:

```bash
sudo apt update
sudo apt install lua5.4 liblua5.4-dev
```

Install [luarocks](https://luarocks.org/):

```bash
wget https://luarocks.org/releases/luarocks-3.12.2.tar.gz
tar zxpf luarocks-3.12.2.tar.gz
cd luarocks-3.12.2
./configure && make && sudo make install
sudo luarocks install luasocket

# check luarocks installation
lua
> require "socket"
```

Download configuration:

```bash
mkdir -p ~/.config && cd ~/.config
git clone git@github.com:gloriallluo/nvim.git
```

Run `:checkhealth lazy` in nvim to verify if `Lazy` is installed correctly.

{: .box-note}
如果 lazy.nvim 安装不成功，可以先注释掉 `init.lua` 中依赖插件的部分，重新启动 Neovim 一次。

## LSPs

### clangd

```bash
sudo apt-get update
sudo apt-get install clangd-12
```

# 记录一些好用的插件

## 插件管理

- [packer.nvim](https://github.com/wbthomason/packer.nvim)

## LSP相关 / 语法支持 / 补全

- [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)
- 补全插件 [nvim-cmp](https://github.com/hrsh7th/nvim-cmp)
- 补全源 [cmp-nvim-lsp](https://github.com/hrsh7th/cmp-nvim-lsp), [cmp-buffer](https://github.com/hrsh7th/cmp-buffer), [cmp-path](https://github.com/hrsh7th/cmp-path), [cmp-cmdline](https://github.com/hrsh7th/cmp-cmdline)
- 更全面的代码解析插件 [treesitter](https://github.com/nvim-treesitter/nvim-treesitter)

## 功能类

- 文件树 [nvim-tree](https://github.com/nvim-tree/nvim-tree.lua)
- 终端 [toggleterm](https://github.com/akinsho/toggleterm.nvim)
- 顶部状态栏 [bufferline](https://github.com/akinsho/bufferline.nvim)
- 项目内搜索 [telescope](https://github.com/nvim-telescope/telescope.nvim)


