# .BOX File Format Specification

**Game:** CyberStorm  
**Purpose:** UI screen layout — widget positions and sizes  
**Derived by:** Static analysis with Ghidra (`Screen_LoadBoxLayout()` and related functions)  
**All field names are inferred.**

---

## 1. Overview

`.BOX` files are binary screen layout descriptors. They define the position and size of UI widgets on a given game screen, and are loaded at screen initialisation time by `Screen_LoadBoxLayout()`, parsed in a single pass from a raw byte buffer.

The format contains **no magic number**, **no version field**, and **no compression**. It is read in full and interpreted directly. Widget graphics and resources are **not** stored in the `.BOX` file — they are bound separately by `Screen_InitWidgetsWithResources()` using a corresponding `.FLX` resource archive.

All integer values are **little-endian**.

---

## 2. File Structure

The file is divided into three consecutive sections with no padding between them.

### 2.1 Screen Header (bytes 0–15)

Four signed 32-bit integers describing the screen's base position and two additional fields.

| Offset | Size | Type     | Field      | Notes |
|--------|------|----------|------------|-------|
| +0     | 4    | INT32 LE | `screen_x` | Base X position of the screen on the 640×480 canvas |
| +4     | 4    | INT32 LE | `screen_y` | Base Y position of the screen on the 640×480 canvas |
| +8     | 4    | INT32 LE | `field2`   | Purpose TBD. Copied into `screen_struct[2]`. Observed value: `0xFF` |
| +12    | 4    | INT32 LE | `field3`   | Purpose TBD. Copied into `screen_struct[3]` |

### 2.2 Reserved / Padding (bytes 16–19)

| Offset | Size | Type     | Field        | Notes |
|--------|------|----------|--------------|-------|
| +16    | 4    | INT32 LE | *(reserved)* | Not used by the loader. Read as `piVar3[4]` but skipped during header copy. Observed value: `0` |

> ⚠ **The widget count is NOT stored in the `.BOX` file.** It is read from `screen_struct[7]`, which must be initialised before calling `Screen_LoadBoxLayout()`.

### 2.3 Widget Records (bytes 20–EOF)

Immediately follows the reserved field. Each record is exactly **20 bytes**. The number of records equals `screen_struct[7]` (the widget count held by the calling screen object).

#### Per-widget record layout (20 bytes)

| Offset | Size | Type     | Field      | Notes |
|--------|------|----------|------------|-------|
| +0     | 4    | INT32 LE | `rel_x`    | X offset relative to `screen_x`. May be negative |
| +4     | 4    | INT32 LE | `rel_y`    | Y offset relative to `screen_y`. May be negative |
| +8     | 4    | INT32 LE | `width`    | Widget width in pixels |
| +12    | 4    | INT32 LE | `height`   | Widget height in pixels |
| +16    | 1    | UINT8    | `extra[0]` | Loaded into widget struct at offset `+0x10` |
| +17    | 1    | UINT8    | `extra[1]` | Loaded into widget struct at offset `+0x11` |
| +18    | 1    | UINT8    | `extra[2]` | Loaded into widget struct at offset `+0x12` |
| +19    | 1    | UINT8    | `extra[3]` | Loaded into widget struct at offset `+0x13` |

#### Absolute position resolution

Absolute screen coordinates are computed at load time and written into each widget struct:

```
abs_x = screen_x + rel_x
abs_y = screen_y + rel_y
```

#### Extra bytes

The four extra bytes at offsets +16–+19 are written directly into the widget struct. Widgets with non-zero extra bytes appear to have associated resource or type identifiers — widgets that form a uniform group (e.g. a menu list) typically share the same `extra[0..1]` value while `extra[2..3]` varies per widget. Widgets with all-zero extra bytes have no special resource binding.

---

## 3. File Size Formula

```
file_size = 20 + (widget_count × 20)
```

A file with 10 widgets is 220 bytes. A file with 0 widgets is 20 bytes (header + reserved only).

**Validation:** if `(file_size − 20) % 20 ≠ 0`, the file is malformed or not a `.BOX` file.

---

## 4. Load Sequence

The following steps are performed by `Screen_LoadBoxLayout()` each time a screen is initialised:

1. Locate the file using `FindResourceFile_InSearchPaths()` — searches the configured asset directories.
2. Get the file size using `FileHandle_GetSize()`.
3. Allocate a scratch buffer of that size using `FLX_Scratch_Alloc()`.
4. Read the entire file into the buffer in a single call using `FileHandle_Read()`.
5. Parse: copy the four screen header fields into `screen_struct[0..3]`.
6. Iterate `widget_count` times, resolving each widget's absolute position and copying extra bytes into the widget struct.
7. Close the file handle.
8. Free the scratch buffer using `FLX_Scratch_Free()`.

