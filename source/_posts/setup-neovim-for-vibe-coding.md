---
title: 在 AI / Vibe Coding 时代，搭一套轻量可用的 Neovim 终端 IDE
date: 2026-04-13 09:26:58
tags: 
  - AI
  - DevOps 
excerpt: AI 编程时代的轻量 Neovim 配置记录
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

# 前言

最近一段时间，**vibe coding** 和 **AI 编程** 越来越火。很多开发场景已经从过去那种“长时间驻留在重型 IDE 里手写代码”，逐渐变成了另一种工作方式：

- 先让 AI 生成代码、补全代码、改代码
- 自己更多是在做审查、验证、微调、跳转、格式化和少量修补
- 终端里的操作时间越来越长，而传统重型 IDE 的使用时间反而变少了

在这种模式下，一个启动快、配置灵活、能够在终端里快速查看和编辑代码的工具，就显得越来越重要。

我并不是想把 Neovim 折腾成一个比肩所有 GUI IDE 的“全家桶”，而是想要一套：

- 足够轻量
- 足够快
- 能在终端中快速看代码
- 有基本的文件树、搜索、终端、LSP、补全、格式化、高亮能力
- 适合 Python / Node.js / Lua 等日常开发场景

也正因为这个原因，我开始重新学习和研究 **Neovim**。

这篇文章记录的是我从零搭建一套可用 Neovim 环境的过程。它不是“最炫配置”，也不是“最复杂方案”，而是一套偏实用主义、适合在 AI 编程时代辅助代码审查和轻量修改的终端 IDE 配置。


# 1. 安装与配置

## 1.1 安装 Neovim

官方安装文档：<https://neovim.io/doc/install/>

macOS 下可以直接使用 Homebrew 安装：

```bash
# 安装 nvim
brew install neovim
nvim --version
````

安装完成后，建议先确认版本。较新的插件和 LSP 配置通常更适配较新的 Neovim 版本。

---

## 1.2 安装 `lazy.nvim`

[`lazy.nvim`](https://github.com/folke/lazy.nvim) 是一个现代化的 Neovim 插件管理器。

这里采用它推荐的 **Structured Setup** 方式，配置目录结构更清晰，后续维护起来也更舒服。详细安装说明可参考官方文档：[https://lazy.folke.io/installation](https://lazy.folke.io/installation)

### `~/.config/nvim/init.lua`

```lua
require("config.lazy")
```

### `~/.config/nvim/lua/config/lazy.lua`

```lua
-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not (vim.uv or vim.loop).fs_stat(lazypath) then
  local lazyrepo = "https://github.com/folke/lazy.nvim.git"
  local out = vim.fn.system({ "git", "clone", "--filter=blob:none", "--branch=stable", lazyrepo, lazypath })
  if vim.v.shell_error ~= 0 then
    vim.api.nvim_echo({
      { "Failed to clone lazy.nvim:\n", "ErrorMsg" },
      { out, "WarningMsg" },
      { "\nPress any key to exit..." },
    }, true, {})
    vim.fn.getchar()
    os.exit(1)
  end
end
vim.opt.rtp:prepend(lazypath)

-- Make sure to setup `mapleader` and `maplocalleader` before
-- loading lazy.nvim so that mappings are correct.
-- This is also a good place to setup other settings (vim.opt)
vim.g.mapleader = " "
vim.g.maplocalleader = "\\"

