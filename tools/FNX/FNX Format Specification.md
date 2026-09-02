# `.FNX` File Format Specification

`.FNX` is a bitmap font format containing font metrics, character widths, glyph offsets, and glyph bitmap data.

All multi-byte values are **little-endian**.

An `.FNX` file may optionally be wrapped in a `PKX:` compressed resource container.

---

## 1. File Structure

An `.FNX` file consists of either:

- a raw FNX payload, or
- a `PKX:` container containing a compressed FNX payload.

When a `PKX:` container is present, decompression produces the FNX payload described in Section 2.

---

## 2. Optional `PKX:` Container

The container header is 24 bytes (`0x18`).

| Offset | Size | Type | Field | Description |
|---:|---:|---|---|---|
| `0x00` | 4 | `uint32` | `magic1` | `0x3A584B50` (`PKX:`) |
| `0x04` | 4 | `uint32` | `magic2` | `0x10011966` |
| `0x08` | 4 | `uint32` | `magic3` | `0x9BAEBACF` |
| `0x0C` | 4 | `uint32` | `codecId` | Compression codec |
| `0x10` | 4 | `uint32` | `compressedSize` | Size of compressed data |
| `0x14` | 4 | `uint32` | `decompressedSize` | Size after decompression |
| `0x18` | — | bytes | `compressedData` | Compressed FNX payload |

`compressedSize` covers all data following the 24-byte header.

If the three magic values are absent, the file is treated as an uncompressed FNX payload.

### 2.1 Compression Codecs

| Codec ID | Format |
|---:|---|
| `1` | RLE |
| `12` (`0x0C`) | Extended RLE |
| `14` (`0x0E`) | LZSS |

---

# 3. FNX Payload Header

The FNX payload begins with a 40-byte (`0x28`) header.

| Offset | Size | Type | Field | Description |
|---:|---:|---|---|---|
| `0x00` | 4 | `char[4]` | `magicTag` | `"FNX:"` |
| `0x04` | 4 | `uint32` | `reserved` | Reserved |
| `0x08` | 4 | — | — | Unused |
| `0x0C` | 4 | `int32` | `glyphHeight` | Height of each glyph in pixels |
| `0x10` | 4 | `int32` | `underlineRow` | Row used for underline rendering |
| `0x14` | 4 | `int32` | `firstCharCode` | Character code of the first glyph |
| `0x18` | 4 | `int32` | `charCount` | Number of glyphs |
| `0x1C` | 4 | `uint32` | `offsetTableOff` | Offset of glyph offset table |
| `0x20` | 4 | `uint32` | `widthTableOff` | Offset of character width table |
| `0x24` | 4 | `uint32` | `bitmapDataOff` | Offset of glyph bitmap data |

The payload begins with the four-byte signature:

```text
FNX:
```

The three table offsets are **file-relative byte offsets**.

---

# 4. Character Range

The font contains `charCount` consecutive character codes beginning at `firstCharCode`.

The character represented by glyph index `i` is:

**`firstCharCode + i`**

The stored character range is therefore:

**`firstCharCode` through `firstCharCode + charCount - 1`**

---

# 5. Glyph Offset Table

The glyph offset table is located at `offsetTableOff`.

It contains one signed 32-bit offset for each glyph:

**`int32[charCount]`**

The entry at index `i` corresponds to:

**`firstCharCode + i`**

Each entry is a byte offset relative to `bitmapDataOff`.

The absolute position of a glyph's bitmap is therefore:

**`bitmapDataOff + glyphOffset[i]`**

There is no per-glyph size field. The bitmap size is determined by the glyph width, font mode, and glyph height.

---

# 6. Character Width Table

The character width table is located at `widthTableOff`.

It always contains 256 unsigned bytes:

**`uint8[256]`**

Each entry corresponds directly to a character code from `0x00` through `0xFF`.

The width for character `C` is stored at:

**`widthTable[C]`**

The table is indexed by the raw character code and is not relative to `firstCharCode`.

A width of `0` indicates a zero-width glyph.

Character `0x20` (space) may have a nonzero width but has no rendered bitmap.

---

# 7. Bitmap Data

Glyph bitmap data begins at `bitmapDataOff`.

Each glyph is located using its corresponding glyph offset.

There is no per-glyph bitmap length field.

Bitmap size is determined as follows:

| Font Mode | Format | Bytes per Row | Bitmap Size |
|---|---|---:|---:|
| `1` | 1bpp | `(width + 7) / 8` | `rowStride × glyphHeight` |
| Other | 8bpp | `width` | `width × glyphHeight` |

---

## 7.1 1bpp Bitmap Format

When `fontMode = 1`, pixels are stored as packed bits.

Each row occupies:

**`ceil(width / 8)` bytes**

Bits are stored **MSB-first**.

Within each byte:

| Bit | Pixel |
|---:|---|
| 7 | First pixel |
| 6 | Second pixel |
| 5 | Third pixel |
| 4 | Fourth pixel |
| 3 | Fifth pixel |
| 2 | Sixth pixel |
| 1 | Seventh pixel |
| 0 | Eighth pixel |

A set bit represents a rendered pixel.

A clear bit represents transparency.

---

## 7.2 8bpp Bitmap Format

When `fontMode` is any value other than `1`, each pixel occupies one byte.

Pixels are stored row-major.

Each row contains exactly `width` bytes.

The bitmap contains:

**`width × glyphHeight` bytes**

A value of `0x00` represents transparency.

Nonzero values are **palette/color indices**, not alpha values.

