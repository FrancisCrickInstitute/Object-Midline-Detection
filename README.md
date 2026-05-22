# Object Midline Detection

[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/FrancisCrickInstitute/Object-Midline-Detection/HEAD?urlpath=%2Fdoc%2Ftree%2Fmidline-detection.ipynb)

Extract a **midline** through each object in a single greyscale image
(e.g. DAPI from a 10x Xenium run). For each detected structure it computes three candidate midlines — a skeleton longest path,
a distance-transform-weighted geodesic path, and a smoothed spline — and exports all three
to GeoJSON so you can pick what works best for your data.

## Try it

Click the **launch binder** badge above to run the notebook in your browser — no install
needed. Everything is documented inline; just run the cells top to bottom.

## Run locally

```bash
pip install numpy tifffile scikit-image scipy matplotlib skan
jupyter lab midline-detection.ipynb
```
