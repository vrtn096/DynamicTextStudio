# Dynamic Text Studio

A single-file Python tool for **UEFN** (Unreal Editor for Fortnite) that builds **MSDF font atlases** from any
font and **writes text straight into material instances** of a dynamic text material, including proportional
(variable-width) spacing.

It runs inside the editor, in its own window, and touches your project only through UEFN's own editor API.

---

## Features

### Atlas tab: make a font atlas
- Turns any **.ttf / .otf / .ttc** font, including variable fonts with a style picker, into a 2048 × 2048
  **multi-channel signed distance field (MTSDF)** atlas.
- Imports it into the folder you pick, with the texture settings the material expects already applied:
  uncompressed, sRGB off, clamp, mipmaps, all mips kept resident.
- **Measures every glyph's width** and stores it on the texture, so proportional spacing works with that font
  straight away.
- Shows a preview and a report: missing characters, glyphs that do not fit their cell, and stray pixels in the
  distance field.
- Fonts without lowercase reuse their uppercase glyphs automatically.

### Text tab: write into a material instance
- Write up to **4 rows** of text into a material instance, each with its own **left / center / right** alignment.
- **Font picker** lists every `Text_Atlas_MSDF_*` atlas in the project.
- **Spacing modes:**
  - **Monospace**: every letter takes the same slot. Cheapest.
  - **Overlap**: same slots, but wide letters may spill into their neighbours.
  - **Proportional**: each letter takes its real width, so `i`, `l`, `.` and `,` sit tight.
- **Markup** inside a row:
  - `<1>text</>`, `<2>…</>`, `<3>…</>`: draw that text with style 1, 2 or 3 (colours set on the material).
  - `[icon1]`, `[icon2]`, `[icon3]`: place an icon at that point of the row and reserve space for it.
- Optional vertical centering of the rows that hold text.
- Writes **only the parameters that changed**, in a single update, so small edits are quick.

---

## Requirements

