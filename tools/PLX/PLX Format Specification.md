# .PLX File Format Specification

*Reverse-engineered from binary analysis of the same game engine documented
in the companion `.ANX` sprite-format specification. Field names/semantics
inferred from decompiled code and cross-checked two independent ways (see
§5) — verify against a real sample file's hex dump before relying on this
in production if you can.*

**Status: format-complete.** The container layer, payload layout, and the
engine's palette-slot usage convention are all traced end-to-end from real
decompiled code, with the key reserved-slot boundary (10–245) confirmed by
two unrelated functions independently, not just one code path's claim.

All multi-byte fields are **little-endian**. `.PLX` files hold a 256-color
palette used to display the 8bpp indexed pixel data found in this engine's
sprite and image formats (`.ANX`, `.BMX`) — those formats carry no palette
of their own; this is where the actual RGB values live.

---

## 1. Why `.PLX` Exists

Sprites and background images in this engine store 8-bit palette *indices*,
not RGB pixels. The same sprite can be redrawn under different palettes
depending on context, so the palette is a separate resource loaded per
scene rather than baked into each image. Filenames confirmed present in
the engine binary's string table:

```
mainmenu.plx   bioroom.plx   vatdoor.plx   vrloop.plx
medlab.plx     sbstart.plx   simgui.plx    BGMPALS.PLX
```

The naming pattern (`mainmenu`, `bioroom`, `vatdoor`, `vrloop`, `medlab`,
`sbstart`) strongly suggests one palette per game screen/location, loaded
when that screen becomes active.

> **A note on a false lead worth knowing about:** a resource named
> `WORLDCOL` was investigated early on as a possible palette candidate,
> since it loads a 1024-byte block — the right size for 256 RGBA entries.
> It was ruled out: that data is actually consumed as a color-remap /
> team-color lookup table for the sprite-blit pipeline, not as a display
> palette. If you encounter `WORLDCOL` or similarly-named resources while
> working with this engine, don't assume they're palettes without
> checking — `.PLX` is the verified, correct source.

---

## 2. Container Layer (optional)

Identical to the container format used by `.ANX`/`.BMX` in this engine —
if you already have a working `.ANX` reader, its container-handling code
can be reused unchanged for `.PLX`.

### 2.1 Container header (24 bytes / 0x18)

| Offset | Size | Type   | Field             | Notes                                   |
|-------:|-----:|--------|-------------------|--------------------------------------------|
| 0x00   | 4    | uint32 | `magic1`          | Must equal `0x3A584B50` (ASCII `"PKX:"`) |
| 0x04   | 4    | uint32 | `magic2`          | Must equal `0x10011966`                  |
| 0x08   | 4    | uint32 | `magic3`          | Must equal `0x9BAEBACF`                  |
| 0x0C   | 4    | uint32 | `codecId`         | 1–14; selects decompressor (see §2.2)    |
| 0x10   | 4    | uint32 | `compressedSize`  | Must equal `fileSize - 0x18`             |
| 0x14   | 4    | uint32 | `decompressedSize`| Must be in range `[1, 9999999]`          |
| 0x18   | —    | bytes  | `compressedData`  | `compressedSize` bytes, runs to EOF      |

**Detection:** if the first three dwords don't match all three magic
values, the entire file is the raw, uncontained payload — skip straight
to §3. Real `.PLX` files may or may not be compressed; both forms have
been observed conceptually (the container is universal in this engine,
not specific to any one resource type).

### 2.2 Codec dispatch

| codecId | Decompressor | Notes |
|--------:|---------------|-------|
| 1       | RLE (zero-run optimized) | Single-byte control scheme |
| 12 (0xC)| RLE (standard) | Control-byte RLE with an extended 16-bit form |
| 14 (0xE)| LZSS | Sliding-window LZ, 8-bit flag byte per 8 tokens |
| 2–11, 13| *unimplemented* | Not expected in valid files |

Full algorithm details for all three codecs (byte-exact, tested against
real files) are documented in the companion `.ANX` specification, §8/§10/§11
— they are identical here, since both formats share the same container
and codec-dispatch code in the engine binary.

**Decode result:** the decompressed bytes (or the raw file bytes, if no
container was present) are the `.PLX` payload described in §3.

---

## 3. `.PLX` Payload Layout

Unlike `.ANX`, there is **no header at all** — the payload is a flat,
tightly packed table of exactly 256 fixed-size entries:

```
+0x0000  PALETTEENTRY[256]     4 bytes each: {peRed, peGreen, peBlue, peFlags}
                                (native Win32 PALETTEENTRY byte order)
                                = 1024 bytes total, nothing else
```