The actual color represented by a pixel value is supplied by an external color-remapping table or palette.

Consequently, different nonzero pixel values can represent different colors within the same glyph.

The `.FNX` file does not contain RGB color definitions.

---

# 8. Font Metrics

### `glyphHeight`

The number of pixel rows in every glyph bitmap.

### `underlineRow`

The row position used when rendering underlined text.

### `firstCharCode`

The character code associated with glyph index `0`.

### `charCount`

The number of glyphs stored in the font.

### Character Width

Each character has an independent advance width stored in the 256-byte width table.

---

# 9. Typical Data Layout

A typical FNX payload is arranged as:

```text
+-----------------------------+
| FNX Header                  | 0x28 bytes
+-----------------------------+
| Glyph Offset Table          | charCount × 4
+-----------------------------+
| Character Width Table       | 256 bytes
+-----------------------------+
| Glyph Bitmap Data           |
+-----------------------------+
```

The three regions are addressed independently through their header offsets.

A typical 224-glyph font therefore has:

- 40-byte header
- 896-byte glyph offset table
- 256-byte character width table
- Remaining bytes containing glyph bitmaps

---

# 10. Rendering Semantics

The FNX format provides the font data used by the renderer. Some rendering behavior is external to the file.

### Character Lookup

Characters within the stored range use their corresponding glyph.

Characters outside the stored range have no glyph.

### Character Advance

Character advance is determined by the character's entry in the width table.

### Space

The space character uses its width for positioning but does not render bitmap pixels.

### 1bpp Rendering

Each set bit represents a pixel.

### 8bpp Rendering

Each nonzero byte represents a palette/remap index.

The corresponding color is obtained externally rather than stored in the FNX file.

### Rendering Effects

Effects such as bold, italic/skew, underline, and other text rendering modes are applied by the renderer and are not stored in the FNX format.

---

# 11. LZSS Compression — Codec `14`

Codec `14` uses a 4096-byte sliding window.

### Stream Header

Two header forms are supported.

#### Explicit Form

For `headerByte < 8`:

| Size | Field |
|---:|---|
| 1 byte | `headerByte` |
| 2 bytes | `blockCount` |

If the 16-bit block count is `0xFFFF`, a 32-bit block count follows.

#### Packed Form

For `headerByte >= 8`:

- Lower 3 bits specify `segmentFlag`.
- Upper 5 bits encode the block count.

### Token Groups

Compressed data is divided into groups controlled by a flag byte.

A flag byte of `0x00` represents eight literal bytes.

Otherwise, the flag byte contains eight control bits processed MSB-first:

- `0` — literal byte
- `1` — back-reference

Back-reference tokens are 16-bit little-endian values.

The token contains:

- 12-bit distance
- 4-bit length

Decoded ranges:

| Parameter | Range |
|---|---:|
| Distance | `1–4096` bytes |
| Length | `3–18` bytes |

Back-references may overlap the source region.

A final segment-continuation group may contain fewer than eight control bits as specified by `segmentFlag`.

---

# 12. Extended RLE — Codec `12`

Codec `12` uses control bytes with both short and extended forms.

| Control | Operation |
|---|---|
| `0x01–0x7F` | Literal copy |
| `0x00` | Short run-fill |
| `0x81–0xFF` | Short skip |
| `0x80` | Extended operation |

### Short Operations

`0x01–0x7F`:

- Copies `C` literal bytes.

`0x00`:

- Followed by a count byte.
- Followed by a fill byte.
- Repeats the fill byte for the specified count.

`0x81–0xFF`:

- Skips `C & 0x7F` output bytes.

### Extended Operations

`0x80` is followed by a 16-bit little-endian value.

| Value | Operation |
|---:|---|
| `0x0000` | End of stream |
| `0x0001–0x7FFF` | Extended skip |
| `0x8000–0xBFFF` | Extended literal |
| `0xC000–0xFFFF` | Extended run-fill |

Extended literal count:

**`value - 0x8000`**

Extended run-fill count:

**`value - 0xC000`**

An extended run-fill is followed by one fill byte.

---

# 13. RLE — Codec `1`

Codec `1` uses a single-byte control scheme.

| Control | Operation |
|---|---|
| `0x00` | End of stream |
| `0x80` | End of stream |
| `0x01–0x7F` | Literal copy |
| `0x81–0xFF` | Run-fill |

For a run-fill:

- Count = `C & 0x7F`
- The following byte is the value to repeat.

Run lengths range from 1 to 127 bytes.

---

# 14. Format Overview

```text
FNX File
│
├── Optional PKX Container
│   ├── Container Magic
│   ├── Codec ID
│   ├── Compressed Size
│   ├── Decompressed Size
│   └── Compressed FNX Payload
│
└── FNX Payload
    │
    ├── Header
    │   ├── FNX Signature
    │   ├── Reserved Fields
    │   ├── Glyph Height
    │   ├── Underline Row
    │   ├── First Character Code
    │   ├── Character Count
    │   ├── Glyph Offset Table Offset
    │   ├── Width Table Offset
    │   └── Bitmap Data Offset
    │
    ├── Glyph Offset Table
    │
    ├── 256-Byte Character Width Table
    │
    └── Glyph Bitmap Data
        ├── 1bpp packed pixels
        └── 8bpp palette indices
```

The core relationship between the structures is:

**Character code → width table entry**

**Character code → glyph index → glyph offset → bitmap data**

The FNX file stores the glyph geometry and pixel indices; rendering color and visual effects are supplied externally.