# Gaussian Splat Render

A 3D Gaussian splat of a fruit bowl, rendered in the browser with
[Spark](https://github.com/sparkjsdev/spark) on top of THREE.js.

**Live asset:** `assets/fruitbowl-object.sog` — 260,403 splats, 4.9 MB.

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
