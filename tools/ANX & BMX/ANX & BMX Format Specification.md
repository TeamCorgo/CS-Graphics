# .ANX File Format Specification

*Reverse-engineered from binary analysis. Field names/semantics inferred from
decompiled code — verify against real sample files before relying on this in
production.*

**Status: format-complete and validated against real files.** Every byte
of the container layer, codec dispatch, and frame-record structure is
fully derived, verified against decompiled code, and — critically — has
been round-tripped against **three independent real `.ANX` files (six
frames total)** with byte-exact results at every layer, including a
4-frame file that confirms the frame-record array indexing across
multiple records, not just single-frame cases. That real-data testing
caught and fixed one genuine bug in the LZSS algorithm (§8.2/§9) that
pure code-reading had missed; no other discrepancies were found across
any of the three files. The display palette — `.ANX` stores 8bpp indexed
pixels with no embedded palette — has also since been traced to its
source: a separate `.PLX` resource family, fully documented in §15.

All multi-byte fields are **little-endian**. This format is used for
directional/animated unit sprites in a hex-grid strategy game engine.

---

## 1. Container Layer (optional)

`.ANX` files may be stored either as **raw payload** (see §2) or wrapped in a
generic compressed resource container shared by all resource types in this
engine (bitmaps, cursors, etc.). The loader detects the container by checking
for a fixed 3-dword magic at the start of the file.

### 1.1 Container header (24 bytes / 0x18)

| Offset | Size | Type   | Field             | Notes                                   |
|-------:|-----:|--------|-------------------|------------------------------------------|
| 0x00   | 4    | uint32 | `magic1`          | Must equal `0x3A584B50` (ASCII `"PKX:"`) |
| 0x04   | 4    | uint32 | `magic2`          | Must equal `0x10011966`                  |
| 0x08   | 4    | uint32 | `magic3`          | Must equal `0x9BAEBACF`                  |
| 0x0C   | 4    | uint32 | `codecId`         | 1–14; selects decompressor (see §1.2)    |
| 0x10   | 4    | uint32 | `compressedSize`  | Must equal `fileSize - 0x18`             |
| 0x14   | 4    | uint32 | `decompressedSize`| Must be in range `[1, 9999999]`          |
| 0x18   | —    | bytes  | `compressedData`  | `compressedSize` bytes, runs to EOF      |

**Detection logic:** if the first three dwords don't match all three magic
values, treat the entire file as raw, uncontained payload (jump straight to
§2). If they do match, validate `compressedSize`/`decompressedSize` against
the actual file size before proceeding — the reference loader treats a
mismatch as fatal.

### 1.2 Codec dispatch

`codecId` is a 4-bit nibble value (only the low nibble of the stored dword is
significant in the reference implementation, and a nonzero high nibble
triggers a *chained* second pass — see note below):

| codecId | Decompressor                          | Behavior | Full spec |
|--------:|----------------------------------------|----------|-----------|
| 1       | RLE (zero-run optimized)               | Single-byte control scheme: literal copy, run-fill, or end marker | §11 |
| 12 (0xC)| RLE (standard)                         | Control-byte RLE with an extended 16-bit form for long literal/fill/skip runs | §10 |
| 14 (0xE)| LZSS                                   | Sliding-window LZ: one 8-bit flag byte precedes every 8 tokens, each bit selecting literal byte vs. back-reference (offset/length packed into a 16-bit token) | §8 |
| 2–11, 13| *unimplemented*                        | Dispatcher returns 0 / no-op — do not expect these to occur in valid files | — |

> **Chained codecs:** the dispatcher is recursive — if `codecId & 0xF0 != 0`,
> it first recurses on `(codecId >> 4) & 0xF` before applying the low-nibble
> codec. **Verified:** this path's scratch buffer is never initialized by
> any other code in the binary and is not exercised by any traced call —
> see §9 for the full verification. Treat `codecId` as a flat value in
> `{1, 12, 14}`.

**Output:** decompressing yields exactly `decompressedSize` bytes. This
buffer is the raw `.ANX` payload described in §2. If the container magic was
absent, the *entire original file* is the payload instead — decompression is
skipped entirely.

---

## 2. ANX Payload Layout

```
+0x00  uint32          frameCount        (no validation found on this field itself —
                                           see §4 errata)
+0x04  FrameRecord[frameCount]           (12 bytes each, tightly packed)
       ...
       [frame pixel/compressed data, referenced by each record's dataOffset]
```

### 2.1 Frame record (12 bytes)

| Offset (from record start) | Size | Type   | Field        | Notes |
|----------------------------:|-----:|--------|--------------|-------|
| 0x00 | 4 | int32  | `dataOffset` | Signed byte offset **from the start of this record** to the frame's pixel data (raw or compressed, per `codec`) |
| 0x04 | 2 | uint16 | `width`      | Frame width in pixels |
| 0x06 | 2 | uint16 | `height`     | Frame height in pixels |
| 0x08 | 1 | uint8  | *reserved*   | See verification note below |
| 0x09 | 1 | uint8  | `codec`      | `0` = pixel data is raw/uncompressed; nonzero = per-frame compressed, decode via the same codec dispatcher as §1.2 |
| 0x0A | 2 | —      | *reserved*   | See verification note below |

