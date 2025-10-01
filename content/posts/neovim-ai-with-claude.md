---
title: "NeoVim AI with Claude"
date: 2025-10-01T22:40:28+02:00
description: "Setup NeoVIM with Claude Code"
---

Make the magic AI happen without leaving nvim

### Install Claude Code

Installing the Claude Code CLI, https://www.claude.com/product/claude-code

```
npm install -g @anthropic-ai/claude-code
or for Arch user only
yay -Sy claude-code https://aur.archlinux.org/packages/claude-code
claude # setup with API Usage Billing and token from https://console.anthropic.com/settings/keys
```

### Setup NeoVIM

As for plugin went with [greggh/claude-code.nvim](https://github.com/greggh/claude-code.nvim#readme) which has window navigation, buffer updates, claude cli flags and felt better integrated overall than (coder/claudecode.nvim)[https://github.com/coder/claudecode.nvim].

Just copy the [configuration](https://github.com/greggh/claude-code.nvim#configuration) to some lua file (e.g. init.lua) in your vim config. You also need to add the claude-code plugin and its dependency:

e.g. using [vim plug](https://github.com/junegunn/vim-plug)

```
Plug 'nvim-lua/plenary.nvim'
Plug 'greggh/claude-code.nvim'
```

### Github MCP Setup

Here is the [official doc(https://docs.claude.com/en/docs/claude-code/mcp).

Here is an example adding the [Github MCP server](https://github.com/github/github-mcp-server)
Using the `user` scope here, meaning it will be available all the time in any project. Also make sure your github token is set in the `GITHUB_TOKEN` env var.

```
cat <<EOF > github.mcp.json
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
Checking MCP server health...

github: https://api.githubcopilot.com/mcp/ (HTTP) - ✓ Connected
```

Here we go you are all set!

I am running a slightly [modified version](https://github.com/greggh/claude-code.nvim/pull/79) which allow for custom window navigation keymap. :theo:
