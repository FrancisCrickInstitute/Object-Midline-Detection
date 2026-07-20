# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A small image-analysis pipeline that extracts a **midline** (medial axis / spine) through
each object in a single-channel greyscale image (e.g. DAPI from a 10x Xenium run). For every
detected structure it computes three candidate midlines and exports them, plus the object
boundaries, to GeoJSON.

There is no application code, package, or test suite — this is a scientific analysis
script/notebook pair operating on one example image.

## Running

```bash
pip install numpy tifffile scikit-image scipy matplotlib skan networkx
jupyter lab midline-detection.ipynb
```

`midline-detection.ipynb` is the canonical, documented version (meant to be run top to
bottom, also runnable via the Binder badge in README.md with no local install).
`midline-detection.py` is a plain-script duplicate of the same pipeline — useful for
running headless — but it is currently untracked/WIP, has no GeoJSON export step, and
writes different output filenames than the notebook (see Outputs below).

Note: `requirements.txt` is missing `networkx`, which both the notebook and the `.py`
script import (via `skan`'s branch graph). Install it manually as shown above.

There is no lint, format, test, or CI configuration in this repo.

## Pipeline architecture

Both the notebook and the script implement the same sequence of stages; understanding
the flow across them is more useful than any single function:

1. **`load_mask(path)`** — reads the 16-bit TIFF, max-projects if a stack/multichannel
   image sneaks in, Gaussian-blurs (`SMOOTH_SIGMA`) to suppress DAPI speckle, thresholds
   with global Otsu, then cleans up the binary mask (fill holes, remove small holes
   `MIN_HOLE_AREA`, morphological closing `CLOSE_RADIUS`, drop debris
   `MIN_OBJECT_AREA`). Returns `(raw_intensities, mask)`.
2. **`label` + `regionprops`** (skimage) split the mask into individual structures, each
   processed independently by every method below.
3. Three independent midline methods, run per-object:
   - **`longest_skeleton_path`** — skeletonize the object, build a branch graph with
     `skan`/`networkx`, and take the geodesic-longest endpoint-to-endpoint path via
     Dijkstra.
   - **`geodesic_longest_path`** — distance-transform the object (ridge = medial axis),
     build a cost map that's cheap near the ridge, then find the two farthest-apart
     skeleton pixels and route between them with `skimage.graph.route_through_array`.
     This is the basis for method 3.
   - **`smooth_path`** — fits a cubic B-spline (`splprep`/`splev`) through the geodesic
     path, resamples at `n_out` uniform points, and clips any point that falls outside
     the mask (so smoothing can't bow the curve out through a concave boundary).
4. **Visualization** — a 3-panel comparison figure (skeleton path / geodesic path /
   smoothed spline), each panel also showing the object outlines from
   `find_contours`.
5. **GeoJSON export** (notebook only, see `all_paths` cell) — writes
   `outputs/midlines.geojson` (one `LineString` per object per method, tagged with
   `label`/`method`) and `outputs/boundaries.geojson` (one `Polygon` per object, tagged
   with `label`/`area_px`). Both share the per-object `label` so they can be joined on
   reload.

### Coordinate convention

Internally all geometry is `(row, col)` = `(y, x)` image order (numpy/skimage
convention). GeoJSON requires `(x, y)`, so every export step swaps to `[col, row]`
when writing coordinates. Since image `y` increases *downwards*, overlaying GeoJSON
output on the source image with `imshow` is correct as-is, but plotting on
conventional (y-up) axes needs the y-axis inverted.

### Tunable parameters

Both the notebook (top "Parameters" cell) and the script (module-level constants) expose
the same four knobs — adjust these rather than the pipeline logic when the mask looks
wrong on a new dataset:

- `SMOOTH_SIGMA` — pre-threshold Gaussian blur; higher suppresses more speckle but rounds
  off fine structure.
- `MIN_OBJECT_AREA` — debris cutoff.
- `MIN_HOLE_AREA` — interior gaps below this are filled.
- `CLOSE_RADIUS` — morphological closing radius, bridges thin gaps before skeletonizing.

If the structure count printed after masking looks wrong (debris counted as objects,
touching structures merged), fix it by adjusting these masking parameters — every
downstream step (all three midline methods, GeoJSON export) works off the resulting
labels.

## Directory layout

- `inputs/` — source TIFFs (currently one example Xenium DAPI image).
- `outputs/` — generated PNGs and GeoJSON; these are committed example outputs, not
  gitignored build artifacts, so treat existing files here as reference/example data
  rather than freely overwritable scratch space.