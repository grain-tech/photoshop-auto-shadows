---
name: photoshop-batch-shadow
description: Batch apply drop shadow effects to product/SKU images via Photoshop MCP. Auto-apply when the user asks to add shadows to multiple images, batch process product photos, apply layer effects to SKU images, or automate Photoshop layer styles across files. Requires the Photoshop MCP proxy to be running on port 3001 with the UXP plugin connected.
metadata:
  author: jiahui
  version: "1.0"
---

# Photoshop Batch Shadow

Apply consistent drop shadow effects to batches of product images via the Photoshop MCP proxy.

## Prerequisites

Before running, verify:
1. Proxy is running: `lsof -i :3001`
2. UXP plugin is loaded and connected in Photoshop
3. Python venv exists at `~/.00bx-photoshop-mcp/mcp/.venv/`

If not set up, see `photoshop-mcp-setup/README.md` for installation instructions.

## How it works

The batch script connects to Photoshop via a WebSocket proxy and for each image:

1. Opens the image in Photoshop
2. Expands the canvas (to prevent shadow cutoff)
3. Applies the drop shadow via batchPlay
4. Saves as PSD and PNG to separate output folders
5. Closes the document

## Shadow presets

There are two shadow presets. The script auto-detects which to use based on whether the image contains a wooden board.

### Detection logic

Before applying a shadow, the script should determine the image type:
1. Open the image in Photoshop
2. Take a screenshot/thumbnail or analyze the image visually
3. If the image features a **wooden board** (wooden platter, chopping board, wooden tray) → use **Wooden Board** preset
4. All other images (plates, bowls, standalone products, white backgrounds) → use **Non Wooden Board** preset

When building the batch script, use Claude's vision capability to classify each image before processing. Save a thumbnail/screenshot and pass it to the LLM with the prompt: "Does this product image feature a wooden board/platter? Reply YES or NO."

### Non Wooden Board preset (default)

Reference: `Abalone Yu Sheng (shadow).psd`

| Setting | Value |
|---------|-------|
| Blend Mode | Normal |
| Color | Near-black (RGB ~0.8, 1.1, 1.1) |
| Opacity | 80% |
| Angle | 130° (not global light) |
| Distance | 150px |
| Spread | 0px |
| Size (blur) | 50px |
| Noise | 3% |
| Contour | Linear |
| Layer Effects Scale | 400% |

### Wooden Board preset

Reference: `Orh Nee Hokkaido Cheese Tart (shadow) - 2.psd`

| Setting | Value |
|---------|-------|
| Blend Mode | Normal |
| Color | Near-black (RGB ~0.8, 1.1, 1.1) |
| Opacity | 80% |
| Angle | 130° (not global light) |
| Distance | 45px |
| Spread | 0px |
| Size (blur) | 30px |
| Noise | 3% |
| Contour | Linear |
| Layer Effects Scale | 400% |

### Key differences

The wooden board preset uses a **tighter shadow** (distance 45px vs 150px, blur 30px vs 50px) since wooden boards sit closer to the surface.

## Running the batch

### Folder structure

```
<working-dir>/
├── input/           ← source images (any subfolder structure)
├── output-psd/      ← PSD outputs
├── output-png/      ← PNG outputs
└── batch_shadow.py  ← the batch script
```

### Execution

```bash
~/.00bx-photoshop-mcp/mcp/.venv/bin/python batch_shadow.py
```

### Customizing

The script should define two shadow setting dicts: `SHADOW_WOODEN_BOARD` and `SHADOW_NON_WOODEN_BOARD`. For each image, detect the type and select the appropriate preset.

To change the canvas expansion (default +1000px each side), modify the `resizeCanvas` call width/height values in `process_image()`.

To change input/output paths, modify `INPUT_DIR`, `OUTPUT_PSD`, `OUTPUT_PNG` at the top of the script.

## Sending commands to Photoshop

The proxy accepts socket.io connections on `ws://localhost:3001`. Commands use this format:

```python
command = {
    "application": "photoshop",
    "action": "<action_name>",
    "options": { ... }
}
```

### Key actions

| Action | Options | Description |
|--------|---------|-------------|
| `openFile` | `filePath` | Open an image |
| `getLayers` | (none) | List all layers |
| `executeBatchPlayCommand` | `commands`, `layerId` | Run raw batchPlay |
| `resizeCanvas` | `width`, `height`, `anchor` | Resize canvas (absolute px) |
| `saveDocumentAs` | `filePath`, `fileType` | Save as PSD/PNG/JPG |
| `getDocuments` | (none) | List open documents |

### Applying layer effects via batchPlay

```python
commands = [{
    "_obj": "set",
    "_target": [
        {"_ref": "property", "_property": "layerEffects"},
        {"_ref": "layer", "_id": layer_id}
    ],
    "to": {
        "_obj": "layerEffects",
        "scale": {"_unit": "percentUnit", "_value": 400},
        "dropShadow": { ... }
    }
}]
send_command("executeBatchPlayCommand", {"commands": commands, "layerId": layer_id})
```

### Reading layer effects from a reference file

To extract shadow settings from an existing PSD:

```python
commands = [{
    "_obj": "get",
    "_target": [
        {"_property": "layerEffects"},
        {"_ref": "layer", "_id": layer_id}
    ]
}]
result = send_command("executeBatchPlayCommand", {"commands": commands})
# result["response"][0]["layerEffects"]["dropShadow"] contains all settings
```

## Troubleshooting

- **Timeout on canvas resize**: Use `resizeCanvas` action (built-in) instead of batchPlay `canvasSize`
- **Shadow cutoff**: Increase canvas expansion before applying shadow (default 1000px each side)
- **"No clients registered"**: Reload the UXP plugin and click Connect in the PS panel
- **Plugin crash after close**: The close command may error after saveDocumentAs changes the doc name — this is harmless and handled by the script
