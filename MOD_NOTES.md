# Gridiron Legends — player-model mod notes

The on-field players now use the rigged American football player model (helmet,
shoulder pads, jersey, pants) instead of the mocap-suit figure. Team colours,
jersey numbers and helmet logo decals still work. Everything outside the
mocap-character code path is byte-identical to the original game.

## The new character (`models/character.js`)

Built from `american-football-player_rigged_10mb.fbx` (41-bone Mixamo-rigged
model, 8 skinned meshes). The model's Hips sat at Y = 259.2286 in FBX units;
the game is built around Hips Y = 104.2749, so an additional uniform bake
scale **S = 104.2749 / 259.2286 = 0.4022508** was applied in Blender 4.2.3
using the snapshot bake (every bone's head+tail captured in armature space
first, then scaled; each mesh's vertex data and parent-space translation
scaled by the same S; pose zeroed). The scale is baked into the geometry —
the FBX was exported with `global_scale = 1.0`, not exporter scaling.

Verified on the exported FBX (raw binary probe + fresh Blender import):

- Hips Lcl Translation Y = **104.27489** (target 104.2749 ± 0.5)
- **41 bones**, colon-form `mixamorig:` names, **8 meshes, all skinned**
  (Body_2, Body_3_2, Eyes, shirt, shoes, helmet_1, covers, Loft)
- **Zero animations** embedded (the game supplies all clips)
- Feet rest at min-Y ≈ −1.0 — that is the source model's own stance
  (−2.51 units before scaling); at game scale it is ~0.5 % of a player's
  height, so no extra offset was added.
- Helmet is coherent with the head in bind pose (centroid ≈ Head bone +
  (−1.9, +7.8, +3.5); shell top ≈ 24 units above the Head bone origin).

### Textures

All texture references were stripped from every material **except**
`Bodymat`, which is wired to the real `FuseModel_Body_Diffuse.png`
(1024×1024) and embedded in the FBX — the probe confirms exactly **one**
embedded media item (the PNG, 1,109,604 bytes). No Normal/Specular/Gloss/
Opacity/PSD references remain, so nothing tries to load external files.
The game's suit-texture surgery (`teamTex`) never touches this model
(see below); `CH.tex` becomes the body diffuse and is only used by the
legacy path.

## The throw clip (`models/quarterback_pass.js`)

`Quarterback Pass.fbx` from Drive turned out to be **byte-identical to the
FBX inside the original `character.js`** (md5 `8bfd256a4241e8bf9a1c8325f1b02a61`,
2,416,624 bytes) — i.e. it is the mocap character whose own longest embedded
animation the game always used for `CH.raw.throw`. It is now wrapped as a
normal clip module and loaded through CLIPS as `['throw','quarterback_pass','m']`
(family `m`, hip scale `k = CH.hipY/104.275` — exactly how the old embedded
throw was scaled). The line that harvested the character's own animation was
removed so the two can never clash; the loading status now reads "of 17".

## Clip compatibility (all 16 clip modules)

Each clip's longest action was imported and its track node names compared
against the new character's 41-bone set. The new skeleton lacks the
Middle/Ring/Pinky finger chains of the clip rigs; three.js would ignore
those tracks anyway, and `fixClip` now drops them explicitly (below).
**Every other track in every clip targets a bone the new model has.**

| clip module | rig bones | track nodes | covered | dropped (fingers only) |
|---|---|---|---|---|
| idle, running, jogging, running_backward, strafe_left, strafe_right, tackle, falling_down, getting_up, kick, celebration | 57 | 57 | 41 | 16 |
| football_catch, football_stance, football_hike, defender, quarterback_pass | 65 | 52 | 34 | 18 |

The 65-bone family keys only 52 bones even on the original rig; on the new
model the 7 unkeyed bones are end bones (HeadTop_End, Toe_Ends, finger tips)
which simply follow their parents — same as before.

## index.html changes (only the CH / CLIPS / loadChar / fixClip / mkSkinned region)

1. **CLIPS** gains `['throw','quarterback_pass','m']`; status string
   "of 16" → "of 17".
2. **loadChar** — after the character loads, it now records
   `CH.hasHelmet` (true when any mesh name matches /helmet/i — true for the
   new model) and builds `CH.boneSet`, the set of the character's object
   names with `:` removed (the same normalisation the game already relies
   on, e.g. `mixamorigHips`).
