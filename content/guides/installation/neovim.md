---
title: "Neovim"
date: 2026-10-06
authors: [Nykenik, S1M0N38]
---

When using Neovim for LÖVE, you can use S1M0N38's [`love2d.nvim`](https://github.com/S1M0N38/love2d.nvim) plugin.

> [!IMPORTANT]
> This guide covers the setup. The plugin also ships full documentation, so run `:help love2d` to read it. Learning to navigate `:help` is one of the most valuable Neovim skills you can pick up.

## Disclaimer
This guide assumes you have [LÖVE installed](/guides/installation/love), [lua_ls](https://luals.github.io/) installed and a plugin manager. Neovim 0.12.2 or newer is required, so update Neovim before installing if you're on an older version.

You don't need `nvim-lspconfig` anymore. love2d.nvim configures `lua_ls` with Neovim's built-in LSP functions, so there is nothing to wire up by hand.

If you don't have any of these, the easiest way is to install a ready-made Neovim configuration, or a "Neovim distribution".

If something doesn't work, run `:checkhealth love2d`. It checks your Neovim version, the LÖVE binary, `lua_ls`, the type definitions, and the Tree-sitter parsers.

## Setup

### Installation
The plugin works with any plugin manager, or with none at all. This guide uses `lazy.nvim`.

> [!IMPORTANT]
> love2d.nvim 3.0.0 renamed its commands. `:LoveRun` and `:LoveStop` are gone; use `:Love run` and `:Love stop` instead. If you followed an older version of this guide, update your keymaps.

If you have a single-file setup, go to your `init.lua` and add this to your lazy setup:
```lua
require("lazy").setup({
  -- Other plugins go here.
  {
    "S1M0N38/love2d.nvim",
    version = "3.*",
    event = "VeryLazy",
    opts = {},
    keys = {
      { "<leader>v", "", desc = "LÖVE", ft = "lua" },
      { "<leader>vr", "<cmd>Love run<cr>", desc = "Run LÖVE", ft = "lua" },
      { "<leader>vw", "<cmd>Love watch<cr>", desc = "Watch LÖVE", ft = "lua" },
      { "<leader>vi", "<cmd>Love info<cr>", desc = "Info LÖVE", ft = "lua" },
      { "<leader>vs", "<cmd>Love stop<cr>", desc = "Stop LÖVE", ft = "lua" },
      { "<leader>vo", "<cmd>Love output<cr>", desc = "Output panel", ft = "lua" },
    },
  },
})
```

If you have a multi-file setup, create a new file in your plugins folder called `love2d.lua` and copy this into it:
```lua
return {
  "S1M0N38/love2d.nvim",
  version = "3.*",
  event = "VeryLazy",
  opts = {},
  keys = {
    { "<leader>v", "", desc = "LÖVE", ft = "lua" },
    { "<leader>vr", "<cmd>Love run<cr>", desc = "Run LÖVE", ft = "lua" },
    { "<leader>vw", "<cmd>Love watch<cr>", desc = "Watch LÖVE", ft = "lua" },
    { "<leader>vi", "<cmd>Love info<cr>", desc = "Info LÖVE", ft = "lua" },
    { "<leader>vs", "<cmd>Love stop<cr>", desc = "Stop LÖVE", ft = "lua" },
    { "<leader>vo", "<cmd>Love output<cr>", desc = "Output panel", ft = "lua" },
  },
}
```

**Without a plugin manager.** Neovim ships with `vim.pack`, so no extra plugin manager is needed:
```lua
vim.pack.add({
  {
    src = "https://github.com/S1M0N38/love2d.nvim",
    version = vim.version.range("3"),
  },
})
require("love2d").setup({})
vim.keymap.set("n", "<leader>vr", "<cmd>Love run<cr>", { desc = "Run LÖVE" })
vim.keymap.set("n", "<leader>vw", "<cmd>Love watch<cr>", { desc = "Watch LÖVE" })
vim.keymap.set("n", "<leader>vi", "<cmd>Love info<cr>", { desc = "Info LÖVE" })
vim.keymap.set("n", "<leader>vs", "<cmd>Love stop<cr>", { desc = "Stop LÖVE" })
vim.keymap.set("n", "<leader>vo", "<cmd>Love output<cr>", { desc = "Output panel" })
```

### Post-installation
After installing the plugin, you can configure it. It has three options:
- `path_to_love_bin`: The path to the `love` binary. Leave it as `"love"` if LÖVE is installed system-wide. On macOS, if you installed LÖVE as an `.app`, set it to `"/Applications/love.app/Contents/MacOS/love"`.
- `lsp`: Whether love2d.nvim should configure `lua_ls` for LÖVE projects automatically. It's `true` by default. Set it to `false` if you prefer to configure `lua_ls` yourself.
- `output`: Controls the floating output panel. It opens automatically by default. Set it to `false` to keep it closed, or pass a table of window options to customize it.

To change these values, modify the `opts` table in the plugin installation:
```lua
return {
  -- Other options go here.
  opts = {
    -- These are the default values.
    path_to_love_bin = "love",
    lsp = true,
    output = nil,
  },
  -- Keymaps go here.
}
```

## Usage
With the plugin set up, you can now develop games with LÖVE and Neovim.

Start Neovim from your project folder. love2d.nvim finds the project by looking for a `main.lua` or `conf.lua` in the current folder or any parent folder. You don't need to open the exact folder containing `main.lua`.

All commands are grouped under a single `:Love` command with subcommands:
- `:Love run` (`<leader>vr`): Run the project.
- `:Love watch` (`<leader>vw`): Run the project and restart it every time you save a Lua file. This is the usual way to work.
- `:Love stop` (`<leader>vs`): Stop the running project and exit watch mode.
- `:Love info` (`<leader>vi`): Show the detected project, its entry point, and whether it's running.
- `:Love output` (`<leader>vo`): Show or hide the output panel containing the game's `print` and error output.

> [!NOTE]
> By default LÖVE opens a graphical error screen when the game crashes, so that output never reaches Neovim. To send errors to the output panel and to your editor's diagnostics, your game needs a custom error handler — see `:help love2d-gamesetup`.

No extra configuration is needed for `love.` completions to work. When love2d.nvim detects a LÖVE project, it configures `lua_ls` with LÖVE's type definitions and the LuaJIT runtime, so completions, hover help (`K`), and go-to-definition work as in any other Lua project. When you leave the project, those settings are removed.

## GLSL highlighting
As a bonus, you can enable shader syntax highlighting.

To do this, install the GLSL parser with `:TSInstall glsl`. After doing so, Neovim will recognize the strings passed to `love.graphics.newShader()` as GLSL code.