> **Verified — reserved bytes are confirmed dead in this build.** Not only
> does no traced code path read `+0x08`/`+0x0A` individually, but the *entire
> containing dword* (`+0x08` through `+0x0B`) is, in one place
> (`BlitAnimationFrame`), copied wholesale into a cache struct — and that
> cached copy is itself never read back by any other code (checked via
> xref: exactly one reference, the write itself). This is about as strong
> a confirmation of "unused" as static analysis can give without a real
> sample file's hex dump. Treat as padding.
>
> **Real-data note:** in the one real frame record decoded so far
> (`HRC023B0.ANX`, §8.5), `+0x08 = 0x00` but `+0x0A/+0x0B` was **`0x3832`
> — nonzero.** This is consistent with "unused padding containing leftover
> author-tool bytes" (garbage-in-padding is completely normal and doesn't
> imply the bytes are read), but it does mean these bytes are **not
> reliably zero** — don't use "is this field zero" as a validity check
> anywhere in a decoder.
>
> **Format is shared across resource types.** The identical 12-byte
> record layout and `base + 4 + index*12` addressing scheme is used for
> both `.ANX` (unit sprites) and `.BMX` (effect sprites) in the traced
> code — this is a general per-resource-frame-array convention in the
> engine, not something specific to `.ANX` that might not generalize.

**Frame selection (engine behavior, not part of the file format itself):**
the game engine indexes into this array using `frameIndex = facing % 6` (a
6-directional hex-grid facing scheme) combined with unit-type-specific
animation-state offsets, then clamps the final index against `frameCount`.
**This selection logic is scoped to the engine, not the `.ANX` format** — as
far as the file format is concerned, its only contract is "an array of
`frameCount` 12-byte records, indexable 0 to `frameCount-1`, each pointing
to a decodable frame." A generic `.ANX` reader doesn't need to replicate
the engine's directional-facing heuristics at all — it only needs to be
able to decode any given record by index. Record `i` sits at payload
offset `0x04 + i * 12`.

### 2.2 Frame pixel data

- **Format:** 8 bits per pixel, palette-indexed (no embedded palette in this
  file — palette is supplied externally by the engine, via a separate
  `.PLX` resource; see §15 for the fully-traced format).
- **Transparency:** pixel value `0x00` is a hard color-key — it is never
  plotted and is treated as fully transparent by every blit path observed.
- **Location:** `recordStart + dataOffset`.
- **Expected size:** `width * height` bytes once decoded — but see the
  caveat below.
- **Decoding:** if `codec == 0`, the bytes at that location are the raw
  pixels — read directly, no decode step. If `codec != 0`, treat the bytes
  at that location as a stream for the shared codec dispatcher (§1.2),
  using `codec` as the codec ID.

> **Verified caveat — do not assume `width*height` bounds the decode loop.**
> In the traced binary, per-frame decompression writes into a **shared,
> statically-allocated global scratch buffer**, not a buffer freshly sized
> to `width*height`. All references to this buffer's size/pointer fields
> (`DAT_004a4098`/`DAT_004a409c`) are *reads* — nothing in the traced code
> ever writes them, meaning they're baked-in static data whose actual value
> this analysis could not extract (no raw memory/data-byte-read tool was
> available). Separately, and more importantly: **nothing in the traced
> code asserts that the decoded length equals `width*height`** — each codec
> self-terminates independently (RLE via its `0x00`/`0x80` end markers,
> LZSS via its `blockCount`/`segmentFlag` header), and the only check
> afterward is that decoded length doesn't *exceed* the destination
> buffer's declared capacity.
>
> **Recommendation for a clean-room decoder:** allocate `width * height`
> bytes as the destination, run the appropriate codec's decoder until *it*
> signals completion (its own terminator/count, not a fixed byte budget),
> and treat `width*height` as a **sanity check** on the result (warn/fail if
> mismatched) rather than as the authoritative stopping condition fed into
> the decode loop itself. This matches the spirit of the original code
> without inheriting its fixed-scratch-buffer implementation detail.

---

## 3. Reference Decode Algorithm

```python
def load_anx(file_bytes: bytes) -> list[Frame]:
    # --- Layer 1: optional container ---
    if len(file_bytes) >= 0x18 and (
        u32(file_bytes, 0x00) == 0x3A584B50 and
        u32(file_bytes, 0x04) == 0x10011966 and
        u32(file_bytes, 0x08) == 0x9BAEBACF
    ):
        codec_id          = u32(file_bytes, 0x0C)
        compressed_size   = u32(file_bytes, 0x10)
        decompressed_size = u32(file_bytes, 0x14)
        assert compressed_size == len(file_bytes) - 0x18
        assert 1 <= decompressed_size <= 9_999_999
        payload = decompress(codec_id, file_bytes[0x18:], decompressed_size)
    else:
        payload = file_bytes

    # --- Layer 2: ANX structure ---
    frame_count = u32(payload, 0x00)
    # No validated bound found on frame_count itself in the traced binary —
    # apply your own sane sanity limit (e.g. reject absurdly large values)
    # rather than relying on the format to guarantee one.

    frames = []
    for i in range(frame_count):
        rec_off = 0x04 + i * 12
        data_offset = i32(payload, rec_off + 0x00)
        width       = u16(payload, rec_off + 0x04)
        height      = u16(payload, rec_off + 0x06)
        codec       = payload[rec_off + 0x09]

        pixel_start = rec_off + data_offset
        expected_size = width * height
        if codec == 0:
            pixels = payload[pixel_start : pixel_start + expected_size]
        else:
            # Run the codec to its own natural termination (see §5/§7/§8),
            # NOT a fixed read of `expected_size` bytes — the codecs
            # self-terminate independently. Use expected_size only to
            # sanity-check the result afterward.
            pixels = decompress(codec, payload[pixel_start:])
            assert len(pixels) == expected_size, "sanity check — see §2.2 caveat"

        frames.append(Frame(width, height, pixels))  # pixel 0x00 == transparent

    return frames
```

---

## 4. Errata (corrections from earlier analysis passes)

