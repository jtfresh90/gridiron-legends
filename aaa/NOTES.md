# AAA / WebGPU Gridiron — Phase 1 NOTES

Staged showcase proving the new renderer + first detailed player model.
**Stage only — do NOT push.** Parent sequences pushes.

Staging dir: `~/workspace/gridiron-legends/aaa-phase1/`
Page: `index.html` · Model: `models/player_aaa.glb` (+ `.blend` source)

## What Phase 1 is

A single detailed football player on a field, under three.js **r187 WebGPURenderer**
with automatic WebGL 2 fallback, playing the existing idle / run / throw clips.
Two kits (Broncos #10 white/orange, Washington #16 burgundy/gold) as a runtime
material/texture swap on the same mesh. Orbit controls, High/Medium quality
toggle, on-screen backend indicator (WebGPU green / WebGL 2 amber / none red).

The current game, test4, test4-diag, live root and the Claude build-#5 job were
NOT touched — this is a parallel lane.

## Renderer setup

- three.js **r187** (Oct 30, 2026), installed via npm into `work/node_modules/three`
  and **vendored** into `vendor/` (three.webgpu.js, three.core.js, three.tsl.js +
  addons: FBXLoader, GLTFLoader, OrbitControls, RoomEnvironment, BufferGeometryUtils,
  SkeletonUtils, fflate.module.js). The page uses an **import map**
  (`"three" → ./vendor/three.webgpu.js`, `"three/addons/" → ./vendor/addons/`),
  so it runs fully offline — no CDN dependency.
- `new WebGPURenderer({antialias:true})` + `await init()` — r187 auto-falls-back
  to WebGL 2 when WebGPU is unavailable. The page reads
  `renderer.backend.isWebGPUBackend` for the badge.
- Field: 110×49 plane with a canvas texture (mowing stripes, yard lines, numbers,
  navy GRIDIRON end zones). Sky: gradient canvas on a BackSide sphere + fog.
  Lights: directional sun (2048 shadows, PCFSoft) + hemisphere + RoomEnvironment
  via PMREM (environmentIntensity 0.35, in try/catch).
- Model is kept in **native FBX units (~4.2 tall)** so clip position tracks match
  the skeleton exactly (as in the current game); the page scales the root group
  ×0.45 for display. Animation scales consistently with the group.
- Clips: `clips/idle.fbx` (1,027,980 B), `running.fbx` (458,620 B),
  `quarterback_pass.fbx` (2,416,624 B) — extracted from the test4 clip modules,
  **not modified**. Crossfade 0.25 s; football visible for idle/throw, hidden for run.

## Model build method

Base: `~/workspace/gridiron-legends/player-restore/american-football-player_restored.fbx`
(41-bone Mixamo rig, colon-form names). The build is **scripted Blender Python**
(`work/build_model.py`, Blender 4.2.3), NOT a from-scratch sculpt — the rig, body,
jersey, pants, shoes and skin are the restored model; the detailed parts are added
on top and parented to the bones. This keeps the 41-bone rig exact so the existing
clips drive the model with no retargeting.

### Added parts (all bone-parented with the operator method — see lesson 1)
- **Helmet shell** — NEW smooth ellipsoid (UV sphere 48×32, Solidify 0.008 inward,
  smooth shading), navy Principled clearcoat. The restored `helmet_1` was a faceted
  stylised shell with a pointed crown and is **deleted** (as is the Guardian-cap
  `covers`). Face opening cut below the brow line, forward-facing surfaces only.
- **Facemask cage** — real geometry: 3 horizontal bars + 2 verticals + 2 cheek bars
  (tube-along-curve, silver metallic). Sits just in front of the nose.
- **Chinstrap + chin cup**, **padding band** (torus, scaled in y to the head's
  ellipse) + **ear pads** (inside the shell), **6 crown vents** (dark slots),
  **centre stripes** (Washington: white + 2 gold, under a `stripe_washington`
  parent the page hides for Broncos), **side logo decals** (shell-conforming
  patches via KD-tree, UV-mirrored on the left; Broncos = slanted "BRONCOS"
  wordmark, Washington = gold serif "W").
- **Neck roll** (torus at Spine2), **cleat studs** (14 cones under the soles,
  foot-parented), **football** (RightHand-parented, with lace boxes as children).
- **Wrist cuffs** (white cylinders around the wrists, read as wristbands).
- **Visor (Loft) REMOVED** — the original visor was sized for the small stylised
  helmet and stuck out in front of the face like a beak with the new shell.
  Phase 2: remodel a visor that fits the new shell.
- **Glove ellipsoids REMOVED** — a rigid ellipsoid cannot cover a splayed hand:
  too small and the palm pokes through, too big and it reads as a white ball on
  the wrist (tried 4 sizes). Hands stay bare (as in the original model).
  Phase 2: a proper glove = a copy of the hand mesh, slightly inflated and
  re-skinned to the hand/finger bones.

### Textures (`work/make_textures.py`, PIL + numpy)
Recolour by nearest-anchor palette mapping (WHITE/MAROON/GOLD/DARK) with luminance
preservation, so folds/shading survive. Two fixes after viewing the first pass:
grey pixels forced to the DARK anchor (killed orange speckle in mesh-dot areas);
Washington MAROON→GOLD uses a brighter luminance scale (gold reads properly).
- `broncos_jersey.png` 2835×1134 (457,805 B) — white/navy/orange, "10" front+back
  (DejaVu Sans Bold, navy fill, orange outline, centred on the UV-probe centres).
- `washington_jersey.png` 2835×1134 (428,194 B) — burgundy/gold, "16" (white fill,
  gold outline). `washington_pants.png` 2048² (651,758 B) — source as-is (already
  white + burgundy/gold stripe). `broncos_pants.png` 2048² (297,318 B).
- `jersey_fabric_normal.png` 1024² tileable weave + wrinkle normal map (739,525 B),
  applied at 6×4 repeat.
- `decal_w.png` gold serif "W" (17,609 B). `decal_horse.png` (15,974 B) — **a
  slanted white "BRONCOS" wordmark, NOT a horse**: two polygon horse silhouettes
  both rendered as an unreadable white blob at 512 px. A proper horse mark is
  Phase 2 work (trace from the photo reference with more care).

### Build iterations (what went wrong, so it is not repeated)
1. **v1 — bone parenting doubled every transform.** `matrix_parent_inverse =
   bone_world⁻¹ @ obj.matrix_world` doubles the transform when the object is
   unparented (matrix_local == matrix_world, so new_world = old_world²). Proven
   by a controlled test. **Fix: use `bpy.ops.object.parent_set(type='BONE')`
   with the bone active — it preserves the world position exactly.**
2. **v2 — helmet shell was half the size it needed to be.** The shell was sized
   from the HEAD BONE's offset (x ±0.10) instead of the head MESH (x ±0.201,
   y −0.304..0.206, z 3.71–4.22). The head poked straight through the shell.
   **Fix: measure the mesh, not the bone.** New shell centre (0, −0.05, 3.97),
   radii (0.23, 0.28, 0.285). All head parts (cage, chin, pads, vents, stripes,
   decals) were repositioned to the true head frame.
