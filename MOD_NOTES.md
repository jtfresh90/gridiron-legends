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

---

## Phase 3 — corrected materials: skin reads as skin, head reads as one helmet

Joshua's phone screenshot of the live game (Nov 10) showed the players
looking half-dressed: one arm in solid team colour and the other bare skin,
jersey-coloured flecks on skin, and a stark white blob over the helmet.
Phase 3 diagnosed this from the model data (Blender census of the shipped
character.js FBX, per-slot geometry maps, ray-cast coverage scans, and
UV-footprint sampling of the embedded body texture) and fixed it in
`modPartMat()` only — no model data changed.

### Root cause 1 — Body_2's fabric slots are body regions, not jersey

Phase 1's map assumed every non-Bodymat slot on the body mesh was jersey.
The census says otherwise. Body_2's slots:

| slot | faces | what it actually is |
|---|---|---|
| Bodymat | 22,573 | the skin body (carries the embedded body texture) |
| Fabric-Silk | 700 | the figure's ENTIRE right arm, shoulder to hand |
| Fabric-Linen | 357 | patches on the other arm/shoulder |
| Default | 912 | shin-high socks, both legs, symmetric |

All four slots share the mesh's single UV layer into the one body-skin
atlas (the embedded FuseModel_Body_Diffuse.png — pure skin: face, arms,
torso, legs, a painted briefs band; no clothing painted anywhere).
Sampling the texture over each slot's UV footprint gives mean colours
indistinguishable from Bodymat skin (Silk 0.761/0.536/0.455 vs Bodymat
arms 0.759–0.763/0.532–0.536/0.446–0.448). So the Silk arm and Linen
patches are skin regions the exporter happened to tag with fabric
materials — but their FBX materials carry no texture of their own, and
phase 1 flat-coloured them with the jersey colour: one solid team-colour
arm, coloured flecks on the other.

Fix: for Body_2's Fabric-Silk/Fabric-Linen slots, `modPartMat` now assigns
the body texture (`CH.tex`, the same texture object Bodymat uses) with a
white tint — they render as skin, symmetric with the rest of the body. If
the texture is ever unavailable, the code falls through to the old flat
mapping. `Default` keeps the jersey colour on purpose: symmetric
team-coloured socks are a legitimate kit read and were phase 1's intent.

### Root cause 2 — 'covers' is a helmet cover, not shoulder pads

Phase 1 coloured `covers` #e9e9e9 on the assumption it was shoulder pads.
Geometry says no: in the original source FBX the covers mesh occupies the
helmet's space (Z 4.81–5.91 vs the helmet's 4.88–6.07, same X span, both
parented to the scene root like the helmet), and in the shipped file it
wraps the helmet as a padded shell with a face opening — a Guardian-Cap
style helmet cover. There is NO shoulder-pad mesh anywhere in this model;
the jersey's own yoke forms the shoulder line. The fixed near-white colour
made the head read as a white jumble over the team-coloured shell.

Fix: `covers` now takes `t.helmet`, like the shell — the head reads as one
kitted helmet (81% of the front head silhouette in helmet colour + 15%
visor, up from 48% helmet + 33% stark white).

### Corrected mapping (modPartMat, phase 3)

