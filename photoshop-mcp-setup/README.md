# Photoshop MCP Setup for Claude Code

Automate Adobe Photoshop from Claude Code via the 00bx-photoshop-mcp server.

## Prerequisites

- Adobe Photoshop 2025 or 2026
- Adobe UXP Developer Tools (from Creative Cloud)
- Python 3.10+
- Node.js 18+
- Claude Code

## Install

### 1. Install the MCP package

```bash
npx -y 00bx-photoshop-mcp
```

### 2. Run the installer

The installer copies files to `~/.00bx-photoshop-mcp/` and sets up a Python venv.

```bash
PKG_DIR=$(ls -d ~/.npm/_npx/*/node_modules/00bx-photoshop-mcp | head -1)
bash "$PKG_DIR/install.sh"
```

If the install fails with a Python version error, the MCP SDK requires Python 3.10+. Install it and recreate the venv:

```bash
brew install python@3.12
rm -rf ~/.00bx-photoshop-mcp/mcp/.venv
cd ~/.00bx-photoshop-mcp/mcp
/opt/homebrew/opt/python@3.12/bin/python3.12 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Then install the proxy dependencies:

```bash
cd ~/.00bx-photoshop-mcp/adb-proxy-socket && npm install
```

### 3. Fix the UXP plugin manifest

The default manifest has invalid icon definitions that cause Photoshop to reject the plugin. Replace `~/.00bx-photoshop-mcp/uxp/ps/manifest.json` with:

```json
{
  "id": "com.00bx.photoshop-mcp",
  "name": "Photoshop MCP Agent",
  "version": "1.0.0",
  "main": "index.html",
  "host": [
    {
      "app": "PS",
      "minVersion": "26.0.0"
    }
  ],
  "manifestVersion": 5,
  "entrypoints": [
    {
      "type": "panel",
      "id": "vanilla",
      "minimumSize": { "width": 300, "height": 200 },
      "maximumSize": { "width": 300, "height": 200 },
      "preferredDockedSize": { "width": 300, "height": 200 },
      "preferredFloatingSize": { "width": 300, "height": 200 },
      "label": { "default": "Photoshop MCP Agent" }
    }
  ],
  "requiredPermissions": {
    "network": { "domains": "all" },
    "localFileSystem": "fullAccess"
  }
}
```

### 4. Enable UXP Developer Mode

This must be done **before** opening Photoshop.

If you have the UXP CLI installed:

```bash
# Accept the terms prompt
echo "" | arch -x86_64 /tmp/node-v20.19.0-darwin-x64/bin/node \
  /private/tmp/uxp-cli/node_modules/.bin/uxp devtools enable
```

Or open **Adobe UXP Developer Tools** app and it will enable automatically.

### 5. Add MCP to Claude Code settings

Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "adobe-photoshop": {
      "command": "/Users/YOUR_USERNAME/.00bx-photoshop-mcp/mcp/.venv/bin/python",
      "args": ["/Users/YOUR_USERNAME/.00bx-photoshop-mcp/mcp/ps-mcp.py"],
      "timeout": 30000
    }
  }
}
```

Replace `YOUR_USERNAME` with your macOS username.

## Usage (every session)

Each time you want to use the Photoshop MCP:

### 1. Start the proxy

```bash
cd ~/.00bx-photoshop-mcp/adb-proxy-socket && node proxy.js
```

Keep this running in a terminal. Background alternative:

```bash
nohup node proxy.js > /tmp/ps-proxy.log 2>&1 &
```

### 2. Open Photoshop

Open Photoshop **after** dev mode is enabled.

### 3. Load the UXP plugin

Open **Adobe UXP Developer Tools**, click **Add Plugin**, navigate to `~/.00bx-photoshop-mcp/uxp/ps/`, select `manifest.json`, then click **Load**.

Or via CLI:

```bash
arch -x86_64 /tmp/node-v20.19.0-darwin-x64/bin/node \
  /private/tmp/uxp-cli/node_modules/.bin/uxp plugin load \
  --manifest ~/.00bx-photoshop-mcp/uxp/ps/manifest.json
```

### 4. Connect the plugin

In Photoshop, open the **Photoshop MCP Agent** panel (Plugins menu) and click **Connect**. Check "Connect on Launch" so it auto-connects next time.

### 5. Restart Claude Code

The MCP tools will be available after restart.

## Verify

Test from Claude Code by asking it to list open Photoshop documents or get layer info.

## Troubleshooting

**"Plugin rejected due to invalid object"**
- The manifest has issues. Use the fixed manifest from step 3 above.
- Ensure Photoshop was opened **after** dev mode was enabled. Quit PS and reopen.

**"Could not connect to photoshop"**
- Check proxy is running: `lsof -i :3001`
- Check the UXP plugin is loaded and shows "Connected" in the panel
- Reload the plugin from UXP Developer Tools

**"No clients registered for application: photoshop"**
- The UXP plugin lost connection to the proxy. Click **Connect** in the PS panel, or reload the plugin.

**Canvas resize times out on large images**
- Use the built-in `resizeCanvas` command instead of batchPlay `canvasSize`.

**Plugin load fails with "modal state"**
- Dismiss any open dialogs in Photoshop and try again.

**UXP CLI architecture error (x86_64 vs arm64)**
- On Apple Silicon, run the CLI through Rosetta with an x86 Node binary:
  ```bash
  # Download x86 Node
  curl -sL "https://nodejs.org/dist/v20.19.0/node-v20.19.0-darwin-x64.tar.gz" -o /tmp/node-x64.tar.gz
  cd /tmp && tar xzf node-x64.tar.gz

  # Install UXP CLI
  mkdir /tmp/uxp-cli && cd /tmp/uxp-cli
  PATH="/opt/homebrew/opt/node@20/bin:$PATH" npm init -y
  PATH="/opt/homebrew/opt/node@20/bin:$PATH" npm install --ignore-scripts @adobe/uxp-devtools-cli
  cd node_modules/@adobe/uxp-devtools-helper
  PATH="/opt/homebrew/opt/node@20/bin:$PATH" npm install fs-extra tar
  PATH="/opt/homebrew/opt/node@20/bin:$PATH" node scripts/devtools_setup.js

  # Run CLI commands with x86 Node
  arch -x86_64 /tmp/node-v20.19.0-darwin-x64/bin/node /tmp/uxp-cli/node_modules/.bin/uxp apps list
  ```

## Architecture

```
Claude Code ↔ MCP Server (Python/stdio) ↔ Proxy (Node/WebSocket :3001) ↔ UXP Plugin ↔ Photoshop
```

## What's included

323 tools covering: filters, layer styles, shapes, selections, text, transforms, adjustments, canvas operations, layer management, and raw batchPlay execution.
