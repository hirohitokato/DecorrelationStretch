# Decorrelation Stretch Image Enhancer

A self-contained browser tool for applying **Decorrelation Stretch** to an image or to a selected rectangular region of an image.

This tool runs entirely in the browser using HTML, JavaScript, Canvas, and an inline Web Worker.

In Japanese, See [README-ja.md](./README-ja.md).

## Features

- Single-file HTML application.
- Runs locally in a modern web browser.
- No external dependencies.
- Image loading by click or drag-and-drop.
- Original image and enhanced image are shown side by side.
- Rectangle selection on the original image.
- Decorrelation Stretch can be applied to the full image or to the selected rectangular region.
- Processing is executed in a Web Worker so the UI remains responsive.
- Presets for weak, medium, strong, and very strong enhancement.
- Manual control of all major algorithm parameters.
- PNG export.
- Drag export of the enhanced PNG from the result pane.
- JSON metadata export for reproducibility and audit workflows.
- Processing log including parameters, covariance, eigenvalues, eigenvectors, scales, and output range.

## Files

The project is implemented as a single file:

```text
src/index.html
```

## Quick start

1. Open `decorrelation-stretch.html`.
2. Click the original image area, or drag and drop an image file onto it.
3. Select a preset.
4. Optionally drag a rectangle on the original image to limit the enhancement region.
5. Click **補正実行**.
6. Compare the enhanced image with the original.
7. Save the result with **PNG保存**.
8. Save processing metadata with **パラメータJSON保存**.

## Presets

| Preset | `targetStd` | `lowPercentile` | `highPercentile` | `maxScale` | `samplePixels` |
|---|---:|---:|---:|---:|---:|
| 色差強調 弱 | 40 | 1.0 | 99.0 | 5 | 250000 |
| 色差強調 中 | 55 | 1.0 | 99.0 | 8 | 250000 |
| 色差強調 強 | 70 | 0.5 | 99.5 | 10 | 250000 |
| かなり強い | 85 | 0.5 | 99.5 | 12 | 250000 |

Use weaker presets first in audit-sensitive workflows. Stronger presets are useful for exploration, but they can also amplify noise, JPEG artifacts, fur texture, dirt, shadows, and other non-semantic patterns.

## Parameters

| Parameter | Meaning |
|---|---|
| `targetStd` | Target standard deviation for each decorrelated principal component. Larger values produce stronger color separation. |
| `lowPercentile` | Lower percentile used for output normalization. Values below this percentile are mapped toward black. |
| `highPercentile` | Upper percentile used for output normalization. Values above this percentile are mapped toward white. |
| `maxScale` | Upper bound for each component scale factor. This prevents tiny-variance directions from being amplified too aggressively. |
| `samplePixels` | Maximum number of pixels sampled from the effective region to estimate the RGB covariance matrix. Set to `0` to use all pixels. |
| `randomSeed` | Seed for reproducible random sampling. |

## Region selection

You can drag on the original image to define a rectangular region. When a region is selected:

- RGB statistics are estimated only from the selected region.
- The transform is applied only to pixels inside the selected region.
- Pixels outside the selected region remain unchanged in the result.
- The selected region is recorded in the metadata JSON.

This is useful when the target information occupies only part of the image. For example, if a sprayed number is located on an animal body, selecting the body or number area can reduce the influence of unrelated background colors.

By default, the tool applies Decorrelation Stretch to the full image.

## Output files

### PNG image

The enhanced image is saved as PNG.

PNG is preferred because it is lossless. JPEG compression can add artifacts, and Decorrelation Stretch may make those artifacts more visible.

### Metadata JSON

The metadata JSON contains information required to reproduce and review the processing result.

Typical fields include:

```json
{
  "algorithm": "whole_image_decorrelation_stretch",
  "appVersion": "0.1.0",
  "createdAt": "2026-06-07T00:00:00.000Z",
  "sourceFileName": "input.jpg",
  "width": 4000,
  "height": 3000,
  "region": {
    "x": 0,
    "y": 0,
    "width": 4000,
    "height": 3000
  },
  "params": {
    "targetStd": 55,
    "lowPercentile": 1,
    "highPercentile": 99,
    "maxScale": 8,
    "samplePixels": 250000,
    "randomSeed": 0
  },
  "meanRgb": [42.1, 39.8, 36.4],
  "covarianceRgb": [[...], [...], [...]],
  "eigenvalues": [...],
  "eigenvectors": [[...], [...], [...]],
  "rawScales": [...],
  "appliedScales": [...],
  "lowRgb": [...],
  "highRgb": [...],
  "userAgent": "...",
  "note": "This output is an enhanced visualization derived from the original image. Use it as an inspection aid, not as a replacement for the original evidence image."
}
```

## How it works

### 1. Pixel representation

The browser decodes the image into Canvas `ImageData`.

Each pixel is stored as RGBA:

```text
[R, G, B, A]
```

The algorithm uses only RGB values for color statistics and transformation. The alpha channel is preserved in the final output.

For each processed pixel, define the RGB vector:

```text
x_i = [R_i, G_i, B_i]^T
```

If the effective region contains `N` pixels, the RGB data can be considered as a matrix:

```text
X ∈ R^{N×3}
```

Each row corresponds to one pixel.

### 2. Mean RGB vector

The tool estimates the mean RGB vector from the effective region:

```text
μ = [μ_R, μ_G, μ_B]^T
```

where:

```text
μ_R = (1/N) Σ R_i
μ_G = (1/N) Σ G_i
μ_B = (1/N) Σ B_i
```

If `samplePixels > 0`, the mean is estimated from a reproducible random sample of the region. If `samplePixels = 0`, all pixels in the region are used.

The centered RGB vector is:

```text
x'_i = x_i - μ
```

Mean centering is necessary because covariance describes variation around the average color, not absolute RGB values.

### 3. RGB covariance matrix

The RGB covariance matrix is computed from centered RGB vectors:

```text
C = 1 / (N - 1) Σ x'_i x'_i^T
```

For RGB images, this is a `3 × 3` symmetric matrix:

```text
C =
[ var(R)    cov(R,G)  cov(R,B) ]
[ cov(G,R)  var(G)    cov(G,B) ]
[ cov(B,R)  cov(B,G)  var(B)   ]
```

This matrix describes how RGB channels vary together.

In many natural images, the RGB channels are strongly correlated. Illumination changes often increase or decrease R, G, and B together. Ordinary brightness or contrast adjustment mostly stretches this dominant brightness-like direction. It does not explicitly separate small chromatic differences from correlated RGB variation.

### 4. Eigenvalue decomposition

The tool performs eigenvalue decomposition of the symmetric covariance matrix:

```text
C = E Λ E^T
```

where:

- `E` is a matrix of eigenvectors.
- `Λ` is a diagonal matrix of eigenvalues.

The eigenvectors define the principal axes of the RGB color distribution. The eigenvalues represent variance along those axes.

The implementation uses a small Jacobi method specialized for a `3 × 3` real symmetric matrix. This avoids external numerical libraries.

The eigenpairs are sorted in descending order of eigenvalue:

```text
λ_0 ≥ λ_1 ≥ λ_2
```

### 5. Transform to decorrelated component space

Each centered RGB vector is rotated into the eigenvector basis:

```text
y_i = E^T x'_i
```

Equivalently, depending on row/column convention, the implementation computes dot products between the centered RGB vector and each eigenvector.

In this space, the components are decorrelated under the estimated covariance model:

```text
cov(Y) = Λ
```

This means that each component corresponds to an independent principal direction of color variation.

### 6. Stretch each component

Each decorrelated component is scaled so that its standard deviation approaches `targetStd`.

The raw scale for component `j` is:

```text
rawScale_j = targetStd / sqrt(max(λ_j, ε))
```

where `ε` is a small value used to avoid division by zero.

