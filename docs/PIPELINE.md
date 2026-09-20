# The Gaussian Splatting Pipeline, End to End

Everything installed on this Mac, what each piece does, why it's there, and
how to undo it. Written so nothing in this setup is a black box.

---

## 1. The mental model

A 3D Gaussian Splat is **a few million translucent 3D ellipsoids** ("Gaussians",
or "splats") floating in space. Each one has a position, a size and orientation,
a colour, and an opacity. Render them back-to-front with alpha blending from any
camera angle and you get a photo-real image.

There is **no mesh, no texture, no lighting calculation**. Appearance is baked in.
That's why it looks photographic and why you can't relight it.

Getting from video to splat is three separate problems, and they're solved by
three different programs:

| Stage | Question it answers | Tool |
|---|---|---|
| 1. Frame selection | Which frames are worth using? | `sharp_frames.py` (ours) |
| 2. Structure from Motion | Where was the camera for each frame? | COLMAP |
| 3. Optimisation | What splats reproduce these photos? | Brush |
| 4. Compression | How do I ship this over the web? | splat-transform |
| 5. Rendering | How do I draw it in a browser? | Spark + three.js |

Stage 2 is the one people underestimate. If COLMAP can't work out the camera
poses, nothing downstream can save you.

---

## 2. What is installed and where

### Homebrew packages (system-wide)

```
colmap  4.1.1_3   36 MB   + 101 dependencies
ffmpeg  9.0.1_1   52 MB   + 14 dependencies
```

Installed with `brew install colmap ffmpeg`. COLMAP's 101 dependencies are
mostly linear-algebra and image libraries (Ceres Solver, Eigen, Boost, FreeImage,
SuiteSparse). That's why it's a heavy install despite a small binary.

Remove with `brew uninstall colmap ffmpeg && brew autoremove`.

### Self-contained, in `~/Documents/splat/`

```
bin/brush            144 MB   prebuilt Rust binary, v0.3.0
.venv/               167 MB   Python venv with opencv 5.0.0
bin/sharp_frames.py  170 lines
bin/colmap-sparse     30 lines
bin/splat             86 lines
```

Nothing here touches the system. Delete the folder and it's all gone.

The venv matters: opencv is installed **inside** `~/Documents/splat/.venv`, not
system-wide. That's deliberate — Python packages installed globally collide with
each other across projects, and Homebrew's Python actively blocks it. The cost is
that a venv bakes absolute paths into its scripts, so **moving the folder breaks
it** and you must rebuild:

```bash
python3 -m venv .venv && .venv/bin/pip install opencv-python
```

### Run on demand, never installed

`splat-transform` and `vercel` run through `npx`, which downloads to a cache
rather than installing. That cache is currently **1.0 GB** at `~/.npm/_npx`.
Safe to clear any time with `npm cache clean --force`; npx re-downloads as needed.

### The one system change

A single line appended to `~/.zshrc`:

```bash
export PATH="$HOME/Documents/splat/bin:$PATH"
```

Delete that line to fully revert. That's the complete list of changes outside
the two project folders.

---

## 3. Stage 1 — Frame selection

### The problem

A 90-second 60fps video has 5,400 frames. You want ~250. The naive approach,
`ffmpeg -vf fps=2`, takes whatever frame lands on each tick — blurry or not.

**Camera shake is harmless. Motion blur is fatal.** COLMAP solves each frame's
pose independently, so a jittery path doesn't bother it. But a blurred frame has
smeared features, which both breaks feature matching and bakes fuzz into the
final splat.

### The technique: variance of the Laplacian

The Laplacian is a second-derivative filter — it responds to edges. A sharp image
has lots of strong edges, so the variance of its Laplacian is high. A blurred
image has soft gradients, so the variance collapses.

```python
gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
score = cv2.Laplacian(gray, cv2.CV_64F).var()
```

`sharp_frames.py` divides the video into N equal time windows and keeps the
**single sharpest frame from each**. That gives even coverage of the camera path
AND sharp frames — you can't get both by sampling at a fixed rate.

The `--report` flag prints a sparkline of sharpness over time, so you can see
which part of the orbit you rushed.

### Why absolute scores don't matter

A Laplacian variance of 195 means nothing on its own — it depends on resolution,
texture and lighting. What matters is the **ratio to the median**. This capture's
worst frame scored 106 against a median of 195: nothing below half the median,
so no soft sections.

---

## 4. Stage 2 — COLMAP and Structure from Motion

This is the stage that actually decides whether your capture works.

### What SfM does

Given a pile of photos, work out simultaneously:
- where each camera was (position + orientation = **pose**)
- the camera's internal parameters (focal length, distortion = **intrinsics**)
- a sparse 3D point cloud of the scene