-- Setup lazy.nvim
require("lazy").setup({
  spec = {
    -- import your plugins
    { import = "plugins" },
  },
  -- Configure any other settings here. See the documentation for more details.
  -- colorscheme that will be used when installing plugins.
  install = { colorscheme = { "habamax" } },
  -- automatically check for plugin updates
  checker = { enabled = true },
})
```

这套结构下，后续插件都放在：

```text
~/.config/nvim/lua/plugins/
```

每个插件一个文件，便于拆分与管理。

## 1.3 IDE 常用插件

下面是我实际使用的一套基础插件组合，已经可以组成一个相对完整的终端 IDE 环境。

### 1.3.1 `nvim-tree`：文件树

用于提供左侧文件树浏览能力。

```lua
return {
  "nvim-tree/nvim-tree.lua",
  dependencies = {
    "nvim-tree/nvim-web-devicons",
  },
  config = function()
    -- 官方建议禁用 netrw
    vim.g.loaded_netrw = 1
    vim.g.loaded_netrwPlugin = 1

    vim.opt.termguicolors = true

    require("nvim-tree").setup({
      filters = {
        dotfiles = false,
        git_ignored = false,
      },
      view = {
        width = 30,
        side = "left",
      },
    })

    vim.keymap.set("n", "<leader>e", "<cmd>NvimTreeToggle<CR>", {
      silent = true,
      desc = "Toggle NvimTree",
    })
  end,
}
```

说明：

* `dotfiles = false`：显示隐藏文件
* `git_ignored = false`：显示被 `.gitignore` 忽略的文件
* `<leader>e`：切换文件树

### 1.3.2 `telescope.nvim`：文件与内容搜索

用于快速搜索文件、全文搜索、查看 buffer、帮助标签等。

#### 安装依赖

```bash
brew install ripgrep fd
```

#### 插件配置

```lua
return {
  "nvim-telescope/telescope.nvim",
  dependencies = {
    "nvim-lua/plenary.nvim",
    {
      "nvim-telescope/telescope-fzf-native.nvim",
      build = "make",
    },
  },
  keys = {
    { "<leader>ff", "<cmd>Telescope find_files<CR>", desc = "Find Files" },
    { "<leader>fg", "<cmd>Telescope live_grep<CR>", desc = "Live Grep" },
    { "<leader>fb", "<cmd>Telescope buffers<CR>", desc = "Buffers" },
    { "<leader>fh", "<cmd>Telescope help_tags<CR>", desc = "Help Tags" },
  },
  config = function()
    local telescope = require("telescope")

    telescope.setup({})

    pcall(telescope.load_extension, "fzf")
  end,
}
```

常用快捷键：

* `<leader>ff`：搜索文件
* `<leader>fg`：全文搜索
* `<leader>fb`：查看当前 buffer
* `<leader>fh`：查看帮助标签

### 1.3.3 `toggleterm.nvim`：终端

用于在 Neovim 内嵌终端，避免频繁切出编辑器。

```lua
return {
  "akinsho/toggleterm.nvim",
  version = "*",
  config = function()
    require("toggleterm").setup({
      size = 20,
      open_mapping = [[<c-\>]],
      direction = "horizontal",
      start_in_insert = true,
      persist_size = true,
    })

    vim.api.nvim_create_autocmd("TermOpen", {
      pattern = "term://*",
      callback = function()
        local opts = { buffer = 0 }
        vim.keymap.set("t", "<esc>", [[<C-\><C-n>]], opts)
        vim.keymap.set("t", "<C-h>", [[<C-\><C-n><C-w>h]], opts)
        vim.keymap.set("t", "<C-l>", [[<C-\><C-n><C-w>l]], opts)
      end,
    })
  end,
}
```

说明：

* `Ctrl + \`：打开 / 关闭终端
* 在终端模式中按 `Esc`：退出终端插入模式
* `Ctrl-h` / `Ctrl-l`：在终端和其他窗口之间切换


### 1.3.4 LSP：语言服务器

用于提供跳转定义、查看引用、悬浮信息、重命名、诊断等能力。

这里使用：

* `mason.nvim`：安装和管理语言服务器
* `mason-lspconfig.nvim`
* `nvim-lspconfig`

```lua
return {
  {
    "mason-org/mason.nvim",
    opts = {},
  },

  {
    "mason-org/mason-lspconfig.nvim",
    dependencies = {
      "mason-org/mason.nvim",
      "neovim/nvim-lspconfig",
    },
    opts = {
      ensure_installed = {
        "pyright",
        "ts_ls",
        "lua_ls",
      },
    },
  },

  {
    "neovim/nvim-lspconfig",
    dependencies = {
      "mason-org/mason.nvim",
      "mason-org/mason-lspconfig.nvim",
      "hrsh7th/cmp-nvim-lsp",
    },
    config = function()
      local capabilities = require("cmp_nvim_lsp").default_capabilities()

      local function on_attach(_, bufnr)
        local map = function(keys, func, desc)
          vim.keymap.set("n", keys, func, { buffer = bufnr, desc = desc })
        end

        map("gd", vim.lsp.buf.definition, "Go to definition")
        map("gr", vim.lsp.buf.references, "References")
        map("K", vim.lsp.buf.hover, "Hover")
        map("<leader>rn", vim.lsp.buf.rename, "Rename")
        map("<leader>ca", vim.lsp.buf.code_action, "Code action")
        map("[d", vim.diagnostic.goto_prev, "Prev diagnostic")
        map("]d", vim.diagnostic.goto_next, "Next diagnostic")
      end

      vim.lsp.config("pyright", {
        on_attach = on_attach,
        capabilities = capabilities,
      })

      vim.lsp.config("ts_ls", {
        on_attach = on_attach,
        capabilities = capabilities,
      })

      vim.lsp.config("lua_ls", {
        on_attach = on_attach,
        capabilities = capabilities,
        settings = {
          Lua = {
            diagnostics = {
              globals = { "vim" },
            },
            workspace = {
              checkThirdParty = false,
            },
          },
        },
      })

      vim.lsp.enable("pyright")
      vim.lsp.enable("ts_ls")
      vim.lsp.enable("lua_ls")
    end,
  },
}
```

这套配置默认支持：

* Python：`pyright`
* TypeScript / JavaScript：`ts_ls`
* Lua：`lua_ls`

如果是 Python Monorepo 项目，还需要结合 `pyrightconfig.json` 或 `pyproject.toml` 中的 `[tool.pyright]` 做额外配置，尤其是：

* 根目录统一 `.venv`
* monorepo 下的 `libs`
* `extraPaths`
* workspace root 识别

### 1.3.5 `nvim-cmp`：自动补全

用于提供补全菜单，配合 LSP 使用效果更好。

```lua
return {
  {
    "hrsh7th/nvim-cmp",
    dependencies = {
      "hrsh7th/cmp-nvim-lsp",
      "hrsh7th/cmp-path",
      "hrsh7th/cmp-buffer",
      "L3MON4D3/LuaSnip",
      "saadparwaiz1/cmp_luasnip",
    },
    config = function()
      local cmp = require("cmp")
      local luasnip = require("luasnip")

      cmp.setup({
        snippet = {
          expand = function(args)
            luasnip.lsp_expand(args.body)
          end,
        },
        mapping = cmp.mapping.preset.insert({
          ["<C-k>"] = cmp.mapping.select_prev_item(),
          ["<C-j>"] = cmp.mapping.select_next_item(),
          ["<CR>"] = cmp.mapping.confirm({ select = true }),
          ["<C-Space>"] = cmp.mapping.complete(),
        }),
        sources = cmp.config.sources({
          { name = "nvim_lsp" },
          { name = "luasnip" },
          { name = "buffer" },
          { name = "path" },
        }),
      })
    end,
  },
}
```

常用快捷键：

* `Ctrl-j`：选择下一个补全项
* `Ctrl-k`：选择上一个补全项
* `Ctrl-Space`：手动触发补全
* `Enter`：确认补全

### 1.3.6 `bufferline.nvim`：缓冲区标签栏

用于显示打开的文件标签，体验更接近图形 IDE。

```lua
return {
  "akinsho/bufferline.nvim",
  version = "*",
  dependencies = {
    "nvim-tree/nvim-web-devicons",
  },
  config = function()
    require("bufferline").setup({
      options = {
        numbers = "none",
        offsets = {
          {
            filetype = "NvimTree",
            text = "File Explorer",
            text_align = "left",
            separator = true,
          },
        },
        show_buffer_close_icons = true,
        show_close_icon = true,
      },
    })

    vim.keymap.set("n", "<S-h>", "<cmd>BufferLineCyclePrev<CR>", {
      desc = "Prev Buffer",
    })
    vim.keymap.set("n", "<S-l>", "<cmd>BufferLineCycleNext<CR>", {
      desc = "Next Buffer",
    })
    vim.keymap.set("n", "<leader>bp", "<cmd>BufferLinePick<CR>", {
      desc = "Pick Buffer",
    })
  end,
}
```

常用快捷键：

* `Shift-h`：切到左边 buffer
* `Shift-l`：切到右边 buffer
* `<leader>bp`：选择 buffer

### 1.3.7 `conform.nvim`：格式化

用于统一管理格式化器，并支持保存时自动格式化。

```lua
return {
  "stevearc/conform.nvim",
  event = { "BufWritePre" },
  cmd = { "ConformInfo" },
  keys = {
    {
      "<leader>fm",
      function()
        require("conform").format({
          async = true,
          lsp_fallback = true,
        })
      end,
      mode = "",
      desc = "Format buffer",
    },
  },
  opts = {
    notify_on_error = true,

    format_on_save = function(bufnr)
      -- 某些文件类型你不想自动格式化，可以在这里排除
      local ignore_filetypes = {}
      if ignore_filetypes[vim.bo[bufnr].filetype] then
        return
      end

      return {
        timeout_ms = 2000,
        lsp_fallback = true,
      }
    end,

    formatters_by_ft = {
      lua = { "stylua" },

      python = { "ruff_format" },

      javascript = { "prettier" },
      javascriptreact = { "prettier" },
      typescript = { "prettier" },
      typescriptreact = { "prettier" },
      json = { "prettier" },
      jsonc = { "prettier" },
      yaml = { "prettier" },
      markdown = { "prettier" },
      html = { "prettier" },
      css = { "prettier" },
      scss = { "prettier" },

      sh = { "shfmt" },
      bash = { "shfmt" },
      zsh = { "shfmt" },
    },
  },
}
```

说明：

* 保存时自动格式化
* 手动格式化快捷键：`<leader>fm`
* 优先建议使用项目中已有的 formatter 配置，例如：

  * `pyproject.toml`
  * `.prettierrc`
  * `.editorconfig`
  * `.stylua.toml`

这样可以最大程度保持和项目已有规范一致。

### 1.3.8 Tree-sitter：代码高亮

用于提供更精细的语法高亮和代码结构解析。

#### 安装依赖

```bash
brew install tree-sitter-cli
```

#### 插件配置

```lua
return {
  "nvim-treesitter/nvim-treesitter",
  lazy = false,
  build = ":TSUpdate",
  config = function()
    require("nvim-treesitter").setup({
      install_dir = vim.fn.stdpath("data") .. "/site",
    })

    require("nvim-treesitter").install({
      "lua",
      "python",
      "javascript",
      "typescript",
      "tsx",
      "json",
      "yaml",
      "markdown",
      "bash",
      "html",
      "css",
    })

    vim.api.nvim_create_autocmd("FileType", {
      pattern = {
        "lua",
        "python",
        "javascript",
        "typescript",
        "typescriptreact",
        "javascriptreact",
        "json",
        "yaml",
        "markdown",
        "sh",
        "html",
        "css",
      },
      callback = function()
        vim.treesitter.start()
      end,
    })
  end,
}
```

说明：

* 这里使用的是 `nvim-treesitter` 新主线配置方式
* `tree-sitter-cli` 必须安装，否则 parser 无法正确编译
* 如果要确认是否生效，可以在代码中执行：

```vim
:InspectTree
```

如果能看到对应语法树结构，说明 Tree-sitter 已经正常工作


# 2. 常用使用技巧

除了插件配置，Neovim 的日常使用方式也和传统图形编辑器不太一样。下面记录几个最常用的技巧。

## 2.1 系统复制粘贴

为了让 Neovim 和系统剪贴板打通，可以在配置中加入：

### `~/.config/nvim/lua/config/keymaps.lua`

```lua
-- 开启剪贴板同步，使得 Neovim 与系统共享剪贴板
vim.opt.clipboard = "unnamedplus"

