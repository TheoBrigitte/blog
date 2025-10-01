---
title: "mcp-time: A Model Context Protocol Server for Time Operations"
date: 2025-10-01T20:50:00+02:00
description: "Enable AI assistants to perform time and date operations through a Model Context Protocol server with natural language support"
---

Working with time and dates is a common task in AI interactions, but AI models often struggle with current times, timezone conversions, and relative date calculations. **mcp-time** solves this by providing a [Model Context Protocol (MCP)](https://github.com/modelcontextprotocol) server that gives AI assistants standardized tools for time operations.

![mcp-time](/mcp-time.png)

## What is mcp-time?

Mcp-time is an MCP server written in Go that acts as a bridge between AI assistants and robust time-handling capabilities. It enables AI tools to:

- Get current time in any timezone
- Parse natural language time expressions like "yesterday" or "next month"
- Convert times between timezones
- Add or subtract durations
- Compare times across timezones

## Key Features

**⏰ Time Manipulation** - Get current time, convert between timezones, and add or subtract durations

**🗣️ Natural Language Parsing** - Understands relative time expressions like "yesterday", "5 minutes ago", or "next month"

**⚖️ Time Comparison** - Compare two different times with timezone-aware comparisons

**🎨 Flexible Formatting** - Supports predefined formats (RFC3339, Kitchen) and custom Go time layouts

**✅ MCP Compliance** - Fully compatible with the Model Context Protocol standard

**🔄 Multiple Transports** - Supports `stdio` for local integrations and `HTTP stream` for network access

## Installation

Mcp-time works with any MCP-compatible AI assistant including [Cursor](https://cursor.com/), [Claude Desktop](https://claude.ai/download), and [Claude Code](https://www.claude.com/product/claude-code).

### Using npx (Recommended)

The easiest method using Node.js:

```json
{
  "mcpServers": {
    "mcp-time": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@theo.foobar/mcp-time"]
    }
  }
}
```

### Using Docker

Run in an isolated container:

```json
{
  "mcpServers": {
    "mcp-time": {
      "type": "stdio",
      "command": "docker",
      "args": ["run", "--rm", "-i", "theo01/mcp-time:latest"]
    }
  }
}
```

### Using Binary

Download and install directly:

```bash
# Replace OS-ARCH with your platform (linux-amd64, darwin-arm64, etc.)
curl -Lo mcp-time https://github.com/TheoBrigitte/mcp-time/releases/latest/download/mcp-time.OS-ARCH
install -D -m 755 ./mcp-time ~/.local/bin/mcp-time
```

Then configure your MCP client:

```json
{
  "mcpServers": {
    "mcp-time": {
      "type": "stdio",
      "command": "mcp-time"
    }
  }
}
```

## Available Tools

### `current_time`

Get the current time in any timezone and format.

```
What time is it in Tokyo?
```

### `relative_time`

Parse natural language time expressions.

```
What was the date 3 weeks ago?
What time will it be in 45 minutes?
```

### `convert_timezone`

Convert times between timezones.

```
Convert 2:30 PM EST to Tokyo time
```

### `add_time`

Add or subtract durations from a time.

```
What time will it be in 45 minutes?
Add 2 hours to 3:30 PM
```

### `compare_time`

Compare two times with timezone awareness.

```
Is 3 PM EST before 8 PM GMT?
```

## MCP Resources and Prompts

Beyond tools, mcp-time provides MCP resources and prompts for enhanced functionality:

**Resources:**
- `time://formats` - Available time formats
- `time://current/{timezone}` - Current time in any timezone
- `time://timezone-info/{timezone}` - Detailed timezone information
- `time://timezones/popular` - Popular timezones reference
- `time://guide/relative-expressions` - Relative time expressions guide

**Prompts:**
- `time_format_helper` - Interactive help with time format strings
- `timezone_conversion_guide` - Guidance on timezone conversions
- `relative_time_examples` - Natural language time expression examples

## Usage Example

Once configured, you can naturally ask your AI assistant time-related questions:

```
You: What time is it in London?
AI: It's currently 19:45 BST in London

You: When is "next Friday at 2pm EST" in Tokyo time?
AI: Next Friday at 2pm EST is Saturday at 3am JST in Tokyo
```

## How It Works

Under the hood, mcp-time:

1. Implements the Model Context Protocol to expose time operations as tools
2. Uses [araddon/dateparse](https://github.com/araddon/dateparse) for flexible date parsing
3. Uses [tj/go-naturaldate](https://github.com/tj/go-naturaldate) for natural language processing
4. Provides both stdio and HTTP stream transports for different integration scenarios
5. Leverages Go's comprehensive `time` package for accurate timezone handling

The server runs locally and communicates with AI assistants through the MCP protocol, enabling seamless time operations without sending data to external services.

## Why MCP?

The Model Context Protocol standardizes how AI assistants interact with external tools and data sources. Instead of building custom integrations for each AI platform, mcp-time implements MCP once and works across all compatible assistants. This approach:

- Reduces development and maintenance overhead
- Ensures consistent behavior across platforms
- Makes it easy for users to add time capabilities to their AI tools
- Follows an open standard that's gaining industry adoption

## Recent Updates (v0.4.0)

- Added npx installation method for easier setup
- Enhanced `compare_time` tool with timezone support
- Fixed timezone data handling in Docker images
- Improved Windows compatibility

## Conclusion

Mcp-time demonstrates the power of the Model Context Protocol for extending AI capabilities. By providing a standardized interface for time operations, it enables AI assistants to handle date and time queries accurately and naturally, without the limitations of their training data cutoff.

Whether you're building AI applications that need current time information, working across timezones, or just want your AI assistant to understand "what time was it 3 hours ago in Paris?", mcp-time provides a robust and easy-to-integrate solution.

Check out the project on [GitHub](https://github.com/TheoBrigitte/mcp-time) and try it with your favorite MCP-compatible AI assistant!