- **LZSS segment-continuation bit budget — corrected via real-data testing.**
  An earlier pass always processed a full 8 bits per flag byte, including on
  the one-shot `segmentFlag`-driven continuation group. Testing against a
  real `HRC023B0.ANX` file (§8.5) exposed this: the decoder overshot by 9
  output bytes and 6 input bytes on that exact file. Re-examining the
  decompiled code showed the bit-counter variable is read from
  `segmentFlag`'s value *before* the exhaustion check, and is only reset to
  8 in the normal (non-continuation) branch — so the continuation group
  processes only `segmentFlag` bits, not 8. Fixed in §8.2/§8.4; the
  corrected decoder now matches the real file exactly.
- **`frameCount` bound — corrected.** An earlier pass of this spec claimed
  `frameCount` was validated to `0 < frameCount <= 0xE`. That was a
  misattribution: the `1..14` range check in the traced code applies to the
  container's **`codecId`** field (§1.1), not to `frameCount`. Re-verified
  directly against the decompiled `LoadResourceFile_WithDecompression` —
  **no code path was found that validates `frameCount` itself.**
- **Per-frame decompression sizing — corrected.** An earlier pass implied
  the original code decodes each compressed frame into a buffer sized
  exactly to `width*height`. Verified this is not accurate: the original
  uses a fixed/shared global scratch buffer (see §2.2 caveat) and never
  explicitly asserts the decoded length equals `width*height`. This does
  not change the recommended decoder design (§2.2), but the earlier
  phrasing overstated what the original binary actually guarantees.

## 5. Resource Resolution Fallback

Verified in `GetUnitAnimationFramePointer`: if the primary named resource
(e.g. `HRC_042_A_3.ANX`, built via `RTL_FormatString`) fails to load — file
not found in any search path or archive — the loader does **not** treat
this as fatal on the first attempt (it's called with the "fatal on missing"
flag cleared). Instead, it falls back to loading a **fixed placeholder
resource**, hardcoded as `HRC023B0.ANX`, this time with the fatal flag set.
Only if *that* also fails does the code raise a fatal error. Implementers
building a lenient/compatible reader should replicate this: missing
resources are expected to have a fallback, not to always be a hard error.

## 6. Archive Container Format (verified)

For resources loaded from a bundled archive rather than loose files on
disk (`OpenResourceFile_FromArchive`), the archive's directory table was
fully derived from the decompiled binary-search logic:

**Directory entry (16 bytes each, sorted by name for binary search):**

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0x00 | 12 | `name` | Uppercased, space/null-padded filename, compared byte-for-byte (`RTL_MemCmp`) against the uppercased lookup name |
| 0x0C | 4  | `dataOffset` | Absolute offset into the archive file where this entry's data begins |

**At `dataOffset` in the archive file itself:**

| Offset (from `dataOffset`) | Size | Field |
|---|---|---|
| +0x00 | 4 | `entrySize` — byte length of the entry's data payload |
| +0x04 | — | The actual resource bytes (e.g. a `.ANX` file, container-wrapped or raw, per §1) |

So: binary-search the sorted directory table for a case-insensitive name
match, read the 4-byte size prefix at the matched `dataOffset`, then treat
the following `entrySize` bytes exactly as if they were a standalone loose
`.ANX` file (container detection and everything else in this spec applies
unchanged).

## 7. Open Questions / Unverified

- **Reserved frame-record bytes (`+0x08`, `+0x0A`)** — as strong as static
  analysis alone can confirm: no traced code reads them individually, and
  the one place that copies the containing dword wholesale never reads that
  copy back either. Not literally *impossible* that some other binary (an
  editor/tool) reads them, but nothing in this game binary does.
  **Format-complete as padding** unless a real file's hex dump proves
  otherwise.
- Codec IDs 2–11 and 13 are unimplemented in the traced binary. If real files
  ever use them, this spec is incomplete for those cases.
- The container magic bytes are validated against a fixed engine build; if
  you have files with a different magic, they may come from a different
  engine version.
- Codec chaining (§9) — resolved: verified dead/unreachable in the traced
  binary.
- The exact byte value of the per-frame scratch buffer's fixed capacity
  (`DAT_004a4098`) could not be extracted — this analysis had no tool
  capable of reading raw static data bytes, only decompiling code and
  listing labeled data items. This is a genuinely unknown constant, not a
  guessed one; it does not block a clean-room decoder (see §2.2).
- `LoadResourceFile_WithDecompression` has a caller-supplied-destination-
  buffer code path (used when `param_4 != NULL`) that is never exercised
  by any `.ANX`-loading call site traced in this analysis (all pass `NULL`,
  requesting a freshly allocated buffer). Documented for completeness; not
  relevant to `.ANX` decoding specifically.
- All three codecs — LZSS (§8) and both RLE variants (§10, §11) — are
  verified two ways: (1) against the decompiled control flow, including
  exhaustive boundary-value checks for every range split, and hand-built
  synthetic streams exercising every branch; and (2) against a real
  `HRC023B0.ANX` file (§8.5), decoded byte-exact end-to-end including a
  real bug the synthetic tests alone hadn't caught (the LZSS bit-budget
  issue — see errata). This is the highest confidence level this analysis
  can offer for the three implemented codecs.
- ~~The display palette that turns decoded 8bpp indices into RGB colors
  is not part of the `.ANX` file~~ — **resolved, see §15.** It's supplied
  by separate `.PLX` resource files (one per scene/context), traced end
  to end: container format, payload layout, and exactly how the engine
  merges it with the Windows reserved system palette.

---

## 8. Appendix A: LZSS Codec (codecId 14 / 0x0E) — Full Detail