> ⚠ The format is never decompressed — unlike `.BMX` bitmap files which use `LoadResourceFile_WithDecompression()`, `.BOX` files are read raw.

---

## 5. Relationship to Other Formats

The `.BOX` file handles placement only. The full screen pipeline is:

| Format | Role |
|--------|------|
| `.BOX` | Widget positions and sizes (this format) |
| `.FLX` | Visual resources (bitmaps, animations) bound to widget slots by index |
| `.BMX` | Background/GUI bitmaps loaded separately before the layout |
| `.PLX` | Palette files loaded alongside the background |

After `Screen_LoadBoxLayout()` completes, `Screen_InitWidgetsWithResources()` is called to assign graphics to each widget by looking them up in the `.FLX` archive using the widget's index plus a base resource ID.

---

## 6. Known .BOX Files (CyberStorm)

| Filename        | Widget count | Screen / purpose |
|-----------------|:------------:|------------------|
| `mainmenu.box`  | ?            | Main menu screen |
| `topmenu.box`   | ?            | Top menu |
| `promo.box`     | ?            | Promotional / intro screen |
| `bioroom.box`   | ?            | Bio room — common setup |
| `biovat.box`    | ?            | Bio room — biovat sub-screen |
| `vrmenu.box`    | ?            | Bio room — VR menu |
| `medlab.box`    | ?            | Bio room — Med Lab |
| `Standby.box`   | ?            | Standby screen |
| `pilotgui.box`  | ?            | Pilot GUI |
| `commscrn.box`  | ?            | Communications screen |
| `missbrf.box`   | ?            | Mission briefing |
| `missdbrf.box`  | ?            | Mission debrief |
| `hercbay.box`   | ?            | Herc bay |
| `hercbuy.box`   | ?            | Herc purchase screen |
| `hbmanage.box`  | ?            | Herc bay management |
| `hbupgrad.box`  | ?            | Herc bay upgrade |
| `qstart.box`    | ?            | Quick start |
| `mphbase.box`   | ?            | Multiplayer base |
| `mplaunch.box`  | ?            | Multiplayer launch |
| `Hotseat.box`   | ?            | Hotseat multiplayer |
| `hotbrief.box`  | ?            | Hotseat briefing |
| `BIOMENU.BOX`   | **10**       | Bio room menu — confirmed via file analysis |
| `POPMOVE.BOX`   | ?            | Movement popup |
| `POPWEAP.BOX`   | ?            | Weapon popup |
| `POPSYS.BOX`    | ?            | System popup |
| `POPSHLD.BOX`   | ?            | Shield popup |
| `DMGGUI.BOX`    | ?            | Damage display GUI |
| `BGMGUI.BOX`    | ?            | BGM GUI |
| `POPREPT.BOX`   | ?            | Report popup |
| `POPINFO.BOX`   | ?            | Info popup |
| `POPWORLD.BOX`  | ?            | World popup |
| `SIMALERT.BOX`  | ?            | Simulation alert |
| `MAPALERT.BOX`  | ?            | Map alert |

---

## 7. Reader Pseudocode

```
function ReadBoxFile(path, widget_count):
    data     = ReadAllBytes(path)
    expected = 20 + widget_count * 20
    assert len(data) == expected

    screen_x = INT32_LE(data, 0)
    screen_y = INT32_LE(data, 4)
    field2   = INT32_LE(data, 8)
    field3   = INT32_LE(data, 12)
    // bytes 16–19 reserved / skip

    widgets = []
    for i in 0 .. widget_count - 1:
        off   = 20 + i * 20
        rel_x = INT32_LE(data, off)
        rel_y = INT32_LE(data, off + 4)
        w     = INT32_LE(data, off + 8)
        h     = INT32_LE(data, off + 12)
        extra = data[off+16 : off+20]   // 4 bytes

        widgets.append({
            abs_x:  screen_x + rel_x,
            abs_y:  screen_y + rel_y,
            width:  w,
            height: h,
            extra:  extra,
        })

    return { screen_x, screen_y, field2, field3, widgets }
```

---

## 8. Open Questions

The following fields require further investigation:

- **`field2` / `field3`** — copied into `screen_struct[2]` and `[3]` but their use downstream is not yet traced.
- **`extra[0..3]`** — non-zero on grouped widgets. Likely encodes widget type and/or a resource index into the `.FLX` archive.
- **Widget count source** — not stored in the file; must be known before calling the loader. Likely hardcoded per screen or stored in a screen descriptor table in the executable.

---

*Specification derived by static analysis with Ghidra. All field names are inferred.*
