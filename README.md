# doomwadtostl
WAD to STL

```
#!/usr/bin/env python3
# doom_wad_to_stl_renderer.py  (sturdy defaults)
#
# Export an STL that matches what classic DOOM draws:
# - Floors = thin plates at sector floor heights (void below).
# - Walls = one-sided full-height; two-sided only lower/upper gaps (doors stay open).
# - Optional ceilings as thin plates.
# - Rectangular base plate under the lowest floor.
# No external deps. MIT License.

import argparse, re, struct
from typing import Dict, List, Tuple

LE = "<"  # little endian

# ---------------- WAD parsing ----------------

class WadLump:
    __slots__ = ("offset","size","name")
    def __init__(self, off:int, size:int, name:str):
        self.offset=off; self.size=size; self.name=name

class WadFile:
    def __init__(self, path:str):
        with open(path, "rb") as f: self.data=f.read()
        self._parse_header()
    def _parse_header(self):
        if len(self.data) < 12: raise ValueError("Not a WAD (too short).")
        ident, numlumps, dir_ofs = struct.unpack_from(LE+"4sii", self.data, 0)
        ident = ident.decode("ascii", "ignore")
        if ident not in ("IWAD","PWAD"): raise ValueError(f"Bad WAD ident {ident!r}")
        self.ident=ident; self.numlumps=numlumps; self.dir_offset=dir_ofs
        self.directory=[]
        p=dir_ofs
        for _ in range(numlumps):
            off,size,name_raw = struct.unpack_from(LE+"ii8s", self.data, p)
            name = name_raw.split(b"\x00",1)[0].decode("ascii","ignore")
            self.directory.append(WadLump(off,size,name))
            p+=16
    def read_lump_by_index(self, idx:int)->bytes:
        d=self.directory[idx]; return self.data[d.offset:d.offset+d.size]

MAP_NAME_RE = re.compile(r"^(MAP\d\d|E\dM\d)$", re.IGNORECASE)
def find_maps(wad:WadFile)->List[int]:
    return [i for i,d in enumerate(wad.directory) if MAP_NAME_RE.match(d.name)]

# ---------------- Lump decoders ----------------

def parse_vertexes(raw:bytes)->List[Tuple[int,int]]:
    out=[]; step=4
    for i in range(0,len(raw),step):
        x,y = struct.unpack_from(LE+"hh", raw, i); out.append((x,y))
    return out

def parse_sectors(raw:bytes)->List[Dict]:
    out=[]; step=26
    for i in range(0,len(raw),step):
        floor, ceil = struct.unpack_from(LE+"hh", raw, i)
        out.append({"floor":floor, "ceiling":ceil})
    return out

def parse_sidedefs(raw:bytes)->List[Dict]:
    out=[]; step=30
    for i in range(0,len(raw),step):
        sector = struct.unpack_from(LE+"h", raw, i+28)[0]
        out.append({"sector":sector})
    return out

def parse_linedefs(raw:bytes)->List[Dict]:
    out=[]; step=14
    for i in range(0,len(raw),step):
        v1,v2,flags,typ,tag,rs,ls = struct.unpack_from(LE+"hhhhhhh", raw, i)
        out.append({"v1":v1,"v2":v2,"flags":flags,"type":typ,"tag":tag,"right":rs,"left":ls})
    return out

# ---------------- STL & math ----------------

def write_binary_stl(tris, path:str, header="doom renderer export"):
    header_bytes = header.encode("ascii","ignore")[:80].ljust(80,b"\x00")
    with open(path,"wb") as f:
        f.write(header_bytes)
        f.write(struct.pack(LE+"I", len(tris)))
        for n,a,b,c in tris:
            f.write(struct.pack(LE+"fff", *n))
            f.write(struct.pack(LE+"fff", *a))
            f.write(struct.pack(LE+"fff", *b))
            f.write(struct.pack(LE+"fff", *c))
            f.write(struct.pack(LE+"H", 0))

def normal(a,b,c):
    ax,ay,az=a; bx,by,bz=b; cx,cy,cz=c
    ux,uy,uz = bx-ax, by-ay, bz-az
    vx,vy,vz = cx-ax, cy-ay, cz-az
    return (uy*vz-uz*vy, uz*vx-ux*vz, ux*vy-uy*vx)

def bbox2d(pts:List[Tuple[float,float]]):
    xs=[p[0] for p in pts]; ys=[p[1] for p in pts]
    return min(xs),min(ys),max(xs),max(ys)

def add_prism(tris, loop, z0, z1):
    v0b=(loop[0][0], loop[0][1], z0)
    v0t=(loop[0][0], loop[0][1], z1)
    for i in range(1,len(loop)-1):
        a=(loop[i][0], loop[i][1], z0)
        b=(loop[i+1][0], loop[i+1][1], z0)
        tris.append((normal(v0b,b,a), v0b,b,a))
        A=(loop[i][0], loop[i][1], z1)
        B=(loop[i+1][0], loop[i+1][1], z1)
        tris.append((normal(v0t,A,B), v0t,A,B))
    for i in range(len(loop)):
        j=(i+1)%len(loop)
        aB=(loop[i][0], loop[i][1], z0)
        bB=(loop[j][0], loop[j][1], z0)
        bT=(loop[j][0], loop[j][1], z1)
        aT=(loop[i][0], loop[i][1], z1)
        tris.append((normal(aB,bB,bT), aB,bB,bT))
        tris.append((normal(aB,bT,aT), aB,bT,aT))

def add_wall_panel(tris, x1_mm,y1_mm, x2_mm,y2_mm, z0_mm,z1_mm, thickness_mm):
    dx=x2_mm-x1_mm; dy=y2_mm-y1_mm
    L=(dx*dx+dy*dy)**0.5
    if L <= 0: return
    rx =  dy/L; ry = -dx/L
    hx = rx * (thickness_mm*0.5); hy = ry*(thickness_mm*0.5)
    p1=(x1_mm-hx, y1_mm-hy); p2=(x2_mm-hx, y2_mm-hy)
    p3=(x2_mm+hx, y2_mm+hy); p4=(x1_mm+hx, y1_mm+hy)
    add_prism(tris, [p1,p2,p3,p4], z0_mm, z1_mm)

# ---------------- Geometry helpers ----------------

def sector_heights(side_idx:int, sidedefs, sectors, default_floor=0.0, default_ceil=128.0):
    if 0 <= side_idx < len(sidedefs):
        s = sidedefs[side_idx]["sector"]
        if 0 <= s < len(sectors):
            return float(sectors[s]["floor"]), float(sectors[s]["ceiling"])
    return float(default_floor), float(default_ceil)

def sector_outer_loops(sector_idx:int, verts, linedefs, sidedefs)->List[List[Tuple[float,float]]]:
    edges=[]
    for ld in linedefs:
        r,l = ld["right"], ld["left"]
        r_ok = 0<=r<len(sidedefs) and sidedefs[r]["sector"]==sector_idx
        l_ok = 0<=l<len(sidedefs) and sidedefs[l]["sector"]==sector_idx
        if r_ok and not l_ok: edges.append((ld["v1"], ld["v2"]))
        elif l_ok and not r_ok: edges.append((ld["v2"], ld["v1"]))
    from collections import defaultdict
    out=[]; used=set(); nexts=defaultdict(list)
    for a,b in edges: nexts[a].append(b)
    for a,b in edges:
        if (a,b) in used: continue
        loop=[a,b]; used.add((a,b)); cur=b
        while True:
            nxt=[n for n in nexts[cur] if (cur,n) not in used]
            if not nxt: break
            n=nxt[0]; loop.append(n); used.add((cur,n)); cur=n
            if cur==loop[0]: break
        if loop[0]==loop[-1] and len(loop)>3: loop=loop[:-1]
        pts=[(float(verts[i][0]), float(verts[i][1])) for i in loop]
        area=0.0
        for i in range(len(pts)):
            x1,y1=pts[i]; x2,y2=pts[(i+1)%len(pts)]
            area += x1*y2 - x2*y1
        if area < 0: pts.reverse()
        if len(pts)>=3: out.append(pts)
    return out

# ---------------- Build geometry (renderer-style) ----------------

def build_renderer_floors(verts, linedefs, sidedefs, sectors, scale, zscale, floor_thickness_mm):
    tris=[]
    for s_idx,_ in enumerate(sectors):
        loops = sector_outer_loops(s_idx, verts, linedefs, sidedefs)
        if not loops: continue
        z_top = sectors[s_idx]["floor"] * zscale
        z_bot = z_top - floor_thickness_mm
        for loop in loops:
            loop_mm=[(x*scale, y*scale) for (x,y) in loop]
            add_prism(tris, loop_mm, z_bot, z_top)
    return tris

def build_renderer_walls(verts, linedefs, sidedefs, sectors, scale, zscale, panel_thickness_mm, default_height=128.0):
    tris=[]
    for ld in linedefs:
        v1,v2 = ld["v1"], ld["v2"]
        if not (0<=v1<len(verts) and 0<=v2<len(verts)): continue
        x1_mm = verts[v1][0]*scale; y1_mm = verts[v1][1]*scale
        x2_mm = verts[v2][0]*scale; y2_mm = verts[v2][1]*scale

        rf,rc = sector_heights(ld["right"], sidedefs, sectors, 0.0, default_height)
        lf,lc = sector_heights(ld["left"],  sidedefs, sectors, 0.0, default_height)

        if ld["right"] >= 0 and ld["left"] >= 0:
            if rf != lf:
                z0 = min(rf, lf) * zscale
                z1 = max(rf, lf) * zscale
                if z1 > z0:
                    add_wall_panel(tris, x1_mm,y1_mm, x2_mm,y2_mm, z0, z1, panel_thickness_mm)
            if rc != lc:
                z0 = min(rc, lc) * zscale
                z1 = max(rc, lc) * zscale
                if z1 > z0:
                    add_wall_panel(tris, x1_mm,y1_mm, x2_mm,y2_mm, z0, z1, panel_thickness_mm)
        else:
            if ld["right"] >= 0:
                z0 = rf * zscale; z1 = rc * zscale
            elif ld["left"] >= 0:
                z0 = lf * zscale; z1 = lc * zscale
            else:
                continue
            if z1 > z0:
                add_wall_panel(tris, x1_mm,y1_mm, x2_mm,y2_mm, z0, z1, panel_thickness_mm)
    return tris

def build_base_plate(all_xy_mm:List[Tuple[float,float]], top_z_mm:float, margin_mm=10.0, thickness_mm=4.0):
    tris=[]
    xmin,ymin,xmax,ymax = bbox2d(all_xy_mm)
    xmin-=margin_mm; ymin-=margin_mm; xmax+=margin_mm; ymax+=margin_mm
    add_prism(tris, [(xmin,ymin),(xmax,ymin),(xmax,ymax),(xmin,ymax)],
              top_z_mm - thickness_mm, top_z_mm)
    return tris

# ---------------- CLI ----------------

def read_map_lumps(wad:WadFile, map_dir_index:int)->Dict[str,bytes]:
    needed={"VERTEXES","LINEDEFS","SIDEDEFS","SECTORS"}; out={}
    for i in range(map_dir_index+1, len(wad.directory)):
        name=wad.directory[i].name.upper()
        if MAP_NAME_RE.match(name): break
        if name in needed: out[name]=wad.read_lump_by_index(i)
        if len(out)==4: break
    miss=[k for k in needed if k not in out]
    if miss: raise ValueError(f"Map missing lumps: {miss}")
    return out

def main():
    ap = argparse.ArgumentParser(description="DOOM renderer-style STL (floors + visible walls, optional ceilings).")
    ap.add_argument("wad"); ap.add_argument("out")
    ap.add_argument("--map-name"); ap.add_argument("--map-index", type=int, default=0)

    # Sizing
    ap.add_argument("--autoscale-mm", type=float, default=200.0, help="Fit longest XY to this many mm (0=disable)")
    ap.add_argument("--scale", type=float, default=None, help="mm per DOOM unit (overridden by autoscale if >0)")
    ap.add_argument("--zscale", type=float, default=None, help="mm per DOOM unit for Z (default = scale)")

    # Thickness (physical) — STURDY DEFAULTS
    ap.add_argument("--floor-thickness-mm", type=float, default=3.0, help="Floor plate thickness (mm)")
    ap.add_argument("--wall-thickness-mm",  type=float, default=3.0, help="Wall panel thickness (mm)")
    ap.add_argument("--add-ceilings", type=int, default=0, help="1=add thin ceiling plates")
    ap.add_argument("--ceiling-thickness-mm", type=float, default=2.0)

    # Base (sturdier default)
    ap.add_argument("--base-margin-mm", type=float, default=10.0)
    ap.add_argument("--base-thickness-mm", type=float, default=4.0)

    # Misc
    ap.add_argument("--default-height", type=float, default=128.0)

    args = ap.parse_args()

    wad = WadFile(args.wad)
    if args.map_name:
        name=args.map_name.upper()
        matches=[i for i,d in enumerate(wad.directory) if d.name.upper()==name]
        if not matches: raise SystemExit(f"Map {args.map_name} not found.")
        map_idx=matches[0]; map_name=name
    else:
        maps=find_maps(wad)
        if not maps: raise SystemExit("No maps found.")
        if not (0<=args.map_index<len(maps)): raise SystemExit(f"--map-index 0..{len(maps)-1}")
        map_idx=maps[args.map_index]; map_name=wad.directory[map_idx].name

    lumps=read_map_lumps(wad, map_idx)
    verts=parse_vertexes(lumps["VERTEXES"])
    lines=parse_linedefs(lumps["LINEDEFS"])
    sides=parse_sidedefs(lumps["SIDEDEFS"])
    secs =parse_sectors(lumps["SECTORS"])

    xs=[v[0] for v in verts]; ys=[v[1] for v in verts]
    span_x=(max(xs)-min(xs)) if xs else 1.0
    span_y=(max(ys)-min(ys)) if ys else 1.0
    longest=max(span_x, span_y) or 1.0
    if args.autoscale_mm and args.autoscale_mm>0:
        scale=args.autoscale_mm/float(longest)
    else:
        scale=args.scale if args.scale is not None else 0.05
    zscale=args.zscale if args.zscale is not None else scale

    tris=[]
    tris += build_renderer_floors(verts, lines, sides, secs, scale, zscale, args.floor_thickness_mm)
    tris += build_renderer_walls(verts, lines, sides, secs, scale, zscale, args.wall_thickness_mm, args.default_height)

    all_xy_mm=[(v[0]*scale, v[1]*scale) for v in verts]
    min_floor = min([s["floor"] for s in secs]) if secs else 0.0
    base_top_z = min_floor * zscale
    tris += build_base_plate(all_xy_mm, base_top_z, args.base_margin_mm, args.base_thickness_mm)

    write_binary_stl(tris, args.out, f"DOOM renderer style {map_name}")
    print(f"[OK] Wrote {len(tris)} triangles to {args.out}")
    print(f"[INFO] XY scale = {scale:.5f} mm/u, Z scale = {zscale:.5f} mm/u")
    print(f"[INFO] Model spans ≈ {span_x*scale:.1f} x {span_y*scale:.1f} mm (before base margin)")
    print(f"[INFO] Floors: {args.floor_thickness_mm} mm, Walls: {args.wall_thickness_mm} mm, Base: {args.base_thickness_mm} mm")

if __name__ == "__main__":
    main()
```