The LZSS decompressor implements a standard "distance/length token +
literal" scheme with two implementation-specific optimizations. All
multi-byte fields are little-endian.

### 8.1 Stream header

The first byte (`headerByte`) is overloaded — its value determines how the
rest of the header is laid out:

**`headerByte < 8`** (explicit-length form):

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0x00 | 1 | `headerByte` | value 0–7, becomes `segmentFlag` |
| 0x01 | 2 | `blockCount16` | little-endian u16 |
| 0x03 | — | stream data | *unless escaped, see below* |

If `blockCount16 == 0xFFFF` (escape value), it's discarded and replaced by a
full 32-bit count instead:

| Offset | Size | Field |
|---|---|---|
| 0x03 | 4 | `blockCount32` |
| 0x07 | — | stream data |

**`headerByte >= 8`** (packed form — length embedded in the header byte):

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0x00 | 1 | `headerByte` | `segmentFlag = headerByte & 0x07`, `blockCount = (headerByte >> 3) - 1` |
| 0x01 | — | stream data | |

`blockCount` is the number of **token-groups** (see §8.2) in the stream.
`segmentFlag` (0–7) is consumed once — after `blockCount` groups are
processed, if `segmentFlag != 0` the decoder resets it to 0 and continues
with one more group instead of stopping. In practice this appendix's traced
binary never seems to exercise more than a single segment, but a compliant
decoder should implement the reset in case some `.ANX` files use it.

### 8.2 Token groups

Repeat until `blockCount` groups (plus any `segmentFlag` continuation) are
consumed:

1. Read the next byte as a **flag byte**.
2. **If the flag byte is `0x00`:** raw fast-path — copy the next **8 bytes**
   directly from input to output, unconditionally. No bit testing, no token
   parsing. Advance input by 9 bytes total (flag + 8 data bytes), output by 8.
   This group then counts as one "block" against `blockCount`.
3. **Otherwise:** treat the flag byte as control bits, read **MSB-first**
   (test bit 7, then 6, ... ):
   - **On a normal group** (i.e. `blockCount` has not yet been exhausted):
     process **all 8 bits** of the flag byte.
   - **On the segment-continuation group** (i.e. `blockCount` just went
     negative for the first time, and `segmentFlag != 0`): process **only
     `segmentFlag` bits** of the flag byte (starting from bit 7), **not a
     full 8**. This is a corrected, real-data-verified detail — an earlier
     draft of this spec always processed 8 bits and produced too much
     output on files that use this path (see §8.5 for the concrete example
     that caught it).
   - For each bit processed:
     - **bit = 1 → back-reference token:** read the next 2 bytes as a
       little-endian u16 `token`.
       - `length = (token & 0x0F) + 3` (range 3–18)
       - `distance = (token >> 4) + 1` (range 1–4096)
       - Copy `length` bytes, one at a time, from `output[-distance]` to the
         current output position (byte-by-byte copy so overlapping
         runs — where `length > distance` — replicate correctly, same trick
         as classic LZ77/LZSS).
     - **bit = 0 → literal:** copy 1 byte directly from input to output.

### 8.3 Parameters summary

| Parameter | Value |
|---|---|
| Window size | 4096 bytes (12-bit distance field) |
| Min match length | 3 bytes |
| Max match length | 18 bytes (4-bit length field, `+3` bias) |
| Token size | 16 bits: bits 15–4 = `distance-1`, bits 3–0 = `length-3` |
| Flag byte bit order | MSB first; `1` = match, `0` = literal |
| Fast path | flag byte `0x00` → unconditional raw 8-byte copy |
| Back-reference addressing | Direct linear offset into the output buffer (no ring/circular buffer) |

### 8.4 Reference pseudocode

```python
def lzss_decompress(src: bytes, expected_out_size: int) -> bytes:
    out = bytearray()
    pos = 0

    header = src[pos]
    if header < 8:
        segment_flag = header
        block_count = int.from_bytes(src[pos+1:pos+3], "little")
        pos += 3
        if block_count == 0xFFFF:
            block_count = int.from_bytes(src[pos:pos+4], "little")
            pos += 4
    else:
        segment_flag = header & 0x07
        block_count = (header >> 3) - 1
        pos += 1

    while True:
        # bit_budget is read from segment_flag's CURRENT value *before* the
        # exhaustion check below -- this matters: on the one-shot
        # continuation group it is NOT reset to 8 (verified against real
        # data -- see §8.5).
        bit_budget = segment_flag
        block_count -= 1
        if block_count < 0:
            if segment_flag == 0:
                break                       # done
            segment_flag = 0                # one-shot continuation, consumed
            flag = src[pos]; pos += 1
            # bit_budget keeps its pre-exhaustion value (NOT reset to 8)
        else:
            flag = src[pos]; pos += 1
            if flag == 0x00:
                out += src[pos:pos+8]
                pos += 8
                continue
            bit_budget = 8                  # normal group: full byte

        bits_done = 0
        while bits_done < bit_budget:
            bit = (flag >> (7 - bits_done)) & 1
            bits_done += 1
            if bit:
                token = int.from_bytes(src[pos:pos+2], "little")
                pos += 2
                length   = (token & 0x0F) + 3
                distance = (token >> 4) + 1
                start = len(out) - distance
                for i in range(length):
                    out.append(out[start + i])
            else:
                out.append(src[pos]); pos += 1

    assert len(out) == expected_out_size
    return bytes(out)
```

### 8.5 Worked example — validated against a real file

This is no longer a hypothetical — a real `HRC023B0.ANX` file (the fallback
placeholder resource referenced in §5, 2199 bytes) was decoded end-to-end
using exactly the algorithm above, and the results are byte-exact:

