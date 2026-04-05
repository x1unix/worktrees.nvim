# Worktrees.nvim

Git worktree wrapper for neovim

After using [git-worktree.nvim](https://github.com/ThePrimeagen/git-worktree.nvim) plugin for quite some time I decided to make my own git worktree plugin with a different api and flow/usage.

## Requirements

- neovim nightly (0.7+)
- [plenary.nvim](https://github.com/nvim-lua/plenary.nvim)
- [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) (optional)
- [snacks.nvim](https://github.com/folke/snacks.nvim) (optional)

## Installation

Install the plugin and dependencies with preferred plugin manager

### Packer

```lua
use("nvim-lua/plenary.nvim")

use({
    "Juksuu/worktrees.nvim",
    config = function()
        require("worktrees").setup()
    end,
})
```

## Options

```lua
require("worktrees").setup({
    log_level = <one of vim.log.levels> -- default: vim.log.levels.WARN,
    log_status = <boolean> -- default: true
    worktree_path = <string> -- default: ".."
    switch_file_command  = <string> -- default: Ex
})
```

`log_` options are described in the [Troubleshooting](#troubleshooting) section

`worktree_path` controls where new worktrees are created:

- `".."` uses the default behavior and creates the worktree under the parent
  of the repository root.
- Any other value is used as a custom root directory and new worktrees are
  created under `<worktree_path>/<projectname>/<folder>`.

Path notes:

- `~` is expanded.
- Relative paths are resolved against current Neovim `cwd`.

`switch_file_command` controls what command to use when switching worktrees or removing worktree and no buffer is available for use. Set to nil to disable

## Usage

All the commands and functions this plugin provides utilizes the vim.fn.input function to ask users for required or optional parameters. Optional parameters are indicated with (optional) in the input prompt

### Creating new worktree

New worktree can be created using the provided command GitWorktreeCreate

```
:GitWorktreeCreate
```

or with lua

```lua
:lua require("worktrees").new_worktree()
```

### Switching to another worktree

If a file is open in a buffer when switching, the plugin will try to find the file in the other worktree, if it exists it will change the buffer to correspond to the new worktree file.

Switching can be done using the provided command GitWorktreeSwitch

```
:GitWorktreeSwitch
```

or with lua

```lua
:lua require("worktrees").switch_worktree()
```

### Creating worktree for existing branch

Creating worktree for existing branch can be done with the provided command GitWorktreeCreateExisting

```
:GitWorktreeCreateExisting
```

or with lua

```lua
:lua require("worktrees").new_worktree(true)
```

### Remove existing worktree

Remove worktree for existing branch can be done with the provided command GitWorktreeRemove

```
:GitWorktreeRemove
```

or with lua

```lua
:lua require("worktrees").remove_worktree()
```

## Hooks

You can provide hooks to perform additional actions on worktree addition, removal and switching

```lua
require('worktrees').setup {
    hooks = {
        on_add = function(name, path, branch)
            -- your action here
        end,
        on_before_switch = function(from, to, git_path_info)
            -- your action here
        end,
        on_switch = function(from, to, git_path_info)
            -- your action here
        end,
        on_remove = function(name)
            -- your action here
        end,
    }
}
```

### Example - switch all open buffers to new worktree

```lua
require('worktrees').setup {
    swap_current_buffer = false,
    hooks = {
        on_switch = function(from, to)
            local Path = require 'plenary.path'
            local any_exist = false
            for win in vim.iter(vim.api.nvim_list_tabpages()):map(vim.api.nvim_tabpage_list_wins):flatten() do
                local bufnr = vim.api.nvim_win_get_buf(win)
                local buf_path = Path:new(vim.api.nvim_buf_get_name(bufnr))
                local rel_path = buf_path:make_relative(from)
                local path_in_new_cwd = Path:new(to .. '/' .. rel_path)
                if path_in_new_cwd:exists() then
                    any_exist = true
                    vim.schedule(function()
                        local buf_in_new_cwd = vim.fn.bufnr(path_in_new_cwd:absolute(), true)
                        vim.api.nvim_win_set_buf(win, buf_in_new_cwd)
                        vim.api.nvim_buf_delete(bufnr)
                    end)
                else
                    vim.api.nvim_win_close(win, true)
                end
                if not any_exist then
                    local root = vim.fn.bufnr(to)
                    vim.api.nvim_set_current_buf(root)
                end
            end
        end,
    }
}

```

### Example - per-worktree sessions using `mini.sessions`

```lua
require('worktrees').setup {
    swap_current_buffer = false,
    hooks = {
        on_before_switch = function(from, to, git_path_info)
            -- Persist worktree session before worktree switch.
            MiniSessions.write(nil, { force = true, verbose = false })
            vim.cmd('silent! %bwipeout!')

            -- Prevent dangling LSP sessions.
            vim.lsp.stop_client(vim.lsp.get_clients())

            -- Out of the box, worktrees.nvim changes a working directory on worktree switch.
            -- Optionally, you may prevent this by returning "false".
            -- return false
        end,
        on_switch = function(from, to, git_path_info)
            -- Restore session after worktree changes.
            if vim.fn.filereadable(MiniSessions.config.file) then
              MiniSessions.read(nil, { force = true, verbose = false })
            end
        end,
        on_before_remove = function(path)
            if path ~= vim.loop.cwd() or vim.v.this_session == '' then
              return
            end

            -- Detach session if active worktree will be removed.
            MiniSessions.delete(nil, { force = true, verbose = false })
            vim.v.this_session = ''
        end
    }
}
```

## Telescope

The extension can be loaded with telescope

```lua
require("telescope").load_extension("worktrees")
```

### Switching worktrees with telescope

```lua
require("telescope").extensions.worktrees.list_worktrees(opts)
-- <Enter> - switches to that worktree
```

## Snacks.nvim

Worktrees can also be created, switched and removed using snacks.nvim

```lua
vim.keymap.set("n", "<leader>gws", function() Snacks.picker.worktrees() end)
vim.keymap.set("n", "<leader>gwn", function() Snacks.picker.worktrees_new() end)
vim.keymap.set("n", "<leader>gwr", function() Snacks.picker.worktrees_remove() end)
```

## Troubleshooting

This plugin provides logging to a file which can be used to debug bugs etc.

The log file resides in neovims cache path and the logging level can be changed by changing the `log_level` option in setup. Status logs in neovim messages can also be toggled in the options

```lua
require("worktrees").setup({
    log_level = <one of vim.log.levels> -- default vim.log.levels.WARN,
    log_status = <boolean>, -- default true
})
```

### Upstream setup

For this plugin to work correctly the upstream fetch config needs to be setup correctly. This seems to not be the case when using bare repositories. To check if it is setup correctly run the following command and check that it returns as shown below

```bash
git config --get remote.origin.fetch

+refs/heads/*:refs/remotes/origin/*
```

If not run the following command to fix it

```bash
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
```

## TODO

- [ ] Options to customize behavior