-- 开启鼠标支持（这样你偶尔可以用鼠标选中文字）
vim.opt.mouse = 'a'
```

### 常见复制粘贴操作

* **复制（Yank）**：Normal 模式下按 `y`

  * `yy`：复制整行
  * `yw`：复制一个单词
* **粘贴（Put）**：

  * `p`：粘贴到光标后
  * `P`：粘贴到光标前
* **选中复制**：

  * 按 `v` 进入可视模式
  * 选中文字后按 `y`

## 2.2 多行编辑（Visual Block 模式）

Vim/Neovim 不需要像 VS Code 那样依赖多光标完成大部分批量编辑。很多场景用 **可视块模式** 更高效。

### 场景 1：在多行开头插入相同内容

例如批量注释代码：

1. 光标移动到目标区域第一行开头
2. 按 `Ctrl-v` 进入 Visual Block 模式
3. 用 `j` / `k` 选中多行
4. 按大写 `I`
5. 输入要插入的内容，例如 `#`
6. 按 `Esc`

这时所有选中的行开头都会插入相同内容。

### 场景 2：删除多行的相同部分

1. 按 `Ctrl-v`
2. 选中矩形区域
3. 按 `d` 或 `x`

### 场景 3：在多行末尾插入

1. 按 `Ctrl-v`
2. 选中多行
3. 按 `$` 跳到每行末尾
4. 按大写 `A`
5. 输入内容
6. 按 `Esc`