3. **v3 — face cut ate the shell's sides.** The cut test used `c.y < -0.08` with
   `c` in MESH-LOCAL coords while the brow line was in world coords, so the brow
   check was a no-op and the cut removed a big rectangular notch on the sides.
   **Fix: convert face centres/normals to world space and cut only
   forward-facing surfaces (`normal.y < -0.4`) below the brow.** 176 faces cut
   (vs 420), clean opening.
4. **v4 — stripe floated in front of the face opening.** The stripe arc ran
   15°–165°, extending down into the cut-away face region. **Fix: arc 25°–155°.**
   Ear pads poked through the shell (centre x 0.205 vs shell surface 0.219) —
   moved inside to x ±0.185.
5. **v5/v6 — gloves.** See "Glove ellipsoids REMOVED" above. v6 is the shipped
   build (cuffs only).

## Budget (v6, final)

| Item | Value |
|---|---|
| GLB `models/player_aaa.glb` | **12,579,832 B (12.0 MB)** |
| Meshes / skinned meshes | 59 / 10 |
| Triangles (Node GLTFLoader count) | **113,756** (Blender count 89,360 — see note) |
| Skeleton | exactly **41 bones** (`mixamorig:*`) |
| Textures embedded in GLB | 9 images, all valid PNG/JPEG magic |
| Kit textures (loaded at runtime) | broncos jersey 457,805 B, washington jersey 428,194 B, broncos pants 297,318 B, washington pants 651,758 B, decals ~16–18 KB, fabric normal 739,525 B |
| Clips (FBX, loaded at runtime) | idle 1,027,980 B, running 458,620 B, QB pass 2,416,624 B |
| Page `index.html` | 14,979 B (+ vendored three 4.2 MB) |
| First-load estimate (page + GLB + 3 clips + kit textures) | ~**21 MB** — far under the iOS ~1.5–2 GB tab-kill budget |
| .blend source | 19,674,884 B |

