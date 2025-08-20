# doomwadtostl
WAD to STL

```
#!/usr/bin/env python3
# doom_wad_to_stl_walls.py
#
# Export printable STL with *thick walls* from the first DOOM map in a WAD.
# - Two-sided linedefs: wall spans the vertical overlap between the two sectors.
# - One-sided linedefs: wall spans that sector's floor..ceiling and is offset inward.
# Binary STL output. No external deps.
#
# MIT License

import argparse
import os
import re
import struct
from typing import Dict, List, Tuple

LE = "<"  # little-endian

# ---------------------------
# WAD parsing structures
# ---------------------------

class WadLump:
    __slots__ = ("offset", "size", "name")
    def __init__(self, offset: int, size: int, name: str):
        self.offset = offset
        self.size = size
        self.name = name

class WadFile:
    def __init__(self, path: str):
        with open(path, "rb") as f:
            self.data = f.read()
        self._parse_header()

    def _parse_header(self):
        if len(self.data) < 12:
            raise ValueError("Not a valid WAD (too short).")
        ident, numlumps, infotableofs = struct.unpack_from(LE + "4sii", self.data, 0)
        ident = ident.decode("ascii", errors="ignore")
        if ident not in ("IWAD", "PWAD"):
            raise ValueError(f"Unknown WAD type: {ident!r}")
        self.ident = ident
        self.numlumps = numlumps
        self.dir_offset = infotableofs

        self.directory: List[WadLump] = []
        off = self.dir_offset
        for _ in range(self.numlumps):
            lump_off, lump_size, name_raw = struct.unpack_from(LE + "ii8s", self.data, off)
            name = name_raw.split(b"\x00", 1)[0].decode("ascii", errors="ignore")
            self.directory.append(WadLump(lump_off, lump_size, name))
            off += 16

    def read_lump(self, name: str) -> bytes:
        for d in self.directory:
            if d.name.upper() == name.upper():
                return self.data[d.offset : d.offset + d.size]
        raise KeyError(f"Lump not found: {name}")

    def read_lump_by_index(self, idx: int) -> bytes:
        d = self.directory[idx]
        return self.data[d.offset : d.offset + d.size]

# ---------------------------
# Map discovery
# ---------------------------

MAP_NAME_RE = re.compile(r"^(MAP\d\d|E\dM\d)$", re.IGNORECASE)

def find_maps(wad: WadFile) -> List[int]:
    return [i for i, d in enumerate(wad.directory) if MAP_NAME_RE.match(d.name)]

# ---------------------------
# Lump parsers
# ---------------------------

def parse_vertexes(raw: bytes) -> List[Tuple[int, int]]:
    verts = []
    for i in range(0, len(raw), 4):
        x, y = struct.unpack_from(LE + "hh", raw, i)
        verts.append((x, y))
    return verts

def parse_sectors(raw: bytes) -> List[Dict]:
    sectors = []
    stride = 26
    for i in range(0, len(raw), stride):
        floorZ, ceilZ = struct.unpack_from(LE + "hh", raw, i)
        # textures/light/special/tag not used for geometry here
        sectors.append({"floor": floorZ, "ceiling": ceilZ})
    return sectors

def parse_sidedefs(raw: bytes) -> List[Dict]:
    sides = []
    stride = 30
    for i in range(0, len(raw), stride):
        sector = struct.unpack_from(LE + "h", raw, i+28)[0]
        sides.append({"sector": sector})
    return sides

def parse_linedefs(raw: bytes) -> List[Dict]:
    lines = []
    stride = 14
    for i in range(0, len(raw), stride):
        v1, v2, flags, typ, tag, rightSide, leftSide = struct.unpack_from(LE + "hhhhhhh", raw, i)
        lines.append({
            "v1": v1, "v2": v2,
            "flags": flags, "type": typ, "tag": tag,
            "right": rightSide, "left": leftSide
        })
    return lines

# ---------------------------
# STL writer
# ---------------------------

def write_binary_stl(tris, out_path: str, header_text: str):
    header = header_text.encode("ascii", "ignore")[:80].ljust(80, b"\x00")
    with open(out_path, "wb") as f:
        f.write(header)
        f.write(struct.pack(LE + "I", len(tris)))
        for n, a, b, c in tris:
            f.write(struct.pack(LE + "fff", *n))
            f.write(struct.pack(LE + "fff", *a))
            f.write(struct.pack(LE + "fff", *b))
            f.write(struct.pack(LE + "fff", *c))
            f.write(struct.pack(LE + "H", 0))

def normal(a, b, c):
    ax, ay, az = a; bx, by, bz = b; cx, cy, cz = c
    ux, uy, uz = (bx-ax, by-ay, bz-az)
    vx, vy, vz = (cx-ax, cy-ay, cz-az)
    return (uy*vz - uz*vy, uz*vx - ux*vz, ux*vy - uy*vx)

# ---------------------------
# Wall builder (thick prisms)
# ---------------------------

def sector_heights(side_idx: int, sidedefs, sectors, default_floor=0.0, default_ceil=128.0):
    if 0 <= side_idx < len(sidedefs):
        sidx = sidedefs[side_idx]["sector"]
        if 0 <= sidx < len(sectors):
            sec = sectors[sidx]
            return float(sec["floor"]), float(sec["ceiling"])
    return float(default_floor), float(default_ceil)

def add_prism(tris, quad_bottom, z0, z1):
    """
    quad_bottom: list of 4 (x,y) in order around the rectangle.
    Creates side faces + top/bottom faces (closed prism).
    """
    # Make 8 verts
    v = []
    for (x, y) in quad_bottom:
        v.append((x, y, z0))
    for (x, y) in quad_bottom:
        v.append((x, y, z1))
    # Indices: bottom 0..3, top 4..7
    faces = [
        (0,1,2,3),  # bottom
        (4,5,6,7),  # top
        (0,1,5,4),  # sides
        (1,2,6,5),
        (2,3,7,6),
        (3,0,4,7),
    ]
    for a,b,c,d in faces:
        # two tris (a,b,c) and (a,c,d)
        n1 = normal(v[a], v[b], v[c])
        n2 = normal(v[a], v[c], v[d])
        tris.append((n1, v[a], v[b], v[c]))
        tris.append((n2, v[a], v[c], v[d]))

def thick_wall_quads(verts, linedefs, sidedefs, sectors, scale=1.0, zscale=1.0,
                     wall_thickness=8.0, default_height=128.0):
    tris = []
    half_t = wall_thickness * 0.5

    for ld in linedefs:
        v1i, v2i = ld["v1"], ld["v2"]
        if not (0 <= v1i < len(verts) and 0 <= v2i < len(verts)):
            continue
        (x1, y1) = verts[v1i]; (x2, y2) = verts[v2i]
        dx = x2 - x1; dy = y2 - y1
        L = (dx*dx + dy*dy) ** 0.5
        if L == 0:
            continue

        # Right-hand unit normal for v1->v2 (points to "right" sidedef)
        rx =  dy / L
        ry = -dx / L

        # Heights on each side
        rf, rc = sector_heights(ld["right"], sidedefs, sectors, 0.0, default_height)
        lf, lc = sector_heights(ld["left"],  sidedefs, sectors, 0.0, default_height)

        # Decide vertical span and lateral offsets
        if ld["right"] >= 0 and ld["left"] >= 0:
            # Two-sided: build wall only where the two sectors overlap vertically
            z0 = max(rf, lf)
            z1 = min(rc, lc)
            if z1 <= z0:
                continue  # opening (e.g., doorway) → no wall
            # Center thickness on the linedef
            off_cx, off_cy = 0.0, 0.0
        elif ld["right"] >= 0:
            # One-sided, right only → offset thickness into right sector
            z0, z1 = rf, rc
            off_cx, off_cy = rx * (half_t), ry * (half_t)
        elif ld["left"] >= 0:
            # One-sided, left only → offset thickness into left sector (negative right normal)
            z0, z1 = lf, lc
            off_cx, off_cy = -rx * (half_t), -ry * (half_t)
        else:
            # Degenerate linedef; skip
            continue

        # Build the rectangle corners (2D), thickness centered or offset as above
        # Two parallel lines offset ±half_t along the right-normal
        # Start/end along the original segment direction.
        px1 = x1 + off_cx; py1 = y1 + off_cy
        px2 = x2 + off_cx; py2 = y2 + off_cy

        # Outer/inner offsets
        o1x = px1 + rx * half_t; o1y = py1 + ry * half_t
        o2x = px2 + rx * half_t; o2y = py2 + ry * half_t
        i1x = px1 - rx * half_t; i1y = py1 - ry * half_t
        i2x = px2 - rx * half_t; i2y = py2 - ry * half_t

        # Scale to output units
        quad2d = [
            (i1x * scale, i1y * scale),
            (i2x * scale, i2y * scale),
            (o2x * scale, o2y * scale),
            (o1x * scale, o1y * scale),
        ]
        z0s = z0 * zscale
        z1s = z1 * zscale

        add_prism(tris, quad2d, z0s, z1s)

    return tris

# ---------------------------
# Map read helper
# ---------------------------

def read_map_lumps(wad: WadFile, map_dir_index: int) -> Dict[str, bytes]:
    needed = {"VERTEXES", "LINEDEFS", "SIDEDEFS", "SECTORS"}
    out: Dict[str, bytes] = {}
    for i in range(map_dir_index + 1, len(wad.directory)):
        name = wad.directory[i].name.upper()
        if MAP_NAME_RE.match(name):
            break
        if name in needed:
            out[name] = wad.read_lump_by_index(i)
        if len(out) == 4:
            break
    missing = [k for k in needed if k not in out]
    if missing:
        raise ValueError(f"Map missing required lumps: {missing}")
    return out

# ---------------------------
# CLI
# ---------------------------

def main():
    ap = argparse.ArgumentParser(description="Convert a DOOM WAD map to STL with thick, printable walls.")
    ap.add_argument("wad", help="Input WAD file")
    ap.add_argument("out", help="Output STL path")
    ap.add_argument("--map-name", help="Map name to export (e.g., MAP01 or E1M1)")
    ap.add_argument("--map-index", type=int, default=0, help="If no map-name, choose Nth discovered map (default 0)")
    ap.add_argument("--scale", type=float, default=1.0, help="XY scale (1.0 = DOOM units)")
    ap.add_argument("--zscale", type=float, default=1.0, help="Z scale (vertical)")
    ap.add_argument("--wall-thickness", type=float, default=8.0, help="Wall thickness in DOOM units (pre-scale)")
    ap.add_argument("--default-height", type=float, default=128.0, help="Fallback ceiling if sector missing")
    args = ap.parse_args()

    wad = WadFile(args.wad)

    # Pick map
    if args.map_name:
        name_upper = args.map_name.upper()
        cand = [i for i, d in enumerate(wad.directory) if d.name.upper() == name_upper]
        if not cand:
            raise SystemExit(f"Map {args.map_name} not found.")
        map_idx = cand[0]
        map_name = name_upper
    else:
        maps = find_maps(wad)
        if not maps:
            raise SystemExit("No maps found in WAD.")
        if not (0 <= args.map_index < len(maps)):
            raise SystemExit(f"--map-index out of range (0..{len(maps)-1}).")
        map_idx = maps[args.map_index]
        map_name = wad.directory[map_idx].name

    lumps = read_map_lumps(wad, map_idx)
    verts = parse_vertexes(lumps["VERTEXES"])
    lines = parse_linedefs(lumps["LINEDEFS"])
    sides = parse_sidedefs(lumps["SIDEDEFS"])
    secs  = parse_sectors(lumps["SECTORS"])

    tris = thick_wall_quads(
        verts, lines, sides, secs,
        scale=args.scale, zscale=args.zscale,
        wall_thickness=args.wall_thickness,
        default_height=args.default_height
    )

    if not tris:
        raise SystemExit("No wall geometry generated.")

    header = f"DOOM map {map_name} with thick walls"
    write_binary_stl(tris, args.out, header)
    print(f"Wrote {len(tris)} triangles to {args.out}")

if __name__ == "__main__":
    main()

```
