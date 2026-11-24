---
name: blender-clothing
description: Use for Blender game clothing, garments and outfits. Game clothing in Blender 5.3 - garments from body regions (ease offset, thickness), skin-weight transfer, hiding covered body faces, cloth-sim draping for folds, body-morph shape keys on garments, layered clothing, pose-aware poke-through checks, sim-to-game-mesh cleanup.
license: MIT
metadata:
  author: blender-agent-skills
  version: "1.2.0"
---

# Clothing

The execution model is in `blender-core`. The rig and weights are in `blender-character-rigging`, classic cloth settings in `blender-simulation`, and Roblox layered clothing cages in `blender-export-pipelines` (references/roblox.md).

## One call each
```
scripts/garment.py -- --body Body --group Torso --name Shirt --hide-body
scripts/garment.py -- --body Body --zmin 0.45 --zmax 1.05 --name Pants --offset 0.01 --drape 40 --morphs
scripts/garment.py -- --body Body --group Torso --name Armor --offset 0.02 --thickness 0.008
scripts/garment.py -- --check Shirt --body Body          # pose the rig first: checks the current pose
```

## How pros build game clothing
1. **Cut and gravity first.** Extract and offset the body for close-fitting pieces; draft and join panels for loose woven garments. For a gathered sleeve, put fullness in the sleeve panel and gather it into a separate, narrower wristband. Drape the assembled garment on the rest body. Before texturing, compare it in plain gray to the chosen reference from front and side with arms down, then with an elbow bent. Look for cloth falling from its supports and folding at the bend, not an even cushion around the body.
   - **Isolate clothing before changing anatomy.** Hold the same idle pose, lens and light; compare the unmasked body and dressed character from front and side. On the posed authoring mesh, use `BVHTree.find_nearest` in a common coordinate system and sign each distance against the outward body normal: positive is air, negative is penetration. Measure the same named regions before and after, then verify the settled render in-engine; hiding covered skin can conceal clipping without reducing bulk. Let woven cloth bridge the breastbone and gather at the tuck and cuffs rather than chasing zero clearance everywhere.
   - **Check the moving fit, not just clearance.** Compare identical held poses with cloth off/on. For defects present with cloth off, fit against the posed body; keep the garment's original weights when mapping a correction back to rest space. A maximum push that clears every sampled pose can produce pointed shoulders in neutral. Check the silhouette and prefer same-arm skin weights over enlarging the cap. To find compression folds, drape excess panel length on the standing form while bringing cuffs and tuck to their supports. Transfer only the folds that survive motion review; a standing chest crease can become a permanent shelf in other poses.
2. **Game mesh**: retopo the high/sim result to clean, even quads with loops at the joints (elbows, knees, shoulders get 3 loops), then UV it. Bake folds into normal maps (`blender-texture-baking`) so the low mesh stays light.
3. **Skin**: copy weights from the body (same bones, same influences), clean them (limit 4, normalize, drop < 0.001), then blend toward smoother weights only on loose parts (skirts, coats) and add extra bones for those.
4. **Preview coverage reversibly.** Keep the body intact during fitting. Check with unmasked skin, then apply the existing outfit mask for the dressed view; keep skin under openings. Hold mask coverage fixed across revisions: widening it can hide a bad sleeve join without fixing it. Removing covered faces is a separate production optimization, not a fit correction.
5. **Verify fit, then runtime motion.** Run `stress_poses.py` from `blender-character-rigging` and `--check` in each pose. Use Blender to judge cut, clearance and resting drape; judge moving folds in the target engine on the actual character. In a multi-piece outfit, report fit and motion separately for each piece: a better trouser cut does not finish the shirt, and folds that follow bones do not demonstrate fabric settling after a stop. Use a ring-built sleeve only as a diagnostic control for skinning and simulation: sample the upper arm, elbow and wrist and taper into the cuff. Passing that test says nothing about the cut of the finished sleeve or how it joins the shoulder. Keep the two openings intentional, unwrap along one longitudinal seam, and preserve that UV seam and triangle winding across export. For Unreal motion checks, use `ue-physics-and-chaos`.

## garment.py
- Region: `--group` (body vertex group, weight > 0.5) or a world `--zmin/--zmax` band. The faces are copied in world space and pushed out along the vertex normals by `--offset` (ease: 5–10 mm fitted cotton, 10–20 mm jackets, 15–30 mm armor). Then `Solidify` adds `--thickness` (2–4 mm fabric, 5–10 mm leather/armor), growing outward. UVs carry over from the body.
- Weights: Data Transfer (`POLYINTERP_NEAREST`, all groups by name), applied, limited to `--max-influences`, normalized and cleaned. The body's Armature modifier is added and parented to the rig.
- `--hide-body`: `Hidden_<name>` vertex group (the region shrunk by one ring) + an inverted Mask modifier on the body. Swappable outfits toggle masks. For export, delete those faces on a body copy instead.
- `--drape N`: classic cloth (cotton-ish: mass 0.15, tension 15, bending 0.5, quality 8). The top 8% is pinned, the body collides at 3 mm, it settles N frames, then the result is applied. That gives gravity folds on loose areas. Use 30–60 frames.
- `--morphs`: for each body shape key, a same-named garment shape key, via Surface Deform bound in the rest pose and applied as a shape key per morph. Engines drive body and garment keys together (fat/thin, muscle, breathing). Measured: a 30% body morph pokes through 346 verts. With the matching garment key, 0.
- **Poke check** (also run at the end of every build): casts along each body vertex normal. If the garment is hit inward before outward (or sits < 0.5 mm away), it's poke-through. Verts that hit no garment within 0.1 m aren't covered and are ignored. It reports `covered_verts`, `poke_verts` and `poke_max_m`, uses the **evaluated** pose, and respects the body masks.

## Layered clothing
- Order from the skin out: underwear/base → shirt → vest → jacket → armor. Each layer offsets from the one below (run `garment.py --body Shirt` for a vest over the shirt) and copies weights from the body, not from the layer below.
- **Fit a tuck to the waistband, not just the body.** Hold the trousers fixed. Slice the waistband triangles just below the top edge; compare the outer band and shirt at matching angles around the waist. Cloth eased outward but still inside the band stays invisible. Shape selected side or back folds to turn over the rim while keeping the lower shirt inside; preserve the fitted chest and sleeves. Compare front, side and back at rest and on opposite strides. A wavy hemline is not evidence of fabric folding back over the band.
- If layers intersect at rest, check their cut and overlap first. If a clean overlap separates only in motion, compare the two layers’ skin weights and simulation on/off before enlarging either garment.
- Roblox layered clothing needs `<Name>_InnerCage` + `<Name>_OuterCage` built from Roblox's template cages (same vert order/UVs). Other engines (UE Mutable/Unity) blend shape keys or use body-hiding masks. Same principle.

## Gotchas
- `shape_key_add()` creates keys at value 1.0 (measured). Set `.value = 0` when authoring morphs from Python.
- Surface Deform for **morphs only**: if left live together with an Armature modifier, the pose is applied twice. `--morphs` removes it after baking the keys.
- Apply the body's scale before building. Offsets and thickness are in meters.
- Cloth sims need the body's collision mesh at a modest resolution. Decimate a proxy for the collision if the body is > 50k verts.
- Run `blender-asset-qa` on the final garment (z-fighting with the body, floating parts, normals) and check its tris against `blender-game-optimization` budgets.