| mesh / slot | treatment |
|---|---|
| any mesh, Bodymat | body texture, white tint (unchanged) |
| Body_2 Fabric-Silk / Fabric-Linen | **body texture assigned, white tint** (was flat jersey) |
| Body_2 Default | jersey colour (socks — unchanged, deliberate) |
| shirt | jersey colour (unchanged) |
| Body_3_2 (all slots) | pants colour (unchanged) |
| helmet_1 | team helmet colour (unchanged) |
| covers | **team helmet colour** (was fixed #e9e9e9) |
| Loft | dark translucent visor (unchanged) |
| shoes | #17181c (unchanged) |

### Verification (data-level; Joshua's phone remains the visual gate)

- Census of the shipped character.js FBX == census of the source 10 MB
  FBX, slot-for-slot (8 meshes, 41 bones); three.js r128 FBXLoader source
  checked: material names reach the game verbatim and groups follow the
  FBX material stream, so the name-keyed map is sound.
- Post-fix ray-cast coverage (bind pose, front): HEAD helmet 81% / visor
  15% / skin 4%; TORSO jersey 66% with both arms skin; HIPS pants 65%;
  LEGS pants 43% / jersey socks 42%; FEET shoes 100%.
- Unit test: the real extracted `modPartMat` passes 19/19 assertions
  (textured slots incl. the new fabric branch + no-texture fallback,
  covers/helmet/visor/pants/shirt/socks/shoes, home/away/referee).
- All 5 inline script blocks pass a syntax check; the diff vs phase 2 is
  one function (`modPartMat`) — applyProps, the !hasHelmet path, clips,
  numbers and decals are untouched; character.js is byte-identical.

### Honest flags

- A sliver of Bodymat skin still shows at the waist sides between the
  jersey hem and the (narrow) pants mesh — it is inside the Bodymat slot,
  which must stay skin for the face/arms, so no material-level fix exists;
  covering it would need added geometry (e.g. a pants-coloured waistband),
  deliberately not improvised here.
- If Joshua would rather see the helmet cover in a contrast colour than
  the shell colour, it is a one-line change back — the identification
  (cover, not pads) is the part the data proves.

---

## Phase 4 — selectable helmet models in Edit Player

Joshua asked for the football helmet models from his Drive "Models" folder
(phase-3.5 staging: `helmets_staging/`, census in `helmets_staging/census.md`)
to be selectable per player in Edit Player. The four staged GLM modules now
live in `models/` and Edit Player has a **Helmet** option in Equipment,
alongside Visor/Sleeves/Gloves/etc.

### The option

| value | label | module | notes |
|---|---|---|---|
| 0 | Stock | — | the player model's own headgear (phase-3 behaviour) |
| 1 | Bills SpeedFlex | `helmet_bills` | Buffalo Bills SpeedFlex; own livery, never repainted |
| 2 | Midnight Navy | `helmet_football_helmet_2` | dark-navy textured helmet, deep lineman cage; own texture |
| 3 | SpeedFlex Style | `helmet_football_helmet` | untextured import; shell takes the team helmet colour |
| 4 | Ridged Classic | `helmet_helmet` | game-rip ridged dome; shell + ridge strips take the team colour |

Labels are descriptive, not brand claims: the census found no brand marks in
three of the four sources, so they are named for what they look like (navy
colour scheme, SpeedFlex styling, ridged dome); the Bills model carries the
team's actual livery in its geometry colours, so it keeps the team name.

The option uses the editor's existing gear mechanism unchanged: one more
`tg('helm', …)` chip row in `playerModal()`, written by the existing `pgear`
action to **`p.g.helm`**, persisted by the same save path as every other gear
field, and highlighted with the same `(p.g[k]||0)` rule. Generated players
have no `helm` key and every reader treats a missing/out-of-range value as 0,
so all existing players, rosters and saves default to Stock untouched.

### Rendering (one code path)

Everything that draws a player — Edit Player preview, PLAY NOW team
previews, in-game bodies — goes through `mkBody → mkSkinned`, and the choice
is applied there (`helmIdx(p)` → `hIdx`; referees are excluded twice over:
`mkBody` never sends refs to `mkSkinned`, and `mkSkinned` forces Stock when
`p.ref`). When an import is selected **and loaded**:

- the stock headgear meshes (`helmet_1`, `covers`, `Loft`) are hidden;
- the stock-only add-ons are skipped: logo decals are not added (their
  anchors were measured for the stock shell), and the procedural visor is
  never built on this path anyway;
- the imported helmet is attached to the `mixamorigHead` bone (below).

When Stock is selected — or the import hasn't finished loading — the build
is exactly the phase-3 build: nothing hidden, decals as before. The
`!CH.hasHelmet` fallback path is untouched.

### Attachment math (bind-pose exact, per build)

The staged geometry lives in character space (where a skinned mesh renders
in bind pose), not in head-bone-local space, so the instance is parented to
the Head bone with the local matrix

    L = inverse(headWorld) · objWorld · templateRootLocal

computed from the built clone's own matrices (`root.updateMatrixWorld(true)`
in bind pose — no clip has been applied yet, the mixer is created after).
That makes placement exact by construction for every build: `applyProps`
proportion edits to the neck/head are already inside `headWorld`, so they
are compensated automatically, and a larger head (hd) then scales the helmet
with it, the same way the decal group already rides the head. `L` is stored
with `matrixAutoUpdate=false` so nothing recomposes it. In animation only
bone rotations move (fixClip strips Head position/scale tracks), so the
helmet tracks the head rigidly.