Tri-count note: Node's GLTFLoader reports 113,756 tris vs Blender's 89,360. The
difference is unresolved (likely the exporter's triangulation of the solidified
shell + index counting). Both are inside the 50k–200k Phase 1 budget; the Node
number is the one the page will actually render.

## Verification (all headless, all passing unless stated)

### Node — GLB + clips (`work/verify_node.mjs`, three@0.187.0) — **42/42 PASS**
- GLB loads; 59 meshes, 10 skinned, 113,756 tris; key materials present
  (helmet_shell, cage, jersey, pants, decal, decal_left, stripe_white/gold).
- All 9 embedded textures present with valid PNG/JPEG magic (broncos_jersey,
  jersey_fabric_normal, pant_clean, decal_horse, …).
- Skeleton exactly 41 bones. **Name caveat (resolved, not a bug):** three.js
  `GLTFLoader` AND `FBXLoader` both strip the reserved track-binding characters
  `[] . : /` from node names (`PropertyBinding.sanitizeNodeName`), so the GLB
  bones load as `mixamorigHips` (no colon) and the clip tracks use the same
  sanitized names — that is why the clips bind. The test expects sanitized names.
- Detail nodes present and parented to the right joints (football→RightHand,
  cage→Head, cuffs→hands, stripes→Head, neck roll→Spine2, 6 vents, 14 studs).
- **All 3 clips drive the skeleton**: playing each clip moves the Hips bone, and
  every non-finger track binds (123/123 idle, 123/123 running, 35/35 QB pass).
  The clips also animate Middle/Ring/Pinky finger bones the 41-bone rig does not
  have (17–18 tracks per clip) — those are ignored by the mixer, exactly as in
  the current game (same rig, same clips). Not a defect.
- Texture loading in Node fails with "Couldn't load texture blob:…" — a Node
  artifact (blob: URLs don't resolve in Node); in a browser the embedded
  textures load normally. Verified instead from the GLB JSON (images + magic
  bytes) and from the Cycles renders below.

### Blender — Cycles renders (`work/render_check.py`, CPU 48 samples)
5 views, v6 model: front / back / side / head_front / head_side.
**Viewed and accepted:** the helmet shell wraps the head (navy, smooth, face
opening clean), the cage sits in front of the face, the centre stripe hugs the
crown, the jersey "10" is clean front and back, the football is in the right
hand, pants/cleats/studs read correctly. Known render-only notes: the navy
reads a little light under the render's bright sun (clearcoat reflecting the
grey background — the material itself is navy 0x0a2342); the chin cup reads as
a small dark shape under the chin; the shell's face-opening rim is slightly
stepped where the cut follows the sphere grid.

### Page — static checks
- Module script passes `node --check`.
- Import map is valid JSON; all vendored files exist (three.webgpu.js 1,350,894 B
  + 6 addons + fflate).
- All assets the page references exist (GLB, 3 clips, 6 kit textures).

