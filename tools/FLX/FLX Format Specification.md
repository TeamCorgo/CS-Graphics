# FLX File Format Specification
**Version:** 2.0  
**Derived from:** Ghidra static analysis of game binary (Win32, DirectDraw/WinG)  
**Source functions:** `FLX_Load`, `FLX_TickFrame`, `RLE_Decompress`, `LZSS_Decompress`,  
`FLX_ApplyPaletteChunk`, `FLX_FlushPaletteToScreen`, `FLX_UpdateAndBlit`,  
`FLX_OnFrameAdvance`, `LoadResourceFile_WithDecompression`, `BlitBitmapRect_RawCopy`  
**Target audience:** Render program implementors  

---

## Table of Contents

1. [Overview](#1-overview)
2. [PKX Container (on-disk wrapper)](#2-pkx-container-on-disk-wrapper)
3. [File Header](#3-file-header)
4. [Frame Structure](#4-frame-structure)
5. [Chunk Types](#5-chunk-types)
   - 5.1 [Type 4 — COLOR_256](#51-type-4--color_256)
   - 5.2 [Type 7 — DELTA_FLI](#52-type-7--delta_fli)
   - 5.3 [Type 11 — COLOR_64](#53-type-11--color_64)
   - 5.4 [Type 13 — BLACK](#54-type-13--black)
   - 5.5 [Type 15 — DELTA_FLC](#55-type-15--delta_flc)
   - 5.6 [Type 16 — FULL_RLE](#56-type-16--full_rle)
   - 5.7 [Type 18 — COPY](#57-type-18--copy)
   - 5.8 [Type 19 — MINI](#58-type-19--mini)
   - 5.9 [Type 100 — GAME_RLE](#59-type-100--game_rle)
   - 5.10 [Type 101 — GAME_RLE_DELTA](#510-type-101--game_rle_delta)
   - 5.11 [Type 102 — DIRTY_RECT](#511-type-102--dirty_rect)
   - 5.12 [Type 104 — LZSS_RLE](#512-type-104--lzss_rle)
6. [Pixel Buffer Layout](#6-pixel-buffer-layout)
7. [Palette System](#7-palette-system)
8. [Playback Engine](#8-playback-engine)
9. [Audio Cue Table](#9-audio-cue-table)
10. [PLX Palette Files](#10-plx-palette-files)
11. [Render Program Implementation Guide](#11-render-program-implementation-guide)
12. [Confirmed Assumptions](#12-confirmed-assumptions)
13. [Known Unknowns](#13-known-unknowns)

---

## 1. Overview

FLX is an 8-bit indexed colour animation format used by the game engine. It is a superset of the standard Autodesk FLIC format (FLI/FLC), adding three game-specific chunk types and an on-disk PKX compression wrapper.

### Magic Values

| Magic  | Hex    | Description                               |
|--------|--------|-------------------------------------------|
| FLI    | 0xAF11 | Original Autodesk Animator (speed in jiffies) |
| FLC    | 0xAF12 | Autodesk Animator Pro (speed in milliseconds) |
| FLX    | 0xAF20 | Game engine variant A                     |
| FLX2   | 0xAF21 | Game engine variant B                     |

FLX and FLX2 are structurally identical. The game allocates a pixel surface for both. All frame and chunk formats are the same.

### Key Properties
- **Colour depth:** 8-bit indexed (palette of 256 entries)
- **Pixel storage:** Bottom-up WinG DIB convention (row 0 in memory = bottom of screen)
- **Delta encoding:** Frames accumulate — you cannot seek without replaying from frame 0
- **On-disk:** Almost always wrapped in the PKX LZSS container
- **In-memory:** Raw FLX after decompression

---

## 2. PKX Container (on-disk wrapper)

Nearly all FLX and PLX files on disk are wrapped in the PKX container. Detect and decompress before parsing.

### PKX Header (24 bytes, all little-endian)

```
Offset  Size  Field      Value / Notes
------  ----  ---------  --------------------------------------------------
+0      u32   magic1     0x3A584B50  ('PKX:' as ASCII LE)
+4      u32   magic2     0x10011966
+8      u32   magic3     0x9BAEB0CF or 0x9BAEBACF (both valid, seen in files)
+12     u32   compType   Compression method (see below)
+16     u32   compSize   Byte size of compressed payload (= fileSize - 24)
+20     u32   rawSize    Expected byte size of decompressed output
```

Data begins at byte 24.

### Compression Types

| compType | Meaning                          |
|----------|----------------------------------|
| 0xC      | Game RLE only                    |
| 0xE      | LZSS only                        |
| 0x1      | RLE_ZeroRunOptimised             |
| hi/lo    | Two-pass: hi nibble first, lo second |

For two-pass (e.g. `0xEC`): apply the hi-nibble decompressor first to produce an intermediate buffer, then apply the lo-nibble decompressor to that buffer to produce the final output.

### LZSS Decompressor (compType 0xE)

```
Header byte b0:
  if b0 >= 8:
    groupCount = (b0 >> 3) - 1
    data starts at src[1]
  else:
    cnt16 = u16le(src[1..2])
    if cnt16 == 0xFFFF:
      groupCount = u32le(src[3..6])
      data starts at src[7]
    else:
      groupCount = cnt16
      data starts at src[3]

Decode loop (runs groupCount+1 times):
  ctrl = next byte
  if ctrl == 0:
    Copy next 8 bytes verbatim to output
  else:
    For each of 8 bits in ctrl, MSB first:
      bit = 0 → LITERAL: copy 1 byte from input to output
      bit = 1 → BACKREF: read u16le
                  back   = (u16 >> 4) + 1
                  length = (u16 & 0xF) + 3
                  Copy `length` bytes from output[-back] (may overlap)
```

### Game RLE Decompressor (compType 0xC)

Same algorithm as chunk type 100 (GAME_RLE). See §5.9.

---

## 3. File Header

128 bytes. All fields little-endian. Frame data begins at byte 128 always — `oframe1` is used only for file-streaming and is not needed for in-memory decoding.

```
Offset  Size  Field        Notes
------  ----  -----------  ---------------------------------------------------
+0      u32   fileSize     Total file bytes (after decompression)
+4      u16   magic        0xAF11/AF12/AF20/AF21
+6      u16   frameCount   Playable frames (NOT counting ring/loop frame)
+8      u16   width        Pixels wide
+10     u16   height       Pixels tall
+12     u16   depth        Bits per pixel (always 8)
+14     u16   flags        Bit 0: has ring frame; Bit 1: has oframe2
+16     u32   speed        FLI: jiffies (÷70 → ms); FLC/FLX: milliseconds
                           0 → treat as 70ms (~14fps)
+20     u16   reserved
+22     u32   created      Creation timestamp (FLC/FLX)
+26     u32   creator      Creator program ID (FLC/FLX)
+30     u32   updated      Last update timestamp
+34     u32   updater      Updater ID
+38     u16   aspectX      Pixel aspect ratio X (cosmetic only)
+40     u16   aspectY      Pixel aspect ratio Y (cosmetic only)
+42     u16   extFlags     Extended flags (FLC only)
+44     u16   keyFrames    Key frame count (FLC only)
+46     u16   totalFrames  Total including ring frame (FLC only)
+48     u32   reqMemory    Required memory hint (FLC only)
+52     u16   maxRegions   Max dirty regions (FLC only)
+54     u16   transpNum    Transparent colour count (FLC only)
+56..79 --    reserved     Zero padding
+80     u32   oframe1      File offset of frame 0 (file-streaming only; always 128)
+84     u32   oframe2      File offset of loop-back frame (ring frame target)
+88     u16   palBits      Palette bits (FLC only)
+90..127 --   reserved     Zero padding
```

### Speed Conversion

```
if magic == 0xAF11 (FLI):
    ms_per_frame = round(speed * 1000 / 70)   // jiffies to ms
else:
    ms_per_frame = speed                       // already ms
if ms_per_frame == 0:
    ms_per_frame = 70
```

### oframe2 Usage

`oframe2` is the **loop-back target** — the file offset of the frame to seek to after the last playable frame completes. Frame[0] is always decoded first regardless of `oframe2`. On loop, seek to `oframe2` and resume forward from there.

---

## 4. Frame Structure

Frames are stored sequentially starting at **byte 128**. Walk them in order.

### Frame Header (16 bytes)

```
Offset  Size  Field       Notes
------  ----  ----------  ---------------------------------------------------
+0      u32   frameSize   Total frame bytes including this 16-byte header
+4      u16   magic       0xF1FA = data frame; 0xF1FB = audio/prefix frame
+6      u16   chunkCount  Number of chunks in this frame
+8      u64   reserved    Always zero
```

### Frame Magic Values

| Magic  | Meaning                                              |
|--------|------------------------------------------------------|
| 0xF1FA | Data frame — contains palette and/or pixel chunks    |
| 0xF1FB | Audio/prefix frame — skip entirely; no pixel content |

### Frame Walk Algorithm

```
pos = 128   // always start here

while pos + 16 <= fileSize:
    frameSize  = u32le(pos + 0)
    frameMagic = u16le(pos + 4)
    chunkCount = u16le(pos + 6)

    if frameSize == 0 or pos + frameSize > fileSize:
        break

    if frameMagic == 0xF1FA:
        add pos to frame index
        decode chunks at pos + 16
    // 0xF1FB: consume frameSize, continue

    pos += frameSize
```

### Playable Frame Count

The game plays exactly `frameCount` frames (from header+6), then loops back to `oframe2`. The ring frame at `oframe2` — and any frames indexed beyond `frameCount` — are not displayed. Use `min(indexed_count, frameCount)` as your playable total.

---

## 5. Chunk Types

### Chunk Header (6 bytes)

```
Offset  Size  Field      Notes
------  ----  ---------  -----------------------------------------------
+0      u32   chunkSize  Total chunk bytes including this 6-byte header
+4      i16   chunkType  Signed integer; determines decoder
```

Chunk data begins at `chunkBase + 6`. Next chunk begins at `chunkBase + chunkSize`.

---

### 5.1 Type 4 — COLOR_256

**8-bit palette update.** Updates a subset of the 256-entry palette.

```
+0  i16  packetCount   Number of packets (signed)

Packets (repeated packetCount times):
  u8  skip    Add to current palette index
  u8  count   Entries to set (0 = 256)
  [count × 3 bytes]  R, G, B  (8-bit each, range 0-255)
```

**⚠ GAME-SPECIFIC:** After all packets are applied, the engine zeroes entry 0:

```c
memset(palette[0], 0, 4);   // RTL_MemSet confirmed in FLX_ApplyPaletteChunk
```

Palette entry 0 is always `(0, 0, 0)` after a type-4 chunk. Do not skip this step.

---

### 5.2 Type 7 — DELTA_FLI

**Word-oriented delta, standard FLI format.**

```
+0  u16  lineCount   Number of lines to update

Per line:
  Read signed i16 words until a non-negative value is found:
    bits[15:14] == 10  → skip (word & 0x3FFF) lines
    bits[15:14] == 11  → set last pixel of current line to (word & 0xFF)
  Non-negative word = packet count for this line

Per packet:
  u8   colSkip   Columns to skip
  i8   count
    count > 0  → LITERAL: copy count word-pairs (2 bytes each) from input
    count < 0  → RLE: repeat next word-pair |count| times
```

---

### 5.3 Type 11 — COLOR_64

**6-bit palette update.** Same packet structure as type 4, but components are 6-bit (range 0-63). Shift left 2 bits to convert to 8-bit: `r8 = r6 << 2`.

**⚠ GAME-SPECIFIC:** Entry 0 is **NOT** zeroed after type-11 chunks. Only type 4 zeroes entry 0.

---

### 5.4 Type 13 — BLACK

Fill entire pixel buffer with colour index 0. No data bytes follow the chunk header.

---

### 5.5 Type 15 — DELTA_FLC

**Byte-oriented delta, standard FLC format.**

```
+0  u16  lineCount    Number of lines to update
+2  u16  firstLine    First line index (0-based)

Per line (lineCount lines starting at firstLine):
  u8  packetCount

  Per packet:
    u8   skip    Columns to skip
    i8   count
      count > 0  → LITERAL: copy count bytes from input to output
      count < 0  → RLE: repeat next byte |count| times
```

---

### 5.6 Type 16 — FULL_RLE

**Full-frame RLE, standard FLC format.** Covers all rows.

Per row:
```
  u8  (ignored)   Legacy packet-count byte; discard
  Packets until row is full:
    i8  count
      count > 0  → LITERAL: copy count bytes
      count < 0  → RLE: repeat next byte |count| times
      count == 0 → end of row
```

---

### 5.7 Type 18 — COPY

**Raw uncompressed frame.** Data is `width × height` bytes, row-major, top-to-bottom. Memcpy directly to pixel buffer.

---

### 5.8 Type 19 — MINI

Thumbnail frame (typically 1/16 scale). **Ignore.** Not used for playback.

---

### 5.9 Type 100 — GAME_RLE

**Game-specific full-frame RLE.** Decoded by `RLE_Decompress` in the binary.

This is a **continuous byte stream** with no per-line framing. The output pointer walks linearly through the pixel buffer from position 0.

```
Read byte b continuously until output is full or END opcode:

  b interpreted as signed char (C 'char' type):

  (char)b >= 1   [b = 0x01..0x7F]:
      LITERAL — count = b & 0x3FFF
      Copy next `count` bytes from input to output
      Advance input by 1 (count byte) + count (data bytes)

  b == 0x00:
      RLE FILL — count = nextByte & 0x3FFF, value = byte after that
      Fill `count` output pixels with `value`
      Advance input by 3

  b == 0x80:
      EXTENDED — read signed i16 v from next 2 bytes (little-endian)

      v == 0:          END OF FRAME — stop decoding immediately
      v >= 1:          SKIP — advance output pointer by v pixels (transparent)
      v < 0:
          uMasked = (uint16_t)v & 0x7FFF
          if uMasked < 0x4000:
              EXTENDED LITERAL — copy next uMasked bytes from input to output
          else:
              EXTENDED RLE — count = uMasked & 0x3FFF, value = next byte
              Fill count output pixels with value

  b = 0x81..0xFF   [(char)b = -127..-1]:
      SKIP — advance output pointer by (b & 0x7F) pixels
      Advance input by 1 (only the opcode byte; no data bytes follow)
```

**⚠ CRITICAL:** The comparison `(char)b >= 1` is a **signed** byte test. Byte value `0x80` (128 unsigned, -128 signed) is **negative** and must fall through to the `b == 0x80` EXTENDED branch — NOT the literal branch. This is the single most common source of decoder bugs.

In C: `if ((signed char)b >= 1)`  
In JavaScript: `const bSigned = b < 128 ? b : b - 256; if (bSigned >= 1)`

The frame terminates on `0x80 0x00 0x00` (EXTENDED with i16 = 0). It also terminates implicitly when the output pointer reaches `width × height`.

---

### 5.10 Type 101 — GAME_RLE_DELTA

**GAME_RLE with XY blit offset.**

```
+0  u32  destX   Destination X coordinate
+4  u32  destY   Destination Y coordinate
+8  ...  GAME_RLE stream (identical to type 100)
```

The engine decodes the RLE stream into a temporary surface, then blits it to the main pixel buffer at `(destX, destY)` via `BlitFullBitmap_ToDest`. For a renderer using a single pixel buffer, decoding the RLE stream directly into the main buffer produces equivalent results, since SKIP opcodes preserve existing pixels.

---

### 5.11 Type 102 — DIRTY_RECT

**Metadata only. Contains no pixel data.**

```
+0  u32  x
+4  u32  y
+8  u32  width
+12 u32  height
```

Used by the engine's dirty-rect system to track which screen regions changed. A renderer doing full-frame redraws can safely **skip this chunk entirely**.

---

### 5.12 Type 104 — LZSS_RLE

**Two-stage decompression: LZSS → GAME_RLE.**

Stage 1: Apply the LZSS decompressor (same algorithm as PKX §2) to the chunk data. Output goes into a scratch buffer.

Stage 2: Apply the GAME_RLE decompressor (type 100 §5.9) to the scratch buffer.

The LZSS scratch buffer must be at least `width × height` bytes. In practice, for 640×480 frames, allocate at least **307,200 bytes**.

---

## 6. Pixel Buffer Layout

### Memory Layout

The pixel buffer is `width × height` bytes, 1 byte per pixel (palette index).

**⚠ BOTTOM-UP (WinG DIB convention):** The engine renders via WinG (Windows Graphics library, `WinGRecommendDIBFormat`). WinG DIB pixel buffers are bottom-up — row 0 in memory corresponds to the **bottom** row on screen.

All GAME_RLE and LZSS_RLE decoders write top-to-bottom into memory (first bytes = visual top of image). Because the display is bottom-up, the visual top of the image is at the **end** of the memory buffer.

**For correct rendering:** when converting the indexed pixel buffer to RGBA for display, flip the row order:

```
for y in 0..height-1:
    src_row = (height - 1 - y) * width   // read from end of buffer
    dst_row = y * width                   // write to start of output
    for x in 0..width-1:
        rgba[dst_row + x] = palette[pixels[src_row + x]]
```

---

## 7. Palette System

### Palette Buffer

256 entries, 4 bytes each: `R, G, B, A`. Alpha is always `0x00` in the game's internal palette buffer (confirmed from `FLX_ApplyPaletteChunk`). For rendering, use `A = 255`.

### Palette Update Flow

1. **Initialise:** Fill with a grey ramp or zeros before decoding frame 0.
2. **Per frame:** Apply palette chunks (type 4 or 11) first, in chunk order.
3. **After type-4 chunk:** Zero entry 0 (`palette[0] = {0, 0, 0, 0}`).
4. **Apply pixels:** Use current palette to map pixel indices to RGBA.

### Hardware Palette Range (game only)

The game clamps hardware palette writes to slots 10–245 (`SetPaletteRange_ClampedToUsableSlots`) to avoid overwriting Windows system colours. A standalone renderer may use all 256 entries freely.

### PLX Palette Override

The game loads `.PLX` palette files before starting FLX playback. The PLX sets all 256 palette entries as a baseline, then FLX type-4 chunks partially overwrite specific slots per frame. See §10.

---

## 8. Playback Engine

### Initialisation

```
1. Detect and decompress PKX container if present (§2)
2. Parse file header (§3)
3. Walk all 0xF1FA frames from byte 128, build frame index
4. Playable frames = min(indexed_count, header.frameCount)
5. Find loopFrame = index of frame at offset oframe2
6. Allocate pixel buffer: width × height bytes, fill with 0
7. Initialise palette (grey ramp or zeros)
8. If a PLX palette is available, apply it now
9. Decode frame[0] (always; even when oframe2 > frame[0] offset)
```

### Per-Frame Decode

```
for each chunk in current frame (in order):
    apply chunk to pixel buffer and/or palette
after all chunks:
    if palette changed (type 4 or 11 chunk was present):
        flush palette to renderer
render pixel buffer to screen (with row flip — see §6)
wait speedMs milliseconds
```

### Loop Control

The game engine counter (`param_1[0x11]`) is 1-based. It increments after each frame. When it would reach `frameCount + 1`, it resets to 1 and seeks to `oframe2`.

In 0-based frame indices:
```
if cur + 1 >= playableFrameCount:
    // loop wrap
    reset pixel buffer to 0
    reset palette (apply PLX if present)
    re-decode frames [0 .. loopFrame-1] to rebuild delta state
    cur = loopFrame
else:
    cur += 1

decode frame[cur]
```

**Loop count** (`param_1[0x13]`):
- `-1` (0xFFFFFFFF): infinite loop (all ambient/idle FLX files use this)
- `0`: play once, stop after last frame
- `N`: play N times, then stop

Return flag `0x04` signals animation complete.

### Return Flags (from `FLX_UpdateAndBlit`)

| Bit  | Meaning                                    |
|------|--------------------------------------------|
| 0x01 | New frame decoded this tick                |
| 0x04 | Animation complete (loop count exhausted)  |
| 0x08 | Frame counter just wrapped to loopFrame    |
| 0x10 | Frame index reset                          |
| 0x20 | Palette chunk applied — flush palette now  |

---

## 9. Audio Cue Table

Audio is not embedded in FLX files. The game attaches a separate cue table via `FLX_SetAudioCueTable`. The FLX player fires audio cues frame-accurately during `FLX_OnFrameAdvance`, called on every new frame tick.

### Cue Entry Structure (8 bytes, null-terminated)

```
u16  soundId     Sound asset ID to play
u16  frameIndex  1-based frame counter value to match
u8   volume      0–127
u8   pan         -64 to +63 encoded as unsigned byte
u8   flagsLo     Bit 6: loop
u8   flagsHi     Bit 7: preload on attach; Bit 5: use QueueAndPlay; Bit 6: loop
```

Table is null-terminated: first entry with `soundId == 0` ends the list.

### Flag Bits (flagsHi)

| Bit | Value | Meaning                                  |
|-----|-------|------------------------------------------|
| 7   | 0x80  | Preload sound asset when table is attached |
| 5   | 0x20  | Use `QueueAndPlay` instead of `PlaySoundEffect` |
| 6   | 0x40  | Loop the sound                           |

### Pan Decode

```c
int8_t pan_signed = (int8_t)pan_unsigned;   // sign-extend: 0-63=right, 64-127→-64..-1=left
```

---

## 10. PLX Palette Files

PLX files store a full 256-entry RGBA palette. Used to set the baseline palette before FLX playback, especially for screens where the FLX animation only covers part of the palette.

### Raw PLX Format

Exactly **1024 bytes**: 256 × 4 bytes, `R, G, B, A` per entry. Alpha is always `0x00` in the game's storage format.

### On-disk

PLX files are almost always PKX-compressed (§2). Detect the PKX header, decompress, then read the raw 1024-byte palette.

### Application Order

```
1. Load PLX → apply all 256 palette entries as baseline
2. Start FLX playback
3. Per frame: type-4/11 chunks overwrite specific palette slots
   (the PLX baseline remains for slots the FLX doesn't touch)
```

If you want the PLX to unconditionally override FLX palette chunks, re-apply the PLX after each frame's chunks are processed.

---

## 11. Render Program Implementation Guide

### Minimal Decoder Checklist

```
[ ] Detect PKX header (magic1 = 0x3A584B50)
[ ] Decompress PKX payload (LZSS for compType 0xE, RLE for 0xC)
[ ] Validate FLX magic (0xAF11/12/20/21)
[ ] Parse header: width, height, frameCount, speed, oframe2
[ ] Walk 0xF1FA frames from byte 128 — build offset index
[ ] Allocate pixel buffer: width × height bytes
[ ] Allocate RGBA output buffer: width × height × 4 bytes
[ ] Allocate palette: 256 × 4 bytes (RGBA)
[ ] Implement type-4 palette chunk with entry-0 zeroing
[ ] Implement type-100 GAME_RLE with SIGNED byte opcode dispatch
[ ] Implement type-104 LZSS → GAME_RLE (two-stage)
[ ] Implement row-flip in indexed→RGBA conversion (bottom-up)
[ ] Implement loopFrame seek on wrap (re-decode from 0)
```

### Common Pitfalls

**Pitfall 1 — Byte 0x80 in GAME_RLE decoded as literal**

The `(char)b >= 1` check is signed. `0x80 = 128 unsigned = -128 signed` — it is NOT a literal. It is the EXTENDED opcode. Every decoder that uses unsigned comparison catches this bug.

**Pitfall 2 — Image appears upside-down**

WinG stores rows bottom-up. Flip rows in your indexed→RGBA conversion. Do NOT flip the pixel buffer itself; flip only during output rendering.

**Pitfall 3 — Black screen on delta frames**

Delta frames (type 100, 101, 104) only update changed regions. You must maintain an accumulation buffer across frames. Never clear the pixel buffer between frames — only clear when looping back to frame 0.

**Pitfall 4 — Last frame duplicated**

The ring frame (one frame beyond `frameCount`) is a structural frame stored after the last playable frame. Cap playback at `frameCount`, not `total indexed frames`.

**Pitfall 5 — Wrong frame count after seeking**

`frameCount` in the header is the playable count excluding the ring frame. When seeking to frame N, re-decode from frame 0 through frame N (accumulating delta state) — there is no random access.

**Pitfall 6 — Speed of 0**

Treat speed = 0 as 70ms per frame (~14fps).

**Pitfall 7 — oframe2 not found in frame index**

If the offset in `oframe2` does not match any indexed frame, default to looping from frame 0.

**Pitfall 8 — LZSS scratch buffer too small**

Allocate at least `width × height` bytes for the LZSS scratch buffer used by type-104 chunks. For 640×480 this is 307,200 bytes.

### Recommended Decode Loop

```python
def decode_flx(path):
    buf = open(path, 'rb').read()
    
    # Step 1: strip PKX container
    if buf[:4] == b'PKX:':
        comp_type = u32le(buf, 12)
        raw_size  = u32le(buf, 20)
        payload   = buf[24:]
        buf = pkx_decompress(comp_type, payload, raw_size)

    # Step 2: parse header
    magic      = u16le(buf, 4)
    frameCount = u16le(buf, 6)
    W, H       = u16le(buf, 8), u16le(buf, 10)
    speedMs    = speed_to_ms(magic, u32le(buf, 16))
    oframe2    = u32le(buf, 84)

    # Step 3: index frames
    frames = index_frames(buf)           # list of byte offsets to 0xF1FA frames
    playable = min(len(frames), frameCount)
    loopFrame = find_frame_at(frames, oframe2)  # default 0 if not found

    # Step 4: allocate buffers
    pixels  = bytearray(W * H)
    palette = bytearray(256 * 4)
    init_palette(palette)                # grey ramp or zeros

    # Step 5: decode and display
    cur = 0
    while True:
        decode_frame(buf, frames[cur], pixels, palette, W, H)
        display(to_rgba_flipped(pixels, palette, W, H))
        sleep(speedMs / 1000.0)

        cur += 1
        if cur >= playable:              # loop wrap
            pixels[:] = b'\x00' * (W * H)
            init_palette(palette)
            for f in range(loopFrame):   # re-accumulate delta state
                decode_frame(buf, frames[f], pixels, palette, W, H)
            cur = loopFrame

def decode_frame(buf, frame_offset, pixels, palette, W, H):
    nc = u16le(buf, frame_offset + 6)
    pos = frame_offset + 16
    for _ in range(nc):
        chunk_size = u32le(buf, pos)
        chunk_type = i16le(buf, pos + 4)
        data = buf[pos + 6 : pos + chunk_size]
        apply_chunk(chunk_type, data, pixels, palette, W, H)
        pos += chunk_size
```

---

## 12. Confirmed Assumptions

All items below are confirmed by direct Ghidra binary analysis of the game executable. Source function is noted.

| # | Assumption | Source |
|---|-----------|--------|
| 1 | Frame walk always starts at byte 128 (`param_1[0x15] = 0x80`) | `FLX_Load` |
| 2 | `oframe2` is loop-back target only; frame[0] always decoded first | `FLX_TickFrame` |
| 3 | Type-4 palette chunk zeros entry 0 after applying (`RTL_MemSet`) | `FLX_ApplyPaletteChunk` |
| 4 | Type-11 does NOT zero entry 0 | `FLX_ApplyPaletteChunk` |
| 5 | GAME_RLE uses signed byte comparison: `(char)b >= 1` | `RLE_Decompress` |
| 6 | LZSS: `groupCount = (b0>>3)-1`; loop runs `groupCount+1` times | `LZSS_Decompress` |
| 7 | LZSS: backref `back=(u16>>4)+1`, `length=(u16&0xF)+3` | `LZSS_Decompress` |
| 8 | Pixel buffer is bottom-up WinG DIB (row 0 = screen bottom) | `FUN_004806a0` (WinGRecommendDIBFormat) |
| 9 | Speed field is at header offset +16 | `FLX_Load` |
| 10 | Speed=0 → 70ms (FLI jiffies default) | `FLX_Load` |
| 11 | playable = `min(indexed, frameCount)` — ring frame excluded | `FLX_TickFrame` (`param_1[0x1e]`) |
| 12 | Frame header is exactly 16 bytes | `FLX_TickFrame` (`+0x10` offset) |
| 13 | Chunk header is exactly 6 bytes (u32 size + i16 type) | `FLX_TickFrame` |
| 14 | `chunkSize` includes the 6-byte header | `FLX_TickFrame` |
| 15 | FLX2 (0xAF21) uses the same chunk formats as FLX (0xAF20) | `FLX_TickFrame` (shared dispatch) |
| 16 | Type-102 dirty-rect carries no pixel data | `DirtyRect_MarkRegionFromXYWH` |
| 17 | Audio cue pan: -64..+63 encoded as unsigned byte | `FLX_OnFrameAdvance` |
| 18 | Audio cue frame index is 1-based | `FLX_OnFrameAdvance` (`param_1[0x44]`) |
| 19 | Loop count -1 (0xFFFFFFFF) = infinite | `FLX_TickFrame` |
| 20 | PKX magic3 has two valid variants: 0x9BAEB0CF and 0x9BAEBACF | Observed in files |
| 21 | LZSS scratch buffer must be ≥ width×height bytes | `FLX_AllocGlobalScratchBuffers` |
| 22 | `0xF1FB` prefix frames contain no pixel chunks; skip entirely | `FLX_Load` frame walk |

---

## 13. Known Unknowns

| # | Unknown |
|---|---------|
| 1 | Exact internal format of `0xF1FB` prefix frame data (no decoder found) |
| 2 | Whether type-100 and type-4 ever appear in the same frame (not observed) |
| 3 | Maximum practical LZSS output size for type-104 chunks |
| 4 | Whether `oframe1` ever differs from 128 in practice |
| 5 | `flags` field (+14) bits beyond 0 and 1 — meaning of other bits unknown |
| 6 | PKX `compType` values beyond 0xC, 0xE, and two-pass variants |
| 7 | Behaviour when a frame's `chunkSize` is 0 |

---

*Specification derived entirely from static binary analysis. No original game source code was used or referenced.*