```
codec       = 14 (0x0E)   LZSS
compressed  = 2175 bytes  (0x087F)
output      = 2872 bytes  (0x0B38)
```

Container header (offsets per §1.1) matched exactly: both magic checks
passed, `codecId = 14`, `compressedSize = 2175 = fileSize - 0x18`,
`decompressedSize = 2872`.

LZSS stream header: `headerByte = 0x05` (< 8, explicit-length form) →
`segmentFlag = 5`, `blockCount = 194` (u16 at offset+1), stream data starts
at offset+3.

Running the decoder in §8.4: **consumed exactly 2175 input bytes, produced
exactly 2872 output bytes** — no truncation, no overrun. This is what
caught the bit-budget bug described in §8.2: an earlier version of this
decoder that always processed 8 bits per flag byte (even on the
`segmentFlag`-driven continuation group) overshot by 9 output bytes and 6
input bytes on this exact file. The fix (§8.2/§8.4) resolved it exactly.

The resulting 2872-byte payload parsed as valid `.ANX` structure: `frameCount
= 1`, one 12-byte frame record (`dataOffset = 12`, `width = 97`,
`height = 102`, `codec = 1` — RLE zero-run-optimized). Decoding that frame's
pixel stream with §11's algorithm produced **exactly `97 × 102 = 9894`
bytes**, consuming precisely the rest of the payload with zero bytes left
over. Rendered (with a synthetic test palette, since this render predates
the `.PLX` palette work — see §15 for the real one) it's a clean,
structured flower/starburst-shaped icon with a crisp outline — not
noise — confirming correct decoding at every layer: container → LZSS →
frame record → per-frame RLE → pixels.

---

## 9. Appendix A2: Codec Dispatch — Chaining Behavior (Verified)

`DecompressResourceData_Dispatch(codecId, src, dst)` is implemented as:

```c
if ((codecId & 0xF0) != 0) {
    // "chain": decompress with the HIGH nibble first, into a private
    // scratch struct (DAT_004a40a0 = size, DAT_004a40a4 = buffer ptr),
    // then treat that scratch buffer as the new source.
    DecompressResourceData_Dispatch((codecId >> 4) & 0xF, src, &scratch);
    src = scratch.bufferPtr;
}
lowNibble = codecId & 0xF;
// lowNibble 1  -> RLE_Decompress_ZeroRunOptimized
// lowNibble 12 -> RLE_Decompress
// lowNibble 14 -> LZSS_Decompress
// anything else (2-11, 13) -> returns 0 (unimplemented)
```

**Verified fact:** the scratch struct (`DAT_004a40a0`/`DAT_004a40a4`) used for
the chained/high-nibble call is **never written or read anywhere else in the
binary** — its only references are inside this function's own recursive
call. It is never pre-sized or pre-allocated by any caller. Combined with the
fact that:

- the container-level `codecId` (§1.1) is explicitly validated to the range
  **1–14 inclusive** before this dispatcher is ever called, and
- the per-frame `codec` byte (§2.1) is a plain `uint8` with no observed
  caller restricting it below 16 either, but no sample data or code path in
  this binary was found that produces a value ≥ 16,

**the chaining path is not exercised by any traced code path** and its
scratch buffer is effectively uninitialized on first use. A conformant
decoder should:
- Treat `codecId`/`codec` as a **flat single value in {1, 12, 14}** (the only
  three implemented codecs).
- Treat any other value (0, 2–11, 13, or ≥16) as **invalid/unsupported** and
  fail rather than attempt to emulate the chaining behavior — the reference
  implementation's own chaining path is unverified and should not be
  considered part of a "perfect" reproduction until confirmed against a real
  file that actually uses it.

---

## 10. Appendix C: RLE Codec — codecId 12 (`RLE_Decompress`) — Full Detail

