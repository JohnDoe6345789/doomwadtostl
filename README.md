# doomwadtostl
WAD to STL

```
#!/usr/bin/env python3
# doom_wad_to_stl_solid.py
#
# Printable STL from DOOM WAD:
# - Thick walls (handles door openings).
# - Triangulated floors per sector (so stairs/steps appear).
# - Base plate under the whole model for easy bed removal.
# - Optional auto-scale to a target max XY size (in mm) to avoid slicer scaling.
#
# No external dependencies. MIT License

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
# Geometry utils
# ---------------------------

def bbox_of_points(pts: List[Tuple[float,float]]) -> Tuple[float, float, float, float]:
    xs = [p[0] for p in pts]; ys = [p[1] for p in pts]
    return min(xs), min(ys), max(xs), max(ys)

def add_prism(tris, loop2d, z0, z1):
    """
    Closed prism from a simple polygon (no holes).
    loop2d must be CCW. Adds:
      - top and bottom caps (fan)
      - side walls
    """
    # top/bottom caps (fan)
    v0_bot = (loop2d[0][0], loop2d[0][1], z0)
    v0_top = (loop2d[0][0], loop2d[0][1], z1)
    for i in range(1, len(loop2d)-1):
        a = (loop2d[i][0],     loop2d[i][1],     z0)
        b = (loop2d[i+1][0],   loop2d[i+1][1],   z0)
        n = normal(v0_bot, b, a)        # bottom (reverse winding)
        tris.append((n, v0_bot, b, a))
        A = (loop2d[i][0],     loop2d[i][1],     z1)
        B = (loop2d[i+1][0],   loop2d[i+1][1],   z1)
        n2 = normal(v0_top, A, B)       # top
        tris.append((n2, v0_top, A, B))
    # sides
    for i in range(len(loop2d)):
        j = (i+1) % len(loop2d)
        a_bot = (loop2d[i][0], loop2d[i][1], z0)
        b_bot = (loop2d[j][0], loop2d[j][1], z0)
        b_top = (loop2d[j][0], loop2d[j][1], z1)
        a_top = (loop2d[i][0], loop2d[i][1], z1)
        n1 = normal(a_bot, b_bot, b_top)
        n2 = normal(a_bot, b_top, a_top)
        tris.append((n1, a_bot, b_bot, b_top))
        tris.append((n2, a_bot, b_top, a_top))

def ear_clip_triangulate(poly: List[Tuple[float,float]]) -> List[Tuple[int,int,int]]:
    """
    Minimal ear clipping for simple polygon (no holes), returns triangle indices.
    Ensures CCW before triangulation.
    """
    if len(poly) < 3: return []
    # Ensure CCW
    def signed_area(P):
        A = 0.0
        for i in range(len(P)):
            x1,y1 = P[i]; x2,y2 = P[(i+1)%len(P)]
            A += x1*y2 - x2*y1
        return A*0.5
    P = poly[:]
    if signed_area(P) < 0:
        P = P[::-1]

    indices = list(range(len(P)))
    tris = []

    def is_convex(i0,i1,i2):
        x1,y1 = P[i0]; x2,y2 = P[i1]; x3,y3 = P[i2]
        return (x2-x1)*(y3-y1) - (y2-y1)*(x3-x1) > 0

    def point_in_tri(px,py, ax,ay, bx,by, cx,cy):
        # Barycentric
        v0x,v0y = cx-ax, cy-ay
        v1x,v1y = bx-ax, by-ay
        v2x,v2y = px-ax, py-ay
        den = v0x*v1y - v1x*v0y
        if abs(den) < 1e-12: return False
        u = (v2x*v1y - v1x*v2y)/den
        v = (v0x*v2y - v2x*v0y)/den
        return (u >= 0) and (v >= 0) and (u+v <= 1)

    count = 0
    while len(indices) > 3 and count < 10000:
        ear_found = False
        for k in range(len(indices)):
            i0 = indices[(k-1) % len(indices)]
            i1 = indices[k]
            i2 = indices[(k+1) % len(indices)]
            if not is_convex(i0,i1,i2):
                continue
            ax,ay = P[i0]; bx,by = P[i1]; cx,cy = P[i2]
            any_inside = False
            for j in indices:
                if j in (i0,i1,i2): continue
                px,py = P[j]
                if point_in_tri(px,py, ax,ay, bx,by, cx,cy):
                    any_inside = True; break
            if any_inside: continue
            tris.append((i0,i1,i2))
            del indices[k]
            ear_found = True
            break
        if not ear_found:
            # Fallback: fan triangulation
            base = indices[0]
            for t in range(1, len(indices)-1):
                tris.append((base, indices[t], indices[t+1]))
            indices = indices[:1]
        count += 1

    if len(indices) == 3:
        tris.append((indices[0], indices[1], indices[2]))
    return tris

# ---------------------------
# Walls (thick prisms)
# ---------------------------

def sector_heights(side_idx: int, sidedefs, sectors, default_floor=0.0, default_ceil=128.0):
    if 0 <= side_idx < len(sidedefs):
        sidx = sidedefs[side_idx]["sector"]
        if 0 <= sidx < len(sectors):
            sec = sectors[sidx]
            return float(sec["floor"]), float(sec["ceiling"])
    return float(default_floor), float(default_ceil)

def build_thick_walls(verts, linedefs, sidedefs, sectors,
                      scale=1.0, zscale=1.0, wall_thickness=8.0, default_height=128.0):
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

        # Right-hand unit normal for v1->v2 (points toward "right" sidedef)
        rx =  dy / L
        ry = -dx / L

        rf, rc = sector_heights(ld["right"], sidedefs, sectors, 0.0, default_height)
        lf, lc = sector_heights(ld["left"],  sidedefs, sectors, 0.0, default_height)

        if ld["right"] >= 0 and ld["left"] >= 0:
            z0 = max(rf, lf)
            z1 = min(rc, lc)
            if z1 <= z0:
                continue  # doorway/opening
            off_cx, off_cy = 0.0, 0.0
        elif ld["right"] >= 0:
            z0, z1 = rf, rc
            off_cx, off_cy = rx * (half_t), ry * (half_t)
        elif ld["left"] >= 0:
            z0, z1 = lf, lc
            off_cx, off_cy = -rx * (half_t), -ry * (half_t)
        else:
            continue

        px1 = x1 + off_cx; py1 = y1 + off_cy
        px2 = x2 + off_cx; py2 = y2 + off_cy

        o1x = px1 + rx * half_t; o1y = py1 + ry * half_t
        o2x = px2 + rx * half_t; o2y = py2 + ry * half_t
        i1x = px1 - rx * half_t; i1y = py1 - ry * half_t
        i2x = px2 - rx * half_t; i2y = py2 - ry * half_t

        quad = [
            (i1x * scale, i1y * scale),
            (i2x * scale, i2y * scale),
            (o2x * scale, o2y * scale),
            (o1x * scale, o1y * scale),
        ]
        z0s = z0 * zscale
        z1s = z1 * zscale
        add_prism(tris, quad, z0s, z1s)
    return tris

# ---------------------------
# Sector floor loops
# ---------------------------

def sector_boundary_loops(sector_idx: int, verts, linedefs, sidedefs) -> List[List[Tuple[float,float]]]:
    """
    Extract oriented boundary edges for a sector by walking linedefs:
    - if right sector == s and left != s, add edge v1->v2 (sector interior on left).
    - if left sector == s and right != s, add edge v2->v1 (still interior on left).
    Then chain edges into loops.
    NOTE: Ignores holes (inner loops will come out as separate polygons).
    """
    edges = []
    for ld in linedefs:
        r = ld["right"]; l = ld["left"]
        if 0 <= r and r < len(sidedefs) and sidedefs[r]["sector"] == sector_idx and not (0 <= l < len(sidedefs) and sidedefs[l]["sector"] == sector_idx):
            edges.append( (ld["v1"], ld["v2"]) )
        elif 0 <= l and l < len(sidedefs) and sidedefs[l]["sector"] == sector_idx and not (0 <= r < len(sidedefs) and sidedefs[r]["sector"] == sector_idx):
            edges.append( (ld["v2"], ld["v1"]) )
    # Chain into loops
    from collections import defaultdict
    outgoing = defaultdict(list)
    for a,b in edges:
        outgoing[a].append(b)

    loops = []
    used = set()
    for a,b in edges:
        if (a,b) in used: continue
        chain = [a, b]
        used.add((a,b))
        cur = b
        while True:
            nxts = [n for n in outgoing[cur] if (cur,n) not in used]
            if not nxts:
                break
            n = nxts[0]
            chain.append(n)
            used.add((cur,n))
            cur = n
            if cur == chain[0]:
                break
        if chain[0] == chain[-1] and len(chain) > 3:
            chain = chain[:-1]
        loop = [(float(verts[i][0]), float(verts[i][1])) for i in chain]
        # Force CCW by area
        area = 0.0
        for i in range(len(loop)):
            x1,y1 = loop[i]; x2,y2 = loop[(i+1)%len(loop)]
            area += x1*y2 - x2*y1
        if area < 0:
            loop.reverse()
        if len(loop) >= 3:
            loops.append(loop)
    return loops

def build_sector_floors(verts, linedefs, sidedefs, sectors, scale=1.0, zscale=1.0, floor_thickness=2.0):
    """
    For each sector:
      - find boundary loops (outer edges),
      - triangulate each loop (no holes),
      - extrude downward by floor_thickness to make a plate.
    Stairs/steps appear because each sector uses its own floor Z.
    """
    tris = []
    for sidx, sec in enumerate(sectors):
        z_top = sec["floor"] * zscale
        z_bot = (sec["floor"] - floor_thickness) * zscale
        loops = sector_boundary_loops(sidx, verts, linedefs, sidedefs)
        for loop in loops:
            loop_s = [(x*scale, y*scale) for (x,y) in loop]
            tri_idx = ear_clip_triangulate(loop_s)
            # Top cap
            for a,b,c in tri_idx:
                A = (loop_s[a][0], loop_s[a][1], z_top)
                B = (loop_s[b][0], loop_s[b][1], z_top)
                C = (loop_s[c][0], loop_s[c][1], z_top)
                n = normal(A, B, C)
                tris.append((n, A, B, C))
            # Bottom cap (reversed)
            for a,b,c in tri_idx:
                A = (loop_s[a][0], loop_s[a][1], z_bot)
                B = (loop_s[b][0], loop_s[b][1], z_bot)
                C = (loop_s[c][0], loop_s[c][1], z_bot)
                n = normal(C, B, A)
                tris.append((n, C, B, A))
            # sides
            for i in range(len(loop_s)):
                j = (i+1) % len(loop_s)
                a_top = (loop_s[i][0], loop_s[i][1], z_top)
                b_top = (loop_s[j][0], loop_s[j][1], z_top)
                b_bot = (loop_s[j][0], loop_s[j][1], z_bot)
                a_bot = (loop_s[i][0], loop_s[i][1], z_bot)
                n1 = normal(a_bot, b_bot, b_top)
                n2 = normal(a_bot, b_top, a_top)
                tris.append((n1, a_bot, b_bot, b_top))
                tris.append((n2, a_bot, b_top, a_top))
    return tris

# ---------------------------
# Base plate
# ---------------------------

def build_base_plate(all_points_xy: List[Tuple[float,float]], top_z: float,
                     margin=10.0, thickness=2.0):
    """
    Rectangle around the model (with margin), from (top_z - thickness) to top_z.
    margin & thickness are already in *mm* space (post-scale).
    """
    tris = []
    xmin,ymin,xmax,ymax = bbox_of_points(all_points_xy)
    xmin -= margin; ymin -= margin; xmax += margin; ymax += margin
    loop = [(xmin,ymin),(xmax,ymin),(xmax,ymax),(xmin,ymax)]
    add_prism(tris, loop, top_z - thickness, top_z)
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
    ap = argparse.ArgumentParser(description="Convert a DOOM WAD map to printable STL (walls + floors + base).")
    ap.add_argument("wad", help="Input WAD file")
    ap.add_argument("out", help="Output STL path")
    ap.add_argument("--map-name", help="Map name to export (e.g., MAP01 or E1M1)")
    ap.add_argument("--map-index", type=int, default=0, help="If no map-name, choose Nth discovered map (default 0)")
    # Sizing
    ap.add_argument("--scale", type=float, default=None, help="XY scale in mm per DOOM unit (overridden by --autoscale-mm)")
    ap.add_argument("--autoscale-mm", type=float, default=200.0,
                    help="Auto-scale so the longest XY dimension ≈ this many mm (default 200). Set to 0 to disable.")
    ap.add_argument("--zscale", type=float, default=None, help="Z scale (mm per DOOM unit). Default = same as XY scale.")
    # Geometry
    ap.add_argument("--wall-thickness", type=float, default=8.0, help="Wall thickness in DOOM units (pre-scale)")
    ap.add_argument("--default-height", type=float, default=128.0, help="Fallback ceiling if sector missing")
    ap.add_argument("--floor-thickness", type=float, default=2.0, help="Thickness of each sector floor slab (DOOM units)")
    # Base
    ap.add_argument("--base-margin", type=float, default=10.0, help="Base plate margin around model (mm, after scaling)")
    ap.add_argument("--base-thickness", type=float, default=2.0, help="Base plate thickness (mm)")
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

    # Pre-scale bbox to compute autoscale factor
    xs = [v[0] for v in verts]; ys = [v[1] for v in verts]
    minx, maxx = min(xs), max(xs)
    miny, maxy = min(ys), max(ys)
    span_x = maxx - minx
    span_y = maxy - miny
    longest = max(span_x, span_y) if max(span_x, span_y) > 0 else 1.0

    if args.autoscale_mm and args.autoscale_mm > 0:
        scale = float(args.autoscale_mm) / float(longest)  # mm per DOOM unit
    else:
        scale = args.scale if args.scale is not None else 0.05  # sensible default
    zscale = args.zscale if args.zscale is not None else scale

    tris = []
    # Walls
    tris += build_thick_walls(
        verts, lines, sides, secs,
        scale=scale, zscale=zscale,
        wall_thickness=args.wall_thickness,
        default_height=args.default_height
    )

    if not tris:
        raise SystemExit("No wall geometry generated (empty map?).")

    # Floors (per sector, thin slabs extruded downward)
    tris += build_sector_floors(
        verts, lines, sides, secs,
        scale=scale, zscale=zscale,
        floor_thickness=args.floor_thickness
    )

    # Base plate at the lowest sector floor height
    all_xy = [(v[0]*scale, v[1]*scale) for v in verts]
    min_floor = min([s["floor"] for s in secs]) if secs else 0.0
    base_top_z = min_floor * zscale
    tris += build_base_plate(
        all_points_xy=all_xy,
        top_z=base_top_z,
        margin=args.base_margin,
        thickness=args.base_thickness
    )

    header = f"DOOM map {map_name} (walls+floors+base)"
    write_binary_stl(tris, args.out, header)
    print(f"[OK] Wrote {len(tris)} triangles to {args.out}")
    print(f"[INFO] XY scale = {scale:.5f} mm/u, Z scale = {zscale:.5f} mm/u")
    print(f"[INFO] Model spans ≈ {span_x*scale:.1f} x {span_y*scale:.1f} mm (before base margin)")
    print(f"[INFO] Base thickness = {args.base_thickness} mm, floor slab thickness = {args.floor_thickness*zscale:.2f} mm")
    print("[TIP] If stairs look blocky, that’s classic DOOM sector steps (correct by design).")

if __name__ == "__main__":
    main()

```