It's a chicken-and-egg problem — you need 3D points to solve poses and poses to
triangulate points — solved by bootstrapping from an initial image pair and
adding images incrementally, re-optimising as it goes.

### The three steps

```bash
colmap feature_extractor   # find SIFT keypoints in each image
colmap exhaustive_matcher  # match keypoints between image pairs
colmap mapper              # solve poses + triangulate points
```

**Feature extraction** finds distinctive local patterns (SIFT keypoints) — corners,
texture, anything locally unique. A blank white wall yields almost none. This is
why textureless subjects fail.

**Matching** compares keypoints between image pairs, then runs geometric
verification: given the candidate matches, is there a camera motion that explains
them? Matches that don't fit are discarded as outliers.

Exhaustive matching compares *every* pair — O(n²). For 250 images that's 31,125
pairs. The alternative, sequential matching, only compares nearby frames in time.

> **Why we used exhaustive:** this capture was several orbits at different
> heights. Sequential matching would link consecutive frames but never connect
> the low orbit to the high one, fragmenting the reconstruction. Exhaustive
> catches those cross-orbit matches. Cost: 4 minutes on CPU at ~85 pairs/sec.

**Mapping** is incremental bundle adjustment: pick a good initial pair, triangulate,
add the next best image, re-optimise everything, repeat. Bundle adjustment is a
huge non-linear least-squares problem minimising *reprojection error* — the pixel
distance between where a 3D point lands when projected into a camera and where
its matched keypoint actually is.

### Reading the output

```
Registered images:       250 / 250
Points:                  109,072
Observations:            666,250
Mean track length:       6.11
Mean reprojection error: 0.68 px
```

- **Registered images** — how many got a pose. Anything less than ~90% means
  trouble. 250/250 is ideal.
- **Mean track length 6.11** — each 3D point was seen from ~6 cameras. Above 3 is
  healthy; more means better-constrained geometry.
- **Mean reprojection error 0.68 px** — sub-pixel is excellent. Above ~1.5px
  suggests a bad solve.
- **Multiple `sparse/N` folders = failure.** It means COLMAP couldn't tie the
  capture into one consistent scene and gave you disconnected fragments.

### Predicting success before you commit an hour

The match graph tells you in advance. Query the database directly:

```sql
SELECT count(*) FROM two_view_geometries WHERE rows >= 30;
```

What mattered for this capture: **every image had at least 54 connections** with
30+ geometric inliers, averaging 108. No isolated images means no fragmentation.

7,875 pairs failed entirely — expected and healthy. Those are frames on opposite
sides of the orbit that share no visible surface.

### `--single_camera 1`

Every frame came from one phone, so they share intrinsics. Telling COLMAP that
collapses hundreds of unknowns into one set and makes the solve much more stable.
Only use it when it's true.

### Why we skipped `--dense`

COLMAP's dense stage (multi-view stereo) needs CUDA, which macOS doesn't have.
It doesn't matter: splatting only needs the **sparse** output — camera poses plus
a sparse cloud to initialise the Gaussians. Dense reconstruction is for meshes.

### Version gotcha

COLMAP 4.x renamed CLI options. Most tutorials online predate this and will fail:

```
--SiftExtraction.use_gpu        ->  --FeatureExtraction.use_gpu
--SiftExtraction.max_image_size ->  --FeatureExtraction.max_image_size
--SiftMatching.use_gpu          ->  --FeatureMatching.use_gpu
```

`--SiftExtraction.estimate_affine_shape` kept its old name. When in doubt:
`colmap feature_extractor -h | grep -i <thing>`.

---

## 5. Stage 3 — Training with Brush

Brush is a Rust implementation of 3DGS using **wgpu**, which means it runs on
Metal on macOS. Most 3DGS trainers are CUDA-only; this is why Brush was the
right choice here.

### What one training step actually does

1. **Pick a training view** — one of the 250 photos, with its solved pose.
2. **Project every Gaussian** into that camera. Each splat's 3D covariance
   (built from its scale + rotation quaternion) is projected through a Jacobian
   into a 2D screen-space ellipse.
3. **Tile binning and sort.** The screen is cut into 16x16 tiles. Each splat is
   duplicated once per tile it overlaps, then *all* those pairs are radix-sorted
   by depth. Usually the single biggest cost.
4. **Rasterise.** For each pixel, walk front-to-back through that tile's sorted
   splats, alpha-blending until the pixel saturates.
5. **Loss** — L1 plus 0.2 x SSIM against the real photo.
6. **Backward + Adam.** Gradients flow back through the blending to every splat's
   position, scale, rotation, opacity and colour.

### Densification — why it slows down

Training doesn't use a fixed number of splats. Periodically it **clones and splits**
Gaussians in areas with high positional gradient (i.e. where the image is still
wrong), and prunes near-transparent ones. This is how detail emerges.

