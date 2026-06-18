# Zhenzhen AI UXP

Photoshop UXP MVP for the T8mars/Zhenzhen OpenAI-compatible image APIs.

## Features

- Default Base URL: `https://ai.t8star.org`
- User-editable Base URL saved in the UXP plugin data folder
- API key saved in the UXP plugin data folder
- Text-to-image request via `/v1/images/generations`
- Current-document image edit via `/v1/images/edits`, with up to 8 captured canvas references
- Async polling via `/v1/images/tasks/{task_id}`
- Custom model name override
- Quality, aspect ratio, resolution, and count controls
- Generated result preview and insert-into-current-canvas action
- Two-module UI with separate Connection and Generate panels
- Responsive Flexbox panel content for Photoshop docked or floating panel resizing

## Load In Photoshop

1. Open UXP Developer Tool.
2. Add plugin from this folder.
3. Load the plugin.
4. Open the panel from Photoshop's Plugins menu.

## Notes

This is a UXP-native rewrite of the API workflow. It does not embed ComfyUI, Python, `requests`, `torch`, or CEP/ExtendScript code.

Photoshop controls panel frame resizing. In floating or docked mode, drag the Photoshop panel boundary to resize; the plugin UI reflows inside that host-controlled size.