**Placement proof** (Blender 4.2.3, shipped `character.js` FBX + the four
staged FBXs in one frame; full table in `phase4_work/proof.py` output):

- stock `helmet_1` bbox: centre (−0.0054, −0.0348, 1.6143), size
  (0.2081, 0.2277, 0.2410)
- all four imports: centre (−0.005, −0.035, 1.614) — **centre delta ≤ 0.4 mm
  per axis** vs the stock shell; width (X) ratio 1.000 for all four
- depth/height ratios vs stock: bills 1.044 / 0.924 · Midnight Navy 1.184 /
  1.041 · SpeedFlex Style 1.096 / 1.102 · Ridged Classic 1.167 / 1.035.
  These are the source models' own proportions (the staging width-matched
  them by design — Midnight Navy's cage really is 18% deeper; that is a
  lineman cage, not a placement error). Placement — centre, width, frame —
  is exact; nothing was non-uniformly rescaled to force the extents.
- `bone.matrix_local · L` composes back to identity with max deviation
  4.9e-08; the three.js formula was additionally verified numerically in
  node over 200 randomized bone chains (non-uniform root scale, head-size
  scale, non-identity template roots): worst deviation < 1e-7.

### Colours

Applied per instance in `helmInstance()`; shell materials are cloned before
recolouring, so the cached template stays pristine and two teams can share
a module safely. `helmShell(slug, objectName)` is the whole rule:

- **bills**: nothing repainted — per-face Bills livery (white shell, blue/red
  zones) renders as authored.
- **football_helmet_2**: nothing repainted — its embedded navy texture is
  the design.
- **football_helmet**: object `shell` → `t.helmet`; cage, straps, screws,
  liner and the smoke visor keep their staged flat colours.
- **helmet**: objects `shell` + `shell_strips` → `t.helmet`; facemask,
  mounts, clips, liner and padding keep their staged flat colours.

### Loading

On demand through the existing `getModel()` (same GLM script-tag mechanism
as the clips): the first build that needs a module starts its load; until
it lands, that player renders with the stock helmet (never bald, never
stalled). Load completion calls the game's own `clearBodies()`, so every
preview/body rebuilds with the import in place. A failed load resets the
guard and the next build retries. All four modules together are ~4.3 MB;
nothing is fetched for players who stay on Stock.

### Verification (phase 4)

- `node --check`: all 5 inline script blocks pass.
- Diff vs phase 3: exactly 6 hunks (helmet section before `modPartMat`;
  selection state, stock-hide and decal guard in `mkSkinned`; the attach
  block; the editor chip row). `original/` untouched; the 17 phase-1–3
  model modules byte-identical.
- Node unit test of the **real extracted** phase-4 code: 35/35 assertions —
  `helmIdx` (missing/junk/out-of-range records → Stock, string coercion),
  `helmShell` truth table (10 cases), the load-on-demand state machine
  against a stubbed `getModel` (single in-flight request, rebuild on
  completion, retry after failure, Stock never loads), editor chip labels
  matching `HELMET_OPTS`, and the attach-matrix identity above.
- Blender bbox proof (above); Pages staging regenerated and the split
  character reassembly re-verified (GLM string identical, decoded sha256
  equal).

### Honest flags

- On the **Low** graphics setting (or with motion-capture players switched
  off in the pause menu) the game draws its simple procedural bodies for
  everyone — the game's own fallback, unchanged since phase 1 — and the
  helmet option has no effect there, exactly like team colours/proportions
  on that path. The option applies wherever the replacement model is drawn.
- Imported helmets get **no team logo decals** (by design above); the Bills
  model doesn't need them (its livery is in the model) and the other three
  were never measured for decal anchors.
- In-browser rendering still can't be verified headlessly; Joshua's phone
  remains the visual gate, especially helmet seating at extreme proportion
  edits (the math compensates bind pose exactly; very large head sizes
  scale the helmet with the head, by design).

---

## Phase 5 — Joshua's phase-4 play-test fix pass

Driven by Joshua's play-test verdicts on the live phase-4 build. Every item
was diagnosed from code/data before any fix; where the cause sits in the
original game logic rather than the mod layer, that is stated with the
evidence. Verification is the node rig in `phase5_work/tests/` (real
extracted Art/V/E/G code, real three r128, real model data):
**47/47 V-module tests, 29/29 stadium tests, sim checks pass**; all 5
script blocks pass `node --check`. `original/` untouched.