### jsdom boot test (`work/boot_jsdom.mjs`) — **cannot run the page, documented**
jsdom 24.1.3 does **not** execute `<script type="module">` in this environment
(proven with a minimal module test: the script never runs, no error surfaced,
both from file:// and over a local HTTP server). So the page's runtime — renderer
init, WebGL fallback, model/clip loading, kit/animation/quality buttons — is
**NOT verifiable headlessly via jsdom**. The fallback logic was verified by code
inspection only: `initRenderer()` is wrapped in try/catch, on failure it sets the
badge to "No WebGPU / WebGL" and a status message and returns; `main()` has a
`.catch` that sets "Failed to start". **A real browser boot is still required.**

## What could NOT be verified headlessly (needs a browser / Joshua's phone)
1. **The WebGPU path** — no headless WebGPU here. The page must be opened in a
   browser with WebGPU (iPhone 15 Pro Max on iOS 26 qualifies) and the badge must
   read "WebGPU".
2. **The WebGL 2 fallback path** — same; on a non-WebGPU device the badge must
   read "WebGL 2 (fallback)" and the scene must still render.
3. **Texture display in the page** — the GLB's embedded textures are proven at
   the byte level and the jersey/pants render correctly in Blender, but the page's
   runtime texture swap (Broncos ↔ Washington) needs a browser look.
4. **Kit / animation / quality buttons** — wired in code (`wireUI`, `applyKit`,
   `applyAnim`, `applyQuality`), never clicked in a real browser.
5. **Animation playback in the page** — clips are proven to drive the skeleton
   in Node (Hips moves, all non-finger tracks bind), but the page's mixer loop,
   crossfades and football visibility toggling need a browser look.
6. **Visual quality vs the photos** — the Cycles renders are the closest
   headless proxy; whether the model reads as "AAA" next to his Broncos /
   Washington photos is Joshua's eye verdict, on his phone.

## Phase 2 (not started)
- Full model set: the other kit pieces as separate models (away/home variants),
  a second body type, and the stadium/field dressing ported from the current
  game (the Phase 1 field is a flat plane + sky dome only).
- Feature port: league/schedules, studio pregame, replay, Edit Player, crowd,
  audio — the current game's features, rebuilt on the r187 renderer.
- Model polish: proper horse logo (trace from the photo), a visor remodelled to
  fit the new shell, a skinned glove (hand-mesh copy, re-weighted), shoulder
  pads as real geometry, jersey stitching as geometry (not just the normal map),
  a darker/clearer navy under the page's ACES tone mapping if his eye says so.
- Memory/LOD: device-first loading (on-demand GLB fetch, IndexedDB cache, phone
  LOD tiers) per the MF scale plan, before the model set grows past the iOS
  ~1.5–2 GB tab budget.
- Real-browser verification gate: every phase gets a test URL and Joshua's phone
  verdict before anything reaches a live root — same rule as the current game.

---

## Push prep (Nov 13 — split for the GitHub snapshot-push transport cap)

**Stage only — do NOT push.** Parent sequences pushes. This section records the
split + verification done so the /aaa/ push can be batched.

### The split
`models/player_aaa.glb` (12,579,832 B, sha256
`5ef86253ccfff3f029fba322568d458aee24716c9deba358ca30381637b54e7f`) exceeds the
MCP transport cap (~3.6 MB raw per request), so it is split into 4 equal parts,
each **3,144,958 B** (cap is 3,563,520 B for a 3.4 MB part):

| Part | Size | sha256 (first 16) |
|---|---|---|
| `models/player_aaa_p1.glbpart` | 3,144,958 | `30d5af0fac10327d` |
| `models/player_aaa_p2.glbpart` | 3,144,958 | `33c6f17749abfbd7` |
| `models/player_aaa_p3.glbpart` | 3,144,958 | `4116b00dd4e04dcc` |
| `models/player_aaa_p4.glbpart` | 3,144,958 | `41d10efc2acf041f` |

Reassembly of the 4 parts is **byte-identical** to the original GLB (sha256
`5ef86253…`, verified in Python). The original single-file GLB stays in
`models/` as the source of truth and is **NOT in the push set** (over the cap,
duplicates the parts). The `.blend` (19,674,884 B) is also **NOT pushed**
(source only).

### Page change (`index.html`, 15,037 → 16,088 B)
The player load is now `loadModelParts()`: fetches p1..p4 **in order**, shows
`Loading player model… (part n/4)` in the status line, concatenates the
ArrayBuffers into one buffer, and passes it to
`new GLTFLoader().parseAsync(bytes, '')`. Everything else is unchanged —
same scene, kits (Broncos #10 / Washington #16), clips, quality toggle,
backend badge. Module script passes `node --check`.

### Missing dependency fixed (would have broken the page in a browser)
`vendor/addons/loaders/FBXLoader.js` statically imports
`../curves/NURBSCurve.js`, which was **not vendored** — the page's module
graph would have failed to load in a browser (the Phase 1 static check only
covered the page's direct references). Copied from the installed three r187:
`vendor/addons/curves/NURBSCurve.js` (4,157 B) + `NURBSUtils.js` (10,802 B,
its only relative import). A full transitive-import scan of `vendor/` now
reports **zero missing imports** (three.webgpu.js → ./three.core.js +
./three.tsl.js; GLTFLoader → BufferGeometryUtils + SkeletonUtils; FBXLoader →
fflate + curves). No CDN fallback was needed: every vendor/library file is
under the 3.6 MB cap (largest: three.core.js 1,474,799 B), so the page stays
fully offline as designed.

### Verification (re-run this session)
- **Reassembled GLB** (`work/reassembled/models/player_aaa.glb`, built from
  the 4 parts + symlinked `clips/`): `node work/verify_node.mjs
  work/reassembled` → **42/42 PASS, 0 FAIL, exit 0** ("ALL CHECKS PASSED").
  Control run against the original GLB: also **42/42 PASS**.
- **Module script**: `node --check` on the extracted `<script type="module">` → OK.
- **Asset audit**: every path `index.html` references exists (4 parts, 3 clips,
  7 textures — the vendor files resolve through the import map, not literal
  paths, and were verified by the transitive-import scan above).

### Full push set for /aaa/ (all ≤3.6 MB raw)

| File | Size | UTF-8? |
|---|---|---|
| `index.html` | 16,088 | yes |
| `models/player_aaa_p1..p4.glbpart` (×4) | 3,144,958 each | **no — binary** |
| `clips/idle.fbx` | 1,027,980 | **no — binary** |
| `clips/running.fbx` | 458,620 | **no — binary** |
| `clips/quarterback_pass.fbx` | 2,416,624 | **no — binary** |
| `textures/broncos_jersey.png` | 457,805 | **no — binary** |
| `textures/broncos_pants.png` | 297,318 | **no — binary** |
| `textures/decal_horse.png` | 15,974 | **no — binary** |
| `textures/decal_w.png` | 17,609 | **no — binary** |
| `textures/jersey_fabric_normal.png` | 739,525 | **no — binary** |
| `textures/washington_jersey.png` | 428,194 | **no — binary** |
| `textures/washington_pants.png` | 651,758 | **no — binary** |
| `vendor/three.core.js` | 1,474,799 | yes |
| `vendor/three.tsl.js` | 1,065,544 | yes |
| `vendor/three.webgpu.js` | 1,350,894 | yes |
| `vendor/addons/controls/OrbitControls.js` | 40,918 | yes |
| `vendor/addons/curves/NURBSCurve.js` | 4,157 | yes |
| `vendor/addons/curves/NURBSUtils.js` | 10,802 | yes |
| `vendor/addons/environments/RoomEnvironment.js` | 4,960 | yes |
| `vendor/addons/libs/fflate.module.js` | 91,108 | yes |
| `vendor/addons/loaders/FBXLoader.js` | 113,530 | yes |
| `vendor/addons/loaders/GLTFLoader.js` | 119,861 | yes |
| `vendor/addons/utils/BufferGeometryUtils.js` | 39,153 | yes |
| `vendor/addons/utils/SkeletonUtils.js` | 11,535 | yes |

Push-set total ≈ 23.0 MB (26 files). `work/` (incl. `node_modules`,
`work/reassembled/`, renders) is local-only and never pushed.

### ⚠️ BLOCKER — binary files cannot go through snapshot-push as raw files
The snapshot-push procedure (`/opt/hatch/connectors/github/references/snapshot-push.md`)
and the CLI help both say the snapshot is **UTF-8 only**: "non-UTF-8 content
… reject the whole snapshot", and the tool is for snapshots "without …
binary contents". The sync log (`pages.muse-github-snapshot-sync.json`, 49
records, 88 paths) contains **zero binary files ever pushed** — only .js /
.html / .md. A Python UTF-8 decode test of the actual push-set files confirms
**every binary asset fails UTF-8** (GLB at byte 8, FBX at bytes 23–27, PNGs at
byte 0, all 4 `.glbpart` files at bytes 0–8). So as staged, the push will be
rejected at validation — not just the GLB parts but also the 3 FBX clips and
all 7 PNG textures. Size is fine everywhere; **encoding is the blocker**.

**Remedy (needs parent authorization — it changes how every asset loads):**
wrap each binary asset in base64 JS chunk modules, the repo's established
pattern (`(window.GLM=window.GLM||{})["<name>"]="…"` / append chunks), and
have the page atob-decode them (GLB → `GLTFLoader.parse`, FBX →
`FBXLoader.parse`, PNG → Blob/object URL → `TextureLoader`). Exact base64
sizes (wrapper adds ~100 B/file):

| Asset | Base64 size | JS files |
|---|---|---|
| GLB 12,579,832 B | 16,773,112 | 5 chunks × ~3.35 MB |
| `quarterback_pass.fbx` | 3,222,168 | 1 |
| `idle.fbx` | 1,370,640 | 1 |
| `running.fbx` | 611,496 | 1 |
| 7 PNG textures | 21,300–986,036 each | 1 each |
| **Total** | **≈ 24.5 MB in 15 JS files** | |

**Remedy applied this session — see "REMEDY DONE" below.** The `.glbpart` +
fetch mechanism is superseded by base64 JS chunk modules and is NOT pushed.

### ✅ REMEDY DONE — base64 JS chunk modules (Nov 13, this session)

All 11 binary assets are wrapped in base64 JS chunk modules (the repo's
established GLM pattern, same as the game's `models/*.js`), generated by
`work/make_chunks.py` (re-runnable; it self-verifies every file). The page
was rewired: 15 classic `<script>` tags load the chunk modules in order
before the module script; the module atob-decodes them — GLB →
`GLTFLoader.parseAsync`, FBX → `FBXLoader.parse`, PNG → Blob/object URL →
`TextureLoader`. No `fetch()` of binary files remains in the page (the only
remaining `.glb` mention is a comment with the byte count). The `.glbpart`
split is superseded and NOT pushed. The original binaries + `.blend` stay in
staging as sources of truth and are NOT pushed.

| Asset | Source bytes | Base64 chars | JS files | Largest JS |
|---|---|---|---|---|
| `models/player_aaa.glb` | 12,579,832 | 16,773,112 | 5 (`player_aaa_p1..p5.js`) | 3,354,707 |
| `clips/quarterback_pass.fbx` | 2,416,624 | 3,222,168 | 1 | 3,222,220 |
| `clips/idle.fbx` | 1,027,980 | 1,370,640 | 1 | 1,370,680 |
| `clips/running.fbx` | 458,620 | 611,496 | 1 | 611,539 |
| `textures/broncos_jersey.png` | 457,805 | 610,408 | 1 | 610,458 |
| `textures/broncos_pants.png` | 297,318 | 396,424 | 1 | 396,473 |
| `textures/decal_horse.png` | 15,974 | 21,300 | 1 | 21,347 |
| `textures/decal_w.png` | 17,609 | 23,480 | 1 | 23,523 |
| `textures/jersey_fabric_normal.png` | 739,525 | 986,036 | 1 | 986,092 |
| `textures/washington_jersey.png` | 428,194 | 570,928 | 1 | 570,981 |
| `textures/washington_pants.png` | 651,758 | 869,012 | 1 | 869,064 |
| **Total** | **18,376,231** | **24,455,904** | **15** | — |

Chunk wrapper (GLB, append form, all 5 chunks — first chunk included, the
`||""` handles the missing key; single-file assets use the plain set form):
`(window.GLM=window.GLM||{})["player_aaa"]=((window.GLM||{})["player_aaa"]||"")+"…";`

### Verification (all re-run this session, after the rewire)
- **`work/make_chunks.py`** — all 15 JS files generated, each ≤3,354,707 B;
  every payload re-read from disk, decoded, sha256-compared to the original:
  **"ALL CHUNK CHECKS PASSED"**.
- **`work/reassemble_from_chunks.py`** — decoded the chunk JS files back to
  `work/reassembled/` (GLB + 3 FBX), sha256 == originals:
  **"REASSEMBLY FROM CHUNKS OK"**.
- **`node work/verify_node.mjs work/reassembled`** → **42/42 PASS,
  "ALL CHECKS PASSED"** — now against the *chunk-decoded* GLB and
  *chunk-decoded* clips (the `clips/` symlink was replaced with real decoded
  files for this run). Finger-track warnings are the known ignored tracks.
- **`work/simulate_glm.mjs`** — evaluated the 15 chunk modules in page order
  in a Node vm with a `window` global (browser-equivalent): all 11 GLM keys
  present with exact base64 lengths; decoded GLB / FBX / PNG sha256 ==
  originals; `atob(GLM.player_aaa)` = 12,579,832 bytes:
  **"GLM SIMULATION PASSED"**.
- **`node --check`** — module script OK; all 15 chunk modules parse OK.
- **`work/scan_imports.py`** — 12 relative imports + import-map targets:
  **ZERO MISSING IMPORTS**.
- **Final asset audit** — all 15 `<script src>` exist; all 10 GLM keys the
  page reaches have chunk modules; full push set 29 files, all UTF-8,
  max 3,354,707 B.

### Final push set for /aaa/ (29 files, 29,826,096 B)

| File | Size |
|---|---|
| `models/player_aaa_p1..p4.js` (×4) | 3,354,707 each |
| `models/player_aaa_p5.js` | 3,354,699 |
| `clips/quarterback_pass.js` | 3,222,220 |
| `clips/idle.js` | 1,370,680 |
| `clips/running.js` | 611,539 |
| `textures/jersey_fabric_normal.js` | 986,092 |
| `textures/washington_pants.js` | 869,064 |
| `textures/broncos_jersey.js` | 610,458 |
| `textures/washington_jersey.js` | 570,981 |
| `textures/broncos_pants.js` | 396,473 |
| `textures/decal_w.js` | 23,523 |
| `textures/decal_horse.js` | 21,347 |
| `vendor/three.core.js` | 1,474,799 |
| `vendor/three.webgpu.js` | 1,350,894 |
| `vendor/three.tsl.js` | 1,065,544 |
| `vendor/addons/loaders/GLTFLoader.js` | 119,861 |
| `vendor/addons/loaders/FBXLoader.js` | 113,530 |
| `vendor/addons/libs/fflate.module.js` | 91,108 |
| `vendor/addons/controls/OrbitControls.js` | 40,918 |
| `vendor/addons/utils/BufferGeometryUtils.js` | 39,153 |
| `vendor/addons/utils/SkeletonUtils.js` | 11,535 |
| `vendor/addons/curves/NURBSUtils.js` | 10,802 |
| `vendor/addons/environments/RoomEnvironment.js` | 4,960 |
| `vendor/addons/curves/NURBSCurve.js` | 4,157 |
| `index.html` | 17,276 |
| `NOTES.md` | 25,655 |

Notes: `textures/jersey_fabric_normal.js` is generated and pushed for
completeness but the page does not load it via GLM — that texture is embedded
in the GLB. `work/` (incl. `node_modules`, `reassembled/`, renders) is
local-only, never pushed. **Push batching:** ≤3.6 MB raw per snapshot-push
call — the 5 GLB chunks and `quarterback_pass.js` are each ~3.2–3.4 MB, so
each goes in its own batch; the rest can be grouped. Parent sequences pushes.
