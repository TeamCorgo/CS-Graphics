# PLY Polygon Region Format — Binary File Specification

**Applies to:** MechWarrior 2 / Ghost Bear's Legacy / Mercenaries (Activision, 1995–1996)  
**Extension:** `.ply`  
**Endianness:** Little-endian  
**Scalar types:** 32-bit signed integers (`int32_t`) throughout  
**Reverse-engineered from:** Ghidra static analysis of the Win95 executable

---

## 1. Overview

A `.ply` file defines an ordered array of 2-D convex or concave polygon regions used
for mouse hit-testing in the game's menu and UI screens. Each record describes one
polygon by storing its vertex list and vertex count in a fixed-size 132-byte slot.

The engine loads a `.ply` file when a UI screen initialises, then on every mouse event
iterates every record and calls `Polygon_PointInside(record, mouse_x, mouse_y)` to
determine which region the cursor is over. The index of the first matching record
becomes the active menu item index.

Known files:

| Filename        | Used by screen              |
|-----------------|-----------------------------|
| `bioroom.ply`   | BioRoom pilot selection     |
| `<name>.ply`    | Top menu bar (dynamic load) |

---

## 2. File Layout

```
File = Record[0], Record[1], …, Record[N-1]
```

The record count `N` is derived at load time:

```c
N = file_size_bytes / 0x84;   // integer division; remainder bytes are ignored
```

There is **no file header**. The file begins immediately with the first record at
byte offset `0x00`.

---

## 3. Record Structure

Each record is exactly **132 bytes (0x84)** and is self-contained.

```
Offset  Size   Type       Field
──────────────────────────────────────────────────────────────
0x00    8      int32[2]   vertex[0]  — first (x, y) pair
0x08    8      int32[2]   vertex[1]
0x10    8      int32[2]   vertex[2]
0x18    8      int32[2]   vertex[3]
0x20    8      int32[2]   vertex[4]
0x28    8      int32[2]   vertex[5]
0x30    8      int32[2]   vertex[6]
0x38    8      int32[2]   vertex[7]
0x40    8      int32[2]   vertex[8]
0x48    8      int32[2]   vertex[9]
0x50    8      int32[2]   vertex[10]
0x58    8      int32[2]   vertex[11]
0x60    8      int32[2]   vertex[12]
0x68    8      int32[2]   vertex[13]
0x70    8      int32[2]   vertex[14]
0x78    8      int32[2]   vertex[15]
0x80    4      int32      vertex_count  — number of valid vertex[] entries
0x84    —      —          (end of record / start of next)
```

In C:

```c
#pragma pack(push, 1)
typedef struct {
    int32_t verts[16][2];   /* [v][0] = x, [v][1] = y */
    int32_t vertex_count;   /* 1–16; values outside this range are invalid */
} PLY_Record;               /* sizeof == 0x84 == 132 */
#pragma pack(pop)
```

### 3.1 Vertex array

- Slots `verts[0]` through `verts[vertex_count - 1]` are valid screen coordinates.
- Slots `verts[vertex_count]` through `verts[15]` are **unused padding** and may
  contain any value; the engine does not read them.
- Coordinates are in screen pixels, with the origin `(0, 0)` at the top-left corner
  of the game window. `x` increases rightward; `y` increases downward.
- Values are **signed** — coordinates outside the visible screen area (negative or
  beyond screen dimensions) are legal and are used for off-screen clipping geometry.

### 3.2 vertex_count

- Valid range: **1 – 16** (inclusive).
- The engine's `Polygon_PointInside` function only performs a meaningful hit-test when
  `vertex_count > 2`; records with fewer than 3 vertices will never match.
- The engine does **not** validate this field before use; a corrupted value outside
  1–16 will cause an out-of-bounds read.

---

## 4. Coordinate System

```
(0,0) ──────────────────► x
  │
  │       polygon vertex (x, y)
  │
  ▼
  y
```

Coordinates match the Win95 640×480 screen space used by the game's UI renderer.
Typical values are in the range `[0, 639]` for x and `[0, 479]` for y.

---

## 5. Hit-Test Algorithm

The engine uses a **ray-casting point-in-polygon** test. A horizontal ray is cast
rightward from the query point `(px, py)`, and the number of polygon edge crossings
is counted; an odd count means the point is inside.

Edge cases handled by the engine (via four independent passes with slightly different
tie-breaking rules) include:

- The query point lies exactly on a horizontal edge.
- The query point lies exactly on a vertex.
- The query point lies exactly on a non-horizontal edge.

The function returns `1` (inside) or `0` (outside):

```c
char Polygon_PointInside(PLY_Record *record, int px, int py);
```

The first record (lowest index) whose `Polygon_PointInside` returns `1` wins;
subsequent records are not checked.

---

## 6. Pseudo-code Loader

```c
PLY_Record *g_regions    = NULL;
uint32_t    g_regionCount = 0;

void LoadPLY(const char *filename) {
    FILE *f = fopen(filename, "rb");
    if (!f) { g_regionCount = 0; return; }

    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    rewind(f);

    g_regionCount = (uint32_t)(size / sizeof(PLY_Record));
    g_regions = malloc(g_regionCount * sizeof(PLY_Record));
    fread(g_regions, sizeof(PLY_Record), g_regionCount, f);
    fclose(f);
}

int HitTest(int mouse_x, int mouse_y) {
    for (uint32_t i = 0; i < g_regionCount; i++) {
        if (Polygon_PointInside(&g_regions[i], mouse_x, mouse_y))
            return (int)i;
    }
    return -1;   /* no region hit */
}
```

---

## 7. Writing a .ply File

To create a valid `.ply` file:

1. For each polygon region, populate a `PLY_Record`:
   - Write vertex coordinates into `verts[0]` … `verts[n-1]`.
   - Set `vertex_count = n`.
   - Leave remaining `verts[n]` … `verts[15]` as zero (or any value).
2. Concatenate records in hit-priority order (record 0 is tested first).
3. Write the raw bytes with no header or padding between records.

Minimum file size for one triangle: **132 bytes**.  
Maximum useful polygons per file: limited only by file size; the engine allocates
the full buffer at load time.

---

## 8. Summary of Key Constants

| Constant          | Value  | Notes                              |
|-------------------|--------|------------------------------------|
| Record size       | 0x84   | 132 bytes                          |
| Max vertices      | 16     | per record                         |
| Vertex slot size  | 8      | bytes (two int32s: x, y)           |
| vertex_count offset | 0x80 | byte offset within record          |
| Minimum valid count | 3    | for a meaningful hit-test          |
| Coordinate origin | (0,0)  | top-left of 640×480 screen         |
| Scalar type       | int32  | little-endian signed 32-bit        |