| Offset (within entry) | Size | Field | Notes |
|---|---|---|---|
| +0 | 1 byte | `peRed` | 0–255 |
| +1 | 1 byte | `peGreen` | 0–255 |
| +2 | 1 byte | `peBlue` | 0–255 |
| +3 | 1 byte | `peFlags` | Win32 palette flags (`PC_RESERVED`, `PC_NOCOLLAPSE`, etc. — see §4.2). Not meaningful for pixel-color purposes; safe to ignore in a clean-room reader unless replicating exact GDI palette behavior. |

Entry `i` sits at payload offset `i * 4`. A valid `.PLX` payload must be
at least 1024 bytes (256 × 4); anything shorter is malformed.

---

## 4. Engine Usage — Which Slots Actually Matter

This is the part that isn't obvious from the file bytes alone and
requires tracing the loading code — **not all 256 entries are actually
used from the file.**

### 4.1 The reserved-slot boundary

Traced through two independent, unrelated code paths in the engine, both
agreeing exactly:

1. **`SetPaletteRange_ClampedToUsableSlots`** (the palette-apply function) —
   clamps any update range to slots **10 through 245** before applying it.
2. **`FindNearestPaletteIndex`** (a completely separate "closest color"
   lookup used for building semantic UI-color tables) — independently
   searches only slots **10 through 245** when matching a target RGB
   value against the currently-loaded palette.

Two unrelated functions landing on the exact same boundary is about as
strong a confirmation as static analysis alone can offer.

**What happens to the rest:** slots **0–9** and **246–255** (20 slots
total) get overwritten by `PaletteManager_ApplyEntries` with the result
of the Win32 `GetSystemPaletteEntries()` call — i.e. **the fixed Windows
reserved system palette**, not anything from the `.plx` file. This is
standard practice for 8-bit paletted Windows applications of this era:
those 20 slots are shared across every application on screen
simultaneously, so they can't be freely repurposed by any one app.

| Slot range | Source |
|---|---|
| 0–9 | Windows reserved system palette (`GetSystemPaletteEntries`) — **not** from the `.plx` file |
| 10–245 (236 entries) | The `.plx` file's own data — the real, usable image palette |
| 246–255 | Windows reserved system palette (`GetSystemPaletteEntries`) — **not** from the `.plx` file |

> **Practical consequence:** a `.plx` file's bytes at slots 0–9 and
> 246–255 are not meaningless — they're simply never read by this
> engine's palette-loading path. They might be zero, might be leftover
> data from whatever tool authored the file, or might coincidentally
> resemble real colors. Don't treat their presence or absence as a
> validity signal.

### 4.2 What happens to the loaded palette after merging

For completeness, the pipeline beyond the raw merge (useful if you're
trying to replicate exact on-screen appearance, not just extract colors):

1. **`PaletteManager_ApplyEntries`** merges the `.plx`'s 236 usable
   entries with the 20 system-reserved entries, and for each of the
   236 real entries, checks whether it happens to **duplicate** one of
   the reserved system colors — if so, it's flagged `PC_RESERVED`
   (shareable/collapsible); otherwise `PC_NOCOLLAPSE` (protect this
   exact color, don't let Windows collapse/dither it into a neighboring
   system color when the palette is realized). A `CreatePalette` /
   `SelectPalette` / `RealizePalette` sequence follows, for hardware
   palette rendering.
2. **`ConvertLogPaletteToRGBQuad`** separately reorders the same merged
   256 entries from `{R,G,B,flags}` into Win32 `RGBQUAD` order
   (`{B,G,R,reserved}`) for `SetDIBColorTable`, used by the engine's
   software/DIB rendering path.
3. **`RefreshActivePalette`** is a lightweight re-apply — just re-selects
   the already-built palette object into the device context (e.g. on
   window focus regain), without reloading or re-merging anything.

None of this changes what colors are *stored* in the file — it's how the
engine gets them onto the screen. A reader that only wants to extract
usable RGB values doesn't need to replicate any of §4.2, only §4.1's
slot-range rule.

### 4.3 Related engine machinery (context, not required for basic reading)