### A1. All imported helmets recoloured by the game (Joshua override)

- **Change of direction:** phase 4 preserved the Bills livery and the
  Midnight Navy texture by Joshua's then-decision; he reversed it after
  play-testing. `helmShell` now also returns true for the Bills shell-bearing
  parts (`shell`, `jaw_pieces`, `side_panels`, `inner_shell`, `front_piece`)
  and for all three football_helmet_2 parts (`shell`, `facemask`,
  `cage_frame`). Recolour = the existing phase-4 mechanism: clone the
  material, `map=null`, colour = team helmet colour. So the Bills livery is
  dropped, and the Navy's 4096² BaseColor texture is dropped (per Joshua:
  "the Midnight Navy must be coloured too"). Bills `hardware` and the small
  parts keep their staged colours. football_helmet `shell` and helmet
  `shell`/`shell_strips` were already team-coloured in phase 4.
- **Verification:** per-helmet material assertions through the real
  `mkBody`/`helmInstance` flow (colour == team helmet colour, map null;
  hardware/facemask controls unchanged).

### A2. Logo decals on all helmets, bent flat onto each shell

- **Root cause of "logos float":** phase 4 put flat `CircleGeometry` decals
  only on the stock shell, at ±0.127 in the head-bone group — a plane tangent
  to a curved shell; its rim floated up to ~0.012–0.03 (game units) off the
  surface, and imports got no decals at all (phase-4 honest flag).
