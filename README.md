# Gaussian Splat Render

A 3D Gaussian splat of a fruit bowl, rendered in the browser with
[Spark](https://github.com/sparkjsdev/spark) on top of THREE.js.

Two views, switchable in the UI:

| View | Asset | Splats | Size |
|---|---|---|---|
| Bowl only | `assets/fruitbowl-object.sog` | 260,403 | 4.9 MB |
| Full room | `assets/fruitbowl-room.sog` | 1,605,739 | 22 MB |

Both share the same coordinate frame and the same upright correction; only
the crop radius and the framing differ. Assets load lazily — the room scene
is only fetched if you switch to it.

### Crop radius matters

The room view was first built with a sphere of radius 12 around the bowl,
which looked fine head-on but left large black voids when orbiting: the
kitchen's walls and far surfaces sit between r=12 and r=16, so that crop
sliced straight through the room shell.

```
 radius   splats inside   % of scene
      3        274,767       15.2%
      6        296,819       16.4%
     12        921,731       51.0%   <- old crop cut here
     16      1,556,044       86.1%
     20      1,687,098       93.4%   <- current
```

Radius 20 keeps 93.4% of the scene. Black in a splat render means no
splats — either cropped away, or never observed by any camera.

## Running locally

No build step. It's a static page.

```bash
python3 -m http.server 8790
```

Then open http://localhost:8790. A server is required — ES module imports
and the `.sog` fetch both fail under `file://`.

## Deploying to Vercel

Static site, no framework, no build command.

```bash
npx vercel
```

Accept the defaults; leave the build command empty and the output directory
as the project root.

## How it works

`index.html` is the whole app. Spark loads from CDN via an importmap, so
there are no dependencies to install:

```js
const spark = new SparkRenderer({ renderer });   // composites splats into THREE.js
scene.add(spark);
const splat = new SplatMesh({ url: "./assets/fruitbowl-object.sog" });
scene.add(splat);
```

Because Spark integrates with the normal THREE.js scene graph, you can add
meshes, lights, and your own camera logic alongside the splat, and they sort
correctly against each other.

## The tilt correction

COLMAP has no gravity reference — it defines world axes from the first image
pair it registers — so the raw capture sits about 35 degrees off vertical.

True up was measured two independent ways that agreed within **1.6 degrees**:

- a RANSAC plane fit to the tabletop (42% of solid splats were inliers)
- the normal of the plane through all 250 camera positions

That gives `UP_IN_SPLAT = (-0.0049, -0.8176, -0.5758)`, which `index.html`
rotates onto `+Y` with `Quaternion.setFromUnitVectors`. The scene is then
translated so its bounding-box centre sits at the origin.

Note that trying to express this as Euler angles and bake it in with
`splat-transform -r` produced a *worse* tilt, because the tool's Euler
convention differs from the one assumed. `setFromUnitVectors` sidesteps
the convention question entirely.

## Responsive behaviour

The camera positions were tuned on a landscape viewport, so `show()` dollies
the camera back on narrow screens (`fit = clamp(1.45 / aspect, 1, 1.9)`) to
keep roughly the same horizontal framing on a phone.

Canvas sizing uses a `ResizeObserver` on the document element rather than
only `window.resize`, because the latter misses container-driven size changes
— in an embed, a split pane, or device emulation the canvas otherwise stays
stuck at its load-time size.

## Asset format

`.sog` is a ZIP of WebP textures plus a JSON manifest:

| File | Contents |
|---|---|
| `means_l.webp`, `means_u.webp` | positions, split low/high byte for 16-bit precision |
| `quats.webp` | rotations |
| `scales.webp` | scales, 256-entry codebook |
| `sh0.webp` | base colour, 256-entry codebook |
| `shN_centroids.webp`, `shN_labels.webp` | higher-order SH, k-means to 65,536 centroids |
| `meta.json` | splat count, SH band count, bounding box, codebooks |

Storing everything as texture data lets the GPU sample it directly.

## Provenance

| Stage | Result |
|---|---|
| Source video | 12,581 frames, 209 s, 1080x1920 @ 60 fps |
| Frames used | 250, picked by sharpest-per-window (variance of Laplacian) |
| COLMAP | 250/250 registered, 0.68 px mean reprojection error |
| Training | Brush, 30,000 steps, 61 min, 1,806,414 splats |
| Cropped | sphere r=3.0 around the bowl, floaters removed |
| Compressed | `.sog`, SH band 2 — 407 MB to 4.9 MB (83x) |