Measured on this capture:

```
step  5,000      324,033 splats     777 steps/min
step 10,000      956,707            600
step 15,000    1,806,414            405   <- densification stops
step 20,000    1,806,414            373
step 30,000    1,806,414            373
```

Brush stops growth at step 15,000 by default (`--growth-stop-iter`). After that
the count is frozen and the rate flattens — remaining steps just refine.

**Lesson: never extrapolate training time from the first few thousand steps.**
The early rate is measured on a fraction of the final splats. Estimating from
777 steps/min predicted 39 minutes; the real answer was 61.

### Where the compute actually goes

1,806,414 splats x **59 parameters each = 107 million trainable parameters.**

Of those 59: 3 position, 3 scale, 4 rotation, 1 opacity, 3 base colour, and
**45 spherical-harmonic coefficients**. So **76% of all gradient and optimiser
work is view-dependent colour** — modelling how each blob changes appearance
with viewing angle. That's what makes reflections look right, and it's most
of your compute.

### Spherical harmonics, briefly

A splat's colour isn't one RGB value. It's a function over viewing direction,
expanded in spherical harmonics. Degree 0 is a constant (flat colour). Each
higher degree adds angular detail: specular highlights, sheen, view-dependent
shifts. Degree 3 = 16 coefficients per channel x 3 = 48 values.

Practically: **SH degree is your main quality/size dial.**

### Faster next time

```bash
brush proj --with-viewer --sh-degree 2 --total-steps 15000 --max-resolution 1280
```

- `--sh-degree 2` nearly halves parameters. Barely visible on matte subjects.
- `--total-steps 15000` — the 15k result is close to final. 30k → 15k saved
  about half an hour here for a difference hard to see.
- `--max-splats` caps densification directly.

---

## 6. Stage 4 — Compression to `.sog`

407 MB of ply is not shippable. `.sog` got it to 4.9 MB — **83x**.

### What `.sog` is

A **ZIP of WebP images plus a JSON manifest**:

```
means_l.webp, means_u.webp    positions, split low/high byte = 16-bit per axis
quats.webp                    rotations
scales.webp                   scales, quantised to a 256-entry codebook
sh0.webp                      base colour, 256-entry codebook
shN_centroids.webp            higher-order SH, k-means to 65,536 centroids
shN_labels.webp               per-splat index into those centroids
meta.json                     count, SH bands, bounding box, codebooks
```

Three compression ideas stack here:

1. **Quantisation** — float32 becomes 8 or 16-bit with a stored range.
2. **Codebooks** — scales and colours snap to 256 representative values.
3. **k-means clustering** — the expensive SH coefficients are replaced by an
   index into 65,536 learned centroids. That's the `k-means` step in the log.

Storing it all as texture data means the GPU samples it directly — decode is
nearly free.

### The flags that matter

```bash
npx @playcanvas/splat-transform in.ply -N -F -H 2 out.sog
```

- `-H <n>` — drop SH bands above n. **Biggest size lever**, since SH is 76% of
  the data. But see the caveat below.
- `-F` — remove "floaters": splats not contributing to any solid voxel. This is
  what kills the haze.
- `-N` — drop NaN/Inf splats.
- `-S x,y,z,r` / `-B` — crop to a sphere or box.
- `-d n%` — decimate.

### Two hard-won ordering rules

**Crop before filtering floaters.** `-F` builds a voxel grid over the scene's
bounding box. With far-field outlier splats still present, that box was
2.45 km x 4.93 km x 897 m — 86.6 trillion voxels. It simply failed. Crop first.

**`-H` stops mattering once splats dominate.** On the bowl (260k splats), SH
was most of the file. On the room (1.6M splats), SH band 1 came out at 21.7 MB
and band 2 at 22.1 MB — nearly identical, because positions and quaternions now
dominate. Check before assuming.

---

## 7. Stage 5 — Rendering on the web

### Why Spark and not the other one

The local diagnostic viewer used `@mkkellogg/gaussian-splats-3d`. Fine for a
throwaway check, wrong for deployment:

|  | mkkellogg | Spark |
|---|---|---|
| Last release | Jan 2025 | Sep 2026 |
| Reads `.sog` | no | **yes** |
| Architecture | owns the scene | integrates into your scene graph |
| Backing | one developer | World Labs |

The architecture difference is the one you feel. mkkellogg gives you a `Viewer`
that creates its own canvas, camera and render loop — which is why moving the
camera meant reaching into `window.__viewer.camera`. Spark gives you a
`SplatMesh` you add to a scene **you** control, so it composes with meshes,
lights and your own UI, and sorts correctly against them.

### The whole renderer