3. **fixClip** — additionally drops any track whose node is not in
   `CH.boneSet` (when the set exists). All other behaviour identical; with
   no boneSet it behaves exactly as the original.
4. **mkSkinned** — when `CH.hasHelmet` is true:
   - `teamTex()` is not run for the player (the suit pixel surgery must
     never touch the new model's texture).
   - Materials are assigned per part by the new `modPartMat()` helper
     (table below) instead of the suit-texture swap.
   - The procedural helmet (`helmetParts`) and the add-on visor are **not**
     built — the model has its own helmet and visor (`Loft`). The head
     group `hg` is still created, and the **team logo decals are kept**,
     pulled in from x = ±0.152 to **±0.127** (measured: the real shell's
     lateral half-extent at decal height is 11.48 units vs the procedural
     shell's 13.8, so 0.127 keeps the stock ~0.4-unit clearance).
   - When `CH.hasHelmet` is false (character ever reverted), the original
     code path runs unchanged — procedural helmet, visor, teamTex and all.

### Per-part colour map (new model)

| mesh / material slot | colour |
|---|---|
| helmet_1 (Mat.3) | `t.helmet` |
| Loft (frosted glass) — the visor | #101318, transparent, opacity 0.85 |
| shirt (shirt_0) | jersey — home: `t.c1`, away: `t.away` |
| Body_2 — Fabric-Linen, Fabric-Silk, Default slots | jersey |
| Body_2 / Body_3_2 / Eyes — Bodymat slots | embedded body diffuse kept, colour white |
| Body_3_2 — all other slots (pants, Mat.2, Mat.2_2mat) | `t.pants` |
| covers (shoulder pads, all slots) | #e9e9e9 |
| shoes | #17181c |
| referees (any of the above jersey/pants parts) | #111111 |

Notes on slots: on the real model, `covers` uses only its first material
slot (Plastic Matte) and Body_3_2's `Body_3.2` slot has no faces — colouring
is applied per slot anyway, so unused slots are harmless. Body_3_2 is the
pants garment (its `pants` slot is 5,370 of its ~6,100 faces; the Mat.2
slots are waist trim), so the whole piece takes the pants colour.

### Jersey numbers — decision: stock placement kept

Measured in Blender on both models, in the number cylinder's actual
vertical window (Spine2 ±, centre chest only): the chest front surface sits
**15.49 units** from the Spine2 origin on the mocap suit and **15.44 units**
on the new model (the shirt — the pads sit higher, at the neckline, outside
the number window). Since the two agree to 0.05 units, the stock
`ng` constants — position (0,1,0), scale (YD·0.8, YD·0.8, YD·0.6) — were
left exactly as shipped.

## Swapping in the full-fidelity character later

`american-football-player_rigged.fbx` (20,927,644 B, full-density helmet and
pads) can replace the 10 MB version: bake the same S = 0.4022508 with the
same snapshot method, strip textures the same way (Bodymat keeps the
embedded diffuse), export FBX Binary with global_scale 1.0 and no leaf
bones, then re-wrap the result as `models/character.js` in the same
`(window.GLM=window.GLM||{})["character"]="<base64>";` format. No code
changes are needed — detection is by the model's own mesh names.

## Verification done

- Raw FBX probe of the new character: Hips, bone/mesh/skin counts, zero
  animation stacks, exactly one embedded texture (above).
- Fresh Blender import: 41 bones, 8 skinned meshes, bind-pose coherence,
  chest/helmet measurements (above).
- All 17 `models/*.js` decode to Kaydara FBX binaries.
- `node --check` passes on all 5 inline script blocks of the patched
  index.html; the two new/changed functions (`fixClip` filter,
  `modPartMat`) were extracted from the patched file and unit-tested in
  node (track filtering incl. legacy no-boneSet path, hips rescale, all
  colour-map cases, home/away/ref variants).
- Unified diff vs the original index.html: changes confined to the
  character region (original lines 1061–1090); everything else
  byte-identical. `original/` checksums unchanged.

Not verifiable headlessly: actual in-browser rendering (three.js r128
FBXLoader in the page). The FBX follows the same exporter conventions as
the files the game already loads, and all data-level checks pass; a
30-second open of the game on a phone is the final visual check.

---

# PHASE 2 — rounded stadium corners, exterior environment, body proportions (Nov 10 2026)

Built on top of phase 1. All phase-2 changes are confined to 5 hunks in
index.html (61 changed lines vs the phase-1 file): applyProps + its call in
mkSkinned, cornerStand, the corner block in build(), and the surroundings
block in build(). Everything else — including the !CH.hasHelmet path and
all 17 model modules — is byte-identical to phase 1.

## A1. Stadium corners — what was wrong

The four straight stands never actually met: the side stands stop A-1 = 6
units short of the end front-lines (A = Art.APRON = 7) and the end stands
stop 3 units short of the side front-lines (ex - lenE/2 = 3 identically).
The old code papered over each corner with a single diagonal wedge stand
(len dep*1.55, depth dep*.9) that neither reached the straights nor
followed their profile — leaving notched, squared-off corners with visible
gaps (clear in Joshua's reference photo/video: stands end in black corner
gaps). Upper decks had no corner treatment at all; they simply ended.

## A2. What was built

For each corner (sx, sz) — skipped for the 'sides' shape, and for the open
end of 'horse':

- **Cheek stands** extend both straights the rest of the way to the corner
  point O = (±ex, fz), using the existing stand() unchanged: a side cheek
  (len = the 6-unit gap, depth dep, y0 1.7, team-colour front wall, crowd
  max scaled by len/lenS) and an end cheek (len = the 3-unit gap). The
  front walls of the two cheeks meet exactly at O, so the field-side corner
  is closed.
- **cornerStand()** (new function) wraps the corner in a curved sector:
  a quarter-cone (CylinderGeometry sector, theta 3PI/4..5PI/4 in a local
  frame whose -z is the outward bisector, radius 0->dep, 16 segments)
  centred on O. Its height profile, y0 + h*rho/dep with h = dep*.55, is
  exactly the straights' slope profile, so the cone meets both cheeks
  seamlessly along the shared radial edges — same slope colour #3a3f48,
  DoubleSide. Behind it a curved back-wall sector (#2b3038, radius
  dep+.6, same 2.6 height as the straights' back boxes). Crowd is placed
  by the existing people() in polar rows with the straights' palette and
  spacing (spc = max(1.02, sqrt(dep*lenS/cap))), so corner crowd density
  matches the straights and bobs with them.
- **Upper corners** mirror this when uppers exist: upper cheeks extend
  each upper straight to the upper corner point O2 (offset dep+1.5 back,
  depth d2, y0 y2, same under value as the upper straights), and a second
  cornerStand (dep d2) wraps O2 — but only when BOTH adjacent uppers are
  enabled (UP(side) && UP(end)); each cheek is built whenever its own
  upper exists. The upper sector also gets a dark under-curtain sector
  (#191c22) approximating the upper straights' under boxes.

Verified numerically for all 32 ruleset x size x shape x end combinations:
every cheek length is positive (lowers 6.00/3.00 by construction; uppers
10.5-40.5), so no degenerate stands are ever created.

## A3. Exterior environment

Added in build() immediately AFTER the stadGlb hide-block, so it is never
hidden when an imported GLB stadium is used, and only when the stadium is
open-air (!dome — a dome already encloses the scene):

- **Concrete apron**: a plane (#3a3e45, y=-0.015) extending 7 units past
  the stadium's full outer footprint (stands + upper decks + floodlight
  pylons + jumbotron posts, all computed from the same ex/dep/d2 vars).
- **Asphalt ring**: a larger plane (#22242a, y=-0.04) extending ~38 units
  beyond that — parking-lot tone. Both sit between the field plane (y=0)
  and the existing 1400x1400 ground (y=-0.06), which still forms the far
  field, so nothing floats and there are no z-fighting overlaps at the
  game's camera distances.
- **Surroundings ring**: 110 instanced cones (trees, 8-16 tall, four dark
  greens via instanceColor, the same mechanism people() uses) and 30
  instanced boxes (low buildings, #2f3846, 10-44 tall, random yaw) placed
  on an elliptical ring 60-285 units beyond the apron, centred on the
  field centre, with a rectangle rejection push-out so nothing spawns
  inside the apron. All MeshLambertMaterial, so the existing day/dusk/
  night lighting and fog (220-900) apply automatically — trees/buildings
  read as dark silhouettes at night, hazed shapes by day. Camera far
  plane is 1400, so everything is in range, including in the stadium
  editor preview (stadPreview, R=175), which shows the surroundings
  around the bowl.

Cost: 2 planes + 2 instanced meshes, no textures, no new assets.

## B. Body proportions now drive the skinned model

The bug: the editor's per-part build multipliers (player p.b, via
Art.dims — the D object: to/ua/fa/ha/th/ca/ft/nk/hd/ws/raw) were only
consumed by the procedural fallback bodies (mkBody). mkSkinned applied
only the uniform height scale ht/CH.h, so on the animated replacement
model every slider was invisible.

Fix — applyProps(obj, D), called in mkSkinned's CH.hasHelmet branch only,
on the fresh SkeletonUtils clone, before the AnimationMixer is created.
Length params scale the child bone's rest offset from its parent (the
segment length); size params set the bone's own scale:

| bone(s) (mixamorig…)        | param | how applied              |
|-----------------------------|-------|--------------------------|
| Spine, Spine1, Spine2, Neck | to    | offset x to (torso)      |
| Head                        | nk    | offset x nk (neck)       |
| Left/Right ForeArm          | ua    | offset x ua (upper arm)  |
| Left/Right Hand             | fa    | offset x fa (forearm)    |
| Left/Right Leg              | th    | offset x th (thigh)      |
| Left/Right Foot             | ca    | offset x ca (calf)       |
| Left/Right ToeBase          | ft    | offset x ft (foot)       |
| Head                        | hd    | bone scale = hd          |
| Left/Right Hand             | ha    | bone scale = ha          |

Root scale in the hasHelmet branch becomes
sc/(D.raw/PROP_RAW0) on Y, times D.ws on X/Z, where PROP_RAW0 = 1.927 is
dims' raw total at the default build. That mirrors mkBody exactly (its k
divides by the player's own raw, and it applies ws to width): a player
with longer legs stands the same chosen height with proportionally
longer legs, and the weight slider now widens the model. At the default
build every factor is 1, D.raw = PROP_RAW0 and D.ws = 1, so the result
is bit-for-bit the phase-1 transform. The !hasHelmet (mocap suit) path
keeps phase 1's plain setScalar(sc) untouched.

Safety: fixClip strips every position track except Hips and every scale
track, so bone rest offsets/scales are never animated (rotations only) —
the edits persist through all clips. Leg length is applied at the Leg
bones, never by moving Hips, whose position track IS animated. Shoulder
and UpLeg offsets are deliberately not scaled (they set shoulder/hip
width, not segment lengths). Head scale also carries the helmet decal
group (child of Head) outward with a larger head; the jersey-number
group rides the spine chain via offsets, as before.

The editor's live 3D preview goes through the same mkBody -> mkSkinned
path (and its cache key already includes p.b), so preview and field now
show the same proportions.

## Verification (phase 2)

- node --check: all 5 inline script blocks pass.
- Node unit test of the REAL extracted applyProps + dims code against a
  synthetic 41-bone-style stub rig: 26/26 assertions pass — PROP_RAW0
  equals dims({}).raw (1.927); default build leaves every offset and
  scale identical; each factor moves/scales exactly its mapped bones
  (th=1.6 -> both Leg offsets x1.6, Hips never moved; to=1.3 -> Spine/
  Spine1/Spine2/Neck x1.3, Head untouched; hd -> Head scale only;
  ua/fa/ha/ca/ft likewise); thigh+calf chain sums scale as intended.
- Corner geometry swept numerically over all rulesets/sizes/shapes
  (see A2): all cheek lengths positive.
- Diffs: phase1->phase2 index.html = 5 hunks / 61 lines (listed above);
  original/ checksums unchanged (index.html 6b751b21…, character.js
  96f8c123…); all 17 modded model modules byte-identical to phase 1.

## Honest flags

- In-browser rendering still cannot be verified headlessly (same caveat
  as phase 1): the corner/environment/proportion code is plain three.js
  r128 geometry of the same kinds the game already builds, and all
  data-level checks pass, but Joshua's on-phone look is the final gate —
  especially corner seams at grazing angles and the tree/building ring
  density.
- Weight (ws) on the skinned model is new behaviour Joshua didn't have
  before on ANY animated model (phase 1 ignored wt entirely there); it
  is included because mkBody's own semantics apply ws to width and the
  hard requirement was default-build identity, which holds. If he wants
  height-only on the skinned path, dropping D.ws is a one-token change.
- Corner sectors skip the straights' aisle strips and front rail (small
  details; the sectors read as continuous seating).
- Proportions use the game's own stick-figure fractions (dims) for
  height renormalization; the rig's true segment fractions differ
  slightly, so total height after extreme edits can drift a few percent
  from the nominal ht — the per-part changes themselves are exact.