This is a control-byte-driven RLE scheme, distinct from and more elaborate
than codecId 1 (Appendix D). It self-terminates via an explicit end marker
— no external length/count is required by the algorithm itself (though the
container's `decompressedSize` should still be used to validate the result).

### 10.1 Control byte semantics

Read one control byte `C` (0–255) at a time:

| `C` value | Meaning | Bytes consumed (beyond `C`) |
|---|---|---|
| `0x01`–`0x7F` | **Literal copy:** copy `C` bytes directly from input to output | `C` |
| `0x00` | **Short run-fill:** read 1 count byte `K` (0–255), then 1 fill byte `F`; write `F` repeated `K` times | 2 |
| `0x81`–`0xFF` | **Short skip:** advance the output pointer by `C & 0x7F` (1–127) bytes **without writing** (leaves existing buffer contents — caller must pre-fill/zero the output buffer before decoding if this matters) | 0 |
| `0x80` | **Extended form** — read the next 2 bytes as a little-endian `int16` (`raw16`); see §10.2 | 2 (+ more, see below) |

### 10.2 Extended form (`C == 0x80`)

Read `raw16` as a little-endian 16-bit value immediately following the
control byte:

| `raw16` range | Meaning |
|---|---|
| `0x0000` | **End of stream.** Stop decoding; return total bytes written. |
| `0x0001`–`0x7FFF` | **Extended skip:** advance output pointer by `raw16` bytes without writing. |
| `0x8000`–`0xBFFF` | **Extended literal copy:** count `= raw16 - 0x8000` (0–16383); copy that many bytes directly from input (immediately following the 2 length bytes) to output. |
| `0xC000`–`0xFFFF` | **Extended run-fill:** count `= raw16 - 0xC000` (0–16383); read 1 fill byte immediately after the 2 length bytes, write it `count` times. |

(Verified exhaustively against the decompiled bit operations — see the
derivation check: boundary values `0x0000, 0x0001, 0x7FFF, 0x8000, 0xBFFF,
0xC000, 0xFFFF` all map unambiguously to the ranges above with no gaps or
overlaps.)

### 10.3 Reference pseudocode

```python
import struct

def rle_decompress_codec12(src: bytes) -> bytes:
    out = bytearray()
    pos = 0
    while True:
        c = src[pos]; pos += 1

        if 0x01 <= c <= 0x7F:
            out += src[pos:pos+c]
            pos += c

        elif c == 0x00:
            k = src[pos]; f = src[pos+1]; pos += 2
            out += bytes([f]) * k

        elif c == 0x80:
            raw16 = struct.unpack_from("<H", src, pos)[0]; pos += 2
            if raw16 == 0x0000:
                break                                    # end of stream
            elif raw16 <= 0x7FFF:
                out += bytes(raw16)                       # skip (write nothing;
                                                            # caller must pre-init
                                                            # the buffer if content
                                                            # there matters)
            elif raw16 <= 0xBFFF:
                n = raw16 - 0x8000
                out += src[pos:pos+n]
                pos += n
            else:  # 0xC000-0xFFFF
                n = raw16 - 0xC000
                f = src[pos]; pos += 1
                out += bytes([f]) * n

        else:  # 0x81-0xFF
            skip = c & 0x7F
            out += bytes(skip)                             # skip, write nothing

    return bytes(out)
```

> **Important:** the "skip" operations do not write any bytes — they leave
> whatever was already in the output buffer at those positions untouched.
> The Python reference above appends zero bytes as a stand-in for clarity,
> but a byte-perfect reimplementation must pre-allocate/pre-fill the output
> buffer exactly as the target application does (the traced binary always
> `GlobalAlloc`s fresh memory for the destination, which on Windows is
> zero-initialized by `GlobalAlloc` — so in practice "skip" reliably produces
> zero bytes in this codebase, but this is a property of the *allocator*,
> not the codec itself).

---

## 11. Appendix D: RLE Codec — codecId 1 (`RLE_Decompress_ZeroRunOptimized`) — Full Detail

A simpler, single-byte-control RLE scheme (no extended/16-bit form at all).
The "zero-run-optimized" name reflects an internal fast path for filling
long runs of `0x00`, which is a pure performance optimization — the output
is byte-for-byte identical whether or not that fast path triggers.

### 11.1 Control byte semantics

Read one control byte `C` (0–255):

| `C` value | Meaning | Bytes consumed (beyond `C`) |
|---|---|---|
| `0x00` or `0x80` | **End of stream.** Stop decoding; return total bytes written. (Both values terminate identically — there is no extended form in this codec.) | 0 |
| `0x01`–`0x7F` | **Literal copy:** copy `C` bytes directly from input to output. | `C` |
| `0x81`–`0xFF` | **Run-fill:** count `= C & 0x7F` (1–127); read 1 fill byte, write it `count` times. | 1 |

That's the entire control scheme — no 2-byte extended lengths, no separate
skip-without-writing behavior (every op here always writes bytes, unlike
codecId 12's skip operations).

### 11.2 Reference pseudocode

```python
def rle_decompress_codec1(src: bytes) -> bytes:
    out = bytearray()
    pos = 0
    while True:
        c = src[pos]; pos += 1

        if (c & 0x7F) == 0:            # c == 0x00 or c == 0x80
            break                      # end of stream

        if c & 0x80:                   # 0x81-0xFF: run-fill
            count = c & 0x7F
            f = src[pos]; pos += 1
            out += bytes([f]) * count
        else:                          # 0x01-0x7F: literal copy
            out += src[pos:pos+c]
            pos += c

    return bytes(out)
```

(The dword-aligned fast paths in the original for `count >= 20` literal
copies, and `fillByte == 0 and count > 11` run-fills, are pure performance
optimizations reconstructing the same bytes — safely omitted in a
from-scratch reimplementation.)

**Validated against real data — three independent files:** this exact
algorithm has decoded every frame in three real `.ANX` files with zero
byte-level discrepancies:

| File | Container | Frames | Result |
|---|---|---|---|
| `HRC023B0.ANX` (fallback placeholder) | LZSS, 2175→2872 bytes | 1 frame, 97×102 | Exact — 9894/9894 bytes, 0 leftover |
| Unit/creature sprite | LZSS, 2757→3299 bytes | 1 frame, 104×72 | Exact — 7488/7488 bytes, 0 leftover |
| 4-frame portrait sequence | LZSS, 9962→13242 bytes | **4 frames**, 85×85 each | Exact on all 4 — 7225/7225 bytes each, correct offsets (52, 3421, 6837, 10206) |

The third file is the only one so far to exercise multiple records in the
frame array, and every offset landed exactly right — confirming the
`0x04 + i*12` indexing (§2.1) holds across a real multi-frame file, not
just the single-frame cases. Across all three, the only discrepancy found
was the LZSS bit-budget bug (§4 errata) — every other part of the spec
held up on first try.

---

## 12. Appendix B: Supporting Function Reference

For implementers cross-referencing against the original binary, here is the
full renamed call tree involved in reading and decoding an `.ANX` file
(rendering-only functions omitted — see the main writeup for those):

| Function | Role |
|---|---|
| `FindResourceFile_InSearchPaths` | Walks configured search-path entries (loose directories or bundled archives), trying each until the named file is found |
| `FileHandle_Open` | `CreateFileA` wrapper — opens a loose file on disk |
| `OpenResourceFile_FromArchive` | Binary-searches a sorted directory table inside a bundled archive file, then opens a *virtual window* (offset+size) onto the matching entry |
| `FileHandle_SetVirtualWindow` | Restricts a file handle's effective offset/size range, so archive-embedded sub-files behave like standalone files to the reader |
| `FileHandle_Seek` | `SetFilePointer` wrapper, window-aware (adjusts for virtual-window base offset) |
| `FileHandle_GetSize` | Computes size via seek-to-end/seek-back |
| `FileHandle_Read` | `ReadFile` wrapper |
| `FileHandle_Close` | `CloseHandle` wrapper |
| `GlobalAlloc_Checked` | `GlobalAlloc` wrapper with fatal-error check |
| `GlobalFree_Wrapper` | `GlobalFree` wrapper |
| `LoadResourceFile_WithDecompression` | Top-level entry: opens, reads, detects the `PKX:` container, and dispatches to decompression if present |
| `DecompressResourceData_Dispatch` | Reads the codec ID nibble(s) and routes to the matching decompressor (recurses for chained codecs) |
| `LZSS_Decompress` | codecId 14 — see Appendix A |
| `RLE_Decompress` | codecId 12 — control-byte run-length decoder |
| `RLE_Decompress_ZeroRunOptimized` | codecId 1 — RLE variant with a fast path for long zero-byte runs |
| `RTL_StrCpy` / `RTL_StrCat` / `RTL_StrLen` / `RTL_StrUpper` / `RTL_MemCmp` / `RTL_MemSet` | Runtime string/memory helpers used for path building and archive filename lookup |
| `RTL_FormatString` (+ `RTL_FormatString_PutCharCallback`) | sprintf-style formatter, used to build resource filenames like `HRC_%03d_%c_%d.ANX` |
| `ShowFatalErrorDialog` | `MessageBoxA`-based fatal error popup, invoked on any validation failure (bad magic, size mismatch, alloc failure, etc.) |
| `LoadPaletteFile_PLX` | Loads a `.PLX` palette resource (via the same container pipeline as `.ANX`) and applies it — see §15 |
| `SetPaletteRange_ClampedToUsableSlots` | Clamps a palette update range to slots 10–245, stages the entries for `PaletteManager_ApplyEntries` |
| `PaletteManager_ApplyEntries` | Merges staged entries with the Windows reserved system palette (slots 0–9, 246–255), builds/realizes the GDI palette |
| `RefreshActivePalette` | Re-selects the already-built palette into the device context without reloading |
| `ConvertLogPaletteToRGBQuad` | Converts the merged 256-entry palette from `PALETTEENTRY` to `RGBQUAD` order for `SetDIBColorTable` |

---

## 13. Quick Reference: Byte Layout Diagram

```
File (no container):
┌─────────────────────────────────────────────┐
│ frameCount (u32)                             │  0x00
├─────────────────────────────────────────────┤
│ FrameRecord[0]  (12 bytes)                   │  0x04
│ FrameRecord[1]  (12 bytes)                   │  0x10
│ ...                                          │
│ FrameRecord[n-1]                             │
├─────────────────────────────────────────────┤
│ pixel data (referenced by each record's      │
│ dataOffset, raw or per-frame compressed)     │
└─────────────────────────────────────────────┘

FrameRecord (12 bytes):
┌────────────┬────────┬────────┬──────┬───────┬──────┐
│ dataOffset │ width  │ height │ rsvd │ codec │ rsvd │
│  (i32)     │ (u16)  │ (u16)  │ (u8) │ (u8)  │ (u16)│
│  4B        │  2B    │  2B    │ 1B   │  1B   │  2B  │
└────────────┴────────┴────────┴──────┴───────┴──────┘
  0x00         0x04     0x06     0x08   0x09    0x0A
```

---

## 14. `.ANX` vs `.BMX`: Same File Format, Different Consumption

Both extensions share **byte-identical** structure — same `PKX:` container
(§1), same codec dispatch, same payload layout (`frameCount` + array of
12-byte records, §2), same `base + 4 + index*12` addressing. There is no
format-level distinction. Confirmed by reading both branches of
`GetUnitAnimationFramePointer` in full, rather than assuming from the
addressing formula alone (an earlier pass in this analysis had only
verified the addressing matched, not the full surrounding logic).

The divergence is entirely on the **engine's consuming side**:

| | `.ANX` (unit type `1`) | `.BMX` (unit type `4`) |
|---|---|---|
| Filename pattern | `HRC_%03d_%c_%d.ANX` — 3 parameters (unit-type number, a char, a sub-variant number) | `FX_%03d.BMX` — 1 parameter (a flat effect-type ID) |
| Resource cache shape | 3D sparse array: `[unitType][animState][facing]` (168-byte stride per unit type) — many cached resources per unit, one per state×facing combo | Flat 1D array indexed only by effect-type byte — exactly one cached resource per effect ID |
| Frame index selection | Elaborate: animation-state lookup tables, a 4-way `switch` on state codes, hex-facing `% 6`, override flag bits | Trivial: the raw index byte from the caller's struct is used directly, no state machine |
| Bounds check on final index | Explicit — clamps to `0` if `index >= frameCount` | **None** — the index is trusted completely, no comparison against the record array's own `frameCount` |
| Fallback on load failure | Falls back to a placeholder resource (`HRC023B0.ANX`) before giving up | No fallback — straight to `ShowFatalErrorDialog` |

**Practical read:** `.ANX` is for directional, stateful **unit** sprites —
needs per-facing, per-animation-state variants, cached generously,
validated defensively. `.BMX` is for **effects** (explosions, muzzle
flashes, etc.) — one resource per effect type, no facing dimension
(effects apparently aren't drawn direction-dependently), and the engine
trusts its own frame-index bookkeeping enough to skip the bounds check
entirely.

> **Worth flagging as a real engine characteristic, not a documentation
> gap:** the missing bounds check on `.BMX` frame indices means a
> corrupted or oversized effect frame index would be a genuine
> out-of-bounds read in the original binary. Nothing else traced in this
> codebase skips that check — every other frame-array access (`.ANX`,
> and the archive/resource lookups in §6) validates before indexing.
> A clean-room reader implementing `.BMX` playback should add the bounds
> check the original omits, rather than replicate the gap.

---

## 15. Appendix E: The `.PLX` Palette Format (resolves the §2.2/§7 palette gap)

`.ANX` pixels are 8bpp indices with no embedded palette — the palette
turns out to live in a **separate resource family, `.PLX`**, loaded
through the exact same container/decompression pipeline as `.ANX`/`.BMX`
(§1). Traced end-to-end via `LoadPaletteFile_PLX` (formerly
`FUN_00406150`).

### 15.1 Why there are multiple palettes

Confirmed filenames present in the binary's string table:

```
mainmenu.plx   bioroom.plx   vatdoor.plx   vrloop.plx
medlab.plx     sbstart.plx   simgui.plx    BGMPALS.PLX
```

These are **per-scene/per-context palettes** — a different `.plx` loads
depending on which screen or area is active (main menu, a specific room,
a VR sequence, etc.). This is exactly why the palette isn't embedded in
`.ANX` itself: the same sprite resource is reused across multiple palette
contexts, so baking in one fixed palette per sprite would defeat that
reuse.

> **Note on an earlier false lead:** a resource named `WORLDCOL`
> (referenced in `LoadHexMapResources_ByScenario`, §"hex-grid" work
> earlier in this analysis) was investigated as a palette candidate
> since it loads a 1024-byte block — the right size for 256 RGBA
> entries. It was ruled out: that data is actually consumed as a
> color-remap/team-color lookup table for the sprite-blit pipeline
> (`BlitSprite_ColorRemap_*`), not as a base display palette. `.PLX` is
> the correct, verified source.

### 15.2 `.PLX` payload layout

Unlike `.ANX`, the payload (after the optional `PKX:` container, §1, is
stripped exactly as before) has **no header at all** — just a flat,
tightly packed table:

```
+0x0000  PALETTEENTRY[256]     4 bytes each: {peRed, peGreen, peBlue, peFlags}
                                (native Win32 PALETTEENTRY byte order)
                                = 1024 bytes total, no other fields
```

### 15.3 How the loaded palette is actually applied

`LoadPaletteFile_PLX` reads the `.PLX` payload into a buffer, then calls
`SetPaletteRange_ClampedToUsableSlots(buffer, 0, 0x100)` (formerly
`FUN_0047b430`), which does something important:

```c
// paraphrased from the decompiled logic
usableStart = max(rangeStart, 10);           // clamp low end to slot 10
usableEnd   = min(rangeStart+rangeCount-1, 0xF5);  // clamp high end to slot 245
count = usableEnd - usableStart + 1;
copy count entries from payload[usableStart*4 ..] into a staging buffer
```

Then `PaletteManager_ApplyEntries` (formerly `FUN_00480bf0`) fills the
**first 10 and last 10 palette slots (indices 0–9 and 246–255)** with
`GetSystemPaletteEntries()` — i.e. **the fixed Windows reserved system
palette** — overriding whatever the `.plx` file itself contains there.
Only the middle **236 entries (indices 10–245)** actually come from the
`.plx` file's own data. The merged 256-entry table is then:

1. Converted to `RGBQUAD` format via `ConvertLogPaletteToRGBQuad`
   (formerly `FUN_00485490`) — reorders `{R,G,B,flags}` → `{B,G,R,0}` and
   strips the 4-byte `LOGPALETTE` header — for use with `SetDIBColorTable`
   (software/DIB rendering path).
2. Also built into a real GDI palette via `CreatePalette`, then
   `SelectPalette`/`RealizePalette` (hardware palette path).

`RefreshActivePalette` (formerly `FUN_0047b410`) is a lightweight
re-apply — it just re-selects the already-built palette object into the
device context (e.g. on window focus regain), without reloading or
re-merging anything.

### 15.4 What this means for `.ANX` decoding

This closes the loop opened in §2.2/§7: **`.ANX` pixel index 0 being
hard-coded as the transparency key is independent of whatever color
happens to sit in palette slot 0** — slot 0 falls inside the
Windows-reserved range (0–9) and is never populated from any `.plx`
file's own data anyway, so it was never meant to be treated as a "real"
color for sprite rendering purposes.

For a complete, accurate-color `.ANX` renderer: load the relevant
`.plx` for the scene you're rendering (§15.2), splice in your platform's
equivalent of the reserved slots 0–9/246–255 (or simply treat those
slots as unused, since `.ANX`/`.BMX` sprites appear to only ever reference
palette-index 0 for transparency — no evidence was found of sprite
pixels intentionally using other reserved-range indices), and use
entries 10–245 as the true 236-color image palette.

---

## 16. Companion Tool

A standalone HTML viewer (`anx_viewer.html`, no dependencies, no build
step) implements this spec exactly — the same LZSS/RLE algorithms from
§8/§10/§11, byte-for-byte — for drag-and-drop decoding and previewing of
real `.ANX` files or pasted `ucDataBlock`-style C byte arrays. It
currently renders each frame as a grayscale or false-color PNG rather
than true color, since it predates the `.PLX` palette work (§15) — a
future revision could load a `.plx` file alongside the `.ANX` and render
accurate colors using the entry-10–245 mapping described there. Useful
as-is for quickly sanity-checking a new sample file's structure without
writing a script.