```js
const spark = new SparkRenderer({ renderer });   // composites splats
scene.add(spark);
const splat = new SplatMesh({ url: "./assets/x.sog" });
scene.add(splat);
```

Loaded from CDN via an importmap — no build step, no `node_modules`.

---

## 8. The coordinate system problem

This caused more trouble than anything else, so it gets its own section.

**COLMAP has no gravity reference.** It defines world axes from the first image
pair it registers. "Up" in the reconstruction is wherever your phone happened to
be pointing. Also, COLMAP's **+Y points down** by convention, and scale is
arbitrary — there are no metres, only units.

So every splat arrives tilted and mis-scaled, and nothing downstream fixes it.

### Finding true up

Two independent estimates, which agreed within **1.6 degrees**:

1. **RANSAC plane fit to the tabletop.** Repeatedly sample 3 points, fit a plane,
   count inliers; keep the best, then refine with SVD on all inliers. The table
   is horizontal in reality, so its normal is vertical. 42% of solid splats were
   inliers.
2. **Normal of the camera-orbit plane.** You walked around the table roughly
   level, so the best-fit plane through all 250 camera positions is horizontal.
   SVD, smallest singular vector.

When two unrelated methods agree, you can trust the number.

### Applying it — the part I got wrong twice

**Don't use Euler angles.** I computed 144.84 degrees about X and passed it to
`splat-transform -r`. The result was *worse* — 70 degrees off instead of 35 —
because Euler angles depend on a rotation-order convention (XYZ vs ZYX,
intrinsic vs extrinsic) and I guessed the wrong one.

**Do this instead:**

```js
splat.quaternion.setFromUnitVectors(measuredUp, new THREE.Vector3(0, 1, 0));
```

This builds the rotation that takes one vector onto another directly. No
convention to get wrong. One line, correct first time.

> **General lesson:** when you can express a rotation as "map this direction
> onto that direction", do that. Euler angles are a trap.

---

## 9. Mistakes, and what they teach

Kept because the reasoning is more useful than the fix.

**Crop radius too small.** The room view was cropped to a sphere of radius 12
around the bowl. Looked fine head-on; left huge black voids when orbiting.

```
 radius   splats inside   % of scene
     12        921,731       51.0%   <- cut here
     16      1,556,044       86.1%
     20      1,687,098       93.4%
```

The kitchen's walls sit between r=12 and r=16. The crop sliced the room shell
in half. **Black in a splat render means no splats** — either cropped away, or
never observed by any camera. Verify from multiple angles, not just the hero shot.

**Estimating from the fast part.** Predicted 39 minutes of training from the
first 5,000 steps, when only 324k of the eventual 1.8M splats existed. Real
answer: 61 minutes. Extrapolate from steady state, not warm-up.

**Trusting analysis over looking.** My estimates of the bowl's centre were wrong
twice — ray convergence put it on the near edge, a density histogram
underestimated its radius threefold. Both looked right when viewed down the
camera axis, which hid the depth error. Rendering and *looking* found it.

**Assuming two menu items do the same thing.** SuperSplat's File > Open accepts
only `.ssproj`; File > Import accepts the splat formats. Same dialog, completely
different filter.

**Predicting material difficulty.** I predicted the glossy table would smear and
the thin gold wire would come out as fuzz. Both reconstructed cleanly. With 250
well-registered views and sub-pixel error, 3DGS had plenty of signal. The
"specular and thin geometry are hard" heuristic is real but weaker than
sufficient coverage.

---

## 10. Cheat sheet

```bash
# whole pipeline
splat ~/Desktop/video.mov projectname

# stage by stage
~/Documents/splat/.venv/bin/python ~/Documents/splat/bin/sharp_frames.py \
    video.mov proj/images --target 250 --report
colmap-sparse ~/Documents/splat/projects/proj
colmap model_analyzer --path proj/sparse/0
brush proj --with-viewer --total-steps 15000 --export-path proj/output

# compress (crop BEFORE -F)
npx @playcanvas/splat-transform in.ply -N -F -H 2 out.sog

# inspect a .sog
unzip -l file.sog
unzip -p file.sog meta.json | python3 -m json.tool
```

### Diagnostics worth knowing

```bash
# how many pairs matched well?
sqlite3 proj/database.db \
  "SELECT count(*) FROM two_view_geometries WHERE rows >= 30;"

# did every image register?
colmap model_analyzer --path proj/sparse/0
```

---

## 11. Full uninstall

```bash
brew uninstall colmap ffmpeg && brew autoremove   # system packages
rm -rf ~/Documents/splat                          # tools, venv, brush, captures
npm cache clean --force                           # the 1 GB npx cache
# then delete the PATH line from ~/.zshrc
```

Nothing else was touched.