## 2.3 文件操作

在 `nvim-tree` 中，文件管理大多通过单字母快捷键完成。

注意：**这些快捷键需要在文件树窗口中使用。**

### 新建文件或目录：`a`

按下 `a` 后，底部会出现输入框：

* 新建文件：输入文件名，例如 `main.lua`
* 新建目录：输入目录名并以 `/` 结尾，例如 `src/`
* 新建多级路径：例如 `lua/config/init.lua`

### 删除文件或目录：`d`

按下后会出现确认提示，输入 `y` 确认删除。

> 注意：通常是永久删除，不会进入 macOS 废纸篓

### 重命名：`r`

选中文件后按 `r`，修改名称并回车即可。

### 其他常用操作

| 快捷键  | 功能     |
| ---- | ------ |
| `x`  | 剪切文件   |
| `c`  | 复制文件   |
| `p`  | 粘贴文件   |
| `y`  | 复制文件名  |
| `Y`  | 复制相对路径 |
| `gy` | 复制绝对路径 |

## 2.4 格式化

### 保存时自动格式化

在 Python、TS、JS、Lua 等文件中，保存时会自动调用对应的 formatter。

### 手动格式化

```text
<leader>fm
```

也就是：

```text
空格 f m
```

## 2.5 跳转

LSP 接入后，最常用的跳转操作包括：