- **Fix:** `decalPatch()` builds a disc (16 segments × 3 rings) bent around
  the shell's measured centre of curvature, with a vertical taper term for
  the dome rolling over the top. Parameters were **measured per shell** by
  raycasting the actual geometry from the head centre in Blender (first-hit
  surface distances at the decal band, circle fits with residuals
  ≤ 0.0055 game units): stock (hg frame) centre (±0.0116, −0.0389), R 0.1221;
  per-import centres/radii/anchors are tabled in `HELM_DECAL`
  (template-local units). Clearance 0.004–0.006 so nothing sinks.
  Orientation (which way the logo's right edge points) matches the phase-4
  flat decals on both sides.
- **Verification:** patches present (2 per helmet) for stock + all four
  imports through the real build flow; patch-to-shell median gap measured
  in the harness: bills 0.0104/0.0124, navy 0.0074/0.0075, SpeedFlex Style
  0.0064, Ridged Classic 0.0065 (game units, vs the flat decal's float).
  The non-mocap fallback path keeps the original flat decal, unchanged.

### A3. SpeedFlex Style "hole through the helmet with the head"

- **Diagnosis (Blender, staged FBX vs the shipped character):** the shell
  is nearly closed — head-containment raycasts: 6,424/6,463 head samples
  covered; the only open band is the face opening (bottom-front,
  z < ~1.50), which is the design. No head vertex pokes through the shell
  on ANY of the five shells (corrected raycast test; pokes found were
  shoulder vertices below the rim line). The visible see-through slot is
  the model's **`visor` object**: measured bbox y −0.118..−0.022,
  z 1.646..1.744 (character space) — it sits in the BROW band above the
  face opening, not over the eyes, and staging had recoloured it
  transparent smoke (alpha 0.6) on a guess (census flag (a)). Through it,
  the forehead shows — reading exactly as "a hole going through it with
  the head".
- **Fix:** in `helmInstance`, football_helmet's `visor` is forced opaque
  (`transparent=false, opacity=1`, dark #23262b) — a brow panel, which is
  what the geometry is. Shell geometry and placement untouched; Ridged
  Classic untouched, as instructed.
- **Verification:** material assertions in the rig (transparent false,
  opacity 1); placement code paths unchanged for all helmets.

### B4. Proportions (limbs) not applying — ALL players, not just the QB

Joshua's correction stood: the bug hits every player he edits, so phase
2's approach was audited end-to-end instead of patched at the QB.

- **Audit (real game paths, not the phase-2 stub rig):** the entire V
  module body was executed in node with the real libraries and data, and
  every link was measured: (1) the editor writes `p.b` correctly (slider
  handler, preset buttons, Save & Play persistence — `pack()` is a generic
  JSON transform, `p.b` survives); (2) `people()`/`sync()`/`preview()` all
  route through the same `mkBody → mkSkinned`; the body cache is keyed by
  entity and rebuilt on every `V.build`, and the editor preview cache key
  includes `p.b`; (3) `Art.dims` factors reach `applyProps`; (4) the
  shipped character FBX parses to **328 bones — eight nested same-name
  copies of each of the 41 bones**; the real rest offsets live only on the
  layer-0 copies, which are exactly the `getObjectByName` first matches
  `applyProps` edits, and each skinned mesh's skeleton uses a different
  zero-offset layer, so scaling layer-0 offsets moves every layer;
  (5) the mixer does not stomp rest offsets (positions bit-identical
  before/after 10 updates), and skinned vertices demonstrably move
  (0.36+ units) when edited.
- **Honest result:** in a faithful reproduction the proportions **do**
  apply — edited/preview/live segment ratios match the phase-2 design
  values (thigh 1.184 predicted, 1.168 measured mid-pose; the uniform
  0.986 factor is the head-size factor entering the height
  renormalisation). No code-level loss exists anywhere in the chain that
  could be reproduced headlessly; the on-device neutralisation could not
  be reproduced in this environment (no GPU/browser — the standing
  headless limit), so no speculative rewrite was shipped.
- **Fix (hardening, per the mandate):** the application point itself was
  made immune to the whole class of one-time loss. `mkSkinned` now
  captures the mapped bones' REST offsets before `applyProps` (only when
  some factor ≠ 1), stores them on the body (`B.prop`), and `animate()`
  **re-asserts rest × factor after every mixer update**, plus the
  head/hand scales. If anything anywhere (clone timing, bind state, mixer
  internals, a future three version) resets the bones, the next frame
  restores the edit. Default builds keep `B.prop === null` and the exact
  phase-4 path (zero per-frame cost).
- **Verification:** real-flow ratio tests (pre-snap and live) **plus** a
  stomp test: after building an edited body, all mapped bone positions
  and the head scale are forcibly reset to the unedited build's, then 10
  `pose` frames run — the segment ratios return to the design values.
  Joshua's phone remains the acceptance gate for the on-device behaviour.

### B5. Jersey numbers flush on this model

- **Measurement:** in the real `mkSkinned` flow (bind pose), nearest-
  surface distances from the number arcs to the chest (shirt + Body_2
  skinned vertices, torso band): front median 0.0098 (centre 0.0060, edges
  0.0145 — arc too curved and floating), back median 0.0232, p95 0.0437
  (arc too flat and far out; centre floated 0.0431). The arcs were stock
  placements fitted to the mocap suit's chest; this model's chest differs.
  (The skinned path has no nameplate — that exists only on the procedural
  fallback body, which keeps its stock fit.)
- **Fix:** per-mesh scale + offset inside the number group (shared
  geometry untouched, so the procedural body is unaffected): front
  scale 0.90, offset (0.007, 0, −0.035); back scale 0.91, offset
  (0.021, 0, −0.035) — offsets in number-group units.
- **Verification:** refit harness numbers — front min/median/p95
  0.0027/0.0060/0.0096, back 0.0034/0.0064/0.0144 (nothing sunk anywhere;
  back digits at 91% width, the cost of matching the back's curvature —
  the 0.70-scale alternative fit marginally better but visibly squeezed
  the digits, and was rejected).

### C6. DBs falling before the snap

- **Root cause (original clip content + selection logic, measured):**
  `animate()` hard-wires pre-snap CB/S/LB to the `def` clip regardless of
  velocity. The `def` clip (defender.js) is a falling-down performance:
  sampled on this skeleton, hips height 96 → 14 (FBX units) and spine
  pitch → 94°, ending flat on the ground at ~1.8 s of a 3.6 s clip that
  then LOOPS — stand, collapse, lie, pop up, repeat. (The falling_down/
  getting_up clips are indeed unbound on this model, but they were never
  the pre-snap mechanism; the game doesn't use them here. The `stance`
  clip does pitch to 107° but its hips stay at 69 — a plausible
  three-point stance — left alone. The `catch` clip also ends falling;
  noted, out of scope.)
- **Fix:** after `fixClip`, the `def` clip is trimmed to its first 1.0 s
  (the upright ready crouch: hips 96 → ~80, pitch ≤ ~40°), and
  `mkSkinned` sets the `def` action to `LoopPingPong` so the loop has no
  pop. Live-play uses of `def` (coverage, engagement) get the same
  trimmed clip — strictly better there too.
- **Verification:** clip duration == 1.0; sampled hips minimum over two
  full ping-pong cycles > 55 FBX units (never near the ground); action
  loop mode asserted in the rig.

### C7. Tackling looks weird

- **Diagnosis (measured):** the tackle clip's drive/wrap phase is
  0–~1.2 s of its 3.6 s (spine pitch 39° → 67°); by 50% the tackler is
  already horizontal (pitch 84°). `animate()` started the clip at
  `SKIP.tackle = .5` — i.e. tackles began mid-fall, already flat: the
  "instant ragdoll" look. (`SKIP.fall = .4` was checked and is correct:
  the fall clip's hips spike upward to 385 until ~34%.) Separately, the
  sim ends a successful tackle instantly (carrier down + 1.4 s stun)
  while the clip needs ~1.2 s to arrive — a contact-timing mismatch in
  the original game logic that the clip retargeting residual (Nov 10:
  tackle/falling_down/getting_up need a true retargeting pass) also
  feeds; that part is **not** fixed here and is stated plainly.
- **Fix:** `SKIP.tackle` .5 → .08 (starts at the drive). One constant.
- **Verification:** sampled spine pitch at the new start = 23° (vs 77° at
  the old start) in the rig.

### C8. Blocking "terrible and not working"

- **Investigation (headless sim of the real E+G code, diff=1 — an early
  run without `diff` produced NaN mover speeds and was discarded; the
  real UI always sets `diff`):** the blocking SYSTEM works as designed —
  `assignBlocks` matches all 4 DL with a spare blocker left over on
  20/20 sampled plays, engagements happen on every play, engaged rushers
  are slowed to ×0.06 speed. The sim never references the player model,
  so this is **pre-existing game logic, independent of the model swap**.
  What fails the user is the pocket clock: the shed formula's base rate
  .075 (with a shallow .011 slope) frees each rusher in ~13 s *regardless
  of the ratings matchup*, so with 4 rushers the first man is free in
  ~2.8 s and a statue QB was sacked 20/20 at a median ~2.5 s — downfield
  routes (3–4 s) are unthrowable, which reads as "blocking not working".
- **Fix (one formula, contained):** shed rate base .075 → .06, slope
  .011 → .002, clamp [.03,.5] → [.025,.4]. Even matchup ≈ 17 s per
  rusher (first shed ~4 s), dominant OL ~33 s, badly overmatched ~11 s;
  run-play ×1.25 and double-team ÷2.5 multipliers unchanged.
- **Verification:** statue-QB sims across three roster draws: median
  sack 5.73 / 3.23 / 3.47 s (draw-dependent because blitz frequency
  varies; p75 5.5–8.9 s vs ~3.7 s pre-fix), with 40–50% of pockets
  holding ≥ 4.5 s. Unblocked blitzers still arrive in ~1.7 s — that is
  the game's defensive design (throw hot or keep a back in), not the
  shed clock. Full CPU-vs-CPU games complete with sane scores
  (27–20, 20–41, 23–27).

### D9. Weird poles in the stands (all stadium kinds)

- **Root cause (phase-2 code, in the COMMON construction):** `stand()` —
  used for every side, corner and cheek in every variant — draws its
  aisle slats as `BoxGeometry(.9, .08, sl)`: the long axis (slope length
  `sl`) on Z. But the slat is rotated like the deck plane
  (−(π/2 − ang) about X), which maps the box's long axis onto the slope
  **normal** — so each aisle stands straight up out of the crowd slope,
  up to `sl`/2 ≈ 18 units on big decks, every 15 units of stand length.
  The crowd code even skips people at those columns (they were meant to
  be flat aisles). Floodlight pylons and jumbotron posts were checked
  against every footprint including upper decks/corners — all outside
  the stands; not the cause.
- **Fix:** geometry swapped to `BoxGeometry(.9, sl, .08)` — same centre,
  same rotation; the long axis now lies along the slope.
- **Verification:** scene census on real `build()` output (sizes 1/2/4):
  26–66 aisle slats present per stadium, zero pole-oriented.

### D10. Dome doesn't wrap at larger sizes

- **Root cause (measured against the real `build()`):** the dome was a
  hemisphere sized from the FIELD (`max(L,W)*1.35`, height ×0.46) centred
  at −L/2. At NFL size 3 its height over the stand corners (≈30.3) was
  below the stand tops (≈32.2) — poke-through; at size 4 the footprint
  corners fell outside the dome circle entirely. An ellipsoid fitted to
  the footprint sides still leaves the corners at q=1.36 (measured), and
  the freestanding video board stood at the rim with its top at
  topY+13.75, outside any sane roof.
- **Fix:** the dome is now an **extrusion of the footprint itself** — a
  rounded-rectangle curtain wall on the phase-2 footprint extents
  (+10 margin) rising to H0 = topY+13 (above every stand wall, press box
  and the light ring at topY+12), with a bevelled roof plateau at
  RY = topY+20. It wraps by construction at every size. In dome builds
  the video board becomes a centre-hung, double-sided board at midfield
  (as in real domed stadiums) on hangers to a crossbar, replacing the
  rim board + ground posts; open-air builds keep the north board
  unchanged.
- **Verification:** sweep of size 1–4 × pro/college/cfl through the real
  `build()`: every mesh's topmost vertices sit under the computed roof
  surface with ≥ 2.5 units of clearance everywhere (worst margin
  −2.55). Sizes 1–2 behaviour preserved (they contained before, they
  contain now).

### E11. Performance ("gameplay slow", "animations slow the game")

Targeted changes, no visual or gameplay changes except the sanctioned
pixel-ratio cap:

- **Pixel ratio cap 2 → 1.5** (init + resize): fragment cost scales with
  the square — (1.5/2)² = 56% of the pixels on hi-dpi phones. This is
  the one rendering-resolution change, flagged explicitly.
- **Far-body mixer LOD:** bodies more than 60 (Manhattan) units from the
  ball now update their animation mixer at 30 Hz with accumulated dt
  instead of every frame; root position/rotation still update every
  frame, so movement is unaffected — only distant skeletal poses tick
  at half rate. Near bodies untouched.
- **Paused dt=0 skip:** a dt of 0 (pause menu, first frame) no longer
  runs every body's mixer for a guaranteed no-op.
- Not changed (structural, reported honestly): ~90k triangles per
  skinned player × ~25 bodies × shadow pass is the floor cost of the
  phase-1 model; reducing it means re-meshing, which is out of scope.
  Expected gain mechanism: roughly half the mixer CPU for far players,
  ~44% fewer fragment pixels, zero mixer cost while paused. No FPS
  number is claimed — Joshua's phone is the measuring instrument.

### F. Remaining player properties & equipment (Edit Player audit)

Audited the editor against what the replacement model actually has
(`p.g` options vs the model's meshes/material slots):

- **Wired — Visor (g.v) → the model's Loft:** phase 4 rendered the Loft
  (the model's visor) for everyone on the stock path, so Visor=None was
  a dead option on this model. It now hides/shows the Loft. Behaviour
  note: players whose record says no visor no longer display one —
  generated rosters roll visor ownership at 25%, and those players now
  render as generated (the game's own intent).
- **Wired — Cleats (g.cl) → shoes colour:** 0 white, 1 black (phase-4
  default), 2 team primary. The option previously changed nothing on
  this model.
- **Wired — Socks (g.so) → Body_2 'Default' slot** (the body atlas's
  lower-leg band): 0 skin (body texture), 1 team colour (phase-4
  default), 2 white.
- **Not added, with reasons:** sleeves — Body_2's fabric slots are
  asymmetric (Fabric-Silk = the entire right arm, Fabric-Linen =
  patches on the left), so a sleeve colour would give one full sleeve
  and one partial; gloves — the hands live inside the shared Bodymat
  slot with the face/torso skin, no hand-only slot exists; wristbands,
  towel, back plate, hair — no such geometry or material slot exists
  in the model. Adding any of these needs new modelled parts, not
  editor wiring.
- Already editable and working (no change): helmet (phase 4 + A1/A2),
  jersey/pants colours, skin tone, numbers, proportions (B4).

### Process note

Midway through this pass the VM restarted, wiping the /dev/shm working
copy and the phase-2 test rig. Everything was rebuilt — rig first,
calibrated against the phase-2 verified ratios on the pristine file
(1.168/0.949/1.022 reproduced exactly) — and the edits re-applied; the
final measurements are bit-identical to the pre-restart run. The tests
and rig now live in `phase5_work/` so they persist.
