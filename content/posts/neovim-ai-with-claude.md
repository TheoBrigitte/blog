---
title: "NeoVim AI with Claude"
date: 2025-10-01T22:40:28+02:00
description: "Setup NeoVim with Claude Code"
---

Bring AI-powered coding assistance directly into NeoVim without leaving your editor.

{{- (resources.Get "neovim-claude-code.png").RelPermalink -}}

## Install Claude Code

First, install the Claude Code CLI from https://www.claude.com/product/claude-code

```bash
# Using npm
npm install -g @anthropic-ai/claude-code

# Or for Arch Linux users
yay -S claude-code # https://aur.archlinux.org/packages/claude-code

# Setup authentication
claude # Follow prompts to configure API key from https://console.anthropic.com/settings/keys
```

## Setup NeoVim

I chose [greggh/claude-code.nvim](https://github.com/greggh/claude-code.nvim#readme) as it provides better integration with window navigation, buffer updates, and Claude CLI flags compared to [coder/claudecode.nvim](https://github.com/coder/claudecode.nvim).

Add the plugin and its dependency to your plugin manager. For example, using [vim-plug](https://github.com/junegunn/vim-plug):

```vim
Plug 'nvim-lua/plenary.nvim'
Plug 'greggh/claude-code.nvim'
```

Then copy the [configuration](https://github.com/greggh/claude-code.nvim#configuration) into your Lua config file (e.g., `init.lua`).

## GitHub MCP Setup

The [Model Context Protocol (MCP)](https://docs.claude.com/en/docs/claude-code/mcp) extends Claude Code's capabilities by connecting it to external tools and services.

Here's how to add the [GitHub MCP server](https://github.com/github/github-mcp-server). This uses the `user` scope, making it available across all projects. Make sure your GitHub token is set in the `GITHUB_TOKEN` environment variable.

```bash
cat <<EOF > github-mcp.json
{
  "type": "http",
  "url": "https://api.githubcopilot.com/mcp/",
  "headers": {
    "Authorization": "Bearer \${GITHUB_TOKEN}"
  }
}
EOF

claude mcp add-json --scope user github "$(jq -cM . github-mcp.json)"
claude mcp list
```

Expected output:
```
Checking MCP server health...

github: https://api.githubcopilot.com/mcp/ (HTTP) - ✓ Connected
```

You're all set! Claude Code is now integrated with NeoVim and can access GitHub through MCP.

**Note:** I'm running a [modified version](https://github.com/greggh/claude-code.nvim/pull/79) that allows custom window navigation keymaps.
