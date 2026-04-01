# Photoshop Auto-Shadows

Batch apply drop shadow effects to product/SKU images using Claude Code + Photoshop MCP.

## Setup

Follow the [Photoshop MCP setup guide](photoshop-mcp-setup/README.md) to install and configure the MCP server on your machine.

## Usage

1. Clone this repo and open it in Claude Code
2. The skills auto-load — just ask Claude to batch add shadows to your images
3. Make sure the Photoshop MCP proxy is running and the UXP plugin is connected (see setup guide)

## What's included

- **Setup guide** (`photoshop-mcp-setup/README.md`) — step-by-step installation for the Photoshop MCP server
- **Batch shadow skill** (`.claude/skills/photoshop-batch-shadow/`) — teaches Claude Code how to batch process product images with drop shadows
- **Agent Bridge skill** (`.claude/skills/agent-bridge-for-photoshop/`) — teaches Claude Code how to use Agent Bridge for Photoshop automation