The engine also builds a small table of "nearest palette index for a
named/reference color" (`PostProcessLoadedPalette`, using
`FindNearestPaletteIndex` and a perceptually-weighted color-distance
metric, `ColorDistance_WeightedRGB` — weights G×6, R×3, B×1, matching
human luminance sensitivity ordering) for UI elements that need a
specific semantic color (e.g. "the highlight color") regardless of which
scene's palette happens to be active. There's also gradient-ramp
construction (`BuildPaletteGradientRamp`) for UI elements like health
bars that need a smooth-looking color transition despite only having
discrete palette entries to work with — since the renderer can't blend
RGB directly in 8-bit indexed mode, gradients are built by interpolating
in RGB space and snapping each step to the nearest real palette entry.
Neither of these affects the `.plx` file format itself; they're listed
here only because they were found while tracing the palette pipeline and
might be useful context if you're reverse-engineering the same engine
further.

---

## 5. Reference Decode Algorithm

```python
import struct

MAGIC1, MAGIC2, MAGIC3 = 0x3A584B50, 0x10011966, 0x9BAEBACF

def load_plx(file_bytes: bytes) -> list[dict]:
    """Returns a list of 256 dicts: {r, g, b, flags, reserved}.
    Only entries where reserved == False come from the .plx file itself;
    the other 20 are Windows-system-palette territory in the original
    engine and should not be trusted as real image colors."""

    # --- Layer 1: optional container (identical to .ANX, see the
    #     companion spec §1 for full codec algorithms) ---
    if len(file_bytes) >= 0x18 and (
        struct.unpack_from("<I", file_bytes, 0x00)[0] == MAGIC1 and
        struct.unpack_from("<I", file_bytes, 0x04)[0] == MAGIC2 and
        struct.unpack_from("<I", file_bytes, 0x08)[0] == MAGIC3
    ):
        codec_id          = struct.unpack_from("<I", file_bytes, 0x0C)[0]
        compressed_size   = struct.unpack_from("<I", file_bytes, 0x10)[0]
        decompressed_size = struct.unpack_from("<I", file_bytes, 0x14)[0]
        assert compressed_size == len(file_bytes) - 0x18
        assert 1 <= decompressed_size <= 9_999_999
        payload = decompress(codec_id, file_bytes[0x18:], decompressed_size)
    else:
        payload = file_bytes

    # --- Layer 2: flat PALETTEENTRY[256], no header ---
    assert len(payload) >= 1024, "payload too short for a 256-entry palette"

    entries = []
    for i in range(256):
        off = i * 4
        r, g, b, flags = payload[off], payload[off+1], payload[off+2], payload[off+3]
        entries.append({
            "r": r, "g": g, "b": b, "flags": flags,
            "reserved": (i < 10 or i > 245),
        })
    return entries
```

Loading and applying a `.plx` for rendering purposes:

```python
def usable_colors(entries: list[dict]) -> dict[int, tuple[int,int,int]]:
    """Slot index -> (r,g,b) for the 236 entries the engine actually uses."""
    return {i: (e["r"], e["g"], e["b"])
            for i, e in enumerate(entries) if not e["reserved"]}
```

---

## 6. Rendering `.ANX`/`.BMX` Pixels With a Loaded Palette

Given a decoded `.ANX`/`.BMX` frame's pixel index array (see the
companion spec) and a loaded `.plx`'s `entries` list:

```python
def index_to_rgb(index: int, entries: list[dict]) -> tuple[int,int,int,int] | None:
    """Returns (r,g,b,a) or None if this pixel should be treated as
    transparent / not display-worthy."""
    if index == 0:
        return (0, 0, 0, 0)  # .ANX/.BMX transparency convention -- always, regardless of palette
    e = entries[index]
    if e["reserved"]:
        # This shouldn't normally happen for real sprite data -- index 0
        # is the only slot sprites are expected to reference in the
        # reserved range. Treat as a red flag rather than guessing a color.
        return None
    return (e["r"], e["g"], e["b"], 255)
```

---

## 7. Verification Summary

- Container format: shared with `.ANX`, already round-trip tested against
  three real files in the companion spec.
- Payload layout (flat `PALETTEENTRY[256]`, no header): directly read
  from the decompiled loader (`LoadPaletteFile_PLX`) — the loader reads
  a fixed-size buffer and passes it straight to palette-application
  functions with no header-skipping logic, consistent with "no header."
- Reserved-slot boundary (10–245 usable): confirmed by **two independent
  functions** (§4.1) arriving at the identical range from unrelated code
  paths — the strongest form of corroboration available without a
  hex-dump of a real `.plx` file.
- **Not yet tested against a real `.plx` file's actual bytes** (unlike
  the companion `.ANX` spec, which was round-trip verified against three
  real files). If you have one, running it through §5's algorithm and
  checking the resulting colors visually would be the natural next step
  to raise this from "format-complete, code-verified" to
  "format-complete, data-verified."