* `gd`：跳转到定义
* `gr`：查看引用
* `K`：查看悬浮信息
* `Ctrl-o`：跳回上一个位置
* `Ctrl-i`：跳到下一个位置
* `:jumps`：查看跳转历史

其中 `gd` 跳到定义后，最常用的返回方式就是：

```text
Ctrl-o
```

# 3. Python Monorepo 补充说明

如果你的项目是 Python Monorepo，尤其是：

* 根目录统一使用 `.venv`
* 内部有 `libs/`
* 使用 `uv` 管理虚拟环境和依赖

那么通常还需要额外配置 `pyrightconfig.json`，否则容易出现：

1. 明明已经安装的第三方包，LSP 却提示未安装
2. `libs` 里的内部模块无法正确识别

例如根目录统一 `.venv` 的场景，可以使用类似下面的配置：

```json
{
  "venvPath": ".",
  "venv": ".venv",
  "include": ["apps", "libs"],
  "exclude": ["**/node_modules", "**/__pycache__", ".git"],
  "extraPaths": ["libs"],
  "pythonVersion": "3.12",
  "verboseOutput": true
}
```

如果你的项目结构更复杂，比如每个子应用都有自己的导入根，还可能需要使用 `executionEnvironments` 继续细化配置。

# 4. 更简单的方式：直接使用 LazyVim

如果只是想快速拥有一套开箱即用的 Neovim IDE 体验，那么直接使用 [LazyVim](https://github.com/LazyVim/LazyVim) 往往是更省心的方案。

LazyVim 已经集成了大量常用能力，包括：

* 文件浏览器
* 搜索
* LSP
* 格式化
* 高亮
* 终端
* 各类现代化 UI


对于想少踩坑、快速上手的同学，我更推荐直接从 LazyVim 开始，再在其基础上做少量定制。


# 5. 总结

从零配置 Neovim 的过程，本质上是把一个“高效文本编辑器”逐步扩展成一个适合自己工作流的终端 IDE。

这套配置的核心能力包括：

* `lazy.nvim`：插件管理
* `nvim-tree`：文件树
* `telescope.nvim`：搜索
* `toggleterm.nvim`：终端
* `mason.nvim + nvim-lspconfig`：LSP
* `nvim-cmp`：补全
* `conform.nvim`：格式化
* `nvim-treesitter`：高亮与语法树

  它不是唯一方案，也未必是最“高级”的方案，但足够清晰、实用，而且比较适合作为理解 Neovim 生态的入门路径。

  如果目标是：

  * 想理解 Neovim 是怎么拼起来的
  * 想知道每一层能力分别由什么插件提供
  * 想搭一套尽量可控、可维护的个人配置


  那这条路径是很合适的。

  如果目标只是：

  * 快速拥有一套好用的开发环境
  * 少踩坑
  * 少折腾

  那直接从 LazyVim 开始会更轻松。

