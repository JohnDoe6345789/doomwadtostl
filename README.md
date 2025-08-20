# doomwadtostl
WAD to STL

```
#!/usr/bin/env python3
# doom_wad_to_stl_solid.py
#
# Printable STL from DOOM WAD:
# - Thick walls with door openings.
# - Floors: solid-filled down to the base (default) OR thin plates (optional).
# - Base plate under the whole model.
# - Auto-scale to target max XY (mm) so slicer imports at 100%.
#
# MIT License. No external deps.

import argparse
import re
import struct
from typing import Dict, List, Tuple

LE = "<"  # little-endian

# ---------------------------
# WAD parsing
# ---------------------------

class WadLump:
    __slots__ = ("offset", "size", "name")
    def __init__(self, offset: int, size: int, name: str):
        self.offset = offset; self.size = size; self.name = name

class WadFile:
    def __init__(self, path: str):
        with open(path, "rb") as f:
            self.data = f.read()
        self._parse_header()

    def _parse_header(self):
        if len(self.data) < 12: raise ValueError("Not a valid WAD (too short).")
        ident, numlumps, infotableofs = struct.unpack_from(LE + "4sii", self.data, 0)
        ident = ident.decode("ascii", errors="ignore")
        if ident not in ("IWAD", "PWAD"): raise ValueError(f"Unknown WAD type: {ident!r}")
        self.ident = ident; self.numlumps = numlumps; self.dir_offset = infotableofs
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
# STL writer & math
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

def bbox_of_points(pts: List[Tuple[float,float]]) -> Tuple[float, float, float, float]:
    xs = [p[0] for p in pts]; ys = [p[1] for p in pts]
    return min(xs), min(ys), max(xs), max(ys)

def add_prism(tris, loop2d, z0, z1):
    # Fan caps + quad sides; loop2d must be CCW
    v0_bot = (loop2d[0][0], loop2d[0][1], z0)
    v0_top = (loop2d[0][0], loop2d[0][1], z1)
    for i in range(1, len(loop2d)-1):
        a = (loop2d[i][0],     loop2d[i][1],     z0)
        b = (loop2d[i+1][0],   loop2d[i+1][1],   z0)
        tris.append((normal(v0_bot, b, a), v0_bot, b, a))
        A = (loop2d[i][0],     loop2d[i][1],     z1)
        B = (loop2d[i+1][0],   loop2d[i+1][1],   z1)
        tris.append((normal(v0_top, A, B), v0_top, A, B))
    for i in range(len(loop2d)):
        j = (i+1) % len(loop2d)
        a_bot = (loop2d[i][0], loop2d[i][1], z0)
        b_bot = (loop2d[j][0], loop2d[j][1], z0)
        b_top = (loop2d[j][0], loop2d[j][1], z1)
        a_top = (loop2d[i][0], loop2d[i][1], z1)
        tris.append((normal(a_bot, b_bot, b_top), a_bot, b_bot, b_top))
        tris.append((normal(a_bot, b_top, a_top), a_bot, b_top, a_top))

def ear_clip_triangulate(poly: List[Tuple[float,float]]) -> List[Tuple[int,int,int]]:
    if len(poly) < 3: return []
    def signed_area(P):
        A = 0.0
        for i in range(len(P)):
            x1,y1 = P[i]; x2,y2 = P[(i+1)%len(P)]
            A += x1*y2 - x2*y1
        return A*0.5
    P = poly[:]
    if signed_area(P) < 0: P = P[::-1]
    idx = list(range(len(P))); tris = []
    def is_convex(i0,i1,i2):
        x1,y1=P[i0]; x2,y2=P[i1]; x3,y3=P[i2]
        return (x2-x1)*(y3-y1) - (y2-y1)*(x3-x1) > 0
    def point_in_tri(px,py, ax,ay, bx,by, cx,cy):
        v0x,v0y = cx-ax, cy-ay
        v1x,v1y = bx-ax, by-ay
        v2x,v2y = px-ax, py-ay
        den = v0x*v1y - v1x*v0y
        if abs(den) < 1e-12: return False
        u = (v2x*v1y - v1x*v2y)/den
        v = (v0x*v2y - v2x*v0y)/den
        return (u >= 0) and (v >= 0) and (u+v <= 1)
    guard=0
    while len(idx) > 3 and guard < 10000:
        ear=False
        for k in range(len(idx)):
            i0 = idx[(k-1)%len(idx)]; i1 = idx[k]; i2 = idx[(k+1)%len(idx)]
            if not is_convex(i0,i1,i2): continue
            ax,ay=P[i0]; bx,by=P[i1]; cx,cy=P[i2]
            inside=False
            for j in idx:
                if j in (i0,i1,i2): continue
                px,py=P[j]
                if point_in_tri(px,py, ax,ay, bx,by, cx,cy):
                    inside=True; break
            if inside: continue
            tris.append((i0,i1,i2))
            del idx[k]; ear=True; break
        if not ear:
            base=idx[0]
            for t in range(1,len(idx)-1): tris.append((base,idx[t],idx[t+1]))
            idx=idx[:1]
        guard+=1
    if len(idx)==3: tris.append((idx[0],idx[1],idx[2]))
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
    tris = []; half_t = wall_thickness * 0.5
    for ld in linedefs:
        v1i, v2i = ld["v1"], ld["v2"]
        if not (0 <= v1i < len(verts) and 0 <= v2i < len(verts)): continue
        (x1, y1) = verts[v1i]; (x2, y2) = verts[v2i]
        dx = x2 - x1; dy = y2 - y1; L = (dx*dx + dy*dy) ** 0.5
        if L == 0: continue
        rx =  dy / L; ry = -dx / L  # right-hand normal
        rf, rc = sector_heights(ld["right"], sidedefs, sectors, 0.0, default_height)
        lf, lc = sector_heights(ld["left"],  sidedefs, sectors, 0.0, default_height)
        if ld["right"] >= 0 and ld["left"] >= 0:
            z0 = max(rf, lf); z1 = min(rc, lc)
            if z1 <= z0: continue  # opening
            offcx, offcy = 0.0, 0.0
        elif ld["right"] >= 0:
            z0, z1 = rf, rc; offcx, offcy = rx*half_t, ry*half_t
        elif ld["left"] >= 0:
            z0, z1 = lf, lc; offcx, offcy = -rx*half_t, -ry*half_t
        else:
            continue
        px1,py1 = x1+offcx, y1+offcy; px2,py2 = x2+offcx, y2+offcy
        o1x,o1y = px1 + rx*half_t, py1 + ry*half_t
        o2x,o2y = px2 + rx*half_t, py2 + ry*half_t
        i1x,i1y = px1 - rx*half_t, py1 - ry*half_t
        i2x,i2y = px2 - rx*half_t, py2 - ry*half_t
        quad = [(i1x*scale,i1y*scale),(i2x*scale,i2y*scale),(o2x*scale,o2y*scale),(o1x*scale,o1y*scale)]
        add_prism(tris, quad, z0*zscale, z1*zscale)
    return tris

# ---------------------------
# Sector boundaries → loops
# ---------------------------

def sector_boundary_loops(sector_idx: int, verts, linedefs, sidedefs) -> List[List[Tuple[float,float]]]:
    edges = []
    for ld in linedefs:
        r = ld["right"]; l = ld["left"]
        if 0 <= r < len(sidedefs) and sidedefs[r]["sector"] == sector_idx and not (0 <= l < len(sidedefs) and sidedefs[l]["sector"] == sector_idx):
            edges.append((ld["v1"], ld["v2"]))
        elif 0 <= l < len(sidedefs) and sidedefs[l]["sector"] == sector_idx and not (0 <= r < len(sidedefs) and sidedefs[r]["sector"] == sector_idx):
            edges.append((ld["v2"], ld["v1"]))
    from collections import defaultdict
    outgoing = defaultdict(list)
    for a,b in edges: outgoing[a].append(b)
    loops = []; used=set()
    for a,b in edges:
        if (a,b) in used: continue
        chain=[a,b]; used.add((a,b)); cur=b
        while True:
            nxts=[n for n in outgoing[cur] if (cur,n) not in used]
            if not nxts: break
            n=nxts[0]; chain.append(n); used.add((cur,n)); cur=n
            if cur==chain[0]: break
        if chain[0]==chain[-1] and len(chain)>3: chain=chain[:-1]
        loop=[(float(verts[i][0]), float(verts[i][1])) for i in chain]
        # CCW
        area=0.0
        for i in range(len(loop)):
            x1,y1=loop[i]; x2,y2=loop[(i+1)%len(loop)]
            area += x1*y2 - x2*y1
        if area < 0: loop.reverse()
        if len(loop) >= 3: loops.append(loop)
    return loops

# ---------------------------
# Floors (solid or thin)
# ---------------------------

def build_sector_floors_solid(verts, linedefs, sidedefs, sectors, scale, zscale, base_top_z_mm):
    """
    Solid fill: for each sector, extrude its polygon from floorZ down to base_top_z_mm.
    """
    tris=[]
    for sidx, sec in enumerate(sectors):
        z_top = sec["floor"] * zscale   # sector floor height in mm
        z_bot = base_top_z_mm           # down to base top (solid ground)
        if z_top <= z_bot:  # if sector floor is at or below base, skip
            continue
        loops = sector_boundary_loops(sidx, verts, linedefs, sidedefs)
        for loop in loops:
            loop_s = [(x*scale, y*scale) for (x,y) in loop]
            add_prism(tris, loop_s, z_bot, z_top)
    return tris

def build_sector_floors_thin(verts, linedefs, sidedefs, sectors, scale, zscale, floor_thickness):
    """
    Thin plates: useful for diorama style; not used when solid floors enabled.
    """
    tris=[]
    for sidx, sec in enumerate(sectors):
        z_top = sec["floor"] * zscale
        z_bot = (sec["floor"] - floor_thickness) * zscale
        loops = sector_boundary_loops(sidx, verts, linedefs, sidedefs)
        for loop in loops:
            loop_s = [(x*scale, y*scale) for (x,y) in loop]
            add_prism(tris, loop_s, z_bot, z_top)
    return tris

# ---------------------------
# Base plate
# ---------------------------

def build_base_plate(all_points_xy: List[Tuple[float,float]], top_z: float,
                     margin=10.0, thickness=2.0):
    tris=[]
    xmin,ymin,xmax,ymax = bbox_of_points(all_points_xy)
    xmin-=margin; ymin-=margin; xmax+=margin; ymax+=margin
    loop=[(xmin,ymin),(xmax,ymin),(xmax,ymax),(xmin,ymax)]
    add_prism(tris, loop, top_z - thickness, top_z)
    return tris

# ---------------------------
# CLI / main
# ---------------------------

def read_map_lumps(wad: WadFile, map_dir_index: int) -> Dict[str, bytes]:
    needed={"VERTEXES","LINEDEFS","SIDEDEFS","SECTORS"}
    out={}
    for i in range(map_dir_index+1, len(wad.directory)):
        name=wad.directory[i].name.upper()
        if MAP_NAME_RE.match(name): break
        if name in needed: out[name]=wad.read_lump_by_index(i)
        if len(out)==4: break
    missing=[k for k in needed if k not in out]
    if missing: raise ValueError(f"Map missing required lumps: {missing}")
    return out

def main():
    import argparse
    ap = argparse.ArgumentParser(description="Convert a DOOM WAD map to printable STL (walls + floors + base).")
    ap.add_argument("wad"); ap.add_argument("out")
    ap.add_argument("--map-name"); ap.add_argument("--map-index", type=int, default=0)
    # Sizing
    ap.add_argument("--scale", type=float, default=None, help="mm per DOOM unit (overridden by --autoscale-mm)")
    ap.add_argument("--autoscale-mm", type=float, default=200.0, help="Fit longest XY to this many mm (0=disable)")
    ap.add_argument("--zscale", type=float, default=None, help="mm per DOOM unit for Z (default = same as XY)")
    # Geometry
    ap.add_argument("--wall-thickness", type=float, default=8.0, help="Wall thickness in DOOM units (pre-scale)")
    ap.add_argument("--default-height", type=float, default=128.0)
    # Floors & base
    ap.add_argument("--solid-floors", type=int, default=1, help="1=solid fill floors (default), 0=thin slabs")
    ap.add_argument("--floor-thickness", type=float, default=2.0, help="Only used when --solid-floors 0")
    ap.add_argument("--base-margin", type=float, default=10.0)
    ap.add_argument("--base-thickness", type=float, default=2.0)
    args = ap.parse_args()

    wad = WadFile(args.wad)
    # Pick map
    if args.map_name:
        name_upper=args.map_name.upper()
        cand=[i for i,d in enumerate(wad.directory) if d.name.upper()==name_upper]
        if not cand: raise SystemExit(f"Map {args.map_name} not found.")
        map_idx=cand[0]; map_name=name_upper
    else:
        maps=find_maps(wad)
        if not maps: raise SystemExit("No maps found in WAD.")
        if not (0 <= args.map_index < len(maps)): raise SystemExit(f"--map-index out of range (0..{len(maps)-1}).")
        map_idx=maps[args.map_index]; map_name=wad.directory[map_idx].name

    lumps = read_map_lumps(wad, map_idx)
    verts = parse_vertexes(lumps["VERTEXES"])
    lines = parse_linedefs(lumps["LINEDEFS"])
    sides = parse_sidedefs(lumps["SIDEDEFS"])
    secs  = parse_sectors(lumps["SECTORS"])

    # Compute mm scale
    xs=[v[0] for v in verts]; ys=[v[1] for v in verts]
    span_x=(max(xs)-min(xs)) if xs else 1.0
    span_y=(max(ys)-min(ys)) if ys else 1.0
    longest=max(span_x, span_y) or 1.0
    if args.autoscale_mm and args.autoscale_mm > 0:
        scale = float(args.autoscale_mm) / float(longest)
    else:
        scale = args.scale if args.scale is not None else 0.05
    zscale = args.zscale if args.zscale is not None else scale

    tris=[]
    # Walls
    tris += build_thick_walls(verts, lines, sides, secs,
                              scale=scale, zscale=zscale,
                              wall_thickness=args.wall_thickness,
                              default_height=args.default_height)
    if not tris: raise SystemExit("No wall geometry generated.")

    # Base top sits at the LOWEST sector floor
    min_floor = min([s["floor"] for s in secs]) if secs else 0.0
    base_top_z_mm = min_floor * zscale

    # Floors
    if args.solid_floors:
        tris += build_sector_floors_solid(verts, lines, sides, secs,
                                          scale=scale, zscale=zscale,
                                          base_top_z_mm=base_top_z_mm)
    else:
        tris += build_sector_floors_thin(verts, lines, sides, secs,
                                         scale=scale, zscale=zscale,
                                         floor_thickness=args.floor_thickness)

    # Base plate
    all_xy=[(v[0]*scale, v[1]*scale) for v in verts]
    tris += build_base_plate(all_xy, base_top_z_mm, margin=args.base_margin, thickness=args.base_thickness)

    header=f"DOOM map {map_name} (walls + {'solid' if args.solid_floors else 'thin'} floors + base)"
    write_binary_stl(tris, args.out, header)
    print(f"[OK] Wrote {len(tris)} triangles to {args.out}")
    print(f"[INFO] XY scale = {scale:.5f} mm/u, Z scale = {zscale:.5f} mm/u")
    print(f"[INFO] Model XY ≈ {span_x*scale:.1f} x {span_y*scale:.1f} mm (before base margin)")
    print(f"[INFO] Floors: {'SOLID to base' if args.solid_floors else f'thin ({args.floor_thickness*zscale:.2f} mm)'}; Base = {args.base_thickness} mm")

if __name__ == "__main__":
    main()

```
