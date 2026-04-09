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

## CRITICAL: Step-by-step procedure for EVERY image

You MUST follow ALL of these steps IN ORDER for EACH image. Do NOT skip any step.

### Step 1: DETECT image type (MANDATORY — do this FIRST)

Before opening in Photoshop, before doing ANYTHING else:

1. **Use the Read tool** to read the source image file (the Read tool supports PNG/JPG images and will show you the image visually)
2. Look at the image and classify it as one of:
   - **"wooden board"** — food/product sitting on a wooden board, wooden platter, chopping board, or wooden tray
   - **"non wooden board"** — everything else (plates, bowls, standalone products, white/light backgrounds, non-wood trays)
3. **Say out loud** which type you detected and which preset you will use, e.g.: "This is a wooden board image → using Wooden Board preset (distance 45, blur 30)"

⚠️ Do NOT skip this step. Do NOT assume all images are the same type. Do NOT default to one preset.

### Step 2: EXPAND the canvas (MANDATORY)

After opening the image in Photoshop:

1. Get the current document width and height
2. Add +3000px to EACH side: `new_width = width + 6000`, `new_height = height + 6000`
3. Use the `resizeCanvas` action with `anchor: "MIDDLECENTER"`
4. **Verify** the resize succeeded before continuing

⚠️ The canvas MUST be expanded BEFORE applying the shadow. Without this, the shadow WILL be cut off in the exported PNG. The layer effects scale is 400%, so a 150px distance becomes 600px effective — the shadow extends far beyond the image.

### Step 3: Apply the CORRECT shadow preset

Use the batchPlay command matching the type detected in Step 1:
- **Non Wooden Board** → distance: 150, blur: 50
- **Wooden Board** → distance: 45, blur: 30

### Step 4: Save and close

Save as PSD and PNG to the output folders, then close the document.

## Shadow presets

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

To change the canvas expansion (default +3000px each side), modify the `resizeCanvas` call width/height values in `process_image()`.

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

#### Non Wooden Board shadow (distance 150, blur 50)

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
        "dropShadow": {
            "_obj": "dropShadow",
            "enabled": True,
            "present": True,
            "showInDialog": True,
            "mode": {"_enum": "blendMode", "_value": "normal"},
            "color": {"_obj": "RGBColor", "red": 0.789, "grain": 1.084, "blue": 1.095},
            "opacity": {"_unit": "percentUnit", "_value": 80},
            "useGlobalAngle": False,
            "localLightingAngle": {"_unit": "angleUnit", "_value": 130},
            "distance": {"_unit": "pixelsUnit", "_value": 150},
            "chokeMatte": {"_unit": "pixelsUnit", "_value": 0},
            "blur": {"_unit": "pixelsUnit", "_value": 50},
            "noise": {"_unit": "percentUnit", "_value": 3},
            "antiAlias": False,
            "transferSpec": {"_obj": "shapeCurveType", "name": "Linear"},
            "layerConceals": True
        }
    }
}]
send_command("executeBatchPlayCommand", {"commands": commands, "layerId": layer_id})
```

#### Wooden Board shadow (distance 45, blur 30)

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
        "dropShadow": {
            "_obj": "dropShadow",
            "enabled": True,
            "present": True,
            "showInDialog": True,
            "mode": {"_enum": "blendMode", "_value": "normal"},
            "color": {"_obj": "RGBColor", "red": 0.789, "grain": 1.084, "blue": 1.095},
            "opacity": {"_unit": "percentUnit", "_value": 80},
            "useGlobalAngle": False,
            "localLightingAngle": {"_unit": "angleUnit", "_value": 130},
            "distance": {"_unit": "pixelsUnit", "_value": 45},
            "chokeMatte": {"_unit": "pixelsUnit", "_value": 0},
            "blur": {"_unit": "pixelsUnit", "_value": 30},
            "noise": {"_unit": "percentUnit", "_value": 3},
            "antiAlias": False,
            "transferSpec": {"_obj": "shapeCurveType", "name": "Linear"},
            "layerConceals": True
        }
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
- **Shadow cutoff**: Canvas was not expanded enough — must add +3000px each side (total +6000px per dimension) BEFORE applying shadow. The 400% layer effects scale means shadow extends much further than the raw distance value.
- **"No clients registered"**: Reload the UXP plugin and click Connect in the PS panel
- **Plugin crash after close**: The close command may error after saveDocumentAs changes the doc name — this is harmless and handled by the script
