+++
title = "OpenCode in Neovim's World"
date = "2026-04-13"
image = "/images/opencode-with-neovim.webp"
tags = [
  "opencode",
  "neovim",
  "ai",
  "developer-tools",
  "productivity",
]
categories = [
  "Development",
]
+++

I spent the better part of last week trying to find the perfect Neovim plugin for OpenCode. I wanted the full integration: inline ghost text, sidebar chats, and a dedicated keymap for every possible agent action. After cycling through three different experimental plugins, I realized I was doing it wrong.

The best way to use OpenCode with Neovim isn't through a plugin at all. It's by leaning into the terminal. Neovim was designed to live in the shell, and OpenCode is a world-class CLI tool. When you stop trying to force one inside the other, the workflow actually starts to make sense.

## The setup

My current environment is intentionally simple. I run iTerm2 on macOS with a split-pane layout. Neovim stays in the top pane for editing, and OpenCode runs in a smaller pane at the bottom.

This isn't just about aesthetics. By keeping OpenCode in its own shell, I get a persistent history of everything the agent has done. I don't have to toggle a sidebar or check a hidden buffer. If the agent ran a complex migration or a massive refactor, the logs are right there, staring back at me.

To make this feel cohesive, I use a few standard CLI tricks. I keep `opencode` in my path, and I've mapped a quick toggle in iTerm2 to jump between panes. No fancy Lua required, just standard terminal navigation that I already have in my muscle memory.

## Why this beats a plugin

There's a temptation to want your AI assistant "aware" of your editor's state at all times. But OpenCode doesn't need to be inside your editor to understand your code. It reads the filesystem, respects your `.gitignore`, and follows the project structure perfectly from the shell.

When you use a plugin, you're adding a layer of abstraction between you and the agent. If the plugin's UI doesn't expose a specific flag or a new tool, you're stuck waiting for an update. By using the CLI directly, you get the full power of the tool the second it's released.

There's also the safety aspect. In my last post about [OpenCode's permission system](/posts/opencode-guardrails-permissions/), I talked about how critical it is to have guardrails. When the agent is running in a visible terminal pane, I see every `ask` prompt immediately. It doesn't get buried in a notification or a subtle UI change. I'm always one keystroke away from approving or rejecting a destructive command.

## Reviewing what the agent did

This is the most important part of the workflow. When you give an AI agent permission to edit your files, you become a reviewer. Neovim is an incredible tool for this, provided you have the right setup.

The first thing you need is a way to handle file changes that happen outside the editor. Since OpenCode writes directly to disk, Neovim needs to know when to reload. I use this auto-reload snippet in my `init.lua`:

```lua
vim.opt.autoread = true
vim.api.nvim_create_autocmd({"BufEnter", "FocusGained", "CursorHold"}, {
  command = "if mode() != 'c' | checktime | endif",
  pattern = {"*"},
})
```

With this, Neovim silently refreshes the buffer whenever OpenCode finishes a task. No "File changed on disk" prompts, just a seamless update.

### Plugin setup

I don't use much for the review phase, but these three plugins are non-negotiable. If you're using `lazy.nvim`, you can get them running with minimal config:

```lua
return {
  -- Git integration and hunk management
  {
    'lewis6991/gitsigns.nvim',
    config = function()
      require('gitsigns').setup()
    end
  },
  -- Better diff viewing
  {
    'sindrets/diffview.nvim',
    config = function()
      require('diffview').setup()
    end
  },
  -- The classic git wrapper
  'tpope/vim-fugitive',
}
```

If you prefer `packer.nvim`, it's the same idea: just `use 'tpope/vim-fugitive'` and the others. Most of these work great out of the box without any custom Lua.

Once the files are updated, I use `gitsigns.nvim` for a quick sanity check. I can jump between changed hunks using `]c` and `[c`, and preview the exact changes with `<leader>hp`. If the agent went off the rails, `<leader>hr` resets the hunk instantly. It's atomic, fast, and stays within my usual editing flow.

For larger changes, I reach for `diffview.nvim`. Running `:DiffviewOpen` gives me a full-project view of every modification the agent made. I use `<Tab>` and `<S-Tab>` to walk through the file list, and I can stage (`s`) or unstage (`u`) changes directly in the panel. When I'm done reviewing, `:DiffviewClose` takes me right back to where I was.

If I'm feeling old school, I'll use `:Gdiffsplit HEAD` from Fugitive to see a side-by-side diff of a specific file. There's something about the classic vim diff view that makes it easier to spot logic errors that a standard unified diff might miss.

## The end-to-end workflow

Here is how a typical feature implementation looks for me now:

1. **Prompt**: I tell OpenCode what I need in the bottom pane: `opencode "Refactor the user authentication logic to use JWT"`.
2. **Execute**: I watch the agent work. I can see it searching files, running tests, and making edits.
3. **Approve**: If it hits a gated command like `git commit`, I review the proposed message and hit `y` in the terminal.
4. **Review**: I jump to the top pane. My buffers are already updated. I use `]c` to fly through the changes.
5. **Verify**: If anything looks suspicious, I open `:DiffviewOpen` for a deeper look.
6. **Finalize**: Once I'm happy, I run my test suite one last time from the shell and push.

It's a pragmatic, visible, and incredibly fast way to work. You don't need a heavy IDE-like experience to get the benefits of AI coding. You just need a good editor, a solid terminal, and a tool like OpenCode that plays well with both.

Happy coding!
