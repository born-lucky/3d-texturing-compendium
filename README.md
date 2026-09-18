# 3D Texturing: A Deep Practical Compendium

**Game art · VFX · Real-time · Lookdev**  
Compiled September 2026. From Catmull 1974 to OpenPBR and Substance 3D Painter 12.1.

This repository is the public markdown edition of a production reference covering UVs, PBR maps, baking, tools, engines, virtual texturing, compression, and the best books and courses.

---

## Start here (free canon)

1. [The PBR Guide Vol. 1 — Theory](https://www.adobe.com/learn/substance-3d-designer/web/the-pbr-guide-part-1) and [Vol. 2 — Practical Guidelines](https://www.adobe.com/learn/substance-3d-designer/web/the-pbr-guide-part-2) (Wes McDermott / Adobe). The texture-artist bible.
2. [Physically Based Rendering: From Theory to Implementation](https://www.pbr-book.org/) (Pharr, Jakob, Humphreys), 4th ed., free.
3. [OpenPBR Surface spec](https://academysoftwarefoundation.github.io/OpenPBR)
4. [Physically-Based Shading at Disney](https://disneyanimation.com/publications/physically-based-shading-at-disney/) (Burley, SIGGRAPH 2012)
5. [physicallybased.info](https://physicallybased.info) — measured F0 and albedo values
6. [Moving Frostbite to PBR](https://seblagarde.wordpress.com/2014/04/14/moving-frostbite-to-pbr/) (Lagarde & de Rousiers)

---

## 1. What texturing actually is

Texturing is not coloring a model. It is authoring **data images** (textures) and projecting them onto a surface so a **shader** can reconstruct how that surface interacts with light under any lighting.

| Term | Meaning |
|---|---|
| Texture | A 2D (sometimes 3D / procedural) array of data. Each pixel is a **texel**. May store color, roughness, a normal, a mask. |
| Shader | The program that evaluates a BRDF/BSDF. Modern default: Cook–Torrance microfacet + GGX. |
| Material | The container that binds textures + scalars into one look. |
| PBR | A methodology, not a file format. Author maps that represent measurable properties so the asset is consistent under any HDRI. |

A shipped real-time asset is: mesh + UVs + map set + shader permutation. Film lookdev adds displacement, coat, SSS, and UDIM stacks.

---

## 2. Short history

| Year | What changed |
|---|---|
| 1974 | Catmull thesis: texture mapping, (u,v), Z-buffer |
| 1976 | Blinn & Newell photograph the Utah teapot; environment maps |
| 1978 | Blinn bump mapping |
| 1983 | Williams *Pyramidal Parametrics* — mipmaps |
| 1993 | Doom popularizes consumer real-time textured 3D |
| Late 90s | Multitexture, lightmaps, then tangent-space normals |
| 2007–11 | id MegaTexture → id Tech 5 virtual texturing (Rage) |
| 2008 | Disney Ptex (no UVs, per-face textures) |
| 2012 | Burley Disney principled BRDF — industry flips to PBR |
| 2016–18 | Allegorithmic PBR Guide; Substance becomes AAA default |
| 2019+ | Basis Universal + KTX2 |
| 2021+ | UE5 Nanite + Streaming/Runtime Virtual Textures |
| 2024–26 | OpenPBR. Painter 12.1 (June 2026) defaults to it |

---

## 3. UVs, texel density, UDIM, Ptex

U and V are texture-space axes. Bad UVs cannot be painted away.

**Process:** clean topology → mark seams → unwrap → relax distortion → pack (≥80% utilization) → lock texel density → checker + bake test.

**Seams:** hide where the eye expects a break. A hard-edge / smoothing split *always* needs a UV split. Prefer fewer, larger islands.

### Texel density

TD = texture pixels per world unit. 1024 px on a 1 m quad = **10.24 px/cm**.

| Tier | Density | Use |
|---|---|---|
| Distant / mobile | ~2.5–5.12 px/cm | Background |
| Third-person env | ~10.24 px/cm | AAA props, kits |
| FPS / hero | ~20.48 px/cm | Weapons, faces |
| Film hero | much higher + UDIM | Creatures, close-ups |

### Padding (for mips)

| Res | Min | Recommended |
|---|---|---|
| 512 | 2 px | 4 px |
| 1024 | 4 px | 8 px |
| 2048 | 8 px | 16 px |
| 4096 | 16 px | 32 px |

Unique UVs for baking and lightmaps. Overlaps OK for tiling, trims, mirrors — never on lightmap UVs.

### UDIM

Invented at Weta for Mari.

```
UDIM = 1001 + u + (10 * v)    u in [0,9], v >= 0
```

Row 0: 1001…1010. Row 1: 1011…1020. File: `Name.1001.png`. Tiles may differ in resolution (face 4K, mouth 1K). Ten-wide U limit is hardwired.

### Ptex

Disney 2008: one texture per face, no unwrap. Film-standard at WDAS. Weak real-time GPU support, so games stay on UV/UDIM. Mari can paint Ptex and convert later.

**Triplanar** projects from X/Y/Z and blends by normal. No UVs; used for rocks, terrain, and as a Painter fill mode.

---

## 4. PBR the artist must know

Microfacet Cook–Torrance:

```
f = F * D * G / (4 * N·L * N·V)
```

- **F** Fresnel (Schlick)
- **D** GGX / Trowbridge–Reitz
- **G** Smith-GGX shadowing-masking

Dielectric F0 ≈ **0.04** linear (2–5%). Metals F0 ≈ 70–100% and **colored**. Metals have almost no diffuse. Metallic is a linear blend between those two BRDFs (Disney 2012). OpenPBR adds coat, fuzz/sheen, SSS, thin-film, emission.

### Metal/Rough vs Spec/Gloss

| | Metal / Rough | Spec / Gloss |
|---|---|---|
| Maps | Base Color, Metallic, Roughness | Diffuse, Specular RGB, Glossiness |
| Dielectric F0 | Usually hardcoded 0.04 | Painted in specular (~40–75 sRGB) |
| Metal color | In Base Color, metallic=1 | In Specular, diffuse black |
| Used by | glTF, UE, Unity HDRP, Blender, Substance | Older UE specular, some VFX |

Unity Smoothness = 1 − Roughness.

### Linear workflow

Lighting math is linear. Displays are sRGB.

- **sRGB ON:** albedo, emissive color, spec-gloss specular color
- **sRGB OFF:** normal, roughness, metallic, AO, height, ORM, curvature, thickness, IDs

### Authoring ranges

- Dielectric albedo: ~30–240 sRGB. Never pure black/white.
- Dirt/rust/paint on metal = dielectric (metallic 0).
- Metallic is not shininess. Shininess is roughness.
- Never bake lighting into albedo for PBR. Stylized hand-paint is the documented exception.

Approx. metal sRGB F0: Iron 198,198,200 · Gold 255,226,155 · Copper 250,208,192 · Alum 245,246,246 · Chrome 196,197,197 · Silver 252,250,245 · Titanium 193,186,177.

---

## 5. Map encyclopedia

**Shipped / shader maps:** Albedo, Metallic, Roughness (or Gloss/Smoothness), Specular (spec/gloss only), Normal, Height/Displacement, AO, Emissive, Opacity, Cavity, SSS/Thickness, Anisotropy, Coat, Sheen/Fuzz, Transmission/IOR.

**Usually baked, often not shipped:** world-space normal, curvature, position, thickness, ID, bent normals, vertex color.

**Relief ladder:** bump → normal → parallax/POM → tessellated displacement → Nanite / true high-poly. Normals do not change silhouette.

**Normal handedness:** Unreal/DirectX = −Y (flip green). Blender/Maya/OpenGL = +Y. Unity importer converts — **use the engine export preset**. Bake and render must share **MikkTSpace**. BC5 stores XY; reconstruct `Z = sqrt(1 - x² - y²)`.

---

## 6. Formats and compression

8-bit is enough for most game color/rough/metal/AO. 16-bit or EXR for displacement. Power-of-two sizes.

Uncompressed 4K RGBA8 ≈ 64 MB mip0, ~85 MB with mips.

| Format | Rate | Use |
|---|---|---|
| BC1/DXT1 | 4 bpp | Cheap opaque albedo |
| BC3/DXT5 | 8 bpp | Albedo + alpha |
| BC4 | 4 bpp | Solo roughness/metallic/AO |
| **BC5** | 8 bpp | **Normal maps** |
| BC6H | 8 bpp | HDR |
| BC7 | 8 bpp | Hero albedo |
| ASTC 6×6 | ~3.56 bpp | Mobile albedo sweet spot |
| Basis ETC1S | small | Color only — bad for normals |
| Basis UASTC + KTX2 | high | Cross-platform, glTF |

Never BC1 a normal map.

---

## 7. Baking

Rays fire from the low-poly (or an inflated **cage** that shares its topology) to the high-poly. Hits become normal, AO, curvature, position, thickness, ID, height.

- Too tight cage → black holes. Too loose → bleeding.
- Match by name (`part_low` / `part_high`). Explode-bake is the fallback.
- Bake 2× ship res + 2×2/4×4 AA. Padding = UV padding. No diffusion on ID.
- Low-poly must hold the silhouette.

**Bakers:** Marmoset (fastest hero, live cage, skew) · Painter 12.1 (auto-rebake, auto-cage, now skew) · Designer v15 GPU · Mari Bakery (UDIM film) · Blender Cycles (free, slow) · xNormal (legacy).

Marmoset → Painter auto-load suffixes: `_normal_base`, `_world_space_normals`, `_ambient_occlusion`, `_curvature`, `_id`, `_thickness`, `_position`.

---

## 8. Tools (2026)

| Job | Tool |
|---|---|
| Game hero texturing | **Substance 3D Painter** 12.1 (OpenPBR default, skew paint, auto-rebake) |
| Procedural libraries | **Substance 3D Designer** (SBS / SBSAR) |
| Photo → material | **Sampler** |
| Film / 100+ UDIM 8K–32K | **Foundry Mari** 7.5 (Multi-Paint, Bakery, Texture Transfer) |
| UE5 photoreal env | **Quixel Mixer + Megascans + Bridge** |
| Hero bakes + lookdev | **Marmoset Toolbag** |
| Free all-in-one | **Blender** + addons (PBR Painter 3, Quick Bake) |
| Free Painter-class | **ArmorPaint 1.0** (Sept 2026) |
| Sculpt color | **ZBrush Polypaint** |
| 2D / tileables / PSD | **Photoshop**, **Krita** |
| Free Designer-like | **Material Maker** |
| Unwrap | **RizomUV**, UVLayout, Blender |

Painter layer stack: paint layers, fill layers, folders, per-channel blend. Generators + baked maps + **anchor points** (reuse one master mask). Smart materials get you 70%; hand-painted story stops the cookie-cutter look.

Designer recipe: height first → derive normal/curvature/AO → masks → gradient-map color → expose parameters → publish SBSAR.

---

## 9. Methods

- **Hand-painted / stylized** — lighting often *in* the albedo. Readable, unique, slow, hard to relight.
- **Procedural** — resolution-independent, scales, can look generic.
- **Scanned / photo** — instant micro-detail. **Delight the albedo.**
- **High-to-low bake** — backbone of real-time assets.
- **Hybrid (studio default)** — scan or procedural base + generator wear + hand-painted story + tiled micro-normal.
- **AI** — blockouts only until a human fixes seams, TD, and direction.

---

## 10. Engines, packing, VT, Nanite

| Target | Packing |
|---|---|
| UE5 ORM | R=AO G=Roughness B=Metallic + BC5 normal |
| Unity HDRP MaskMap | R=Metallic G=AO B=Detail A=**Smoothness** |
| Unity URP | Metallic R + Smoothness A |
| glTF 2.0 | G=Roughness B=Metallic (AO optional in R) |

**SVT** streams artist textures as tiles. **RVT** rasterizes landscape/decals at runtime. Lineage: SGI clipmaps → id MegaTexture → Rage VT → UE virtual texturing.

**Nanite** is virtualized geometry. Use with virtual textures. Can replace some baked hero normals; keep tiling micro-normals for grain.

---

## 11. Pipelines

**Game prop:** concept → high → retopo → UV at locked TD → bake → Painter → packed export → engine LODs/compression.

**Environment:** modular kit + trims + tileables + vertex blend + decals. Only heroes get unique 4K sets.

**Film creature:** ZBrush displacement → UDIM unwrap → Mari → Arnold/V-Ray/RenderMan lookdev. OCIO/ACES.

**Photogrammetry:** capture (cross-polarize if possible) → retopo → bake → **delight** → Painter refine.

---

## 12. Pro rules

1. UVs and TD are half the job.
2. Bake mesh maps before generators.
3. Albedo has no lighting (unless the project is stylized hand-paint).
4. Metallic is 0 or 1. Roughness tells the story.
5. Use the engine export preset for normal-Y.
6. Pack channels. Budget res by viewing distance.
7. Pad for mips. Validate under several HDRIs.
8. Hybrid wins. Reference is not optional.
9. Name maps so a stranger can ingest them.
10. Read licenses on photos, scans, Megascans/Fab.

---

## 13. Books, courses, path

**Books:** *Real-Time Rendering* 4e (Akenine-Möller et al.) · *Texturing & Modeling: A Procedural Approach* 3e (Ebert / Perlin / Worley) · Ahearn *3D Game Textures* · Kumar *Beginning PBR Texturing* · Shah *Realistic Asset Creation with Adobe Substance 3D* · Belec *Photorealistic Materials and Textures in Blender Cycles* · Birn *Digital Lighting & Rendering*.

**Courses:** Adobe official docs · FlippedNormals Painter + baking · Gnomon Workshop Texturing and Materials (~67h) · Foundry Learn Mari · CGMA · Stylized Station Coloring Book · Pablander Academy.

**Communities:** Polycount, 80.lv, ArtStation, Substance Discord.

**Libraries:** Megascans/Fab, Substance 3D Assets, ambientCG, Poly Haven, CGBookcase, FreePBR, Poliigon.

### A sane 3-month path

1. Weeks 1–2: PBR Guide. Author metal, plastic, painted metal, fabric on a sphere.
2. Week 3: Unwrap a crate, lock TD, bake a bevel.
3. Weeks 4–6: One hero prop, bake to packed engine export. No smart-material-only finish.
4. Month 2: One Designer tileable published as SBSAR.
5. Month 3: One stylized hand-painted asset.
6. Ongoing: one ArtStation breakdown a month.

---

## Disclaimer

Study document compiled September 2026 from primary specs (OpenPBR, glTF, Adobe Substance docs), canonical texts (McDermott, Burley 2012, PBRT, Real-Time Rendering), and current production practice. Software versions move. The physics and the UV rules do not.
