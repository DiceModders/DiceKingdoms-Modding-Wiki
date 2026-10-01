---
summary: Browser-based editor for island codes - paint terrain and nature, place buildings with real footprints
---
# Island Editor

A single-file web tool that decodes an island code, lets you edit it visually and encodes it back. It runs entirely in your browser; nothing is uploaded.

**[Open the Island Editor](island-editor-app.html){ target=_blank }** - source: [`docs/tools/island-editor-app.html`](https://github.com/DiceModders/DiceKingdoms-Modding-Wiki/blob/main/docs/tools/island-editor-app.html)

!!! note "Browser support"
    Needs `CompressionStream` / `DecompressionStream` with the `deflate-raw` format (recent Chrome, Edge, Firefox, Safari).

## Workflow

1. In game, with dev cheats enabled (see [Island Grid](../game-architecture/island-grid.md#developer-cheats)), run `saveisland` so the code is on your clipboard.
2. Paste it into the editor and load it.
3. Edit, then copy the new code.
4. In game run `loadisland`.

## Features

- **Terrain and nature painting**: point, fill, line, rectangle, square, circle/oval, spray; brush size; X/Y symmetry; fill or outline; undo/redo.
- **Coast tools**: outline, grow and shrink the ground layer.
- **Buildings**: pick a type by name, hover footprint preview, rotate (`R` / `Shift+R`), mirror (`M`), click any footprint tile to remove. The footprints are embedded (24 types), taken from the [Buildings](../game-architecture/buildings.md) dump.
- **Checks**: "Check buildings" reports footprint tiles that are out of bounds, on water, under nature or overlapping; "Clear nature under buildings" fixes the most common refusal reason.
- **Decoded JSON view**: the full decoded structure as editable JSON, applied back to the grid.
- **Orientation toggle**: the game draws Y up, so the editor does too by default ("Game orientation" toggle).

## How it works

The editor contains a complete reader/writer for the [Island Save Format](../game-architecture/island-save-format.md): base64, raw DEFLATE via the browser compression streams, MLAPI packed ints, and the RLE tile layers. Buildings, remains and inner cliffs are preserved as decoded so that unedited parts round-trip unchanged.

```js
// decode
const raw   = await inflateRaw(b64ToBytes(code));
const island = parse(raw);          // { buildings, remains, ground, nature, cliffs }

// ... edit island.ground / island.nature / island.buildings ...

// encode
const out = bytesToB64(await deflateRaw(encode(island)));
```

## Known limitations

- Existing buildings' `health`, `disaster` and `turn` can be changed only through the JSON view, not the canvas.
- A Castle added by the editor has been seen **not to appear** in game after `loadisland`; see the open issue on the [Buildings](../game-architecture/buildings.md#placement-rules) page. Use "Check buildings" first.