- **UEFN** with Python scripting available (the tool runs from *Tools → Execute Python Script*).
- **Windows** (the bundled `msdfgen` is a Windows executable).
- For the **Text tab**: a material instance of a material that follows the parameter layout described in
  [Material compatibility](#material-compatibility).
- The **Atlas tab** works on its own: its atlases follow the layout in [Atlas format](#atlas-format).

Nothing else to install: `msdfgen` and `fontTools` are embedded in the script and unpacked on first run to
`%LOCALAPPDATA%\DynamicTextStudio`.

---

## Installation

1. Download **`DynamicTextStudio.py`**.
2. Put it anywhere on your computer. Your project folder is fine, but it does not need to be there.
3. In UEFN: **Tools → Execute Python Script**, then pick `DynamicTextStudio.py`.

The window opens inside the editor. Run the script again at any time to reopen it; it will not open twice.

---

## Usage

### Making an atlas
1. Open the **Atlas** tab.
2. **Browse…** to a font file and pick the face / style if the font has several.
3. Choose the **content folder** (Browse… lists your project's folders) and an **asset name**.
4. Press **Generate Atlas**.

The atlas is imported with the right settings, and its glyph widths are stored on the texture.

### Writing text
1. Open the **Text** tab.
2. Select a material instance in the Content Browser and press **Use Selected**, or paste its path.
3. Optionally pick a **font** ("Keep the instance's font" leaves it as is).
4. Choose the **spacing mode**.
5. Type up to four rows and set each row's alignment.
6. Press **Write to Instance**.

The report on the right lists what was written, characters the atlas does not have, rows that were cut, and
icons that were placed.

#### Good to know
- **Changing the spacing mode recompiles that instance once**, because it toggles static switches. Changing the
  text or the font does not.
- If the instance is **open in the material editor**, the tool closes that editor before writing; otherwise the
  editor would overwrite the new values with its old copy the next time you change something there. Reopen it
  to see the result.
- Characters that are not in the atlas are drawn as spaces and listed in the report.
- Rows hold **32 characters**; longer text is cut.
- If a material instance is also driven by Verse at runtime, Verse overwrites these values when the game starts.

---

## Supported characters

```
0-9  A-Z  a-z
. , = : / $ + - " ? # % ! ' ( ) & @ * ; _ [ ] < > { } | \ ^ ~ ° • € £ ¥ ×
space
```

---

## Atlas format

Every atlas the tool makes follows this layout, so any of them can be swapped into the material:

| Property | Value |
|---|---|
| Size | 2048 × 2048 |
| Grid | 10 × 10 cells (204.8 px each), 100 glyphs |
| Glyph order | `0123456789ABCDEFGHIJKLMNOPQRSTUVXWYZ.,= :/$+-"?#%abcdefghijklmnopqrstuvwxyz!'()&@*;_[]<>{}\|\^~°•€£¥×` (index = glyph code) |
| Letter size | "H" is 0.455 of a cell tall, baseline at 0.560 of the cell |
| Placement | each glyph centred horizontally in its cell |
| Field | MTSDF: median of RGB = shape, alpha = true SDF; 0.5 on the edge, 45 px range |
| Glyph widths | metadata tag `DynamicText.GlyphWidths` on the texture: JSON list of 100 numbers in cell units (index 39 = space advance) |

---

## Material compatibility

The **Text tab** writes these parameters on the instance. A compatible material needs them with these exact names:

| Parameter | Type | Meaning |
|---|---|---|
| `Atlas` | Texture | The font atlas |
| `Row{1-4}_{01-32}` | Scalar | Glyph code per slot (`code + 100 × style`, `-1` = empty) |
| `Row{n}_Align` | Scalar | -1 left, 0 center, 1 right |
| `Row{n}_VerticalOffset` | Scalar | Row height (only when vertical centering is on) |
| `Row{n}_Pos01`…`Pos32`, `Row{n}_PosEnd` | Scalar | Letter start positions and row width in atlas cells (proportional mode); `PosEnd < 0` = monospace |
| `Row{n}_ScaleX`, `Row{n}_Spacing`, `GlyphFitX` | Scalar | Read by the tool for the layout math |
| `Icon{1-3}_Row`, `Icon{n}_Slot`, `Icon{n}_Opacity`, `Icon{n}_PropOffsetX` | Scalar | Icon placement |
| `ProportionalSpacing`, `GlyphOverlap` | Static switch | Spacing mode |

The tool checks for `Row1_01`, `Row1_PosEnd` and `Icon1_PropOffsetX` and refuses instances that lack them.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "UEFN not connected" | Run the script from inside UEFN (Tools → Execute Python Script), not by double-clicking it. |
| A row looks too tight or too loose in Proportional mode | The atlas may have no stored widths (the report says so). Regenerate it with the Atlas tab. |
| Text resets after editing the instance | The instance was open in the material editor while writing. Reopen it after writing, then edit. |
| Letters render garbled | The atlas does not follow the [Atlas format](#atlas-format), for example an old 7 × 7 atlas. |

Messages from the tool appear in UEFN's **Output Log** prefixed with `[Dynamic Text Studio]`.

---

## Credits

Dynamic Text Studio bundles two open source components:

- **[msdfgen](https://github.com/Chlumsky/msdfgen)**: multi-channel signed distance field generator.
  MIT License, © 2016 Viktor Chlumsky.
- **[fontTools](https://github.com/fonttools/fonttools)**: font file library. MIT License, © 2017 Just van Rossum.

Their licence texts are shown in the tool's **About** window.

**Fonts are not included.** Every font has its own licence, so make sure yours allows the use you have in mind
(for example, [SIL Open Font License](https://openfontlicense.org/) fonts from Google Fonts are free to embed).

---

## License

[MIT](LICENSE) © 2026 vrtn096. The bundled msdfgen and fontTools keep their own MIT licences (see [Credits](#credits)).