If `λ_j` is small, the corresponding direction had little variation in the original image. Stretching that direction may make subtle color differences visible.

However, very small eigenvalues can also represent noise, compression artifacts, or nearly constant directions. Therefore the scale is capped:

```text
scale_j = min(rawScale_j, maxScale)
```

The stretched component is:

```text
z_{i,j} = y_{i,j} · scale_j
```

### 7. Transform back to RGB

The stretched vector is transformed back to RGB space:

```text
u_i = E z_i + μ
```

`u_i` is a floating-point RGB vector. It may contain values below `0` or above `255`, so it is not yet directly displayable.

### 8. Percentile-based output normalization

For all transformed pixels in the effective region, the tool computes lower and upper percentile values independently for R, G, and B:

```text
low_c  = percentile(U_c, lowPercentile)
high_c = percentile(U_c, highPercentile)
```

where `c ∈ {R, G, B}`.

Each channel is normalized as:

```text
out_c = 255 · (u_c - low_c) / (high_c - low_c)
```

Then the value is rounded and clamped to `[0, 255]`.

Percentiles are used instead of raw minimum and maximum because a small number of extreme pixels can otherwise dominate the output range. This is especially important for photos containing flash reflections, saturated highlights, sensor noise, or compression artifacts.

### 9. Output composition

The processed RGB values replace the corresponding pixels inside the effective region.

Pixels outside the selected region are copied unchanged.

The alpha channel is copied from the original image.

## Why a Web Worker is used

Decorrelation Stretch requires several full-pixel passes:

1. Mean estimation.
2. Covariance estimation.
3. Transformation.
4. Percentile extraction.
5. Normalization.

For large images, these passes can take noticeable time. Running them on the main browser thread would freeze the UI. The application therefore runs the algorithm in an inline Web Worker and sends progress messages back to the main thread.

## Why WebAssembly is not used

This implementation intentionally avoids WebAssembly.

The heavy part is not the `3 × 3` eigenvalue decomposition; it is the repeated traversal of image pixels. For a first version, plain JavaScript with TypedArrays and a Web Worker is simple and portable.

WebAssembly may be considered later if the project adds heavier operations such as CLAHE, Retinex, denoising, deblurring, batch processing, or very large multi-image workflows.

## Practical interpretation

Decorrelation Stretch can make weak color differences visible, but it can also exaggerate irrelevant patterns.

Be careful with:

- JPEG compression artifacts.
- Sensor noise in low-light images.
- Dirt, fur, fabric texture, or wall texture.
- Blood, mud, shadows, and specular highlights.
- Very strong presets.

Recommended workflow:

1. Start with the original image.
2. Try **色差強調 弱** or **色差強調 中**.
3. If needed, select a region around the target area.
4. Compare the enhanced view with the original.
5. Save both the PNG result and JSON metadata when the result is used for review.
6. Treat features visible only under very strong enhancement as uncertain.

## Limitations

- The tool does not perform OCR.
- The tool does not automatically detect numbers, animals, paintings, characters, or patterns.
- The tool does not remove blur.
- The tool does not perform denoising.
- The tool does not perform illumination correction.
- The tool does not guarantee that an enhanced feature is semantically meaningful.
- The selected-region mode processes only the selected rectangle and leaves the rest of the image unchanged.
- The percentile calculation uses sorted arrays, which is simple and accurate but may be memory-intensive for very large regions.

## Troubleshooting

### The result is too noisy

Use weaker parameters:

- lower `targetStd`
- lower `maxScale`
- use **色差強調 弱**

### The result is too weak

Use stronger parameters:

- increase `targetStd`
- increase `maxScale`
- use **色差強調 強**

### The entire image is influenced by irrelevant background colors

Drag a rectangle around the relevant area and run the correction again.

### Processing is slow

Try selecting a smaller region. Also reduce the input image size if extremely high-resolution images are not required.

### PNG drag export does not work

Use the **PNG保存** button instead. Drag export behavior may vary by browser and operating system.

## License

See [LICENSE](LICENSE).
