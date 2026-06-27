# Change Analysis Report — Open-Pit Mine, Zambia

**Site:** open-pit copper mining area, Zambia
**Imagery:** Sentinel-2, Before = 2023-08-12, After = 2023-09-02 (21-day interval)
**Input 1:** the 3 provided bands — B02 (Blue), B03 (Green), B04 (Red)
**Input 2:** an 11-band Sentinel-2 product built from raw Copernicus data

Two pipelines are submitted:

- **`01_change_detection_rgb`** — the full pipeline on the
  provided 3-band imagery.
- **`02_change_detection_11band`** — the same change-detection approach on
  a richer 11-band product, which additionally enables NDVI.

---

## 1. Method

### 1.1 Data preparation

For each date the bands are read, checked for **CRS / transform / dimension
consistency**, and stacked into a single multi-band image. The two dates are
**co-registered** (the second image is resampled onto the first's grid where the
grids differ) so each pixel compares the same ground location.

- *3-band:* provided data, already AOI-clipped and in metric UTM, so areas are in
  m² directly.
- *11-band:* the product is in geographic coordinates (OGC:CRS84 / lat-lon) at a
  different grid per date, so the pipeline aligns the two dates and then reprojects
  to **UTM 35S (EPSG:32735)** (≈10 m pixels, 100 m²/px) so polygon areas are in
  true metres.

### 1.2 Change detection — two methods

**(a) Provided baseline — Euclidean distance.** The supplied
`compute_change_distance` is applied unchanged: per-pixel Euclidean distance
between the before/after band vectors, normalized to 0–1. Useful as a reference,
but weighting all bands equally it is dominated by the brightest surfaces.

**(b) PCA-based change detection.** The *after* image is first
**radiometrically normalized** to the *before* image (per-band mean/standard-deviation
matching) so the difference reflects real surface change rather than sensor or
atmospheric offset. The per-pixel band-difference vector is then **standardized and
decomposed with PCA**, and the change signal is the Euclidean magnitude of the
leading principal components. PCA concentrates the dominant, decorrelated change
into the first components and pushes correlated illumination noise into the minor
components, which are dropped — producing a cleaner, better-localized map than the
raw Euclidean baseline.

### 1.3 Thresholding and cleanup

The continuous change magnitude is turned into a binary map by: a **P98 percentile
threshold** (P95 flagged far too much — it flags ~5 % of pixels by construction;
P98 keeps only the strongest change), **morphological opening/closing**, and a
**connected-component size filter** (`MIN_REGION = 50` px) that removes isolated
speckle so only spatially-coherent change survives.

The binary change is **vectorized to polygons** and stored in a **SQLite**
`change_features` table with the required schema
`id | date_before | date_after | area_m2 | confidence | geometry`, where
*confidence* is derived from the change magnitude inside each polygon. A
**GeoPackage** is also exported for GIS inspection.

### 1.4 The 11-band pipeline — provenance and NDVI

To work with richer spectral information than RGB, an 11-band product was built
directly from the raw Copernicus data:

1. Downloaded the same-date Sentinel-2 scenes from **Copernicus**.
2. Defined and clipped the **AOI in QGIS**.
3. **Stacked the required bands** into a single multi-band image.
4. Extracted the **ROI in ENVI** and saved a **georeferenced ENVI** file (BIP
   interleave, with a `.hdr`).

This pipeline reads the ENVI pair via GDAL (header supplies shape/dtype/CRS),
aligns the two dates, reprojects to UTM 35S, applies a **seam correction** for the
mosaic illumination step in the *after* image, and runs the same PCA change
detection. Because the 11-band stack includes the near-infrared band, it also
computes **NDVI** = (NIR − Red)/(NIR + Red) for both dates, giving a vegetation
signal that RGB alone cannot provide.


### 1.5 Why PCA

