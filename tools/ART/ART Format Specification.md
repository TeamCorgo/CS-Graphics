# ART / BMX — Read Specification

Dark Reign: The Future of War (Auran, 1997)  
Reconstructed from `SEQUEL.ART` via Ghidra static analysis.

---

## 1. PKX Container Header

24 bytes, all fields little-endian.

| Offset | Size | Value / meaning |
|--------|------|-----------------|
| 0x00   | 4    | `50 4B 58 3A` — magic, must match |
| 0x04   | 4    | `66 19 01 10` — magic, must match |
| 0x08   | 4    | `CF BA AE 9B` — magic, must match |
| 0x0C   | 4    | Compression type (see §2) |
| 0x10   | 4    | Compressed payload size (must equal `file_size − 24`) |
| 0x14   | 4    | Decompressed size in bytes |
| 0x18   | …    | Compressed payload |

---

## 2. Compression Type Field

Split into nibbles. High nibble = first-pass algorithm, low nibble = second-pass (or only).  
If high nibble is non-zero, decompress with it first, then decompress the result with the low nibble.

| ID     | Algorithm |
|--------|-----------|
| `0x01` | ZRL (zero-run RLE) |
| `0x0C` | RLE |
| `0x0E` | LZSS |

All observed `.ART` files use `0x0E` (single-pass LZSS).

---

## 3. LZSS Decompressor (type `0x0E`)

Read the first byte of the payload to set up the loop:

```
first_byte < 8:
    extra_bits = first_byte
    num_chunks = uint16 @ [1]        // little-endian
    if num_chunks == 0xFFFF:
        num_chunks = uint32 @ [3]    // 32-bit override
        data starts at [7]
    else:
        data starts at [3]

first_byte >= 8:
    extra_bits = first_byte & 7
    num_chunks = (first_byte >> 3) - 1
    data starts at [1]
```

Process `num_chunks` full chunks, then one partial chunk of `extra_bits` bits (skip if zero).

### Each full chunk

- Read one flag byte.
- If flag byte == `0x00`: copy next 8 bytes verbatim to output. Done with chunk.
- Otherwise: process 8 bits MSB-first. For each bit:
  - `0` → literal: copy 1 byte from input to output.
  - `1` → back-reference: read 2-byte little-endian token.  
    `offset = (token >> 4) + 1`  
    `length = (token & 0xF) + 3`  
    Copy `length` bytes from `output[current_pos − offset]`.

### Partial final chunk

Same as above but stop after `extra_bits` bits instead of 8.

---

## 4. Decompressed Layout — FrameRecord Array

```
[0x00]  uint32   frame_count
[0x04]  12 bytes Frame 0  (bitmap)
[0x10]  12 bytes Frame 1  (palette — present if frame_count > 1)
[0x1C+] ...      pixel data, then palette data
```

### FrameRecord structure (12 bytes)

| Offset | Size   | Field |
|--------|--------|-------|
| +0     | int32  | `data_offset` — self-relative: data is at `&record + data_offset` |
| +4     | uint16 | Width in pixels (or palette start index for Frame 1) |
| +6     | uint16 | Height in pixels (or palette entry count for Frame 1) |
| +8     | uint8  | Reserved |
| +9     | uint8  | Format flags: low nibble = pixel format (`0xD` = RLE12), high nibble = per-frame compression (0 = none) |
| +10    | 2      | Reserved |

---

## 5. RLE12 Pixel Data (format type `0xD`)

Pixel data starts at `decompressed[4 + frame0.data_offset]`.  
Encoded as `height` scanlines. Each scanline is a sequence of tokens terminated by any byte where `byte & 0x7F == 0` (in practice always `0x00`).

For each token byte `b`:

```
count = b & 0x7F
if count == 0: end of scanline

if b & 0x80:           // run
    color = next byte
    if color == 0:
        dst += count   // transparent skip, write nothing
    else:
        write color × count times
else:                  // literal
    copy next count bytes verbatim
```

Output is 8-bit palette indices, `width × height` bytes, row-major, top-down.

---

## 6. Palette (Frame 1)

Palette data starts at `decompressed[16 + frame1.data_offset]`.

- `frame1.width` field = first palette slot to write (0–255)
- `frame1.height` field = number of entries

Each entry is 4 bytes in **R, G, B, flags** order (Windows `PALETTEENTRY`; flags byte is unused).  
Copy entries into a 256-entry table starting at the given slot index.

Confirmed from `PaletteManager_ApplyEntries`: raw file bytes are cast directly to `LPPALETTEENTRY`.  
The engine converts to BGR for `SetDIBColorTable` internally — the file stores RGB.