The Euclidean baseline treats all bands equally and is driven by the brightest,
highest-variance surfaces — here the near-white tailings and processing areas — so
a small brightness fluctuation there outweighs genuine change on dark vegetation,
producing false positives where nothing changed. PCA standardizes the difference
and isolates the coherent axes of change; the P98 threshold and size filter then
suppress residual bright-surface speckle. The result is a conservative map showing
only the strongest, spatially-coherent change.

---

## 2. Results

### 2.1 3-band pipeline

- **115 change regions**
- **≈ 3.39 km² of detected change**
- **≈ 1.28 % of the scene**

Change concentrates along the **edges of the tailings and pond areas** (where the
water/slurry extent shifts), at **active pit and waste-dump faces**, and in
**narrow linear features** consistent with haul roads and infrastructure. The
surrounding forest is largely unchanged — the expected behaviour and a useful
sanity check. The PCA intensity map localizes change far more cleanly than the
provided Euclidean baseline, which spreads apparent change across the bright
surfaces.

### 2.2 11-band pipeline

- **PCA explained variance:** 0.696 / 0.156 / 0.093 for the leading components
- **P98 threshold → ≈ 2.18 km² changed (≈ 0.83 % of the valid scene)**
- **144 change polygons**
- plus **NDVI** computed for both dates

The 11-band result is in the same range as the 3-band run and again concentrates
on the mining/tailings areas. The leading principal component captures most of the
variance (0.70), and the additional bands — especially NIR via NDVI — provide
vegetation contrast that RGB cannot. Exact agreement with the 3-band figures is
not expected, as the two products differ in source processing, grid, resolution,
and the seam correction applied to the 11-band mosaic.

---

## 3. Interpretation

The detected change is best described as **active open-pit mining surface change**
over a short, dry-season interval, and is almost certainly a **mixture** of
processes:

- **Earthmoving / pit and dump advancement** — genuine excavation and material
  relocation at active faces; the change most relevant to monitoring mine
  expansion.
- **Tailings-pond and water-level change** — much of the strongest signal sits on
  the boundaries of the bright tailings and water bodies. In the Zambian dry season
  (mid-Aug → early Sep) ponds recede and slurry surfaces change, producing large,
  real, but largely **seasonal/operational** change rather than "expansion."
- **Infrastructure** — linear detections consistent with haul-road and facility
  change.
- **Residual noise / artifacts** — some signal near very bright surfaces and from
  acquisition-illumination differences (the mosaic seam in the 11-band data). The
  radiometric normalization, seam correction, P98 threshold and size filter are
  designed to suppress this, though optical data alone cannot remove it entirely.

**Key caveat.** With visible bands alone the method establishes *that* and *where*
a surface changed, not its physical nature; it cannot, by itself, separate genuine
excavation from dewatering or seasonal effects. The reported area is therefore
**"detected surface change," not a direct measurement of mine expansion**, and it
scales with the chosen threshold and minimum-region size, which are explicit
parameters. The 11-band NDVI is a first step toward separating these processes
(vegetation vs. bare-earth change); NDWI would similarly isolate water-extent
change.

---

## 4. Summary

A reproducible change-detection pipeline was built on the provided 3-band
Sentinel-2 imagery — load and stack, radiometric normalization, PCA change
detection (alongside the provided Euclidean baseline), P98 thresholding with a
spatial-coherence filter, vectorization to polygons, storage in a SQLite database,
and visualization with the AOI (static figure + interactive Folium map). It detects
≈ 3.39 km² of coherent surface change concentrated at the active mining and
tailings areas. A second pipeline applies the same method to an 11-band
product built from raw Copernicus data (QGIS clip → band stack → ENVI ROI →
georeferenced export), adds a seam correction and NDVI, and detects ≈ 2.18 km² of
change — demonstrating the approach generalizes beyond RGB. The principal
limitation — that optical change detection identifies change without classifying
its cause — is documented, along with the multispectral indices that address it.
